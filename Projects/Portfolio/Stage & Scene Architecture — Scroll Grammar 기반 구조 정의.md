# Stage / Scene Architecture — Scroll Grammar 기반 구조 정의

## 1. 핵심 정의

내 기준에서 `Stage`는 단순한 섹션(section)이 아니다.

Stage는:

> 하나의 Scroll Interaction Language(스크롤 문법)가 유지되는 영역

이다.

즉:

- Section → 무엇을 보여주는가
- Stage → 어떤 스크롤 연출 규칙으로 경험시키는가

의 차이다.

---

# 2. Section vs Stage

| 구분    | 의미                      |
| ------- | ------------------------- |
| Section | 콘텐츠 단위               |
| Stage   | 스크롤 인터랙션 문법 단위 |

예를 들면:

- Hero
- About
- Projects

이건 단순 콘텐츠 구간일 수 있다.

하지만:

- cinematic transition
- pinned showcase
- layered reveal
- activeIndex orchestration

같은 스크롤 규칙이 유지되는 범위가 곧 Stage다.

---

# 3. Stage를 분리하는 기준

## 3-1. 스크롤 규칙이 바뀌는가?

가장 중요한 기준.

예시:

## IntroStage

유지되는 문법:

- sticky
- hero exit
- lifeMotion enter
- layered transition
- cinematic timing

즉:

스크롤 진행률을

“시네마틱 전환”으로 해석한다.

---

## ProjectsStage

유지되는 문법:

- pin
- monitor showcase
- activeIndex sync
- scroll-driven switching

즉:

스크롤 진행률을

“프로젝트 탐색 인터랙션”으로 해석한다.

---

따라서:

스크롤 해석 방식이 달라지는 순간

Stage가 분리된다.

---

# 3-2. ScrollTrigger Ownership이 바뀌는가?

이것도 매우 중요하다.

## IntroStage의 orchestration

관리 대상:

- hero ↔ life transition
- layered timing
- intro cinematic flow

즉:

Intro 전체의 trigger orchestration ownership을 가진다.

---

## ProjectsStage의 orchestration

관리 대상:

- project pinning
- active project switching
- monitor synchronization
- showcase progression

즉:

완전히 다른 orchestration boundary를 가진다.

---

정리하면:

> Trigger orchestration boundary가 바뀌는 순간
>
> Stage가 분리된다.

---

# 3-3. 사용자 인지 흐름이 바뀌는가?

Stage는 단순 기술 구조가 아니라

Narrative 구조이기도 하다.

예시:

| Stage    | 사용자 인지 흐름             |
| -------- | ---------------------------- |
| Intro    | “이 사람 누구지?”            |
| About    | “어떤 철학과 성향을 가졌지?” |
| Projects | “실제로 뭘 만들었지?”        |
| Contact  | “연락하고 싶다”              |

즉:

사용자의 인지 목적이 바뀌는 순간

Narrative Chapter도 바뀐다.

그리고 이것 역시 Stage 분리 기준이다.

---

# 4. Stage의 본질

결국 Stage는:

> Narrative + Motion + Interaction

의 묶음이다.

단순 layout 단위가 아니다.

---

# 5. 프로젝트 기준 Stage 구조

## IntroStage

### 역할

정체성 소개

### 문법

- cinematic transition
- layered orchestration
- Hero ↔ LifeMotion transition

---

## AboutStage

### 역할

철학 / 성향 / 기술 소개

### 문법

- reveal
- interview storytelling
- progressive reading flow

---

## ProjectsStage

### 역할

작업물 showcase

### 문법

- pinned interaction
- activeIndex orchestration
- monitor synchronization
- scroll-driven switching

---

## ContactStage

### 역할

마무리 CTA

### 문법

- footer reveal
- ending motion
- closing transition

---

# 6. 반대로 Stage가 아닌 것

예시:

- HeroContent
- SkillCard
- ProjectMonitor

이건 Stage가 아니다.

왜냐하면:

- 스크롤 문법이 바뀌지 않음
- orchestration ownership이 없음
- narrative chapter가 아님

즉:

단순 UI 조각(Component/UI Unit)이다.

---

# 7. Scene vs Stage

## Scene

Stage 내부에서 전환되는 장면.

예:

IntroStage 내부:

- HeroScene
- LifeMotionScene

---

## Stage

하나의 Scroll Interaction Language가 유지되는 영역.

즉:

- Scene = 장면
- Stage = 스크롤 문법 시스템

이다.

---

# 8. 중요한 오해 — “스크롤 느낌”과 “스크롤 문법”은 다르다

여기서 중요한 구분이 있다.

## 전역 스크롤 물성(Global Scroll Physics)

이건 Lenis Provider가 담당한다.

예:

```tsx
constlenis = newLenis({
  duration: 1.2,
  easing: (t) => Math.min(1, 1.001 - Math.pow(2, -10 * t)),
  smoothWheel: true,
});
```

이건 사이트 전체의:

- 관성
- 감도
- 보간
- 가속/감속 질감

을 결정한다.

즉:

> 전체 사이트의 스크롤 물성은 통일되어야 한다.

---

# 9. Stage는 스크롤 값을 “어떻게 해석할지”를 결정한다

같은 스크롤 질감을 사용하더라도,

Stage마다 해석 방식은 다르다.

---

## IntroStage

스크롤 진행률:

```
0 → 1
```

해석:

- Hero exit
- LifeMotion enter
- cinematic transition

---

## ProjectsStage

스크롤 진행률:

```
0 → 1
```

해석:

- activeIndex switching
- monitor image transition
- pinned showcase flow

---

## ContactStage

스크롤 진행률:

```
0 → 1
```

해석:

- footer reveal
- CTA emergence

---

즉:

> 스크롤 값은 같지만,
>
> Stage마다 그 값을 해석하는 방식이 다르다.

---

# 10. 전체 구조 관점

```
SmoothScrollProvider
  └─ HomePage
      ├─ IntroStage
      │    └─ Hero → LifeMotion transition
      │
      ├─ AboutStage
      │    └─ reveal / interview storytelling
      │
      ├─ ProjectsStage
      │    └─ pinned showcase interaction
      │
      └─ ContactStage
           └─ ending reveal
```

---

# 11. 최종 정의

## Global Layer

### SmoothScrollProvider

전체 사이트의:

- scroll physics
- inertia
- interpolation
- acceleration feeling

을 담당한다.

즉:

전역 스크롤 물성.

---

## Stage Layer

각 Stage는:

> 동일한 스크롤 질감을 기반으로,
>
> 스크롤 값을 어떤 연출과 장면으로 해석할지를 담당한다.

즉:

- orchestration
- narrative
- interaction language

를 가진다.

---

# 12. 최종 결론

정리하면:

- LenisProvider → 스크롤 “질감”
- Stage → 스크롤 “문법”
- Scene → Stage 내부의 “장면”

이다.

따라서 Stage를 나누는 기준은:

> “스크롤 느낌이 다르다”가 아니라

> 같은 스크롤 물성을 공유하면서,
>
> 무엇을 움직이고,
>
> 어떤 인터랙션 규칙으로 해석하며,
>
> 어떤 narrative 흐름으로 경험시키는가

가 달라지는 지점이다.
