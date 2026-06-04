# 전체 Animation 구조 리팩토링 기준

## 1. 최종 기준

현재 구조는 다음 기준으로 고정한다.

```txt
Scene = 구조
Scene dom = target 수집
Stage = 흐름 연결
Animation = 움직임 정의
```

각 계층의 역할은 다음과 같다.

| 구분                              | 역할                                           |
| --------------------------------- | ---------------------------------------------- |
| `components/scenes`               | 실제 화면 DOM을 그린다                         |
| `components/scenes/{domain}/dom`  | animation target DOM을 찾는다                  |
| `components/stages`               | 여러 scene을 조합하고 scroll 흐름을 관리한다   |
| `components/stages/constants`     | Stage가 직접 찾는 큰 구간 selector를 관리한다  |
| `components/stages/hooks/helpers` | Stage DOM과 animation controller를 연결한다    |
| `src/animations`                  | 전달받은 DOM을 실제로 움직인다                 |
| `src/animations/_shared`          | animation controller 공통 타입/유틸을 관리한다 |

---

# 2. `components/scenes`

## 역할

`Scene`은 실제 화면 구조를 그리는 곳이다.

예시:

```txt
components/scenes/capability/
├─ CapabilityIntroScene.tsx
├─ ExperienceCapabilityScene.tsx
├─ CapabilityNavigatorScene.tsx
├─ CapabilityClosingScene.tsx
```

여기서는 JSX를 작성한다.

```tsx
<section className="capability-intro-scene js-capability-intro">
  <IntroPinnedNarrative />
  <IntroVisualProof />
</section>
```

## 기준

JSX의 `className`은 직접 작성한다.

### 이유

- DOM 구조가 JSX에서 바로 보인다.
- BEM class와 `js-*` animation target을 함께 확인할 수 있다.
- `className`까지 상수화하면 오히려 읽기 어려워진다.

```txt
JSX className = 직접 작성
```

---

# 3. `components/scenes/{domain}/dom`

## 역할

`dom/`은 Scene 내부 DOM 중에서 animation target으로 쓸 요소를 찾는 곳이다.

예시:

```txt
components/scenes/capability/dom/
├─ capabilityIntro.selectors.ts
├─ capabilityIntro.elements.ts
└─ index.ts
```

이 폴더는 **Scene DOM 계약**이라고 보면 된다.

`CapabilityIntroScene` 안에는 다음과 같은 animation target이 있다.

```txt
title
subtitle
phase
character
quote
proof point
```

이 target들을 정의하고 수집하는 곳이 `dom/`이다.

---

## `capabilityIntro.selectors.ts`

```ts
export const CAPABILITY_INTRO_ANIMATION_SELECTORS = {
  visualField: ".js-capability-intro-visual-field",

  titleLayer: ".js-capability-intro-title-layer",
  eyebrow: ".js-capability-intro-eyebrow",
  title: ".js-capability-intro-title",
  subtitle: ".js-capability-intro-subtitle",

  phase01: ".js-capability-intro-phase-01",
  phase02: ".js-capability-intro-phase-02",
} as const;

export const CAPABILITY_INTRO_PROOF_ANIMATION_SELECTORS = {
  character: ".js-capability-intro-proof-character",
  leftPoints: ".js-capability-intro-proof-point-left",
  rightPoints: ".js-capability-intro-proof-point-right",
  quote: ".js-capability-intro-proof-quote",
} as const;
```

---

## `capabilityIntro.elements.ts`

