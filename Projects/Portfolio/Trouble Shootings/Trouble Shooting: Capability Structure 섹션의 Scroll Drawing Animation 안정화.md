# Trouble Shooting: Capability Structure 섹션의 Scroll Drawing Animation 안정화

## 1. 문제 배경

포트폴리오의 `CapabilityStage` 안에는 사용자의 경험 설계 역량을 보여주는 `ExperienceCapabilityScene`이 있고, 그 안에 `StructureCapabilityBlock`이 포함되어 있다.

이 Structure 블록은 단순히 요소가 fade-in 되는 섹션이 아니라, 다음과 같은 순서로 “구조가 그려지는 듯한” 스크롤 기반 애니메이션을 의도했다.

1. 섹션 헤더가 등장한다.
2. 중앙 core node가 나타난다.
3. core에서 stem이 아래로 그려진다.
4. branch line이 이어진다.
5. 각 node가 순차적으로 연결된다.
6. 연결된 구조를 기반으로 하단 카드들이 등장한다.

즉, 목표는 일반적인 reveal animation이 아니라 **scroll progress에 따라 구조가 차근차근 그려지는 scroll drawing animation**이었다.

하지만 구현 과정에서 다음과 같은 문제가 반복적으로 발생했다.

- 모바일에서는 애니메이션이 이미 100% 완료된 상태로 보였다.
- 데스크톱에서도 Structure map이 보일 때 이미 최종 상태로 고정되어 있었다.
- ScrollTrigger progress는 정상적으로 0 → 1까지 찍히지만, 화면에서는 “탁” 하고 한 번에 나타나는 것처럼 보였다.
- 모바일과 데스크톱의 레이아웃 구조가 달라 같은 애니메이션 로직으로는 자연스럽게 동작하지 않았다.
- 한 번 실행된 애니메이션은 되감기되지 않아야 했지만, 동시에 스크롤에 따라 차근차근 그려지는 느낌은 유지해야 했다.

결과적으로 이 문제는 단일 원인이 아니라, **ScrollTrigger config, GSAP timeline 구조, max progress 제어, CSS transform, responsive layout 차이**가 함께 얽힌 복합적인 이슈였다.

---

## 2. 초기 구조

기존 `CapabilityStage`는 하나의 scroll config만 사용하고 있었다.

```tsx
export const CAPABILITY_STAGE_SCROLL_CONFIG = {
  structure: {
    start: "top 78%",
    end: "bottom 62%",
    scrub: 1.1,
  },
  ...
} as const;
```

그리고 hook에서는 각 섹션을 반복 등록했다.

```tsx
const MAX_PROGRESS_TARGET_KEYS = [
  "introProof",
  "structure",
  "ai",
  "visual",
  "navigatorIntro",
] as const;

MAX_PROGRESS_TARGET_KEYS.forEach((key) => {
  registerMaxProgressTrigger({
    triggerElement: elements[key],
    config: CAPABILITY_STAGE_SCROLL_CONFIG[key],
    controller: controllers[key],
    registerTrigger,
  });
});
```

`registerMaxProgressTrigger`는 말 그대로 현재 progress보다 더 큰 값만 반영하는 구조다.

즉, 스크롤을 아래로 내리며 progress가 증가할 때만 애니메이션이 진행되고, 다시 위로 올려도 progress가 감소하지 않는다.

이 구조는 이번 요구사항과 잘 맞았다.

요구사항은 다음과 같았다.

```
스크롤에 따라 차근차근 그려져야 한다.
하지만 한 번 그려진 뒤에는 되감기되면 안 된다.
```

따라서 `Structure`는 일반 `registerProgressTrigger`가 아니라, 최종적으로는 `registerMaxProgressTrigger`를 유지하는 방향이 맞았다.

---

## 3. 첫 번째 문제: desktop / mobile scroll config를 분리해야 했다

처음에는 desktop과 mobile이 같은 scroll config를 공유했다.

하지만 Structure 블록의 레이아웃은 breakpoint에 따라 크게 달라졌다.

### Desktop

- branch가 가로 방향으로 그려진다.
- node들이 좌우로 넓게 배치된다.
- map과 card grid가 한 화면에서 비교적 넓게 보인다.

