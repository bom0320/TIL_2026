# CEO, Specialist, Planning, Execution 책임 및 Workflow 설계

## 1. 설계 배경

AI Workspace의 최종 목표는 여러 AI Agent가 단순히 의견을 제시하거나, CEO가 판단을 종합한 뒤 단일 Worker에게 구현을 맡기는 시스템이 아니다.

**CEO가 여러 Specialist를 조직하고, 각 Specialist가 자신의 전문 분야에서 분석·판단·상세 설계·구현·검증을 수행하는 AI Development Organization**을 구축하는 것이다.

Human은 상위 목표와 제약을 제공한다.

CEO는 이를 분석하여 적절한 Specialist를 구성하고, 각 분야의 독립적인 판단을 수집한다.

Specialist들의 판단은 비교·종합되며, Human은 제안된 해결 방향을 승인하거나 수정할 수 있다.

이후 CEO는 승인된 방향을 실제 개발 업무로 분해하고, 담당 Specialist에게 업무를 배정한다.

각 Specialist는 승인된 결정을 바탕으로 상세 설계를 진행하며, 필요한 경우 기존 v1 Planning 기능을 활용하여 실행 가능한 TaskDraft를 생성한다.

Harness는 이를 TaskContract로 구성하고, 승인된 범위 안에서 Specialist가 실제 업무를 수행하도록 통제한다.

CEO는 결과를 평가하고 추가 작업, 재설계, 재할당, Human Escalation 등을 수행한다.

---

## 2. 핵심 개념과 책임

가장 중요한 설계 원칙은 **AI 직원의 정체성과 직원이 사용하는 기능을 구분하는 것**이다.

### 2.1 CEO — 조직 운영 및 의사결정

CEO는 전체 Goal을 책임지는 상위 Agent다.

주요 책임:

- Human Goal과 Constraints 분석
- 필요한 Specialist 선정
- 전문가별 분석 업무 배분
- Specialist Decisions 비교·종합
- 해결 방향 제안
- Human Approval 요청
- 승인된 해결 방향을 실제 개발 업무로 분해
- 개발 업무의 담당자와 의존성 결정
- 작업 결과 검토 및 재작업·재배치 판단
- 최종 결과 보고

CEO는 모든 코드를 직접 작성하는 역할이 아니다.

CEO의 핵심 책임은 **무엇을 해야 하고, 누가 수행해야 하며, 결과를 어떻게 관리할지 결정하는 것**이다.

### 2.2 Specialist — 전문 분야의 실제 업무 수행자

Specialist는 Frontend, Backend, UX, QA, Reviewer 등 전문적인 역할을 가진 AI 직원이다.

Specialist는 단순한 의견 생성 Agent가 아니다.

업무에 따라 다음 능력을 활용할 수 있다.

```
Specialist
 ├─ Analyze
 │   └─ 문제와 Repository 분석
 │
 ├─ Decide
 │   └─ 전문적인 판단과 해결 방안 제시
 │
 ├─ Design
 │   └─ 승인된 방향에 따른 상세 설계
 │
 ├─ Plan
 │   └─ 구체적인 실행 작업 명세 작성
 │
 ├─ Execute
 │   └─ 코드 수정, 테스트 작성 등
 │
 └─ Review
     └─ 결과 검토 및 개선 사항 제안
```

다만 모든 Specialist가 이 기능을 전부 사용해야 하는 것은 아니다.

업무 유형과 승인된 Authority에 따라 사용 가능한 능력이 결정된다.

예를 들어 Frontend Specialist는 판단과 코드 구현을 모두 수행할 수 있지만, 읽기 전용 Reviewer는 실제 Repository 수정 권한이 없을 수 있다.

**Specialist는 업무의 주체이며, Planning과 Execution은 업무 수행에 사용하는 기능이다.**

### 2.3 Planner — 업무 구체화 기능

Planner는 기존 v1에서 구현한 Planning 기능이다.

기존 v1에서는 Planner가 다음을 함께 담당했다.

