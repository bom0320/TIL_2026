# AI Workspace의 Command Runner와 Process Lifecycle

**Code**

```tsx
import { spawn } from "node:child_process";

export type CommandResult = {
  stdout: string;
  stderr: string;
  exitCode: number | null;
  timedOut: boolean;
  error?: Error;
};

type CommandOptions = {
  cwd: string;
  timeoutMs: number;
  shell?: boolean;
};

const FORCE_KILL_DELAY_MS = 250;

export function runCommand(
  command: string,
  args: string[],
  options: CommandOptions
): Promise<CommandResult> {
  return new Promise((resolve) => {
    const child = spawn(command, args, {
      cwd: options.cwd,
      shell: options.shell ?? false,
      detached: process.platform !== "win32",
      stdio: ["ignore", "pipe", "pipe"],
    });
    let stdout = "";
    let stderr = "";
    let spawnError: Error | undefined;
    let timedOut = false;
    let closed = false;
    let forceKillCompleted = false;
    let exitCode: number | null = null;

    const killProcessTree = (signal: NodeJS.Signals): void => {
      if (child.pid === undefined) {
        return;
      }

      try {
        if (process.platform === "win32") {
          child.kill(signal);
        } else {
          process.kill(-child.pid, signal);
        }
      } catch (error) {
        if (
          !(error instanceof Error && "code" in error && error.code === "ESRCH")
        ) {
          spawnError ??=
            error instanceof Error ? error : new Error(String(error));
        }
      }
    };

    const finish = (): void => {
      if (!closed || (timedOut && !forceKillCompleted)) {
        return;
      }

      resolve({
        stdout,
        stderr,
        exitCode,
        timedOut,
        ...(spawnError === undefined ? {} : { error: spawnError }),
      });
    };

    child.stdout.on("data", (chunk: Buffer | string) => {
      stdout += chunk.toString();
    });
    child.stderr.on("data", (chunk: Buffer | string) => {
      stderr += chunk.toString();
    });
    child.once("error", (error) => {
      spawnError = error;
    });
    child.once("close", (code) => {
      exitCode = code;
      closed = true;
      finish();
    });

    const timeout = setTimeout(() => {
      timedOut = true;
      killProcessTree("SIGTERM");

      setTimeout(() => {
        killProcessTree("SIGKILL");
        forceKillCompleted = true;
        finish();
      }, FORCE_KILL_DELAY_MS);
    }, options.timeoutMs);

    child.once("close", () => {
      clearTimeout(timeout);
    });
  });
}
```

## 왜 `command-runner.ts` 가 필요한가

AI Workspace의 Worker와 Verification은 결국 외부 프로그램을 실행한다.

Worker에서는: `codex exec …` 같은 명령을 내리고,

Verification에서는:

```
pnpm test
git diff --check
```

같은 명령을 내린다. 이 명령들을 각각의 모듈이 직접 실행하면 timeout, stdout 수집, process 종료 방식이 서로 달라질 수 있다. 그래서 외부 Command 실행 규칙을 하나로 모은 것이 `command-runner.ts` 이다.

```
Worker
↓
runCommand()

Verification
↓
runCommand()
```

즉, `command-runner.ts` 는:

> **Harness가 외부 OS Process를 실행하고 제어하기 위한 공통 실행 계층**

## 전체 흐름

```
runCommand()
    ↓
spawn()
    ↓
Process 생성
    ↓
stdout / stderr 수집
    ↓
종료 대기
    │
    ├─ 정상 종료
    │    ↓
    │  exitCode 저장
    │
    └─ timeout
         ↓
       SIGTERM
         ↓
       250ms 대기
         ↓
       SIGKILL
    ↓
CommandResult 반환
```

## 주요 타입 및 설정

### `CommandResult`

명령어가 끝난 뒤 반환할 결과물의 구조

```tsx
export type CommandResult = {
  stdout: string;
  stderr: string;
  exitCode: number | null;
  timedOut: boolean;
  error?: Error;
};
```

`CommandResult` 는 외부 Command 하나를 실행한 결과를 표현한다.

