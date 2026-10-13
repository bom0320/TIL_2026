# Future Architecture Reference — Framework & Library Adoption

## 목적

`ai-workspace`는 모든 Agent Runtime과 인프라를 직접 구현하는 프로젝트가 아니다.

프로젝트의 기본 원칙은 다음과 같다.

```
범용 Agent 기능
→ 기존 기술 활용

개발 Agent에 특화된 규칙
→ ai-workspace가 직접 설계
```

> **Build domain semantics. Adopt commodity infrastructure.**

그리고 기술이 존재한다는 이유만으로 먼저 도입하지 않는다.

> **Adopt when the problem exists, not because the tool exists.**

---

# 1. AI Workspace와 외부 기술의 관계

`ai-workspace`의 핵심은 특정 AI Framework가 아니다.

AI Coding Agent가 실제 Repository에서 **개발자와 팀이 원하는 방식으로 일할 수 있는 ADE(Agent Development Environment)와 Development Governance Harness**를 만드는 것이 목적이다.

```
AI Workspace
│
├─ 개발 환경 / Governance
│  ├─ TaskContract
│  ├─ Repository Boundary
│  ├─ Context Policy
│  ├─ Authority
│  ├─ Evidence
│  ├─ Verification
│  ├─ Decision
│  ├─ Conflict / Trade-off
│  └─ Human Approval
│
└─ 필요한 실행 기능 위임
   ├─ Agent Runtime
   ├─ Workflow / Orchestration
   ├─ Policy Engine
   ├─ Durable Execution
   ├─ Observability
   └─ Tool Protocol
```

외부 Framework와 Library는 **AI Workspace를 대체하는 것이 아니라, AI Workspace가 필요로 하는 범용 실행 기능을 제공하는 부품**으로 본다.

---

# 2. 현재 구조는 유지한다

현재 구현된 다음 요소는 계속 AI Workspace가 소유한다.

```
Goal
↓
Planning
↓
TaskContract
↓
Human Approval
↓
Isolated Git Worktree
↓
Worker
↓
Scope Enforcement
↓
Verification
↓
Evidence
↓
Human Review
↓
Apply / Commit / Push
```

특히 유지할 핵심은:

### TaskContract

Human의 자연어 Goal을 즉시 Agent 실행 권한으로 바꾸지 않고 명시적인 실행 계약으로 변환한다.

### Git Worktree Isolation

Agent의 작업을 원본 Repository와 분리한다.

### Scope Enforcement

실제 changed paths를 확인하여 Agent가 허용 범위를 벗어났는지 검사한다.

### Verification

Agent의 자기 보고보다 실제 command execution 결과를 사용한다.

### Evidence

diff, changed paths, verification result 등 실제 관찰 결과를 보존한다.

### Human Governance

```
Task Approval
≠ Change Review
≠ Apply Approval
≠ Commit Approval
≠ Push Approval
```

중요한 권한을 하나의 `approve`로 합치지 않는다.

### Fast Path

단순하고 명확한 작업은 불필요한 AI Planning을 거치지 않는다.

---

# 3. Framework를 넣는다고 기존 구조를 갈아엎지 않는다

미래에 Framework가 필요해지더라도:

```
AI Workspace
↓
External Runtime / Framework
↓
Coding Agent
```

처럼 **실행 책임을 위임하는 방향**으로 사용한다.

AI Workspace의 Domain Model이 Framework의 `run`, `step`, `session`, `state` 같은 개념에 종속되어서는 안 된다.

즉:

```
Framework 교체
→ Governance semantics 유지

Coding Agent 교체
→ Governance semantics 유지
```

가 가능해야 한다.

---

# 4. 현재 단계에서는 Framework를 먼저 넣지 않는다

v2에서는 우선 현재 Codex 기반으로 실제 vertical slice를 완성한다.

```
Human Goal
↓
RepositorySnapshot
↓
Specialist Selection

├─ Frontend Specialist
│  ├─ Scoped Context
│  ├─ Read-only Authority
│  └─ AgentDecision
│
└─ UX Specialist
   ├─ Scoped Context
   ├─ Read-only Authority
   └─ AgentDecision

↓
Decision Validation
↓
Preserve Decisions
```

v2에서 검증할 핵심은:

```
같은 Repository 상태를 보았는가?

각 Specialist에게
필요한 Context만 제공되었는가?

권한이 실제로 제한되었는가?

Specialist들이 서로의 판단에
영향받지 않았는가?

Decision이 Evidence와
Repository까지 추적 가능한가?
```

이다.

Agent를 병렬 실행하거나 복잡한 Workflow Framework를 사용하는 것은 v2의 성공 조건이 아니다.

---

# 5. v2에서 직접 만들 영역

v2에서는 다음 Development Governance 개념에 집중한다.

```
RepositorySnapshot

SpecialistDefinition
SpecialistContext
SpecialistAuthority

SpecialistInvocation

AgentDecision
EvidenceReference
```

예:

```
RepositorySnapshot
        │
        ├─ Frontend Specialist
        │       ↓
        │   AgentDecision
        │
        └─ UX Specialist
                ↓
            AgentDecision
```

각 `AgentDecision`은 최소한:

```
Recommendation
Evidence
Risk
Uncertainty
```

를 구조화한다.

---

# 6. v2에서 아직 만들지 않을 것

다음은 우선순위가 아니다.

```
CEO Agent
Coordinator Agent
Agent 토론
Consensus
Conflict Resolution
Shared Memory
Generic Agent Memory
Parallel Scheduler
Durable Execution
Automatic Retry
Decision Synthesis
```

특히:

```
DecisionRun
DecisionRound
```

도 이름부터 먼저 만들지 않는다.

실제 개발 과정에서 고유한 Development Governance 의미가 확인될 때만 Domain Model로 승격한다.

예를 들어 `DecisionRound`가:

> 동일 RepositorySnapshot을 기준으로 생성된 여러 독립 Decision의 비교 단위

라는 의미가 실제로 필요해지면 그때 도입할 수 있다.

반대로 단순히 Framework의 workflow step을 다시 표현하는 것이라면 만들지 않는다.

---

# 7. Framework 도입 시점

v2 MVP를 먼저 직접 구현한 뒤 실제 문제를 관찰한다.

예를 들어:

```
Specialist invocation 코드가 지나치게 반복됨

Context 전달 / isolation 관리가 복잡함

여러 실행 branch 관리가 어려움

Failure routing이 복잡함

Human interruption이 필요함

State transition이 복잡해짐

순차 / 병렬 실행 관리가 어려움
```

이런 문제가 실제로 발생하면 그때 Framework를 평가한다.

---

# 8. Framework Spike

실제 문제가 확인되면 작은 PoC로 비교한다.

예:

```
현재 직접 orchestration
VS
LangGraph / Agent Runtime 사용
```

비교 기준:

```
코드량

복잡도

Context isolation

Authority enforcement integration

State handling

Failure handling

Human interruption

Observability

Testability

Latency

Dependency / maintenance cost
```

Framework가 실제 문제를 더 잘 해결한다면 채택한다.

그렇지 않다면 현재 구현을 유지한다.

---

# 9. 외부 기술별 미래 역할

## Agent Runtime / Orchestration

후보:

```
LangGraph
OpenAI Agents SDK
기타 Agent Framework
```

필요할 가능성이 있는 영역:

```
Agent routing
multiple invocation
state transition
conditional workflow
parallel branch
Human interruption
```

단, Framework를 사용한다고 Agent의 reasoning이나 코딩 성공률이 자동으로 향상되는 것은 아니다.

---

## Schema Validation / Structured Output

현재 TypeScript 프로젝트에서는 기존 `Zod`를 우선 유지한다.

```
LLM Output
↓
TaskDraft / AgentDecision
↓
Zod Validation
```

Schema validation이 보장하는 것은 **구조적 유효성**이다.

```
schema valid
≠
semantically correct

schema valid
≠
good decision

schema valid
≠
hallucination 없음
```

따라서 단순히 Pydantic AI를 도입하기 위해 Python 환경을 추가할 필요는 없다.

---

## Policy / Authorization

후보:

```
OPA
```

예:

```
AI Workspace Domain Rule

"Reviewer는 repository write 금지"

↓
OPA

allow / deny

↓
Runtime

write tool call 차단
```

OPA가 정책의 **내용**을 정하는 것이 아니다.

