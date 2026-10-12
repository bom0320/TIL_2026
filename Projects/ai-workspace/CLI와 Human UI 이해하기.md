# CLI와 Human UI 이해하기

## 1. CLI란?

CLI는 **Command Line Interface**의 약자로, 터미널에서 프로그램을 실행하고 입력을 주고받는 방식이다.

내 프로젝트에서는:

```
pnpm dev
```

를 실행하면 `package.json`의 script를 통해 `cli.ts`가 실행된다.

예:

```json
{
  "scripts": {
    "dev": "tsx src/cli/cli.ts"
  }
}
```

실행 흐름:

```
pnpm dev
  ↓
src/cli/cli.ts 실행
  ↓
AI Workspace 시작
```

즉:

> **`cli.ts` = 터미널 기반 AI Workspace의 진입점(entry point)**

---

## 2. Human UI란?

Human UI는 **사람이 시스템의 상태를 보고, 입력하거나 판단하는 접점**이다.

꼭 웹페이지일 필요는 없다.

### 터미널 기반 Human UI

예:

```
Planning completed

Objective:
README 설명을 더 명확하게 수정

Allowed Paths:
- README.md

Verification:
- git diff --check

Execute this task?

> Approve
  Reject
```

이처럼 터미널에서 사람이 결과를 확인하고 선택하면:

> Terminal-based Human UI

라고 볼 수 있다.

### 웹 기반 Human UI

나중에 같은 기능을 웹으로 만들면:

```
Task Proposal

Objective
README 설명을 더 명확하게 수정

[ Approve ] [ Reject ]
```

처럼 만들 수도 있다.

핵심은:

> Human UI = 사람이 보고, 입력하고, 승인하거나 판단하는 인터페이스

이다.

---

# 3. `cli.ts`와 Human UI의 관계

둘은 같은 개념은 아니다.

```
cli.ts
= CLI 프로그램 전체의 진입점

Human UI
= CLI 실행 중 사람이 상호작용하는 부분
```

예:

```
cli.ts
  ↓
프로그램 시작

Goal 입력
  ↓
Human UI

Planning
  ↓
Application Logic

TaskContract 확인
  ↓
Human UI

Approve / Reject
  ↓
Human UI

Execution
  ↓
Application Logic
```

즉:

> Human UI는 CLI 안에 포함될 수 있다.

---

# 4. CLI에서 Human UI는 어떻게 구현하는가?

가장 단순하게는 Node의 표준 입력인 `stdin`을 사용하면 된다.

예:

```tsx
import { createInterface } from "node:readline/promises";
import { stdin as input, stdout as output } from "node:process";

const rl = createInterface({
  input,
  output,
});

const answer = await rl.question("Execute this task? (y/n): ");

if (answer === "y") {
  // Execution 시작
}
```

실제로 터미널에는:

```
Execute this task? (y/n):
```

가 나타나고 사용자가 값을 입력할 수 있다.

조금 더 발전시키면:

```
? Execute this task?

❯ Approve
  Reject
```

처럼 방향키로 선택하는 interactive CLI도 만들 수 있다.

하지만 현재 단계에서는 별도의 UI 라이브러리를 추가하지 않고 stdin 기반으로 시작해도 충분하다.

---

# 5. `pnpm dev`는 무슨 역할인가?

내 프로젝트에서는:

```
pnpm dev
```

를 단순히 "개발 서버 실행"이라고 보면 안 된다.

`dev`는 `package.json`에서 임의로 붙인 script 이름이다.

예를 들어:

```tsx
{
  "scripts": {
    "dev": "tsx src/cli/cli.ts"
  }
}
```

라면:

```
pnpm dev
```

의 의미는:

> AI Workspace CLI 프로그램을 실행한다.

이다.

그래서 최종적으로는 이런 사용 흐름도 가능하다.

```
$ pnpm dev

AI Workspace

Repository:
> .

What do you want to accomplish?
> README 프로젝트 설명을 더 명확하게 수정해줘

Planning...

Task Proposal

Objective:
README 프로젝트 설명을 더 명확하게 수정

Allowed Paths:
- README.md

Verification:
- git diff --check

Execute this task?

> Approve
  Reject
```

즉:

```
pnpm dev
= 프로그램 실행

cli.ts
= 프로그램 진입점

Human UI
= 실행 중 사람이 입력/판단하는 부분
```

이다.

---

# 6. 실제 기능은 어디에 있는가?

