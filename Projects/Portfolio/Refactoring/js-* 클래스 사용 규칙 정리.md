# `.js-*` 클래스 사용 규칙 정리

## 1. 핵심 원칙

`.js-*` 클래스는 **스타일링용 클래스가 아니라 DOM 선택용 클래스**로만 사용한다.

즉, CSS/SCSS에서는 `.js-*` 클래스를 기준으로 스타일을 작성하지 않고,
애니메이션이나 DOM 탐색이 필요한 요소를 `querySelector`로 찾기 위한 용도로만 사용한다.

```tsx
<section className="capability-intro-scene js-capability-intro">...</section>
```

```scss
/* (O) 스타일은 BEM class 기준 */
.capability-intro-scene {
  ...
}

/* (X) js-* 클래스는 스타일링에 사용하지 않음 */
.js-capability-intro {
  ...
}
```

---

## 2. Stage root는 `ref`로 잡는다

Stage의 최상위 root 요소는 `.js-*` 클래스로 찾지 않고, `ref`로 직접 잡는다.

```tsx
<section ref={stageRef} className="capability-stage">
  ...
</section>
```

따라서 아래처럼 root에 `js-*` 클래스를 중복으로 붙일 필요는 없다.

```tsx
<section ref={stageRef} className="capability-stage js-capability-stage">
  ...
</section>
```

위 방식은 불필요하다.

이유는 이미 아래처럼 root DOM을 직접 가지고 있기 때문이다.

```ts
const stage = stageRef.current;
```

즉, Stage root는 `querySelector`로 다시 찾을 필요가 없다.

---

## 3. `.js-*` 클래스는 하위 animation target에만 붙인다

Stage 내부에서 `stage.querySelector()`로 찾아야 하는 하위 요소에만 `.js-*` 클래스를 붙인다.

예를 들어 CapabilityStage에서는 다음과 같은 요소들이 해당된다.

```tsx
<section className="capability-intro-scene js-capability-intro">...</section>
```

```tsx
<div className="capability-navigator-pin js-capability-navigator-pin">...</div>
```

이런 요소들은 Stage root 내부에서 애니메이션 타겟으로 찾아야 하므로 `.js-*` 클래스가 필요하다.

---

## 4. 클래스의 역할 분리

하나의 요소에 BEM class와 js class를 함께 사용할 수 있다.

```tsx
<section className="capability-intro-scene js-capability-intro">...</section>
```

이때 역할은 명확히 분리한다.

| 클래스                   | 역할                         |
| ------------------------ | ---------------------------- |
| `capability-intro-scene` | 스타일링용                   |
| `js-capability-intro`    | DOM 선택 / 애니메이션 타겟용 |

즉, 스타일은 BEM class 기준으로 작성하고,
GSAP이나 `querySelector`에서 사용할 대상만 `.js-*` 클래스로 지정한다.

---

## 5. selector 상수는 `querySelector` 전용으로 둔다

selector 상수는 JSX의 `className`에 직접 넣는 용도가 아니다.

```ts
export const CAPABILITY_STAGE_SELECTORS = {
  intro: ".js-capability-intro",
  introPinned: ".js-capability-intro-pinned",
  introProof: ".js-capability-intro-proof",

  structure: ".js-capability-structure",
  ai: ".js-capability-ai",
  visual: ".js-capability-visual",

  navigatorIntro: ".js-capability-navigator-intro",
  navigatorPin: ".js-capability-navigator-pin",
  navigatorLayer: ".js-capability-navigator-layer",

  closing: ".js-capability-closing",
} as const;
```

위 상수는 다음과 같은 용도로만 사용한다.

```ts
stage.querySelector<HTMLElement>(CAPABILITY_STAGE_SELECTORS.intro);
```

JSX에서 아래처럼 사용하는 것은 피한다.

```tsx
className={CAPABILITY_STAGE_SELECTORS.intro} // X
```

JSX에서는 className을 직접 명시한다.

```tsx
<section className="capability-intro-scene js-capability-intro">...</section>
```

---

## 6. IntroStage 적용 규칙

IntroStage도 CapabilityStage와 같은 패턴으로 맞춘다.

Stage root는 `ref`로 잡고, root에는 `.js-*` 클래스를 붙이지 않는다.