1. Repository 조사
2. 기술적 해결 방향 판단
3. 실행 작업 범위 구체화
4. TaskDraft 생성

CEO와 Specialist가 존재하지 않았기 때문에 Planner가 기술적 판단과 업무 명세 작성을 함께 수행한 것이다.

그러나 v4에서는 이미 Specialist들이 해결 방향을 판단하고 Human이 승인한 상태에서 개발 업무가 시작된다.

따라서 v4에 통합되는 Planner의 책임은 다음과 같이 좁혀야 한다.

> **승인된 해결 방향과 담당 Specialist의 상세 설계를 실행 가능한 TaskDraft로 구체화한다.**

Planner는 Specialist의 전문적 판단을 대체하지 않는다.

이미 승인된 설계를 임의로 변경하거나 새로운 해결 방향을 독자적으로 선택해서는 안 된다.

또한 Planner를 반드시 별도의 AI 직원으로 구현할 필요는 없다.

담당 Specialist가 공통 Planning 기능을 이용하여 자신의 상세 설계와 TaskDraft 생성을 하나의 작업 흐름으로 수행할 수 있다.

즉, `Specialist → Planner`는 반드시 서로 다른 두 Agent의 순차 호출을 의미하지 않는다.

### 2.4 Worker — 실행 역할 및 Runtime

Worker는 별도의 고정된 AI 직원을 의미하지 않는다.

기존 v0에서 구현한 Codex Worker는 TaskContract를 기반으로 실제 Repository 작업을 수행하는 실행 기능이다.

v4에서는 담당 Specialist가 자신의 승인된 업무를 수행할 때 이 실행 기능을 재사용할 수 있다.

```
Frontend Specialist
    ├─ Frontend 전문성
    ├─ 승인된 업무
    ├─ Planning 기능
    └─ Execution 기능
         └─ 기존 v0 Codex Worker 활용
```

Frontend Specialist와 QA Specialist가 동일한 Codex 실행 엔진을 사용하더라도, 전문 역할과 업무 범위 및 Context는 다를 수 있다.

따라서 다음과 같이 이해한다.

- Specialist는 **누가 일하는가**를 나타낸다.
- Worker Runtime은 **그 일을 어떤 실행 메커니즘으로 수행하는가**를 나타낸다.

### 2.5 Harness — 거버넌스 및 통제

Harness는 AI 조직의 실행 환경과 운영 규칙을 소유한다.

주요 책임:

- Context 접근 범위 관리
- Specialist Authority 검증
- Human Approval 관리
- TaskContract 구성 및 검증
- 실행 환경 격리
- 파일 수정 Scope 검사
- Verification 실행
- Evidence와 Artifacts 보존
- 승인되지 않은 작업 및 권한 확대 방지

CEO와 Specialist는 자율적으로 판단할 수 있지만, 자신의 권한을 임의로 확대할 수는 없다.

---

## 3. 전체 Workflow

AI 조직의 실행 과정을 두 단계로 구분한다.

- **Phase A:** 전문가 의사결정과 해결 방향 승인
- **Phase B:** 승인된 방향을 바탕으로 상세 설계 및 개발 실행

### Phase A — Decision & Approval

```
Human Goal / Constraints
            ↓
CEO Intent Analysis
            ↓
CEO Staffing Proposal
            ↓
Harness Validation
            ↓
Human Staffing Approval
            ↓
Authorized Specialist Invocation
            ↓
Independent Specialist Decisions
 ├─ Frontend Decision
 ├─ Backend Decision
 ├─ UX Decision
 └─ QA Decision
            ↓
Decision Comparison (v3)
            ↓
CEO Synthesis
            ↓
Human Governance
 ├─ Approve
 ├─ Override
 └─ Reject
            ↓
      Approved Decision
```

이 단계의 목적은 **어떤 해결 방식을 채택할지 결정하는 것**이다.

