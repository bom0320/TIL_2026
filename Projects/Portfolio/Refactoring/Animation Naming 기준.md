# Stage / Scene / Animation 네이밍 기준

## 1. 최종 네이밍 기준

현재 구조에서는 Stage, Scene, Scene dom, Animation의 책임이 다르기 때문에 네이밍도 역할 기준으로 분리한다.

```txt
Stage = 큰 스크롤 흐름
Scene = 화면 DOM 구조
Scene dom = animation target 수집
Animation = 실제 움직임 controller
```

따라서 네이밍도 다음 기준으로 고정한다.

```txt
Stage 관련 파일 = Stage를 붙인다.
Scene 컴포넌트 = Scene을 붙인다.
Scene dom 파일 = {scene}.selectors.ts / {scene}.elements.ts
Animation controller 파일 = {sceneOrPart}.animation.ts
```

---

# 2. Stage 계층 네이밍

## 역할

Stage는 큰 스크롤 흐름을 담당한다.

따라서 Stage 계층의 파일명에는 `Stage`를 붙인다.

## 예시

```txt
CapabilityStage.tsx
useCapabilityStageAnimation.ts
capabilityStageSelectors.ts
capabilityStageScrollConfig.ts
capabilityStageElements.ts
capabilityStageControllers.ts
```

이 파일들은 모두 Stage 흐름과 관련되어 있다.

- Stage 컴포넌트
- Stage hook
- Stage selector
- Stage scroll config
- Stage elements
- Stage controllers

따라서 이름에 `Stage`를 붙이는 것이 맞다.

## Stage 네이밍 규칙

```txt
{domain}Stage*
```

예시:

```txt
capabilityStageSelectors.ts
capabilityStageElements.ts
capabilityStageControllers.ts
capabilityStageScrollConfig.ts
```

---

# 3. Scene 계층 네이밍

## 역할

Scene은 실제 화면 DOM 구조를 담당한다.

따라서 Scene 컴포넌트에는 `Scene`을 붙인다.

## 예시

```txt
CapabilityIntroScene.tsx
ExperienceCapabilityScene.tsx
CapabilityNavigatorScene.tsx
CapabilityClosingScene.tsx
```

이건 현재 구조처럼 유지하면 된다.

---

# 4. Scene dom 파일 네이밍

## 역할

Scene dom은 Scene 내부의 animation target DOM을 찾는 역할이다.

다만 dom 파일명에는 `Scene`을 굳이 붙이지 않는다.

이유는 경로가 이미 Scene 계층임을 말해주기 때문이다.

```txt
components/scenes/capability/dom/
```

이 경로 안에 있는 파일은 당연히 `capability scene DOM` 관련 파일이다.

---

## 추천 구조

Intro 기준으로는 다음처럼 둔다.

```txt
components/scenes/capability/dom/
├─ capabilityIntro.selectors.ts
├─ capabilityIntro.elements.ts
└─ index.ts
```

여기서 각각의 의미는 다음과 같다.

```txt
capabilityIntro = 대상 Scene
.selectors = selector 정의
.elements = DOM element 수집
```

---

## 너무 긴 이름은 피한다

다음처럼 이름을 너무 길게 만들 필요는 없다.

```txt
capabilityIntroSceneAnimationSelectors.ts ❌
capabilityIntroSceneAnimationElements.ts ❌
```

이렇게 쓰면 의미는 명확하지만, 파일명이 과하게 장황해진다.

---

## `capabilityIntro.selectors.ts`가 적절한 이유

경로까지 합쳐서 읽으면 의미가 충분히 명확하다.

```txt
components/scenes/capability/dom/capabilityIntro.selectors.ts
```

이 경로는 다음처럼 해석된다.

```txt
capability scene의 intro DOM selector
```

따라서 `capabilityIntro.selectors.ts` 정도면 충분하다.

---

# 5. Animation 파일 네이밍

## 역할

Animation은 실제 움직임 controller를 정의한다.

따라서 파일명은 `*.animation.ts`로 통일한다.

## 예시

```txt
animations/capability/intro/
├─ capabilityIntro.animation.ts
├─ capabilityIntroProof.animation.ts
└─ index.ts
```

## Animation 네이밍 규칙

```txt
{sceneOrPart}.animation.ts
```

예시:

```txt
capabilityIntro.animation.ts
capabilityIntroProof.animation.ts
capabilityClosing.animation.ts
contactIntro.animation.ts
contactFooter.animation.ts
```

---

# 6. 최종 구조 예시

## `src/components/stages`

```txt
src/components/stages/
├─ CapabilityStage.tsx
├─ constants/
│  ├─ capabilityStageSelectors.ts
│  ├─ capabilityStageScrollConfig.ts
│  └─ index.ts
├─ hooks/
│  ├─ useCapabilityStageAnimation.ts
│  └─ helpers/
│     └─ capability/
│        ├─ capabilityStageElements.ts
│        ├─ capabilityStageControllers.ts
│        └─ index.ts
```

---

## `src/components/scenes/capability`

```txt
src/components/scenes/capability/
├─ CapabilityIntroScene.tsx
├─ ExperienceCapabilityScene.tsx
├─ CapabilityNavigatorScene.tsx
├─ CapabilityClosingScene.tsx
├─ dom/
│  ├─ capabilityIntro.selectors.ts
│  ├─ capabilityIntro.elements.ts
│  └─ index.ts
└─ index.ts
```

---

## `src/animations/capability/intro`

```txt
src/animations/capability/intro/
├─ capabilityIntro.animation.ts
├─ capabilityIntroProof.animation.ts
└─ index.ts
```

---

# 7. 함수 네이밍 기준

## Stage helper 함수

Stage helper 함수는 Stage의 큰 DOM 수집과 controller 생성을 담당한다.