```ts
import {
  CAPABILITY_INTRO_ANIMATION_SELECTORS,
  CAPABILITY_INTRO_PROOF_ANIMATION_SELECTORS,
} from "./capabilityIntro.selectors";

export function getCapabilityIntroAnimationElements(scope: HTMLElement | null) {
  return {
    visualField: scope?.querySelector<HTMLElement>(
      CAPABILITY_INTRO_ANIMATION_SELECTORS.visualField
    ),

    titleLayer: scope?.querySelector<HTMLElement>(
      CAPABILITY_INTRO_ANIMATION_SELECTORS.titleLayer
    ),
    eyebrow: scope?.querySelector<HTMLElement>(
      CAPABILITY_INTRO_ANIMATION_SELECTORS.eyebrow
    ),
    title: scope?.querySelector<HTMLElement>(
      CAPABILITY_INTRO_ANIMATION_SELECTORS.title
    ),
    subtitle: scope?.querySelector<HTMLElement>(
      CAPABILITY_INTRO_ANIMATION_SELECTORS.subtitle
    ),

    phase01: scope?.querySelector<HTMLElement>(
      CAPABILITY_INTRO_ANIMATION_SELECTORS.phase01
    ),
    phase02: scope?.querySelector<HTMLElement>(
      CAPABILITY_INTRO_ANIMATION_SELECTORS.phase02
    ),
  };
}

export function getCapabilityIntroProofAnimationElements(
  scope: HTMLElement | null
) {
  return {
    character: scope?.querySelector<HTMLElement>(
      CAPABILITY_INTRO_PROOF_ANIMATION_SELECTORS.character
    ),

    leftPoints:
      scope?.querySelectorAll<HTMLElement>(
        CAPABILITY_INTRO_PROOF_ANIMATION_SELECTORS.leftPoints
      ) ?? [],

    rightPoints:
      scope?.querySelectorAll<HTMLElement>(
        CAPABILITY_INTRO_PROOF_ANIMATION_SELECTORS.rightPoints
      ) ?? [],

    quote: scope?.querySelector<HTMLElement>(
      CAPABILITY_INTRO_PROOF_ANIMATION_SELECTORS.quote
    ),
  };
}

export type CapabilityIntroAnimationElements = ReturnType<
  typeof getCapabilityIntroAnimationElements
>;

export type CapabilityIntroProofAnimationElements = ReturnType<
  typeof getCapabilityIntroProofAnimationElements
>;
```

---

## `dom/index.ts`

```ts
export {
  getCapabilityIntroAnimationElements,
  getCapabilityIntroProofAnimationElements,
  type CapabilityIntroAnimationElements,
  type CapabilityIntroProofAnimationElements,
} from "./capabilityIntro.elements";
```

---

# 4. `components/stages`

## 역할

`Stage`는 여러 Scene을 조합하고, 스크롤 흐름을 관리한다.

예시:

```txt
components/stages/
├─ constants/
├─ hooks/
├─ CapabilityStage.tsx
├─ ContactStage.tsx
└─ IntroStage.tsx
```

`CapabilityStage`는 이런 큰 구조만 담당한다.

```tsx
<section ref={stageRef} className="capability-stage">
  <CapabilityIntroScene />
  <ExperienceCapabilityScene />
  <CapabilityNavigatorScene />
  <CapabilityClosingScene />
</section>
```

Stage는 Scene 내부의 `title`, `subtitle`, `quote`, `character` 같은 세부 DOM을 직접 알 필요가 없다.

---

# 5. `components/stages/constants`

## 역할

Stage가 직접 찾아야 하는 큰 DOM selector를 관리한다.

예시:

```txt
components/stages/constants/
├─ capabilityStageSelectors.ts
├─ capabilityStageScrollConfig.ts
```

여기에 들어가는 것은 큰 단위 selector다.

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

## 기준

```txt
Stage가 직접 querySelector 하는 큰 구간 selector
→ components/stages/constants
```

예시:

```txt
.js-capability-intro
.js-capability-intro-proof
.js-capability-navigator-pin
.js-capability-closing
```

## 여기에 넣지 않는 것

다음과 같은 세부 animation target은 Stage constants에 넣지 않는다.

```txt
.js-capability-intro-title
.js-capability-intro-subtitle
.js-capability-intro-proof-character
.js-capability-intro-proof-quote
```

이런 selector는 `components/scenes/capability/dom` 쪽에서 관리한다.

---

# 6. `components/stages/hooks/helpers`

