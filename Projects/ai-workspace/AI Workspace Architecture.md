# AI Workspace Architecture

## 1. 가장 먼저 보는 기준

파일 위치는 아래 순서로 판단한다.

```
1. 이 코드는 어떤 use-case에 속하는가?
2. 그 use-case 안에서 어떤 책임을 가지는가?
3. 그 책임이 다른 use-case에서도 독립적으로 의미가 있는가?
4. 외부 기술을 다루는 구현 세부사항인가?
```

중요한 원칙:

> **사용 기술이 아니라 책임과 소유권으로 배치한다.**

예:

```
Codex를 사용한다
≠ codex 폴더에 무조건 모은다

Planning 판단을 한다
→ application/planning

Execution 작업을 수행한다
→ application/execution
```

---

# 2. 최상위 레이어 의미

현재 기준:

```
src/
├─ application/
├─ contracts/
├─ services/
└─ infrastructure/
```

## `application/`

사용자의 목적을 실제 workflow로 수행하는 곳.

예:

```
Goal → TaskContract
TaskContract → ExecutionResult
```

즉:

```
planning
execution
```

같은 **use-case 단위**가 들어간다.

---

## `contracts/`

application 하나에 종속되지 않는 시스템 전반의 계약.

예:

```
GoalSpec
TaskContract
```

이런 타입은 Planning 내부 상태가 아니라 여러 레이어에서 공통으로 의미가 있다.

그래서:

```
contracts/
```

에 둔다.

---

## `services/`

특정 use-case의 내부 단계가 아니라,

> **독립 capability로서 의미가 있는 기능**

을 둔다.

예:

```
artifacts
evidence
scope
verification
```

이 기능들은 Execution에서 주로 사용되지만 개념 자체가:

```
artifact 저장
evidence 수집
scope 검사
verification 수행
```

처럼 독립적이다.

---

## `infrastructure/`

Git, Process, filesystem, Codex CLI 같은:

> **기술적 실행 수단**

을 다룬다.

예:

```
git
process
```

나중에 generic Codex runner가 실제로 필요해진다면:

```
infrastructure/codex
```

가 될 수 있다.

---

# 3. `application/<use-case>` 구조

기본 구조는:

```
application/
└─ <use-case>/
   ├─ flow/
   ├─ model/
   ├─ protocol/   // 필요한 경우
   ├─ <role>/     // 필요한 경우
   └─ index.ts
```

예:

```
application/
├─ planning/
└─ execution/
```

이 둘은 파일 개수가 같을 필요는 없다.

중요한 건 **같은 분류 기준을 적용하는 것**이다.

---

## 각 폴더의 의미

### `flow/`

```
flow
= 해당 use-case가 실제로 진행되는 과정의 application logic
```

단순히 orchestration만 의미하지 않는다.

예를 들어 Planning이라면:

```
Goal
→ repository 조사
→ Planner 판단
→ 추가 inspection
→ TaskDraft
→ TaskContract
```

이 과정에 직접 속하는 로직이 `flow`에 들어간다.

즉:

> **use-case가 시작되어 결과를 만들기까지 필요한 진행 로직**

이라고 보면 된다.

`flow` 내부는 필요에 따라 더 세분화할 수 있다.

예:

```
flow/
├─ plan-task.ts
├─ orchestration/
├─ inspection/
└─ transformation/
```

여기서:

```
orchestration
= 흐름 제어

inspection
= 흐름 중 필요한 정보 수집

transformation
= 흐름 중 데이터 형태 변환
```

이다.

중요한 점은 `inspection`, `transformation`은 `model`, `protocol`과 같은 architectural segment가 아니라 **flow 내부 구현 책임을 나누는 하위 grouping**이라는 점이다.

---

### `model/`

```
model
= 해당 use-case가 소유하는 상태, 설정, 결과
```

예:

Planning:

```
PlanningState
PlanningLimits
PlanningResult
RepositoryInfo
```

Execution:

```
ExecutionState
ExecutionResult
```

중요:

```
model
≠ 모든 타입을 넣는 곳
```

시스템 전반의 계약은 `contracts`,

역할 사이 통신 타입은 `protocol`에 둔다.

---

### `protocol/`

```
protocol
= 컴포넌트나 역할 사이에서 주고받는 구조화된 메시지 계약
```

