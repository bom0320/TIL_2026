AI Workspace Task Execution Flow와 ExecutionState
====

## 왜 이 구조를 만들었는가

AI Workspace의 v0 목표는 Codex에게 단순히 코드를 수정시키는 것이 아니다.

```tsx
TaskContract
→ 격리된 Workspace 생성
→ Worker 실행
→ 변경 Evidence 수집
→ Scope 검사
→ Verification
→ 변경사항 보존
→ Workspace 정리
→ 최종 결과 기록
```

이라는 통제 가능한 실행 흐름을 만드는 것이 목적이다.

현재 이 흐름의 중심은 두 파일이다.

```
**task-executor.ts**
= 전체 실행 순서를 결정

**execution-stages.ts**
= 각 실행 단계의 실제 동작
```

- `task-excutor.ts` 는 세부 구현을 직접 수행하기보다 각 단계를 **어떤 순서로 호출**할지 결정한다.
- `execution-stages.ts` 는 Worker, Evidence, Scope, Verifiation 단계에서 실제로 무엇을 할지 구현한다.

## 전체 실행 흐름

현재 `executeTask()` 의 흐름은 다음과 같다.

```
executeTask()
    │
    ├─ ExecutionState 생성
    │
    ├─ Run Directory 생성
    │
    ├─ Git Worktree 생성
    │
    ├─ Base Commit 확보
    │
    ├─ Worker 실행
    │
    │    ├─ 실패 → Changed Paths 수집
    │    │
    │    └─ 성공
    │         ↓
    │       Scope 검사
    │         ↓
    │       Verification
    │         ↓
    │       Scope 재검사
    │
    ├─ changes.patch 보존
    │
    ├─ Worktree Cleanup
    │
    └─ ExecutionResult 확정
```

핵심은 Worker가 성공했다고 바로 Task 성공으로 판단하지 않는다는 것이다.

```
Worker 성공
≠
Task 성공
```

Worker가 정상적으로 끝난 뒤에도 실제 변경 범위와 Verification 결과를 확인해야한다.

## `ExecutionState`는 무엇인가

Task가 시작되면 가장 먼저 다음 상태를 만든다.

```tsx
const state: ExecutionState = {
  evidence: {},
  failures: [],
};
```

타입은 다음과 같다.

```tsx
export type ExecutionState = {
  evidence: ExecutionResult["evidence"];
  failures: ExecutionFailure[];
  scope?: ExecutionResult["scope"];
};
```

`ExecutionState`는 **Task 실행 도중 생기는 정보를 계속 누적하는 상태 객체**다.

처음에는:

```tsx
{
  evidence: {},
  failures: []
}
```

이다.

Worker가 성공하면:

```tsx
{
  evidence: {
    workerOutput: "..."
  },
  failures: []
}
```

변경된 파일을 수집하면:

```tsx
{
  evidence: {
    workerOutput: "...",
    changedPaths: [
      "src/a.ts"
    ]
  },
  failures: []
}
```

Scope 검사를 수행하면:

```tsx
{
  evidence: {
    workerOutput: "...",
    changedPaths: [
      "src/a.ts"
    ]
  },
  scope: {
    passed: true,
    violations: []
  },
  failures: []
}
```

Verification까지 수행하면:

```tsx
{
  evidence: {
    workerOutput: "...",
    changedPaths: [
      "src/a.ts"
    ],
    verification: {
      passed: true,
      commands: [...]
    }
  },
  scope: {
    passed: true,
    violations: []
  },
  failures: []
}
```

즉 하나의 `state` 객체가 실행이 진행될수록 계속 채워진다.

---

## 왜 `state`를 계속 반환하지 않는가

다음과 같이 호출한다.

```tsx
await runWorkerStage(task, state, workspaceRoot, options);
```

그리고 `runWorkerStage()` 내부에서는:

```tsx
state.evidence.workerOutput =
  await runCodexWorker(...);
```

처럼 직접 `state`를 수정한다.

그래서 다음처럼 다시 대입하지 않는다.

```tsx
state = await runWorkerStage(...);
```

JavaScript 객체는 참조로 전달되기 때문에 각 함수가 **같은 `state` 객체를 보고 수정**한다.

```
executeTask
    │
    └──── state ────────┐
                        │
                        ▼
                 runWorkerStage
                        │
                        └─ workerOutput 추가
                        ↓
                 runScopeStage
                        │
                        ├─ changedPaths 추가
                        └─ scope 추가
                        ↓
             runVerificationStage
                        │
                        └─ verification 추가
```

모든 Stage가 서로 다른 상태를 만드는 것이 아니라 **한 Task의 동일한 state 하나를 공유한다.**

---

## Worker Stage

`task-executor.ts`에서는 다음처럼 호출한다.

```tsx
const workerSucceeded = await runWorkerStage(
  task,
  state,
  workspaceRoot,
  options
);
```

실제 Worker 실행은 `execution-stages.ts`에 있다.

```tsx
state.evidence.workerOutput = await runCodexWorker(
  task,
  workspaceRoot,
  options.workerTimeoutMs
);
```

성공하면:

```tsx
return true;
```

실패하면:

```tsx
addExecutionFailure(state, "worker", error);

return false;
```

즉 Worker Stage의 역할은:

```
Codex 실행
→ 출력 저장
→ 성공 / 실패 반환
```

이다.

---

## Worker가 실패해도 Changed Paths를 수집하는 이유

`task-executor.ts`에는 다음 분기가 있다.

```tsx
if (!workerSucceeded) {
  collectChangedPathsEvidence(state, workspaceRoot, baseCommit);
}
```

