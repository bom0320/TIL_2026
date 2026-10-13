# v1 E2E Latency Troubleshooting

## 문제

v1 E2E를 실제로 실행해보니 기능 자체는 정상적으로 동작했지만, 작은 README 수정 작업에도 전체 실행 시간이 지나치게 길었다.

초기 측정값은 다음과 같았다.

```text
Planning:      53.7s
Worker:       167.0s
Verification:  17.5s
Total:        265.1s
```

단순히 "LLM이 느리다"라고 보기에는 실행 시간이 너무 길었다.

실제 흐름을 살펴보니 핵심 문제는 모델 자체의 속도보다 다음과 같은 **중복 작업과 불필요한 LLM 호출**에 있었다.

```text
불필요한 Planning round
중복 Verification
중복 Repository 탐색
```

이번 개선의 목표는 AI 호출 자체를 빠르게 만드는 것이 아니라,

> **AI를 불필요하게 여러 번 호출하지 않도록 Harness의 실행 구조를 개선하는 것**

이었다.

---

# 1. Telemetry 추가

먼저 어디에서 시간이 소비되는지 정확히 알아야 했다.

기존에는 전체 실행 시간이 길다는 사실만 알 수 있었고, 각 단계별 병목을 구분하기 어려웠다.

그래서 다음 단계의 실행 시간을 측정하도록 Telemetry를 추가했다.

```text
Planning
Approval Wait
Workspace

Worker
  - Setup
  - Codex Process
  - Output Processing

Verification
Cleanup
Total
```

측정 결과는 CLI에 출력하고, 각 실행의 `result.json`에도 함께 기록하도록 했다.

## 확인된 결과

실제 병목은 Harness 자체의 처리 로직이 아니었다.

대부분의 시간은 다음 두 구간에서 발생했다.

```text
Planning의 반복 LLM 호출
Worker의 Codex Process
```

반면 다음 작업들은 거의 비용이 없었다.

```text
Workspace 생성
Cleanup
Execution Artifact 처리
```

즉 최적화의 방향도 자연스럽게 명확해졌다.

```text
Harness 내부 코드 최적화
X

불필요한 LLM 호출과 중복 작업 제거
O
```

---

# 2. Worker와 Verification의 책임 분리

## 문제

초기에는 Worker가 `TaskContract`의 verification 명령을 확인한 뒤 직접 검증까지 수행했다.

예를 들어:

```text
pnpm typecheck
pnpm test
git diff --check
```

를 Worker가 실행했다.

하지만 Worker 실행이 끝난 뒤 Harness의 공식 Verification 단계에서도 동일한 명령을 다시 실행하고 있었다.

결과적으로 다음 구조가 만들어졌다.

```text
Worker
├─ 코드 수정
└─ Verification 실행

↓

Harness
└─ 공식 Verification 실행
```

같은 검증 작업이 두 번 실행되고 있었다.

## 개선

Worker와 Harness의 책임을 명확히 분리했다.

```text
Worker
= 변경 생성

Harness
= 공식 Verification
```

Worker prompt에도 다음 원칙을 명시했다.

```text
full test/typecheck를 직접 수행하지 않는다.
변경 생성에 집중한다.
공식 검증은 Harness의 Verification 단계가 담당한다.
```

## 결과

Worker 실행 시간은 다음과 같이 감소했다.

```text
167.0s
→
88.2s
```

약 **47% 감소**했다.

이후 추가 최적화를 적용한 실행에서는:

```text
55.7s
```

수준까지 내려갔다.

이 결과를 통해 Worker 내부 검증이 실제 latency에서 상당한 비중을 차지하고 있었음을 확인할 수 있었다.

---

# 3. Planning Warm Start

## 문제

작은 README 수정 작업인데도 Planning은 3 rounds를 사용했다.

기존 흐름은 다음과 같았다.

```text
Goal

↓

Planner

↓

inspect

↓

Planner

↓

inspect

↓

Planner

↓

complete
```

Planner가 첫 호출 시점에 Repository에 대한 정보를 거의 가지고 있지 않기 때문에, 초반 round를 Repository 탐색에 소비하고 있었다.

작업 자체는 단순해도 Planner 입장에서는 매번 Cold Start였다.

## 개선

첫 Planner 호출 전에 기본적인 Repository Context를 미리 제공했다.

예를 들어:

```text
README
package.json
Top-level repository structure
Goal에 명확히 언급된 파일
```