```tsx
"use client";

import { useRef } from "react";

import { AboutScenes, HeroScene, LifeMotionScene } from "@/components/scenes";
import { useIntroStageAnimation } from "./hooks/useIntroStageAnimation";

export default function IntroStage() {
  const stageRef = useRef<HTMLElement | null>(null);

  useIntroStageAnimation(stageRef);

  return (
    <section ref={stageRef} className="intro-stage">
      <div className="intro-stage__sticky">
        <HeroScene />
        <LifeMotionScene />
        <AboutScenes />
      </div>
    </section>
  );
}
```

---

## 7. IntroStage 하위 scene에는 `.js-*` 클래스를 붙인다

Stage 내부에서 애니메이션 타겟으로 사용할 하위 scene에는 `.js-*` 클래스를 붙인다.

```tsx
export default function HeroScene() {
  return <section className="hero-scene js-intro-hero">...</section>;
}
```

```tsx
export default function LifeMotionScene() {
  return <section className="life-motion js-intro-life-motion">...</section>;
}
```

```tsx
export default function AboutScenes() {
  return <section className="about-scenes js-intro-about">...</section>;
}
```

---

## 8. IntroStage selector 예시

IntroStage의 selector 상수는 root를 제외하고, 하위 animation target만 관리한다.

```ts
export const INTRO_STAGE_SELECTORS = {
  hero: ".js-intro-hero",
  lifeMotion: ".js-intro-life-motion",
  about: ".js-intro-about",
} as const;
```

여기서 root를 넣지 않는 이유는 root를 selector로 찾는 것이 아니라 `stageRef`로 직접 잡기 때문이다.

---

## 9. `getIntroStageElements.ts` 예시

```ts
import { INTRO_STAGE_SELECTORS } from "../../../constants";

export function getIntroStageElements(stage: HTMLElement) {
  return {
    root: stage,

    hero: stage.querySelector<HTMLElement>(INTRO_STAGE_SELECTORS.hero),

    lifeMotion: stage.querySelector<HTMLElement>(
      INTRO_STAGE_SELECTORS.lifeMotion
    ),

    about: stage.querySelector<HTMLElement>(INTRO_STAGE_SELECTORS.about),
  };
}

export type IntroStageElements = ReturnType<typeof getIntroStageElements>;
```

이 구조를 사용하면 IntroStage도 CapabilityStage와 같은 흐름을 가진다.

```txt
stageRef
→ stage
→ getIntroStageElements(stage)
→ 하위 animation target 수집
→ animation controller에서 사용
```

---

## 10. `.js-*` 네이밍 규칙

`.js-*` 클래스는 scene 이름만 단독으로 쓰기보다, Stage prefix를 붙이는 것이 좋다.

```ts
// 추천
.js-intro-hero
.js-intro-life-motion
.js-intro-about
```

```ts
// 비추천
.js-hero-scene
.js-life-motion
.js-about
```

Stage prefix를 붙이면 어떤 Stage에서 사용하는 animation target인지 명확해진다.

CapabilityStage도 이미 같은 방식에 가깝다.

```txt
.js-capability-intro
.js-capability-structure
.js-capability-navigator-pin
```

따라서 IntroStage도 다음처럼 맞추는 것이 좋다.

```txt
.js-intro-hero
.js-intro-life-motion
.js-intro-about
```

---

## 11. 최종 정리

`.js-*` 클래스는 지금 당장 전체를 한 번에 갈아엎기보다는, 먼저 규칙을 정하고 Stage 단위로 천천히 맞추는 것이 안정적이다.

최종 규칙은 다음과 같다.

1. Stage root는 `ref`로 잡는다.
2. Stage root에는 굳이 `.js-*` 클래스를 붙이지 않는다.
3. `.js-*` 클래스는 하위 animation target에만 붙인다.
4. `.js-*` 클래스는 스타일링에 사용하지 않는다.
5. 스타일은 BEM class로 작성한다.
6. selector 상수는 `querySelector` 전용으로 사용한다.
7. selector 상수를 JSX `className`에 직접 넣지 않는다.
8. `.js-*` 네이밍에는 Stage prefix를 붙인다.
9. Stage마다 `getStageElements()` 유틸을 두고 DOM 수집 책임을 분리한다.
10. 리팩토링은 Stage 단위로 점진적으로 진행한다.

이 방식으로 정리하면 스타일 클래스와 애니메이션 타겟 클래스의 역할이 분리되고,
GSAP animation controller도 Stage 단위로 더 안정적으로 관리할 수 있다.
