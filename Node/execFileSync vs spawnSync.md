# execFileSync vs spawnSync

## 둘 다 뭐하는 얜지

둘 다 Node.js가 **외부 프로그램을 실행하게 해주는 기능**이다.

내 프로그램은 TS/Node로 돌아가는데 중간에 이런 걸 실행해야 한다.

```bash
git worktree add ...
codex exec ...
pnpm typecheck
pnpm test
```

Node.js 자신이 Git이나 pnpm 기능을 갖고 있는 건 아니다.

그래서..

```
Node 프로그램
   ↓
child_process
   ↓
git / codex / pnpm 같은 외부 프로그램 실행
```

해야 하는 것이다.

## execFileSync

`execution-workspace.ts` 에서는 이런 식이였다.

```tsx
execFileSync(
  "git",
  ["-C", repositoryRoot, "worktree", "add", "--detach", workspacePath, "HEAD"],
  {
    encoding: "utf8",
    stdio: "pipe",
  }
);
```

```tsx
execFileSync(실행할_프로그램, 프로그램에_전달할_인자들, 실행_옵션);
```

즉, 이 코드는 “git” 이라는 프로그램을 실행하고,

```tsx
["-C", repositoryRoot, "worktree", "add", "--detach", workspacePath, "HEAD"];
```

를 Git 한테 인자로 넘기는 것이다. 즉 터미널로 치면..

```bash
git -C <repositoryRoot> worktree add --detach <workspacePath> HEAD
```

이 방식은 실행할 프로그램(git)과, 각 인자들(-C, repositoryRoot, worktree…) 을 정확히 알고 있다.

그래서 “Git이라는 프로그램 실행할 거니까, 여기 인자들 하나씩 줄께” 라는 방식을 `execFileSync("git", ["-C", ...])` 을 써서 표현하는게 자연스러움.

### Codex Worker도 똑같다.

`codex-worker.ts`

```tsx
execFileSync("codex", ["exec", "--sandbox", "workspace-write", prompt], {
  cwd: workspaceRoot,
  encoding: "utf8",
  stdio: ["ignore", "pipe", "pipe"],
});
```

```
프로그램
= codex

인자
= exec
= --sandbox
= workspace-write
= prompt
```

```bash
codex exec --sandbox workspace-write "<prompt>"
```

여기도 마찬가지로 프로그램 구조가 확정되어 있다

> 내 Hareness가 Codex를 실행한다.

라고 결정했으니, `execFileSync` 를 쓰는게 맞다.

## 그런데 Verification Runner는 상황이 다르다. \_spawnSync

`verification-runner.ts`

```tsx
import { spawnSync } from "node:child_process";

export type VerificationCommandResult = {
  command: string;
  passed: boolean;
  exitCode: number | null;
  stdout: string;
  stderr: string;
};

export type VerificationResult = {
  passed: boolean;
  commands: VerificationCommandResult[];
};

export function runVerification(
  verification: string[],
  workspaceRoot: string
): VerificationResult {
  const commands = verification.map((command) => {
    const result = spawnSync(command, {
      cwd: workspaceRoot,
      encoding: "utf8",
      shell: true,
    });
    const stderr = [result.stderr, result.error?.message]
      .filter((value): value is string => Boolean(value))
      .join("\n");

    return {
      command,
      passed: result.status === 0 && result.error === undefined,
      exitCode: result.status,
      stdout: result.stdout ?? "",
      stderr,
    };
  });

  return {
    passed: commands.every((command) => command.passed),
    commands,
  };
}
```

Verification Runner 입력 부분을 봐보자.

```tsx
verification: string[]
```

실제 TaskContract:

```json
["pnpm typecheck", "pnpm test", "pnpm dev ./tasks/example.json ."]
```

여기서 문제가 생긴다. Verification Runner 입장에서는 실행 프로그램이 뭔지 모른다.

- 첫 번째는 pnpm typecheck
- 두 번째는 pnpm test
- 나중에는 npm run build 또는 git diff —check 같은 것도 들어올 수 있다.

즉, Runner는 “git을 실행한다” 처럼 흐로그램이 미리 정해진 게 아니다.

> “TaskContract에 들어있는 명령 문자열을 실행하라.”

그래서 지금 코드가:

```tsx
spawnSync(command, {
  cwd: workspaceRoot,
  encoding: "utf8",
  shell: true,
});
```

### 여기서 `shell: true`

