# Next.js에서 모바일 Header가 첫 로드 시 깨지는 문제 (Hydration / Responsive Rendering 이슈)

## 문제 상황

모바일 환경에서 페이지를 새로고침하거나 처음 진입할 때:

- 처음엔 데스크탑 헤더를 잠깐 보여줬다가 0.1초 정도 뒤에 "아 모바일이네!"하ㅏ고 모바일 헤더로 바꿔치기함으로써, 끊기는 현상 발생
- 즉, Header가 잠깐 데스크탑 메뉴 형태로 렌더되고, 이후 햄버거 버튼 UI로 교체되면서 레이아웃이 튀는 현상이 발생한다. 결과적으로 "UI가 깨진 것처럼" 보이는 순간이 존재한다.

특히 헤더 영역에서 구조가 바뀌면서 시각적으로 플래시처럼 보인다.

## 원인

`isMobile` 값을 JS로 판단해서 렌더 자체를 조건 분기하고 있었기 때문

- 초기 렌더에서는 `isMobile` 의 기본 값이 `false`
  즉, Header에서 이런 식으로 하고 있음,

  - `isMobile` 이 true 이면 -> 햄버거 버튼 보여줌
  - `isMobile`이 false 이면 -> 데크스탑 메뉴(HERO/ABOUT/…) 보여줌

  문제는 `isMobile` 이 처음엔 무조건 false로 시작한다는 것이다.

- 그래서 첫 화면은 데스크탑 Header(메뉴 nav)가 먼저 렌더
- 이후 `useEffect`가 실행되면서 `window.innerWidth` 를 확인하고 `isMobile=true` 로 변경
- 그 순간 DOM 구조가 데스크탑 메뉴 -> 햄버거 버튼으로 교체
- 결과적으로 모바일에서 "첫 로드 때 UI가한번 튀는 **플래시/깨짐**"" 발생

핵심은:

초기 렌더 시점엔 브라우저 width 를 정확히 사용할 수 없어(SSR/하이드레이션 타이밍), 조건 분기 렌더가 순간적으로 잘못된 UI를 보여줄수 있다.

## 당시 Header 구조

모바일 여부를 다음과 같이 JS state로 관리하고 있었다.

```ts
const [isMobile, setIsMobile] = useState(false);

useEffect(() => {
  const mobile = window.innerWidth < 768;
  setIsMobile(mobile);
}, []);
```

그리고 렌더를 이렇게 분기 했다:

```ts
{
  isMobile ? (
    <button className="menu-toggle">...</button>
  ) : (
    <nav className="menu">...</nav>
  );
}
```

즉,

- 모바일이면 햄버거 버튼 렌더
- 아니면 데스크탑 메뉴 렌더

### + 비교 분석 - 왜 Hero는 문제가 없었을가?

Header는 모바일 여부를 JS 상태(`isMobile`)로 판단하여 렌더 구조 자체를 분기했다.

반면 HeroSection은 다음과 같은 차이가 있었다.

- 모바일/ 데스크탑 여부에 따른 렌더 구조 분기 없음
- HTML 구조는 항상 동일
- 반응형 처리는 SCSS의 `@media`로만 제어

```scss
@media (max-width: 768px) {
  .hero {
    flex-direction: column;
  }
}
```

즉, Hero는

> "모바일인지 판단해서 DOM을 바꾸는 방식" 이 아니라,
> "동일한 DOM에 대해 css가 시각적 표현만 바꾸는 방식"

이었기 때문에 첫 렌더 시점에 구조 변화가 발생하지 않았다.

그 결과

- hydration mismatch 없음
- UI 플래시 없음
- 레이아웃 점프 없음

| 구분             | Header                    | Hero            |
| ---------------- | ------------------------- | --------------- |
| 반응형 처리 방식 | JS state 기반 조건 렌더   | CSS media query |
| 초기 렌더 구조   | 모바일 여부에 따라 달라짐 | 항상 동일       |
| 첫 로드 안정성   | UI 점프 발생              | 안정적          |
| 추천 방식        | 조건 렌더                 | CSS 기반 제어   |

**🔨핵심 인사이트**

- "뷰포트 기반 반응형"은 CSS가 가장 안정적
- 렌더 구조를 JS로 바꾸는 방식은 hydration 시점에 불일치를 만들 수 있음
- 가능하면 "구조는 고정, 표현만 변경"이 안정적인 설계

---

## 본질적 원인

이 문제의 핵심은:

> "반응형 UI를 JS 상태로 렌더 구조 자채를 분기했기 때문"

특히 Next.js 환경에서는:

- 초기 렌더는 서버 기준 (SSR)
- 브라우저에서 hydration 과정이 존재
- `window` 는 초기 렌더 시점에 안정적으로 사용할 수 없음

따라서 "브라우저 width에 따라 렌더 구조를 바꾸는 방식"은 초기 렌더와 실제 환경 사이에 불일치를 만들 가능성이 높다.

## 해결 방법

**기존 방식**

- JS로 모바일 여부 판단
- 렌더 구조를 조건 분기

**변경 방식**

- 데스크탑 메뉴와 모바일 버튼을 모두 렌더
- CSS media query로 보이기/ 숨기기 제어

```ts
<nav className="menu menu--desktop">...</nav>
<button className="menu-toggle menu--mobile">...</button>
```

```scss
.menu--mobile {
  display: none;
}
.menu--desktop {
  display: flex;
}

@media (max-width: 768px) {
  .menu--desktop {
    display: none;
  }
  .menu--mobile {
    display: inline-flex;
  }
}
```

이렇게 하면:

- 렌더 구조는 항상 동일
- css가 환경에 맞게 시각적 표시만 제어
- hydration mismatch 및 UI 플래시 방지

## 배운점

1. 반응형 UI는 가능하면 css로 해결하는 것이 가장 안정적이다.
2. 브라우저 width를 기반으로 렌더 구조를 분기하면 초기 렌더 불일치가 발생할 수 있다.
3. Next.js 환경에서는 "초기 렌더 타이밍"을 항상 고려해야 한다.
4. 구조를 단순화하면 버그도 줄어든다.
