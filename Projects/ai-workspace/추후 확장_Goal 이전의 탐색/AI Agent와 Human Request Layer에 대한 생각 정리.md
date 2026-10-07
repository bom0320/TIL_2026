# AI Agent와 Human Request Layer에 대한 생각 정리

## 1. 소프트웨어 개발자의 AI 활용 역량

AI 활용 역량은 단순히 좋은 프롬프트를 작성하거나 좋은 모델을 사용하는 것만을 의미하지 않는다.

실제 개발에서는 AI가 실수할 수 있다는 것을 전제로 두고, **AI가 안전하게 작업할 수 있는 환경을 설계하는 능력**이 중요하다.

예를 들어:

- Git Worktree를 이용해 작업 환경을 격리한다.
- `allowedPaths` 등으로 수정 가능한 범위를 제한한다.
- AI가 작업을 완료했다고 말하는 것만 믿지 않는다.
- 실제 Test / Verification 결과를 Evidence로 수집한다.
- 여러 Agent를 사용할 경우 각각 필요한 Context만 전달한다.

핵심은 AI의 비결정성을 프롬프트만으로 해결하려 하지 않고, **환경과 시스템으로 통제하는 것**이다.

---

# 2. AI Agent는 무엇인가

일반적인 LLM이 입력에 대해 결과를 생성하는 모델이라면, Agent는 그 모델이 실제 환경에서 행동할 수 있도록 만든 시스템에 가깝다.

Agent는 보통 다음과 같은 방식으로 동작한다.

```
Reason
↓
Tool Use
↓
Result 확인
↓
다시 Reason
↓
다음 행동
```

예를 들어 코딩 Agent는:

- Repository를 탐색하고
- 파일을 읽고
- 코드를 수정하고
- Terminal 명령을 실행하고
- Test를 돌리고
- 실패하면 다시 수정한다.

즉 한 번의 답변으로 끝나는 것이 아니라 **계획 → 행동 → 관찰 → 수정**을 반복하면서 작업한다.

Agent가 이런 구조를 가지는 이유 중 하나는 LLM의 비결정적인 결과를 실제 도구와 검증을 통해 보완하기 위해서다.

---

# 3. 그래서 Goal이 중요하다

Agent는 목표를 달성하기 위한 **How**는 어느 정도 스스로 결정할 수 있다.

하지만 **무엇을 해야 하는가(What)** 는 명확해야 한다.

예를 들어:

```
"모바일 프로젝트 캐러셀의 애니메이션 떨림을 개선한다."
```

처럼 Goal이 주어지면 Agent가 Repository를 탐색하면서 어떤 파일을 읽고 어떻게 수정할지 판단할 수 있다.

Goal이 중요한 이유는 Agent에게:

- 작업 방향
- 탐색 기준
- 종료 조건

을 제공하기 때문이다.

---

# 4. 그런데 항상 Goal이 명확한 것은 아니다

여기서 의문이 생겼다.

실제로 개발하면서 AI에게 하는 요청은 항상:

```
"이 기능을 구현해줘."
"이 버그를 고쳐줘."
```

같은 실행 요청만 있는 것이 아니다.

오히려 다음과 같은 요청도 많다.

```
"여기 구조 좀 이상하지 않아?"

"이 컴포넌트 리팩토링해야 할 것 같은데 어떻게 생각해?"

"이 코드에서 문제될 만한 부분 있어?"

"성능이 안 좋은 것 같은데 어디부터 봐야 할까?"

"이 PR 이 정도면 충분해?"

"이 코드 왜 이렇게 작성된 거야?"

"A 방식이랑 B 방식 중 뭐가 나아?"

"이 Repository에서 개선할 만한 곳 찾아봐."
```

이런 요청을 바로 `GoalSpec`으로 변환하면 문제가 생긴다.

예를 들어:

```
Human:
"이거 리팩토링 필요하지 않을까?"
```

라는 말을 AI가:

```
GoalSpec:
"이 컴포넌트를 리팩토링한다."
```

