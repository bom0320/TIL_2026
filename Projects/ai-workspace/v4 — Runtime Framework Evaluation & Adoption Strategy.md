# v4 — Runtime Framework Evaluation & Adoption Strategy

## 1. 배경 및 목적

AI Workspace의 v4 목표는 **CEO Agent가 자율적으로 AI 조직을 운영하고, Harness가 그 의사결정의 실행을 통제하는 환경을 구축하는 것**이다.

기존 v0~v3에서 구현한 기능을 재작성하는 것이 아니라 하나의 조직 운영 흐름으로 연결한다.

- **v0:** Safe Execution — 안전한 Repository 실행 및 검증
- **v1:** Goal-to-Task Planning — Goal을 TaskContract로 변환
- **v2:** Independent Decision — Specialist의 독립적인 판단
- **v3:** Structured Comparison — 판단의 합의·충돌·Trade-off 분석
- **v4:** CEO Governance — 조직 구성, 의사결정, Human 승인, Repository 실행 및 최종 보고

v4에서는 Agent Staffing, 동적 Routing, Human Approval, 실행 상태 관리 등의 요구가 발생하므로 외부 Runtime Framework의 도입 가치를 평가한다.

**목적은 프레임워크를 사용하는 것이 아니라, 조직 운영과 실행 통제에 필요한 기능을 적절한 책임 계층에 배치하는 것이다.**

---

## 2. v4 목표 Architecture

```
Human Goal
    ↓
CEO Agent
├─ Intent / Task Analysis
├─ Specialist Staffing
├─ Context / Budget Proposal
└─ Staffing Proposal
    ↓
Harness Governance
├─ Authority Validation
├─ Budget / Scope Validation
└─ Proposal Validation
    ↓
Human Staffing Approval
    ↓
Harness Orchestration
├─ Approved Specialist Execution
├─ Context Isolation
├─ State Management
└─ Evidence Validation
    ↓
Independent AgentDecision[] (v2)
    ↓
Structured Comparison (v3)
    ↓
CEO Synthesis
├─ Recommendation
├─ Rationale / Provenance
├─ Unresolved Issues
└─ Human Escalation
    ↓
Harness Governance
    ↓
Human Decision
├─ Approve
├─ Override
└─ Reject
    ↓
Approved Decision
    ↓
TaskContract / Planning (v1)
    ↓
Repository Execution (v0)
├─ Worker
├─ Scope Enforcement
├─ Verification
└─ Evidence
    ↓
CEO Executive Report
    ↓
Human
```

이 전체 흐름을 **하나의 v4 범위**로 유지한다.

---

## 3. CEO Agent / Harness / Runtime의 책임 경계

### CEO Agent — Autonomous Decision Maker

CEO는 AI 조직 운영에 필요한 판단을 자율적으로 수행한다.

주요 책임:

- Human Goal 분석
- 필요한 Specialist 선정
- Context / Budget / Staffing 제안
- Specialist 결과 종합
- Recommendation과 Trade-off 판단
- 추가 전문성 필요 여부 판단
- Human Escalation 제안
- 실행 결과 해석 및 Executive Reporting

CEO는 판단과 제안의 주체지만, 제한 없이 Agent나 Tool을 실행할 권한을 갖지는 않는다.

### Harness — Governance & Control Owner

Harness는 CEO의 판단이 실제 행동으로 이어질 수 있는지 확인하고 실행 경계를 강제한다.

주요 책임:

- Authority / Scope / Budget 정책
- Staffing Proposal 검증
- Human Approval Enforcement
- Specialist Context Isolation
- Structured Output / Reference Validation
- TaskContract 검증
- Repository Execution Governance
- Verification / Evidence Provenance
- 최종 Human 권한 보존

**CEO의 판단은 Harness의 검증과 필요한 Human 승인 이후에만 실행될 수 있다.**

### Runtime Framework — Execution Mechanism