Specialist는 자신의 전문 분야에서 해결 방안을 판단하고 Evidence, Risks, Uncertainty를 제시한다.

현재 v2에서는 Specialist가 서로 독립적으로 판단한다.

v3에서는 이 판단들을 Concern 단위로 비교한다.

CEO는 비교 결과를 바탕으로 최종 Recommendation을 제안하며, Human이 방향을 승인한다.

추가 검토가 필요하다면 CEO가 다시 전문가 판단을 요청할 수 있지만, 이러한 반복 조정은 향후 구현할 영역이다.

### Phase B — Detailed Design & Execution

```
Approved Decision
        ↓
CEO Work Decomposition
        ↓
Specialist Assignment
        ↓
담당 Specialist
 ├─ 승인된 Decision 확인
 ├─ 담당 업무 분석
 ├─ 전문적인 판단
 └─ 상세 설계
        ↓
Planning 기능 활용 (v1 재사용)
        ↓
TaskDraft
        ↓
Harness — TaskContract 구성 및 검증
        ↓
Human Task Approval
        ↓
담당 Specialist 실행 (v0 활용)
        ↓
Scope / Verification / Review
        ↓
ExecutionResult
        ↓
CEO Result Evaluation
 ├─ Continue
 ├─ Rework
 ├─ Reassign
 ├─ Escalate
 └─ Complete
```

여기서 중요한 점은 다음과 같다.

**Specialist는 Planning 이전에 이미 자신의 업무를 분석하고 상세 설계를 수행한다.**

Planning은 그 판단 결과를 실행 가능한 작업 명세로 구체화하는 기능이다.

이후 승인된 TaskContract를 이용해 담당 Specialist가 직접 업무를 수행한다.

단, 업무 유형에 따라 TaskDraft가 필요하지 않을 수도 있다.

예를 들어 순수한 분석 업무는 AgentDecision을, 상세 설계 업무는 설계 산출물을 반환할 수 있다.

현재 TaskContract는 주로 Repository 수정 작업을 실행하기 위한 계약이다.

---

## 4. 기존 v1이 실제로 작동하는 방식

현재 v1의 흐름을 먼저 이해해야 v4 통합 과정에서 무엇이 변경되는지 명확해진다.

### 4.1 기존 v1 Workflow

```
Human Goal
    ↓
GoalSpec
    ↓
inspectRepository()
    ↓
inspectInitialFiles()
    ↓
PlanningState
    ↓
createPlannerContext()
    ↓
Codex Planner
    │
    ├─ inspect
    │    ↓
    │  Harness가 추가 파일 조사
    │    ↓
    │  PlanningState 갱신
    │    ↓
    │  Planner 재호출
    │
    └─ complete
         ↓
       TaskDraft
         ↓
buildTaskContract()
         ↓
     TaskContract
         ↓
   Human Task Approval
         ↓
    v0 Execution
```

단순한 텍스트 수정은 Fast Path로 Planner LLM 호출 없이 TaskDraft가 생성될 수 있다.

### 4.2 Planner가 실제로 하던 일

기존 Planner는 Human Goal과 Repository Context만을 보고 판단했다.

예를 들어:

> 로그인 중복 요청 문제를 해결한다.

Planner는 관련 파일을 조사하고 어떤 방식으로 구현할지 판단한 뒤 다음 정보를 구체화했다.

- 수정할 파일
- 수정할 목표
- 변경하면 안 되는 파일
- 구현 제약
- 작업 성공 조건
- 검증 명령

즉, 기존 Planner는 **기술적 판단과 실행 작업의 명세화**를 한 번에 수행했다.

### 4.3 기존 v1의 한계

현재 v1은 기본적으로 다음 관계를 사용한다.

```
GoalSpec
   ↓
Planner
   ↓
TaskDraft
   ↓
TaskContract
```

이 구조에는 아직 다음 정보가 없다.

- CEO가 승인받은 해결 방향
- 담당 Specialist의 Identity
- CEO가 배분한 개별 Assignment
- Specialist의 상세 설계 산출물
- 여러 TaskContract 사이의 의존성
- CEO의 결과 평가 및 재작업 과정