라고 해석하면 안 된다.

Human은 실제로:

```
"정말 리팩토링할 필요가 있는지 판단해줘."
```

라고 한 것일 수도 있기 때문이다.

따라서:

> 판단 요청 ≠ 실행 요청

이고,

> Human의 모든 요청이 Goal은 아니다.

라는 결론이 나온다.

---

# 5. Human Request Layer

그래서 장기적으로는 `GoalSpec`보다 위에 **Human Request를 먼저 해석하는 Layer**가 필요하다.

```
Human Request
      ↓
Intent Analysis
      ↓
 ┌───────────────┐
 Execute
 Investigate
 Review
 Explain
 Decide
 Discover
 └───────────────┘
```

`GoalSpec`은 Human의 모든 입력을 표현하는 것이 아니라,

> 실제로 실행하기로 결정된 목표

를 표현하는 Contract로 둔다.

---

# 6. Request Intent 종류

## Execute

Human이 이미 원하는 결과를 알고 있고 실제 수정을 요청한 경우.

```
"이 컴포넌트를 분리해줘."
"로그인 버그를 수정해줘."
"모바일 애니메이션을 개선해줘."
```

흐름:

```
Human Request
↓
Execute
↓
GoalSpec
↓
Planning
↓
TaskContract
↓
Execution
↓
Verification
```

현재 v1에서 우선 다루려는 영역이다.

---

## Investigate

바로 수정하기보다 **분석과 판단을 먼저 요청하는 경우**.

```
"여기 리팩토링 필요해 보여?"
"이 구조 문제 있어?"
"왜 렌더링이 느린 것 같아?"
```

흐름:

```
Human Request
↓
Investigate
↓
Repository Investigation
↓
Findings
↓
Recommendation
↓
Human Decision
```

이후 Human이:

```
"그럼 그렇게 수정해줘."
```

라고 승인했을 때 비로소:

```
GoalSpec
↓
Planning
↓
TaskContract
↓
Execution
```

으로 넘어간다.

핵심은:

> Investigation이 자동으로 Execution으로 이어지면 안 된다.

는 것이다.

---

## Review

이미 만들어진 결과물이 기준을 만족하는지 검사한다.

```
"이 PR 괜찮아?"
"이 구현 이 정도면 충분해?"
"이 변경사항 제대로 된 거 맞아?"
```

```
Artifact / Diff / PR
+
Review Criteria
↓
Review
↓
Findings
↓
Pass / Concern / Fail
```

차이는:

```
Investigate
= 무엇이 문제인지 찾는다.

Review
= 이미 만들어진 결과가 기준을 만족하는지 검사한다.
```

이다.

---

## Explain

수정이나 평가가 목적이 아니라 **Human이 코드를 이해하고 싶을 때**다.

```
"이 함수 왜 이렇게 작성된 거야?"
"이 worktree 로직이 어떻게 동작해?"
"왜 여기서 rmSync를 호출하는 거야?"
```

```
Human Request
↓
Explain
↓
Relevant Context
↓
Explanation
```

이 경우 결과는 코드 변경이 아니라 **Human의 이해**다.

---

## Decide

여러 선택지 사이의 Trade-off를 판단해 달라는 경우.

```
"model 폴더에 둘까 planning 바로 아래에 둘까?"
"Context를 하나로 합칠까 분리할까?"
"A와 B 중 어떤 방식이 나아?"
```

```
Human Request
↓
Decide
↓
Options
↓
Trade-off Analysis
↓
Recommendation
↓
Human Decision
```

단순한 AI의 취향이 아니라:

```
선택지
+
Repository Context
+
Constraints
+
Trade-off
```

를 근거로 판단해야 한다.

---

## Discover

Human이 구체적인 문제조차 지정하지 않고 **AI에게 개선할 문제 자체를 찾게 하는 경우**다.

```
"이 Repository에서 개선할 곳 찾아봐."
"구조적으로 위험한 부분 있어?"
"지금 가장 먼저 손볼 만한 곳이 어디야?"
```

