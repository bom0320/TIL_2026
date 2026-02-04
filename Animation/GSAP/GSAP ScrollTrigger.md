# GSAP ScrollTrigger 개념 정리

## ScrollTrigger 란?

GSAP ScrollTrigger는 **스크롤 위치를 기준으로 애니메이션을 제어할 수 있게 해주는 GSAP 플러그인이다.**

기존 GSAP 애니메이션이

- 페이지 로드
- 클릭 같은 이벤트

를 기준으로 동작했다면, ScrollTrigger는 **스크롤을 기준으로 애니메이션의 시작,진행, 종료를 결정** 한다.

> 스크롤을 하나의 **타입라인(Time Line)**처럼 다룰 수 있게 해준다.

## 왜 ScrollTrigger가 필요한가?

### 기존(Time-based) 애니메이션의 한계

```ts
gsap,
  from(".title", {
    opacity: 0,
    y: 40,
    duration: 1,
  });
```

- 페이지에 들어오자마자 실행됨
- 사용작 **보든 말든** 애니메이션은 끝남
- 스크롤과 상관 없음
- 즉, **스크롤 중심 UI에서는 어색함**

### ScrollTrigger가 해결하는 것

ScrollTrigger는 애니메이션을:

- 특정 섹션에 **진입했을 때**
- 스크롤 어지쯤에서 시작/끝낼지
- 스크롤에 **연동해서(scrub) 진행할지**

를 정밀하게 제어할 수 있다.

### ScrollTrigger가 가능한 것

ScrollTrigger를 사용하면 다음과 같은 UI 연출이 가능하다.

- 특정 섹션이 화면에 등장할 때 애니메이션 실행
- 스크롤하는 동안 애니메이션을 자연스럽게 진행
- 섹션을 스크롤 중에 고정(pin)하여 스토리텔링연출
- 스크롤 위치에 따라 애니메이션 상태를 정확하게 제어

요즘 포트폴리오, 렌딩 페이지에서 자주 보이는 **스크롤 기반 인터렉션 UI** 는 대부분 ScrollTrigger로 구현된다.

## ScrollTrigger의 핵심 개념 3가지

### 1. trigger

스크롤 기준이 되는 DOM 요소

```ts
scrollTrigger: {
    trigger: sectionRef.current,
}
```

- "이 요소가 화면에 들어올 때부터 계산 시작"
- 보통 섹션 전체(section)를 기준으로 잡는다.

### 2. start/ end

애니메이션이 시작/끝나는 스크롤 지점

```ts
start: "top 75%",
end: "top 40%",
```

의미:

- **`start` :** trigger의 top이 viewPort의 75% 지점에 닿을 때
- **`end` :** trigger의 top이 viewport의 40% 지점에 닿을 때

이 구간이 **애니메이션의 진행 구간**

### 3. scrub

스크롤과 애니메이션 진행을 연결할지 여부

```ts
scrub: true;
```

- 스크롤을 내리면 -> 애니메이션 진행
- 스크롤을 멈추면 -> 애니메이션도 멈춤
- 위로 스크롤하면 -> 애니메이션도 되감김

## Time-based vs. ScrollTrigger (개념 비교)

| 구분        | Time-based | ScrollTrigger |
| ----------- | ---------- | ------------- |
| 기준        | 시간       | 스크롤        |
| 시작        | 자동       | 스크롤 위치   |
| 사용자 개입 | 없음       | 있음          |
| 멈춤/되감기 | x          | o             |
| 체감        | 영상       | 인터랙션      |

## 기본 사용 예제

```scss
gsap.fromTo {
    ".box",
    { opacity: 0, y: 40 },
    {
        opacity: 1,
        y: 0,
        scrollTrigger: {
            trigger: ".section",
            start: "top 80%",
            end: "top 50%"
            scrub: true,
        },

    }
}
```

- `.section` 에 진입하면서
- `.box` 가 스크롤에 따라 서서히 나타남

## ScrollTrigger를 써야 하는 대표적인 이유

### 적합한 경우

- About / Skills / Projects 같은 **콘텐츠 섹션**
- 사용자가 **읽고 머무르는 영역**
- "스크롤 자체가 내비게이션"인 페이지
- 채워지는 텍스트, reveal 애니메이션

### 부적합한 경우

- Hero 인트로
- 로딩 애니메이션
- 반복(loop) 애니메이션
- hover 애니메이션

## ScrollTrigger에서 가장 중요한 설계 포인트

### 애니메이션 기준은 "누가 결정하냐?"

- Time-based -> 개발자가 시간으로 결정
- ScrollTrigger -> 사용자가 스크롤로 결정
  그래서 ScrollTrigger는:
  > **연출이 아니라 인터랙션 설계 도구**에 가깝다.

## 실전에서 자주 쓰는 패턴

### 1. 섹션 진입 시 한 번만 실행

```ts
scrollTrigger: {
    trigger: section,
    start: "top 80%",
    once: true,
}
```

### 2. 스크롤에 따라 채워지는 효과

```ts
scrollTrigger: {
    trigger: section,
    start: "top 75%",
    end: "top 40%",
    scrub: true,
}
```

### 3. 여러 애니메이션을 같은 기준으로

- trigger 같은 sectionRef로 통일
- 섹션 단위 일정한 UX 가능

## React에서 ScrollTrigger를 쓸 때 주의점

- (X) `document.querySelector` 남발 금지
- (O) `ref` 기반으로 trigger 지정
- (O) `gsap.context + ctx.revert()` 사용 (클린업)

```ts
useLayoutEffect(() => {
  const ctx = gsap.context(() => {
    // ScrollTrigger setup
  }, root);

  return () => ctx.revert();
}, []);
```

## 한 문장 요약

ScrollTrigger는 스크롤 위치를 시간처럼 사용하는 애니메이션 컨트롤러이다.

즉, '보여주는 애니메이션'이 아니라 '사용자가 조작하는 애니메이션'이다.
