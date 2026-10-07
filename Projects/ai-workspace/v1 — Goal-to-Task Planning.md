# v1 — Goal-to-Task Planning

> **자연어 Goal과 대상 Repository를 입력받아 Repository Context를 조사하고, Human이 승인할 수 있는 실행 가능한 TaskContract를 자동 생성한다.**

---

## 1. v0에서 무엇을 만들었는가

v0에서는 최종 AI Software Organization Harness 전체를 만든 것이 아니다.

v0에서 구현한 것은 그 아래쪽의 **Reliable Worker Harness**, 즉 실행 엔진이다.

현재는 사람이 직접 `TaskContract` JSON을 작성해서 실행하지만, 이것이 최종 사용 방식은 아니다.

v0의 핵심 가치는 다음과 같다.

```text
AI가 나중에 작업 명령을 생성하더라도
↓
정해진 범위 안에서 실행하고
↓
변경 범위를 검사하고
↓
실제 명령으로 검증하고
↓
실행 결과와 증거를 보존할 수 있다.
```

즉, 지금까지 만든 것은:

> **AI가 안전하게 일할 수 있는 작업장**

이다.

---

## 2. v0가 필요한 이유

AI에게 단순히 다음과 같이 요청할 수도 있다.

```text
"kimbom.dev 모바일 애니메이션 문제 고쳐줘"
```

AI는 Repository를 읽고 직접 수정할 수 있다.

하지만 이 방식에는 중요한 통제 장치가 없다.

- 어디까지 수정해도 되는가?
- 어떤 파일은 건드리면 안 되는가?
- 무엇을 실행해서 성공을 검증해야 하는가?
- Worker의 작업이 실패했는가?
- 요청하지 않은 파일을 수정하지 않았는가?
- 실행 결과를 어떻게 보존할 것인가?
- 다음 Agent는 어떤 결과를 신뢰해야 하는가?

이 문제를 해결하기 위해 v0에서 다음 구성요소를 만들었다.

```text
TaskContract
Execution Workspace
Worker
Scope Enforcement
Verification
Evidence
Artifacts
```

이 계층의 핵심 목적은:

> **AI에게 자유를 주되, 경계와 증거는 Harness가 기계적으로 강제하는 것**

이다.

---

## 3. 최종적으로 원하는 흐름

최종적으로는 Human이 `TaskContract`를 직접 작성하지 않는다.

예를 들어 Human이:

```text
"포트폴리오 모바일 애니메이션 떨림을 개선해줘"
```

라고 요청하면 시스템은 다음과 같이 동작한다.

```text
Human
↓
Planner / CEO
↓
Goal 분석
↓
Repository 조사
↓
TaskContract 생성
↓
Execution Harness
↓
Worker 실행
↓
Scope / Verification / Evidence
↓
결과 평가
↓
필요하면 다음 Task 생성
↓
Human에게 최종 보고
```

따라서 다음과 같은 데이터:

```json
{
  "objective": "...",
  "allowedPaths": ["..."],
  "verification": ["..."]
}
```

는 최종적으로 Human이 직접 작성하는 입력 포맷이 아니다.

이것은:

> **Planner와 Worker 사이에서 사용되는 구조화된 실행 계약**

이 된다.

---

# 4. v0와 v1의 차이

## v0

```text
Human
↓
TaskContract 직접 작성
↓
Execution Harness
↓
Worker
```

v0의 책임은:

> **TaskContract → 안전한 실행**

이다.

---

## v1

```text
Human
↓
GoalSpec
↓
Repository Inspection
↓
Task Planner
↓
TaskContract 자동 생성
↓
Human Approval
↓
Execution Harness
↓
Worker
```

v1에서는 v0 앞에 다음 기능을 추가한다.

```text
Goal
→
TaskContract
```

즉:

> **v1 = Planning Layer**

이다.

---

## 이후 v2

v2부터는 처음 설계했던 AI Software Organization Harness에 가까워진다.

```text
Human
↓
CEO / Planner
↓
Repository 분석
↓
Task Decomposition
↓
Staffing / Coordination
↓
여러 TaskContract 생성
↓
Specialist Workers
↓
Evidence
↓
Verifier
↓
Retry / Re-plan
↓
Executive Report
```

다만 이것은 아직 구현하지 않는다.

---

# 5. Milestone

```text
v0 = Reliable Worker Harness 완료

v1 = Goal → TaskContract 자동 생성

v2 = Multi-task / Organization Orchestration
```

현재 다음 개발 단계는 **v1**이다.

---

# 6. v1 목표

v1의 목표는 다음과 같다.

> **Human이 자연어 Goal과 대상 Repository를 제공하면, AI가 Repository를 read-only로 조사하고 Repository 사실에 근거한 하나의 실행 가능한 TaskContract를 생성한다.**

전체 흐름:

```text
Human Goal
+
Target Repository
+
Optional Constraints
↓
Repository Inspection
↓
Relevant Context 수집
↓
Task Analysis
↓
TaskContract 생성
↓
TaskContract Schema Validation
↓
Human Review / Approval
↓
기존 v0 executeTask()
```

즉:

```text
v0

TaskContract
↓
Execution
```

앞에:

```text
v1

Goal
↓
Planning
↓
TaskContract
```

을 붙이는 것이다.

---

# 7. Human 입력 — GoalSpec

v1에서는 단순한 문자열 하나보다 최소한의 Human Governance 경계를 두는 것이 좋다.

예:

```ts
type GoalSpec = {
  objective: string;
  targetRepository: string;
  constraints?: string[];
};
```

예를 들어:

```json
{
  "objective": "프로젝트 모바일 애니메이션 떨림을 개선한다",
  "targetRepository": "/Users/omb/Documents/kimbom.dev",
  "constraints": [
    "새 dependency를 추가하지 않는다",
    "desktop behavior를 변경하지 않는다"
  ]
}
```

여기서 중요한 원칙은:

```text
Human이 지정한 Constraint
≠
Planner가 추론한 Constraint
```

이다.

Human이 지정한 제약은 Planner가 임의로 삭제하거나 완화하면 안 된다.

---

# 8. Repository Inspection

Planner는 Goal만 보고 TaskContract를 만들어서는 안 된다.

먼저 대상 Repository를 **read-only**로 조사한다.

```text
Target Repository
↓
Repository Inspection
↓
RepositoryContext
```

초기 RepositoryContext는 다음 정도면 충분하다.

```ts
type RepositoryContext = {
  repositoryName: string;
  fileTree: string[];
  packageScripts: Record<string, string>;
  instructions?: string;

  relevantFiles?: Array<{
    path: string;
    content: string;
  }>;
};
```

Repository Inspection이 조사할 대상은 예를 들어:

```text
Repository structure
AGENTS.md
package.json
package scripts
관련 source files
테스트 구조
기존 project rules
```

등이다.

중요한 경계는:

```text
Planner
= Repository를 관찰한다.

Worker
= Repository를 변경한다.
```

이다.

Planning 단계에서는 Repository를 수정하지 않는다.

---

# 9. Relevant Context 수집

Repository 전체를 그대로 Planner에게 넘기는 것은 적절하지 않다.

따라서 Inspection 결과에서 Goal과 관련된 Context를 선별한다.

예를 들어 Goal이:

```text
"모바일 프로젝트 캐러셀 떨림을 개선해줘"
```

라면 관련 Context는 다음과 같이 좁혀질 수 있다.

```text
src/components/projects/**
관련 animation controller
carousel helper
관련 style
mobile breakpoint
관련 tests
AGENTS.md
package scripts
```

즉:

```text
Repository
↓
Inspection
↓
Relevant Context Selection
↓
Planner
```

구조를 따른다.

---

# 10. Planner의 책임

Planner는 다음 입력을 받는다.

```text
GoalSpec
+
RepositoryContext
```

그리고 하나의 `TaskContract`를 생성한다.

AI가 결정해야 하는 항목은 다음과 같다.

```text
objective
targetRepository
allowedPaths
forbiddenPaths
constraints
acceptanceCriteria
verification
```

예를 들어 Human이:

```text
"프로젝트 모바일 애니메이션 떨림을 개선해줘"
```

라고 요청하면 Planner는 Repository를 조사한 뒤 다음과 같은 계약을 만들 수 있다.

```json
{
  "objective": "모바일 환경에서 프로젝트 캐러셀 애니메이션 떨림을 줄인다",

  "allowedPaths": ["src/components/projects/**", "src/styles/**"],

  "forbiddenPaths": ["package.json"],

  "constraints": [
    "새 dependency를 추가하지 않는다",
    "desktop behavior를 변경하지 않는다"
  ],

  "acceptanceCriteria": [
    "모바일 swipe 중 불필요한 보간이 제거된다",
    "desktop carousel 동작은 유지된다"
  ],

  "verification": ["pnpm typecheck", "pnpm test"]
}
```

---

# 11. TaskContract는 여전히 Runtime Source of Truth

Planner가 생성한 JSON을 바로 실행하면 안 된다.

반드시 기존 TaskContract Schema를 통과해야 한다.

```text
LLM Output
↓
Candidate TaskContract
↓
taskContractSchema.parse()
↓
Validated TaskContract
```

즉:

```text
AI의 출력
= 제안

Zod Schema
= 실행 가능한 Contract의 기준
```

이다.

Planner가 잘못된 데이터를 생성하면 실행 단계로 넘어가지 않는다.

---

# 12. Human Approval Gate

v1에서 반드시 들어가야 하는 핵심 기능이다.

Planner가 TaskContract를 생성했다고 해서 바로 Worker를 실행하지 않는다.

```text
GoalSpec
↓
AI Planning
↓
TaskContract
↓
Human Review
↓
Approve
↓
executeTask()
```

Human이 검토해야 하는 주요 항목은 다음과 같다.

```text
Objective

Allowed Paths

Forbidden Paths

Constraints

Acceptance Criteria

Verification
```

v1부터 AI가 처음으로:

- 어디를 수정할지
- 어디를 수정하지 않을지
- 무엇을 성공으로 판단할지
- 어떤 명령으로 검증할지

를 결정하기 시작한다.