Planning 예:

```
PlannerContext
PlannerDecision
InspectRequest
InspectResult
TaskDraft
```

흐름으로 보면:

```
Harness
→ PlannerContext
→ Planner

Planner
→ PlannerDecision
→ Harness
```

즉 내부 상태가 아니라 **역할 사이 communication format**이다.

`protocol`은 항상 필요한 것은 아니다.

역할 간 구조화된 경계가 분명할 때만 만든다.

---

### `<role>/`

해당 use-case 안에 명확한 역할이 존재할 때 둔다.

예:

Planning:

```
planner/
└─ codex-planner.ts
```

Execution:

```
worker/
└─ codex-worker.ts
```

중요한 기준:

```
codex-planner
= Codex 코드라서 planner에 있는 것이 아님

Planning Planner 역할을 구현하기 때문에 planner에 있음
```

같은 원리로:

```
codex-worker
= Execution Worker 역할 구현체
```

다.

---

### `index.ts`

```
index.ts
= 해당 use-case의 public boundary
```

외부에서는 내부 디렉터리 구조보다 `index.ts`를 통해 필요한 API를 사용하도록 한다.

예:

```
planTask
executeTask
PlanningResult
ExecutionResult
```

내부 파일 위치가 바뀌어도 외부 사용자가 영향을 덜 받게 하는 경계다.

---

# 4. `flow/`의 의미

`flow`를:

```
오케스트레이션만 넣는 폴더
```

라고 정의하지 않는다.

우리 프로젝트에서 `flow`는:

> **해당 use-case가 실제로 진행되는 과정의 application logic**

이다.

즉 Planning이라면:

```
Goal
→ Repository 조사
→ Planner 판단
→ 추가 context 확보
→ TaskDraft
→ TaskContract
```

Execution이라면:

```
TaskContract
→ workspace 생성
→ worker 실행
→ scope 검사
→ verification
→ artifact 저장
→ ExecutionResult
```

이 과정에 직접 속하는 application logic이 `flow`다.

---

## `flow` root

root에는 가능한 한 use-case entry를 둔다.

예:

```
planning/flow/plan-task.ts
execution/flow/execute-task.ts
```

의미:

```
planTask()
= Planning 시작점

executeTask()
= Execution 시작점
```

---

# 5. `flow/orchestration`

workflow를 **어떤 순서로 진행할지 제어**하는 코드.

예:

```
planning-loop.ts
stages.ts
result-finalizer.ts
```

역할:

```
다음 단계 결정
반복 제어
stage 진행
실패 누적
workflow 종료 처리
```

즉 orchestration은:

> **일 자체보다 일의 순서와 진행 상태를 관리한다.**

---

# 6. `flow` 내부 세부 폴더

`flow` 안에서도 파일이 많아지면 세부 책임별로 나눌 수 있다.

예:

```
planning/flow/
├─ plan-task.ts
├─ orchestration/
├─ inspection/
└─ transformation/
```

### `inspection/`

```
inspection
= workflow 진행에 필요한 사실을 수집
```

예:

```
inspectRepository
inspectFiles
```

Planner가 판단할 수 있도록 repository context를 확보하는 책임이다.

---

### `transformation/`

```
transformation
= workflow를 제어하지 않고 데이터 형태를 변환
```

예:

```
PlanningState
→ PlannerContext
```

```
TaskDraft + GoalSpec
→ TaskContract
```

즉:

```
createPlannerContext
buildTaskContract
```

같은 함수가 들어간다.

---

## 세부 폴더를 use-case root로 올리지 않는 이유

다음처럼 만들지 않는다.

```
planning/
├─ flow/
├─ model/
├─ protocol/
├─ planner/
├─ inspection/
└─ transformation/
```

이유는 분류 기준이 섞이기 때문이다.

```
flow / model / protocol
= architectural responsibility

inspection / transformation
= flow 내부 작업 종류
```

따라서:

```
flow/
├─ inspection/
└─ transformation/
```

처럼 두는 것이 더 일관적이다.

---

# 7. `model/`

`model`은:

> **해당 application/use-case가 소유하는 상태, 설정, 결과**

를 둔다.

Planning:

```
PlanningState
PlanningLimits
PlanningResult
RepositoryInfo
```

