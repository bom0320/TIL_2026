# GSAP Animation Architecture (React / Next.js)

## 목적

React 환경에서 GSAP 애니메이션을 구현할 때 다음 문제를 방지하기 위해 애니메이션 구조를 분리한다.

- 컴포넌트 내부에 애니메이션 코드가 과도하게 쌓이는 문제
- ScrollTrigger와 React lifecycle이 뒤섞이는 문제
- 애니메이션 로직 재사용이 어려운 문제
- 유지보수 시 수정 위치가 불명확한 문제

핵심 원칙

```
TSX → 언제 실행되는가 (scroll / lifecycle)
animation.ts → 어떻게 움직이는가 (motion / gsap)
```

---

# 1. 책임 분리 원칙

## 1.1 TSX (React Component)

React 컴포넌트는 **애니메이션 실행 시점과 스크롤 문맥을 관리한다.**

### 포함되는 것

- `useLayoutEffect`
- `gsap.context`
- `ScrollTrigger`
- `trigger`
- `start`
- `end`
- `scrub`
- `once`
- `sectionRef`
- DOM selection (`querySelector`, `gsap.utils.toArray`)

### 역할

```
언제 애니메이션이 실행되는가
```

### 예시

```tsx
useLayoutEffect(() => {
  constsection = sectionRef.current;
  if (!section) return;

  constctx = gsap.context(() => {
    constrows = gsap.utils.toArray<HTMLElement>(".interview-row", section);

    rows.forEach((row) => {
      consttween = interviewAnimation.row(row);

      ScrollTrigger.create({
        trigger: row,
        start: "top 85%",
        once: true,
        animation: tween,
      });
    });
  }, section);

  return () => ctx.revert();
}, []);
```

---

# 2. animation.ts (Animation Module)

animation 파일은 **GSAP 모션 생성 로직을 담당한다.**

여기서는 ScrollTrigger를 사용하지 않는다.

### 포함되는 것

- `gsap.from`
- `gsap.fromTo`
- `opacity`
- `x`
- `y`
- `scale`
- `duration`
- `ease`
- `stagger`

### 역할

```
어떻게 움직이는가
```

### 예시

```tsx
import gsap from"gsap";

exportconstinterviewAnimation= {
  title: (target:HTMLElement) =>
gsap.fromTo(
target,
      { autoAlpha:0, y:24 },
      {
        autoAlpha:1,
        y:0,
        duration:0.8,
        ease:"power2.out",
        paused:true,
      }
    ),

  row: (target:HTMLElement) =>
gsap.fromTo(
target,
      { autoAlpha:0, y:20 },
      {
        autoAlpha:1,
        y:0,
        duration:0.7,
        ease:"power2.out",
        paused:true,
      }
    ),
}asconst;
```

animation.ts는 **ScrollTrigger를 알지 못한다.**

---

# 3. 절대 섞지 않는 규칙

구조가 무너지지 않도록 다음 규칙을 유지한다.

### animation.ts에 넣지 말 것

```
ScrollTrigger
trigger
start
end
scrub
once
```

### TSX에 넣지 말 것

```
duration
ease
x
y
opacity
stagger
```

---

# 4. 이 구조의 장점

## 4.1 책임 분리가 명확하다

```
TSX → 실행 조건
animation.ts → 모션 정의
```

---

## 4.2 디자인 수정이 쉬움

예

```
애니메이션 속도를 조금 느리게 해주세요
```

수정 위치

```
animation.ts
```

---

## 4.3 스크롤 연출 수정이 쉬움

예

```
좀 더 늦게 등장하게 해주세요
```

수정 위치

```
TSX (start 값 수정)
```

---

## 4.4 애니메이션 재사용 가능

동일한 animation 함수를 여러 섹션에서 사용할 수 있다.

예

```
Hero Section
Project Section
Interview Section
About Section
```

---

# 5. 프로젝트 구조

추천 구조

```
src
 ├ components
 │   └ about
 │       └ AboutInterview.tsx
 │
 ├ animations
 │   ├ interviewAnimation.ts
 │   ├ heroAnimation.ts
 │   └ projectAnimation.ts
 │
 └ data
     └ interviews.ts
```

---

# 6. React + GSAP 필수 패턴

### 1. useLayoutEffect 사용

레이아웃 기반 애니메이션에서는 `useLayoutEffect`가 안정적이다.

```tsx
useLayoutEffect;
```

---

### 2. gsap.context 사용

React Strict Mode에서 안전한 애니메이션 관리를 위해 사용한다.

```
gsap.context(..., sectionRef)
```

장점

- scope 제한
- cleanup 자동 처리
- DOM 충돌 방지

---

### 3. cleanup 필수

컴포넌트 unmount 시 애니메이션을 제거해야 한다.

```
return () =>ctx.revert();
```

---

# 7. 설계 철학

이 구조는 다음 기준을 따른다.

```
스크롤 문맥 = React 책임
모션 생성 = Animation 책임
```

즉

```
WHEN → Component
HOW → Animation Module
```

---

# 8. 언제 이 구조를 사용해야 하는가

이 패턴은 다음 상황에서 특히 유용하다.

- 섹션별 ScrollTrigger 전략이 다른 경우
- 애니메이션 재사용이 필요한 경우
- 포트폴리오 프로젝트
- 유지보수가 중요한 프로젝트

---

# 요약

```
TSX → 언제 실행되는가
animation.ts → 어떻게 움직이는가
```

이 규칙을 유지하면 애니메이션 구조가 **확장 가능하고 유지보수 가능한 형태**로 유지된다.