을 Initial Context에 포함했다.

즉 Planning을:

```text
Cold Start
→
Warm Start
```

로 변경했다.

Planner prompt도 함께 수정했다.

```text
충분한 Context가 있으면 즉시 complete한다.

단순히 더 알아보기 위해 inspect하지 않는다.

TaskContract 생성을 위해 실제로 필요한 경우에만
추가 파일을 요청한다.
```

## 결과

Planning 시간과 round 수가 모두 감소했다.

```text
53.3s / 3 rounds

→

30.0s / 2 rounds
```

약 **44% 감소**했다.

이후 반복 실행에서도:

```text
약 35s / 2 rounds
```

수준으로 유지됐다.

---

# 4. Planning Budget을 Planner에게 공유

## 문제

Planning 과정에서 한 번은 Planner가 한 round에서 너무 많은 파일을 요청해 실패했다.

```text
Inspection request exceeds maxFilesPerRequest.
```

Harness에는 이미 다음 제한이 존재했다.

```text
maxRounds
maxFilesPerRequest
maxTotalFiles
```

하지만 문제는 **Planner가 이 제한을 알지 못한다는 것**이었다.

기존 구조는 사실상 다음과 같았다.

```text
Planner
→ 원하는 만큼 파일 요청

↓

Harness
→ limit 초과 여부 검사

↓

실패
```

Planner는 사용할 수 없는 요청을 생성하고, Harness는 사후에 이를 차단하고 있었다.

## 개선

PlannerContext에 현재 Planning Budget을 전달했다.

```ts
limits;
remainingInspectionBudget;
```

이후 구조는 다음처럼 변경됐다.

```text
Planner
= 현재 budget을 알고 요청을 결정

Harness
= 실제 limit을 최종 강제
```

즉 Planner에게 제약 조건에 대한 정보를 제공하되, 최종 안전성은 여전히 Harness가 보장한다.

## 결과

불필요한 limit 초과 실패를 줄일 수 있었고, 기존에 실패하던 동일 Goal도 정상적으로 Planning을 완료했다.

중요한 점은 limit 자체를 완화한 것이 아니라:

> **Planner가 자신이 사용할 수 있는 자원을 알고 판단하도록 만든 것**

이다.

---

# 5. Planning Context를 Worker가 재사용

## 문제

Planning 단계에서 이미 README 같은 파일을 읽었는데도 Worker는 다시 Repository를 탐색하고 있었다.

기존 흐름은 다음과 같았다.

```text
Planning
→ Repository 파일 조사
→ TaskContract 생성

↓

Worker
→ Repository 다시 조사
→ 수정
```

Planning과 Worker 사이에서 Repository Context가 단절돼 있었다.

TaskContract에는 다음 정보만 담겨 있었다.

```text
무엇을 해야 하는가
어떤 범위에서 해야 하는가
어떻게 검증할 것인가
```

하지만 Planning 과정에서 확보한 실제 Repository 정보는 Worker에게 전달되지 않았다.

## 개선

TaskContract와 별도로 `WorkerContext`를 도입했다.

역할은 다음처럼 구분했다.

```text
TaskContract
= 무엇을 해야 하는가

WorkerContext
= Planning에서 이미 확보한 Repository 참고 정보
```

Worker prompt에도 Context 재사용 원칙을 추가했다.

```text
Planning Context를 먼저 활용한다.

이미 전달받은 파일은 불필요하게 다시 읽지 않는다.

필요한 경우에만 workspace에서 재확인한다.

실제 source of truth는 항상 workspace이다.
```

Context가 지나치게 커지는 것을 막기 위해 크기 제한도 추가했다.

```text
파일당 최대 64 KiB
전체 최대 128 KiB
```

## 결과

Worker baseline은 약:

```text
55.7s
```

였다.

Context reuse 적용 후 한 실행에서는:

```text
42.1s
```

까지 감소했다.

하지만 다른 실행에서는:

```text
68.0s
```

가 나오기도 했다.

따라서 현재 판단은 다음과 같다.

```text
구조적인 개선
= 확실함

성능 개선 효과
= 추가 샘플 필요
```

LLM 기반 실행은 자체적인 latency 변동이 있기 때문에 단일 실행 결과만으로 성능 향상을 단정하기는 어렵다.

---

# 전체 성과

초기 E2E 결과:

```text
Planning:      53.7s
Worker:       167.0s
Verification:  17.5s
Total:        265.1s
```

개선 후 대표 실행:

```text
Planning:      30.0s
Worker:        55.7s
Verification:  18.7s
Total:        117.7s
```

전체 실행 시간은:

```text
265.1s
→
117.7s
```

약 **55% 감소**했다.

이번 개선에서 제거한 주요 낭비는 다음과 같다.

```text
불필요한 Planning round
중복 Verification
중복 Repository 탐색
```

---

# 현재 남아 있는 병목

현재 각 단계의 실행 시간은 대략 다음 수준이다.

```text
Planning: 30~35s
Worker:   40~70s
Verify:   18~20s
```

현재 가장 큰 병목은 여전히:

```text
Planner LLM 호출
Codex Worker LLM 호출
```

이다.

반대로 Harness 자체의 overhead는 거의 무시할 수 있는 수준이다.

따라서 다음 최적화 역시 Harness 내부의 미세한 코드 성능보다 **LLM 호출 횟수와 Context 비용을 줄이는 방향**이 더 효과적이다.

---

# 다음 개선 후보

## 1. Deterministic Planning Fast Path

README 수정처럼 대상 파일이 명확하고 작업이 단순한 경우에는 Planner LLM을 호출하지 않는 Fast Path를 고려할 수 있다.

예:

```text
Goal

↓

명확한 단일 파일
+
단순 수정 작업

↓

Harness가 직접 TaskDraft 생성
```

이 경우 현재:

```text
Planning: 30~35s
```

인 구간을 거의:

```text
Planning: ~0s
```

수준까지 줄일 수 있다.

핵심은 모든 Planning을 제거하는 것이 아니라:

> **Planning이 필요하지 않은 작업을 먼저 구분하는 것**

이다.

---

## 2. Selective File Inspection

현재는 Planner에게 파일 전체 내용을 전달하는 경우가 있다.

큰 파일이라면 처음부터 전체를 읽기보다 다음 정보만 먼저 제공할 수 있다.

```text
상위 N줄
Markdown Heading
주요 구조
파일 metadata
```

추가 내용이 필요한 경우에만 특정 range를 다시 읽는다.

목표는 다음과 같다.

```text
Prompt 크기 감소
Token 사용량 감소
LLM latency 감소
```

---

## 3. Context Deduplication

현재 `inspectedFiles`를 통해 Context를 재사용하고 있지만, 같은 내용이 Planner와 Worker prompt에 여러 형태로 중복 삽입될 가능성은 남아 있다.

따라서 이후에는:

```text
같은 파일
같은 내용
같은 Repository 정보
```

가 불필요하게 반복되지 않도록 Context deduplication을 더 엄격하게 적용할 수 있다.

---

## 4. Task 성격별 Verification

현재 README 같은 문서 수정에도 TaskContract에 따라:

```text
pnpm typecheck
pnpm test
git diff --check
```

가 모두 실행될 수 있다.

하지만 문서-only 작업이라면 전체 애플리케이션 검증은 비용 대비 과할 수 있다.

예를 들어:

```text
Document-only task

→

git diff --check
```

처럼 Verification 전략을 Task 성격에 따라 다르게 선택할 수 있다.

즉 Verification 역시:

```text
항상 동일한 검증

↓

Task-aware Verification
```

으로 발전시킬 수 있다.

---

# 결론

이번 개선 과정에서 가장 크게 느낀 점은 다음이다.

> **AI 시스템의 성능 최적화는 모델 자체를 빠르게 만드는 문제가 아니라, 모델에게 불필요한 일을 시키지 않는 구조를 만드는 문제에 가깝다.**

이번에는:

```text
Planning Warm Start
Worker / Verification 책임 분리
Planning Budget 공유
Planning Context 재사용
```

을 적용했다.

그 결과 전체 E2E latency를:

```text
265.1s
→
117.7s
```

로 약 **55% 줄일 수 있었다.**

그리고 현재 다음 단계는 단순한 추가 튜닝이 아니라,

```text
모든 작업이 동일한 Planning 비용을 지불해야 하는가?
모든 작업이 동일한 Verification을 거쳐야 하는가?
이미 확보한 Context를 어떻게 더 효율적으로 재사용할 것인가?
```

를 기준으로 **작업의 성격에 따라 Harness가 실행 경로를 다르게 선택하는 방향**으로 넘어가고 있다.