Execution:

```
ExecutionState
ExecutionResult
```

중요:

```
model
≠ 모든 데이터 타입
```

타입의 형태가 아니라 **누가 그 개념을 소유하는가**를 본다.

---

# 8. `protocol/`

`protocol`은:

> **역할이나 컴포넌트 사이에서 주고받는 구조화된 계약**

이다.

예:

```
PlannerContext
PlannerDecision
InspectRequest
InspectResult
TaskDraft
```

`PlanningResult` 같은 최종 application 결과와는 성격이 다르다.

```
PlanningResult
= Planning use-case의 최종 결과

PlannerDecision
= Planner ↔ Harness 사이의 통신 계약
```

---

# 9. 역할 폴더: `planner/`, `worker/`

특정 use-case 안에서 명확한 역할이 존재하면 별도 폴더를 둔다.

```
planning/
└─ planner/

execution/
└─ worker/
```

역할 기준으로 배치하고 사용 기술 기준으로 배치하지 않는다.

---

# 10. Codex 관련 기준

앞으로 Codex 관련 코드는 다음 기준으로 판단한다.

```
Planner ≠ Codex
Worker ≠ Codex
```

정확히는:

```
Planner can use Codex
Worker can use Codex
```

따라서:

```
application/planning/planner/codex-planner.ts
application/execution/worker/codex-worker.ts
```

처럼 역할 기준으로 분리한다.

---

## Generic Codex 기능이 생기면

나중에 Planner와 Worker가 공통으로:

```
codex exec
sandbox
timeout
stdout/stderr
process failure
config
```

같은 기술 메커니즘을 사용하게 되고 공통 책임이 실제로 생긴다면:

```
infrastructure/
└─ codex/
   └─ run-codex.ts
```

같은 형태로 추출할 수 있다.

판단 기준:

```
Planning 개념을 앎
→ application/planning

Execution 개념을 앎
→ application/execution

Codex 실행 방법만 앎
→ infrastructure
```

---

# 11. `services/`로 빼는 기준

단순히:

```
실제로 무언가를 한다
```

고 service가 아니다.

service로 빼려면:

> **해당 use-case를 떠나도 독립 capability로 의미가 있는가?**

를 본다.

예:

```
scope 검사
verification 실행
artifact 저장
evidence 수집
```

은 독립 capability로 보기 쉽다.

반대로:

```
PlanningState → PlannerContext
TaskDraft → TaskContract
```

는 Planning 전용 개념을 강하게 안다.

따라서 Planning 내부에 둔다.

---

# 12. `transformation`과 `orchestration` 차이

### Orchestration

```
무엇을 다음에 할지 결정
반복 여부 판단
stage 순서 제어
실패 처리
workflow 종료 제어
```

### Transformation

```
입력 데이터
↓
규칙 적용
↓
다른 데이터 형태
```

예:

```
createPlannerContext
= PlanningState → PlannerContext

buildTaskContract
= TaskDraft → TaskContract
```

이 둘은 workflow 순서를 결정하지 않는다.

따라서 orchestration과 구분한다.

---

# 13. `finalizer` 기준

finalizer는 workflow가 끝난다는 이유만으로 만들지 않는다.

기준:

> **종료 처리 자체가 독립 책임일 정도로 복잡한가?**

Execution은:

```
state 조립
passed 계산
evidence / scope / failures 반영
artifacts 반영
result 저장
report failure 처리
최종 ExecutionResult 반환
```

등이 있어서:

```
finalizeExecutionResult()
```

가 의미 있다.

Planning은 현재:

```
TaskContract 생성
PlanningResult 반환
```

정도로 종료 처리가 단순하다.

그래서 별도:

```
finalizePlanningResult()
```

를 만들 필요가 없다.

중요:

```
구조적 대칭성을 위해 abstraction을 만들지 않는다.
실제 책임이 생겼을 때 만든다.
```

---

# 14. Planning과 Execution 구조가 달라도 되는 이유

Planning과 Execution은 같은 application layer에 있지만 내부 복잡도는 다르다.

예:

```
execution/
├─ flow/
├─ model/
└─ worker/
```

Planning:

```
planning/
├─ flow/
│  ├─ orchestration/
│  ├─ inspection/
│  └─ transformation/
├─ model/
├─ protocol/
└─ planner/
```