```
Repository
↓
Discovery
↓
Candidate Issues
↓
Impact / Risk / Cost 평가
↓
Prioritized Opportunities
```

즉:

```
Goal-based
= Human이 무엇을 할지 결정한다.

Discovery-based
= AI가 무엇을 할 가치가 있는지도 탐색한다.
```

라는 차이가 있다.

---

# 7. Intent와 Skill은 다른 개념이다

`Intent`는:

> Human이 무엇을 원하는가

이고,

`Skill`은:

> 그 요청을 어떤 전문적인 방법으로 해결할 것인가

이다.

예를 들어:

```
Human:
"이 컴포넌트 리팩토링 필요해?"
```

라면:

```
Human Request
↓
Investigate Intent
↓
refactoring-review Skill
↓
Repository Analysis
↓
Findings
↓
Recommendation
```

처럼 처리할 수 있다.

예상할 수 있는 Skill은:

```
refactoring-review
architecture-review
performance-investigation
accessibility-review
dependency-audit
test-quality-review
```

등이다.

따라서:

```
Human Request
= Human이 무엇을 원하는가

Intent
= 요청을 어떤 작업으로 해석했는가

Skill
= 그 요청을 어떤 방법으로 해결할 것인가

GoalSpec
= 실행하기로 확정된 목표

TaskContract
= 실행 가능한 작업 계약
```

으로 역할을 분리할 수 있다.

---

# 8. Context Target

Intent뿐 아니라 **Human이 무엇을 대상으로 요청했는가**도 알아야 한다.

예:

```
"이 파일 봐봐."
"이 함수만 설명해줘."
"이 PR 리뷰해줘."
"Repository 전체 구조를 봐줘."
```

장기적으로는:

```
type ContextTarget =
  | { type: "repository" }
  | { type: "directory"; path: string }
  | { type: "file"; path: string }
  | { type: "diff" }
  | { type: "pull-request"; id: string };
```

같은 구조를 둘 수 있다.

이를 통해 항상 Repository 전체를 읽는 것이 아니라 **요청과 관련된 범위만 조사할 수 있다.**

---

# 9. Authority Level

Human이 요청했다고 해서 AI에게 모든 권한이 주어지는 것도 아니다.

요청에 따라 AI가 할 수 있는 범위를 분리할 필요가 있다.

```
observe
= 읽고 분석

propose
= 해결책 제안

plan
= TaskContract 생성

execute
= 코드 수정

commit
= Git commit

merge
= Merge
```

예를 들어:

```
"여기 구조 이상하지 않아?"
```

는:

```
observe
+
propose
```

정도면 충분하다.

반대로:

```
"그 방향으로 수정해줘."
```

라고 했을 때 `execute` 권한까지 올라간다.

핵심 원칙은:

> Question ≠ Permission to modify

이다.

---

# 10. AI의 판단에는 Evidence가 필요하다

AI가 단순히:

```
"이 구조는 좋지 않습니다."
```

라고 말하는 것은 충분하지 않다.

Repository를 실제로 확인하고:

```
- 하나의 파일에 API 호출, 상태 관리, UI rendering이 함께 존재한다.
- 동일한 에러 처리 로직이 여러 곳에 중복되어 있다.
- 해당 컴포넌트가 여러 도메인의 책임을 동시에 가진다.
```

와 같은 Evidence를 제시한 뒤:

```
따라서 책임 분리를 고려할 가치가 있다.
```

라는 결론을 내리는 편이 낫다.

즉:

```
Claim
↓
Evidence
↓
Conclusion
```

구조를 가져가야 한다.

특히:

- Investigate
- Review
- Decide
- Discover

에서는 Evidence가 중요하다.

장기적으로 판단 결과를:

```
finding
evidence
reasoning
recommendation
```

형태로 구조화할 수도 있다.

---

# 11. Investigation과 Execution을 분리한다

Investigation은 기본적으로 **Read-only**로 둔다.

