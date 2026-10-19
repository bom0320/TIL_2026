# v4 — Runtime Framework의 역할과 책임 경계

## 1. Framework 도입을 검토하는 이유

AI Workspace는 **자율적으로 판단하고 협업하는 AI 조직과, 그 조직의 행동을 통제하는 Repository Harness를 결합한 개발 환경**이다.

v0~v3에서는 안전한 실행, Planning, Specialist의 독립 판단, Decision Comparison을 각각 구현했다.

v4에서는 이를 하나의 조직 운영 흐름으로 연결한다.

```
Human Goal
    ↓
CEO Agent — Staffing / Routing 판단
    ↓
Harness — Authority / Budget 검증
    ↓
Human — Staffing Approval
    ↓
Harness — 승인된 Specialist 실행
    ↓
Independent Decision (v2)
    ↓
Structured Comparison (v3)
    ↓
CEO Agent — Synthesis / Recommendation
    ↓
Harness — Validation / Escalation
    ↓
Human — Approve / Override / Reject
    ↓
TaskContract / Repository Execution (v1 / v0)
    ↓
CEO — Executive Report
    ↓
Human
```

이 과정에서는 단순한 LLM 호출을 넘어 동적 Routing, Human Approval, 실행 상태 관리, 실패 처리 등의 문제가 발생한다.

따라서 Framework의 도입 목적은 **AI의 판단을 대신하거나 Governance를 위임하는 것이 아니라, 이러한 조직 Workflow를 안정적으로 실행하고 관리하는 것**이다.

---

## 2. 책임 경계 — CEO / Harness / Runtime / Human

### CEO Agent — Autonomous Decision Maker

**무엇을 해야 하는가?**

- Goal과 업무 목적 분석
- 필요한 Specialist 선택 및 Staffing Proposal 생성
- Specialist별 업무와 Context 제안
- 실행 결과와 Decision Comparison 종합
- Recommendation / Unresolved Issue / Escalation 제안
- 최종 Executive Report 생성

CEO는 조직 운영을 위한 판단의 주체다. 그러나 그 판단만으로 권한이 확대되거나 실제 작업이 실행되지는 않는다.

### Harness — Governance & Control Owner

**그 행동을 허용할 수 있는가?**

- Authority / Scope / Budget 정책 정의
- Staffing Proposal 및 Agent 실행 요청 검증
- Human Approval 강제
- Context Isolation 및 접근 범위 제한
- Schema / Reference / Evidence Validation
- TaskContract 및 Repository Execution 통제
- Verification과 Human 최종 권한 보존

Harness는 CEO 대신 어떤 제품 방향이 최선인지 결정하는 것이 아니라, CEO의 판단이 **허용된 조건에서만 행동으로 이어지도록 강제**한다.

### Runtime — Execution Mechanism

**허용된 행동을 실제로 어떻게 수행하고 관리하는가?**

- Agent / Model 호출
- Workflow 상태 관리 및 조건부 실행
- Human 승인 대기와 재개
- 실행 Checkpoint 및 Recovery
- Tool 실행과 기술적 접근 제한
- Timeout / Failure Handling
- 실행 이벤트와 Trace 수집

Runtime이 제공하는 기능을 사용하더라도 실행 정책과 Domain의 의미는 Harness가 소유한다.

### Human — Final Authority

Human은 Goal과 주요 제약을 정의하고, Staffing 및 중요한 판단을 승인하거나 수정하며, 최종 코드 반영·배포 권한을 유지한다.

**핵심 원칙**

> CEO는 무엇을 할지 판단한다.
>
> Harness는 그 행동이 허용되는지 검증한다.
>
> Runtime은 허용된 행동을 실제로 실행하고 관리한다.
>
> Human은 중요한 결정에 대한 최종 권한을 가진다.

---

## 3. 실제 경계는 어떻게 구분하는가?

예를 들어 CEO가 UX Specialist의 추가 투입이 필요하다고 판단했다고 하자.