또한 현재 `planTask()`는 한 번의 Planning 요청에서 하나의 TaskContract를 반환한다.

이것은 기존 v1이 잘못 설계되었다는 의미가 아니다.

v1이 처음에는 단일 Goal을 안전한 실행 명세로 변환하는 것을 목표로 했기 때문이다.

---

## 5. v4에서 v1 Planner의 책임 변화

### 기존 v1

```
Human Goal
     ↓
Planner
 ├─ Repository 조사
 ├─ 해결 방법 판단
 ├─ 수정 범위 판단
 └─ TaskDraft 생성
```

### v4 통합 후 목표

```
Approved Decision
        +
CEO Assignment
        ↓
담당 Specialist
 ├─ 업무 분석
 ├─ 전문적인 판단
 └─ 상세 설계
        ↓
Planning 기능
 ├─ 실행 범위 구체화
 ├─ Constraints 구조화
 ├─ Acceptance Criteria 정의
 └─ Verification 정의
        ↓
TaskDraft
```

기존 Planner가 담당하던 기술적 판단의 상당 부분을 Specialist의 분석·설계 과정에서 수행하게 되는 것이다.

따라서 Planning은 승인된 판단과 상세 설계를 실행 계약에 필요한 형식으로 변환하는 역할에 집중한다.

### 중요한 설계 원칙

**Planner는 승인된 설계의 대체자가 아니라 구체화 도구다.**

예를 들어 Frontend Specialist가 다음과 같이 설계했다고 가정한다.

> `useLogin`에서 Pending 상태를 관리하고, `LoginButton`은 해당 상태를 참조하여 중복 제출을 차단한다.

Planning 기능은 이 판단을 바탕으로 다음을 정의한다.

```
Objective
- 로그인 중복 제출 방지

Allowed Paths
- src/hooks/useLogin.ts
- src/components/LoginButton.tsx

Constraints
- 기존 API 인터페이스 유지
- 승인된 상태 관리 설계 준수

Acceptance Criteria
- Pending 중 추가 요청 미발생
- 로딩 상태가 사용자에게 표시됨

Verification
- pnpm typecheck
- pnpm test
```

Planner가 기존 설계를 무시하고 새로운 상태 관리 방식이나 라이브러리를 독자적으로 선택하는 것은 목표 책임에 맞지 않는다.

다만 상세 설계와 실제 Repository 사이에 충돌이 발견되면 조용히 다른 방식으로 구현하는 것이 아니라 Specialist 또는 CEO에게 재검토를 요청해야 한다.

---

## 6. Assignment, TaskDraft, TaskContract의 차이

세 개념은 서로 다른 책임을 가진다.

| 개념         | 핵심 질문                     | 책임                              |
| ------------ | ----------------------------- | --------------------------------- |
| Assignment   | 누가 어떤 일을 맡을 것인가?   | CEO 업무 배분                     |
| TaskDraft    | 이 일을 어떻게 실행할 것인가? | 실행 명세 초안                    |
| TaskContract | 어떤 조건에서 실행할 것인가?  | Harness가 구성하는 공식 실행 명세 |

### 6.1 Assignment

CEO가 직원에게 맡기는 업무다.

예시:

```
Specialist: Frontend

Objective:
로그인 중복 요청 방지 기능 구현

Basis:
Human이 승인한 해결 방향

Expected Outcome:
안정적인 로그인 상태 관리
```

이 단계에서는 업무의 담당자와 목적이 중심이다.

### 6.2 TaskDraft

담당 Specialist의 상세 설계를 바탕으로 Planning 기능이 작성하는 작업 초안이다.

현재 v1의 TaskDraft에는 다음 필드가 있다.

```
type TaskDraft = {
  objective: string;
  targetRepository: string;
  allowedPaths: string[];
  forbiddenPaths: string[];
  constraints: string[];
  acceptanceCriteria: string[];
  verification: string[];
};
```