## 역할

Stage에서 필요한 DOM을 찾고, animation controller를 생성해서 묶는다.

예시:

```txt
components/stages/hooks/helpers/capability/
├─ capabilityStageElements.ts
├─ capabilityStageControllers.ts
└─ index.ts
```

---

## `capabilityStageElements.ts`

Stage가 관리할 큰 DOM을 찾는다.

```ts
import { CAPABILITY_STAGE_SELECTORS } from "../../../constants";

export function getCapabilityStageElements(stage: HTMLElement) {
  return {
    intro: stage.querySelector<HTMLElement>(CAPABILITY_STAGE_SELECTORS.intro),

    introProof: stage.querySelector<HTMLElement>(
      CAPABILITY_STAGE_SELECTORS.introProof
    ),

    structure: stage.querySelector<HTMLElement>(
      CAPABILITY_STAGE_SELECTORS.structure
    ),

    ai: stage.querySelector<HTMLElement>(CAPABILITY_STAGE_SELECTORS.ai),

    visual: stage.querySelector<HTMLElement>(CAPABILITY_STAGE_SELECTORS.visual),

    navigatorIntro: stage.querySelector<HTMLElement>(
      CAPABILITY_STAGE_SELECTORS.navigatorIntro
    ),

    navigatorPin: stage.querySelector<HTMLElement>(
      CAPABILITY_STAGE_SELECTORS.navigatorPin
    ),

    closing: stage.querySelector<HTMLElement>(
      CAPABILITY_STAGE_SELECTORS.closing
    ),
  };
}

export type CapabilityStageElements = ReturnType<
  typeof getCapabilityStageElements
>;
```

---

## `capabilityStageControllers.ts`

Stage가 찾은 큰 DOM을 Scene dom helper로 넘겨서 animation controller를 생성한다.

```ts
import {
  AICapabilityAnimation,
  CapabilityClosingAnimation,
  CapabilityIntroAnimation,
  CapabilityIntroProofAnimation,
  CapabilityNavigatorAnimation,
  StructureCapabilityAnimation,
  VisualCapabilityAnimation,
} from "@/animations/capability";

import {
  getCapabilityIntroAnimationElements,
  getCapabilityIntroProofAnimationElements,
} from "@/components/scenes/capability/dom";

import type { CapabilityStageElements } from "./capabilityStageElements";

export function createCapabilityStageControllers(
  elements: CapabilityStageElements
) {
  return {
    intro: CapabilityIntroAnimation.create(
      getCapabilityIntroAnimationElements(elements.intro)
    ),

    introProof: CapabilityIntroProofAnimation.create(
      getCapabilityIntroProofAnimationElements(elements.introProof)
    ),

    structure: StructureCapabilityAnimation.create(elements.structure),
    ai: AICapabilityAnimation.create(elements.ai),
    visual: VisualCapabilityAnimation.create(elements.visual),

    navigatorIntro: CapabilityNavigatorAnimation.createIntro(
      elements.navigatorIntro
    ),

    closing: CapabilityClosingAnimation.create(elements.closing),
  };
}

export type CapabilityStageControllers = ReturnType<
  typeof createCapabilityStageControllers
>;

export function resetCapabilityProgressControllers(
  controllers: CapabilityStageControllers
) {
  controllers.intro.setProgress(0);
  controllers.introProof.setProgress(0);
  controllers.structure.setProgress(0);
  controllers.ai.setProgress(0);
  controllers.visual.setProgress(0);
  controllers.navigatorIntro.setProgress(0);
  controllers.closing.setProgress(0);
}

export function destroyCapabilityStageControllers(
  controllers: CapabilityStageControllers
) {
  controllers.intro.destroy();
  controllers.introProof.destroy();
  controllers.structure.destroy();
  controllers.ai.destroy();
  controllers.visual.destroy();
  controllers.navigatorIntro.destroy();
  controllers.closing.destroy();
}
```

---

# 7. `useCapabilityStageAnimation.ts`

## 역할

