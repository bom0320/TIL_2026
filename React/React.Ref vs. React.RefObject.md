# React.Ref vs. React.RefObject이 차이와 타입 포함 관계

## 1. React에서 ref는 왜 필요한가

React는 원래 이렇게 설계되어 있다.

- 화면을 직접 만지지 말고
- **state -> render 결과** 만 신경 써라
  그런데 현실에서는 이런 상황이 생김:
- input에 포커스를 주고 싶다
- 스크롤 위치를 읽고 싶다
- GSAP 으로 DOM을 직접 애니메이션 하고 싶다.

그래서 React는 **DOM에 직접 접근하는 공식 통로**로 ref라는 개념을 제공한다.

## 2. ref를 DOM에 연결하는 방법은 두 가지다.

React는 **ref를 전달하는 방법을 2개** 허용함. (중요!)

### 2-1. 객체 ref (Object Ref) - 가장 많이 사용하는 방식

#### 생성

```ts
const myRef = useRef<HTMLDivElement>(null);
```

#### 사용

```ts
<div ref={myRef} />
```

#### 접근

```ts
myRef.current;
```

#### 구조

```ts
{
  current: HTMLDivElement | null;
}
```

#### 📌 특징

- `.current` 라는 **고정된 공간**이 있음
- React가 거기에 DOM을 넣어줌
- GSAP, focus, scroll 조작에 최적

- **이 객체의 타입 이름이 `React.RefObject<T>`**

### 2-2. 함수 ref (Callback Ref) = 덜 쓰지만 존재하는 방식

#### 사용

```tsx
<div
  ref={() => {
    console.log(el);
  }}
/>
```

여기서 `el`은 :

- 마운트 시 -> DOM
- 언마운트 시 -> null

#### 📌 특징

- `.current` X 없음
- React가 DOM을 함수 인자로 바로 전달
- 렌더 타이밍을 세밀하게 제어할 때 사용

- 이건 **객체가 아니라 함수임**

## 3. React는 "ref"를 하나의 타입으로 묶어야 했다.

React 입장에서는 이런 상황임:

> "ref 자리에
> 객체도 올 수 있고,
> 함수도 올 수 있고,
> null 도 올 수 있음"

그래서 만든 **상위 타입**이 바로 이것

```ts
React.Ref<T>;
```

#### 실제 정의 (개념적으로)

```ts
React.Ref<T> =
  | React.RefObject<T>
  | (instance: T | null) => void
  | null
```

즉,

- `Ref` = 가능한 모든 ref 형태
- 너무 넓은 타입

## 4. 여기서 TypeScript 문제가 발생한다.

네가 Props를 이렇게 정의했다고 해보자:

```ts
type Props = {
  fillReactRef: React.Ref<SVGRectElement>;
};
```

TypeScript는 이렇게 해석함

- "이 ref는.." - 객체일 수도 있고 - 함수 일 수도 있고 - null 일 수도 있다.
  그래서 아래 코드에서

```ts
fillRectRef.current;
```

TypeScript가 말함:

- (X) 잠깐만 함수 ref에는 `.current` 가 없잖아?
  - 에러 발생!!

## 5. 그래서 RefObject를 쓰는 것이다.

내가 실제로 쓰는것은

```ts
const fillRectRef = useRef<SVGRectElement>(null);
```

이건 **100% 객체 ref고, 함수 ref일 가능성은 0%이다.**
그래서 props 타입도 이렇게 **정확하게 맞춰주는 것** 이다.

```ts
type AboutTitleProps = {
  fillRectRef: React.RefObject<SVGRectElement | null>;
};
```

이제 TS는 확신:

> "아 이건 `.current` 있는 ref 구나?"

그래서 이게 가능해진다.

```ts
fillRectRef.current;
```

## 6. 한 페이지 요약

### ref에는 두 종류가 있다.

1. **객체 ref ->** `useRef`
2. **함수 ref ->** `ref={(el) => ...}`

### React.Ref

- 두 종류를 전부 포함한 상위 타입
- 그래서 `.current` 를 보장하지 않음

### React.RefObject

- 객체 ref 전용 타입
- `.current` 사용 가능
