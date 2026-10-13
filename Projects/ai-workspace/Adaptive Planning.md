# Adaptive Planning — Fast Path / Trivial Task Detection / Planning Cost Optimization

### 문제

기존 Planning은 작업이 매우 단순한 경우에도 항상 Codex Planner를 호출했다.

예를 들어:

```
README.md의 소개 문구를 더 명확하게 수정해줘
```

처럼 수정 대상과 범위가 이미 명확한 작업도:

```
Repository inspection
→ Initial inspection
→ Codex Planner
→ complete
```

과정을 거쳤다.

이 경우 실제 작업 난이도와 관계없이 최소 1회의 LLM 호출이 발생했고, 작은 수정에서도 Planning latency가 크게 발생했다.

초기 E2E 측정에서는:

```
Planning:      53.7s
Worker:       167.0s
Verification: 17.5s
Total:        265.1s
```

으로 나타났다.

핵심 문제는 Harness의 로컬 처리 속도가 아니라, **단순 작업에도 동일한 LLM Planning 경로를 적용하고 있다는 점**이었다.

---

### Goal

```
Goal
 ↓
Repository inspection
 ↓
Initial exact-match inspection
 ↓
Trivial task detection
 ├─ YES → Fast Path TaskDraft → TaskContract
 │          rounds = 0
 │          LLM 호출 = 0
 │
 └─ NO  → 기존 Planning Loop
            Codex Planner
            inspect / complete
```

현재 `planTask()`는 이미:

```
resolveRepositoryRoot()
→ inspectRepository()
→ inspectInitialFiles()
```

순서로 Repository context와 Goal에서 명시된 파일을 선탐색하고 있었다.

따라서 새로운 Planning 시스템을 별도로 만드는 대신, **`inspectInitialFiles()` 직후에 Fast Path 판정을 추가**했다.

---

### 1. Trivial Task Detection

새로운 `createFastPathTaskDraft()`가 현재 Planning State를 기준으로 Fast Path 사용 가능 여부를 판단한다.

Fast Path는 의도적으로 보수적으로 제한했다.

현재 주요 조건은:

```
- inspected file이 정확히 1개
- 해당 파일이 실제 repository inventory에 존재
- 대상 파일이 .md 또는 .txt
- Goal에서 대상 파일이 명시적으로 언급됨
- 문구 / 텍스트 / 오타 / 설명 등의 textual edit
- 수정 / 교정 / 다듬기 등의 단순 변경
- 복잡하거나 파괴적인 intent가 없음
```

예:

```
README.md의 소개 문구를 더 명확하게 수정해줘
README의 오타를 수정해줘
notes.txt의 문구를 수정해줘
```

→ Fast Path

반면:

```
README 구조를 리팩토링하고 코드 예제를 추가해줘
README.md와 package.json의 문구를 수정해줘
README.md를 삭제해줘
example.ts의 로직을 수정해줘
```

→ 기존 Planner로 fallback

Fast Path에서는 **false positive보다 false negative를 허용하는 방향**으로 설계했다.

즉 애매한 작업을 잘못 Fast Path로 실행하는 것보다, 단순 작업을 기존 Planner로 보내는 편을 선택한다.

---

### 2. Fast Path

Trivial Task로 판정되면 Codex Planner를 호출하지 않고 바로 `TaskDraft`를 생성한다.

```
PlanningState
 ↓
createFastPathTaskDraft()
 ↓
TaskDraft
 ↓
buildTaskContract()
 ↓
TaskContract
```

기존 `TaskDraft`, `buildTaskContract()`, `TaskContract` validation을 그대로 재사용하기 때문에 별도의 계약 생성 흐름을 추가하지 않았다.

결과는:

```
rounds = 0
Planner call = 0
```

이 된다.

반대로 Fast Path 조건을 만족하지 않으면 기존 흐름을 그대로 사용한다.

```
runPlanningLoop()
→ Codex Planner
→ inspect / complete
```

따라서 Fast Path는 기존 Planning을 대체하는 것이 아니라 **앞단에 추가된 최적화 경로**다.

---

### 3. 안전 장치

Fast Path에서 잘못된 target을 선택하지 않도록 몇 가지 제한도 추가했다.

예를 들어:

```
../README.md
../../README.md
..\README.md
docs/../../README.md
```

처럼 parent-directory traversal 표현이 포함된 요청은 Fast Path에서 거부한다.

또한 repository 내에:

```
docs/guide.md
examples/guide.md
```

처럼 같은 basename이 여러 개 존재하면:

```
guide.md의 문구를 수정해줘
```

만으로는 target이 명확하지 않기 때문에 Planner로 fallback한다.

---

### 4. E2E 결과

Fast Path 적용 후 동일한 수준의 README 수정 작업을 다시 실행했다.

입력:

```
README.md의 소개 문구를 더 명확하게 수정해줘
```

결과:

```
Planning: completed
Rounds: 0
Planning Duration: 0.0s
```

전체 실행:

```
Planning:       0.0s
Approval Wait:  6.4s
Workspace:      0.1s
Worker:        13.4s
Verification:   0.0s
Cleanup:        0.0s
Total:         20.0s
```

초기 측정과 비교하면:

```
Planning
53.7s → 0.0s

Total
265.1s → 20.0s
```

전체 실행 시간은 약 **92% 감소**했다.

Human Approval에 사용된 6.4초를 제외하면 Harness의 실제 자동 처리 시간은 약:

```
13.5s
```

수준이었다.

---

### 성과

이번 개선으로 Planning은 더 이상 모든 작업을 동일한 비용으로 처리하지 않는다.

기존:

```
모든 Goal
→ LLM Planner
```

현재:

```
Goal
 ↓
Adaptive Planning
 ├─ trivial → deterministic Fast Path
 └─ complex → LLM Planning
```

즉 단순한 작업에서는 LLM 자체를 제거하고, 복잡한 작업에서만 LLM Planning 비용을 지불하도록 변경했다.

핵심 성과는:

```
Planning rounds: 1+ → 0
Planner LLM calls: 1+ → 0
Planning latency: 53.7s → 0.0s
Total latency: 265.1s → 20.0s
```

이다.

이 단계에서는 Fast Path를 더 일반화하지 않고, **확실한 trivial task만 deterministic하게 처리하는 수준**에서 마무리했다.
