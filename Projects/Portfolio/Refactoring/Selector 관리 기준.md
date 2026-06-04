# Selector 관리 기준

## 1. JSX의 `className`은 직접 작성한다

JSX에서는 `className`을 상수화하지 않고 직접 작성한다.

```tsx
<section className="capability-intro-scene js-capability-intro">
```

### 이유

- 실제 DOM 구조가 JSX 안에서 바로 보인다.
- BEM 스타일 클래스와 `js-*` animation target을 함께 확인할 수 있다.
- `className`까지 상수화하면 JSX 가독성이 떨어지고, 과한 리팩토링이 될 수 있다.

따라서 기본 기준은 다음과 같다.

```txt
JSX className = 직접 작성
```

---

## 2. Stage hook/helper가 찾는 큰 DOM selector는 `*_STAGE_SELECTORS`에서 관리한다

Stage hook 또는 helper에서 직접 `querySelector`로 찾는 selector는 Stage 단위 constants로 관리한다.

```ts
export const CAPABILITY_STAGE_SELECTORS = {
  intro: ".js-capability-intro",
  introPinned: ".js-capability-intro-pinned",
  introProof: ".js-capability-intro-proof",

  navigatorPin: ".js-capability-navigator-pin",
  closing: ".js-capability-closing",
} as const;
```

### 기준

```txt
Stage hook/helper에서 querySelector 하는 selector
→ *_STAGE_SELECTORS
```

### 여기에 들어가는 selector

- stage 내부 scene root
- pinned 영역
- scroll trigger 기준 DOM
- controller에 넘겨줄 scope DOM
- navigator pin/layer 같은 큰 구조 DOM

즉, Stage가 직접 제어하거나 animation controller에 넘겨줘야 하는 큰 단위의 DOM만 `*_STAGE_SELECTORS`에 둔다.

---

## 3. Animation controller 내부 target selector는 animation 파일 안에서 관리한다

특정 animation controller 안에서만 사용하는 세부 target selector는 해당 animation 파일 내부의 `SELECTORS` 객체에서 관리한다.

```ts
const SELECTORS = {
  title: ".js-capability-intro-title",
  subtitle: ".js-capability-intro-subtitle",
  phase01: ".js-capability-intro-phase-01",
  phase02: ".js-capability-intro-phase-02",
} as const;
```

### 기준

```txt
해당 animation controller 안에서만 querySelector 하는 selector
→ animation 파일 내부 SELECTORS
```

### 여기에 들어가는 selector

- title
- subtitle
- card
- item
- quote
- character
- icon
- line
- progress bar

즉, Stage가 알 필요 없는 세부 animation target은 animation 파일 내부에 둔다.

---

## 4. 이 기준이 괜찮은 이유

이 기준은 각 영역의 책임이 명확하다.

```txt
JSX
= 실제 DOM 구조를 보여준다.

Stage selectors
= Stage가 어떤 큰 구간을 제어하는지 보여준다.

Animation SELECTORS
= 해당 animation이 어떤 내부 요소를 움직이는지 보여준다.
```

이렇게 나누면 나중에 유지보수할 때 selector의 위치가 덜 꼬인다.

- Stage에서 찾는 요소인지
- 특정 animation 내부에서만 쓰는 요소인지
- JSX에서 구조를 보여주기 위한 클래스인지

역할을 기준으로 구분할 수 있다.

---

# 적용 예시

## Capability

### `CAPABILITY_STAGE_SELECTORS`

Stage hook/helper가 찾는 큰 단위 selector만 관리한다.

```ts
export const CAPABILITY_STAGE_SELECTORS = {
  intro: ".js-capability-intro",
  introPinned: ".js-capability-intro-pinned",
  introProof: ".js-capability-intro-proof",

  navigatorPin: ".js-capability-navigator-pin",
  closing: ".js-capability-closing",
} as const;
```

### `capabilityIntro.animation.ts`

Intro animation 내부에서만 쓰는 세부 target은 animation 파일 내부에서 관리한다.

