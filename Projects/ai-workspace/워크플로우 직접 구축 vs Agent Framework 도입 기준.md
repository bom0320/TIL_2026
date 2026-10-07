# 워크플로우 직접 구축 vs Agent Framework 도입 기준

## 1. 현재 내가 만들고 있는 것

이 프로젝트의 목적은 단순히 AI Agent에게 Repository를 연결하고 코드를 수정하게 만드는 것이 아니다.

이미 Coding Agent들은 다음과 같은 작업을 상당히 잘 수행한다.

```
Repository 읽기
↓
관련 파일 탐색
↓
코드 수정
↓
Test / Typecheck
↓
결과 반환
```

따라서 프로젝트의 핵심은 Agent 자체의 코딩 능력을 다시 만드는 것이 아니다.

내가 만들고자 하는 것은:

> **AI Agent들이 적절한 역할·Context·권한 안에서 판단하고, 필요한 경우 여러 전문 Agent가 협업하며, 검증 가능한 근거를 기반으로 Human이 이해할 수 있는 결론까지 만들어내도록 하는 개발 환경**

이다.

즉:

```
Better Model
```

보다

```
Better Environment for Agents
```

를 만드는 것이 목표다.

---

# 2. 직접 만들어야 하는 것과 빌려 써도 되는 것

앞으로 가장 중요하게 구분해야 할 것은:

> **이 기능이 프로젝트의 차별점인가, 아니면 범용 Workflow Infrastructure인가?**

이다.

## 직접 설계해야 하는 핵심 Domain

다음은 이 프로젝트 자체의 철학과 차별점이므로 직접 설계해야 한다.

```
GoalSpec

TaskContract

RepositoryContext

RelevantContext

Evidence Semantics

Agent Role

Context Routing

Staffing Policy

Authority / Scope

Conflict Model

Discussion Protocol

Human Approval Policy

Decision Synthesis

Executive Report
```

즉 다음 질문에 대한 답이다.

```
누가 작업해야 하는가?

왜 그 Agent가 필요한가?

어떤 Context를 줘야 하는가?

어디까지 행동할 수 있는가?

어떤 의견 차이를 Conflict라고 볼 것인가?

언제 Discussion이 필요한가?

무엇을 Evidence로 인정할 것인가?

언제 Human에게 Escalation할 것인가?

여러 의견을 어떻게 하나의 Decision으로 합칠 것인가?
```

이것들은 LangGraph 같은 프레임워크가 대신 결정해주지 않는다.

---

# 3. 직접 만들 필요가 없을 수도 있는 Infrastructure

반대로 다음 영역은 프로젝트 자체의 차별점보다는 Workflow Engine의 범용 기능에 가깝다.

```
Checkpoint

State Persistence

Pause / Resume

Retry Scheduling

Graph Traversal

Parallel Execution

Fan-out / Fan-in

Streaming

Long-running Workflow State

Cancellation

Workflow Resume
```

이런 기능을 직접 구현하는 것이 프로젝트의 목적은 아니다.

따라서 나중에 이 영역의 구현 비용이 커진다면 검증된 Agent Framework를 사용하는 것이 합리적이다.

---

# 4. 왜 Agent Workflow가 복잡해지는가

가장 근본적인 이유는 LLM이 일반적인 함수처럼 완전히 결정론적으로 동작하지 않기 때문이다.

예를 들어 일반 프로그램은:

```
Input A
→ 항상 Output B
```

를 기대하기 쉽지만,

LLM은:

```
같은 Goal
↓
조금 다른 Plan

같은 Prompt
↓
조금 다른 Structured Output

같은 Context
↓
다른 판단
```

이 나올 수 있다.

따라서 Workflow가 커질수록 단순한 `if / else` 이상의 상태 관리와 예외 처리가 필요해진다.

---

# 5. 복잡성이 발생하는 주요 영역

## 5.1 State Management / Rollback

예를 들어 하나의 Goal이:

```
Task A
↓
Task B
↓
Task C
```

로 계획되었다고 하자.

Task A는 성공했지만 Task B에서 Verification이 실패할 수 있다.

그러면 시스템은 판단해야 한다.

```
Task B만 Retry할 것인가?

↓

Task A 결과 자체가 문제였는가?

↓

Planning을 다시 수행해야 하는가?

↓

어느 Evidence까지 유효한가?

↓

어떤 TaskContract는 폐기해야 하는가?
```

