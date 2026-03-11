# GSAP + React에서 ref vs. selector 사용 기준

React 환경에서 GSAP 애니메이션을 구현할 때 **DOM 접근 방식과 구조를 일관되게 유지하기 위한 기준** 을 정리한다.

## 1. 기본 원칙

GSAP 애니메이션 구현 시 다음 기준을 따른다.

```
단일 요소 → ref
반복 요소 → selector
부모 scope → ref
자식 탐색 → querySelector
```

이 구조를 사용하면 코드 복잡도를 낮추고 유지보수를 쉽게 할 수 있다.

## 2. 단일 요소는 ref 사용

한번만 존재하는 요소는 `ref` 로 직접 참조한다.

대표적인 예

- section container
- title
- description
- svg group
- canvas

ex)

```ts
const sectionRef = useRef<HTMLElement | null>(null);
const fillGroupRef = useRef<SVGElement | null>(null);
const descRef = useRef<HTMLParagraphElement | null>(null);
```

이 방식의 장점

- DOM 탐색 비용 없음
- 클래스 변경에 영향 없음
- 코드 의도 명확

## 3. 반복 요소는 selector 사용

`map()` 으로 생성되는 리스트는 selector로 찾는 것이 일반적이다.

- **selector 란?** : selector는 클래스, id, 태그 이름 등을 이용해서 DOM을 찾는 방식이다.

ex)

```ts
const rows = gsap.utils.toArray;
".interview-row", section;
```

ref 배열을 사용하는 방식

```ts
rowsRef.current[i] = el;
```

은 코드가 길어지고 관리가 복잡해지기 때문에 일반적으로는 사용하지 않는다.

# 4. 부모는 ref, 내부 요소는 querySelector

GSAP에서는 부모 요소를 기준으로 자식 요소를 찾는 패턴이 많이 사용된다.

예

```tsx
rows.forEach((row) => {
  constq = row.querySelector(".interview-row__q");
  constcontent = row.querySelector(".interview-row__content");
});
```

장점

- DOM 탐색 범위가 제한됨
- 코드 간결
- ref 관리 불필요

---

# 5. GSAP scope는 ref 기반으로 설정

`gsap.context()` 사용 시 scope를 ref로 설정한다.

```tsx
constctx = gsap.context(() => {
  // animations
}, sectionRef);
```

장점

- selector 범위 제한
- React unmount 시 자동 cleanup
- 메모리 누수 방지

---

# 6. 직접 제어 대상은 ref 사용

GSAP에서 직접 애니메이션 target으로 사용하는 요소는 ref로 관리한다.

예

SVG animation

```tsx
gsap.to(fillGroupRef.current, { opacity: 1 });
```

이 경우 selector보다 ref가 안정적이다.

---

# 7. 실제 프로젝트 적용 예

## ref 사용

```
sectionRef
fillGroupRef
titleRef
descRef
```

## selector 사용

```
.interview-row
.interview-row__q
.interview-row__content
```

---

# 8. 권장 구조 예시

```tsx
constsectionRef = useRef(null);

useLayoutEffect(() => {
  constctx = gsap.context(() => {
    constcards = gsap.utils.toArray(".card");

    cards.forEach((card) => {
      consttitle = card.querySelector(".card-title");

      gsap.from(card, {
        opacity: 0,
        y: 30,
      });
    });
  }, sectionRef);

  return () => ctx.revert();
}, []);
```

---

# 9. 아키텍처 요약

```
React ref
 ├─ section scope
 ├─ animation target
 └─ single DOM elements

Selector
 ├─ 반복 요소
 └─ 자식 요소 탐색
```

---

# 핵심 요약

React + GSAP 환경에서는 다음 구조를 유지하는 것이 가장 안정적이다.

```
단일 요소 → ref
반복 요소 → selector
부모 scope → ref
자식 탐색 → querySelector
```

---
