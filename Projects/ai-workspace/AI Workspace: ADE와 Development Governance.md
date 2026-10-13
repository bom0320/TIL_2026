# AI Workspace: ADE와 Development Governance

## 요약

> `ai-workspace`는 개발자와 AI Coding Agent가 실제 Repository에서 안전하고 효율적으로 협업할 수 있도록 작업 방식, 권한, 검증 규칙을 코드화하는 **Agent Development Environment(ADE) / Development Governance Harness**다.

## 1. 가장 중요한 원칙

`ai-workspace`를 만들면서 모든 AI Agent 기능과 인프라를 직접 구현하는 것이 목표가 아니다.

이미 잘 만들어진 Agent Framework, Library, Engine, Tool이 해결하는 범용 문제는 최대한 활용하고, `ai-workspace`는 **Software Development에 특화된 규칙과 작업 환경**을 직접 설계한다.

```
범용 Agent 기능
→ 기존 기술 활용

개발 Agent에 특화된 규칙
→ ai-workspace가 직접 설계
```

즉 프로젝트의 기본 원칙은 다음과 같다.

> **Build domain semantics. Adopt commodity infrastructure.**

특별할 이유가 없는 범용 기술은 다시 만들지 않는다.

반대로 개발 Agent가 실제 Repository에서 **어떤 규칙 아래 일해야 하는지**에 대한 부분은 `ai-workspace`가 직접 정의한다.

---

# 2. 왜 이 프로젝트를 만드는가

`ai-workspace`는 새로운 AI Coding Agent를 만드는 프로젝트가 아니다.

Cursor, Codex, Claude Code, Devin 같은 Coding Agent는 이미 코드를 작성하고 수정하는 능력이 강하다.

문제는:

> **그 Agent가 내가 원하는 방식으로 일하는가?**

이다.

개발자와 팀마다 일하는 방식은 다르다.

```
어떤 파일까지 수정해도 되는가
어떤 검증을 반드시 통과해야 하는가
원본 Repository를 언제 수정해도 되는가
어떤 작업은 사람의 승인이 필요한가
commit / push는 어떤 조건에서 허용되는가
Agent의 결과를 어떤 Evidence로 신뢰할 것인가
여러 Agent의 판단이 다르면 어떻게 처리할 것인가
```

기성 Coding Agent는 코딩은 잘하지만, 이런 **개발자나 팀 고유의 작업 방식과 개발 정책**까지 대신 정해주지는 않는다.

그래서 `ai-workspace`는:

> **개발자와 팀의 작업 방식, 권한, 검증 규칙을 코드화하여 AI Agent가 안전하고 효율적으로 일할 수 있는 Development Harness를 만드는 프로젝트다.**

쉽게 표현하면:

```
Coding Agent
= 실제 일을 하는 주체

AI Workspace
= Agent가 일하는 작업장 + 규칙 + 안전장치
```

---

# 3. 개발자의 성향과 팀의 규칙을 환경으로 만든다

`ai-workspace`는 단순히 AI에게 코딩 명령을 전달하는 프로그램이 아니다.

개발자가 AI와 **어떻게 일하고 싶은지**를 시스템으로 표현하는 환경에 가깝다.

예를 들어 어떤 개발자는:

```
원본 Repository를 AI가 바로 수정하는 것이 싫다.
→ isolated Git worktree에서만 작업시킨다.

AI가 "테스트 통과했습니다"라고 말하는 것을 그대로 믿고 싶지 않다.
→ Harness가 직접 verification command를 실행한다.

AI가 요청하지 않은 파일까지 건드리는 것이 싫다.
→ allowedPaths / forbiddenPaths를 둔다.

AI가 마음대로 commit / push하는 것이 싫다.
→ Human Approval을 별도로 둔다.

작은 README 수정에 1분씩 Planning하는 것이 싫다.
→ Fast Path를 둔다.
```

즉 이런 기능들은 단순 구현 디테일이 아니라:

> **“나는 AI Coding Agent와 이런 방식으로 일하고 싶다.”**

라는 개발자의 작업 방식을 코드로 만든 것이다.

---

# 4. 현재 구현도 이미 이 방향에 있다

## TaskContract

자연어 Goal을 Agent에게 바로 실행시키지 않는다.

먼저:

```
무엇을 해야 하는가
어디까지 수정할 수 있는가
무엇을 하면 안 되는가
어떤 조건을 만족해야 하는가
어떻게 검증할 것인가
```