### Investigation에서 허용

```
read files
list files
inspect repository
inspect package.json
inspect dependencies
inspect git metadata
analyze architecture
inspect diff
```

### Investigation에서 금지

```
write files
delete files
git checkout
git commit
git reset
dependency install
code modification
Worker execution
```

즉:

```
Investigation
= 관찰 / 분석 / 판단

Execution
= Repository State 변경
```

으로 구분한다.

---

# 12. Human Approval이 실행의 경계다

예를 들어:

```
Human:
"이 컴포넌트 리팩토링 해야 할까?"

↓
Investigation

↓
Recommendation

"animation state와 rendering 책임이 섞여 있습니다.
전체 구조를 다시 만드는 것보다
animation controller만 먼저 분리하는 것이 적절해 보입니다."

↓
Human:
"응 그렇게 해줘."

↓
GoalSpec
↓
Planning
↓
TaskContract
↓
Execution
```

즉:

```
판단
↓
제안
↓
Human 승인
↓
실행
```

이라는 구조를 가져간다.

이게 현재 프로젝트에서 생각하고 있는 **Human-in-the-loop** 원칙과 연결된다.

---

# 13. 최종적으로 생각하는 장기 구조

```
Human Request
      │
      ▼
Intent Analysis
      │
      ├── Execute ───────────────→ GoalSpec
      │                              │
      │                              ▼
      │                           Planning
      │                              │
      │                              ▼
      │                         TaskContract
      │                              │
      │                              ▼
      │                           Execution
      │
      ├── Investigate ───────→ Findings
      │                           │
      │                           ▼
      │                     Recommendation
      │                           │
      │                           ▼
      │                     Human Approval
      │                           │
      │                           └────→ GoalSpec
      │
      ├── Review ───────────→ Review Result
      │
      ├── Explain ──────────→ Explanation
      │
      ├── Decide ───────────→ Recommendation
      │
      └── Discover ─────────→ Opportunities
                                  │
                                  ▼
                             Human Decision
                                  │
                                  └────→ GoalSpec
```

여기서 `GoalSpec`을 다시 정의하면:

> **시스템의 최초 입력이 아니라, Human과 AI가 실행하기로 합의한 시점의 실행 목표 Contract**

라고 볼 수 있다.

---

# 14. 하지만 현재 v1에서는 여기까지 하지 않는다

Human Request Layer 전체를 지금 바로 구현하려는 것은 아니다.

현재 v1에서는 **Human이 이미 실행 Goal을 결정했다는 것을 전제로 한다.**

현재 범위:

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

즉 현재 v1의 핵심 문제는:

> **주어진 Goal을 Repository에 근거해서 어떻게 안전한 TaskContract로 변환할 것인가?**

이다.

Human Request / Intent Analysis는 장기적으로 위에 추가할 Layer이고, 지금은 우선:

```
Goal
↓
Repository 분석
↓
Planning
↓
TaskContract
↓
Execution
```

이라는 **Goal → Execution Loop를 먼저 완성한다.**

---

# 핵심 결론

이번 고민에서 가장 중요한 것은 `GoalSpec`의 위치가 명확해졌다는 것이다.

처음에는:

```
Human
↓
GoalSpec
```

이라고 단순하게 생각했지만,

실제 Human의 요청을 생각해보면:

```
Human Request
↓
Intent Analysis
↓
판단 / 조사 / 설명 / 리뷰 / 선택 / 실행
```

과정이 먼저 필요할 수 있다.

그리고 **실제로 Repository를 변경하기로 결정된 순간에만 `GoalSpec`으로 전환한다.**

따라서:

> Human Request ≠ GoalSpec

> 판단 요청 ≠ 실행 요청

> Question ≠ Permission to modify

> Investigation ≠ Execution

이며,

현재 v1에서는 이 전체 문제를 한꺼번에 해결하지 않고 **명확한 Execute Goal이 이미 존재한다는 전제하에 Goal → TaskContract → Execution 구조부터 완성한다.**