- stdout = 정상 출력
- sterr = 에러 출력
- exitCode = process 종료 코드
- timedOut = timeout이 발생했는지
- error = Process 실행 자체에서 발생한 오류

정상적인 경우,

```tsx
{
  stdout: "Tests passed",
  stderr: "",
  exitCode: 0,
  timedOut: false
}
```

처럼 반환 될 수 있다.

### `exitCode` 와 `error` 차이

둘의 의미가 다르다. 예를 들어, `pnpm test` 명령 자체는 실행됐지만 테스트가 실패했다면:

```tsx
{
  exitCode: 1,
  timedOut: false
}
```

일 수 있다. 즉,

- Process 실행은 성공
- 내부 작업은 실패

반대로, 시스템에서 실행 파일 자체를 찾을 수 없다면 `ENOENT` 같은 오류가 발생할 수 있다. 이 경우에는, “Process 실행 자체 실패”이므로 error에 저장된다.

### `CommandOptions`

명령어를 실행시킬 때 필요한 옵션

```tsx
type CommandOptions = {
  cwd: string;
  timeoutMs: number;
  shell?: boolean;
};
```

- `cwd`
  ```tsx
  cwd: options.cwd;
  ```
  Command가 실행될 기준 Directory다. AI Workspace에서는 보통 격리된 Worktree 경로가 들어간다.
  - 원본 Repository (X)
  - Execution Worktree (O)
    따라서, Codex와 Verification이 원본 Repo가 아니라, 격리된 환경에서 실행된다.
- `timeoutMs`
  ```tsx
  timeoutMs: number;
  ```
  Command가 실행될 수 있는 최대 시간이다. Agent나 테스트가 무한 대기 상태에 빠지는 것을 방지한다.
- `Shell`
  ```tsx
  shell?: boolean;
  ```
  기본값은 false,
  ```tsx
  shell: options.shell ?? false;
  ```
  Worker처럼 Command와 Arguments를 명확히 분리할 수 있다면 Shell 없이 직접 진행한다.
  반면,
  ```tsx
  pnpm test && git diff --check
  ```
  처럼 shell 문법을 사용하는 Verification Command는 `shell: true` 가 필요할 수 있다.

### `FORCE_KILL_DELAY_MS = 250`

프로세스를 우아하게 종료(`SIGTERM`)시킨 후, 완전히 죽지 않을 때 강제 종료(`SIGKILL`)로 넘어가기까지 기다리는 유예 시간(0.25초)이다.

## 핵심 도작 흐름 (`runCommand` )

이 함수는 `Promise` 를 반환하므로 async/await와 함께 사용할 수 있다. 내부 동작은 크게 3가지 축으로 들어간다.

#### 왜 Promise를 사용해?

`spawn()` 은 Process가 끝날 때까지 기다린 뒤 결과를 반환하는 함수가 아니다. Process를 시작한 뒤 이벤트를 통해 결과를 전달한다.

```tsx
spawn
↓
Process 실행
↓
stdout data
↓
stderr data
↓
close
```

그래서 Promise로 감싸, 다음처럼 사용할 수 있게 만든다.

```tsx
const result = await runCommand(...)
```

### 1. 자식 프로세스(Child Process) 생성

#### spawn(command, args, options) 함수가 받는 3가지 인자

| **인자명** | **타입** | **핵심 역할** | **예시 (`git commit -m "init"`)** |
| ---------- | -------- | ------------- | --------------------------------- |

| **`command`**
_(첫 번째, 필수)_ | `string` | **실행할 프로그램(실행 파일)의 이름**
• 명령어의 가장 첫 단어만 입력
• 시스템 환경 변수(PATH)나 절대 경로에서 찾음 | `'git'` |
| **`args`**
_(두 번째, 선택)_ | `string[]` | **프로그램에 전달할 옵션/인자 목록**
• 공백을 기준으로 쪼개서 배열로 전달
• 하나의 덩어리 옵션은 공백이 있어도 배열 한 칸에 넣음 | `['commit', '-m', 'init']` |
| **`options`**
_(세 번째, 선택)_ | `Object` | **실행 환경 및 제어 설정**
• `cwd`: 명령어가 실행될 폴더 경로
• `shell`: 쉘 환경(cmd, bash) 사용 여부
• `stdio`: 입출력 스트림 연결 방식 (pipe 등) | `{ cwd: '/my-project' }` |