### Mobile / Tablet

- branch가 세로 방향으로 바뀐다.
- node들이 세로 라인을 중심으로 좌우 번갈아 배치된다.
- 화면 높이가 좁기 때문에 header, map, cards가 긴 세로 흐름으로 배치된다.

즉, 같은 `start`, `end`, `scrub` 값으로는 자연스러운 scroll timing을 맞출 수 없었다.

그래서 config를 분리했다.

```tsx
export const CAPABILITY_STAGE_DESKTOP_SCROLL_CONFIG = { ... };

export const CAPABILITY_STAGE_MOBILE_SCROLL_CONFIG = {
  ...CAPABILITY_STAGE_DESKTOP_SCROLL_CONFIG,
  structure: {
    start: "...",
    end: "...",
    scrub: ...,
  },
};
```

그리고 `gsap.matchMedia()`를 사용해 viewport에 따라 다른 config를 등록했다.

```tsx
const media = gsap.matchMedia();

media.add("(min-width: 901px)", () => {
  return setupCapabilityTriggers(CAPABILITY_STAGE_DESKTOP_SCROLL_CONFIG);
});

media.add("(max-width: 900px)", () => {
  return setupCapabilityTriggers(CAPABILITY_STAGE_MOBILE_SCROLL_CONFIG);
});
```

이 작업을 하면서 기존에는 필요 없던 `CapabilityStageScrollConfig` 타입도 생성했다.

이유는 config가 하나일 때는 타입 추론만으로 충분했지만, desktop/mobile처럼 같은 역할의 config 객체가 여러 개 생기고, 하나의 함수에 교체해서 넣는 구조가 되면서 공통 타입이 필요해졌기 때문이다.

```tsx
type ScrollTriggerConfig = {
  start: string;
  end: string;
  scrub: number;
};

type NavigatorPinScrollConfig = {
  start: string;
  scrub: number;
  itemScrollLengthMultiplier: number;
  anticipatePin: number;
};

export type CapabilityStageScrollConfig = Record<
  CapabilityStageNormalScrollKey,
  ScrollTriggerConfig
> & {
  navigatorPin: NavigatorPinScrollConfig;
};
```

여기서 `navigatorPin`만 별도로 분리한 이유는 일반 섹션과 config 구조가 다르기 때문이다.

일반 섹션은 다음 형태를 가진다.

```tsx
{
  start: string;
  end: string;
  scrub: number;
}
```

하지만 `navigatorPin`은 동적으로 `end`를 계산하고, `itemScrollLengthMultiplier`, `anticipatePin`을 사용한다.

```tsx
{
  start: string;
  scrub: number;
  itemScrollLengthMultiplier: number;
  anticipatePin: number;
}
```

따라서 `navigatorPin`을 제외한 key만 `Record`로 묶고, `navigatorPin`은 별도로 정의했다.

---

## 4. 두 번째 문제: key 분리와 타입 구조 정리

기존 hook 내부에는 반복 등록 대상 key가 직접 선언되어 있었다.

```tsx
const MAX_PROGRESS_TARGET_KEYS = [
  "introProof",
  "structure",
  "ai",
  "visual",
  "navigatorIntro",
] as const;
```

하지만 `elements`, `controllers`, `scrollConfig` 모두 동일한 key를 사용하고 있었다.

예를 들면 `structure` 하나의 key로 다음 세 가지가 연결된다.

```tsx
elements.structure;
controllers.structure;
scrollConfig.structure;
```

이 구조를 명확하게 관리하기 위해 stage key를 별도 파일로 분리했다.

```tsx
export const CAPABILITY_STAGE_KEYS = [
  "intro",
  "introProof",
  "structure",
  "ai",
  "visual",
  "navigatorIntro",
  "navigatorPin",
  "closing",
] as const;

export type CapabilityStageKey = (typeof CAPABILITY_STAGE_KEYS)[number];

export const CAPABILITY_STAGE_PROGRESS_KEYS = [
  "introProof",
  "structure",
  "ai",
  "visual",
  "navigatorIntro",
] as const;

export type CapabilityStageProgressKey =
  (typeof CAPABILITY_STAGE_PROGRESS_KEYS)[number];
```

