# 일반 수동 Workspace vs AI Execution Workspace

| 구분            | 일반 Worktree (Orca 수동 Workspace)             | AI Execution Workspace (Harness 자동 생성)                            |
| --------------- | ----------------------------------------------- | --------------------------------------------------------------------- |
| **명령어 예시** | 브랜치 지정 (예: `feature/something`)           | `git worktree add --detach ... HEAD`                                  |
| **상태**        | 특정 **Branch**에 연결됨                        | Branch 없이 커밋을 직접 가리키는 **Detached HEAD**                    |
| **사용 주체**   | **사람 (개발자)**                               | **AI Agent (Codex 등)**                                               |
| **주 목적**     | 지속적인 개발, 커밋, 추후 Merge                 | 단기 임시 작업, 검증, 결과물 확인 후 삭제                             |
| **장점 / 특징** | 작업 내용을 브랜치로 관리하며 이어 나갈 수 있음 | 원본 브(`main`)를 건드릴 위험 없이 안전하게 독립된 공간에서 실행 가능 |

## 핵심 차이점

### 브랜치 연결 여부

- **일반 Worktree:**
  - `main` 이나, `feat/my-task` 같은 브랜치에 부착되어 있다.
  - 그래서 코드를 수정하고 커밋하면 해당 브랜치에 차곡차곡 쌓인다.
- **Excution Workspace:**
  - 브랜치 없이 특정 커밋(ex: `dd0d669`)에 직접 연결(Detached HEAD)된다. 브랜치라는 개념이 없기 때문에 여기서 작업하다가 워크트리를 통째로 날려도 원본 브랜치에는 아무런 영향을 주지 않는다.

### 사용 목적의 차이

- 사람용:
  - “내가 이 기능을 계속 발전시켜서 나중에 PR(Pull Request)을 올리거나 머지해야지” 하는 개발 및 지속 작업용이다.
- AI 에이전트용:
  - “에이전트한테 잠깐 시켜서 파일이 제대로 생성되나 테스트해보고, 결과물만 쏙 빼낸 뒤 버리면 된임시 실행/검증 공간”이다.

## `/tmp/ai-workspace-manual-test` 는 무엇인가

`/tmp/ai-workspace-manual-test` 는 Git의 특별한 문법이 아니라, 새 Worktree를 실제 파일시스템 어디에 만들지 지정한 경로이다.

```bash
git worktree add --detach /tmp/ai-workspace-manual-test HEAD
```

이 명령은 다음 의미이다.

> 현재 Repository의 HEAD 커밋을 기준으로,
> 새로운 Detached Worktree를 만들고,
> 그 파일들을 /tmp/ai-workspace-manual-test 에 배치한다.

경로를 나누면:

```bash
/tmp
└── ai-workspace-manual-test
```

- `/tmp` : mapOs/Linux의 임시 파일 저장 공간
- `ai-workspace-manual-test` : 사용자가 임의로 저장한 worktree 폴더 이름

즉, `ai-workspace-manual-test` 라는 이름 자체에는 Git이나 Harness의 특별한 의미가 없다.

예를 들어, `git worktree add --detach ~/Desktop/test-worktree HEAD` 도 가능,

차이는 단순히 Worktree가 생성되는 위치뿐이다.

## 수동 Worktree 생성과 `createExecutionWorkspace()` 의 관계

`execution-workspace.ts`

```tsx
import { randomUUID } from "node:crypto";
import { execFileSync } from "node:child_process";
import { rmSync } from "node:fs";
import { tmpdir } from "node:os";
import { join, resolve } from "node:path";

export function createExecutionWorkspace(repositoryRoot: string): string {
  const workspacePath = join(
    tmpdir(),
    `ai-workspace-execution-${randomUUID()}`
  );

  try {
    execFileSync(
      "git",
      [
        "-C",
        repositoryRoot,
        "worktree",
        "add",
        "--detach",
        workspacePath,
        "HEAD",
      ],
      { encoding: "utf8", stdio: "pipe" }
    );
  } catch (error) {
    rmSync(workspacePath, { force: true, recursive: true });

    let detail = error instanceof Error ? error.message : String(error);

    if (
      typeof error === "object" &&
      error !== null &&
      "stderr" in error &&
      typeof error.stderr === "string" &&
      error.stderr.trim()
    ) {
      detail = error.stderr.trim();
    }

    throw new Error(`Failed to create execution workspace: ${detail}`, {
      cause: error,
    });
  }

  return resolve(workspacePath);
}
```

