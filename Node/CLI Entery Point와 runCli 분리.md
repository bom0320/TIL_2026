# CLI Entrey Point 와 runCli 분리

## 핵심 개념

CLI 프로그램에서는 보통 두 가지의 책임을 분리하는 것이 좋다.

```
운영체제 / Node.js 실행 환경
        ↓
CLI Entry Point
        ↓
애플리케이션 로직
```

- `process.argv`, `process.exitCode` 처럼 Node.js 실행 환경과 직접 연결되는 코드
- 실제로 입력을 검사하고 작업을 수행하는 프로그램 내부 로직
  즉, 이 두가지를 분리한다. 현재 구조에서는 이 역할을 다음처럼 나눈다.

```
process.argv
   ↓
runCli(args)
   ↓
Task 로딩 및 검증
   ↓
exit code 반환
```

### process.arv 란?

`process.arv` 는 Node.js 프로그램을 실행할 때 전달된 명령줄 인자들을 담고 있는 배열이다.
예를 들어

```json
tsx src/cli.ts ./tasks/example.json
```

을 실행하면 개념적으로 다음과 같은 값이 들어온다.

```ts
process.argv = ["/path/to/node", "/path/to/src/cli.ts", "./tasks/example.json"];
```

각 값들은 다음가 같다.

- `process.argv[0]` = Node.js 실행 파일 검토
- `process.argv[1]` = 현재 실행 중인 파일 경로
- `process.argv[2] 이후` = 사용자가 전달한 실제 argument
  그래서 보통:

```ts
process.argv.slice(2);
```

를 사용한다. 그래서 결과는 `["./tasks/example.json"]`

## 왜 `process.argv` 를 코드 안에서 바로 사용하지 않는가?

다음처럼 작성할 수도 있다.

```ts
const taskPath = process.argv[2];

if (!taskPath) {
  console.error("Task file path is required.");
  process.exit(1);
}

const task = loadTaskContract(taskPath);
```

작동 자체에는 문제가 없다. 하지만 이 구조에서는 프로그램 로직이 `process.argv` 라는 Node.js 전역 환경에 직접 의존한다.

```
Program Logic
     │
     └── process.argv 직접 사용
```

그러면 테스트하거나 다른 곳에서 재사용하기 어려워진다. 그래서 현재 구조에서는:

```ts
export function runCli(args: string[]): number {
  ...
}
```

처럼 일반 함수로 분리한다. 그리고 실제 실행 환경에서만:

```ts
runCli(process.argv.slice(2));
```

이걸 실행한다. 구조는 다음과 같다.

```
process.argv
     ↓
일반 string[]
     ↓
runCli(args)
```

`runCli` 입장에서는 인자가 어디에서 왔는지 알 필요가 없다.

### runCli(args)의 역할

`runCli` 은 CLI의 실제 동작을 담당한다.

```ts
export function runCli(args: string[]): number {
  const taskPath = args[0];

  if (!taskPath) {
    console.error("Error: Task file path is required.");
    return 1;
  }

  // Task 로딩 및 검증...

  return 0;
}
```

즉 다음과 같이 생각할 수 있다.

```
입력
args: string[]

↓

CLI 로직

↓

출력
exit code: number
```

예를 들어:
`runCli([]);` 이면,

```
argument 없음
↓
에러 출력
↓
1 반환
```

반대로 `runCli(["./tasks/example.json"]);` 이면:

```
Task 경로 존재
↓
Task 로딩
↓
검증
↓
성공
↓
0 반환
```

## 직접 실행 vs. Import

CLI 파일은 두 가지 방식으로 사용될 수 있다.

### 직접 실행

터미널에서

```bash
tsx src/cli.ts ./tasks/example.json
```

이라고 실행하는 경우이다. 이때 `cli.ts` 는 프로그램의 시작점이다.

```
Terminal
   ↓
cli.ts
   ↓
runCli(...)
```

따라서 실제 CLI 로직을 실행해야 한다.

### Import

텍스트나 다른 코드에서:

```ts
import { runCli } from "./cli.js";
```

처럼 가져오는 경우이다. 이때 `cli.ts` 는 프로그램 시작점이 아니라, 단순한 모듈이다.

```
Test
 ↓
import runCli
 ↓
runCli 함수만 사용
```

따라서 import 했다는 이유만으로 CLI가 자동으로 실행되면 안 된다.

## 왜 직접 실행 여부를 확인하는가?

현재 코드는 다음과 같은 구조를 가진다.

```ts
const isEntryPoint =
  process.argv[1] !== undefined &&
  import.meta.url === pathToFileURL(resolve(process.argv[1])).href;

if (isEntryPoint) {
  process.exitCode = runCli(process.argv.slice(2));
}
```

의미는 단순하다

> 현재 `cli.ts` 가 직접 실행된 경우에만 `runCli()`을 실행한다.

즉,

```
직접 실행
tsx src/cli.ts ...

→ isEntryPoint = true
→ runCli 실행
```

인 반면,

```ts
import { runCli } from "./cli.js"

→ isEntryPoint = false
→ 자동 실행하지 않음
```

이렇게 동작한다.

## 왜 이 구분이 필요한가?

만약 cli.ts 최상단에 바로

```ts
const task = loadTaskContract(process.argv[2]);
console.log(task);
```