`useCapabilityStageAnimation.ts`는 실제 스크롤 흐름을 관리한다.

여기서 담당하는 것은 다음과 같다.

- `ScrollTrigger` 생성
- progress 계산
- `controller.setProgress(progress)` 호출
- `activeNavigatorIndex` 관리
- cleanup 처리

즉:

```txt
Stage Hook = 언제 움직일지 관리
```

---

# 8. `src/animations`

## 역할

`animations`는 실제 GSAP 움직임만 정의한다.

예시:

```txt
src/animations/
├─ _shared/
├─ capability/
├─ about/
├─ contact/
├─ transitions/
└─ header.animation.ts
```

여기서는 가능하면 `querySelector`를 하지 않는다.
Scene dom에서 수집한 element를 받아서 움직임만 정의한다.

```txt
animations = 받은 element로 GSAP timeline만 작성
```

---

# 9. `src/animations/_shared`

## 역할

모든 animation controller가 공유하는 공통 타입과 유틸을 관리한다.

예시:

```txt
animations/_shared/
├─ controller.ts
├─ progress.ts
└─ index.ts
```

---

## `controller.ts`

```ts
export type AnimationController = {
  setProgress: (progress: number) => void;
  destroy: () => void;
};

export const createNoopController = (): AnimationController => ({
  setProgress: () => {},
  destroy: () => {},
});
```

---

## `progress.ts`

```ts
import gsap from "gsap";

export const clampProgress = (progress: number) =>
  gsap.utils.clamp(0, 1, progress);
```

---

## `index.ts`

```ts
export { createNoopController, type AnimationController } from "./controller";

export { clampProgress } from "./progress";
```

---

# 10. `animations/capability/intro`

## 역할

Intro Scene의 실제 움직임을 정의한다.

예시:

```txt
animations/capability/intro/
├─ capabilityIntro.animation.ts
├─ capabilityIntroProof.animation.ts
└─ index.ts
```

이 폴더에서는 더 이상 `querySelector`를 하지 않는다.

---

## `capabilityIntro.animation.ts`

`elements`를 받아서 움직임만 정의한다.

```ts
import gsap from "gsap";

import {
  clampProgress,
  createNoopController,
  type AnimationController,
} from "@/animations/_shared";

import type { CapabilityIntroAnimationElements } from "@/components/scenes/capability/dom";

const TITLE_INITIAL_SCALE = 3.8;

const CapabilityIntroAnimation = {
  create(elements: CapabilityIntroAnimationElements): AnimationController {
    const {
      visualField,
      titleLayer,
      eyebrow,
      title,
      subtitle,
      phase01,
      phase02,
    } = elements;

    if (
      !visualField ||
      !titleLayer ||
      !eyebrow ||
      !title ||
      !subtitle ||
      !phase01 ||
      !phase02
    ) {
      console.warn("[CapabilityIntroAnimation] Missing elements", elements);

      return createNoopController();
    }

    const timeline = gsap.timeline({
      paused: true,
      defaults: {
        ease: "none",
      },
    });

    // Initial
    gsap.set(visualField, {
      autoAlpha: 1,
      scale: 1,
      transformOrigin: "center center",
    });

    gsap.set(titleLayer, {
      autoAlpha: 1,
      y: 0,
    });

    gsap.set(title, {
      autoAlpha: 0,
      scale: TITLE_INITIAL_SCALE,
      y: 0,
      transformOrigin: "center center",
    });

    gsap.set(eyebrow, {
      autoAlpha: 0,
      y: 16,
    });

    gsap.set(subtitle, {
      autoAlpha: 0,
      y: 18,
    });

    gsap.set([phase01, phase02], {
      autoAlpha: 0,
      y: 28,
    });

    // Title reveal
    timeline.to(
      title,
      {
        autoAlpha: 1,
        duration: 0.12,
      },
      0.08
    );

    // Title scale
    timeline.to(
      title,
      {
        scale: 1,
        duration: 0.42,
        ease: "power2.out",
      },
      0.16
    );

    // Header reveal
    timeline
      .to(
        eyebrow,
        {
          autoAlpha: 1,
          y: 0,
          duration: 0.1,
          ease: "power2.out",
        },
        0.56
      )
      .to(
        subtitle,
        {
          autoAlpha: 1,
          y: 0,
          duration: 0.12,
          ease: "power2.out",
        },
        0.62
      );

    // Phase 01
    timeline.to(
      phase01,
      {
        autoAlpha: 1,
        y: 0,
        duration: 0.14,
        ease: "power2.out",
      },
      0.74
    );

    // Phase transition
    timeline
      .to(
        phase01,
        {
          autoAlpha: 0,
          y: -24,
          duration: 0.12,
          ease: "power2.inOut",
        },
        0.86
      )
      .to(
        phase02,
        {
          autoAlpha: 1,
          y: 0,
          duration: 0.14,
          ease: "power2.out",
        },
        0.9
      );

    const setProgress = (progress: number) => {
      timeline.progress(clampProgress(progress));
    };

    const destroy = () => {
      timeline.kill();

      gsap.set(
        [visualField, titleLayer, eyebrow, title, subtitle, phase01, phase02],
        {
          clearProps: "all",
        }
      );
    };

    return {
      setProgress,
      destroy,
    };
  },
};

export default CapabilityIntroAnimation;
```

