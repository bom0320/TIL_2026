# Stage Animation 리팩토링 정리

## 1. 리팩토링 목적

포트폴리오의 `IntroStage`, `CapabilityStage`, `ContactStage`에서 스크롤 애니메이션을 관리하는 방식이 서로 달라지고 있었다.

기존에는 각 Stage hook 안에서 다음 역할들이 섞여 있었다.

```txt
DOM 탐색
애니메이션 controller 생성
ScrollTrigger 등록
progress 초기화
cleanup 처리
```

이 구조는 처음에는 빠르게 구현하기 좋았지만, Stage가 늘어나면서 다음 문제가 생겼다.

- `querySelector` 위치가 파일마다 달라짐
- animation controller 반환 형식이 통일되지 않음
- `ScrollTrigger` 등록 방식이 Stage마다 다르게 보임
- 어떤 애니메이션이 역재생 가능한지, 한 번만 실행되는지 코드에서 바로 파악하기 어려움
- cleanup 책임이 hook 내부에 과하게 몰림

그래서 Stage 단위 애니메이션 구조를 다음 기준으로 정리했다.

---

## 2. 최종 구조 기준

Stage 애니메이션은 다음 흐름으로 통일했다.

```txt
Stage root
→ ref로 잡는다

Stage 내부 animation target
→ get*StageElements(stage)에서 수집한다

Animation controller 생성
→ create*StageControllers(elements)에서 담당한다

ScrollTrigger 등록
→ use*StageAnimation.ts에서 담당한다

초기화 / 정리
→ reset*StageControllers()
→ destroy*StageControllers()
```

즉, 각 파일의 책임을 다음처럼 나눴다.

```txt
Stage.tsx
→ stageRef 생성 및 hook 호출

get*StageElements.ts
→ DOM 수집 담당

*StageControllers.ts
→ animation controller 생성 / reset / destroy 담당

use*StageAnimation.ts
→ ScrollTrigger 연결 담당
```

---

## 3. Stage root와 js class 규칙

Stage root는 `ref`로 직접 잡기 때문에 `js-*` class를 붙이지 않는다.

예시:

```tsx
<section ref={stageRef} className="capability-stage">
```

```tsx
<section ref={stageRef} className="intro-stage">
```

```tsx
<section ref={stageRef} className="contact-stage">
```

반면 Stage 내부의 실제 animation target은 `js-*` class를 사용한다.

예시:

```tsx
<section className="hero js-intro-hero">
```

```tsx
<section className="life-motion js-intro-life-motion">
```

```tsx
<div className="about-scenes js-intro-about">
```

```tsx
<div className="contact-intro js-contact-intro">
```

```tsx
<footer className="contact-footer js-contact-footer">
```

정리하면:

```txt
Stage root
→ ref로 잡음
→ js-* class 불필요

하위 animation target
→ querySelector로 찾아야 함
→ js-* class 사용
```

---

## 4. selectors의 역할

`*_STAGE_SELECTORS`는 JSX `className`에 직접 넣기 위한 값이 아니라, `querySelector`에서 사용하기 위한 selector 상수다.

예시:

```ts
export const INTRO_STAGE_SELECTORS = {
  hero: ".js-intro-hero",
  lifeMotion: ".js-intro-life-motion",
  about: ".js-intro-about",
} as const;
```

```ts
export const CONTACT_STAGE_SELECTORS = {
  intro: ".js-contact-intro",
  footer: ".js-contact-footer",
} as const;
```

`root` selector는 만들지 않는다.
Stage root는 이미 `stageRef.current`로 잡고 있기 때문이다.

---

## 5. getStageElements 패턴

DOM 탐색은 hook 안에서 직접 하지 않고, `get*StageElements(stage)`로 분리했다.

예시:

```ts
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
```

```ts
export function getContactStageElements(stage: HTMLElement) {
  return {
    intro: stage.querySelector<HTMLElement>(CONTACT_STAGE_SELECTORS.intro),

    footer: stage.querySelector<HTMLElement>(CONTACT_STAGE_SELECTORS.footer),
  };
}
```

이렇게 하면 hook은 DOM을 직접 찾지 않고, 이미 수집된 `elements`만 사용한다.

---

## 6. animation controller 반환 형식 통일

스크롤 progress로 제어되는 animation은 모두 다음 형태의 controller를 반환하도록 맞췄다.

```ts
type ProgressAnimationController = {
  setProgress: (progress: number) => void;
  destroy: () => void;
};
```

이렇게 하면 Stage hook에서 모든 animation을 동일하게 다룰 수 있다.

```ts
controllers.intro.setProgress(0);
controllers.footer.setProgress(0);

controllers.intro.destroy();
controllers.footer.destroy();
```

`gsap.timeline()`을 직접 반환하던 animation도 controller 형태로 감쌌다.

기존:

```ts
return timeline;
```

변경:

```ts
return {
  setProgress(progress: number) {
    timeline.progress(clampProgress(progress));
  },

  destroy() {
    timeline.kill();
  },
};
```

