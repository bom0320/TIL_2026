# Application State, LangGraph State, Governance Ledger의 책임 구분

## 1. 문제 인식

AI Workspace에는 이미 실행 데이터와 상태를 관리하는 구조가 존재한다.

- **Application State:** Staffing 등 개별 기능의 실행 과정에서 발생하는 상태 관리
- **Governance Ledger:** System Policy, Human Approval, Scope, Reservation 등의 영속 기록
- **LangGraph State:** Workflow의 Node 간 데이터 전달 및 실행 상태 관리

따라서 다음 의문이 발생했다.

> 이미 Application과 Governance Ledger에서 데이터를 관리하고 있는데, LangGraph State까지 별도로 필요한 이유는 무엇인가? 같은 데이터를 중복 저장하는 것은 아닐까?

핵심은 데이터를 저장한다는 행위 자체가 아니라, **각 시스템이 어떤 목적의 상태를 소유하는지 구분하는 것**이다.

---

## 2. 세 가지 State의 책임

| 구분        | Application State                   | LangGraph State                                  | Governance Ledger                    |
| ----------- | ----------------------------------- | ------------------------------------------------ | ------------------------------------ |
| 목적        | 개별 기능 실행                      | 기능 간 Workflow 연결                            | 승인·권한 증명                       |
| 범위        | Application 내부                    | 여러 Node 및 Phase                               | Governance 전반                      |
| 주요 데이터 | Inspection, Context, 실행 중간 상태 | Goal, Proposal, Decision, 단계별 결과            | Policy, Approval, Scope, Reservation |
| 주요 질문   | 이 기능을 어떻게 수행하고 있는가?   | 어디까지 실행했고 다음에 무엇을 수행해야 하는가? | 현재 이 행동을 허용할 수 있는가?     |
| 영속성      | 구현 방식에 따라 다름               | Checkpointer 사용 시 보존·복원 가능              | 기존 서명된 Ledger에 영속 저장       |

세 영역은 일부 동일한 데이터를 참조할 수 있지만, **데이터에 대한 책임과 신뢰 수준은 다르다.**

### Application State

개별 Application이 자신의 기능을 수행하기 위해 사용하는 내부 상태다.

예를 들어 `runStaffing()`은 다음을 관리한다.

- Repository Snapshot
- Context Discovery 및 Inspection
- CEO Staffing Loop
- Staffing Draft 검증
- 최종 StaffingProposal 생성

이 내부 상태를 모두 LangGraph로 이동시킬 필요는 없다.

Application은 기존 기능을 독립적으로 수행하고, 최종 결과인 `StaffingRunResult`를 반환하면 된다.

### LangGraph State

여러 Application의 실행 결과를 다음 Node로 전달하기 위한 Workflow 상태다.

예를 들어:

```
runStaffing()
    ↓
StaffingRunResult
    ↓
LangGraph State
    ↓
Approval Node
```

Graph는 Staffing 내부의 Inspection 기록 전체를 소유할 필요가 없다.

다음 Approval Node가 필요로 하는 Proposal과 결과 상태만 전달하면 된다.

또한 **LangGraph State 자체가 영속 저장소는 아니다.**

- State: Node 간 공유하는 실행 데이터
- Graph Runtime: 실행 순서와 분기 관리
- Checkpointer: 실행 상태의 저장 및 복원

Checkpointer를 연결하지 않은 현재 Graph State는 장기적으로 복원 가능한 실행 이력을 제공하지 않는다.

### Governance Ledger

승인과 권한의 신뢰된 기록을 관리한다.

- System Policy
- Human Delegation
- ApprovalRecord
- ApprovedOperatingScope
- InvocationReservation
- Claim 상태

Ledger는 단순히 실행 상태를 기억하는 저장소가 아니다.

Harness가 실제 실행 권한을 검증할 때 사용하는 **신뢰된 기록의 기준**이다.

---

## 3. 실제 Workflow에서의 역할

```
Human Goal
    ↓
LangGraph
    ↓
runStaffing()
    └─ Application 내부 State 관리
    ↓
StaffingProposal
    ↓
LangGraph State
    └─ Proposal을 Approval 단계로 전달
    ↓
Human Staffing Approval
    ↓
Governance Ledger
    └─ ApprovalRecord / ApprovedOperatingScope 저장
    ↓
LangGraph State
    └─ Scope ID 등 다음 단계에 필요한 참조 전달
    ↓
Authorization
    └─ Governance Ledger에서 현재 권한 재검증
    ↓
Specialist 실행
```

