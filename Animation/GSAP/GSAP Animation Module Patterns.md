# GSAP Animation Module Patterns_React

### 개요

React + GSAP 환경에서 애니메이션 모듈은 크게 두 가지 유형으로 나눌 수 있다.

**1. Tween / Timeline Factory**

**2. Motion Controller**

이 두 유형은 역할과 책임이 다르며, 프로젝트 구조 설계 시 구분해 두는 것이 중요하다.

## 1. Tween / Timeline Factory

### 정의

대상 요소를 받아 **GSAP tween 또는 timeline을 생성하여 반환하는 모듈.**

이 유형은 **애니메이션 표현 자체를 정의하는 역할을 한다.**

### 특징

- stateless (내부 상태 거의 없음)
- 특정 요소의 모션 정의
- ScrollTrigger와 쉽게 분리 가능
- 대부분의 reveal/ entrance animation 에 사용

### 역할

**어떻게 움직이는가**

### 예시

```ts
import gsap from "gsap";

const InterviewAnimation = {
  title(target: HTMLElement) {
    return gsap.fromTo(
      target,
      {
        autoAlpha: 0,
        y: 24,
      },
      {
        autoAlpha: 1,
        y: 0,
        duration: 0.8,
        ease: "power2.out",
        paused: true,
      }
    );
  },
};

export default InterviewAnimation;
```

### 사용 위치

React 컴포넌트에서 ScrollTrigger와 결합한다.

```ts
const tween = InterviewAnimation.title(title);

ScrollTrigger.create({
  trigger: title,
  start: "top 85%",
  animation: tween,
});
```

### 언제 사용해야 하는가

- reveal animation
- fade in / slide up
- stagger animation
- element entrance animation

## 2. Motion Controller

### 정의

GSAP tween 만 반환하는 것이 아니라 **내부 상태와 제어 API를 포함하는 애니메이션 컨트롤러 모듈**

이 유형은 **애니메이션을 정의하는 것이 아니라 움직임 시스템을 제공한다.**

### 특징

- 내부 state 보유
- animation loop / ticker 사용 가능
- 외부 입력(progress 등)에 반응
- refresh / update / destroy API 제공

### 역할

**움직임을 제어하는 시스템 제공**

### 예시

```ts
import gsap from "gsap";

const LifeMotionAnimation = {
  track(track: HTMLDivElement, viewport: HTMLDivElement) {
    const setX = gsap.quickSetter(track, "x", "px");

    let distance = 0;
    let currentX = 0;
    let targetX = 0;

    const refresh = () => {
      distance = Math.max(0, track.scrollWidth - viewport.clientWidth);
    };

    const setProgress = (progress: number) => {
      targetX = -distance * progress;
    };

    const tick = () => {
      currentX += (targetX - currentX) * 0.12;
      setX(currentX);
    };

    refresh();
    gsap.ticker.add(tick);

    return {
      get distance() {
        return distance;
      },

      refresh,

      setProgress,

      destroy() {
        gsap.ticker.remove(tick);
        setX(0);
      },
    };
  },
};

export default LifeMotionAnimation;
```

### 사용 방식

React 컴포넌트에서 controller를 생성한다.

```ts
const controller = LifeMotionAnimation.track(track, viewport);

controller.setProgress(progress);
controller.refresh();

return () => controller.destroy();
```

### 언제 사용해야 하는가

- parallax animation
- scroll progress 기반 animation
- drag interaction
- ticker 기반 smoothing animation
- custom animation systems

## 두 유형의 차이

| 구분      | Tween Factory        | Motion Controller      |
| --------- | -------------------- | ---------------------- |
| 역할      | 모션 정의            | 움직임 제어            |
| 상태      | 거의 없음            | 내부 state 존재        |
| 반환값    | tween / timeline     | controller object      |
| 사용 방식 | ScrollTrigger에 연결 | 외부에서 제어          |
| 예시      | fade-in, reveal      | parallax, scroll track |

## React + GSAP 아키텍처에서의 위치

React 프로젝트에서의 책임 분리는 다음과 같다.

### TSX

- lifecycle
- ScrollTrigger
- DOM 연결
- controller 연결

### animation module

- gsap motion 생성
- motion controller 구현

## 정리

GSAP Animation module 은 두 가지 형태로 설계할 수 있다

### Tween Factory

- 애니메이션 표현 정의

### Motion Controller

- 애니메이션 동작 시스템 제공

두 유형을 구분하면 **애니메이션 구조 설계가 훨씬 명확해진다.**

### 핵심 요약

**Tween Factory = 보여주는 모션**

**Motion Controller = 움직임을 운영하는 시스템**

### 권장 사용

```
Reveal Animation → Tween Factory
Scroll / Interaction Animation → Motion Controller
```
