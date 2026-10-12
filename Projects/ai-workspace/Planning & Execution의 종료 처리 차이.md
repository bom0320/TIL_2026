### 핵심 원칙

`finalizer`는 **workflow가 끝난다는 이유만으로 무조건 만드는 게 아니다.**

중요한 기준은:

> **종료 과정 자체가 독립된 책임으로 볼 만큼 복잡한가?**

이다.

---

## Execution에 finalizer가 있는 이유

Execution은 마지막에 해야 할 일이 많다.

대략:

```
ExecutionState
↓
passed 계산
↓
evidence / scope / failures 조립
↓
artifacts 반영
↓
retainedWorkspace 반영
↓
result.json 저장
↓
저장 실패 시 report failure 추가
↓
ExecutionResult 반환
```

즉 단순히 값을 반환하는 게 아니라:

- 상태를 최종 결과로 조립하고
- 성공 여부를 계산하고
- 결과를 저장하고
- 저장 실패까지 실패 상태에 반영해야 한다.

그래서 종료 처리 자체가 하나의 책임이 된다.

```
finalizeExecutionResult()
= Execution workflow의 종료 처리
```

따라서 `flow/orchestration`에 별도로 두는 의미가 있다.

---

## Planning에는 왜 별도 finalizer가 없는가

현재 Planning의 종료는 훨씬 단순하다.

```
PlannerDecision = complete
↓
TaskDraft
↓
buildTaskContract()
↓
PlanningResult 생성
↓
return
```

예를 들면 마지막 처리가 사실상:

```tsx
return {
  status: "completed",
  task,
  state,
  rounds,
};
```

정도라면 별도의:

```tsx
finalizePlanningResult();
```

를 만드는 것은 책임 분리가 아니라 단순한 함수 쪼개기가 된다.

그래서 지금은 만들지 않는 게 낫다.

---

## `buildTaskContract`와 finalizer는 다르다

Planning에 있는:

```tsx
buildTaskContract();
```

는 workflow 종료 함수가 아니다.

역할은:

```
TaskDraft
+ GoalSpec
+ IDs
↓
TaskContract
```

즉 데이터 변환과 Harness 정책 적용이다.

주요 책임:

- `taskId` 주입
- `goalId` 주입
- `targetRepository` 고정
- Human constraints 보존
- 중복 constraint 제거
- schema validation

그래서:

```
buildTaskContract
= transformation

finalizeExecutionResult
= orchestration finalization
```

으로 구분한다.

---

## 중요한 설계 기준

Planning과 Execution의 구조를 억지로 대칭으로 만들 필요는 없다.

잘못된 접근:

```
Execution에 finalizer 있음
→ Planning에도 finalizer 만들어야 함
```

좋은 접근:

```
실제 책임이 존재하는가?
↓
Yes → 분리
No  → 만들지 않음
```

즉:

> **구조적 대칭성보다 실제 책임을 우선한다.**

---

## 나중에는 Planning finalizer가 생길 수도 있음

예를 들어 Planning에 Human Approval이 붙고 종료 과정이 이렇게 커진다면:

```
TaskContract 생성
↓
approval state 생성
↓
decision history 저장
↓
planning artifact 저장
↓
planning report 생성
↓
PlanningResult 반환
```

그때는:

```
finalizePlanningResult()
```

또는

```
completePlanning()
```

같은 별도 종료 책임을 만드는 게 자연스럽다.

---

## 현재 기준 정리

```
Execution
→ 종료 과정이 복잡함
→ finalizer 존재

Planning
→ 종료 과정이 단순함
→ 별도 finalizer 없음

buildTaskContract
→ 종료 처리 아님
→ transformation
```

한 줄로 요약하면:

> **Finalizer는 workflow의 끝이라서 만드는 게 아니라, workflow 종료 자체가 독립된 책임이 되었을 때 만든다.**