```ts
const SELECTORS = {
  title: ".js-capability-intro-title",
  subtitle: ".js-capability-intro-subtitle",
  phase01: ".js-capability-intro-phase-01",
} as const;
```

### `capabilityIntroProof.animation.ts`

Intro proof animation 내부에서만 쓰는 세부 target도 해당 animation 파일 내부에서 관리한다.

```ts
const SELECTORS = {
  character: ".js-capability-intro-proof-character",
  pointLeft: ".js-capability-intro-proof-point-left",
  quote: ".js-capability-intro-proof-quote",
} as const;
```

---

## Contact

### Stage hook/helper가 찾는 경우

```ts
export const CONTACT_STAGE_SELECTORS = {
  hero: ".js-contact-hero",
  footer: ".js-contact-footer",
} as const;
```

### Contact animation 내부에서만 쓰는 경우

```ts
const SELECTORS = {
  title: ".js-contact-title",
  form: ".js-contact-form",
  card: ".js-contact-card",
} as const;
```

---

## IntroStage / Hero / About

다른 Stage도 동일한 기준을 적용한다.

```txt
Stage가 찾는 root / pin / section selector
→ *_STAGE_SELECTORS

Animation이 찾는 title / text / image / line selector
→ animation 파일 내부 SELECTORS
```

---

# 예외 기준

다음 경우에는 animation 내부 selector도 외부 constants로 분리할 수 있다.

## 1. 같은 세부 selector를 2개 이상의 animation 파일이 공유하는 경우

예를 들어 같은 `.js-target-title`을 여러 animation controller가 동시에 참조한다면, 한 곳에서 관리하는 편이 낫다.

## 2. 같은 세부 selector를 hook/helper에서도 직접 찾는 경우

Stage hook/helper와 animation controller가 같은 세부 DOM을 함께 찾기 시작하면 내부 selector로만 두기 애매해진다.

## 3. selector 목록이 너무 길어서 animation 파일 상단을 심하게 가리는 경우

animation 파일 상단의 `SELECTORS`가 너무 길어져 실제 animation 로직을 읽기 어렵다면 분리할 수 있다.

## 4. 하나의 scene 안에서 여러 animation controller가 같은 target을 같이 제어하는 경우

여러 controller가 같은 DOM target을 공유하면 scene 단위 selector constants로 분리하는 것이 더 안정적이다.

---

# 예외 상황에서의 분리 예시

예외에 해당하는 경우에는 scene 단위 selector 파일을 만들 수 있다.

```txt
animations/capability/intro/introSelectors.ts
```

예시:

```ts
export const CAPABILITY_INTRO_SELECTORS = {
  title: ".js-capability-intro-title",
  subtitle: ".js-capability-intro-subtitle",
  character: ".js-capability-intro-character",
  quote: ".js-capability-intro-quote",
} as const;
```

하지만 현재 Intro 구조에서는 이 예외에 해당하지 않는다.

따라서 지금은 animation 파일 내부 `SELECTORS`로 관리하는 편이 적절하다.

---

# 최종 기준 요약

## Selector 관리 기준

- JSX의 `className`은 직접 작성한다.
  스타일 클래스와 `js-*` animation target을 함께 확인할 수 있어 DOM 구조를 읽기 쉽기 때문이다.

- Stage hook/helper가 직접 찾는 큰 DOM selector는 `*_STAGE_SELECTORS`에서 관리한다.
  scene root, pinned 영역, scroll trigger 기준 요소, controller scope로 넘길 DOM이 여기에 해당한다.

- Animation controller 내부에서만 사용하는 세부 target selector는 각 animation 파일의 `SELECTORS` 객체에서 관리한다.
  title, subtitle, card, quote, character처럼 해당 animation만 제어하는 요소가 여기에 해당한다.

- 같은 세부 selector를 여러 animation/controller/helper가 공유하기 시작하면 그때 scene 단위 selector constants로 분리한다.
