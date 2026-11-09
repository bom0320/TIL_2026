# AI Workspace v4 — Framework 도입 및 Workflow 통합 방향

## 1. 문제 인식

현재 AI Workspace는 v0~v3에서 개발한 개별 기능을 v4의 전체 워크플로로 통합하는 단계에 있다.

- **v0:** 안전한 Repository 실행 및 검증
- **v1:** Goal 기반 Planning 및 TaskContract 생성
- **v2:** Specialist의 독립적인 의사결정
- **v3:** 의사결정 결과의 구조화된 비교
- **v4:** CEO 중심 조직 운영, Governance, Authorization, 전체 실행 관리

각 기능을 구현하는 것뿐 아니라 승인, 권한, 데이터 전달, 실행 순서, 실패 처리까지 연결해야 하므로 설계와 구현에 상당한 시간이 소요되고 있다.

여기서 다음 의문이 생겼다.

> 이미 존재하는 Agent Framework나 Workflow Runtime이 해결하는 문제까지 직접 구현하면서 불필요하게 시간을 사용하고 있는 것은 아닐까?

핵심은 프레임워크 사용 여부 자체가 아니라, **AI Workspace가 직접 소유해야 할 책임과 외부 기술을 활용해야 할 책임을 구분하는 것**이다.

---

## 2. 판단: Domain 설계와 Workflow 실행은 다르다

현재 개발하는 기능은 크게 두 영역으로 나눌 수 있다.

### A. Domain & Governance — 직접 구현

AI Workspace만의 운영 규칙과 판단 기준에 해당한다.

- System Policy 및 Human Delegation
- Staffing Proposal 검증과 승인
- ApprovedOperatingScope 및 Authorization
- Context 접근 범위와 Budget 통제
- AgentDecision 및 Evidence 검증
- Human Approval과 실행 권한의 구분
- 버전 간 데이터 계약 및 신뢰 경계

이는 단순한 실행 기술이 아니라 **AI Workspace가 어떤 조건에서 AI의 행동을 허용하고, 결과를 어떻게 신뢰할지 정의하는 핵심 책임**이다.

프레임워크를 사용하더라도 이 영역은 직접 설계하고 유지해야 한다.

### B. Workflow Runtime — 외부 기술 활용

개별 기능을 하나의 실행 흐름으로 연결하고 관리하는 범용 기술 영역이다.

- 기능 간 실행 순서 관리
- 실행 상태 전달 및 조건부 분기
- Human 승인 대기와 중단·재개
- 실패에 따른 재시도 및 복구
- Agent 실행의 반복과 조정
- 장기 워크플로의 체크포인트 관리

이 영역은 직접 구현하는 대신 **LangGraph.js를 활용한다.**

단, 모든 함수 호출이나 내부 반복문까지 LangGraph로 대체하지 않는다. 복잡한 상태 관리와 재개가 필요한 상위 워크플로에 적용한다.

---

## 3. LangGraph.js의 역할

LangGraph.js는 기존 기능을 대체하는 것이 아니라 **기존 기능들을 연결하고 전체 실행 흐름을 관리하는 Workflow Runtime**이다.

### 책임 관계

| 구성 요소    | 책임                                                    |
| ------------ | ------------------------------------------------------- |
| CEO          | 문제 해석, 인력 배치, 의사결정 종합, 진행 평가          |
| Harness      | Governance, Authorization, Context, Evidence, 승인 규칙 |
| LangGraph.js | 워크플로 상태, 실행 순서, 분기, 중단·재개               |
| Codex        | 개별 CEO, Specialist, Worker의 AI 실행                  |
| Human        | 운영 정책, 주요 의사결정과 변경사항의 최종 승인         |

예를 들어 기존 기능은 다음처럼 유지한다.

- `runStaffing()` — Staffing 수행
- `approveStaffingProposal()` — Staffing 승인 처리
- `authorizeSpecialistInvocation()` — 개별 호출 권한 검사 및 예약
- `runCodexSpecialist()` — Specialist 실행
- `compareDecisions()` — 독립 판단 비교