TaskDraft는 Codex에게 전달할 업무 지시를 구조화한 것이다.

하지만 아직 공식 TaskContract는 아니다.

### 6.3 TaskContract

현재 코드에서는 TaskDraft에 `id`, `goalId`를 부여하고 원본 Human Goal의 Repository와 Constraints를 반영한다.

```
TaskDraft
   +
Task ID / Goal ID
   +
Original Human Constraints
   +
Schema Validation
   ↓
TaskContract
```

TaskContract는 목표, 수정 범위, 제약, 성공 조건, 검증 명령을 명시한 실행 계약이다.

**TaskContract 생성 자체가 Human Approval이나 실행 권한 발급을 의미하지는 않는다.**

현재 `buildTaskContract()`는 계약 데이터를 구성하고 Zod Schema를 검증한다. 별도의 정책 검증과 Human Approval은 다른 단계의 책임이다.

### 6.4 TaskContract는 업무 단위로 생성한다

```
Human Goal
   ↓
Approved Decision
   ↓
CEO Work Decomposition
   │
   ├─ Frontend Assignment
   │    ├─ TaskDraft A
   │    └─ TaskContract A
   │
   ├─ QA Assignment
   │    ├─ TaskDraft B
   │    └─ TaskContract B
   │
   └─ Additional Assignment
        ├─ TaskDraft C
        └─ TaskContract C
```

하나의 Specialist가 여러 TaskContract를 수행할 수 있다.

반대로 모든 Assignment에 TaskContract가 필요한 것은 아니다.

따라서 `Specialist 1명 = TaskContract 1개` 관계를 전제로 설계하지 않는다.

---

## 7. Specialist와 Planning 기능의 실제 관계

이 부분은 반드시 명확하게 유지한다.

**Specialist와 Planner는 동일한 개념이 아니지만, Planner가 반드시 별도의 Agent일 필요도 없다.**

Specialist는 직원의 역할과 전문성을 나타낸다.

Planner는 실행 작업을 구체화하는 기능이다.

따라서 다음 두 가지 구현 방식이 가능하다.

### 방식 A — Specialist가 Planning 기능을 사용

```
Frontend Specialist
 ├─ 업무 분석
 ├─ 상세 설계
 ├─ Planning 기능 사용
 │    └─ TaskDraft 생성
 └─ 승인 후 Execution
```

Specialist가 자신의 업무를 계획하고, 공통 Planning 기능을 이용해 TaskDraft를 만든다.

### 방식 B — 별도 Planner 호출

```
Frontend Specialist
     ↓
상세 설계 산출물
     ↓
공통 Planner Agent
     ↓
TaskDraft
     ↓
Frontend Specialist Execution
```

이 방식에서는 상세 설계와 명세 구체화가 서로 다른 Agent 호출로 분리된다.

단, 별도 Planner가 이미 결정된 해결 방향을 다시 판단하지 않도록 제한해야 한다.

### 현재 권장 방향

직원 역할과 기능을 분리하고, 기존 v1 Planning을 재사용 가능한 Capability로 유지한다.

별도의 Planner Agent 실행이 필요한지, Specialist 실행 안에서 Planning을 함께 수행할지는 실제 지연·품질·Context 전달 요구를 확인해 결정한다.

**개념적으로 Specialist의 상세 설계가 Planning보다 먼저 존재해야 하지만, 구현상 반드시 두 번의 LLM 호출로 나누어야 하는 것은 아니다.**

---

## 8. Staffing과 Governance의 관계

### Staffing

Staffing은 어떤 Specialist가 필요한지, 어떤 업무를 맡길지 판단하는 기능이다.

현재 v4의 Staffing은 초기 의사결정 단계에서의 전문가 구성을 중심으로 설계되어 있다.

핵심 산출물은 다음과 같다.

- StaffingAssignmentDraft
- StaffingDraft
- StaffingProposal

