# v1 E2E Troubleshooting

## 1. Execution Worktree에서 `pnpm typecheck`, `pnpm test` 실패

### 문제

v1 E2E 테스트에서 Planning → Human Approval → Execution까지는 정상적으로 진행됐지만,
Verification 단계에서 아래 명령이 실패했다.

```bash
pnpm typecheck
pnpm test
```

실제 오류는 다음과 같았다.

```
Cannot find type definition file for 'node'
Cannot find package 'vitest'
Local package.json exists, but node_modules missing
```

### 원인

Execution은 원본 Repository를 직접 수정하지 않고,

격리된 Git worktree에서 수행한다.

```
Original Repository
  ↓
git worktree
  ↓
Execution Workspace
```

하지만 `node_modules`는 Git tracked file이 아니기 때문에

새로 생성된 worktree에는 포함되지 않는다.

즉 원본 Repository에는 dependency가 설치되어 있어도:

```
original/
├─ package.json
├─ src/
└─ node_modules/

execution-worktree/
├─ package.json
├─ src/
└─ node_modules/   ← 없음
```

상태가 되었다.

그래서 worktree 안에서 실행된 `pnpm typecheck`, `pnpm test`가

필요한 dependency를 찾지 못하고 실패했다.

### 해결

현재 v1 범위에서는 자동으로 `pnpm install`을 실행하지 않고,

원본 Repository에 이미 설치된 dependency를 Execution Workspace에서 재사용하도록 했다.

구조:

```
Original Repository
└─ node_modules/
      ↑
      │ symlink
      │
Execution Workspace
└─ node_modules/
```

Execution Workspace 생성 후

원본 Repository의 `node_modules` 항목들을 symlink로 연결했다.

이를 통해:

- 원본 dependency 환경 재사용
- 불필요한 재설치 방지
- 자동 dependency installation 범위 확장 방지

가 가능해졌다.

원본 Repository에 `node_modules`가 없는 경우에는

별도 설치를 수행하지 않고 기존 동작을 유지한다.

### 결과

수정 후 동일한 E2E 테스트를 다시 실행한 결과:

```
Codex Worker: complete
Verification: passed
Changed Paths: 1
```

까지 정상적으로 통과했다.

---

## 2. Planner가 Verification에 자연어를 생성한 문제

### 문제

Planning 결과의 `verification` 항목에 다음과 같은 값이 생성됐다.

```
git diff -- README.md로 변경 범위와 문구의 정확성을 검토한다.
```

이 값은 shell command처럼 보이지만 실제로는 자연어가 섞여 있다.

Verification Runner는 `verification` 배열의 모든 값을

실제 shell command로 실행하기 때문에,

Planner가 설명 문장을 넣으면 잘못된 검증이 수행될 수 있다.

더 큰 문제는 이런 문자열이 항상 명확하게 실패하지 않는다는 점이다.

예를 들어 Git이 뒤쪽 문자열을 pathspec으로 해석하면

실질적인 검증을 하지 않았는데도 exit code 0이 반환될 수 있다.

즉:

```
command passed
≠
meaningful verification passed
```

문제가 생길 수 있었다.

### 원인

기존 Planner prompt에는

`verification`이 반드시 실행 가능한 shell command여야 한다는 제약이 충분히 강하지 않았다.

Planner는:

```
검증 방법
```

과

```
실행할 command
```

를 같은 것으로 해석할 수 있었다.

### 해결

Codex Planner prompt에 다음 제약을 명시적으로 추가했다.

```
Verification entries MUST be executable shell commands.
Do not put prose, explanations, or natural-language instructions in verification.
Each entry must be runnable exactly as written.
```

올바른 예:

```
git diff --check
pnpm typecheck
pnpm test
```

잘못된 예:

```
README 변경 내용을 검토한다
git diff로 변경 내용을 확인한다
git diff -- README.md로 변경 범위를 확인한다
```

즉 `verification`은:

```
"무엇을 확인할지"
```

가 아니라

```
"실제로 실행할 명령"
```

만 포함하도록 역할을 명확히 했다.

### 결과

수정 후 생성된 TaskContract는 다음과 같이 나왔다.

```
"verification": [
  "pnpm typecheck",
  "pnpm test",
  "git diff --check"
]
```

모두 실제 실행 가능한 command였고,

E2E Verification도 정상 통과했다.

---

## 3. 이번 Troubleshooting에서 확인한 점

이번 E2E 테스트를 통해

단순히 Planning과 Execution이 연결됐는지만 확인한 것이 아니라,

격리 실행 환경과 Verification Contract의 실제 문제를 발견했다.

최종적으로 다음 흐름이 정상 동작했다.

```
Human Goal
↓
Repository Inspection
↓
Planning
↓
TaskContract
↓
Human Approval
↓
Execution Workspace
↓
Codex Worker
↓
Scope Check
↓
Verification
↓
Artifacts
↓
Result
```

특히 다음 두 가지를 확인했다.

1. 격리된 Execution Workspace는 코드만 분리해서는 충분하지 않고,

   검증에 필요한 실행 환경도 함께 고려해야 한다.

2. Agent가 생성하는 Verification은

   자연어 설명이 아니라 기계가 그대로 실행할 수 있는 명령으로 제한해야 한다.

이 두 문제를 해결한 뒤

v1의 Goal → Planning → Approval → Execution E2E 흐름이 정상적으로 통과했다.