같은 코드가 있다면.. 테스트에서:

```ts
import { runCli } from "./cli.js";
```

하는 순간 해당 코드까지 실행될 수 있다. 그러면

```
import
 ↓
process.argv 읽음
 ↓
실제 파일 접근
 ↓
console 출력
 ↓
심하면 process.exit()
```

같은 부작용이 발생한다. 즉, 단순히 함수 하나를 테스트하고 싶었는데 실제 CLI 프로그램 전체가 실행되는 문제가 생긴다.

그래서

```
함수 정의
≠
프로그램 실행
```

이를 분리하는 것이다.

## 테스트가 쉬워지는 이유

`runCli` 을 함수로 분리하면 실제 터미널은 실행하지 않아도 테스트할 수 있다.

```ts
expect(runCli([])).toBe(1);
```

이렇게 바로 호출할 수 있다. 만약 분리하지 않았으면 실제로

```
child process 실행
↓
CLI 실행
↓
stderr 캡처
↓
exit code 확인
```

와 같은 무거운 테스트가 필요할 수 있다. 즉, 실제 CLI 실행 대신,

```ts
runCli(...)
```

만 직접 호출해서 핵심 로직을 테스트할 수 있게 된다.

## process.exit() 대신 return 을 사용하는 이유

`runCli()` 안에서 다음처럼 작성할 수도 있다.

```ts
process.exit(1);
```

하지만 이렇게 하면 프로그램 프로세스 자체가 종료된다. 테스트 중이라면

```
Vitest
 ↓
runCli([])
 ↓
process.exit(1)
 ↓
테스트 프로세스 종료
```

가 될 수 있다. 그래서 내부 로직에는.. return 1만 한다.
그리고 실제 Entry Point에서

```ts
process.exitCode = runCli(...);
```

로 운영체제와 연결한다. 역할을 나누면

- **runCli**
  - 결과는 실패이며 code는 1이다.
- **Entry Point** - 그럼 05에 exit code 1로 전달한다.
  가 된다.

## Exit Code란?

CLI 프로그램은 종료될 때 운영체제에 숫자를 반환할 수 있다 .
일반적으로 0이면 성공, 1이면 실패를 의미한다.
`console.log()` 나 `console.error()` 가 사람에게 보여주는 정보라면 exit code는 다른 프로그램이나 CI에게 알려주는 결과이다.
예를 들어 CI는 프로그램 출력 문장을 이해하지 않아도:

```
exit code = 1
```

이면 작업 실패로 판단할 수 있다.

## 전체 실행 흐름

현재 구조를 전체적으로 보면:

```
Terminal
   ↓
Node.js / tsx
   ↓
process.argv
   ↓
CLI Entry Point
   ↓
process.argv.slice(2)
   ↓
runCli(args)
   ↓
Task Path 확인
   ↓
loadTaskContract()
   ↓
TaskContract 검증
   ↓
성공 / 실패 판단
   ↓
0 또는 1 반환
   ↓
process.exitCode
   ↓
Operating System
```

## 설계 관점에서의 의미

이 구조의 핵심은 단순히 테스트를 편하게 만드는 데만 있지 않다.
외부 실행 환경과 내부 애플리케이션 로직 사이에 경계를 만든 것이다.

```
외부 환경
process.argv
process.exitCode

       ↓ Boundary

내부 로직
runCli(args)
```

`runCli`은 가능한 한 다음처럼 동작한다.

```
필요한 값을 입력받는다.
↓
내부에서 판단한다.
↓
결과를 반환한다.
```

반대로 프로그램 곳곳에서:

```
process.argv
process.env
process.cwd()
```

같은 전역 환경을 직접 읽기 시작하면 코드가 외부 환경과 강하게 결합된다.

## Trust Boundary와의 연결

앞에서 본 TaskContract 검증도 비슷한 구조였다.

```
외부 JSON
↓
unknown
↓
Zod Validation
↓
TaskContract
```

여기선

```
Node.js 실행 환경
↓
process.argv
↓
일반 args
↓
runCli
```

둘 다 공통적으로:

> 외부 환경과 프로그램 내부 로직 사이의 경계를 명확하게 만든다.
> 라는 설계 원칙을 따른다.

## 핵심 정리

#### `process.argv`

- Node.js가 제공하는 실제 CLI입력 환경이다.

#### `runCli(args)`

- 외부 환경에서 입력을 전닯다아 실제 CLI 로직을 수행하는 함수이다.

#### 직접 실행

```
tsx src/cli.ts ...
```

cli.ts가 프로그램의 시작점이므로 `runCli()`을 실행한다.

#### Import

```ts
import { runCli } from "./cli.js";
```

함수를 가져오기만 하는 것으로 CLI를 자동 실행하지 않는다.

#### return 0/1

- 내부 함수가 성공/실패 결과를 반환한다

#### process.exitCode

- 그 결과를 실제 운영체제에 전달한다.

### 한 문장으로

> CLI Entry Point는 Node.js 실행 환경과 연결되고, `runCli()`은 실제 애플리케이션 로직을 담당하도록 분리한다. 덕분에 import 시 부작용을 막고, 테스트하기 쉬우며 외부 실행 환경과 내부 로직의 경계를 명확하게 유지할 수 있다.