CLI가 실제 기능 자체는 아니다.

실제 기능은 Application Layer에 있다.

예:

```
application/
├─ planning/
│  └─ planTask()
│
└─ execution/
   └─ executeTask()
```

각 역할은:

```
planTask()
= Goal을 분석해 TaskContract를 만드는 기능

executeTask()
= TaskContract를 Repository에서 실행하는 기능
```

이다.

CLI는 이 기능을 호출한다.

```
CLI
 ↓
planTask()
 ↓
executeTask()
```

따라서:

> CLI는 입구이고, Application은 실제 일을 수행하는 곳이다.

---

# 7. 왜 `approve-task.ts` 같은 파일을 따로 두는가?

기술적으로는 모든 코드를 `cli.ts`에 넣어도 작동한다.

예:

```tsx
const goal = await askGoal();

const planning = await planTask(...);

console.log(planning.task);

const approved = await askApproval();

if (approved) {
  await executeTask(...);
}
```

하지만 기능이 늘어나면 `cli.ts`가 너무 커진다.

그래서 역할을 분리한다.

예:

```
cli/
├─ cli.ts
│
├─ prompts/
│  ├─ ask-goal.ts
│  └─ approve-task.ts
│
├─ presenters/
│  ├─ planning-result.ts
│  └─ execution-result.ts
│
└─ commands/
   ├─ plan.ts
   └─ execute.ts
```

각 역할은:

```
prompts/
= 사람에게 입력받기

presenters/
= 결과를 사람이 읽을 수 있게 출력

commands/
= Application 기능 호출

cli.ts
= 전체 흐름 연결
```

이다.

즉 `approve-task.ts`가 반드시 필요한 것은 아니다.

> 코드의 책임을 분리하기 위해 따로 만드는 것

이다.

---

# 8. 현재 AI Workspace에서의 전체 흐름

v1 기준으로 원하는 흐름은:

```
pnpm dev
  ↓
cli.ts
  ↓
Human UI
Goal 입력
  ↓
planTask()
  ↓
Repository 조사
  ↓
TaskContract 생성
  ↓
Human UI
TaskContract 확인
  ↓
Approve / Reject
  ↓
Approve
  ↓
executeTask()
  ↓
Verification
  ↓
Execution Result
```

즉 v0에서:

```
TaskContract 직접 작성
 ↓
Execution
```

이었다면,

v1에서는:

```
Goal
 ↓
Planning
 ↓
TaskContract
 ↓
Human Approval
 ↓
Execution
```

로 발전한다.

---

# 9. 왜 지금 터미널 기반 Human UI가 적절한가?

AI Workspace의 현재 목적은 UI 제품을 만드는 것이 아니다.

검증하려는 것은:

```
Agent가 올바르게 판단하는가?

Repository를 필요한 만큼 조사하는가?

TaskContract를 제대로 생성하는가?

Human의 제약을 지키는가?

허용된 범위만 수정하는가?

Verification을 통과하는가?
```

이다.

따라서 v0~v5에서는 웹 UI보다 터미널이 오히려 적합하다.

터미널에서는:

```
Agent 판단
TaskContract
Conflict
Agreement
CEO Decision
Evidence
Verification
Telemetry
```

등을 직접 확인하기 쉽다.

---

# 10. 버전별 CLI / Human UI 발전 방향

## v0 — Single Worker

목표:

> Agent 하나가 Repository에서 안전하게 작업 가능

CLI:

```
TaskContract
 ↓
Worker
 ↓
Verification
 ↓
Result
```

출력 예:

```
Workspace created
Worker complete
Changed Paths: 1
Verification passed

Result: SUCCESS
```

이 단계는 일반적인 로그 출력만으로 충분하다.

---

## v1 — Goal → TaskContract

목표:

> Human의 Goal을 Repository 기반 TaskContract로 변환

여기서 처음 interactive CLI를 도입한다.

```
Goal 입력
 ↓
Planning
 ↓
TaskContract 표시
 ↓
Approve / Reject
 ↓
Execution
```

필요한 Human UI:

```
1. Goal 입력
2. TaskContract 확인
3. Approve / Reject
```

현재 구현해야 할 범위가 여기다.

---

## v2 — 두 전문 Agent의 독립 판단

목표:

> 서로 다른 역할을 가진 Agent가 독립적으로 판단

터미널 출력:

```
[Frontend Agent]

Recommendation:
...

Reason:
...

[Product Agent]

Recommendation:
...

Reason:
...
```

