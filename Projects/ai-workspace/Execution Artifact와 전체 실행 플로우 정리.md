# Execution Artifact와 전체 실행 플로우 정리

## 1. Artifact가 왜 필요한가

현재 `ai-workspace`의 Execution은 대략 다음 구조로 동작한다.

```
TaskContract
    ↓
Execution Workspace 생성
    ↓
AI Worker 실행
    ↓
Scope 검사
    ↓
Verification
    ↓
Workspace 삭제
```

여기서 문제가 하나 있다.

AI가 실제로 작업하는 공간은 임시 Git Worktree이고, 실행이 끝나면 이 Worktree는 삭제된다.

즉 아무런 기록을 남기지 않는다면:

```
AI가 파일 수정
    ↓
검증 완료
    ↓
Worktree 삭제
    ↓
AI가 무엇을 수정했는지 사라짐
```

이 되어버린다.

그래서 실행 결과를 영구적으로 보존하기 위한 것이 **Execution Artifact**다.

Artifact는 단순 로그라기보다,

> 임시 실행 환경에서 발생한 결과를 나중에도 검토할 수 있는 증거로 변환하는 장치

에 가깝다.

---

# 2. 최종적으로 남는 Artifact

한 번의 Task가 실행되면 다음과 같은 디렉터리가 만들어진다.

```
.ai-workspace/
└── runs/
    └── run-XXXXXX/
        ├── task.json
        ├── base-commit.txt
        ├── changes.patch
        └── result.json
```

각 파일의 역할은 다음과 같다.

| 파일              | 역할                                             |
| ----------------- | ------------------------------------------------ |
| `task.json`       | 이번 실행에서 AI에게 무엇을 시켰는지             |
| `base-commit.txt` | AI가 작업하기 전 Repository 상태                 |
| `changes.patch`   | AI가 실제로 변경한 코드                          |
| `result.json`     | 실행 성공 여부, Evidence, Failure 등의 최종 결과 |

따라서 Artifact만 보면 다음 질문에 답할 수 있다.

```
무엇을 시켰는가?
→ task.json

어느 코드 상태에서 시작했는가?
→ base-commit.txt

AI가 실제로 무엇을 바꿨는가?
→ changes.patch

결과가 성공했는가?
무엇이 실패했는가?
어떤 검증을 통과했는가?
→ result.json
```

---

# 3. 전체 Execution 흐름

현재 `executeTask()`의 흐름은 다음과 같다.

```
executeTask()
    │
    ├─ 1. createExecutionRun()
    │      └─ run directory 생성
    │      └─ task.json 저장
    │
    ├─ 2. createExecutionWorkspace()
    │      └─ 임시 Git Worktree 생성
    │
    ├─ 3. getExecutionBaseCommit()
    │      └─ 작업 시작 시점 HEAD 저장
    │
    ├─ 4. runWorkerStage()
    │      └─ AI Worker가 실제 코드 수정
    │
    ├─ 5. runScopeStage()
    │      └─ 허용된 파일만 수정했는지 검사
    │
    ├─ 6. runVerificationStage()
    │      └─ test / typecheck 등의 검증 실행
    │
    ├─ 7. runScopeStage()
    │      └─ Verification 이후 다시 Scope 검사
    │
    ├─ 8. preserveExecutionChanges()
    │      └─ base-commit.txt 저장
    │      └─ changes.patch 저장
    │
    ├─ 9. removeExecutionWorkspace()
    │      └─ 임시 Worktree 삭제
    │
    └─ 10. finalizeExecutionResult()
           └─ result.json 저장
```

---

# 4. `createExecutionRun()`

```tsx
export function createExecutionRun(
  task: TaskContract,
  runsRoot = resolve(process.cwd(), ".ai-workspace", "runs")
): string;
```

역할은 이번 실행을 위한 기록 공간을 만드는 것이다.

먼저:

```tsx
mkdirSync(runsRoot, { recursive: true });
```

를 통해:

```
.ai-workspace/runs
```

디렉터리가 없으면 생성한다.

그 후:

```tsx
const runDirectory = mkdtempSync(join(runsRoot, "run-"));
```

를 실행한다.

결과는 예를 들어:

