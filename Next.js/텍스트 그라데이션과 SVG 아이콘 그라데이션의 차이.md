텍스트 그라데이션과 SVG 아이콘 그라데이션의 차이
===

## 오늘 배운 것

오늘은 포트폴리오 `VisualCapabilityCard`를 구현하면서, 텍스트에 적용하는 그라데이션과 SVG 아이콘에 적용하는 그라데이션이 서로 다른 방식으로 동작한다는 것을 알게 되었다.

처음에는 기존에 만들어둔 `GradientText` 컴포넌트를 아이콘에도 사용할 수 있을 것이라고 생각했다. 하지만 `GradientText`는 텍스트를 위한 방식이고, lucide 아이콘은 SVG 기반의 도형이기 때문에 같은 방식으로 처리할 수 없었다.

---

## 문제 상황

포트폴리오의 Visual 섹션에서 일부 카드 아이콘에 민트 그라데이션을 적용하고 싶었다.

예를 들어 Typography, Storytelling 카드의 아이콘에 Apple 스타일의 은은한 민트 그라데이션을 주고 싶었다.

처음에는 기존에 만들어둔 `GradientText` 컴포넌트를 재사용하면 될 것 같았다.

```tsx
export default function GradientText({
  children,
  className = "",
}: GradientTextProps) {
  return <span className={`gradient-text ${className}`}>{children}</span>;
}
```

이 컴포넌트는 텍스트에는 잘 동작한다.

```tsx
<GradientText>감각까지.</GradientText>
```

하지만 lucide 아이콘에 적용하려고 하면 원하는 대로 동작하지 않았다.

```tsx
<GradientText>
  <Icon />
</GradientText>
```

이 방식으로는 SVG 아이콘의 stroke에 그라데이션이 적용되지 않았다.

---

## 왜 GradientText는 텍스트에는 되고, 아이콘에는 안 될까?

`GradientText`는 보통 CSS에서 다음과 같은 방식으로 동작한다.

```scss
.gradient-text {
  background: linear-gradient(...);
  background-clip: text;
  -webkit-background-clip: text;
  color: transparent;
  -webkit-text-fill-color: transparent;
}
```

이 방식은 텍스트의 글자 모양을 기준으로 배경 그라데이션을 잘라내는 방식이다.

즉 동작 흐름은 다음과 같다.

```
배경에 그라데이션을 깐다
↓
텍스트 모양으로 배경을 자른다
↓
글자 안에만 그라데이션이 보인다
```

그래서 텍스트에는 잘 맞는다.

하지만 lucide 아이콘은 텍스트가 아니다. lucide 아이콘은 내부적으로 SVG 구조를 가진다.

```html
<svg>
  <path />
  <line />
  <circle />
</svg>
```

즉 아이콘은 글자가 아니라 `path`, `line`, `circle` 같은 SVG 도형이다.

`background-clip: text`는 텍스트 glyph에 적용되는 방식이지, SVG 내부의 stroke나 fill에 적용되는 방식이 아니다.

그래서 `GradientText`로 SVG 아이콘을 감싸도 아이콘 stroke에는 그라데이션이 먹지 않는다.

---

## 텍스트와 SVG 아이콘의 차이

| 대상            | 구조                     | 그라데이션 방식            |
| --------------- | ------------------------ | -------------------------- |
| 텍스트          | 글자 glyph               | `background-clip: text`    |
| lucide 아이콘   | SVG path / line / circle | `stroke: url(#gradientId)` |
| 면형 SVG 아이콘 | SVG shape                | `fill: url(#gradientId)`   |

핵심은 텍스트와 SVG는 렌더링 방식이 다르다는 것이다.

텍스트는 글자 모양으로 배경을 잘라내는 방식이 자연스럽고, SVG는 도형의 `stroke`나 `fill` 자체에 그라데이션 paint를 적용해야 한다.

---

## 해결 방식

lucide 아이콘은 대부분 선형 아이콘이다. 그래서 `fill`이 아니라 `stroke`를 제어해야 한다.

먼저 카드마다 고유한 SVG gradient id를 만든다.

```tsx
const rawId = useId();
const gradientId = `visual-icon-gradient-${rawId.replace(/:/g, "")}`;
```

`useId()`를 사용하는 이유는 카드가 여러 개 있을 때 SVG gradient id가 겹치지 않도록 하기 위해서다.

React의 `useId()`는 `:r1:` 같은 형태의 값을 만들 수 있으므로, SVG url 참조에서 문제가 생기지 않도록 `:`를 제거했다.

```tsx
rawId.replace(/:/g, "");
```

그다음 accent가 필요한 카드에만 CSS 변수를 심는다.

```tsx
const style = item.accent
  ? ({
      "--visual-icon-gradient": `url(#${gradientId})`,
    } as CSSProperties)
  : undefined;
```

이렇게 하면 실제 HTML에는 다음과 비슷한 값이 들어간다.

```html
<article
  style="--visual-icon-gradient: url(#visual-icon-gradient-r1)"