예를 들어 `command = "pnpm typecheck"` 를 했을때.. → 컴퓨터 입장에서는 이게 그냥 문자 하나 “pnpm type check” but 원하는 값은

```
프로그램 = pnpm
인자 = typecheck
```

그래서 Shell에게 넘긴다.(`shell: true` ) 그러면..다음처럼 된다.

```
Node
 ↓
Shell
 ↓
"pnpm typecheck" 해석
 ↓
pnpm 프로그램 실행
 ↓
typecheck 전달
```

### Shell이 정확이 뭐냐

터미널이랑 헷갈릴 수 있는데 구분하자.

터미널은 입출력을 보여주는 창에 가깝고, Shell은:

> 내가 입력한 명령어를 해석해서 프로그램을 실행하는 프로그램이다.

맥이면 보통: `zsh` 같은 Shell을 쓴다.

즉, 터미널에서 `pnpm test` 라고 쓰면..

```
나
 ↓
Terminal
 ↓
zsh
 ↓
pnpm 실행
 ↓
test 인자 전달
```

대략 이런 구조이다. `spawnSync(..., {shell: true})` 는 Node가 그 과정을 대신하는 것

### 그러면 `exeFileSync` 도 shell 쓰면 되는거 아닌가?

기술적으로는 여러 선택지가 있다. 하지만.. 굳이 Shell 이 필요하지 않는 경우에는 안 쓰는게 좋다.

즉, Git 명령

```tsx
execFileSync("git", ["-C", repositoryRoot, "worktree", "add"]);
```

와 같은 경우에는 Node가 직접 “git 실행”을 하면 되니까 중간에 Shell을 낄 이유가 없다. Node ⇒ Git 이면 충분

반대로, Verification은

```tsx
Node
 ↓
Shell
 ↓
TaskContract 명령 해석
```

가 필요하다.

## 둘을 비교하면

### Execution Workspace

```
Node

 ↓ execFileSync

git

 ↓ args

-C repo
worktree
add
...
```

### Codex Worker

```
Node

 ↓ execFileSync

git

 ↓ args

-C repo
worktree
add
...
```

### Verification Runner

```
Node

 ↓ spawnSync + shell:true

Shell

 ↓ 문자열 해석

"pnpm test"

 ↓

pnpm
  └─ test
```

## Child Process와 동기 실행 (`spawnSync`)

- `spawn` 은 새 프로세스를 만들어 외부 프로그램을 실행한다는 뜻이다.
- `ai-workspace` 가 부모 프로세스라면 `pnpm test`, `git`, `codex` 등은 자식 프로세스로 실행되낟.

```
AI Workspace (Parent Process)
   ├─ git
   ├─ codex
   ├─ pnpm typecheck
   └─ pnpm test
```

즉, Harnes가 Git, TypeScript, Vitest 등을 직접 구현하는 것이 아니라, **외부 도구를 실행하고 결과를 조율하는 역할**을 한다.

`spawnSync` 의 `Sync` 은 **동기 실행**을 의미한다. 명령이 끝날 때까지 현재 코드 실행이 멈추고 기다린다.

```
pnpm test 실행
↓
완료될 때까지 대기
↓
결과 반환
↓
다음 코드 실행
```

반대로, `spawn()` 은 비동기 방식이라 명령어 실행 중이어도 다음 코드가 먼저 실행될 수 있다. 여러 Agent나 Repository를 병렬 처리할 때는 비동기 방식이 유용할 수 있지만, 현재 V0에서는 실행 순서와 상태 관리가 단순한 동기 방식이 적절하다.

현재 Verification Runner도:

```tsx
verification.map((command) => {
  spawnSync(command);
});
```

형태이므로 실제 실행은:

```tsx
pnpm typecheck
↓ 완료 대기
pnpm test
↓ 완료 대기
pnpm build
↓ 완료 대기
```

순서로 진행된다.

`spawnSync()` 가 끝날 때까지 Node 실행 흐름을 멈추는 것을 **Blocking**이라고 한다. 지금은 검증 명령을 하나씩 확실하게 실행하고 결과를 수집해야 하므로 의도적으로 Blocking 방식을 사용하고 있다.

## 요약

> 실행할 프로그램과 인자를 Harness가 알고 있으면 `execFileSync`, TaskContract가 제공한 완성된 shell command 문자열을 실행해야 해서 Verification에서는 `spawnSync + shell: true`를 사용.