여기가 실제 os Process가 실행되는 부분이다.

```tsx
const child = spawn(command, args, {
  cwd: options.cwd,
  shell: options.shell ?? false,
  detached: process.platform !== "win32", // Windows가 아니면 프로세스 그룹 분리
  stdio: ["ignore", "pipe", "pipe"], // stdin은 무시, stdout/stderr은 버퍼링용 파이프 연결
});
```

- stdio:

  ```tsx
  stdio: ["ignore", "pipe", "pipe"];

  // 순서: stdin, stdout sterr
  ```

  Harness가 Process에 사용자 입력을 직접 넣지는 않지만, Process가 출력하는 내용은 수집한다.

### 2. 출력 데이터 수집 및 상태 관리

#### stdout / stderr 수집

- `stdout` / `stderr`: 프로그램이 실행되면서 뱉너내는 텍스트 조각들을 문자열에 계속 이어붙여서 누적한다. → 즉, Process의 전체 출력 로그를 최종 `CommandResult` 에 담을 수 있다.

  ```tsx
  child.stdout.on("data", (chunk: Buffer | string) => {
    stdout += chunk.toString();
  });

  child.stderr.on("data", (chunk: Buffer | string) => {
    stderr += chunk.toString();
  });
  ```

#### 에러 발생 시, spawn에 기록

- 프로세스 생성 실패나 실행 중 에러가 발생하면 `spanwnError` 에 기록한다.
- process가 정상적으로 먼저 종료되면 예약되어 있던, Timeout을 제거한다. 그러지 않으면 Process가 이미 끝난 뒤에도 timeout callbak이 실행될 수 있다.

  ```tsx
  child.once("error", (error) => {
    spawnError = error;
  });

  child.once("close", () => {
    clearTimeout(timeout);
  });
  ```

### 3. Command 내부 상태

이 변수들을 명령어의 내부 상태(State)라고 부르는 이유는, 외부 명령어가 실행되어 끝날 때까지 ‘현재 어떤 단계가 있고 어떤 일이 일어났는지’를 실시간으로 기록하고 기억하는 저장소이기 때문이다.

```tsx
// 이 변수들은 Commad 하나의 실행 상태

let spawnError: Error | undefined; // 프로그램 실행이 실패했는가
let timedOut = false; // 시간 초과가 발생했는가
let closed = false; // 프로그램과 입출력 스트림이 완전히 종료되었는가
let forceKillCompleted = false; // 강제 종료(SIGKILL)까지 확실히 끝났는가
let exitCode: number | null = null; // 프로그램이 어떤 성적으로 종료되었는가
```

`spawn` 으로 실행한 외부 명령어는 실행 즉시 끝나는 게 아니라, 시간이 걸리는 **비동기 작업**이다. 따라서 이 변수들이 없으면 프로그램은 명령어가 성공했는지, 에러가 났는지, 타임아웃에 걸려 강제로 죽었는지를 알 방법이 없다.

```
ExecutionState
= Task 하나 전체

Command 내부 상태
= 외부 Process 하나
```

#### 정상 종료 처리

Process가 끝나면,

```tsx
child.once("close", (code) => {
  exitCode = code;
  closed = true;
  finish();
});
```

를 실행한다. Code를 저장하고, `closed = true` 로 Process가 종료됐음을 표시한다.

### 4. 프로세스 트리 종료 (핵심 로직)

`killProcessTree` 함수는 타임아웃이 발생했을 때, 실행 중인 프로그램뿐만 아니라 그 프로그램이 배경에서 몰래 띄운 하위 프로그램(자식의 자식들)까지 한 마리도 남김없이 싹 다 청소하는 함수이다.

```tsx
const killProcessTree = (signal: NodeJS.Signals): void => {
  if (child.pid === undefined) return;
  try {
    if (process.platform === "win32") {
      child.kill(signal); // Windows는 자식만 종료
    } else {
      process.kill(-child.pid, signal); // POSIX는 그룹 전체(-pid)를 종료
    }
  } catch (error) { ... }
};

```