를 명시적인 `TaskContract`로 만든다.

즉:

```
Human Goal
↓
TaskContract
↓
Human Approval
↓
Agent Execution
```

이라는 실행 경계를 만든다.

---

## Git Worktree Isolation

Agent가 원본 Repository에서 바로 작업하지 않는다.

```
Original Repository
↓
Detached Worktree
↓
Agent 작업
↓
Scope / Verification
↓
Human Review
↓
Apply Approval
↓
Original Repository 반영
```

Agent가 실수하더라도 원본 작업 공간에 바로 영향을 주지 않는다.

---

## Scope Enforcement

```
allowedPaths
forbiddenPaths
```

를 이용해 Agent가 작업할 수 있는 범위를 정의한다.

중요한 점은:

```
Agent:
"요청한 것만 수정했습니다."

≠

실제로 요청한 파일만 변경됨
```

이라는 것이다.

Harness가 실제 변경 경로를 확인한다.

---

## Verification

Agent의 자기 보고를 그대로 성공으로 인정하지 않는다.

```
Agent Claim
"테스트를 통과했습니다."

↓
Harness

pnpm test
pnpm typecheck
git diff --check

↓
Actual Evidence
```

즉 핵심 원칙은:

> **Evidence over Claims**

이다.

---

## Human Approval

사람의 승인을 하나의 `approve`로 뭉치지 않는다.

```
Task Approval
≠ Change Review
≠ Apply Approval
≠ Commit Approval
≠ Push Approval
```

각 단계는 서로 다른 권한이다.

Agent에게 자율성을 주더라도 최종적인 위험한 행동의 권한은 Human이 가진다.

---

## Fast Path

모든 작업에 똑같은 AI Planning을 사용하지 않는다.

```
단순하고 명확한 작업
→ Fast Path

복잡하거나 모호한 작업
→ Planner
```

즉 Harness는 안전성만을 위한 것이 아니라 **개발 생산성과 실행 비용도 고려한다.**

---

# 5. 개인 작업 환경에서 시작해 팀의 환경으로 확장할 수 있다

처음에는:

```
내가 AI Coding Agent와
더 편하고 안전하게 일하기 위한 환경
```

으로 시작할 수 있다.

하지만 개발 규칙을 일반화하면:

```
개인 개발자의 작업 방식
↓
팀의 Development Policy
↓
조직의 AI Agent Governance
```

까지 확장할 수 있다.

예를 들어 A팀은:

```
main 직접 push 금지
PR 전 test + lint 필수
Reviewer는 read-only
dependency 변경은 Human Approval 필수
```

를 원할 수 있다.

B팀은:

```
작은 UI 수정은 자동 적용 가능
API 변경은 Reviewer 확인 필수
security 변경은 별도 Specialist 필요
production push는 Human만 가능
```

을 원할 수 있다.

즉:

> **같은 Coding Agent를 사용하더라도 Harness의 정책은 팀마다 달라질 수 있다.**

`ai-workspace`는 바로 이 차이를 표현하고 강제하는 환경이다.

---

# 6. 그렇다고 모든 것을 직접 만들지는 않는다

이 프로젝트가 중요한 이유는 **모든 걸 직접 만들어서가 아니다.**

오히려 반대다.

```
범용 Agent 기능
→ 기존 기술 활용

개발 Agent에 특화된 규칙
→ ai-workspace가 직접 설계
```

이 경계를 제대로 나누는 것이 중요하다.

---

# 7. 기존 Framework / Library / Engine / Tool이 담당하는 영역

## Agent Framework

Agent의 실행 흐름을 관리한다.

예:

```
LangGraph
CrewAI
Microsoft Agent Framework
```

주로 해결하는 문제:

```
Agent 실행
routing
workflow
parallel execution
handoff
state transition
multi-agent coordination
```

---

## Library / SDK

특정 기능을 코드에서 가져다 사용한다.

예:

```
OpenAI Agents SDK
Zod
Pydantic
```

주로 해결하는 문제:

```
schema validation
structured output
tool calling
guardrails
HITL primitive
```

---

## Infrastructure / Engine

시스템 레벨 문제를 해결한다.

예:

```
OPA
Temporal
Restate
OpenTelemetry
```

주로 해결하는 문제:

```
policy evaluation
authorization
checkpoint / resume
durable execution
observability
```

---

## Tool