Runtime은 Harness가 허용한 작업을 실제로 진행하는 기술적 수단이다.

도입 대상 책임:

- Workflow State Management
- Conditional Routing
- Agent Execution Lifecycle
- Human Interrupt / Resume
- Checkpoint / Recovery
- Execution Tracing

반면 `AgentDecision`, `ComparisonResult`, `SynthesisResult`, `GovernanceDecision`의 의미와 검증 규칙은 Framework가 아닌 Harness가 소유한다.

---

## 4. Framework 후보 비교

| 후보                  | 강점                                                           | 고려 사항                                                  |
| --------------------- | -------------------------------------------------------------- | ---------------------------------------------------------- |
| **LangGraph.js**      | Stateful Workflow, 조건부 분기, Interrupt / Resume, Checkpoint | Graph / State 설계 복잡도                                  |
| **CrewAI**            | 역할 기반 AI 조직, Manager, Delegation, Crews / Flows          | Python 기반으로 기존 TypeScript Harness와 통합 비용 발생   |
| **OpenAI Agents SDK** | Agent-as-Tool, Handoff, Tool Approval, Tracing                 | Workflow 전체의 상태·승인 경계는 별도 설계 필요            |
| **Current Runtime**   | 기존 Codex CLI와 Harness 재사용, 의존성 최소화                 | 실행 상태·재개·동적 라우팅 기능을 직접 구현해야 할 수 있음 |

### 검토 결과

**LangGraph.js를 v4의 1순위 Framework Evaluation 후보로 선정한다.**

선정 이유:

1. 기존 TypeScript 코드와 자연스럽게 통합할 수 있다.
2. 기존 v0~v3 application capability를 재사용하기 쉽다.
3. CEO의 동적 판단과 Harness의 승인·검증 흐름을 명시적으로 연결할 수 있다.
4. Human 승인 대기와 실행 재개를 지원한다.
5. Framework 내부에 Domain / Governance 정책을 종속시키지 않고 구성할 수 있다.

CrewAI 역시 CEO 중심 조직 모델에 적합하지만, 현재 프로젝트에서는 언어 간 통합 비용과 Domain 책임 중복 가능성을 추가로 고려해야 한다.

---

## 5. LangGraph.js 도입 방향

LangGraph는 **Harness의 Domain Layer를 대체하지 않고 Workflow Runtime 역할로만 활용**한다.

```
AI Workspace
│
├─ Domain / Governance
│   ├─ Authority
│   ├─ StaffingProposal
│   ├─ AgentDecision
│   ├─ ComparisonResult
│   ├─ SynthesisResult
│   └─ GovernanceDecision
│
├─ CEO / Specialist Execution
│   └─ Codex Runtime (현재 방식 재사용 가능)
│
├─ Workflow Orchestration
│   └─ LangGraph.js (도입 검토)
│       ├─ State
│       ├─ Routing
│       ├─ Interrupt / Resume
│       └─ Checkpoints
│
└─ Repository Harness
    ├─ Planning
    ├─ Execution
    ├─ Verification
    └─ Evidence
```

### 설계 원칙

**1. 기존 기능을 재구현하지 않는다.**

`runDecision()`, `compareDecisions()`, `planTask()`, `executeTask()` 등의 기존 기능을 재사용한다. 다만 CEO의 동적 Staffing이나 승인된 Decision의 Planning 전달에 필요한 확장은 별도로 설계한다.

**2. Workflow State와 Domain State를 구분한다.**

LangGraph의 Checkpoint는 실행 재개를 위한 상태다. 이것이 자동으로 검증 가능한 Domain provenance나 영구적인 Governance 기록이 되는 것은 아니다.

**3. Context Isolation을 유지한다.**

Workflow가 전체 State를 관리하더라도 Specialist에게는 Harness가 허용한 Context만 전달한다.

**4. Human Approval을 우회하지 않는다.**

