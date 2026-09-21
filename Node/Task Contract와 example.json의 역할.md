# Task Contract와 `example.json`의 역할

## 1. 먼저 현재 위치부터 이해하기

AI Workspace의 최종 목표는 대략 다음과 같은 흐름이다.

```text
Human
  ↓
Goal
  ↓
CEO / Control Plane
  ↓
Task Contract 생성
  ↓
Repository Worker
  ↓
Repository Harness
  ↓
Repository 작업
  ↓
Verification
  ↓
Evidence
```

하지만 **현재 여기까지 구현된 것은 아니다.**

현재 실제 코드는 다음까지만 존재한다.

```text
Task Contract JSON
        ↓
파일 읽기
        ↓
JSON.parse()
        ↓
Zod Validation
        ↓
유효한 TaskContract인지 판단
        ↓
CLI 결과 출력
```

즉 지금은 AI Worker를 실행하는 단계가 아니라,

> **앞으로 Worker에게 전달할 작업 계약서를 어떤 구조로 만들고, 그 계약서가 올바른지 프로그램이 검증할 수 있게 만드는 단계**

다.

---

# 2. `example.json`은 무엇인가?

`tasks/example.json`은 현재 **Task Contract의 예시 파일**이다.

예를 들면 다음과 같은 정보를 담는다.

```json
{
  "id": "task-example-001",
  "goalId": "goal-example-001",

  "objective": "Load and validate a TaskContract from JSON",

  "targetRepository": "ai-workspace",

  "allowedPaths": ["src/task-loader.ts"],

  "forbiddenPaths": ["docs/architecture.md"],

  "constraints": ["Do not access the target Repository"],

  "acceptanceCriteria": ["The TaskContract loads and validates successfully"],

  "verification": ["pnpm typecheck", "pnpm test"]
}
```

쉽게 말하면:

> **“이번 작업에서 무엇을 해야 하고, 어디까지 허용하며, 무엇을 하면 안 되고, 언제 성공으로 판단할 것인지”를 구조화한 작업 계약서다.**

---

# 3. 각 필드는 무엇을 의미하는가?

### `objective`

무엇을 해야 하는가.

```text
로그인 버그를 수정한다.
```

---

### `targetRepository`

어느 Repository를 대상으로 하는가.

```text
hyoit-FE
```

---

### `allowedPaths`

작업하면서 접근하거나 수정할 수 있는 범위.

```text
src/auth/**
```

---

### `forbiddenPaths`

작업 중 건드리면 안 되는 범위.

```text
.env
package.json
```

---

### `constraints`

작업하면서 지켜야 할 추가 제약.

```text
새 dependency를 추가하지 않는다.
```

---

### `acceptanceCriteria`

무엇이 만족되어야 작업을 성공했다고 할 수 있는가.

```text
로그인 성공 후 홈 화면으로 이동한다.
```

---

### `verification`

성공 여부를 어떤 방법으로 확인할 것인가.

```text
pnpm typecheck
pnpm test
```

즉 Task Contract는 크게 다음을 정의한다.

```text
무엇을 할 것인가
+
어디까지 할 수 있는가
+
무엇을 하면 안 되는가
+
어떤 상태가 성공인가
+
성공을 어떻게 증명할 것인가
```

---

# 4. 왜 Repository만 넘기지 않고 Task Contract가 필요한가?

Repository만 AI에게 넘기면 다음 정보가 없다.

```text
무슨 작업을 해야 하지?

어느 파일까지 봐도 되지?

어느 파일을 수정하면 안 되지?

리팩토링까지 해도 되나?

언제 작업이 끝난 거지?

무슨 테스트를 돌려야 하지?
```

예를 들어 AI에게 단순히:

```text
hyoit-FE 좀 고쳐줘.
```

라고 하는 것보다,

```text
Objective
→ 로그인 redirect 오류 수정

Allowed Scope
→ src/auth/**

Forbidden Scope
→ package.json

Constraint
→ 새로운 dependency 추가 금지

Acceptance Criteria
→ 로그인 성공 후 Home으로 이동

Verification
→ pnpm typecheck
→ pnpm test
```

처럼 범위를 명확히 하는 것이 훨씬 안전하다.

그래서 Task Contract는:

> **Control Plane과 실제 Repository 작업 사이에 있는 공식적인 작업 경계**

라고 볼 수 있다.

---

# 5. 지금 `example.json`을 AI Worker가 직접 사용하는가?

**아직 아니다.**

이 부분을 정확하게 구분해야 한다.

현재 구현:

```text
example.json
    ↓
task-loader.ts
    ↓
Zod
    ↓
유효한 TaskContract인지 검사
```

아직 없는 것:

```text
TaskContract
    ↓
AI Worker 실행
    ↓
Repository 접근
    ↓
파일 수정
    ↓
테스트 실행
```

즉 현재 `example.json`은:

> **앞으로 시스템에서 사용할 Task Contract가 실제 파일로 어떻게 생기는지를 보여주고, 그것을 프로그램이 읽고 검증할 수 있는지 테스트하기 위한 샘플**

이다.

---

# 6. 지금 왜 파일 형태로 만들었는가?

최종 시스템에서도 Task Contract가 반드시 `.json` 파일로 영구 저장되어야 하는 것은 아니다.

현재 파일로 만든 가장 큰 이유는:

```text
TaskContract라는 데이터 구조를

눈으로 확인하고
↓
직접 수정해보고
↓
CLI에 넣어보고
↓
Validation 테스트하기 쉬우니까
```

다.

즉 지금은 학습과 초기 구현을 위해:

```text
tasks/example.json
```

이라는 명시적인 파일을 사용하고 있다.

---

# 7. 나중에도 Task마다 JSON 파일이 계속 생기는가?

반드시 그런 것은 아니다.

최종 구현 방법은 아직 결정하지 않아도 된다.

가능한 방식은 여러 가지다.

### 방법 A — 파일로 저장

```text
tasks/
├─ task-001.json
├─ task-002.json
└─ task-003.json
```

장점:

```text
작업 기록 보존
Audit 가능
재실행 가능
디버깅 쉬움
```

---

### 방법 B — 실행할 때만 임시 파일 생성

```text
Human 요청
↓
Task Contract 생성
↓
.tmp/task-123.json
↓
작업 실행
↓
삭제
```

---

### 방법 C — 파일 자체를 만들지 않음

```text
Human
 ↓
CEO
 ↓
TaskContract 객체
 ↓
Zod Validation
 ↓
Worker
```

즉:

```ts
const task = {
  objective: "...",
  allowedPaths: [...],
};
```

같은 객체를 메모리에서 바로 넘길 수도 있다.

따라서:

> **Task Contract라는 개념과 JSON 파일은 같은 것이 아니다.**

Task Contract는 **데이터 계약**이고,

JSON 파일은 그 계약을 표현하는 **현재의 한 가지 저장 방식**이다.

이 구분이 중요하다.

---

# 8. `"뭐뭐 고쳐줘"` 하면 나중에는 어떻게 될까?

최종적으로 생각하고 있는 흐름은 이런 형태에 가깝다.

```text
Human

"효잇 로그인 redirect 버그 고쳐줘."

        ↓

CEO / Control Plane

요청 분석

        ↓

Task Contract 생성

{
  objective: "...",
  targetRepository: "hyoit-FE",
  allowedPaths: [...],
  forbiddenPaths: [...],
  acceptanceCriteria: [...],
  verification: [...]
}

        ↓

Validation

        ↓

Worker

        ↓

Repository Harness

        ↓

hyoit-FE 작업

        ↓

Verification

        ↓

Evidence

        ↓

CEO

        ↓

Human에게 결과 보고
```

따라서 최종적으로는 사람이 매번:

```text
tasks/my-task.json
```

을 직접 작성할 필요가 없을 수도 있다.

Control Plane이 Human의 요청을 분석해서 Task Contract를 만들 수 있기 때문이다.

---

# 9. 그렇다면 `ai-workspace run ./tasks/my-task.json`은 언제 사용하는가?

이 명령은 개념적으로:

> **“이 Task Contract를 AI Workspace에 입력해서 실행 흐름을 시작해라.”**

라는 의미다.

하지만 주의해야 한다.

현재 Repository에 실제로 구현된 명령은:

```bash
pnpm dev ./tasks/example.json
```

이다.

`ai-workspace run ...`이라는 완성된 CLI 인터페이스가 현재 존재하는 것은 아니다.

현재는:

```text
pnpm dev ./tasks/example.json
          ↓
      CLI 실행
          ↓
  example.json 읽기
          ↓
      Zod 검증
```

까지만 수행한다.

나중에 CLI가 발전하면 개념적으로:

```bash
ai-workspace run task.json
```

같은 형태가 될 수 있다는 이야기다.

---

# 10. 이 명령어를 Codex에게 입력하는 것인가?

**아니다.**

여기서 가장 많이 헷갈릴 수 있다.

```bash
ai-workspace run ...
```

같은 명령은 Codex에게 보내는 프롬프트가 아니다.

이것은:

```text
Codex CLI 명령어
```

가 아니라

```text
내가 만드는 ai-workspace 프로그램의 CLI
```

다.

즉 개념적으로:

```text
Terminal

$ ai-workspace run task.json

        ↓

내 프로그램 실행

        ↓

Task Contract 읽기

        ↓

Harness / Worker 실행

        ↓

필요하다면 내부에서 Codex 같은 Agent Runtime 사용
```