이 작업의 목적은 단순히 타입을 줄이기 위한 것이 아니라, stage 내부에서 반복되는 section key를 한 곳에서 관리하기 위해서였다.

즉, key는 단순 문자열이 아니라 다음 세 가지를 연결하는 공통 이름표 역할을 한다.

```
structure
 ├─ elements.structure
 ├─ controllers.structure
 └─ scrollConfig.structure
```

이후 `constants/index.ts`에서 barrel export를 통해 깊은 상대경로 import를 줄였다.

```tsx
export {
  CAPABILITY_STAGE_KEYS,
  CAPABILITY_STAGE_PROGRESS_KEYS,
  type CapabilityStageKey,
  type CapabilityStageProgressKey,
} from "./capabilityStageKeys";
```

이로 인해 다른 파일에서는 직접 `capabilityStageKeys.ts` 또는 `capabilityStageSelectors.ts`로 들어가지 않고, constants index를 통해 import할 수 있게 되었다.

---

## 5. 세 번째 문제: `.js-*` selector 연결 여부 확인

애니메이션이 “아예 안 먹는 것처럼” 보였기 때문에 처음에는 `.js-*` selector 연결 문제를 의심했다.

Structure 애니메이션은 다음 흐름으로 DOM을 찾는다.

```tsx
JSX className
↓
CAPABILITY_STAGE_SELECTORS
↓
getCapabilityStageElements()
↓
getStructureCapabilityAnimationElements()
↓
StructureCapabilityAnimation.create()
```

Structure root는 다음 selector로 수집된다.

```
structure: ".js-capability-structure"
```

그리고 Structure 내부 요소는 다음 selector로 수집된다.

```tsx
export const CAPABILITY_EXPERIENCE_SELECTORS = {
  structure: {
    header: ".js-structure-capability-header",
    core: ".js-structure-capability-core",
    stem: ".js-structure-capability-stem",
    branch: ".js-structure-capability-branch",
    nodes: ".js-structure-capability-node",
    cards: ".js-structure-capability-card",
    cardIcons: ".js-structure-capability-card-icon",
    cardTitles: ".js-structure-capability-card-title",
    cardMessages: ".js-structure-capability-card-message",
    cardDescs: ".js-structure-capability-card-desc",
  },
};
```

DOM 수집 함수는 다음과 같다.

```tsx
export const getStructureCapabilityAnimationElements = (
  root: HTMLElement | null
): StructureCapabilityAnimationElements => {
  const selectors = CAPABILITY_EXPERIENCE_SELECTORS.structure;

  return {
    root,
    header: queryElement(root, selectors.header),
    core: queryElement(root, selectors.core),
    stem: queryElement(root, selectors.stem),
    branch: queryElement(root, selectors.branch),
    nodes: queryElements(root, selectors.nodes),
    cards: queryElements(root, selectors.cards),
    cardIcons: queryElements(root, selectors.cardIcons),
    cardTitles: queryElements(root, selectors.cardTitles),
    cardMessages: queryElements(root, selectors.cardMessages),
    cardDescs: queryElements(root, selectors.cardDescs),
  };
};
```

콘솔 로그를 통해 확인한 결과:

```tsx
root: HTMLElement
header: HTMLElement
core: HTMLElement
stem: HTMLElement
branch: HTMLElement
nodes: 6
cards: 6
...
```

즉, `.js-*` 연결 자체는 문제가 아니었다.

이 과정을 통해 원인을 selector 문제가 아니라 **progress와 timeline 설계 문제**로 좁힐 수 있었다.

---

## 6. 네 번째 문제: ScrollTrigger progress는 정상인데 화면은 최종 상태처럼 보였다

다음 로그를 통해 Structure progress가 정상적으로 증가하는 것을 확인했다.

```tsx
Structure progress 0.004
Structure progress 0.023
Structure progress 0.040
...
Structure progress 0.991
Structure progress 1
```

이 로그는 중요한 의미를 가진다.

```
ScrollTrigger 연결 정상
controller 연결 정상
setProgress 호출 정상
```