></article>
```

그리고 카드 내부에 보이지 않는 SVG gradient 정의를 추가한다.

```tsx
{
  item.accent ? (
    <svg
      className="experience-capability-visual-card__gradient-def"
      width="0"
      height="0"
      aria-hidden="true"
      focusable="false"
    >
      <defs>
        <linearGradient id={gradientId} x1="0%" y1="0%" x2="100%" y2="100%">
          <stop offset="0%" stopColor="#a8fff1" />
          <stop offset="45%" stopColor="#62f3dd" />
          <stop offset="100%" stopColor="#1bb7a6" />
        </linearGradient>
      </defs>
    </svg>
  ) : null;
}
```

여기서 `linearGradient`는 React 컴포넌트가 아니라 SVG 표준 태그다.

`defs`는 화면에 바로 그리지 않고, 나중에 참조할 리소스를 정의하는 영역이다.

즉 이 SVG는 화면에 보이기 위한 요소가 아니라, 아이콘 stroke에서 참조할 그라데이션을 정의하는 용도다.

---

## SCSS에서 SVG stroke에 그라데이션 적용하기

lucide 아이콘 내부의 `path`, `line`, `circle` 등에 직접 stroke를 적용해야 한다.

```scss
.experience-capability-visual-card--accent
  .experience-capability-visual-card__icon
  svg
  * {
  stroke: var(--visual-icon-gradient);
}
```

이렇게 하면 TSX에서 만든 CSS 변수 값이 SVG 내부 요소의 stroke에 적용된다.

전체 흐름은 다음과 같다.

```
TSX에서 gradientId 생성
↓
linearGradient id로 SVG 그라데이션 정의
↓
CSS 변수 --visual-icon-gradient에 url(#gradientId) 저장
↓
SCSS에서 svg 내부 요소의 stroke에 변수 적용
↓
아이콘 선이 민트 그라데이션으로 보임
```

---

## 최종 코드 구조

```tsx
import Image from "next/image";
import { useId } from "react";
import type { CSSProperties } from "react";

import type { VisualCapabilityItem } from "@/data/capability/experience";

import { VISUAL_ICON_MAP } from "./visualCapabilityIconMap";

type VisualCapabilityCardProps = {
  item: VisualCapabilityItem;
};

export default function VisualCapabilityCard({
  item,
}: VisualCapabilityCardProps) {
  const rawId = useId();
  const gradientId = `visual-icon-gradient-${rawId.replace(/:/g, "")}`;

  const Icon = VISUAL_ICON_MAP[item.icon];
  const hasImage = Boolean(item.image);

  const style = item.accent
    ? ({
        "--visual-icon-gradient": `url(#${gradientId})`,
      } as CSSProperties)
    : undefined;

  return (
    <article
      style={style}
      className={`experience-capability-visual-card experience-capability-visual-card--${
        item.id
      } experience-capability-visual-card--${item.variant} ${
        item.accent ? "experience-capability-visual-card--accent" : ""
      }`}
    >
      {item.accent ? (
        <svg
          className="experience-capability-visual-card__gradient-def"
          width="0"
          height="0"
          aria-hidden="true"
          focusable="false"
        >
          <defs>
            <linearGradient id={gradientId} x1="0%" y1="0%" x2="100%" y2="100%">
              <stop offset="0%" stopColor="#a8fff1" />
              <stop offset="45%" stopColor="#62f3dd" />
              <stop offset="100%" stopColor="#1bb7a6" />
            </linearGradient>
          </defs>
        </svg>
      ) : null}

      {hasImage && item.image ? (
        <div className="experience-capability-visual-card__media">
          <Image
            src={item.image.src}
            alt={item.image.alt}
            fill
            className="experience-capability-visual-card__image"
          />
        </div>
      ) : null}

      <div className="experience-capability-visual-card__overlay" />

      <div className="experience-capability-visual-card__content">
        <div className="experience-capability-visual-card__icon">
          <Icon aria-hidden="true" />
        </div>

        <h3 className="experience-capability-visual-card__title">
          {item.title}
        </h3>

        {item.description ? (
          <p className="experience-capability-visual-card__desc">
            {item.description}
          </p>
        ) : null}
      </div>
    </article>
  );
}
```

SCSS는 다음처럼 처리한다.

```scss
.experience-capability-visual-card__gradient-def {
  position: absolute;

  width: 0;
  height: 0;

  overflow: hidden;
}

.experience-capability-visual-card__icon svg {
  width: 58px;
  height: 58px;

  stroke-width: 2.45;
  overflow: visible;
}

.experience-capability-visual-card--accent
  .experience-capability-visual-card__icon
  svg
  * {
  stroke: var(--visual-icon-gradient);
}
```

---

## 이번에 얻은 기준

앞으로 그라데이션을 적용할 때는 대상의 렌더링 구조를 먼저 봐야 한다.

```
글자에 그라데이션을 넣고 싶다
→ GradientText 사용

SVG 선 아이콘에 그라데이션을 넣고 싶다
→ linearGradient + stroke url 사용

SVG 면 아이콘에 그라데이션을 넣고 싶다
→ linearGradient + fill url 사용
```

즉 `GradientText`를 무리하게 확장해서 모든 그라데이션에 쓰려고 하면 안 된다.

컴포넌트의 역할은 명확하게 나누는 것이 좋다.

```
GradientText
= 텍스트 전용 그라데이션 컴포넌트

VisualCapabilityCard의 SVG gradient
= SVG 아이콘 전용 그라데이션 처리
```

---

## 정리

오늘 가장 크게 배운 것은 “그라데이션을 어디에 적용하느냐에 따라 접근 방식이 완전히 달라진다”는 점이다.

텍스트는 `background-clip: text`로 처리할 수 있지만, SVG 아이콘은 텍스트가 아니므로 내부 도형의 `stroke`나 `fill`에 직접 그라데이션을 적용해야 한다.

이번 구현을 통해 텍스트 스타일링과 SVG 스타일링의 차이를 더 명확히 이해하게 되었고, 앞으로 아이콘이나 벡터 그래픽에 효과를 줄 때는 SVG의 구조를 먼저 확인해야겠다고 느꼈다.