이 흐름에서 세 영역의 역할은 명확하다.

- Application은 개별 업무를 수행한다.
- LangGraph는 업무 간 실행 순서와 데이터 전달을 관리한다.
- Governance는 실제 행동의 허용 여부를 결정한다.

---

## 4. 중복 상태를 방지하는 설계 원칙

### ① Application 내부 State를 Graph에 복제하지 않는다

`runStaffing()`이 관리하는 내부 Inspection State나 CEO Loop State를 Graph에 다시 구현하지 않는다.

Graph에는 다음 Node에 필요한 결과만 전달한다.

### ② 기존 Result 계약을 재사용한다

`StaffingRunResult`에 Proposal과 실패 정보가 포함돼 있다면, Graph State에 동일한 정보를 별도 필드로 반복 저장하지 않는다.

단, Node 간 전달이나 재개를 위해 일부 정보만 보존하는 편이 적절하다면 필요한 값만 선택할 수 있다.

### ③ Graph State를 승인 근거로 사용하지 않는다

예를 들어 Checkpoint에 다음 정보가 저장됐다고 가정한다.

```
scopeId: "scope-123"
next: "specialist-decisions"
```

이 정보는 Workflow의 진행 위치와 Scope 참조를 나타낼 뿐이다.

Specialist 실행 전에는 반드시 Governance Ledger를 통해 현재 Scope의 유효성, Context 접근 권한, Budget 등을 다시 검증해야 한다.

**Checkpoint 복원은 권한 복원이 아니다.**

### ④ Checkpoint는 중복 실행 방지를 자동 보장하지 않는다

예를 들어:

```
Invocation Claim 성공
    ↓
Codex 실행 시작
    ↓
프로세스 종료
    ↓
LangGraph 재개
```

Graph가 다시 실행된다고 기존 Invocation의 실행 권한이 되살아나는 것은 아니다.

또한 새로운 Authorization을 무조건 수행하면 별도의 예산을 소비하면서 중복 실행할 위험도 있다.

따라서 Reservation, Claim, 중복 실행 방지 정책은 LangGraph와 별도로 유지해야 한다.

---

## 5. AI Workspace의 최종 설계 판단

| 데이터                                       | 소유 영역                        |
| -------------------------------------------- | -------------------------------- |
| CEO Inspection 및 Staffing Loop 내부 정보    | Staffing Application             |
| StaffingProposal의 구조와 생성 규칙          | Staffing Application             |
| 다음 Node에 전달할 Staffing 결과             | LangGraph State                  |
| Human Approval 및 Delegation 기록            | Governance Ledger                |
| ApprovedOperatingScope / Reservation / Claim | Governance Ledger                |
| Node 간 Decision 및 Comparison 결과          | LangGraph State                  |
| Workflow 실행 위치와 중단·재개               | LangGraph Runtime / Checkpointer |

**핵심 원칙**

1. 하나의 데이터가 여러 시스템에 존재하더라도 각 시스템의 목적과 신뢰 수준을 구분한다.
2. Application 내부 구현 상태를 LangGraph에 무조건 노출하지 않는다.
3. Graph State에는 실제로 전달·재개에 필요한 데이터만 저장한다.
4. Domain 모델은 기존 Application과 Contracts가 소유한다.
5. 승인 및 실행 권한의 기준은 Governance Ledger로 유지한다.
6. Checkpoint나 Graph State를 권한 증명 또는 정확히 한 번 실행의 보장으로 취급하지 않는다.

---

## 6. 최종 결론

LangGraph State와 Governance Ledger는 모두 데이터를 관리하지만, 같은 문제를 해결하는 시스템은 아니다.

**Application State는 개별 업무의 진행을 관리하고, LangGraph State는 업무 간 실행 흐름을 연결하며, Governance Ledger는 AI의 행동을 허용할 권한을 증명한다.**

따라서 LangGraph를 도입한다는 이유로 기존 Application State나 Ledger를 대체해서는 안 된다.

중요한 것은 새로운 State를 많이 만드는 것이 아니라, **어떤 데이터를 어디에서 소유하고, 어떤 데이터만 다음 단계에 전달해야 하는지 명확히 정의하는 것**이다.

이 기준을 유지하면 기존 Application의 독립성과 Governance의 신뢰 경계를 보존하면서 LangGraph의 상태·분기·재개 기능을 활용할 수 있다.