```
정책 의미
→ AI Workspace

정책 평가
→ OPA
```

로 책임을 나눈다.

---

## Durable Execution

후보:

```
Temporal
Restate
Framework checkpoint 기능
```

현재 CLI에서는 Human이 바로 응답하므로 아직 필요성이 낮다.

향후:

```
Web UI
Slack Approval
장시간 Human 대기
Process restart
Server crash
Resume
```

가 실제 요구가 되면 평가한다.

---

## Observability

후보:

```
OpenTelemetry
LangSmith
기타 tracing backend
```

Agent run, tool call, latency 등의 범용 Trace 저장 기능은 기존 도구를 활용할 수 있다.

다만 AI Workspace의:

```
Evidence
Decision
Repository provenance
Human governance history
```

와 일반적인 Runtime Trace는 같은 개념으로 취급하지 않는다.

---

## Tool Interoperability

후보:

```
MCP
```

범용 Tool protocol을 새로 만드는 대신 필요하면 기존 표준을 활용한다.

---

## Developer Environment Tool

후보:

```
tmux
Orca
기타 ADE tooling
```

이들은 AI Workspace의 Governance Domain 자체가 아니라 개발자가 Agent process를 실행하고 관찰하는 작업 환경 도구다.

---

# 10. v3 / v4에서 Framework 필요성이 더 커질 수 있다

v2는 비교적 단순하다.

```
Specialist A 판단
Specialist B 판단
↓
저장
```

하지만 v3부터:

```
Independent Decisions
↓
Comparison
↓
Agreement
Conflict
Trade-off
Uncertainty
```

가 추가된다.

v4에서는:

```
Decisions
↓
Synthesis
↓
Recommendation
↓
Human Escalation
↓
Approve / Override / Reject
```

처럼 Workflow 상태가 많아질 수 있다.

이 시점에는:

```
workflow graph
conditional routing
parallel branch
interrupt
state persistence
```

같은 범용 Runtime 기능의 필요성이 더 명확하게 나타날 수 있다.

따라서 **Framework 도입의 본격적인 판단 시점은 v2 MVP 이후, 특히 v3/v4 설계 과정이 될 가능성이 높다.**

---

# 11. 장기 개발 순서

```
v0
Safe Execution
→ Agent 행동 통제

v1
Goal → TaskContract
→ 실행 권한 전달 전 통제

↓ 현재

v2 설계
→ Governed Independent Decisions

↓

v2 Codex 기반 MVP
→ RepositorySnapshot
→ Specialist
→ Context / Authority
→ AgentDecision

↓

실제 Runtime / Orchestration 문제 관찰

↓

Framework Spike
→ 필요하면 기존 기술 도입

↓

v3
Decision Comparison
→ Agreement / Conflict / Trade-off

↓

v4
Explainable Synthesis / Human Governance

↓

필요 시
Durable Execution / Web UI / External Approval

↓

v5
Single-Agent vs Governed Multi-Agent Evaluation
```

---

# 핵심 원칙

미래 설계에서 계속 유지해야 할 기준은 이것이다.

```
범용 Agent 기능
→ 기존 기술 활용

개발 Agent에 특화된 규칙
→ ai-workspace가 직접 설계
```

그리고 새로운 기술이나 개념을 추가하기 전에:

```
1. 이건 Development Governance의 고유한 문제인가?

YES
→ 직접 설계

NO
↓

2. 범용 Runtime / Infrastructure 문제인가?

YES
→ 기존 기술을 먼저 조사

↓

3. 지금 실제로 그 문제가 존재하는가?

NO
→ 아직 도입하지 않음
```

## 최종 방향

> **AI Workspace는 Framework를 많이 직접 구현하는 프로젝트가 아니라, 기존 Agent와 Infrastructure를 활용하면서 개발자와 팀이 원하는 방식으로 AI Agent와 일할 수 있는 ADE와 Development Governance Harness를 만드는 프로젝트다.**

따라서 미래 Framework/Library 도입은 **현재 구조를 교체하기 위한 리팩토링이 아니라, 검증된 범용 책임을 외부 기술에 위임하면서 AI Workspace의 Development Governance 영역을 더 명확하게 만드는 과정**으로 본다.