```
CEO
→ UX Specialist 추가 필요
→ 이유 / Context / Budget 제안

Harness
→ 등록된 Specialist인가?
→ 요청 Context가 허용 범위인가?
→ Budget 제한을 초과하는가?
→ 추가 Human Approval이 필요한가?

Human
→ 필요한 경우 Staffing 승인

Runtime
→ 승인된 UX Specialist 호출
→ Context 전달
→ 실행 상태 / Timeout 관리
→ 결과 반환

Harness
→ Output / Evidence Validation
→ 다음 단계 진행 여부 결정
```

여기서 **전문가가 필요하다는 판단은 CEO의 책임**이고, **그 전문가를 실제로 호출할 수 있는지 결정하는 규칙은 Harness의 책임**이다.

Runtime은 그 결과를 실행한다.

### 책임을 구분하는 기준

**Domain / Governance**

> 이 규칙이 변경되면 AI Workspace의 판단 의미나 허용되는 행동이 달라지는가?

그렇다면 Harness가 소유해야 한다.

**Runtime**

> 같은 Domain 규칙을 유지하면서 실행 기술을 다른 것으로 교체할 수 있는가?

그렇다면 Runtime의 책임일 가능성이 높다.

단, Workflow의 단계 순서나 승인 조건처럼 실행과 정책이 만나는 부분에서는 **Harness가 정책을 정의하고 Runtime이 이를 수행**하는 형태로 나눠야 한다.

---

## 4. Runtime은 두 가지 책임으로 구분할 수 있다

### Agent Execution Runtime

개별 Agent를 실행한다.

```
SpecialistContext
    ↓
Codex Runtime
    ↓
AgentDecision
```

주요 책임:

- 모델 호출
- Prompt / Context 전달
- Structured Output 수신
- 개별 호출의 Timeout / Error Handling
- 허용된 Tool 실행

현재 Codex CLI가 이 역할을 담당하고 있다.

### Workflow Runtime

여러 Agent와 Human의 활동을 하나의 업무 흐름으로 연결한다.

```
CEO Staffing
    ↓
Human Approval
    ↓
Specialist Execution
    ↓
Comparison
    ↓
CEO Synthesis
    ↓
Human Governance
    ↓
Repository Execution
```

주요 책임:

- Workflow State
- Conditional Routing
- Agent 실행 단계 연결
- Human Interrupt / Resume
- Checkpoint / Recovery
- Workflow Trace

**LangGraph.js는 주로 이 Workflow Runtime의 후보로 검토한다.**

따라서 LangGraph를 도입한다고 해서 기존 Codex Runtime이나 v0~v3 Application 기능을 모두 대체할 필요는 없다.

---

## 5. v4 요구사항과 Framework의 책임

| 영역              | CEO / Harness가 소유                                        | Runtime이 지원                         |
| ----------------- | ----------------------------------------------------------- | -------------------------------------- |
| Staffing          | 전문가 선택, Proposal 정책, 승인 조건                       | Agent 호출 및 Routing                  |
| Structured I/O    | Domain Schema, Validation                                   | 입력·출력 전달, 구조화 응답            |
| Context Isolation | 허용 Context 및 권한 결정                                   | 선택된 Context 전달 및 실행 제한       |
| Provenance        | Recommendation → Comparison → Decision → Evidence 추적 규칙 | 입력·출력 및 참조 정보 전달            |
| HITL              | 승인 대상, 권한, Approve / Override / Reject 의미           | 실행 중단·대기·재개                    |
| Guardrails        | Authority / Scope / Budget 정책과 위반 처리                 | Tool 접근 제한, 실행 차단 기능         |
| Tracing           | 필요한 감사·관찰 정보 정의                                  | 실행 이벤트, 지연 시간, 호출 기록 수집 |
| Failure Handling  | 실패 이후 진행·차단·Escalation 정책                         | Timeout, 예외 전달, 상태 복구          |

여기서 특히 다음을 혼동하지 않는다.

**Provenance ≠ Tracing**

- Provenance: 어떤 Recommendation이 어떤 Decision과 Evidence를 근거로 만들어졌는가?
- Tracing: 어떤 Agent가 언제 실행되어 어떤 입력과 출력을 주고받았는가?

**Human Governance ≠ HITL Runtime**

- Human Governance: 무엇을 누가 승인해야 하며 그 승인의 의미는 무엇인가?
- HITL Runtime: 승인 대기 중 실행을 중단하고 이후 안전하게 재개하는 방법