즉 문제는 ScrollTrigger가 실행되지 않는 것이 아니었다.

그럼에도 화면에서는 애니메이션이 보이지 않고 이미 완성된 상태처럼 보였다.

원인은 `start`, `end` 구간이 너무 짧거나, 사용자가 실제 map을 볼 때 이미 progress가 1에 도달했기 때문이었다.

처음 모바일 config는 다음처럼 설정되어 있었다.

```tsx
structure: {
  start: "top bottom",
  end: "top 30%",
  scrub: 0.45,
}
```

이 설정은 `Structure` article의 top이 화면 하단에 닿으면 시작하고, top이 화면 30% 지점까지 올라오면 끝나는 구조다.

하지만 모바일에서 Structure 블록은 header, map, cards를 포함하는 긴 세로 흐름이므로, map이 실제로 화면 중앙에 보일 때는 이미 progress가 100%가 되어 있었다.

그래서 사용자는 애니메이션이 “안 먹는 것”처럼 느꼈지만, 실제로는 **너무 일찍 끝나고 있었던 것**이다.

---

## 7. 다섯 번째 문제: desktop과 mobile의 scroll timing 기준이 달라야 했다

처음에는 mobile과 desktop 모두 `top → top` 기준으로 조정했다.

예:

```tsx
structure: {
  start: "top 76%",
  end: "top 36%",
  scrub: 0.7,
}
```

하지만 이 방식은 너무 짧게 끝났다.

반대로 `bottom` 기준을 너무 강하게 쓰면 너무 느려졌다.

예:

```tsx
structure: {
  start: "top 78%",
  end: "bottom 64%",
  scrub: 0.75,
}
```

이 경우 Structure 블록 전체의 bottom이 기준이 되기 때문에 progress 구간이 너무 길어졌다.

결국 desktop과 mobile에서 각각 다른 기준값을 잡았다.

최종적으로 desktop은 구버전의 긴 scroll drawing 느낌을 참고해 다음처럼 조정했다.

```tsx
structure: {
  start: "top 78%",
  end: "bottom 62%",
  scrub: 1.1,
}
```

mobile은 너무 길어지면 답답해지므로 `top → center` 기준을 사용했다.

```tsx
structure: {
  start: "top 82%",
  end: "center 36%",
  scrub: 0.85,
}
```

이렇게 나눈 이유는 다음과 같다.

```
desktop
- 화면이 넓고 map이 빠르게 보인다.
- 노드와 branch가 넓게 퍼져 있어 drawing 구간이 길어야 자연스럽다.
- bottom 기준을 사용해 전체 블록 흐름에 맞춘다.

mobile
- 화면이 좁고 세로 흐름이 길다.
- bottom 기준을 쓰면 너무 느려진다.
- center 기준을 사용해 적당한 길이로 drawing을 유지한다.
```

---

## 8. 여섯 번째 문제: GSAP transform이 CSS transform을 덮는 문제

SCSS에서는 core, stem, branch, node를 중앙 정렬하기 위해 다음과 같은 transform을 사용하고 있었다.

```scss
transform: translateX(-50%);
```

예:

```scss
.experience-capability-structure-map__core {
  left: 50%;
  transform: translateX(-50%);
}

.experience-capability-structure-map__branch {
  left: 50%;
  transform: translateX(-50%);
}

.experience-capability-structure-map__node--1 {
  transform: translateX(-50%);
}
```

그런데 GSAP에서도 transform 계열 값을 사용하고 있었다.

```
scale: 0.74
scaleX: 0
scaleY: 0
y: 14
```

GSAP는 transform을 통합 관리하기 때문에, CSS에서 설정한 `translateX(-50%)`가 덮일 수 있다.

이 문제를 해결하기 위해 GSAP 초기 상태에서 `xPercent: -50`을 직접 추가했다.

```scss
gsap.set(core, {
  autoAlpha: 0,
  xPercent: -50,
  scale: 0.74,
  filter: "blur(10px)",
});
```

stem, branch, nodes에도 동일하게 적용했다.

```scss
gsap.set(stem, {
  xPercent: -50,
  scaleY: 0,
});
```