LangGraph는 이러한 Application API를 호출하는 상위 실행 흐름을 구성한다.

단, **LangGraph의 실행 상태가 승인이나 권한의 근거를 대체해서는 안 된다.** Governance Ledger와 Harness가 여전히 권한의 신뢰된 기준이다.

---

## 4. 기존 로드맵은 폐기하지 않는다

R1~R6는 AI Workspace가 구현해야 할 기능과 데이터 계약을 정의한 계획이다.

LangGraph 도입은 해당 계획을 폐기하는 것이 아니라, **각 기능을 통합하는 구현 방식을 보완하는 결정**이다.

| 순서 | 작업                                   | 상태 및 방향                       |
| ---- | -------------------------------------- | ---------------------------------- |
| 1    | R1. 공통 모델 소유권 정리              | 구현 완료, 유지                    |
| 2    | R2. Staffing Finalization              | 구현 완료, 유지                    |
| 3    | Human Staffing Approval                | 구현 완료, 유지                    |
| 4    | R4. Context Discovery 및 Authorization | 현재 작업 마무리                   |
| 5    | R3. Decision 실행 계약 개선 및 v3 연결 | 기존 계약 설계·실행 검증 유지      |
| 6    | CEO Synthesis · Human Governance       | 기존 계획 유지                     |
| 7    | R5. Planning Handoff 개선              | Phase B 계약 직접 설계             |
| 8    | 다중 업무 실행·검증·보고               | LangGraph 기반 실행 조정 활용      |
| 9    | R6. 통합 검증                          | 기존 검증에 Runtime 통합 검증 추가 |

### 변경할 부분: LangGraph 통합 시점 명확화

기존 순서를 다음과 같이 구체화한다.

**Step 4 — Authorization 마무리**

기존 Scope 검증, 예산 예약, 실행 입력 생성의 안전성을 확보한다.

**Step 5 — R3 Decision 실행 계약 개선**

저장된 Reservation과 Invocation Digest를 확인한 뒤 v2 Specialist를 실행하고, 그 결과를 v3 Comparison으로 전달할 계약을 정한다.

**Step 5 이후 — LangGraph 최소 통합**

기존 Application API를 연결하여 다음 Phase A 흐름을 실제로 실행한다.

Human Goal → Staffing → Approval → Authorization → Specialist Decision → Comparison

이때 상태 전달, 실패 분기, Human 승인 대기·재개의 기본 동작을 검증한다.

**Step 6~7 — CEO Synthesis 및 Human Governance**

이미 연결한 워크플로에 CEO 종합 판단과 Human의 해결 방향 승인을 추가한다.

**Phase B — Planning 및 다중 업무 실행**

기존 v1 Planning과 v0 Execution을 재사용하고, LangGraph를 이용해 업무 조정·반복·진행 평가 흐름을 확장한다.

따라서 LangGraph 통합은 기존 로드맵을 대체하는 별도 프로젝트가 아니라, **Step 5 이후부터 실제 워크플로를 연결하면서 적용하는 실행 기술**로 취급한다.

---

## 5. 주의할 설계 경계

### ① Framework가 Authorization을 대신하지 않는다

LangGraph가 실행 순서를 관리하더라도 Specialist를 호출하기 전에는 Harness의 Authorization을 통과해야 한다.

### ② Workflow State와 Governance Ledger를 구분한다

LangGraph State는 실행 진행 상황을 관리한다.

Governance Ledger는 승인 근거, Scope, Reservation 등의 권한 기록을 관리한다.

워크플로를 재개했다고 이전 승인이나 권한을 자동으로 신뢰해서는 안 된다.

### ③ 재개 기능이 중복 실행 방지를 자동 보장하지 않는다

프레임워크의 체크포인트와 재개 기능만으로 외부 Agent 실행이 정확히 한 번 수행된다고 보장할 수 없다.