#### 왜 그냥 `child.kill()` 을 안 쓰고 이 함수를 만들었나?

일반적으로 `child.kill()` 을 쓰면 가장 대가리 프로그램만 죽고, 그 밑에 상주하던 하위 프로그램들은 죽지 않은 채 메모리에 좀비처럼 남아 계속 돌아가는 현상이 발생한다. 이를 막기 위해 “가족 전체”를 찾아서 죽이는 특수한 로직이 필요한 것이다.

#### 분석하기

> “맥/리눅스 환경에서 마이너스 PID(-child.pid) 문법을 사용해, 자식 프로세스가 뒤에서 양산하 ㄴ하위 프로세스들까지 통째로 동반 자살 시키는 안전장치 코드”

- `if (child.pid === undefined) return;`
  - 프로그램이 애초에 안 켜졌거나 이미 죽어서 주민번호(PID)가 없다면 아무것도 하지 않고 종료한다.
    - **child.pid ( 프로세스 주민등록번호) :** 내가 방금 실행한 자식 프로세스의 고유 ID 번호 (Process ID)이다. 컴퓨터에게 방금 켠 그 프로그램 좀 꺼줘라고 말할 때, 정확한 타겟을 지정하기 위해 이 번호(pid)를 사용한다.
- `if (process.platform === "win32") { child.kill(signal); }`
  - 운영체제가 윈도우(Windows)일 때 작동한다. 윈도우는 아래의 ‘그룹 종료’ 기능이 지원되지 않아서 우선 눈 앞의 자식 프로세스 하나만 먼저 죽인다.
    - process.platform (운영체제 확인기) : 현재 이 Node.js 코드가 어떤 운영체제(OS) 위에서 돌아가고 있는지 알려주는 문자열 변수 (ex. 윈도우: “win32”, 맥: “darwin”, 리눅스 : “linux”)
      - 윈도우와 맥/리눅스는 프로그램을 강제 종료하는 내부 방식이 완전히 다르기 때문에 분기 처리를 해줘야 한다.
    - **child.kill()** : Node.js가 내가 직접 생성한 자식 프로세스(child)에게 “이제 그만 종료해”라고 신호(Signal)를 보내는 메서드이다.
      - 이 메서드는 내가 spawn으로 실행했던 바로 그 대가리 프로그램 하나만 조준해서 종료 신호를 보낸다.
- **`else { process.kill(-child.pid, signal); }`** (★ 가장 핵심이자 생소한 부분)
  - 운영체제가 맥(Mac)이나 리눅스(Linux)일 때 작동한다.
  - PID 마이너스(`-`) 기호를 붙여서 던지면, OS는 “이 번호를 대장으로 하는 프로세스 그룹 전체를 한 번에 폭ㅍ해라”하는 특수 명령으로 알아듣는다. 덕분에 하위 좀비 프로세스까지 깔끔하게 한 번에 죽는다.
    - process.kill() : Node.js가 운영체제(OS)에게 “컴퓨터야, 내가 불러주는 PID를 가진 녀석들 좀 강제로 죽여줘”라고 요청하는 메서드이다.
      - 왜 child.kill 대신 이걸 썼냐면, child.kill은 그룹 전체를 죽이는 기능이 없다. 반면 OS 청부업자인 `process.kill(-PID, ...)` 은 앞에 마이너스를 붙여서 주면 “이 번호랑 엮인 패거리(프로세스 그룹) 전체를 싹 날려줘”라는 특수 청부가 가능하기 때문에 이 메서드를 사용한 것이다.

### 5. `finish()` 역할

`finish()`는 **Command 결과를 언제 최종 반환할지 판단하는 함수**다.

```tsx
const finish = (): void => {
  if (!closed || (timedOut && !forceKillCompleted)) {
    return;
  }

  resolve({
    stdout,
    stderr,
    exitCode,
    timedOut,
    ...(spawnError === undefined ? {} : { error: spawnError }),
  });
};
```

조건은 두 가지다.