여기서 중요한 것은:

```
Git Repository Rollback
```

과

```
Workflow State Rollback
```

을 구분하는 것이다.

Repository 상태는 Git / Worktree / Base Commit을 이용하면 상당 부분 관리할 수 있다.

하지만 더 어려운 것은:

```
어느 판단까지 유효한가
어느 Evidence를 폐기할 것인가
기존 Agent의 의견을 유지할 것인가
Human Approval 상태는 유효한가
Replanning이 필요한가
```

같은 **논리적 Workflow State**다.

이런 상태 관리가 복잡해질 경우 Checkpoint 기반 Framework의 가치가 커진다.

---

# 6. Structured Output 문제

v1부터 실제로 만나게 될 가능성이 높은 문제다.

Planner가:

```
Goal
↓
TaskContract
```

를 생성할 때 LLM 출력은 항상 완벽하지 않을 수 있다.

예:

```
JSON 문법 오류

필수 field 누락

잘못된 verification 형태

존재하지 않는 allowedPath

Repository에 없는 script 사용
```

따라서 단순히:

```
JSON.parse()
```

로 끝낼 수 없다.

최소한:

```
LLM Output
↓
Syntax Validation
↓
Schema Validation
↓
Repository-grounded Validation
↓
Policy Validation
```

이 필요하다.

예:

```
Candidate TaskContract
↓
Zod safeParse
↓
FAIL
↓
Validation Error
↓
Planner에게 Feedback
↓
Retry
```

정도는 직접 구현할 수 있다.

하지만 이런 `Validation → Feedback → Retry` 패턴이 여러 Agent와 여러 Schema에서 반복되기 시작하면 Framework나 공통 Runtime abstraction을 고려할 수 있다.

---

# 7. Structured Output이 모든 문제를 해결하지는 않는다

Schema를 사용하면 JSON 구조 자체는 강하게 통제할 수 있다.

하지만 다음은 여전히 가능하다.

```
{
  "allowedPaths": [
    "src/components/DoesNotExist.tsx"
  ]
}
```

문법적으로도 맞고 Schema에도 맞지만 Repository에는 없는 파일이다.

따라서:

```
Schema Validation
```

만으로는 부족하다.

반드시:

```
Schema Validation
+
Repository Grounding
+
Policy Validation
```

이 함께 있어야 한다.

이 때문에 현재 만들고 있는 `RepositoryContext`와 `TaskContract`의 의미가 있다.

---

# 8. Human-in-the-Loop가 복잡해지는 시점

현재 CLI처럼 실행이 짧게 끝나는 단계에서는 복잡하지 않다.

하지만 장기적으로:

```
CEO
↓
Staffing Proposal
↓
Human Approval 필요
↓
SYSTEM PAUSE
```

가 생길 수 있다.

Human이 10초 뒤 승인할 수도 있지만 몇 시간 또는 며칠 뒤에 돌아올 수도 있다.

그러면 시스템은:

```
현재 Workflow State 저장

↓

Process 종료 가능

↓

Human Approval

↓

이전 State 복구

↓

정확한 지점부터 Resume
```

해야 한다.

이런 **Durable Execution**이 실제로 필요해지면 Pause / Resume / Checkpoint를 직접 만드는 비용이 커진다.

이때는 LangGraph 같은 Framework를 검토할 가치가 높다.

---

# 9. Multi-Agent에서 발생하는 복잡성

처음 Agent 두 개를 실행하는 것 자체는 어렵지 않다.

예:

```
const [frontend, ux] = await Promise.all([
  runFrontendAgent(),
  runUxAgent(),
]);
```

따라서:

```
Agent 2개
=
바로 복잡한 Framework 필요
```

는 아니다.

초기 Multi-Agent는 단순하게:

```
Frontend Agent
↓
FrontendFinding

UX Agent
↓
UXFinding

둘 다 완료
↓
Synthesis
```

로 시작할 수 있다.

복잡성이 커지는 것은 다음과 같은 요구가 생길 때다.

```
Agent 실행 중간에 서로 상태 공유

한 Agent 실패 시 다른 Agent Cancellation

부분 Retry

Streaming

Pause / Resume

Dynamic Agent 추가

Discussion Loop

Long-running State

Concurrent Modification
```