```
.ai-workspace/runs/run-A12BCD/
```

처럼 실행마다 별도의 디렉터리가 생긴다.

그리고 바로:

```tsx
writeFileSync(
  join(runDirectory, "task.json"),
  `${JSON.stringify(task, null, 2)}\n`
);
```

를 통해 TaskContract를 저장한다.

예:

```tsx
{
  "id": "task-readme-001",
  "objective": "README 소개 문구를 수정한다",
  "allowedPaths": ["README.md"]
}
```

### 왜 Workspace보다 먼저 만들까?

`executeTask()`를 보면:

```tsx
runDirectory = createExecutionRun(task);
```

이 먼저 실행되고,

그 이후에:

```tsx
workspaceRoot = createExecutionWorkspace(repositoryRoot);
```

가 실행된다.

Workspace 생성 자체가 실패할 수도 있기 때문이다.

실행 디렉터리를 먼저 확보하면:

```
Workspace 생성 실패
Worker 실행 실패
Verification 실패
```

같은 상황도 최종 `result.json`에 기록할 수 있다.

즉 Execution이 시작되는 순간부터 기록 가능하도록 만드는 구조다.

---

# 5. Execution Workspace

다음으로:

```tsx
workspaceRoot = createExecutionWorkspace(repositoryRoot);
```

를 호출한다.

이 부분은 Artifact가 아니라 Git Infrastructure 영역이다.

원본 Repository가:

```
/Users/bom/Documents/ai-workspace
```

라면 AI에게 원본 Repository를 직접 수정시키지 않고 별도의 Worktree를 만든다.

예:

```
/tmp/ai-workspace-X1234
```

구조는:

```
Original Repository
        ↓
Temporary Git Worktree
        ↓
AI Worker
```

가 된다.

이렇게 하면 AI가 이상한 수정을 해도 원본 Repository는 직접 영향을 받지 않는다.

---

# 6. `getExecutionBaseCommit()`

Workspace를 만들고 바로 다음 코드를 실행한다.

```tsx
artifacts.baseCommit = getExecutionBaseCommit(workspaceRoot);
```

함수 내부는 사실상 다음 Git 명령을 실행한다.

```bash
git -C <workspaceRoot> rev-parse HEAD
```

결과:

```
a102e7892c...
```

이 값이 작업 시작 시점의 기준점이다.

예를 들어:

```
base commit
    ↓
a102e78
```

인 상태에서 Worker가 파일을 수정한다면 나중에:

```
a102e78
vs
현재 Worktree
```

를 비교해서 AI가 무엇을 수정했는지 알아낼 수 있다.

그래서 반드시 Worker 실행 **이전**에 Base Commit을 확보해야 한다.

---

# 7. Worker 실행

그 다음:

```tsx
const workerSucceeded = await runWorkerStage(
  task,
  state,
  workspaceRoot,
  options
);
```

가 실행된다.

여기가 실제 AI 작업 단계다.

예를 들어 Task가:

```
README 소개 문구를 수정한다
```

라면 Worker가:

```
README.md 수정
```

을 수행한다.

조금 더 복잡한 작업이면:

```
src/foo.ts 수정
src/bar.ts 생성
README.md 수정
```

같은 변경이 생길 수 있다.

이 변경은 아직 Artifact에 저장된 것이 아니라 **Worktree 안에만 존재한다.**

---

# 8. Worker가 실패한 경우

```tsx
if (!workerSucceeded) {
  collectChangedPathsEvidence(state, workspaceRoot, baseCommit);
}
```

Worker가 실패했다고 해서 아무것도 수정하지 않았다는 뜻은 아니다.

예:

```
README.md 수정
    ↓
src/foo.ts 수정
    ↓
Codex 실행 중 오류 발생
```

이럴 수 있다.

그래서 Worker가 실패하더라도:

```tsx
collectChangedPathsEvidence(...)
```

를 통해 어떤 파일을 변경했는지 Evidence를 수집한다.

예:

```
state.evidence.changedPaths = [
  "README.md",
  "src/foo.ts"
];
```

즉 실패한 실행도 가능한 한 사실을 남기는 구조다.

---

# 9. Scope 검사

Worker가 성공하면:

```tsx
runScopeStage(task, state, workspaceRoot, baseCommit);
```

를 실행한다.

TaskContract가:

```
{
  "allowedPaths": ["README.md"]
}
```

라고 되어 있는데 AI가:

```
README.md
package.json
```

을 수정했다고 해보자.

그러면:

```
README.md
→ 허용

package.json
→ 허용되지 않음
```

이므로 Scope 실패가 발생한다.

Scope 검사는 결국:

```
Base Commit
vs
현재 Worktree
```

를 비교해서 변경 파일을 알아내는 과정이다.

그래서 여기에서도 `baseCommit`이 필요하다.

---

# 10. Verification

Scope를 통과하면:

```tsx
await runVerificationStage(task, state, workspaceRoot, options);
```

가 실행된다.

예를 들어 TaskContract에:

```tsx
{
  "verification": [
    "pnpm test",
    "pnpm typecheck",
    "git diff --check"
  ]
}
```

가 들어있다면 해당 명령어들을 실행한다.

즉:

```
AI가 요구사항을 구현했는가?
```

뿐만 아니라:

```
그 코드가 실제로 정상적인가?
```

를 확인하는 단계다.

---

# 11. Scope 검사를 두 번 하는 이유

Verification이 끝난 후:

```tsx
runScopeStage(task, state, workspaceRoot, baseCommit);
```

를 다시 실행한다.

처음 보면 중복처럼 보일 수 있지만 이유가 있다.

Verification 명령어 자체가 파일을 생성하거나 수정할 수 있기 때문이다.

예:

```
Worker
↓
README.md만 수정
↓
Scope 통과
↓
pnpm test
↓
snapshot 파일 생성
```

그러면 최종 상태는:

```
README.md
__snapshots__/foo.snap
```

가 되어 있을 수도 있다.

즉 Worker 직후에는 허용 범위였지만 Verification까지 끝난 최종 상태에서는 Scope를 벗어날 수 있다.

그래서 최종 변경 상태를 다시 검사한다.

---

# 12. `preserveExecutionChanges()`

Execution에서 Artifact와 가장 직접적으로 연결되는 부분이다.

```tsx
preserveExecutionChanges(workspaceRoot, runDirectory, baseCommit);
```

각 인자의 의미는:

```
workspaceRoot
→ AI가 실제로 작업한 곳

runDirectory
→ Artifact를 저장할 곳

baseCommit
→ 어느 시점과 비교할지
```

이다.

이 함수는 최종적으로:

```
base-commit.txt
changes.patch
```

를 생성한다.

---

# 13. 왜 임시 Git Index를 사용하는가

`preserveExecutionChanges()` 내부에는:

```tsx
const temporaryIndex = join(runDirectory, `.patch-index-${randomUUID()}`);
```

가 있다.

그리고:

```tsx
const gitEnvironment = {
  ...process.env,
  GIT_INDEX_FILE: temporaryIndex,
};
```

를 사용한다.

Git은 기본적으로:

```
Working Tree
    ↓
Index / Staging Area
    ↓
Commit
```

구조를 가진다.

그런데 Artifact를 만들기 위해 원래 Repository의 staging area를 건드리면 안 된다.

따라서 별도의 임시 Index를 만든다.

```
실제 Repository Index
→ 건드리지 않음

Temporary Index
→ Artifact 생성 용도로 사용
```

이라고 보면 된다.

---

# 14. `git read-tree`

다음 코드:

```tsx
execFileSync("git", ["-C", workspaceRoot, "read-tree", baseCommit], {
  env: gitEnvironment,
});
```

은 임시 Index를 Base Commit 상태로 초기화한다.

개념적으로는:

```
Temporary Index
    ↓
작업 시작 당시 상태로 맞춤
```

이다.

---

# 15. `git add -A`

다음으로:

```tsx
execFileSync("git", ["-C", workspaceRoot, "add", "-A", "--"], {
  env: gitEnvironment,
});
```

를 실행한다.

중요한 점은 실제 Git Index가 아니라 **Temporary Index**에 추가한다는 것이다.

- `A`를 사용하기 때문에:

```
수정된 파일
새로 생성된 파일
삭제된 파일
```

모두 포함된다.

예:

```
README.md 수정
src/new-file.ts 생성
src/old-file.ts 삭제
```

전부 기록된다.

---

# 16. `changes.patch` 생성

다음 명령이 핵심이다.

```tsx
execFileSync("git", [
  "-C",
  workspaceRoot,
  "diff",
  "--cached",
  "--binary",
  "--full-index",
  "--no-color",
  baseCommit,
  "--",
]);
```

사실상:

```tsx
git diff --cached --binary --full-index <baseCommit>
```

를 실행한다.

결과는:

```
diff --git a/README.md b/README.md
index 1234567..8910111 100644
--- a/README.md
+++ b/README.md
@@ -1,3 +1,3 @@
-Old description
+New description
```

같은 형태가 된다.

이 내용을:

```
changes.patch
```

에 저장한다.

---

# 17. 왜 그냥 `git diff`를 사용하지 않는가

단순히:

```
git diff
```

만 사용하면 새로 생성된 Untracked File을 제대로 포함시키기 어렵다.

예:

```
src/new-file.ts
```

를 AI가 새로 만들었다면 일반 `git diff`만으로는 놓칠 수 있다.

현재 구현은:

```
Temporary Index 생성
↓
git add -A
↓
git diff --cached
```

방식을 사용하기 때문에:

```
수정
생성
삭제
Binary 변경
```

까지 하나의 Patch로 안정적으로 저장할 수 있다.

---

# 18. `base-commit.txt`

Artifact 보존 중:

```tsx
writeFileSync(join(runDirectory, "base-commit.txt"), `${baseCommit}\n`);
```

도 실행된다.

예:

```
a102e7892c...
```

만 저장된 간단한 파일이다.

그런데 이 정보가 중요한 이유는 `changes.patch`가:

```
어떤 Repository 상태를 기준으로 생성된 Patch인가?
```

를 알아야 하기 때문이다.

즉:

```
base-commit.txt
+
changes.patch
```

가 한 쌍이라고 보면 된다.

---

# 19. 임시 Index 정리

Artifact 생성이 끝나면:

```tsx
finally {
  if (patchDescriptor !== undefined) {
    closeSync(patchDescriptor);
  }

  rmSync(temporaryIndex, { force: true });
}
```

를 실행한다.

즉:

```
changes.patch file descriptor 닫기

Temporary Git Index 삭제
```

를 수행한다.

Artifact를 생성하기 위해 잠깐 사용한 내부 자원을 정리하는 단계다.

---

# 20. 왜 Patch 저장 후에 Worktree를 삭제하는가

`executeTask()`에서는 다음 순서를 가진다.

```tsx
preserveExecutionChanges(...);

changesPreserved = true;
```

그 후에만:

```tsx
if (changesPreserved) {
  removeExecutionWorkspace(...);
}
```

를 수행한다.

이 순서가 중요하다.

만약 반대로:

```
Worktree 삭제
↓
Patch 저장
```

을 한다면 AI가 수정한 파일이 이미 사라졌기 때문에 Patch를 만들 수 없다.

현재 구조는:

```
AI 작업
↓
변경사항 Patch로 보존
↓
보존 성공 확인
↓
Worktree 삭제
```

이다.

즉 **증거를 확보한 후 실행 환경을 폐기한다.**

---

# 21. Patch 저장에 실패하면 왜 Worktree를 남기는가

다음 코드가 있다.

```tsx
let changesPreserved = false;
let cleanupSucceeded = false;
```

Patch 저장에 실패하면:

```tsx
changesPreserved = false;
```

상태가 된다.

그러면:

```tsx
if (changesPreserved) {
  removeExecutionWorkspace(...)
}
```

조건이 false이므로 Worktree를 삭제하지 않는다.

왜냐하면:

```
Patch 저장 실패
+
Worktree 삭제
```

가 동시에 일어나면 AI가 만든 코드가 완전히 사라지기 때문이다.

따라서 최소한 Worktree 자체라도 보존한다.

---

# 22. `retainedWorkspace`

마지막에는:

```tsx
const retainedWorkspace =
  changesPreserved && cleanupSucceeded ? undefined : workspaceRoot;
```

를 만든다.

정상적인 경우:

```
Patch 저장 성공
+
Worktree 삭제 성공
```

이면:

```tsx
retainedWorkspace = undefined;
```

이다.

Worktree를 남겨둘 필요가 없기 때문이다.

반대로:

```
Patch 저장 실패
```

또는:

```
Cleanup 실패
```

라면:

```tsx
retainedWorkspace = workspaceRoot;
```

로 남긴다.

즉 최종 Result를 본 사람이:

> 실행 환경이 아직 이 위치에 남아 있다.

는 사실을 알 수 있다.

---

# 23. `finalizeExecutionResult()`

마지막으로:

```tsx
return finalizeExecutionResult(state, artifacts, retainedWorkspace);
```

를 호출한다.

여기에는 실행 동안 모아온 정보가 들어간다.

### `state`

```
Evidence
Failures
Scope Result
Verification Result
```

등 실행 자체에 대한 정보.

### `artifacts`

```
run directory
base commit
```

등 저장된 Artifact 정보.

### `retainedWorkspace`

정리되지 못한 Worktree가 있을 경우 위치.

이들을 합쳐 최종 `ExecutionResult`를 만든다.

예:

```tsx
{
  "passed": true,
  "evidence": {
    "changedPaths": [
      "README.md"
    ]
  },
  "failures": [],
  "artifacts": {
    "directory": ".ai-workspace/runs/run-X1234",
    "baseCommit": "a102e7892c"
  }
}
```

그리고 `finalizeExecutionResult()` 내부에서:

```
writeExecutionResult(...)
```

를 호출하여:

```
result.json
```

으로 저장한다.

---

# 24. Artifact 함수와 `executeTask()`의 연결

각 함수가 실제 어디서 사용되는지 정리하면 다음과 같다.

## `createExecutionRun()`

```tsx
runDirectory = createExecutionRun(task);
```

실행 시작 시 사용.

역할:

```
run directory 생성
+
task.json 저장
```

---

## `getExecutionBaseCommit()`

```tsx
artifacts.baseCommit = getExecutionBaseCommit(workspaceRoot);
```

Worker 실행 전에 사용.

역할:

```
AI가 작업하기 전 Git 상태 확보
```

---

## `preserveExecutionChanges()`

```tsx
preserveExecutionChanges(workspaceRoot, runDirectory, baseCommit);
```

Worker, Scope, Verification이 끝난 뒤 사용.

역할:

```
base-commit.txt 생성
changes.patch 생성
```

---

## `writeExecutionResult()`

직접 `executeTask()`가 호출하지 않고:

```
executeTask()
↓
finalizeExecutionResult()
↓
writeExecutionResult()
```

형태로 호출된다.

역할:

```
result.json 생성
```

---

# 25. Artifact와 Evidence의 차이

헷갈리기 쉬운 부분이다.

## Evidence

실행 과정에서 얻은 **사실을 메모리상의 데이터 구조로 기록**한다.

예:

```tsx
state.evidence = {
  changedPaths: ["README.md"],
  verification: ...
};
```

질문으로 표현하면:

> 실행 중 무엇을 확인했는가?

---

## Artifact

그 실행 정보를 **파일 형태로 영구 보존**한다.

예:

```
task.json
base-commit.txt
changes.patch
result.json
```

질문으로 표현하면:

> 실행이 끝난 뒤 무엇을 남길 것인가?

따라서:

```
Evidence
→ 실행 도중의 증거 데이터

Artifact
→ 실행 후에도 남는 증거 파일
```

이라고 구분하면 된다.

---

# 26. `artifacts`와 `services/artifacts`도 다르다

코드에:

```tsx
const artifacts = {
  directory: runDirectory,
};
```

라는 변수도 있고,

```
services/artifacts
```

모듈도 있다.

둘은 다른 개념이다.

### `services/artifacts`

Artifact를 생성하고 저장하는 **기능 모듈**

```
createExecutionRun
getExecutionBaseCommit
preserveExecutionChanges
writeExecutionResult
```

---

### `artifacts`

현재 Execution에서 생성된 Artifact의 위치와 정보를 들고 있는 **결과 데이터**

예:

```
{
  directory: ".../run-X123",
  baseCommit: "a102..."
}
```

즉:

```
services/artifacts
→ Artifact를 만드는 코드

artifacts 변수
→ 만들어진 Artifact에 대한 정보
```

이다.

---

# 27. Architecture 관점에서 역할 구분

현재 구조를 크게 보면 다음과 같다.

```
Application / Execution
│
│ executeTask()
│ 전체 실행 흐름 제어
│
├─────────────────────────
│
├─ Infrastructure / Git
│
│  createExecutionWorkspace()
│  removeExecutionWorkspace()
│
│  → 실제 Git / Worktree 조작
│
├─────────────────────────
│
├─ Orchestration
│
│  runWorkerStage()
│  runScopeStage()
│  runVerificationStage()
│
│  → 실행 단계를 조율
│
├─────────────────────────
│
└─ Services / Artifacts

   createExecutionRun()
   getExecutionBaseCommit()
   preserveExecutionChanges()
   writeExecutionResult()

   → 실행 결과 보존
```

`executeTask()`는 각각에게 일을 시키는 **Orchestrator**에 가깝고,

Artifact Service는:

> 실행 결과를 안전하게 남겨라

라는 하나의 책임을 가진다.

---

# 28. 실제 예시

TaskContract가 다음과 같다고 하자.

```
{
  "id": "task-readme-001",
  "goalId": "goal-readme-001",
  "objective": "README 소개 문구를 수정한다",
  "allowedPaths": [
    "README.md"
  ],
  "verification": [
    "git diff --check"
  ]
}
```

Execution 시작:

```
executeTask()
```

### 1.

```
createExecutionRun()
```

결과:

```
run-X123/
└── task.json
```

### 2.

```
createExecutionWorkspace()
```

결과:

```
/tmp/ai-workspace-ABC
```

### 3.

```
getExecutionBaseCommit()
```

결과:

```
a102e78
```

### 4.

Worker 실행:

```
README.md 수정
```

### 5.

Scope 검사:

```
변경 파일:
README.md

allowedPaths:
README.md

→ 통과
```

### 6.

Verification:

```
git diff --check

→ 성공
```

### 7.

최종 Scope 검사:

```
README.md만 변경

→ 통과
```

### 8.

Artifact 보존:

```
preserveExecutionChanges()
```

결과:

```
run-X123/
├── task.json
├── base-commit.txt
└── changes.patch
```

### 9.

Worktree 삭제:

```
/tmp/ai-workspace-ABC
→ 삭제
```

### 10.

최종 결과 저장:

```
result.json
```

최종:

```
run-X123/
├── task.json
├── base-commit.txt
├── changes.patch
└── result.json
```

---

# 29. 이 설계의 핵심

현재 Execution의 핵심은 단순히:

```
AI가 코드를 수정한다.
```

가 아니다.

실제로는:

```
AI에게 격리된 공간을 제공한다.
        ↓
허용된 범위만 작업하게 한다.
        ↓
실제 검증을 돌린다.
        ↓
무엇을 했는지 증거를 남긴다.
        ↓
임시 실행 환경을 제거한다.
```

라는 구조다.

Artifact는 이 중:

```
무엇을 했는지 증거를 남긴다.
```

를 담당한다.

---

# 30. 한 문장으로 정리

### Workspace

> AI가 안전하게 작업하기 위한 임시 작업장.

### Evidence

> 실행 과정에서 실제로 관찰된 사실.

### Artifact

> 그 실행이 끝난 뒤에도 검토할 수 있도록 남긴 영구적인 증거.

### `executeTask()`

> Workspace 생성부터 Worker, Scope, Verification, Artifact 보존, Cleanup까지 전체 Execution Lifecycle을 조율하는 함수.

최종적으로 현재 Execution은 다음 철학을 가진다.

```
Task를 실행한다
    ↓
결과만 믿지 않는다
    ↓
실제 변경과 검증 결과를 수집한다
    ↓
그 증거를 Artifact로 남긴다
    ↓
그 후 임시 작업장을 제거한다
```

즉 현재 Artifact 시스템의 핵심 목적은 **AI가 “성공했다”고 말한 것을 믿는 것이 아니라, 무엇을 실제로 했는지 재현 가능하고 검토 가능한 형태로 남기는 것**이다.
