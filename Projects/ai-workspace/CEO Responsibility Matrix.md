# CEO Responsibility Matrix

이제 가장 먼저 **CEO가 어떤 종류의 결정을 내리는 주체인지 부터 명확하게 정의**하도록 하자.

### 중요한 설계 기준

> CEO는 Harness의 명령을 전달하는 단순 Coordinator가 아니라,
> Human Goal을 달성하기 위해 필요한 판단을 독립적으로 수행하**는 AI 조직의 의사결정자**다.
>
> 단, 판단할 권한과 행동을 실행할 권한은 다르며, 모든 실행 요청은 Harness의 Governance Boundary를 통과해야 한다.

이 전제에서 출발해 CEO가 무엇을 판단하고, 무엇을 제안하며, 무엇을 직접 실행할 수 없는지 구분해야 한다.

## 1. CEO 는 어떤 존재인가?

CEO를 단순한 Workflow Coordinator로 정의하면 안 된다.

Coordinator는 정해진 절차를 진행시키는 역할에 가깝지만,
우리가 원하는 CEO는 **목표를 이해하고 상황에 따라 조직의 운영 방식을 결정하는 주체**이다.

| 구분         | Workflow Coordinator    | CEO Agent                         |
| ------------ | ----------------------- | --------------------------------- |
| Goal 분석    | 미리 정의된 단계 실행   | 목표와 제약을 해석                |
| Staffing     | 정해진 Agent 호출       | 필요한 전문가 판단                |
| Coordination | 고정된 실행 순서 관리   | 상황에 따라 실행 전략 제안        |
| Decision     | 결과 전달               | 판단을 종합해 방향 추천           |
| Failure      | 사전에 정해진 오류 분기 | 재검토·추가 조사·중단 필요성 판단 |
| Reporting    | 실행 로그 출력          | 성과·위험·한계 해석               |

여기서 CEO가 Coordinator의 모든 실행 책임까지 가져갈 필요는 없다.

CEO는 조직 운영 전략을 판단하고, LangGraph는 승인된 전략의 실행 LifeCycle을 관리하는 구조가 적절하다.

## 2. CEO Responsibility Matrix

우선 CEO의 책임을 여덟 영억으로 구분하자.

| 책임                | CEO가 판단하는 것                      | 권한 경계                       |
| ------------------- | -------------------------------------- | ------------------------------- |
| Intent Analysis     | Goal의 의도, 요구사항, 제약, 불확실성  | Human Goal을 임의 변경 불가     |
| Staffing            | 어떤 Specialist가 왜 필요한가          | 승인 없이 Agent 호출 불가       |
| Context Routing     | 각 Specialist에게 어떤 정보가 필요한가 | 허용 Context 밖 접근 불가       |
| Budget Planning     | 필요한 호출·시간·비용 예산             | 승인된 Budget 초과 불가         |
| Coordination        | 어떤 순서와 조건으로 협업할 것인가     | 승인된 Staffing 범위 내 운영    |
| Synthesis           | 어떤 접근을 추천할 것인가              | Evidence 추적 및 검증 필요      |
| Escalation          | 추가 판단이나 Human 개입이 필요한가    | 필수 승인 조건을 생략할 수 없음 |
| Executive Reporting | 무엇을 수행했고 어떤 결과가 나왔는가   | 실제 실행 결과를 변경할 수 없음 |

여기엔 중요한 구분이 있다.

> **CEO는 의사결정을 생성하는 주체이지만, 자신이 생성한 의사결정에 실행 권한까지 부여하는 주체는 아니다.**

이를 구조로 표현하면 다음과 같다.

!스크린샷 2026-10-19 오후 4.09.09.png

## 3. CEO의 권한은 세 가지로 구분하자

이 부분이 이후 `CEOAuthority` 모델의 기초가 된다.

#### A. Decision Authority — 판단할 권한

CEO 가 Goal을 분석하고 해결 방향을 선택할 수 있는 권한이다.

예를 들면 다음 행동들이 해당한다.

- Goal의 핵심 문제 분석
- 필요한 전문 분야 판단
- 여러 대안의 우선순위 결정
- Specialist 의견 종합
- 해결 방향 추천
- 추가 검토 필요성 판단