```
1. Process가 실제로 종료됐는가? → closed
2. timeout이 났다면 강제 종료 절차까지 끝났는가? → forceKillCompleted
```

즉 정상 종료라면 `close` 이후 바로 결과를 반환하고, timeout이라면:

```
timeout
→ SIGTERM
→ 250ms
→ SIGKILL
→ finish()
→ CommandResult 반환
```

순서로 끝난다.

---

#### 왜 `clearTimeout()`을 하는가

```tsx
child.once("close", () => {
  clearTimeout(timeout);
});
```

Process가 timeout 전에 정상 종료됐다면 예약해둔 timeout 처리를 취소한다.

안 하면 이미 끝난 Process에 나중에 `SIGTERM`, `SIGKILL`을 보내는 불필요한 동작이 생길 수 있다. 붙여넣은 텍스트(1)

---

#### `ExecutionState`와 차이

```
ExecutionState
= Task 전체 실행 상태

CommandResult
= Command 하나의 실행 결과
```

즉:

```
Task
 ├─ Codex Command
 ├─ pnpm test Command
 └─ git diff --check Command
```

각 Command마다 `CommandResult`가 생기고, 그 결과들이 상위에서 `ExecutionState`의 Evidence로 들어간다.

---

### 6. Timeout 전체 흐름

실제 코드는 다음과 같다.

```tsx
const timeout = setTimeout(() => {
  timedOut = true;
  killProcessTree("SIGTERM");

  setTimeout(() => {
    killProcessTree("SIGKILL");
    forceKillCompleted = true;
    finish();
  }, FORCE_KILL_DELAY_MS);
}, options.timeoutMs);
```

이를 시간 순서대로 보면:

```
runCommand 시작
↓
options.timeoutMs 동안 기다림
↓
시간 초과
↓
timedOut = true
↓
SIGTERM
↓
250ms 대기
↓
SIGKILL
↓
forceKillCompleted = true
↓
finish()
```

여기서 중요한 점은 **처음부터 SIGKILL을 보내지 않는 것**이다.

먼저:

```
SIGTERM
```

으로 종료할 기회를 주고,

그래도 확실하게 Process Tree를 정리하기 위해:

```
SIGKILL
```

까지 수행한다.

즉 Harness는:

> “시간 초과니까 당장 죽여”

보다는

> “먼저 종료를 요청하고, 짧은 유예 시간 이후에도 남아 있을 가능성에 대비해 강제 종료한다”

라는 정책을 가진다.

---

## Event가 여러 개 있는데 왜 `once`와 `on`을 다르게 사용할까?

현재 코드에는:

```tsx
child.stdout.on("data", ...)
child.stderr.on("data", ...)
```

와:

```tsx
child.once("error", ...)
child.once("close", ...)
```

가 같이 존재한다. 붙여넣은 텍스트(1)

둘의 차이는:

```
on
= 이벤트가 발생할 때마다 실행

once
= 최초 한 번만 실행
```

이다.

stdout은 Process가 실행되는 동안 여러 번 출력될 수 있다.

```
data
data
data
data
```

그래서:

```tsx
.on("data")
```

를 사용한다.

반면 `close`는 최종 종료 시점 하나만 필요하기 때문에:

```tsx
.once("close")
```

를 사용한다.

정리하면:

```
stdout / stderr
= 여러 조각이 계속 들어옴
→ on

error / close
= 해당 Process Lifecycle에서 한 번 처리하면 됨
→ once
```

---

## `CommandResult`는 성공/실패 판정 자체가 아니다

이 부분이 Harness 구조에서 중요하다.

`runCommand()`의 최종 결과는:

```tsx
{
  stdout,
  stderr,
  exitCode,
  timedOut,
  error,
}
```

이다.

하지만 `runCommand()`는:

> 이 Task가 성공했는가?

를 판단하지 않는다.

예를 들어:

```
{
  exitCode: 1,
  timedOut: false,
}
```

라는 사실만 반환한다.

그 `1`이 Worker 실패인지, Verification 실패인지 판단하는 것은 상위 계층이다.

즉 책임이 다음처럼 나뉜다.

