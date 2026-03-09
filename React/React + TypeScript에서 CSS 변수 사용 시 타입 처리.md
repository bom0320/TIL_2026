# React + TypeScript에서 CSS 변수 사용 시 타입 처리

## 1. 문제 상황

React에서 `style` prop에 **CSS 변수**를 사용하면 TypeScript 에러가 발생한다.

```tsx
<div
  style={{
    "--span": span,
    "--ratio": ratio,
  }}
/>
```

에러 예시

```
Type '{ "--span": number }' is not assignable to type 'CSSProperties'
```

### 원인

React의 `style` prop은 다음 타입을 사용한다.

```ts
React.CSSProperties;
```

이 타입은 **표준 CSS 속성만 허용**한다.

예

```
color
width
height
margin
```

하지만 CSS 변수는

```
--span
--ratio
--progress
--card-width
```

처럼 **사용자가 자유롭게 만들 수 있는 값**이기 때문에 타입 정의에 포함되어 있지 않다.

그래서 TypeScript는 이를 **유효한 CSS 속성으로 인식하지 못한다.**

---

# 2. 해결 방법 1 — 타입 단언 (가장 일반적인 방식)

TypeScript에게 해당 객체를 `CSSProperties`라고 **강제로 지정**한다.

```tsx
style={{
  "--span": span,
  "--ratio": ratio,
} as React.CSSProperties}
```

### 의미

TypeScript에게 다음과 같이 말하는 것과 같다.

```
이 객체는 CSS 스타일 객체가 맞다.
타입 검사를 하지 말고 그대로 사용해라.
```

이를 **Type Assertion(타입 단언)** 이라고 한다.

---

# 3. Copilot이 제안한 방식

Copilot은 다음과 같은 코드를 제안했다.

```tsx
style={
  {
    ["--span" as const]: span,
    ["--ratio" as const]: ratio,
  } as React.CSSProperties
}
```

이 코드에는 두 가지 타입 지정이 있다.

---

## 3.1 `"--span" as const`

```ts
["--span" as const];
```

의미

```
"--span"을 string이 아니라
정확히 "--span"이라는 문자열 리터럴 타입으로 취급해라
```

예

```ts
let a = "--span";
// 타입: string
```

```ts
let a = "--span" as const;
// 타입: "--span"
```

즉 **값이 변하지 않는 고정된 문자열 타입**으로 만든다.

---

## 3.2 `as React.CSSProperties`

```ts
} as React.CSSProperties
```

의미

```
이 객체는 CSSProperties라고 간주해라
```

즉 TypeScript 타입 검사를 **우회**하는 것이다.

---

# 4. 실무에서 가장 많이 쓰는 패턴

실무에서는 보통 다음 방식이 가장 많이 사용된다.

```tsx
style={{
  "--span": span,
  "--ratio": ratio,
} as React.CSSProperties}
```

Copilot 방식처럼 `as const`를 추가하는 경우는 상대적으로 드물다.

---

# 5. 더 확장 가능한 방법 (타입 확장)

CSS 변수를 자주 사용하는 경우 타입을 확장할 수 있다.

```ts
type CSSVars = React.CSSProperties & {
  [key: `--${string}`]: string | number;
};
```

사용 예

```tsx
const style: CSSVars = {
  "--span": span,
  "--ratio": ratio,
};
```

이 방식은

- CSS 변수 사용을 타입적으로 허용
- `as` 캐스팅을 줄일 수 있음

---

# 6. 정리

React + TypeScript에서 CSS 변수 사용 시 발생하는 문제의 흐름

```
React style prop
↓
React.CSSProperties 사용
↓
표준 CSS 속성만 허용
↓
CSS 변수(--xxx)는 타입에 없음
↓
TypeScript 에러 발생
```

해결 방법

1️. 타입 단언 사용

```ts
as React.CSSProperties
```

2️. CSS 변수 타입 확장

```ts
[key: `--${string}`]
```

---

# 7. 느낀 점

- React에서 CSS 변수를 사용할 때 TypeScript와 충돌이 자주 발생한다.
- 대부분의 프로젝트에서는 `as React.CSSProperties` 방식으로 해결한다.
- CSS 변수를 많이 사용하는 프로젝트에서는 타입 확장 방식이 더 유지보수에 유리하다.