```scss
gsap.set(branch, {
  xPercent: -50,
  scaleX: isMobileLayout ? 1 : 0,
  scaleY: isMobileLayout ? 0 : 1,
});
```

```scss
gsap.set(nodes, {
  autoAlpha: 0,
  xPercent: -50,
  scale: 0.86,
  y: 14,
});
```

이 수정으로 GSAP가 transform을 제어하더라도 중앙 정렬이 깨지지 않게 만들었다.

---

## 9. 일곱 번째 문제: responsive layout에 따라 branch 방향이 달라졌다

desktop에서는 branch가 가로선이다.

```scss
width: min(860px, 90%);
height: 3px;
```

mobile/tablet에서는 branch가 세로선으로 바뀐다.

```scss
width: 3px;
height: 260px;
```

처음 애니메이션은 desktop 기준으로만 작성되어 있었다.

```scss
gsap.set(branch, {
  scaleX: 0,
});

timeline.to(branch, {
  scaleX: 1,
});
```

하지만 mobile에서는 branch가 세로 방향이므로 `scaleX`가 아니라 `scaleY`가 필요하다.

이를 해결하기 위해 viewport에 따라 scale property를 분기했다.

```tsx
const isMobileLayout = window.matchMedia("(max-width: 900px)").matches;
const branchScaleProperty = isMobileLayout ? "scaleY" : "scaleX";
```

초기 상태도 분기했다.

```scss
gsap.set(branch, {
  xPercent: -50,
  scaleX: isMobileLayout ? 1 : 0,
  scaleY: isMobileLayout ? 0 : 1,
  transformOrigin: isMobileLayout ? "top center" : "center center",
});
```

timeline에서는 computed property를 사용했다.

```tsx
timeline.to(branch, {
  [branchScaleProperty]: 1,
  duration: 1.4,
  ease: "none",
});
```

이렇게 해서 desktop에서는 가로로, mobile에서는 세로로 branch가 그려지도록 분리했다.

---

## 10. 여덟 번째 문제: mobile node line이 CSS variable을 덮고 있었다

base SCSS에서는 node line이 CSS variable을 사용해 animation될 수 있게 되어 있었다.

```scss
transform: translateX(-50%) scaleY(var(--node-line-scale, 1));
opacity: var(--node-line-opacity, 1);
```

하지만 tablet 이하 반응형 SCSS에서는 다음처럼 덮고 있었다.

```scss
transform: translateY(-50%);
opacity: 1;
```

이러면 GSAP에서 설정한 값이 반영되지 않는다.

```scss
"--node-line-scale":0,"--node-line-opacity": 0;
```

즉, mobile에서는 line drawing이 아니라 처음부터 선이 보이는 상태가 될 수 있었다.

이를 수정하기 위해 tablet SCSS에서 transform과 opacity를 다시 CSS variable 기반으로 연결했다.

```scss
.experience-capability-structure-map__node::before {
  transform: translateY(-50%) scaleX(var(--node-line-scale, 1));
  transform-origin: left center;
  opacity: var(--node-line-opacity, 1);
}
```

좌우 방향에 따라 transform-origin도 분리했다.

```scss
.experience-capability-structure-map__node--1::before,
.experience-capability-structure-map__node--3::before,
.experience-capability-structure-map__node--5::before {
  left: 100%;
  transform-origin: left center;
}

.experience-capability-structure-map__node--2::before,
.experience-capability-structure-map__node--4::before,
.experience-capability-structure-map__node--6::before {
  right: 100%;
  transform-origin: right center;
}
```

이 수정으로 mobile/tablet에서도 node line이 GSAP progress에 따라 차근차근 그려지게 만들었다.

---

## 11. 아홉 번째 문제: timeline이 reveal animation처럼 짜여 있었다

처음 Structure timeline은 대부분 duration이 매우 짧았다.

```scss
duration: 0.07
duration: 0.08
duration: 0.1
```

node 등장 간격도 매우 촘촘했다.

```tsx
const startAt = `nodeDraw+=${orderIndex * 0.035}`;
```

카드 내부 요소도 거의 붙어서 등장했다.

```tsx
nodeDraw += 0.25;
nodeDraw += 0.28;
nodeDraw += 0.31;
nodeDraw += 0.34;
```

