# GSAP Ticker와 Lenis의 관계

## “Ticker 기반”이란 무엇인가?

### 1. 개요

Lenis를 사용하다 보면

“GSAP Ticker 기반으로 Lenis를 구동한다”는 표현을 자주 보게 된다.

이 말은 단순히 GSAP와 Lenis를 같이 사용한다는 의미가 아니다.

핵심은 **애니메이션 실행 주기(Heartbeat)를 하나로 통합한다**는 것이다.

즉:

- Lenis는 스크롤 계산을 담당하고
- GSAP는 애니메이션 타이밍을 담당하는데

이 둘이 서로 다른 루프에서 동작하면 미세한 타이밍 오차가 발생할 수 있다.

그래서 Lenis의 `raf()`를 GSAP의 `ticker`에 등록하여

**모든 움직임을 GSAP의 프레임 루프 안에서 실행**시키는 방식이 사용된다.

---

# 2. Ticker 기반 vs Tween 기반

## Ticker 기반 (프레임 단위 업데이트)

### 정의

브라우저의 `requestAnimationFrame`과 동기화된

GSAP의 내부 루프(`gsap.ticker`)에 함수를 등록하여

매 프레임마다 실행하는 방식.

### 특징

- 종료 시점이 없음
- 매 프레임 지속적으로 실행됨
- 실시간 계산에 적합
- 스크롤/물리 기반 인터랙션에 많이 사용

### Lenis에서 사용하는 이유

Lenis는 현재 스크롤 위치를 지속적으로 계산해야 한다.

따라서:

- 사용자의 입력
- inertia(관성)
- lerp
- velocity

같은 값들을 매 프레임 갱신해야 한다.

이 과정을 GSAP ticker 안에서 처리하면:

- GSAP 애니메이션
- ScrollTrigger
- Lenis 스크롤 계산

이 모두가 완전히 동일한 타이밍에서 실행된다.

결과적으로:

- 스크롤 밀림 감소
- 프레임 어긋남 방지
- 부드러운 인터랙션 유지

가 가능해진다.

---

## Tween 기반 (목표 중심 애니메이션)

### 정의

특정 값이 일정 시간 동안 변화하도록 만드는 방식.

예시:

```tsx
gsap.to(".box", {
  x: 100,
  duration: 1,
});
```

### 특징

- 시작과 끝이 존재
- duration 기반
- 특정 상태 변화에 적합
- 일반적인 UI 애니메이션에 사용

### 대표 사용 사례

- opacity 변경
- 카드 등장 애니메이션
- scale 효과
- hover transition
- modal open/close

---

# 3. 왜 Lenis는 Ticker 기반이어야 할까?

Lenis는 단순 애니메이션이 아니라

“실시간 스크롤 엔진”에 가깝다.

따라서 Tween처럼:

```tsx
duration: 1;
```

같은 개념으로 동작하지 않는다.

대신:

```
현재 위치 → 목표 위치
```

를 지속적으로 보간(interpolation)하면서 움직인다.

즉:

```
매 프레임 계산
```

이 핵심이다.

---

# 4. ScrollTrigger와 함께 사용할 때 중요한 이유

Lenis와 ScrollTrigger를 함께 사용할 경우

Ticker 기반 연동은 사실상 필수에 가깝다.

이유는:

| 문제                         | 원인                    |
| ---------------------------- | ----------------------- |
| 스크롤과 애니메이션이 어긋남 | 서로 다른 RAF 루프 사용 |
| 특정 구간에서 끊김           | 프레임 타이밍 불일치    |
| ScrollTrigger 계산 오차      | 스크롤 위치 갱신 지연   |

Ticker 기반으로 통합하면:

- 스크롤 계산
- ScrollTrigger 업데이트
- GSAP 타임라인

이 모두 같은 프레임에서 실행된다.

---

# 5. 실제 구현 코드

```tsx
import Lenis from "lenis";
import gsap from "gsap";
import ScrollTrigger from "gsap/ScrollTrigger";

gsap.registerPlugin(ScrollTrigger);

const lenis = new Lenis();

const update = (time: number) => {
  lenis.raf(time * 1000);
};

gsap.ticker.add(update);

lenis.on("scroll", ScrollTrigger.update);

gsap.ticker.lagSmoothing(0);
```

---

# 6. 코드 해설

## lenis.raf(time \* 1000)

GSAP ticker의 `time`은 초(second) 단위다.

하지만 Lenis는 밀리초(ms)를 사용한다.

따라서:

```
time*1000
```

으로 변환해야 한다.

---

## gsap.ticker.add(update)

GSAP의 메인 루프에 Lenis를 등록한다.

이 순간부터:

```
GSAP heartbeat == Lenis heartbeat
```

상태가 된다.

---

## lenis.on("scroll", ScrollTrigger.update)

Lenis가 스크롤될 때마다

ScrollTrigger에게 스크롤 위치가 변경되었음을 알려준다.

이 코드가 없으면:

- trigger 계산
- pin 위치
- scrub animation

등이 정상 동작하지 않을 수 있다.

---

## gsap.ticker.lagSmoothing(0)

GSAP는 기본적으로 프레임 드랍이 발생하면

갑작스럽게 시간을 보정한다.

하지만 스크롤 기반 인터랙션에서는

이 보정이 오히려 부자연스러운 점프를 만들 수 있다.

그래서:

```
gsap.ticker.lagSmoothing(0);
```

으로 보정을 비활성화한다.

특히:

- ScrollTrigger
- Lenis
- scrub animation

조합에서 중요하다.

---

# 7. 핵심 구조 정리

```
브라우저 RAF
   ↓
GSAP Ticker
   ↓
Lenis raf()
   ↓
스크롤 위치 계산
   ↓
ScrollTrigger.update()
   ↓
GSAP Timeline 반응
```

즉:

```
하나의 프레임 루프 안에서
모든 인터랙션이 동시에 움직인다.
```

---

# 8. 한 줄 요약

> “Ticker 기반”이란
>
> Lenis의 스크롤 계산 엔진을 GSAP의 heartbeat에 연결하여
>
> 스크롤과 애니메이션이 하나의 프레임 루프 안에서 동기화되도록 만드는 방식이다.