이러한 판단은 매번 Human 승인을 받을 필요가 없다.

단, CEO가 판단했다는 사실이 곧 그 결과의 정확성을 보장하지는 않는다.

#### B. Delegation Authority — 업무를 위임할 권한

CEO가 Specialist에게 업무를 맡기는 권한이다.

여기서는 두 행위를 구분해야 한다.

```
CEO
  ↓
"UX Specialist가 필요하다"
  ↓
Staffing Proposal 생성
  ↓
Harness Validation
  ↓
Human Staffing Approval
  ↓
Approved Delegation
  ↓
Specialist Execution
```

CEO는 자유롭게 전문가 투입을 제안할 수 있다.

하지만 실제 위임은 승인된 Staffing과 Authority 안에서만 이루어져야 한다.

한 번 승인받았다고 모든 후속 위임을 다시 승인받을 필요는 없다. 승인된 범위 내에서의 실행 순서나 작업 판단은 CEO가 자율적으로 할 수 잇어야 한다.

반대로 새로운 Specialist를 추가하거나 승인된 Context·Budget을 실질적으로 확대한다면 재검증 및 재승인이 필요하다.

#### C. Execution Authority — 실제 행동을 수행할 권한

이 부분을 가장 엄격하게 분리해야 한다.

CEO에게 **Repository 변경 권한을 직접 부여하지 않는 방향을 제안**한다.

```
CEO
  ↓
Approved Decision
  ↓
Task Planning (v1)
  ↓
TaskContract
  ↓
Human Task Approval
  ↓
Repository Harness (v0)
  ↓
Worker Execution
```

CEO는 무엇을 구현할지 판단하지만, 실제 파일 수정은 기존 Worker에게 맡긴다.

이렇게 해야 v0에서 구축한 Safe Workspace, Scope Enforcement, Verification, Evidence를 그대로 재사용할 수 있다.

## 4. CEO가 어디까지 자율적으로 행동할 수 있을까?

여기서는 **Approved Operation Boundary(승인된 운영 범위)**라는 개념을 도입하는게 좋다.

아직 타입 이름으로 확정하자는 뜻은 아니고, Authority 설계를 위한 개념이다.

예를 들어 Human이 다음 구성을 승인했다고 해보자.

```
Approved Organization
├─ Goal
│  └─ Improve login form UX
│
├─ Specialists
│  ├─ Frontend
│  └─ UX
│
├─ Context
│  ├─ LoginForm.tsx
│  └─ useLogin.ts
│
├─ Budget
│  ├─ Maximum agent calls
│  └─ Maximum execution time
│
└─ Repository Snapshot
   └─ commit SHA
```

이 승인 이후 CEO의 행동은 다음과 같이 구분할 수 있다.

| 행동                           | 처리                      |
| ------------------------------ | ------------------------- |
| 승인된 Frontend Agent 실행     | 허용 범위 검증 후 실행    |
| 승인된 UX Agent 실행           | 허용 범위 검증 후 실행    |
| 승인된 전문가의 실행 순서 조정 | 정책 범위 내 자율 판단    |
| 새로운 Security Agent 추가     | 새로운 Staffing Approval  |
| 승인되지 않은 파일 요청        | 재검증 및 필요 시 재승인  |
| Budget 확대 요청               | 정책 검증 및 필요 시 승인 |
| 직접 Repository 수정           | CEO 권한으로는 불허       |

이 구조라면 **CEO의 자율성**을 살리면서도 **AI 조직의 권한이 무제한으로 확대되는 것**을 막을 수 있다.

핵심은 CEO의 모든 결정을 승인하는 것이 아니라, **실제 권한이 부여되거나 변경되는 지점을 통제**하는 것이다.

## 5. 실제 시나리오로 검증해보자

Human이 다음 Goal을 전달했다고 가정하자

> 로그인 폼에서 중복 제출을 방지하고, 로딩 중 사용자 경험을 개선해 줘

#### Phase 1. Intent & Staffing

CEO는 Goal을 분석해 다음처럼 판단한다.