이 구조는 scroll drawing animation이 아니라, 짧은 reveal animation에 가깝다.

그래서 scroll config를 아무리 늘려도 사용자가 보기에는 “타닥” 하고 나타나는 느낌이 났다.

문제를 해결하기 위해 timeline 자체를 더 긴 구간으로 재설계했다.

핵심은 duration을 실제 시간으로 보지 않고, **scroll progress 안에서 차지하는 비율**로 보는 것이다.

```tsx
timeline
  .to(header, { duration: 1.2 }, 0)
  .to(core, { duration: 1 }, 1)
  .to(stem, { duration: 1 }, 2)
  .to(branch, { duration: 1.4 }, 3);
```

node도 절대 위치를 사용해 순차적으로 배치했다.

```tsx
const startAt = 4.4 + orderIndex * 0.62;
```

각 node는 line이 먼저 그려지고, 이후 node 원이 나타나도록 했다.

```tsx
timeline
  .to(
    node,
    {
      "--node-line-opacity": 1,
      "--node-line-scale": 1,
      duration: 0.48,
    },
    startAt
  )

  .to(
    node,
    {
      autoAlpha: 1,
      scale: 1,
      y: 0,
      filter: "blur(0px)",
      duration: 0.42,
    },
    startAt + 0.36
  );
```

cards는 node가 모두 그려진 뒤 등장하도록 뒤쪽 구간으로 밀었다.

```tsx
.to(cards, {
  autoAlpha: 1,
  y: 0,
  duration: 1.1,
  stagger: {
    each: 0.18,
    from: "start",
  },
}, 8.4)
```

이렇게 timeline 자체를 긴 scroll drawing 구조로 재설계하면서, 노드와 라인이 스크롤에 따라 차근차근 연결되는 흐름을 만들 수 있었다.

---

## 12. 열 번째 문제: `gsap.to(timeline, { progress })`가 스크롤 드로잉을 방해했다

중간에 animation을 부드럽게 만들기 위해 다음과 같이 timeline progress 자체에 tween을 걸었다.

```tsx
gsap.to(timeline, {
  progress: nextProgress,
  duration: isMobileLayout ? 0.65 : 0.4,
  ease: "power3.out",
  overwrite: true,
});
```

하지만 이 방식은 scroll drawing animation과 맞지 않았다.

이 코드는 scroll position과 timeline progress를 1:1로 연결하는 것이 아니라, progress를 별도 tween으로 따라가게 만든다.

즉, 스크롤에 따라 정확히 선이 그려지는 게 아니라, 스크롤이 멈춘 뒤 timeline이 부드럽게 따라오는 방식이 된다.

원하는 것은 다음이었다.

```
스크롤을 조금 내리면 그만큼만 선이 그려진다.
스크롤을 더 내리면 다음 노드가 연결된다.
다시 올려도 되감기되지는 않는다.
```

따라서 `gsap.to(timeline, { progress })`는 제거하고, 다시 직접 progress를 연결했다.

```tsx
const setProgress = (progress: number) => {
  timeline.progress(clampProgress(progress));
};
```

되감기 방지는 animation 내부가 아니라 `registerMaxProgressTrigger`에서 처리하도록 역할을 분리했다.

---

## 13. 최종 해결 방향

최종적으로 Structure animation은 다음 구조로 정리되었다.

### 1. Structure는 `registerMaxProgressTrigger` 유지

```tsx
CAPABILITY_STAGE_PROGRESS_KEYS.forEach((key) => {
  registerMaxProgressTrigger({
    triggerElement: elements[key],
    config: scrollConfig[key],
    controller: controllers[key],
    registerTrigger,
  });
});
```

이렇게 해야 한 번 실행된 애니메이션이 되감기되지 않는다.

### 2. `setProgress`는 timeline에 직접 연결

```tsx
const setProgress = (progress: number) => {
  timeline.progress(clampProgress(progress));
};
```

이렇게 해야 scroll progress와 drawing progress가 정확히 연결된다.

### 3. timeline은 긴 scroll drawing timeline으로 설계