이 시점부터 Concurrency와 State Management가 본격적인 문제가 된다.

---

# 10. Discussion도 처음부터 복잡하게 만들 필요는 없다

처음부터 자유로운 Agent 대화를 구현하면:

```
Agent A
↓
Agent B 반박
↓
Agent A 재반박
↓
Agent C 추가 의견
↓
다시 A
↓
...
```

와 같이 끝없는 토론과 Token 낭비가 생길 수 있다.

따라서 Discussion은 자유 채팅보다 **구조화된 협업**이어야 한다.

예:

```
Independent Analysis
↓
CEO / Moderator

↓

Agreement
Disagreement
Conflict Point
Unknown

↓

Conflict Point만 다시 Agent에게 전달

↓

Response

↓

CEO Decision
```

즉:

> 모든 Agent가 서로 모든 Context를 공유하며 자유롭게 대화하도록 만들 필요는 없다.

오히려 **계층적인 Discussion 구조**가 비용과 Context Pollution을 통제하기 쉽다.

---

# 11. Right Agent × Right Context

Multi-Agent의 핵심은 Agent 수가 아니다.

```
Agent 많음
≠
좋은 시스템
```

오히려:

```
Right Agent
×
Right Context
```

가 중요하다.

예:

```
UX Agent

PRODUCT
DESIGN
User Flow
관련 UI
관련 Component
```

정도만 필요할 수 있다.

반면 Frontend Agent는:

```
TaskContract
Component
Hook
API
State
Test
Architecture Reference
```

가 필요할 수 있다.

모든 Agent에게 Repository 전체를 주는 것은:

```
Context 증가
↓
Token 증가
↓
Noise 증가
↓
판단 품질 저하 가능성
```

으로 이어질 수 있다.

---

# 12. Framework를 너무 일찍 쓰면 생기는 문제

Framework가 좋다고 해서 지금 바로 도입할 필요는 없다.

현재는 아직:

```
어떤 State가 필요한가?

어떤 Context가 필요한가?

어떤 Flow가 실제로 반복되는가?

어디가 정말 복잡해지는가?
```

를 발견하는 단계다.

이때 Framework를 먼저 넣으면:

```
내 문제를 이해하는 것
+
Framework의 추상화를 이해하는 것
```

을 동시에 해야 한다.

또한 내가 설계해야 할 문제를 Framework가 제공하는 개념에 억지로 맞출 가능성도 있다.

따라서 현재는 작은 Workflow를 직접 구현하면서 실제 abstraction point를 발견하는 것이 좋다.

---

# 13. 현재 v0 / v1에서의 판단

현재 단계에서는 Framework 없이 직접 구현한다.

## v0

```
Reliable Worker

TaskContract
Safe Workspace
Scope Enforcement
Verification
Evidence
Run Artifacts
```

목표:

> Agent 하나가 안전하고 반복 가능하게 작업할 수 있도록 한다.

---

## v1

```
Human
↓
GoalSpec
↓
RepositoryContext
↓
RelevantContext
↓
Planner
↓
TaskContract
↓
Execution
```

목표:

> Human이 TaskContract를 직접 작성하지 않아도 Repository에 근거한 Task를 생성할 수 있도록 한다.

현재 정도의 순차적이고 결정론적인 Workflow는 직접 구현하는 편이 더 단순하다.

---

# 14. 첫 Multi-Agent 단계도 직접 구현 가능

v1 다음에 바로 거대한 AI 조직을 만들 필요는 없다.

첫 실험은:

```
Goal
↓
Relevant Context

     ┌────────────┐
     │            │
     ▼            ▼

Frontend       UX
Agent          Agent

     │            │
     └──────┬─────┘
            ▼
         Synthesis
            ↓
         Decision
```

정도면 충분하다.

이 과정에서 실제로:

```
Single Agent

vs

Independent Multi-Agent

vs

Multi-Agent + Discussion
```

의 결과 차이를 비교할 수 있다.

---

# 15. Framework 도입 Trigger

Framework 도입은 버전 번호로 결정하지 않는다.

```
v2니까 LangGraph
```

처럼 정하지 않는다.

다음과 같은 증상이 실제 코드에서 나타날 때 검토한다.