---

# 11. 전체 흐름

Capability animation 흐름은 다음 순서로 이어진다.

```txt
CapabilityStage.tsx
↓
useCapabilityStageAnimation(stageRef)
↓
getCapabilityStageElements(stage)
↓
createCapabilityStageControllers(stageElements)
↓
getCapabilityIntroAnimationElements(elements.intro)
↓
CapabilityIntroAnimation.create(introAnimationElements)
↓
ScrollTrigger progress
↓
controllers.intro.setProgress(progress)
↓
GSAP timeline progress 변경
```

---

# 12. 최종 역할 한 줄 정리

## Scene

```txt
화면 DOM을 만든다.
```

## Scene dom

```txt
animation target DOM을 찾는다.
```

## Stage

```txt
큰 scene 구간을 찾고 scroll 흐름을 관리한다.
```

## Stage controller helper

```txt
scene dom elements를 animation controller에 연결한다.
```

## Animation

```txt
전달받은 DOM을 실제로 움직인다.
```

---

# 13. 헷갈리면 안 되는 기준

## `components/stages/constants`

큰 구간 selector만 둔다.

```txt
.js-capability-intro
.js-capability-intro-proof
.js-capability-closing
```

---

## `components/scenes/capability/dom`

세부 animation target selector를 둔다.

```txt
.js-capability-intro-title
.js-capability-intro-subtitle
.js-capability-intro-proof-character
.js-capability-intro-proof-quote
```

---

## `animations`

`querySelector`를 하지 않는다.

```txt
받은 element로 GSAP만 작성한다.
```

---

# 14. 작업 순서

리팩토링은 다음 순서로 진행한다.

```txt
1. components/scenes/capability/dom 만들기
2. capabilityIntro.selectors.ts 만들기
3. capabilityIntro.elements.ts 만들기
4. dom/index.ts 만들기
5. capabilityIntro.animation.ts를 elements 인자로 받게 수정
6. capabilityIntroProof.animation.ts도 동일하게 수정
7. capabilityStageControllers.ts에서 scene dom helper를 거쳐 animation create
8. 빌드 에러 확인
9. 스크롤 동작 확인
```

---

# 15. 최종 결론

```txt
Scene은 DOM을 알고,
Scene dom은 target을 수집하고,
Stage는 흐름을 연결하고,
Animation은 움직임을 정의한다.
```

이 기준으로 가면 selector와 animation 책임이 섞이지 않는다.

Stage는 큰 흐름만 보고,
Scene은 자신의 DOM 구조를 책임지고,
Animation은 전달받은 DOM으로 움직임만 만든다.
