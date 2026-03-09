# GSAP quickSetter + Transform Alias 개념 정리

## `gsap.quickSetter()` 개념

`quickSetter()` 는 **특정 속성을 매우 자주 업데이트할 때 성능을 최적화하기 위한 함수 생성 메서드** 이다.

일반 `gsap.set()` 과의 차이:

| 방식                 | 특징                                           |
| -------------------- | ---------------------------------------------- |
| `gsap.set()`         | 호출할 때마다 내부 처리 (파싱, 초기화 등) 수행 |
| `gsap.quickSetter()` | 특정 속성 전용 **setter 함수**를 미리 생성     |

즉,

```ts
setX(100);
```

처럼 **값만 넣어서 매우 빠르게 업데이트** 할 수 있다.

주로 사용되는 상황

- 마우스 트래킹
- 스크롤 인터랙션
- ticker 기반 애니메이션
- 초당 수십~수백 번 업데이트 되는 애니메이션

**Learn More :**
[ GSAP 공식 문서의 quickSetter 가이드](<https://gsap.com/docs/v3/GSAP/gsap.quickSetter()/>)

## 코드 참고해보기

```ts
// track(HTMLDivElement)

const setX = gsap.quickSetter(track, "x", "px") as (value: number) => void;
```

1. `gsap.quickSetter(track, "x", "px") ` :

   - track 요소의 x 속성(transform: translateX) 을 px 단위로 변경하는 **최적화된 함수**를 만든다

2. `as (value: number) => void (Type Assertion)` :

- **`as`** : TS에게 "이 변수의 타입은 내가 지정한 것이 맞으니 의심하지 마"라고 알려주는 키워드
- **`(value: number) => void`** : setX라는 변수가 **숫자(number)을 인자로 받고 아무것도 반환하지 않는 (void) 함수** 임을 정의한다. -**이유** : 기본적으로 `quickSetter` 가 반환하는 타입이 포괄적일 수 있기 때문에, 실제 사용할 때 setX(100)처럼 숫자를 넣었을 때, 에러가 나지 않도록 타입을 강제하는 것이다.

결국 이 코드는 **"track의 x 좌표를 px 단위로 수정하는 setX 를 만드는데, 이 함수는 숫자형 인자 하나만 받는 함수"** 야 라고 선언하는 것과 같다.

---

## `"x"` 속성의 의미 (GSAP Alias)

중요한 부분:
GSAP 은 **CSS transform 을 쉽게 다루기 위한 별칭 (Alias)** 을 제공한다.

| GSAP 속성 | 실제 CSS      |
| --------- | ------------- |
| x         | translateX    |
| y         | translateY    |
| rotation  | rotate        |
| scale     | scale         |
| skewX     | skewX         |
| xPercent  | translateX(%) |

따라서 `gsap.quickSetter(track, "x", "px")` 라고 쓰면, GSAP은 내부적으로 **"track(HTMLDicElement)의 Inline style 중 transform 속성 안에 translateX 값을 px 단위로 넣어야지"** 라고 자동으로 해석한다.

Ex)

```ts
gsap.set(".box", { x: 100 });
```

실제론 내부적으로

```css
transform: translateX(100px);
```

로 변환된다.

#### GSAP Cheat Sheet_Core Transform Properties

[GSAP 공식 치트 시트](https://gsap.com/cheatsheet/)

## 왜 HTML 요소에 `x` 속성이 없는데 동작할까?

`HTMLDivElement` 에는 실제로 이런 속성이 없다.

```ts
element.x; // (x)
```

하지만 GSAP 은 다음 과정을 거친다.

### 동작 흐름

#### 1. 개발자 코드

```ts
setX(100);
```

#### 2. GSAP 내부 처리

```ts
100 + "px";
```

#### 3. DOM Style 적용

```css
track.style..transform = "translateX(100px)"
```

즉, GSAP 엔진이 Transform을 대신 관리한다.

### TypeScript에서의 상황

코드에서 `as (value: number) => void` 를 붙인 이유도 바로 이것이다. TS 입장에서는 HTMLDivElement에 x라는 속성이 없으니 "이게 뭐야?"라고 할 수 있지만, 개발자는 **"GSAP이 이걸 받아서 알아서 처리해줄 거야"** 라는 것을 알기 때문에 타입을 강제로 지정해준 것이다.

## 왜 Alias를 쓰는가

CSS Transform 은 원래 문자열이다.

ex)

```css
transform: translateX(100px) rotate(45deg);
```

이걸 TS(JS)에서 매번 수정하면

- 문자열 파싱
- transform 분해
- 재조합
  이 과정이 필요하다.

하지만 GSAP 는 내부적으로

```
x
y
rotation
scale
```

같은 숫자 값만 관리 한다. 그래서 **더 빠르고 더 안정적인 애니메이션** 이 가능하다.

## quickSetter와 Alias의 관계

`quickSetter`는 transform alias와 함께 사용할 때 가장 효율적이다.

ex)

```ts
const setX = gsap.quickSetter(track, "x", "px");
```

이 경우

```ts
setX(value);
```

만 호출하면 GSAP 내부 엔진이

```
translateX(value px)
```

로 처리한다.