따라서 이 단계에서 자동 실행까지 허용하면 Planning 오류와 Execution 오류를 구분하기 어려워진다.

Human 승인 전에는 Worker를 실행하지 않는다.

---

# 13. 초기 아키텍처 원칙과의 관계

최종 Harness에서는 CEO와 Worker가 긴 자연어 대화로 직접 연결되지 않는다.

다음과 같은 구조화된 프로토콜을 사용한다.

```text
Staffing Proposal
Shared State
Task Contract
Evidence
Telemetry
```

v1에서는 이 중 모든 계층을 구현하지 않는다.

현재 구조는:

```text
Planner
↓
TaskContract
↓
Worker
↓
Evidence
```

이다.

즉 초기 원칙인:

```text
❌ Planner ↔ Worker 자유 대화

✅ Planner
   ↓
   Structured Contract
   ↓
   Worker
```

는 v1부터 이미 적용한다.

---

# 14. v1에서 하지 않을 것

다음 기능은 의도적으로 v1 범위에서 제외한다.

```text
Multi Agent

CEO Agent 전체 구현

Staffing Proposal

Agent Staffing

Shared State

Shared Memory

Task Decomposition

Multiple TaskContracts

Automatic Retry

Automatic Re-plan

Token Budget

Agent Budget

Automatic Merge

Executive Report
```

현재 실행 경로는 사실상:

```text
Planner
↓
Single Worker
```

로 고정되어 있다.

따라서 지금 Staffing이나 Multi-Agent 구조를 만들면 실제 의사결정보다 추상화가 먼저 생기게 된다.

v1의 책임은 끝까지:

> **One Goal → One Good TaskContract**

이다.

---

# 15. v1 성공 기준

v1이 완료되었다고 판단하기 위한 기준은 다음과 같다.

1. Human이 자연어 Goal을 입력할 수 있다.
2. 대상 Repository를 지정할 수 있다.
3. Human Constraint를 선택적으로 지정할 수 있다.
4. Repository를 read-only로 조사한다.
5. Repository 구조와 프로젝트 규칙을 파악한다.
6. Goal과 관련된 Context를 수집한다.
7. Repository 사실에 근거한 하나의 TaskContract를 생성한다.
8. 생성된 TaskContract가 기존 Zod Schema를 통과한다.
9. Human이 생성된 Contract를 확인할 수 있다.
10. Human 승인 없이는 실행되지 않는다.
11. 승인된 Contract를 기존 v0 `executeTask()`에 전달할 수 있다.
12. 기존 v0의 Scope / Verification / Evidence / Artifacts 흐름이 그대로 동작한다.

---

# 16. v1 완료 상태

v1의 완료 상태를 한 문장으로 정의하면:

> **Human Goal을 Repository 사실에 근거한, 검증 가능하고 승인 가능한 하나의 TaskContract로 변환할 수 있는 Planning Layer**

이다.

전체 흐름은 다음과 같다.

```text
┌──────────────────────────────────────┐
│ Human Governance                     │
│                                      │
│ Goal                                 │
│ Repository                           │
│ Constraints                          │
└────────────────┬─────────────────────┘
                 │ GoalSpec
                 ▼
┌──────────────────────────────────────┐
│ Repository Inspection                │
│                                      │
│ file tree                            │
│ AGENTS.md                            │
│ package scripts                      │
│ relevant source                      │
│ project rules                        │
└────────────────┬─────────────────────┘
                 │ RepositoryContext
                 ▼
┌──────────────────────────────────────┐
│ Task Planning                        │
│                                      │
│ Goal Analysis                        │
│ Context Interpretation               │
│ Scope Decision                       │
│ Constraint Preservation              │
│ Acceptance Criteria                  │
│ Verification Selection               │
└────────────────┬─────────────────────┘
                 │
                 ▼
            TaskContract
                 │
                 ▼
        Zod Schema Validation
                 │
                 ▼
┌──────────────────────────────────────┐
│ Human Approval Gate                  │
│                                      │
│ APPROVE / REJECT                     │
└────────────────┬─────────────────────┘
                 │ approved
                 ▼
┌──────────────────────────────────────┐
│ v0 Reliable Worker Harness           │
│                                      │
│ Workspace                            │
│ Worker                               │
│ Scope Enforcement                    │
│ Verification                         │
│ Evidence                             │
│ Artifacts                            │
└──────────────────────────────────────┘
```

---

# 17. 구현 순서

v1 개발은 다음 순서로 진행한다.

```text
1. GoalSpec 정의
↓
2. RepositoryContext 정의
↓
3. Repository Inspection 구현
↓
4. Relevant Context Selection
↓
5. Planner 입력/출력 경계 정의
↓
6. GoalSpec + RepositoryContext → TaskContract
↓
7. TaskContract Zod Validation
↓
8. CLI Planning Flow 연결
↓
9. Human Approval Gate
↓
10. 기존 executeTask() 연결
```

첫 구현 단위는:

> **GoalSpec과 RepositoryContext의 최소 모델을 정의하고, Repository를 read-only로 조사할 수 있게 만드는 것**

으로 잡는다.