Worker가 실패했다고 해서 아무 작업도 하지 않은 것은 아닐 수 있다.

예를 들어:

```
Codex 실행
↓
a.ts 수정
↓
b.ts 수정
↓
Timeout 발생
```

할 수 있다.

그래서 Worker 실패 여부와 상관없이:

> 실패하기 전까지 실제로 어떤 파일을 수정했는가

를 Evidence로 남긴다.

---

## Changed Paths Evidence

`collectChangedPathsEvidence()`는 실제 변경된 파일을 수집한다.

```tsx
state.evidence.changedPaths = collectChangedPaths(workspaceRoot, baseCommit);
```

성공하면:

```tsx
return true;
```

실패하면:

```tsx
addExecutionFailure(state, "evidence", error);

return false;
```

이다.

여기서 중요한 구분은 다음과 같다.

```
changedPaths
= 실제로 관찰된 사실

scope
= 그 사실을 기준으로 내린 판단
```

---

## Scope Stage

Worker가 성공하면 Scope 검사를 수행한다.

```tsx
runScopeStage(task, state, workspaceRoot, baseCommit);
```

먼저 최신 Changed Paths를 수집한다.

```tsx
collectChangedPathsEvidence(...)
```

그다음:

```tsx
const scope = checkScope(
  state.evidence.changedPaths ?? [],
  task.allowedPaths,
  task.forbiddenPaths
);
```

를 통해 TaskContract와 실제 변경 범위를 비교한다.

예:

```tsx
changedPaths = ["src/a.ts", "README.md"];

allowedPaths = ["src/a.ts"];
```

라면:

```tsx
scope = {
  passed: false,
  violations: ["README.md"],
};
```

가 된다.

그리고 결과를:

```tsx
state.scope = scope;
```

로 저장한다.

---

## Scope가 실패하면 Verification을 하지 않는 이유

`task-executor.ts`의 흐름은 다음과 같다.

```tsx
if (!workerSucceeded) {
  ...
} else if (
  runScopeStage(...)
) {
  await runVerificationStage(...);
}
```

즉 Scope가 `false`를 반환하면 Verification까지 가지 않는다.

```
Worker 성공
↓
Scope 실패
↓
Verification 실행 안 함
```

이미 TaskContract에서 허용한 변경 범위를 벗어났기 때문에 테스트 성공 여부를 확인하기 전에 실패로 본다.

---

## Verification Stage

Scope가 통과하면 Verification을 실행한다.

```tsx
await runVerificationStage(task, state, workspaceRoot, options);
```

내부에서는:

```tsx
state.evidence.verification = await runVerification(
  task.verification,
  workspaceRoot,
  options.verificationTimeoutMs
);
```

TaskContract에 정의된 Verification Command를 실제로 실행한다.

예:

```tsx
"verification": [
  "git diff --check",
  "pnpm test"
]
```

결과는 `state.evidence.verification`에 저장된다.

즉 Worker가:

```
테스트 통과했습니다
```

라고 말한 것을 그대로 믿는 것이 아니라 Harness가 실제 명령을 다시 실행한다.

```
Worker 주장
≠
Harness Evidence
```

---

## 왜 Scope를 두 번 검사하는가

Verification이 끝난 뒤 다시 Scope를 검사한다.

```tsx
runScopeStage(task, state, workspaceRoot, baseCommit);
```

이유는 Verification Command도 Repository를 수정할 수 있기 때문이다.

예를 들어:

```tsx
eslint --fix
```

같은 명령이 있다면:

```
Worker 종료
↓
1차 Scope 검사
↓
Verification 실행
↓
파일 추가 수정 가능
↓
2차 Scope 검사
```

가 필요하다.

즉:

```
1차 Scope
= Worker가 범위를 지켰는가

2차 Scope
= Verification까지 끝난 최종 상태가 범위를 지켰는가
```

를 확인한다.

---

## Failure도 `state`에 누적된다

공통 실패 기록은:

```tsx
addExecutionFailure(state, stage, error);
```

형태로 수행된다.

내부에서는:

```tsx
state.failures.push({
  stage,
  message: errorMessage(error),
});
```

한다.

예:

```tsx
{
  stage: "worker",
  message: "Codex worker timed out"
}
```

처럼 실행 단계별 실패를 남긴다.

따라서 `state`에는 성공 Evidence뿐 아니라 실패 정보도 함께 누적된다.

---

## `ExecutionState`에서 `ExecutionResult`로

모든 실행 과정이 끝나면:

```tsx
return finalizeExecutionResult(state, artifacts, retainedWorkspace);
```

를 호출한다.

여기서 지금까지 누적된 `ExecutionState`를 최종 `ExecutionResult`로 만든다.

```
ExecutionState
= 실행 중 계속 변하는 상태

ExecutionResult
= 실행이 끝난 뒤 확정된 결과
```

라고 보면 된다.

---

## 핵심 흐름

```
TaskContract
↓
ExecutionState 생성
↓
Worker
→ workerOutput 누적
↓
Changed Paths
→ changedPaths 누적
↓
Scope
→ scope 누적
↓
Verification
→ verification 누적
↓
Failure 발생 시
→ failures 누적
↓
ExecutionResult 확정
```

### 정리

`task-executor.ts`는 **언제 어떤 Stage를 실행할지 결정**하고, `execution-stages.ts`는 **각 Stage에서 하나의 `ExecutionState`를 계속 갱신**한다.

즉 이 구조의 핵심은:

> 하나의 Task가 실행되는 동안 발생한 사실, 판단, 실패를 하나의 `ExecutionState`에 계속 누적하고, 마지막에 이를 `ExecutionResult`로 확정하는 것이다.