```
command-runner.ts

"실제로 무슨 일이 일어났는가?"
↓
CommandResult 반환

Worker / Verification

"그 결과를 성공으로 볼 것인가?"
↓
상위 의미로 해석
```

이것이 `command-runner.ts`가 범용적으로 Worker와 Verification 둘 다에서 사용될 수 있는 이유다.

---

## `CommandResult`와 `ExecutionState`는 다른 레벨이다

앞에서 정리한 `ExecutionState`와 여기의 상태를 섞어 생각하면 헷갈리기 쉽다.

### Command 수준

```
Command 하나
↓
runCommand()
↓
CommandResult
```

예:

```
pnpm test

→ stdout
→ stderr
→ exitCode
→ timedOut
```

---

### Task 수준

```
Task 하나
↓
Worker
↓
Scope
↓
Verification
↓
ExecutionState
```

Task 하나 안에서는 여러 Command가 실행될 수 있다.

```
ExecutionState

├─ Codex 실행
│    └─ CommandResult
│
├─ pnpm typecheck
│    └─ CommandResult
│
├─ pnpm test
│    └─ CommandResult
│
└─ git diff --check
     └─ CommandResult
```

즉:

```
CommandResult
= 하나의 Process 실행 사실

ExecutionState
= Task 전체 실행 사실을 누적하는 상태
```

이다.

---

## Harness에서 `command-runner.ts`의 위치

전체 구조에서 보면:

```
TaskContract
↓
task-executor.ts
↓
execution-stages.ts
↓
┌──────────────────────────┐
│ Worker                   │
│   ↓                      │
│ runCodexWorker()         │
│   ↓                      │
│ runCommand()             │
│                          │
│ Verification             │
│   ↓                      │
│ runVerification()        │
│   ↓                      │
│ runCommand()             │
└──────────────────────────┘
↓
실제 OS Process
```

즉 `command-runner.ts`는 Task orchestration을 담당하지 않는다.

그보다 아래에서:

```
명령 실행
출력 수집
Timeout 관리
Process 종료
결과 반환
```

이라는 공통 책임만 담당한다.

---

## 왜 이것도 Harness의 일부인가?

AI Agent를 실행한다고 하면 보통:

```
Prompt 전달
↓
AI가 코드 수정
```

정도로 생각하기 쉽다.

하지만 실제 개발 Harness에서는 Agent가:

```
무한 실행될 수도 있고
자식 Process를 만들 수도 있고
오류로 종료될 수도 있고
부분적인 출력만 남길 수도 있고
테스트가 무한 대기할 수도 있다
```

따라서 Harness가 AI Worker를 신뢰하기 전에 먼저 **실행 환경 자체를 통제할 수 있어야 한다.**

`command-runner.ts`가 담당하는 것은 바로 이 부분이다.

```
AI에게 일을 맡긴다
```

에서 끝나는 게 아니라:

```
AI Process를 실행한다
↓
시간을 제한한다
↓
출력을 수집한다
↓
필요하면 종료한다
↓
결과를 구조화한다
```

까지 Harness가 책임진다.

---

## 최종적으로 기억할 흐름

```
runCommand()
↓
spawn()
↓
Process 실행
↓
stdout / stderr 수집

        ┌─ 정상 종료
        │    ↓
        │   close
        │    ↓
        │   finish()
        │
        └─ Timeout
             ↓
           SIGTERM
             ↓
           250ms
             ↓
           SIGKILL
             ↓
           finish()

↓
CommandResult
```

그리고 Harness 계층에서는:

```
Task Executor
↓
Execution Stage
↓
Worker / Verification
↓
Command Runner
↓
OS Process
```

라고 이해하면 된다.

---

그리고 Windows 경로는:

```
child.kill(signal);
```

만 사용하므로 현재 구현상 **Windows에서는 실제 Process Tree 전체 종료를 보장하지 않는다.** 이 부분도 나중에 크로스플랫폼 보완 포인트로 남겨두면 좋아.

---

### 한 줄 정리

> `command-runner.ts`는 외부 Process를 실행하고, 출력·timeout·종료를 제어한 뒤 결과를 `CommandResult`로 반환하는 Harness의 저수준 실행 계층이다.
