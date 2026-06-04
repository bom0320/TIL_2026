# Stage와 Scene의 차이

## 1. 핵심 정의

```txt
Stage = 흐름과 연결
Scene = 화면 구조
Animation = 움직임 방식
```

Stage와 Scene은 둘 다 화면을 구성하지만, 관심사가 다르다.

- **Stage**는 여러 Scene을 묶고 스크롤 흐름을 제어하는 큰 컨테이너다.
- **Scene**은 실제 화면 내부 DOM을 그리는 단위다.
- **Animation**은 Scene 안의 DOM을 어떤 방식으로 움직일지 정의한다.

---

# 2. Stage란?

Stage는 여러 Scene을 하나의 흐름으로 연결하는 큰 단위다.

예를 들어 `CapabilityStage`는 다음과 같은 Scene들을 묶는다.

```tsx
<section ref={stageRef} className="capability-stage">
  <CapabilityIntroScene />
  <ExperienceCapabilityScene />
  <CapabilityNavigatorScene />
  <CapabilityClosingScene />
</section>
```

Stage가 보는 것은 세부 DOM이 아니라 큰 구간이다.

```txt
CapabilityIntroScene
ExperienceCapabilityScene
CapabilityNavigatorScene
CapabilityClosingScene
```

## Stage의 관심사

Stage는 다음과 같은 것을 관리한다.

- intro scene은 어디에 있는가?
- proof section은 어디에 있는가?
- navigator pin 영역은 어디에 있는가?
- closing은 어디서 시작하는가?
- 스크롤 progress 몇 %에서 어떤 controller를 움직일 것인가?
- 어떤 Scene을 어떤 순서로 연결할 것인가?

즉, Stage는 **화면의 큰 구간과 스크롤 흐름**을 관리한다.

---

# 3. Scene이란?

Scene은 실제 화면 내부 DOM을 그리는 단위다.

예를 들어 `CapabilityIntroScene` 내부에는 다음과 같은 요소들이 들어갈 수 있다.

```txt
title layer
eyebrow
title
subtitle
phase 01
phase 02
character
proof point
quote
```

이런 요소들은 Stage가 직접 알 필요가 없다.
이들은 `CapabilityIntroScene` 내부 구조에 해당한다.

예시:

```tsx
<h2 className="capability-intro-title-layer__title js-capability-intro-title">
```

```tsx
<p className="capability-intro-proof__quote js-capability-intro-proof-quote">
```

이런 DOM은 Stage의 관심사가 아니라 Scene 내부의 세부 요소다.

---

# 4. “DOM 구조가 다르다”는 말의 의미

각 Scene은 서로 다른 DOM 구조를 가진다.

## CapabilityIntroScene 구조 예시

```txt
intro
├─ pinned narrative
│  ├─ visual field
│  ├─ title layer
│  │  ├─ eyebrow
│  │  ├─ title
│  │  └─ subtitle
│  └─ phase layer
│     ├─ phase01
│     └─ phase02
└─ visual proof
   ├─ proof points
   ├─ character
   └─ quote
```

## CapabilityNavigatorScene 구조 예시

```txt
navigator
├─ intro text
├─ pin area
├─ list
├─ monitor
├─ layer
└─ active content
```

## CapabilityClosingScene 구조 예시

```txt
closing
├─ statement
└─ scroll cue
```

즉, 각 Scene은 내부에서 움직일 대상이 다르다.

```txt
Intro
= title / phase / character / quote를 움직임

Navigator
= list / monitor / layer / active item을 움직임

Closing
= statement를 움직임
```

그래서 Scene마다 DOM 구조와 animation target이 달라진다.

---

# 5. Stage와 Scene 비교

| 구분          | Stage                            | Scene                            |
| ------------- | -------------------------------- | -------------------------------- |
| 역할          | 큰 흐름 제어                     | 실제 화면 구조 구성              |
| 관심사        | section, pin, progress, trigger  | title, card, quote, image, item  |
| selector 성격 | 큰 구간 selector                 | 내부 animation target selector   |
| 예시          | `.js-capability-intro`           | `.js-capability-intro-title`     |
| 파일 위치     | `components/stages`              | `components/scenes/...`          |
| 책임          | Scene들을 어떻게 이어붙일지 관리 | 장면 안에 어떤 DOM이 있는지 관리 |