개발자가 별도로 사용하는 작업 환경이다.

예:

```
Git
tmux
Orca
Docker
```

예를 들어:

```
tmux
= terminal/process environment

Orca
= Agent Development Environment

AI Workspace
= Development Governance Harness
```

서로 담당하는 레이어가 다르다.

---

# 8. 범용 Governance 기능도 기존 기술을 활용한다

Governance라고 해서 전부 직접 만들어야 하는 것도 아니다.

예를 들어 다음은 범용 문제다.

### Guardrails / Validation

```
Input 검사
Output 검사
Tool call 검사
위험한 행동 차단
```

후보:

```
OpenAI Agents SDK 등
```

---

### Tracing / Observability

```
Agent run
Tool call
handoff
guardrail result
execution trace
```

후보:

```
OpenAI Agents SDK
LangSmith
OpenTelemetry
```

---

### Policy / Authorization

```
이 Agent가
이 행동을 할 권한이 있는가?
```

후보:

```
OPA
```

---

### Durable Execution / Resume

```
프로세스 장애
장시간 Human Approval 대기
checkpoint
resume
```

후보:

```
Temporal
Restate
```

따라서 다음 같은 기능은 특별한 이유가 없다면 직접 만들지 않는다.

```
guardrail engine
generic tracing backend
generic policy engine
approval pause / resume infrastructure
checkpoint / restart runtime
generic scheduler
generic workflow engine
generic message transport
generic agent memory
```

---

# 9. AI Workspace가 직접 설계해야 하는 영역

반대로 다음은 `ai-workspace`가 직접 정의해야 할 영역이다.

```
TaskContract

Repository execution boundary

Repository consistency rules

Context policy

Specialist authority policy

Decision schema

Evidence semantics

Verification semantics

Scope semantics

Conflict / Agreement / Trade-off model

Human approval / override / escalation semantics

Decision traceability

Evaluation criteria
```

왜냐하면 이것들은 단순 기술 문제가 아니라 **Software Development라는 도메인에서 어떤 규칙을 적용할 것인가**의 문제이기 때문이다.

예를 들어:

```
"Reviewer는 repository write 권한을 가지지 않는다."

"같은 판단 비교 단위의 Specialist는
같은 commit SHA를 기준으로 판단해야 한다."

"Agent의 done 자기 보고는 Evidence로 인정하지 않는다."

"Decision은 실제 Repository Evidence까지 추적 가능해야 한다."

"Conflict는 단순히 의견이 다르다는 뜻이 아니라
서로 양립할 수 없는 Recommendation /
Evidence / Trade-off가 존재한다는 의미다."

"최종 Recommendation은
Decision → Evidence → Verification까지
추적 가능해야 한다."
```

이 규칙의 **내용**을 LangGraph나 Temporal, OPA가 대신 만들어주지는 않는다.

---

# 10. 기존 기술과 AI Workspace의 관계

예를 들어:

```
AI Workspace Domain Rule

"Reviewer는 repository write 금지"

↓
OPA

allow / deny

↓
Agent Runtime

Tool call 차단
```

여기서:

```
OPA
= 정책을 평가하는 엔진

AI Workspace
= 어떤 정책을 적용할지 정의
```

한다.

Human Approval 역시:

```
AI Workspace Rule

"위험한 Delivery는 Human Approval 필요"

↓
Agent Framework / SDK

interrupt / pause

↓
Human

Approve / Reject
```

와 같이 동작할 수 있다.

즉 외부 기술은 **규칙을 실행하는 메커니즘**이고, `ai-workspace`는 **어떤 규칙을 적용할지 정의하는 시스템**이다.

---

# 11. Multi-Agent도 목적 자체는 아니다

향후 여러 Specialist를 사용하더라도 목표는:

```
Frontend Agent
UX Agent
Reviewer Agent
CEO Agent
```

처럼 Agent 수를 늘리는 것이 아니다.

중요한 것은 각 판단을 다음처럼 다루는 것이다.

```
어떤 Repository 상태를 봤는가

어떤 Context를 받았는가

어떤 Authority를 가졌는가

무슨 Recommendation을 했는가

어떤 Evidence를 근거로 사용했는가

어떤 Risk와 Uncertainty가 있는가

다른 Specialist와 무엇이 충돌하는가

무슨 Trade-off가 존재하는가

Human은 무엇을 결정해야 하는가
```

예를 들어:

```
Same Repository Snapshot
        │
        ├─ Frontend Decision
        │   ├─ recommendation
        │   ├─ evidence
        │   ├─ risk
        │   └─ uncertainty
        │
        ├─ UX Decision
        │   ├─ recommendation
        │   ├─ evidence
        │   ├─ risk
        │   └─ uncertainty
        │
        └─ Reviewer Decision
            ├─ recommendation
            ├─ evidence
            ├─ risk
            └─ uncertainty

        ↓

Agreement / Conflict / Trade-off

        ↓

Verification

        ↓

Explainable Recommendation

        ↓

Human Decision
```

이 구조에서 중요한 것은 Agent를 병렬로 실행하는 기술 자체가 아니다.

중요한 것은 **Agent들의 판단을 신뢰 가능하게 비교하고 Human이 결정할 수 있도록 만드는 것**이다.

---

# 12. 앞으로의 설계 원칙

새로운 기능을 만들려고 할 때 가장 먼저 다음을 묻는다.

```
이 기능은 Development Governance에
특화된 규칙인가?

YES
→ ai-workspace가 직접 설계
```

아니라면:

```
이 기능은 범용 Agent Runtime /
Infrastructure 문제인가?

YES
→ 기존 Framework / Library / Tool을 먼저 조사
```

그래서 다음과 같은 개념도 무조건 직접 만들지 않는다.

```
DecisionRun
DecisionRound
Coordinator
CEO Agent
Shared State
Recovery
Scheduler
```

먼저 판단한다.

```
이 개념이 Development Governance에서
정말 고유한 의미를 가지는가?

OR

기존 Agent Framework의
run / session / state / step / workflow를
이름만 바꿔 다시 만들고 있는가?
```

전자인 경우에만 Domain Model로 만든다.

---

# 13. 외부 기술 도입 원칙

기존 기술을 활용한다고 해서 모든 Framework와 Library를 한 번에 넣는 것도 아니다.

또 하나의 원칙은:

> **Adopt when the problem exists, not because the tool exists.**

예를 들어:

```
현재 durable execution 문제가 없음
→ Temporal 도입 안 함

Policy가 몇 개의 단순 조건문으로 충분함
→ OPA 도입 안 함

Multi-Agent workflow가 아직 단순함
→ LangGraph 도입 안 함

Tracing 요구가 커짐
→ OpenTelemetry / LangSmith 검토
```

필요한 문제가 생겼을 때 기존 기술을 평가하고 도입한다.

---

# 14. AI Workspace의 위치

전체적으로 보면 다음과 같다.

```
Human
  ↓
AI Workspace
────────────────────────
Development Governance

TaskContract
Repository Boundary
Authority
Context Policy
Decision
Evidence
Verification
Conflict / Trade-off
Human Approval / Escalation
Evaluation
────────────────────────
  ↓
Agent Runtime / Framework
────────────────────────
Codex
LangGraph
OpenAI Agents SDK
etc.
────────────────────────
  ↓
Infrastructure / Tools
────────────────────────
Git Worktree
OPA
Temporal / Restate
OpenTelemetry
MCP
tmux
etc.
────────────────────────
  ↓
Repository
```

모든 기술을 반드시 사용하는 것이 아니라, 필요에 따라 교체하거나 추가할 수 있다.

`ai-workspace`의 핵심은 특정 Framework가 아니다.

> **개발 Agent에게 적용되는 Governance Policy와 작업 방식이 핵심이다.**

---

# 최종 정의

> **AI Workspace는 새로운 AI Coding Agent나 범용 Multi-Agent Framework를 만드는 프로젝트가 아니다.**
>
> **개발자와 팀의 작업 방식, 권한, 검증 규칙을 코드화하여 기존 AI Coding Agent가 실제 Repository에서 안전하고 효율적으로 일할 수 있도록 만드는 Development Governance Harness다.**
>
> **범용 Agent 기능과 Infrastructure는 기존 기술을 활용하고, Software Development에 특화된 규칙은 AI Workspace가 직접 설계한다.**

가장 짧게 정리하면:

```
범용 Agent 기능
→ 기존 기술 활용

개발 Agent에 특화된 규칙
→ ai-workspace가 직접 설계
```

그리고 이 프로젝트가 궁극적으로 만드는 것은:

> **AI Agent 자체가 아니라, AI Agent와 내가 원하는 방식으로 일할 수 있는 개발 환경이다.**