Framework가 실행을 재개하더라도 Staffing, Task Approval, Apply, Commit, Push 등 기존 권한 조건은 Harness가 다시 확인한다.

**5. 부작용 있는 실행은 재개에 안전해야 한다.**

Checkpoint 재실행 시 Worker, Apply, Commit, Push 등이 중복 수행되지 않도록 별도 보호 장치를 설계한다.

---

## 6. 도입 전 Runtime Spike

LangGraph를 바로 정식 도입하지 않고, 격리된 실험에서 기존 방식과 비교한다.

**제안 브랜치:** `spike/langgraph-runtime`

### 검증할 최소 흐름

```
Human Goal
    ↓
Mock CEO Staffing
    ↓
Harness Proposal Validation
    ↓
Human Staffing Approval
    ├─ Reject → Stop
    └─ Approve
          ↓
    Existing Decision / Comparison
          ↓
    Mock CEO Synthesis
          ↓
    Human Governance
        ├─ Approve
        ├─ Override
        └─ Reject
```

### 핵심 검증 항목

- Human 승인 전 Specialist 실행을 차단할 수 있는가?
- 프로세스 종료 후에도 영속적 Checkpoint를 통해 승인 대기 상태를 복구할 수 있는가?
- 조건부 Routing과 Human Override가 정상 동작하는가?
- 기존 application API와 Domain 모델을 유지할 수 있는가?
- 각 Specialist의 Context Isolation을 보존할 수 있는가?
- Workflow 재개 시 불필요한 Agent 재실행과 중복 부작용을 방지할 수 있는가?
- 기존 함수 기반 Orchestration 대비 복잡도를 실질적으로 줄이는가?

Spike에서는 새로운 CEO 판단 품질까지 평가하지 않는다. **Framework의 실행·상태 관리 능력을 검증하는 데 집중한다.**

---

## 7. Framework 채택 기준

### Adopt

LangGraph가 기존 Harness의 Domain / Governance 경계를 유지하면서 Routing, HITL, State Recovery 등을 유의미하게 단순화한다면 채택한다.

### Partial Adopt

특정 Runtime 기능만 가치가 있다면 해당 부분에 제한적으로 적용한다. 별도 LLM Runtime이나 Domain Model까지 함께 교체하지 않는다.

### Keep Current

추가 Graph / State 관리가 오히려 복잡도를 증가시키거나 현재 TypeScript 함수 기반 Orchestration으로 충분하다면 기존 구조를 유지한다.

**Framework 도입 여부는 기능 목록이 아니라, 실제 구현 복잡도와 책임 경계를 기준으로 결정한다.**

---

## 8. 현재 결정 사항

| 항목                       | 결정                                               |
| -------------------------- | -------------------------------------------------- |
| v4 목표                    | CEO Governance / AI Organization Orchestration     |
| CEO 역할                   | 자율적 Staffing, Synthesis, Coordination           |
| Harness 역할               | Authority, Approval, Validation, Execution Control |
| Framework 1순위            | LangGraph.js                                       |
| 도입 상태                  | Evaluation Pending                                 |
| 검증 방법                  | Isolated Runtime Spike                             |
| 기존 v0~v3                 | 재사용 원칙                                        |
| Domain ownership           | Harness 유지                                       |
| CEO / Specialist 모델 실행 | 우선 기존 Codex 방식 고려                          |
| 최종 채택                  | Spike 결과에 따라 결정                             |

## 결론

> **CEO Agent는 자율적으로 AI 조직을 운영하고, Harness는 그 의사결정이 허용된 권한과 검증 규칙 안에서 실행되도록 통제한다. LangGraph.js는 이러한 조직 Workflow의 상태·분기·중단·재개를 지원하는 Runtime 후보로 평가한다.**

현재 시점에서는 LangGraph.js의 도입을 확정하지 않는다.

**다음 작업은 `spike/langgraph-runtime`에서 최소 Workflow를 검증하고, 그 결과를 Architecture Decision으로 기록하는 것이다.**