```
header: 0 ~ 1.2
core: 1 ~ 2
stem: 2 ~ 3
branch: 3 ~ 4.4
nodes: 4.4 ~ 8.2
cards: 8.4 ~ 11.4
```

이처럼 각 단계가 timeline 안에서 넓은 구간을 차지하도록 구성했다.

### 4. desktop/mobile config는 별도 조정

desktop:

```scss
structure: {
  start: "top 78%",
  end: "bottom 62%",
  scrub: 1.1,
}
```

mobile:

```scss
structure: {
  start: "top 82%",
  end: "center 36%",
  scrub: 0.85,
}
```

desktop은 긴 drawing 구간을 확보하기 위해 `bottom` 기준을 사용했고, mobile은 너무 느려지지 않도록 `center` 기준을 사용했다.

---

## 14. 최종 결과

이 과정을 통해 다음 문제가 해결되었다.

- Structure animation이 화면에 보일 때 이미 100% 완료되어 있는 문제 해결
- mobile에서 branch가 잘못된 방향으로 그려지는 문제 해결
- GSAP transform과 CSS `translateX(-50%)` 충돌 문제 해결
- mobile node line이 CSS variable을 무시하고 고정 표시되는 문제 해결
- scroll progress에 맞춰 node line과 node가 차근차근 연결되는 흐름 구현
- 한 번 그려진 뒤에는 되감기되지 않도록 one-way reveal 유지
- desktop과 mobile의 layout 차이에 맞춘 scroll timing 분리

---

## 15. 배운 점

이번 Trouble Shooting에서 가장 중요한 포인트는 **애니메이션이 안 먹는 것처럼 보인다고 해서 곧바로 selector나 GSAP 코드 문제라고 단정하면 안 된다는 것**이었다.

실제로는 다음 순서로 원인을 좁혀야 했다.

```
1. DOM target이 잡히는가?
2. ScrollTrigger progress가 발생하는가?
3. progress가 controller까지 전달되는가?
4. timeline이 실제 target에 style을 적용하는가?
5. CSS transform과 충돌하지 않는가?
6. start/end 구간이 실제 사용자가 보는 화면 흐름과 맞는가?
7. animation timeline 자체가 scroll drawing에 적합한 구조인가?
```

이번 문제는 1~3번은 정상이었다.

진짜 문제는 5~7번에 있었다.

즉, 단순한 “애니메이션 버그”가 아니라 **스크롤 기반 인터랙션 설계 문제**였다.

특히 ScrollTrigger 기반 애니메이션에서는 `start`, `end`, `scrub`만 조정한다고 자연스러운 결과가 나오지 않는다.

timeline 내부 구조가 짧은 reveal animation으로 설계되어 있으면, scroll 구간을 아무리 조정해도 사용자는 “탁 하고 나타난다”고 느낄 수 있다.

따라서 scroll drawing animation을 만들 때는 다음 기준을 지켜야 한다.

```
- timeline 자체를 길게 설계한다.
- 각 단계가 scroll progress 안에서 충분한 구간을 가지게 한다.
- one-way animation은 registerMaxProgressTrigger에서 처리한다.
- animation 내부에서는 progress를 직접 연결한다.
- responsive layout이 달라지면 branch 방향, transform origin, scroll timing도 함께 분기한다.
```

이번 리팩토링을 통해 단순히 애니메이션을 고친 것이 아니라, `CapabilityStage`의 scroll animation architecture를 더 안정적으로 분리할 수 있었다.

최종적으로 `StructureCapabilityAnimation`은 다음 책임을 가진다.

```
- Structure 내부 요소의 초기 상태 설정
- desktop/mobile branch 방향 분기
- core/stem/branch/node/card 순서의 긴 drawing timeline 구성
- 전달받은 progress를 timeline에 반영
- destroy 시 GSAP style 정리
```

그리고 `useCapabilityStageAnimation`은 다음 책임만 가진다.

```
- viewport별 scroll config 선택
- ScrollTrigger 등록
- one-way progress 제어
- navigator pin 처리
- controller cleanup
```

이렇게 역할을 분리하면서, 이후 AI/Visual/Navigator 구간도 같은 방식으로 안정적으로 관리할 수 있는 기반을 만들었다.