이라는 구조다.

Codex는 **ai-workspace 안에서 Worker 역할을 수행하기 위해 연결될 수 있는 실행 도구 중 하나**가 될 수 있다.

---

# 11. Zod Validation은 왜 필요한가?

Task Contract가 이렇게 잘못 생성될 수도 있다.

```json
{
  "objective": null,
  "targetRepository": 123,
  "allowedPaths": "src/auth"
}
```

JSON 문법 자체는 정상일 수 있다.

하지만 Task Contract로서는 잘못된 데이터다.

그래서:

```text
외부에서 들어온 Task

        ↓

unknown

        ↓

Zod Validation

        ↓

정상 TaskContract
```

과정을 거친다.

즉 Zod는:

> **잘못 만들어진 계약서가 실행 단계까지 넘어가지 못하도록 막는 첫 번째 검증 경계**

다.

---

# 12. 하지만 Zod가 작업 범위를 실제로 막아주는 것은 아니다

이것도 중요하다.

Task Contract에:

```json
{
  "allowedPaths": ["src/auth"],
  "forbiddenPaths": ["package.json"]
}
```

이라고 써 있다고 해서 현재 시스템이 실제로:

```text
package.json 접근 차단
```

을 해주는 것은 아니다.

현재는 단지:

```text
allowedPaths가 string 배열인가?
forbiddenPaths가 string 배열인가?
```

를 검사한다.

즉 현재:

```text
Policy Definition
✅
```

은 존재하지만,

```text
Policy Enforcement
❌
```

는 아직 없다.

---

# 13. 이걸 실제로 강제하는 것이 Repository Harness다

앞으로 Repository Harness가 만들어지면:

```text
Worker

"package.json 읽어줘"

       ↓

Repository Harness

TaskContract 확인

allowedPaths?
forbiddenPaths?

       ↓

❌ 접근 거부
```

같은 일을 담당하게 된다.

따라서:

```text
TaskContract
= 작업 규칙을 정의

Repository Harness
= 그 규칙을 실제로 집행

Worker
= 그 규칙 안에서 실제 작업 수행
```

이라고 보면 된다.

---

# 14. Architecture와 어떻게 연결되는가?

Architecture에서 말하는:

> Repository Worker는 Task Contract 범위를 임의로 확대하지 않는다.

라는 원칙을 앞으로 코드로 구현하기 위한 출발점이 지금 TaskContract다.

최종적으로는:

```text
Human / CEO
      ↓
Task Contract

"이 일만 해."

      ↓
Worker

"알겠습니다."

      ↓
Repository Harness

"진짜 그 범위 밖으로 못 나가게 내가 통제한다."
```

가 된다.

중요한 점은 Worker에게 단순히:

```text
범위 밖으로 나가지 마.
```

라고 프롬프트로 부탁하는 것이 아니라,

**Harness가 실제 실행 권한을 통제하는 구조**를 만드는 것이다.

---

# 15. 현재 우리가 만든 것은 정확히 무엇인가?

현재까지 만든 것은:

```text
AI Worker ❌
Repository Harness ❌
CEO ❌
Multi-Agent ❌

TaskContract Schema ✅
JSON Loader ✅
Zod Validation ✅
CLI Entry Point ✅
관련 테스트 ✅
```

이다.

따라서 현재 단계를 한 문장으로 표현하면:

> **AI에게 일을 시키기 전에 사용할 표준 작업 계약(TaskContract)을 정의하고, 외부에서 들어온 계약서가 올바른 형식인지 프로그램이 안전하게 읽고 검증할 수 있는 첫 번째 기반을 만든 단계다.**

---

# 최종 흐름 정리

## 현재

```text
example.json
    ↓
Task Loader
    ↓
JSON.parse
    ↓
Zod
    ↓
Valid TaskContract
```

## 다음 단계

```text
TaskContract
    ↓
Repository 연결
    ↓
Scope 확인
    ↓
Read-only Repository Harness
```

## 더 이후

```text
Human Goal
    ↓
CEO
    ↓
TaskContract
    ↓
Worker
    ↓
Harness
    ↓
Repository
    ↓
Verification
    ↓
Evidence
    ↓
CEO Report
    ↓
Human
```

## 핵심 한 문장

> `example.json`은 지금 당장 AI에게 코딩을 시키는 파일이 아니라, 앞으로 AI Worker에게 전달할 **“제한된 작업 계약(Task Contract)”의 형태를 먼저 정의하고 검증하기 위한 샘플 입력 파일**이다. 현재는 계약서를 읽고 검증하는 단계이며, 이후 Repository Harness와 Worker가 붙으면서 실제 실행 계약으로 사용된다.