`git worktree add --detach /tmp/ai-workspace-manual-test HEAD` 이 명령어와 본질적으로 같은 작업을 한다.

`createExecutionWorkspace()` 내부에서는:

```tsx
execFileSync("git", [
  "-C",
  repositoryRoot,
  "worktree",
  "add",
  "--detach",
  workspacePath,
  "HEAD",
]);
```

를 실행하기에 이걸 터미널 명령어로 바꾸면:

```bash
git -C <repositoryRoot> worktree add --detach <workspacePath> HEAD
```

즉, `createExecutionWorkspace()` 는

> 사용이 직접 입력할 Git Worktree 명령을 Node.js 코드로 자동화한 함수

### 실제 Harness에서는 왜 UUID를 사용하는가

`createExcutionWorkspace()` 에서는 고저오딘 이름 대신:

```tsx
const workspacePath = join(tmpdir(), `ai-workspace-execution-${randomUUID()}`);
```

처럼 랜덤 UUID를 붙인다. 실제로은 이런 경로가 생성된다.

```
/tmp/ai-workspace-execution-11506a94-9cc8-4860-aa29-5ce67fa9e01b
```

이렇게 하는 이유는 여러 작업이 동시에 실행되어도 서로같은 폴더를 사용하지 않게 하기 위해서이다.

고정 경로라면:

```
Agent A → /tmp/ai-workspace
Agent B → /tmp/ai-workspace
```

처럼 충돌할 수 있기에.. UUID를 사용하여

```
Agent A
→ /tmp/ai-workspace-execution-a123...

Agent B
→ /tmp/ai-workspace-execution-b456...
```

처럼 각 실행이 독립된 공간을 갖게 한다.

## Execution Workspace의 핵심 목적

Execution Workspace를 만드는 이유는 단순히 Repository를 복사하기 위해서가 아니다.

핵심은:

> 원본 Repository와 Agent의 작업 공간을 분리하는 것

```
원본 Repository
main
│
│ 현재 HEAD commit을 기준으로
│
└───────────────┐
                ↓
Execution Workspace
Detached HEAD
```

Codex는 아래쪽 Exeution Workspace에서만 작업한다.

```
원본 main
→ 그대로 유지

Execution Workspace
→ Codex가 파일 수정
→ 테스트
→ Verification
```

그래서 Agent가 잘못된 수정을 하더라도 우선 원본 Working tree에는 직접 영향을 주지 않는다

### Detached HEAD와 임시 Workspace가 잘 맞는 이유

Execution Workspace는 장시간 개발할 공간이 아니다.

일반적인 사람의 작업은:

```
Branch 생성
→ 개발
→ Commit
→ Push
→ PR
→ Merge
```

의 흐름이 필요하다. 반면 Agent의 일회성 Worker 실행은:

```
현재 commit 기준 Workspace 생성
→ Agent 작업
→ 검증
→ 결과 확인
→ Workspace 제거
```

가 목적이다. 그래서 굳이 새로운 Branch를 생성하지 않고, `--detach` 를 이용해 특정 commit에서 바로 시작하는 것이 단순하다.

### 현재 AI Workspace의 단계

지금까지 만든 구조는 다음과 같다.

```
Human / CLI
    ↓
TaskContract
    ↓
Repository resolve
    ↓
createExecutionWorkspace()
    ↓
Detached Git Worktree
    ↓
Codex Worker
    ↓
Verification Runner
```

즉, Execution Workspace는 단순한 임시 폴더가 아니라,

> Repository Worker가 원본 코드와 분리된 상태에서 작업하고 검증받는 실행 경계

라고 볼 수 있다.