```
Intent
- 중복 제출 문제 해결
- 로딩 중 사용자 피드백 개선

Staffing Proposal
- Frontend Specialist
  → 비동기 제출 흐름과 중복 요청 검토

- UX Specialist
  → 로딩 상태와 피드백 방식 검토
```

CEO가 Staffing을 제안하고, Harness가 권한과 Context를 검증한 뒤 Human이 승인한다.

#### Phase 2. Independent Decisions

승인된 Specialist가 독립적으로 분석한다.

```
Frontend
- Pending 상태에서 Submit 비활성화
- 중복 요청 차단

UX
- Pending 상태 시각적 피드백
- 반복 클릭 방지
```

기존 v2가 이 역할을 담당한다.

#### Phase 3. Comparison & Synthesis

v3 Comparator가 Concern별 관계를 분석한다.

CEO는 그 결과를 바탕으로 판단한다.

```
CEO Synthesis
- Submit 중복 실행을 막는다.
- Pending 상태를 사용자에게 표시한다.

Rationale
- Frontend의 중복 요청 방지 근거
- UX의 상태 피드백 근거

Unresolved
- 서버 측 중복 요청 방어 여부 미확인
```

여기서 CEO는 여러 판단을 조합해 해결 방향을 추천할 수 있다.

단, Comparator결과에 없던 사실이나 검증되지 않은 주장을 확정된 Evidence로 취급해선 안 된다.

#### Phase 4. Governance & Execution

Human이 해결 방향을 승인하면 기존 v1 Planning을 통해 구체적인 TaskContract를 작성한다. 이후 Human Task Approval을 거쳐 v0 Worker가 실행한다.

CEO는 실제 Verification 결과를 받아 Executive Report를 작성한다.

CEO는 결과를 해석하지만, Verification 실패를 성공으로 바꿔 보고할 수 없다.

## 6. CEO가 소유하지 않아야 할 책임

이것도 명확히 정해두자

| CEO가 소유하지 않는 책임       | 실제 소유자     |
| ------------------------------ | --------------- |
| Authority / Scope 정책 정의    | Harness / Human |
| Staffing Approval의 최종 승인  | Human           |
| Evidence Reference 무결성 검증 | Harness         |
| 모델 프로세스 lifecycle        | Agent Runtime   |
| Workflow Checkpoint / Resume   | LangGraph       |
| TaskContract 유효성 검증       | Harness         |
| Repository Scope Enforcement   | 기존 v0         |
| Apply / Commit / Push 승인     | Human           |
| 실행 결과의 원본 기록          | Harness         |

CEO는 이러한 시스템이 제공하는 결과아 정책을 참고해 판단할 수 있지만, 정책이나 검증 결과를 임의로 다시 정의하지는 않는다.

## CEO Responsibility Definition — 설계안

> CEO Agent는 Human Goal을 달성하기 위해 AI 조직의 구성, 업무 배분, 협업 전략, 판단 종합 및 결과 보고를 자율적으로 수행하는 의사결정자다.
>
> CEO는 승인된 운영 범위 안에서 조직 활동을 조정할 수 있지만, 새로운 실행 권한을 스스로 부여하거나 Human Approval을 대체할 수 없다.
>
> Harness는 CEO의 실행 요청을 정책과 승인 상태에 따라 검증하며, Runtime은 허용된 Workflow를 수행한다. 실제 Repository 변경은 기존 TaskContract와 Safe Execution 경계를 통해 진행한다.

그리고 다음 단계 Step 1.2를 진행할 때는 다음 세 가지를 구체화하면 된다.

1. Authority Scope — CEO에게 부여된 권한을 어떤 데이터로 표현할 것인가?
2. Authority Validation — 실행 요청을 어떤 조건으로 Allowed / Approval Required / Denied로 분류할 것인가?
3. Authority Lifecycle — 승인된 권한은 언제 유효하고 언제 재검증·재승인이 필요한가?

특히 다음 Step에서는 `CEOAuthority`를 단순한 `canStaff`, `canExecute` 같은 Boolean 조합으로 만들지 않는 게 중요하다. 무엇을, 어떤 조건과 범위에서 허용하는지 표현할 수 있는 모델이 필요하기 때문이다.