기존 Reservation, Invocation Digest, 실행 이력과 멱등성 검증을 별도로 설계해야 한다.

### ④ 기존 기능을 전면 재작성하지 않는다

Application 내부의 단순한 함수 호출, 검증 과정, 제한된 반복 로직은 그대로 유지한다.

여러 Application을 연결하는 상위 Workflow에 LangGraph를 도입한다.

### ⑤ Framework를 위한 불필요한 추상화를 추가하지 않는다

범용 Agent 인터페이스, Provider Registry, Runtime 추상화 등을 필요하기 전에 설계하지 않는다.

현재 사용 중인 Codex 기반 실행과 기존 Application API를 우선 재사용한다.

---

## 6. 예상 효과와 트레이드오프

**기대 효과**

- 상위 워크플로의 실행 순서와 상태 관리 코드 감소
- Human Approval을 포함한 중단·재개 기능 활용
- CEO의 반복적인 판단과 업무 재배치 흐름 확장 용이
- 기존 Application 기능을 독립적으로 유지하며 통합 가능

**새롭게 발생하는 비용**

- LangGraph State 및 노드 설계
- 기존 Application과 Runtime 사이의 데이터 변환
- Checkpoint 저장 및 복구 정책
- 재시도 시 중복 실행과 권한 우회 방지
- Framework API와 버전에 대한 의존성

따라서 프레임워크를 도입한다고 개발 시간이 무조건 줄어드는 것은 아니다.

특히 단순한 순차 호출에는 직접 작성한 함수가 더 명확할 수 있다.

LangGraph의 가치는 **복잡한 분기, 반복, 중단·재개가 필요한 전체 워크플로에서 검증해야 한다.**

---

## 7. 다음 단계에서 증명할 것

LangGraph 최소 통합 후 다음 항목을 확인한다.

1. 기존 v0~v3 Application을 대규모 수정 없이 재사용할 수 있는가?
2. Staffing부터 Decision·Comparison까지 실제 E2E로 연결되는가?
3. Human 승인 없이 다음 단계로 진행하지 않는가?
4. 승인된 Context만 Specialist에게 전달되는가?
5. 중단·재개 또는 재시도 시 권한과 예산을 우회하지 않는가?
6. 기존 Governance Ledger와 Workflow State의 책임이 분리되어 있는가?
7. 직접 구현한 워크플로와 비교해 상태·분기·복구 관리의 복잡도가 실제로 줄어드는가?

기능이 동작하는지만 확인하지 않고, **기존 설계의 신뢰 경계를 유지하면서 프레임워크 도입의 실질적 이점이 있는지** 검증한다.

---

## 8. 최종 설계 결정

**결정: 기존 R1~R6 로드맵과 Application 설계는 유지하고, LangGraph.js를 v4 전체 워크플로의 실행 Runtime으로 도입한다.**

기존 문서에서 확정한 LangGraph.js 채택 방향을 유지한다. 현재 코드에 설치·통합되지 않은 상태와 설계상 도입 결정을 구분한다.

LangGraph의 통합은 Step 5의 실행 계약을 확립한 뒤 최소 Phase A 연결부터 시작하고, 이후 Synthesis와 Phase B로 확장한다.

이를 통해 다음 원칙을 유지한다.

> **Harness는 AI가 무엇을, 어떤 권한과 조건에서 수행할 수 있는지 결정한다. Runtime은 허용된 작업이 어떤 순서와 상태로 실행되는지 관리한다.**

지금까지의 Governance 설계를 폐기할 필요는 없다. 다만 앞으로는 범용 실행 인프라를 직접 구현하는 범위를 줄이고, 기존 기능이 실제 Repository에서 하나의 완성된 워크플로로 작동하는지를 우선 검증한다.

**목표는 프레임워크를 만드는 것이 아니라, 신뢰할 수 있는 AI Development Organization을 완성하는 것이다.**