여기서는 별도의 새로운 Human UI보다:

> Agent별 결과를 명확하게 구분해서 출력

하는 것이 중요하다.

---

## v3 — Conflict / Agreement 구조화

목표:

> 두 Agent의 의견 차이를 구조화

CLI:

```
Agent Comparison

Agreement
---------
- README 수정 필요
- 프로젝트 목적 명확화 필요

Conflict
--------
Frontend Agent:
기술 설명 강화

Product Agent:
사용자 가치 설명 강화

Uncertainty
-----------
docs 수정 필요 여부
```

여기서는 Human UI보다 `presenter`가 중요해진다.

---

## v4 — CEO Agent

목표:

> CEO가 여러 Agent의 판단을 종합하고 최종 의사결정을 설명

CLI:

```
CEO Decision

Decision:
README.md만 수정

Reason:
1. 두 Agent 모두 README 수정에는 동의
2. docs 수정 필요성에는 합의하지 않음
3. 현재 Goal 범위에서는 README 수정으로 충분

Final TaskContract:
...
```

그리고:

```
Approve CEO Decision?

> Approve
  Reject
```

v1에서 만든 Human Approval 구조를 그대로 재사용할 수 있다.

---

## v5 — Single Agent vs Multi-Agent 비교

목표:

> Multi-Agent 구조가 실제로 더 나은 결과를 만드는지 비교

CLI:

```
Experiment Result

Single Agent
------------
Success: yes
Changed Files: 3
Verification: passed
Time: 18.4s
Cost: ...

Multi-Agent
-----------
Success: yes
Changed Files: 2
Verification: passed
Time: 31.8s
Cost: ...

Comparison
----------
Scope Precision: Multi-Agent
Cost Efficiency: Single Agent
Conflict Detection: Multi-Agent only
```

이 단계에서는 CLI가 실험 결과 확인 도구가 된다.

---

# 11. 버전별 CLI 구현 시점

```
v0
기본 로그 출력
        ↓
v1
Interactive CLI 도입
Goal / Approve / Reject
        ↓
v2
Agent별 결과 Presenter
        ↓
v3
Conflict / Agreement Presenter
        ↓
v4
CEO Decision + Human Approval 재사용
        ↓
v5
Experiment / Metrics Presenter
```

즉:

> CLI 전체를 지금 한꺼번에 완성하는 것이 아니라 버전이 발전하면서 필요한 Human UI와 Presenter를 하나씩 추가한다.

---

# 12. 현재 시점에서 필요한 CLI 구조

지금 v1에서는 이 정도면 충분하다.

```
src/
├─ cli/
│  ├─ cli.ts
│  │
│  ├─ commands/
│  │  ├─ plan.ts
│  │  └─ execute.ts
│  │
│  ├─ prompts/
│  │  ├─ ask-goal.ts
│  │  └─ approve-task.ts
│  │
│  └─ presenters/
│     └─ task-contract.ts
│
├─ application/
│  ├─ planning/
│  └─ execution/
│
├─ contracts/
├─ infrastructure/
└─ services/
```

v2~v5에서 필요할 때:

```
presenters/
├─ agent-opinion.ts
├─ agent-comparison.ts
├─ ceo-decision.ts
└─ experiment-result.ts
```

를 추가하면 된다.

---

# 핵심 정리

```
pnpm dev
= AI Workspace 프로그램 실행

cli.ts
= 프로그램의 터미널 진입점

CLI
= 터미널을 통해 AI Workspace를 사용하는 인터페이스

Human UI
= CLI 안에서 사람이 입력하고 판단하는 부분

prompt
= Human에게 입력/선택을 받는 부분

presenter
= 시스템 결과를 Human에게 보여주는 부분

Application
= 실제 Planning / Execution 기능
```

그리고 현재 v1의 목표 흐름은:

```
Human
 ↓
Goal
 ↓
CLI
 ↓
Planning
 ↓
TaskContract
 ↓
Human Approval
 ↓
Execution
 ↓
Verification
 ↓
Result
```

이다.

**지금은 v1에서 `Goal 입력 → TaskContract 표시 → Approve/Reject → Execution`까지 터미널 기반으로 연결하면 된다.** 이후 v2~v5가 진행될수록 같은 CLI 위에 Agent Opinion, Conflict/Agreement, CEO Decision, Experiment Result를 순서대로 추가하면 된다.