이후 개발 단계에서 필요한 Implementation Assignment는 별도 설계가 필요하다.

### Governance

Governance는 승인된 업무의 실행 권한을 통제한다.

핵심 책임은 다음과 같다.

- Approved Scope 확인
- Specialist Identity 및 Assignment 확인
- 요청 Context 범위 검사
- 호출 예산 검사
- 권한 확대 차단
- 필요한 경우 Human Reapproval 요청

CEO가 StaffingProposal이나 TaskDraft를 생성했다고 해서 실행 권한을 얻는 것은 아니다.

현재 구현된 `evaluateInvocationRequest()`는 호출 자격만 평가한다.

실제 Scope 발급, 예산 예약, 실행 허가는 후속 구현 영역이다.

### 권한 흐름

```
CEO / Specialist Proposal
          ↓
    Harness Validation
          ↓
    Human Approval
          ↓
   Harness Authorization
          ↓
      Agent Execution
          ↓
 Verification / Evidence
          ↓
    CEO Result Evaluation
```

AI의 판단과 시스템의 실행 권한은 분리한다.

---

## 9. 기존 v0~v4의 역할 정리

| 버전 | 현재 구현 역할                         | 최종 조직에서의 역할                            |
| ---- | -------------------------------------- | ----------------------------------------------- |
| v0   | Safe Execution                         | 각 Specialist의 구현 작업 실행·검증             |
| v1   | Goal-to-Task Planning                  | 승인된 업무를 TaskDraft·TaskContract로 구체화   |
| v2   | Independent Decision                   | Specialist의 독립적인 전문 판단                 |
| v3   | Structured Comparison                  | 전문가 판단의 Agreement·Conflict·Trade-off 비교 |
| v4   | CEO Governance & Orchestration 개발 중 | 전체 조직의 의사결정·업무 분배·권한·진행 관리   |

기존 v0~v3는 v4로 대체하는 대상이 아니다.

v4는 기존 기능을 조직 단위의 Workflow로 연결하는 상위 계층이다.

다만 현재 v2 Specialist는 읽기 전용 판단 Agent이고, v0의 Codex Worker는 Specialist 정체성을 모르는 독립 실행 기능이다.

두 기능을 Specialist 중심으로 통합하는 것은 앞으로 구현해야 한다.

---

## 10. 현재 구현 상태와 앞으로 필요한 작업

### 현재 구현되어 있는 것

- v0 TaskContract 기반 Codex Execution
- Git Worktree 격리
- Scope 및 Verification
- Execution Evidence / Artifacts
- v1 Goal → TaskDraft → TaskContract
- v1 Planning Context / Inspection Loop / Fast Path
- v2 독립 Specialist Decision
- v3 Decision Comparison
- v4 Staffing 및 Governance Domain Schema 일부
- v4 Snapshot 기반 Staffing Context Discovery
- v4 Invocation Eligibility Evaluation

### 아직 구현되지 않은 것

- 실제 CEO Agent 및 Staffing Loop
- Staffing Proposal Finalization / Human Staffing Approval
- ApprovedOperatingScope 발급 및 실제 Invocation Authorization
- CEO Synthesis 및 Human Decision Governance
- ApprovedDecision Domain과 승인 이력
- 승인된 Decision을 Specialist 상세 설계에 전달하는 흐름
- 상세 설계 산출물을 v1 Planning에 전달하는 계약
- 개별 Assignment와 TaskContract의 연결 관계
- Specialist Identity를 반영한 실제 Code Execution
- 여러 Specialist의 작업 의존성 및 실행 관리
- 실패 이후 재작업·재계획·재할당
- Worktree 변경사항 통합과 충돌 처리
- CEO Executive Report
- LangGraph.js 기반 전체 Workflow와 Pause / Resume

앞으로의 목표와 현재 코드가 이미 제공하는 기능을 혼동하지 않는다.

---

## 11. 앞으로의 설계에서 고정할 원칙

### 원칙 1. CEO와 Specialist의 책임을 구분한다