---

## 7. null-safe animation controller

`querySelector`의 결과는 항상 `HTMLElement | null`이 될 수 있다.
따라서 animation create 함수는 `null`을 받아도 깨지지 않게 작성했다.

```ts
if (!target) {
  return {
    setProgress: () => {},
    destroy: () => {},
  };
}
```

이 패턴을 사용하면 DOM target이 없는 경우에도 Stage hook이 안전하게 동작한다.

---

## 8. ScrollTrigger 등록 helper

공통으로 반복되는 ScrollTrigger 등록 패턴은 helper로 분리했다.

### registerProgressTrigger

스크롤 progress를 그대로 controller에 전달한다.

```txt
스크롤 아래로: 0 → 1
스크롤 위로: 1 → 0
```

즉, 역재생 가능한 애니메이션에 사용한다.

```ts
registerProgressTrigger({
  triggerElement,
  config,
  controller,
  registerTrigger,
});
```

### registerMaxProgressTrigger

현재까지 도달한 최대 progress만 유지한다.

```txt
스크롤 아래로: 0 → 1
스크롤 위로: 1 유지
```

즉, 한 번 reveal된 뒤 되감기지 않아야 하는 애니메이션에 사용한다.

```ts
registerMaxProgressTrigger({
  triggerElement,
  config,
  controller,
  registerTrigger,
});
```

---

## 9. Stage별 적용 기준

### IntroStage

`IntroStage`는 `stageRef → elements → controllers` 구조로 정리했다.

단, ScrollTrigger 등록은 직접 작성했다.

이유:

```txt
heroToLife
→ viewport height 기반 end 계산이 필요함

lifeToAbout
→ 하나의 progress를 enterProgress / sceneProgress로 나눠서 사용함
```

따라서 `registerProgressTrigger`로 억지로 추상화하지 않고, 직접 `createScrollTrigger`를 사용했다.

---

### CapabilityStage

`CapabilityStage`는 여러 reveal 구간이 있어 max-progress 대상만 배열로 묶었다.

```ts
const MAX_PROGRESS_TARGET_KEYS = [
  "introProof",
  "structure",
  "ai",
  "visual",
  "navigatorIntro",
] as const;
```

각 구간은 다음 기준으로 나눴다.

```txt
intro
→ registerProgressTrigger
→ 역재생 가능

introProof / structure / ai / visual / navigatorIntro
→ registerMaxProgressTrigger
→ 한 번 reveal되면 유지

closing
→ registerProgressTrigger
→ 역재생 가능

navigatorPin
→ pin / activeIndex / layer transition이 필요하므로 직접 createScrollTrigger 사용
```

`closing`은 처음에는 max-progress 대상에 포함되어 있었지만, 역재생 가능해야 하므로 일반 progress trigger로 분리했다.

---

### ContactStage

`ContactStage`도 `elements → controllers → trigger 등록` 구조로 정리했다.

동작 기준은 다음과 같다.

```txt
contact intro
→ registerMaxProgressTrigger
→ 한 번 나타나면 유지

contact footer
→ registerProgressTrigger
→ 스크롤 방향에 따라 역재생 가능
```

즉, Contact에서는 intro와 footer가 서로 다른 동작을 가지므로 key 배열 반복문으로 묶지 않고 명시적으로 등록했다.

---

## 10. 최종 리팩토링 기준

Stage animation hook에서 지켜야 할 기준은 다음과 같다.

```txt
1. Stage root는 ref로 잡는다.
2. hook 내부에서 직접 querySelector를 하지 않는다.
3. DOM 탐색은 get*StageElements.ts에서만 한다.
4. animation 생성은 create*StageControllers.ts에서 한다.
5. animation controller는 setProgress / destroy를 가진다.
6. ScrollTrigger 등록은 use*StageAnimation.ts에서 한다.
7. 역재생 가능한 애니메이션은 registerProgressTrigger를 사용한다.
8. 한 번만 reveal되는 애니메이션은 registerMaxProgressTrigger를 사용한다.
9. pin, activeIndex, progress split처럼 특수한 케이스는 직접 createScrollTrigger를 사용한다.
10. cleanup은 trigger.kill()과 destroy*StageControllers()로 정리한다.
```

---

## 11. 앞으로 애니메이션 폴더에서 정리할 부분

Stage hook 구조는 어느 정도 정리되었지만, `animations/` 폴더는 아직 추가 정리가 필요하다.

다음 기준으로 정리할 예정이다.

```txt
1. 모든 scroll-driven animation은 controller 형태로 반환한다.
2. document.querySelector 사용을 줄이고 element 주입 방식으로 바꾼다.
3. null-safe controller 패턴을 통일한다.
4. cleanup에서 clearProps 범위를 정리한다.
5. timeline 반환과 controller 반환이 섞이지 않게 한다.
6. 애니메이션 파일명과 export 방식을 통일한다.
```

이후 컴포넌트 리팩토링은 애니메이션 형식이 더 안정된 뒤 진행하는 것이 좋다.
