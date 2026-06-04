# TypeScript 타입 기준

## 결론 먼저

```
입력 계약 / 구조 계약 = 명시적 타입
함수 결과를 그대로 따라가는 타입 = ReturnType
```

더 짧게 외우면:

```
Elements 타입 = 명시적 타입
Controllers 타입 = ReturnType
```

---

# 1. 명시적 타입을 쓰는 경우

명시적 타입은 **이 구조가 반드시 지켜져야 하는 계약**일 때 쓴다.

즉, 함수가 이 타입에 맞춰서 반환해야 하고, 빠뜨리면 TypeScript가 잡아줘야 하는 경우.

## 사용 기준

```
다른 파일이 이 타입을 입력값으로 신뢰한다
→ 명시적 타입
```

또는

```
이 타입이 프로젝트 구조의 약속이다
→ 명시적 타입
```

---

## 예시 1. Stage Elements

```ts
exporttypeCapabilityStageElements= {
  intro:HTMLElement|null;
  introProof:HTMLElement|null;
  structure:HTMLElement|null;
  ai:HTMLElement|null;
  visual:HTMLElement|null;
  navigatorIntro:HTMLElement|null;
  navigatorPin:HTMLElement|null;
  closing:HTMLElement|null;
};
```

이건 명시적 타입이 맞음.

이유:

```
CapabilityStage는 반드시 intro, structure, ai, visual, closing 등의 큰 DOM 구역을 제공해야 한다.
```

만약 실수로 `visual`을 return에서 빼면 TypeScript가 잡아줘야 함.

```ts
export function getCapabilityStageElements(
stage:HTMLElement
):CapabilityStageElements {
return {
    intro: ...,
    introProof: ...,
    structure: ...,
    ai: ...,
// visual 빠짐
    closing: ...,
  };
}
```

명시적 타입이면 에러가 남.

---

## 예시 2. Animation Elements

```ts
export type AICapabilityAnimationElements = {
  root: HTMLElement | null;
  header: HTMLElement | null;
  grid: HTMLElement | null;
  cards: HTMLElement[];
  cardIcons: HTMLElement[];
  cardSubtitles: HTMLElement[];
  cardTitles: HTMLElement[];
  cardMessages: HTMLElement[];
  cardDescs: HTMLElement[];
};
```

이것도 명시적 타입이 맞음.

이유:

```
AICapabilityAnimation.create()는 이 DOM들을 받는다고 약속하고 있다.
```

animation 파일에서 이런 식으로 의존함.

```
AICapabilityAnimation.create(elements)
```

그러니까 `grid`, `cards`, `cardTitles` 같은 게 빠지면 타입이 잡아줘야 함.

---

## 명시적 타입 대상 정리

```
get...StageElements 반환 타입
get...AnimationElements 반환 타입
Animation.create(elements)의 elements 타입
외부에서 입력값으로 쓰이는 props/data/controller 계약 타입
```

현재 네 프로젝트 기준:

```
CapabilityStageElements
IntroStageElements
ContactStageElements

CapabilityIntroAnimationElements
CapabilityIntroProofAnimationElements
StructureCapabilityAnimationElements
AICapabilityAnimationElements
VisualCapabilityAnimationElements
```

얘네는 명시적 타입이 맞음.

---

# 2. ReturnType을 쓰는 경우

`ReturnType`은 **함수 결과를 그대로 타입으로 따라가면 되는 경우**에 쓴다.

## 사용 기준

```
타입이 함수 결과와 항상 같이 움직이는 게 자연스럽다
→ ReturnType
```

또는

```
외부에서 그 구조를 계약으로 강하게 고정할 필요가 없다
→ ReturnType
```

---

## 예시. Stage Controllers

```ts
export type CapabilityStageControllers =
  ReturnType<typeofcreateCapabilityStageControllers>;
```

이건 `ReturnType`이 맞음.

이유:

```ts
export function createCapabilityStageControllers(
elements:CapabilityStageElements
) {
return {
    intro:CapabilityIntroAnimation.create(...),
    introProof:CapabilityIntroProofAnimation.create(...),
    structure:StructureCapabilityAnimation.create(...),
    ai:AICapabilityAnimation.create(...),
    visual:VisualCapabilityAnimation.create(...),
    navigatorIntro:CapabilityNavigatorAnimation.createIntro(...),
    closing:CapabilityClosingAnimation.create(...),
  };
}
```

여기서 controller가 하나 추가되면 타입도 같이 따라가는 게 자연스러움.

예를 들어 나중에:

```
navigatorPin:CapabilityNavigatorAnimation.createPin(...)
```

을 추가하면 `CapabilityStageControllers`도 자동으로 같이 확장됨.

이 경우는 타입이 함수 결과를 따라가는 게 편하고 맞음.

---

# 3. 한 줄 판단법

헷갈리면 이 질문을 하면 됨.

## 질문 1

```
이 타입이 함수의 실수를 잡아줘야 하나?
```

맞으면 **명시적 타입**.

예:

```
getCapabilityStageElements가 visual을 빠뜨리면 에러가 나야 한다
→ 명시적 타입
```

---

## 질문 2

```
이 타입은 함수 결과가 바뀌면 같이 바뀌는 게 자연스러운가?
```

맞으면 **ReturnType**.

예:

```
createCapabilityStageControllers가 반환하는 controller 목록은 함수 결과를 그대로 따라가면 된다
→ ReturnType
```

---

# 4. 최종 규칙

메모용으로는 이렇게 적어두면 됨.

```
[TypeScript 타입 기준]

1. 명시적 타입 사용
- 구조가 프로젝트의 계약일 때
- 다른 파일이 입력값으로 의존할 때
- 빠진 속성을 TypeScript가 잡아줘야 할 때
- get...Elements 함수의 반환 타입
- Animation.create(elements)의 elements 타입

예:
CapabilityStageElements
IntroStageElements
ContactStageElements
AICapabilityAnimationElements
StructureCapabilityAnimationElements
VisualCapabilityAnimationElements

2. ReturnType 사용
- 함수 반환값을 그대로 타입으로 따라가면 될 때
- 반환 구조가 함수와 같이 바뀌는 게 자연스러울 때
- create...Controllers 함수의 반환 타입
- 내부 유틸의 단순 파생 타입

예:
CapabilityStageControllers = ReturnType<typeof createCapabilityStageControllers>

3. 현재 프로젝트 기준
DOM Elements 타입 = 명시적 타입
Animation Elements 타입 = 명시적 타입
Stage Elements 타입 = 명시적 타입
Controllers 타입 = ReturnType
```

---

# 지금 네 코드 기준 결론

```
capabilityStageElements.ts
→ 명시적 타입으로 수정

introStageElements.ts
→ 명시적 타입으로 수정

contactStageElements.ts
→ 명시적 타입으로 수정

capabilityExperience.element.ts
→ 명시적 타입 유지

createCapabilityStageControllers.ts
→ ReturnType 유지
```