CEO는 조직의 해결 방향, 업무 배분, 진행 관리를 책임진다.

Specialist는 자신의 전문 분야에서 실제 판단과 업무 수행을 책임진다.

### 원칙 2. Specialist의 상세 설계를 Planning이 덮어쓰지 않는다

Approved Decision과 Specialist의 상세 설계는 TaskDraft 생성의 기준이 되어야 한다.

Planner가 이를 임의로 변경해서는 안 된다.

### 원칙 3. Planning은 Capability다

Planner를 반드시 별도의 직원으로 취급하지 않는다.

기존 v1 기능을 각 Specialist가 사용할 수 있는 공통 Planning Capability로 재사용한다.

### 원칙 4. Worker는 실행 메커니즘이다

기존 v0 Codex Worker는 Specialist가 실제 작업할 때 활용하는 실행 기능이다.

Specialist와 별도의 고정된 개발 직원을 만들 필요는 없다.

### 원칙 5. TaskContract는 업무 단위다

하나의 Specialist가 여러 TaskContract를 수행할 수 있다.

각 TaskContract는 담당 업무의 범위, 제약 및 검증 조건을 가진다.

### 원칙 6. 승인된 결정과 실행 승인은 다르다

Human이 해결 방향을 승인했다고 해서 모든 코드 수정이 자동으로 승인되는 것은 아니다.

TaskContract에 대한 Human Approval과 Harness의 실행 통제가 별도로 필요하다.

### 원칙 7. 기존 구현을 우선 재사용한다

현재 v0, v1, v2, v3를 전면 재작성하지 않는다.

먼저 책임과 데이터 전달 관계를 명확히 한 뒤 필요한 API와 Domain Contract만 확장한다.

---

## 12. 최종 책임 관계

```
Human
 └─ Goal / Constraints
       ↓
CEO
 ├─ 목표 분석
 ├─ 전문가 구성
 ├─ 전문가 판단 수집
 ├─ 해결 방향 종합
 └─ Human Approval 요청
       ↓
Approved Decision
       ↓
CEO
 └─ 실제 개발 업무 배정
       ↓
Specialist
 ├─ 업무 분석
 ├─ 전문적인 판단
 ├─ 상세 설계
 │      ↓
 ├─ Planning Capability
 │      ↓
 ├─ TaskDraft
 │      ↓
 │   Harness
 │   └─ TaskContract 구성·검증
 │      ↓
 │   Human Task Approval
 │      ↓
 ├─ Execution Capability
 │   └─ 코드 수정 및 테스트
 │      ↓
 └─ ExecutionResult
       ↓
Harness Verification / Review
       ↓
CEO Result Evaluation
 ├─ 추가 작업
 ├─ 재작업
 ├─ 재할당
 ├─ Human Escalation
 └─ 완료 보고
```

## 최종 결론

**CEO는 업무를 결정하고 관리한다.**

**Specialist는 전문적인 판단과 상세 설계를 수행하고, 승인된 업무를 실제로 실행한다.**

**Planning은 Specialist의 판단과 설계를 TaskDraft로 구체화하는 기능이다.**

**TaskContract는 Harness가 구성한 공식 실행 명세이며, 별도의 승인 및 권한 통제를 거쳐 실행에 사용된다.**

**Execution은 Specialist가 기존 v0 Worker Runtime을 통해 실제 작업을 수행하는 기능이다.**

따라서 v4의 핵심은 새로운 Planner나 Worker를 여러 개 만드는 것이 아니다.

기존에 구현한 Planning과 Execution을 여러 Specialist가 사용할 수 있도록 연결하고, CEO가 이 직원들의 의사결정과 실제 업무를 관리하도록 만드는 것이다.

> 최종 설계 원칙: **CEO와 Specialist는 판단과 업무 수행의 주체이고, Planning과 Execution은 이들이 활용하는 기능이며, Harness는 전체 조직의 권한과 실행을 통제하는 시스템이다.**