---

# 6. Selector 성격도 다르다

Stage와 Scene의 책임이 다르기 때문에 selector의 성격도 달라진다.

---

## Stage selector

Stage selector는 큰 구간을 찾는다.

```ts
export const CAPABILITY_STAGE_SELECTORS = {
  intro: ".js-capability-intro",
  introPinned: ".js-capability-intro-pinned",
  introProof: ".js-capability-intro-proof",
  navigatorPin: ".js-capability-navigator-pin",
  closing: ".js-capability-closing",
} as const;
```

이 selector들은 이런 질문에 답한다.

```txt
이 Scene이 어디에 있는가?
이 pinned 영역이 어디인가?
이 scroll trigger 기준 DOM은 어디인가?
이 controller에 넘겨줄 scope DOM은 어디인가?
```

즉, Stage selector는 **큰 구간을 찾기 위한 selector**다.

---

## Scene 내부 selector

Scene 내부 selector는 해당 Scene 안의 세부 요소를 찾는다.

```ts
export const CAPABILITY_INTRO_ANIMATION_SELECTORS = {
  title: ".js-capability-intro-title",
  subtitle: ".js-capability-intro-subtitle",
  phase01: ".js-capability-intro-phase-01",
  phase02: ".js-capability-intro-phase-02",
} as const;
```

이 selector들은 이런 질문에 답한다.

```txt
이 Scene 안에서 어떤 글자를 움직일 것인가?
어떤 character를 reveal할 것인가?
어떤 quote를 fade in 시킬 것인가?
어떤 item을 stagger 처리할 것인가?
```

즉, Scene 내부 selector는 **animation target을 찾기 위한 selector**다.

---

# 7. Stage와 Scene의 궁극적인 차이

## 질문

```txt
Stage랑 Scene은 궁극적으로 뭐가 다른가?
```

## 답

```txt
Stage는 장면들을 어떻게 이어붙일지 담당하고,
Scene은 그 장면 안에 어떤 DOM이 있는지 담당한다.
```

더 짧게 정리하면 다음과 같다.

```txt
Stage = 흐름과 연결
Scene = 화면 구조
Animation = 움직임 방식
```

---

# 8. 최종 구조 논리

DOM을 만지는 selector나 elements는 Scene 쪽에 두는 것이 논리적으로 맞다.

왜냐하면 실제 DOM 구조를 아는 쪽은 Stage가 아니라 Scene이기 때문이다.

반대로 Stage는 각 Scene 내부의 title, quote, character 같은 세부 요소까지 알 필요가 없다.
Stage는 Scene들을 스크롤 흐름에 맞게 연결하는 역할만 가지면 된다.

---

# 9. 책임 분리 기준

## `components/scenes/capability/dom`

```txt
Capability Scene 내부 animation target DOM 계약
```

Scene 안에서 어떤 DOM을 animation target으로 삼을지 관리한다.

예:

```txt
title
subtitle
phase
character
quote
proof point
card
item
line
```

---

## `components/stages`

```txt
Capability Scene들을 스크롤 흐름에 연결
```

Stage는 각 Scene의 큰 구간을 찾고, scroll progress에 따라 controller를 실행한다.

예:

```txt
intro scene 시작
intro pinned 영역
proof section
navigator pin 영역
closing section
```

---

## `animations`

```txt
전달받은 DOM을 실제로 움직이는 timeline
```

Animation은 Stage의 흐름 자체를 알 필요가 없고, Scene의 전체 구조를 다 알 필요도 없다.
전달받은 DOM 또는 selector를 기준으로 실제 움직임만 정의한다.

예:

```txt
title reveal
phase crossfade
character scale in
quote fade up
list active transition
monitor content swap
```

---

# 10. 최종 한 줄 정리

```txt
Scene은 DOM을 알고,
Stage는 흐름을 알고,
Animation은 움직임을 안다.
```

이 기준으로 나누면 구조가 덜 꼬인다.

- Stage가 Scene 내부 DOM까지 과하게 알지 않아도 된다.
- Scene은 자신의 DOM 구조를 책임진다.
- Animation은 실제 움직임 방식에만 집중할 수 있다.
- selector 관리 기준도 자연스럽게 나뉜다.