Planning의 `flow`가 더 커 보여도 문제가 아니다.

Execution의 일부 작업은 이미:

```
services/
├─ artifacts/
├─ evidence/
├─ scope/
└─ verification/
```

처럼 독립 capability로 성장했다.

반면 Planning의:

```
inspection
transformation
```

은 아직 Planning flow와 강하게 결합되어 있다.

따라서 내부에 남아 있는 것이다.

맞춰야 하는 것은:

```
파일 개수
폴더 개수
모양
```

이 아니라:

> **책임을 판단하는 기준**

이다.

---

# 15. 구조적 대칭성보다 책임을 우선한다

다음과 같이 생각하지 않는다.

```
Execution에 finalizer가 있으니까
Planning에도 finalizer를 만든다.

Execution에 worker가 있으니까
Planning에도 worker를 만든다.

Planning flow에 폴더가 3개니까
Execution flow에도 3개를 만든다.
```

대신:

```
이 책임이 실제로 존재하는가?
```

를 먼저 본다.

> **모양은 달라도 판단 기준은 같아야 한다.**

---

# 16. 새 파일 위치를 정할 때 체크리스트

앞으로 새 파일이 생기면 이 순서로 판단한다.

```
1. 어떤 use-case에 속하는가?
   → planning / execution / system-wide

2. 해당 use-case가 진행되는 과정에 직접 속하는가?
   → flow

3. 흐름의 순서와 상태를 제어하는가?
   → flow/orchestration

4. flow 안의 특정 작업 단계인가?
   → flow/<responsibility>

5. use-case 전체가 소유하는 상태/설정/결과인가?
   → model

6. 역할 또는 컴포넌트 사이 메시지 계약인가?
   → protocol

7. 명확한 역할 구현체인가?
   → planner / worker

8. 다른 use-case에서도 독립 capability로 의미가 있는가?
   → services

9. Git / Process / Codex CLI 같은 기술적 실행 방법인가?
   → infrastructure

10. 여러 곳에서 쓰일 것 같다는 예상만 있는가?
   → 아직 추출하지 않는다
```

마지막이 중요하다.

> **미래의 재사용 가능성만 보고 abstraction을 만들지 않는다. 실제 공통 책임이 생겼을 때 추출한다.**

---

# 17. 현재 프로젝트 기준 구조

```
src/
├─ application/
│  ├─ planning/
│  │  ├─ flow/
│  │  │  ├─ plan-task.ts
│  │  │  ├─ orchestration/
│  │  │  ├─ inspection/
│  │  │  └─ transformation/
│  │  ├─ model/
│  │  ├─ protocol/
│  │  ├─ planner/
│  │  └─ index.ts
│  │
│  └─ execution/
│     ├─ flow/
│     │  ├─ execute-task.ts
│     │  └─ orchestration/
│     ├─ model/
│     ├─ worker/
│     └─ index.ts
│
├─ contracts/
│
├─ services/
│  ├─ artifacts/
│  ├─ evidence/
│  ├─ scope/
│  └─ verification/
│
└─ infrastructure/
   ├─ git/
   └─ process/
```

---

# 최종 원칙

```
Architecture is responsibility-first.
```

폴더는:

```
사용 기술로 나누지 않는다.
파일 수를 맞추기 위해 나누지 않는다.
구조를 대칭으로 만들기 위해 나누지 않는다.
미래의 재사용 가능성만 보고 추출하지 않는다.
```

대신:

```
누가 이 책임을 소유하는가?
어떤 use-case에 속하는가?
독립 capability인가?
기술적 구현 세부사항인가?
```

를 기준으로 나눈다.

최종적으로 기억할 기준:

```
application
= use-case와 workflow

flow
= 해당 use-case가 실제로 진행되는 과정의 application logic

model
= use-case가 소유하는 상태 / 설정 / 결과

protocol
= 역할 사이의 구조화된 계약

planner / worker
= 명확한 역할 구현체

services
= use-case와 분리 가능한 독립 capability

infrastructure
= Git / Process / Codex CLI 같은 기술적 실행 수단

contracts
= system-wide contract
```

이 기준을 앞으로 프로젝트 구조 판단의 기본 규칙으로 사용한다.