## Trigger 1 — Branch 증가

```
if QA fail
→ retry

if architecture issue
→ architect

if scope increase
→ human approval

if conflict
→ discussion
```

처럼 Workflow branch가 급격히 늘어난다.

---

## Trigger 2 — Pause / Resume

```
Human Approval
↓
몇 시간 / 며칠 뒤 Resume
```

가 필요해진다.

---

## Trigger 3 — Persistent State

프로세스가 종료되어도:

```
현재 Plan
Evidence
Agent Result
Conflict
Approval State
```

를 복구해야 한다.

---

## Trigger 4 — Retry 정책 중복

여러 곳에서:

```
retry
validation retry
agent retry
verification retry
replanning
```

가 반복된다.

---

## Trigger 5 — Parallel Control

```
parallel execution
cancellation
partial failure
fan-out
fan-in
```

을 자주 처리해야 한다.

---

## Trigger 6 — Orchestration 코드 폭증

가장 중요한 기준이다.

```
Workflow Engine을 관리하는 코드
>
Agent Collaboration Domain 코드
```

가 되기 시작한다면,

> 이것은 내가 연구하고 싶은 문제가 아니라 Infrastructure 문제다.

라고 판단하고 Framework를 검토한다.

---

# 16. LangGraph 등을 검토할 때 기대할 역할

나중에 LangGraph 같은 것을 사용한다면 역할은 다음과 같다.

```
LangGraph

Checkpoint
Persistence
Graph Runtime
Pause / Resume
Streaming
Routing
Parallel Execution
```

그 위에:

```
ai-workspace

TaskContract
RepositoryContext
Evidence
Context Routing Policy
Staffing Policy
Conflict Model
Discussion Protocol
Human Governance
Decision Synthesis
```

를 올린다.

즉:

```
Framework
= Workflow 실행 Infrastructure

ai-workspace
= Agent 조직 운영 Domain
```

이다.

Framework를 사용한다고 프로젝트의 차별점이 사라지는 것이 아니다.

---

# 17. 가장 중요한 원칙

프로젝트의 목적은:

> 새로운 Workflow Engine을 만드는 것

이 아니다.

내가 연구하고 싶은 것은:

> **어떤 방식으로 AI Agent를 조직하고, 적절한 Context와 권한을 부여하며, 협업·토론·검증하게 해야 실제 개발 과정에서 더 좋은 결과를 만드는가?**

이다.

따라서 프로젝트 본질이 아닌 Infrastructure 구현에 지나치게 많은 시간을 쓰기 시작하면 이미 해결된 도구를 가져오는 것이 맞다.

---

# 18. 앞으로의 방향

현재:

```
v0
Reliable Execution

↓

v1
Reliable Planning

↓

Independent Multi-Agent Analysis

↓

Structured Discussion

↓

Coordination

↓

AI Organization
```

순서로 확장한다.

각 단계를 실제로 경험하면서 다음 단계에서 필요한 abstraction을 결정한다.

---

# 최종 정리

## 지금 직접 만든다

```
Goal
Repository Grounding
Planning
TaskContract
Evidence
Safe Execution
```

## 나중에도 직접 설계한다

```
Agent Role
Context Routing
Staffing
Authority
Conflict
Discussion
Decision
Human Governance
```

## 필요해지면 Framework에 맡긴다

```
Checkpoint
Persistence
Pause / Resume
Graph Runtime
Retry Infrastructure
Parallel Control
Streaming
```

---

## 판단 기준

> **프로젝트 고유의 Agent 운영 규칙인가?**
>
> → 직접 설계한다.

> **다른 Agent 시스템에서도 똑같이 필요한 범용 실행 Infrastructure인가?**
>
> → 복잡성이 실제로 발생하면 검증된 Framework 사용을 고려한다.

---

# 한 문장으로

> **나는 Agent Workflow Engine 자체를 새로 발명하려는 것이 아니다. AI Agent들이 실제 개발 환경에서 더 잘 판단하고, 적절한 Context 안에서 협업하며, 검증 가능한 근거를 가지고 Human에게 설명 가능한 결론을 만들도록 하는 운영 규칙과 환경을 설계하고 있다. 범용 Workflow Infrastructure가 병목이 되는 순간에는 검증된 Framework를 활용한다.**
