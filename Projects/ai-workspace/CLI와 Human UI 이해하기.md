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

---현재 브랜치의 v1 Goal-to-Task Planning 구현을 기준으로
CLI Human Interface 구조를 아래 형태까지 정리해줘.

목표 구조:

src/
├─ cli/
│ ├─ cli.ts
│ │
│ ├─ commands/
│ │ ├─ plan.ts
│ │ └─ execute.ts
│ │
│ ├─ prompts/
│ │ ├─ ask-goal.ts
│ │ └─ approve-task.ts
│ │
│ └─ presenters/
│ └─ task-contract.ts
│
├─ application/
│ ├─ planning/
│ └─ execution/
│
├─ contracts/
├─ infrastructure/
└─ services/

이번 작업의 목적은 새로운 기능을 크게 추가하는 것이 아니라,
현재 CLI 책임을 더 명확하게 분리하고 v1의 Human Interface를 최소한으로 완성하는 것이다.

중요:

- application/planning, application/execution 내부 로직은 불필요하게 수정하지 말 것
- planTask(), executeTask() 등 기존 public API는 유지할 것
- 새로운 workflow/orchestrator/controller 계층을 추가하지 말 것
- 새로운 UI 라이브러리나 dependency를 추가하지 말 것
- fancy TUI는 만들지 말 것
- CLI는 application의 adapter 역할만 유지할 것

## 1. prompts/ask-goal.ts 추가

현재 repositoryPath와 objective를 CLI argument로만 받는 흐름을
interactive CLI에서도 입력받을 수 있도록 만들어줘.

역할:

- Repository path 입력
- Goal / Objective 입력
- 입력값 반환

예상 UX:

AI Workspace

Repository:

> .

Goal:

> README 프로젝트 설명을 더 명확하게 수정해줘

Node의 readline/promises 등 built-in API를 우선 사용해줘.

반환 타입은 명확하게 정의해줘.

예:

type GoalInput = {
repositoryPath: string;
objective: string;
};

입력값은 trim 처리하고,
빈 값은 허용하지 않게 해줘.

단,
기존 CLI argument 기반 실행 경로가 테스트나 개발 용도로 필요하다면
완전히 제거하지 말고 재사용 가능한 형태로 유지해줘.

예를 들어:

pnpm dev . "README 설명 수정"

도 가능하고,

pnpm dev

만 실행하면 askGoal()로 interactive input을 받는 구조면 좋다.

## 2. presenters/task-contract.ts 추가

현재 approve-task.ts 또는 cli.ts에서 직접:

console.log(JSON.stringify(task, null, 2))

하고 있는 TaskContract 출력 책임을 분리해줘.

예:

presentTaskContract(task)

역할:

- TaskContract를 Human이 확인할 수 있게 출력
- 출력만 담당
- 입력이나 승인 판단은 하지 않음

최소한 아래 항목이 확인 가능해야 함:

- Objective
- Target Repository
- Allowed Paths
- Forbidden Paths
- Constraints
- Acceptance Criteria
- Verification

JSON 그대로 출력해도 되지만,
가능하면 현재 구조를 복잡하게 만들지 않는 범위에서
사람이 읽기 쉬운 형태로 출력해줘.

중요:

Presenter는
"보여주기"만 담당한다.

## 3. prompts/approve-task.ts 책임 축소

현재 approveTask()가 TaskContract 출력과 승인 입력을 같이 담당하고 있다면 분리해줘.

변경 후:

presentTaskContract(task)
= TaskContract 출력

approveTask()
= Approve / Reject 입력만 담당

예:

Execute this task?
Type "Approve" to continue, anything else to reject:

>

또는 비슷한 단순한 UX면 충분하다.

반환:

Promise<boolean>

Approve인 경우에만 true.

그 외 입력은 false.

## 4. cli.ts 정리

cli.ts는 전체 terminal flow를 읽기 쉽게 연결하는 역할만 하게 해줘.

개념적으로:

runCli()
↓
Goal 입력 확보
↓
runPlanCommand(...)
↓
PlanningResult 확인
↓
presentTaskContract(...)
↓
approveTask()
↓
승인 시 runExecuteCommand(...)
↓
Execution Result

코드는 가능한 다음과 같이 읽히게 해줘:

const input = await ...
const planning = await ...
presentTaskContract(...)
const approved = await ...
if (!approved) ...
await runExecuteCommand(...)

즉 cli.ts 안에:

- Goal validation 세부 구현
- TaskContract formatting
- readline 구현
- Execution 세부 로직

을 직접 넣지 말 것.

## 5. commands/plan.ts

현재 역할을 유지하되,
interactive input과 argument input 모두에서 재사용할 수 있게 정리해줘.

역할:

GoalInput
→ GoalSpec 생성/검증
→ planTask() 호출
→ PlanningResult 반환

가능하면 string[] argv에 지나치게 종속되지 않게 해줘.

예를 들어:

runPlanCommand({
repositoryPath,
objective,
})

형태가 더 자연스럽다면 그렇게 리팩토링해도 된다.

단,
기존 테스트와 CLI 호환성을 고려해서 과도하게 바꾸지는 말 것.

## 6. commands/execute.ts

현재 v0 호환성을 유지해줘.

두 경로 모두 가능해야 한다.

v0:
TaskContract JSON file
→ loadTaskContract()
→ runExecuteCommand()

v1:
PlanningResult.task
→ runExecuteCommand()

즉 기존 Execution 단독 실행/테스트 경로는 삭제하지 말 것.

## 7. 테스트

아래 동작을 검증해줘.

1. askGoal()

- repository path 입력 가능
- objective 입력 가능
- 빈 입력 처리

2. presentTaskContract()

- 주요 TaskContract 정보가 출력되는지

3. approveTask()

- Approve → true
- Reject 또는 기타 입력 → false

4. runCli()

- arguments가 있으면 기존 방식으로 Planning 가능
- arguments가 없으면 interactive Goal input 사용
- Planning 실패 시 Approval/Execution 시작 안 함
- Reject 시 Execution 시작 안 함
- Approve 시 PlanningResult.task가 그대로 Execution으로 전달됨

5. 기존 v0 execution CLI 테스트 유지

6. 기존 planning 테스트 유지

## 8. 현재 v1의 책임 경계

아래 원칙을 유지해줘.

cli/
= Human과 terminal 사이의 interface

commands/
= application use-case 호출

prompts/
= Human input / Human decision

presenters/
= Human-readable output

application/
= 실제 Planning / Execution logic

즉:

CLI
→ Application을 호출

Application
→ CLI를 모름

방향을 유지할 것.

## 9. 이번 작업에서 하지 않을 것

- Web UI
- CEO Agent
- Multi-Agent
- Conflict / Agreement
- Retry / Re-plan
- Revise approval flow
- Persistent state
- 새로운 architecture layer
- 새로운 dependency
- 복잡한 terminal UI

이번 범위는 v1 Human Interface 최소 구조까지만이다.

## 10. 최종 기대 흐름

pnpm dev

실행 시:

AI Workspace

Repository:

> .

Goal:

> README 설명을 더 명확하게 수정해줘

Planning...

Task Contract
...

Execute this task?

> Approve

Execution...

Verification: passed

형태로 동작할 수 있게 만들어줘.

기존 argument 실행도 가능하면 유지:

pnpm dev . "README 설명을 더 명확하게 수정해줘"

작업 전 현재:

- src/cli/cli.ts
- src/cli/commands/plan.ts
- src/cli/commands/execute.ts
- src/cli/prompts/approve-task.ts
- package.json
- 관련 테스트

를 먼저 읽고 현재 구현을 최대한 재사용해줘.

작업 후 실행:

pnpm typecheck
pnpm test
git diff --check

마지막 보고 형식:

1. 변경된 파일
2. 새 CLI 구조
3. ask-goal 역할
4. presenter 분리 내용
5. approve-task 책임 변경
6. cli.ts 실행 흐름
7. v0 호환성 유지 방식
8. 테스트 결과
9. 남아 있는 주의점

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