---

## 6. Framework 도입 순서

Framework를 먼저 선택한 뒤 v4를 그 구조에 맞춰 설계하지 않는다.

```
v4 목표와 책임 경계 확정
    ↓
Runtime Requirements 도출
    ↓
Framework 후보 검토
    ↓
작은 Runtime Spike
    ↓
Current Runtime과 비교
    ↓
Adopt / Partial Adopt / Keep Current
    ↓
v4 구현
```

Framework Evaluation은 별도의 제품 버전이 아니라 **v4 구현 방식을 결정하기 위한 Architecture Checkpoint**다.

### 1순위 후보 — LangGraph.js

선정 이유:

- TypeScript 환경에서 기존 Harness 기능 재사용 가능
- State 기반 Workflow 구성
- 조건부 Routing
- Human Interrupt / Resume
- Checkpoint 및 실행 상태 복구 지원
- CEO의 자율적 판단과 Harness의 실행 정책을 별도 단계로 연결하기 쉬움

단, LangGraph 자체가 Harness의 Authority나 Context Isolation 정책을 자동으로 보장하지는 않는다.

### 기타 후보

- **CrewAI:** 역할 기반 조직 및 Manager/Delegation 모델에 강점이 있지만 Python과 TypeScript 사이의 통합 비용이 발생한다.
- **OpenAI Agents SDK:** Agent 실행, Handoff, Tool Approval, Tracing에 강점이 있다.
- **Current Runtime:** 현재 Codex CLI와 TypeScript 함수 기반 Orchestration을 유지하는 방식이다.

현재 결정은 **LangGraph.js 우선 평가**이지, 정식 채택 확정이 아니다.

---

## 7. Runtime Spike에서 검증할 것

제안 브랜치: `spike/langgraph-runtime`

전체 v4를 먼저 구현하지 않고, Framework가 실제로 필요한 문제를 해결하는지만 확인한다.

```
Mock CEO Staffing
    ↓
Harness Proposal Validation
    ↓
Human Staffing Approval
    ├─ Reject → Stop
    └─ Approve
          ↓
    기존 Decision / Comparison 호출
          ↓
    Mock CEO Synthesis
          ↓
    Human Governance
```

검증 항목:

1. Human 승인 전에는 Specialist가 실행되지 않는가?
2. 승인에 따라 Workflow가 정확히 분기하는가?
3. 프로세스 종료 후에도 영속적 Checkpoint를 이용해 승인 대기 상태를 복구할 수 있는가?
4. 기존 `runDecision()` / `compareDecisions()` 및 Domain 모델을 재사용할 수 있는가?
5. Specialist별 Context Isolation을 유지할 수 있는가?
6. 재개 과정에서 Agent나 부작용 있는 작업이 중복 실행되지 않도록 설계할 수 있는가?
7. 기존 함수 기반 Orchestration보다 상태·실행 관리가 실질적으로 단순해지는가?

이 실험은 CEO의 의사결정 품질이 아니라 **Workflow Runtime의 적합성**을 검증한다.

---

## 8. 최종 판단 기준

**Adopt**

Framework가 Governance 경계를 보존하면서 실제 Workflow 복잡도를 줄일 경우 도입한다.

**Partial Adopt**

필요한 Runtime 기능에만 제한적으로 적용한다. Agent Execution Runtime과 Domain Layer 전체를 교체하지 않는다.

**Keep Current**

현재 함수 기반 Orchestration으로 충분하거나 Framework 도입이 불필요한 복잡도를 증가시키면 기존 구조를 유지한다.

---

## 최종 설계 원칙

> **AI 조직에는 자율성을 부여하되, 실행 권한은 Harness가 통제한다.**
>
> CEO Agent는 Staffing, Coordination, Synthesis를 판단하고, Harness는 Authority, Context, Budget, Human Approval과 Verification을 강제한다.
>
> Runtime Framework는 이 조직의 실행·상태·중단·재개를 지원하는 기술적 수단이며, AI Workspace의 Domain과 Governance를 대체하지 않는다.

현재는 LangGraph.js를 1순위 Runtime 후보로 검증하고, 결과에 따라 v4 도입 여부를 결정한다.