### Stage 큰 DOM 수집

```ts
getCapabilityStageElements();
```

### Stage controller 생성

```ts
createCapabilityStageControllers();
```

### Stage controller reset

```ts
resetCapabilityProgressControllers();
```

### Stage controller destroy

```ts
destroyCapabilityStageControllers();
```

이 네이밍은 현재 구조에서 적절하다.

---

# 8. Scene dom 함수 네이밍

Scene 내부 animation target을 수집하는 함수는 다음처럼 둔다.

```ts
getCapabilityIntroAnimationElements();
getCapabilityIntroProofAnimationElements();
```

이 함수들은 `CapabilityIntroScene` 내부에서 animation target으로 쓰일 elements를 가져온다.

## 더 짧은 대안

더 짧게 쓰고 싶다면 다음도 가능하다.

```ts
getCapabilityIntroElements();
getCapabilityIntroProofElements();
```

하지만 현재 구조에서는 기존처럼 `AnimationElements`를 붙이는 것을 추천한다.

```ts
getCapabilityIntroAnimationElements();
```

## 이유

이 함수가 일반 DOM elements를 가져오는 것이 아니라, animation target elements를 가져온다는 의미가 분명해진다.

---

# 9. 타입 네이밍 기준

Scene dom 함수의 반환 타입은 다음처럼 둔다.

```ts
export type CapabilityIntroAnimationElements = ReturnType<
  typeof getCapabilityIntroAnimationElements
>;

export type CapabilityIntroProofAnimationElements = ReturnType<
  typeof getCapabilityIntroProofAnimationElements
>;
```

이 네이밍도 적절하다.

다음처럼 `Scene`까지 넣을 필요는 없다.

```ts
CapabilityIntroSceneAnimationElements ❌
```

너무 길어지고, 경로상 이미 Scene 계층임을 알 수 있기 때문이다.

---

# 10. Selector 상수 네이밍

## 현재 추천

```ts
CAPABILITY_INTRO_ANIMATION_SELECTORS;
CAPABILITY_INTRO_PROOF_ANIMATION_SELECTORS;
```

이 네이밍은 유지하는 편이 좋다.

---

## 선택 A: 명확한 이름

```ts
CAPABILITY_INTRO_ANIMATION_SELECTORS;
CAPABILITY_INTRO_PROOF_ANIMATION_SELECTORS;
```

### 장점

어디서 import해도 의미가 명확하다.

---

## 선택 B: 짧은 이름

```ts
INTRO_ANIMATION_SELECTORS;
INTRO_PROOF_ANIMATION_SELECTORS;
```

### 장점

파일 내부에서는 짧다.

### 단점

다른 파일에서 import했을 때 맥락이 약해진다.

---

## 결론

현재는 A안을 유지한다.

```ts
CAPABILITY_INTRO_ANIMATION_SELECTORS;
CAPABILITY_INTRO_PROOF_ANIMATION_SELECTORS;
```

조금 길어도 import 위치에서 의미가 분명한 편이 더 낫다.

---

# 11. 최종 네이밍 표

| 계층                 | 파일명 규칙                    | 예시                             |
| -------------------- | ------------------------------ | -------------------------------- |
| Stage 컴포넌트       | `{Domain}Stage.tsx`            | `CapabilityStage.tsx`            |
| Stage hook           | `use{Domain}StageAnimation.ts` | `useCapabilityStageAnimation.ts` |
| Stage selector       | `{domain}StageSelectors.ts`    | `capabilityStageSelectors.ts`    |
| Stage scroll config  | `{domain}StageScrollConfig.ts` | `capabilityStageScrollConfig.ts` |
| Stage elements       | `{domain}StageElements.ts`     | `capabilityStageElements.ts`     |
| Stage controllers    | `{domain}StageControllers.ts`  | `capabilityStageControllers.ts`  |
| Scene 컴포넌트       | `{Name}Scene.tsx`              | `CapabilityIntroScene.tsx`       |
| Scene DOM selectors  | `{scene}.selectors.ts`         | `capabilityIntro.selectors.ts`   |
| Scene DOM elements   | `{scene}.elements.ts`          | `capabilityIntro.elements.ts`    |
| Animation controller | `{sceneOrPart}.animation.ts`   | `capabilityIntro.animation.ts`   |

---

# 12. 지금 리팩토링할 네이밍

Intro 기준으로는 다음처럼 맞춘다.

## 이동 전

```txt
animations/capability/intro/dom/
├─ intro.selectors.ts
├─ intro.elements.ts
```

## 이동 후

```txt
components/scenes/capability/dom/
├─ capabilityIntro.selectors.ts
├─ capabilityIntro.elements.ts
```

이 방식이 더 적절하다.

이유는 `components/scenes/capability/dom` 안에는 앞으로 다음 파일들이 추가될 수 있기 때문이다.

```txt
capabilityNavigator.selectors.ts
capabilityNavigator.elements.ts
capabilityClosing.selectors.ts
capabilityClosing.elements.ts
```

따라서 `intro.selectors.ts`처럼 너무 짧게 두면 나중에 이름이 약해진다.

---

# 13. 최종 결론

네이밍은 다음 기준으로 고정한다.

```txt
Stage 관련 파일은 전부 Stage를 붙인다.
Scene 컴포넌트는 Scene을 붙인다.
Scene dom 파일은 {scene}.selectors.ts / {scene}.elements.ts로 둔다.
Animation controller 파일은 *.animation.ts로 둔다.
```

지금 바로 적용할 파일명은 다음과 같다.

```txt
components/scenes/capability/dom/capabilityIntro.selectors.ts
components/scenes/capability/dom/capabilityIntro.elements.ts
```

이렇게 가면 Stage / Scene / Animation 책임이 이름만 봐도 구분된다.
