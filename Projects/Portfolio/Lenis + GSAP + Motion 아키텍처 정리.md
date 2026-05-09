# Lenis + GSAP + Motion 아키텍처 정리

## 1. 개요

> **Lenis로 스크롤 물리를 만들고, GSAP으로 흐름을 설계하고, Motion으로 UI를 완성한다**

고급 웹 인터랙션 구현 시 사용하는 표준 조합:

- **Lenis** → 스크롤 감도 (물리)
- **GSAP** → 스크롤 기반 연출 (구조)
- **Motion** → UI 인터랙션 (디테일)

---

## 2. 역할 분담

| 도구       | 역할        | 설명                               |
| ---------- | ----------- | ---------------------------------- |
| **Lenis**  | 스크롤 엔진 | 부드러운 스크롤, lerp 보간         |
| **GSAP**   | 연출 엔진   | pin, scrub, timeline 기반 인터랙션 |
| **Motion** | UI 엔진     | hover, 클릭, layout 전환           |

---

## 3. 왜 이 조합을 사용하는가

### 3.1 GSAP → 스크롤 연출

- ScrollTrigger 기반
- pin / scrub / timeline 지원
- 복잡한 인터랙션 구현 가능

스크롤 연출은 사실상 GSAP이 표준

---

### 3.2 Motion → 컴포넌트 인터랙션

- React 친화적
- 선언형 애니메이션
- 상태 기반 제어

사용 영역:

- 버튼 hover
- 카드 등장
- 모달
- layout transition

---

### 3.3 Lenis → 스크롤 품질

- native scroll → 보간 적용
- inertia 느낌 추가

효과:

- 스크롤이 “쫀득하게” 느껴짐
- GSAP scrub 퀄리티 상승

---

## 4. 패키지 설치

```
pnpm add lenis motion gsap @gsap/react
```

또는 안정 버전:

```
pnpm add lenis framer-motion gsap @gsap/react
```

---

## 5. Motion 패키지 구조 변화

### 핵심

> “무거워서 버린 게 아니라, 구조가 분리된 것”

---

### 기존

```tsx
import { motion } from "framer-motion";
```

---

### 최신 (권장 방향)

```tsx
import { motion } from "motion/react";
```

---

### 변화 요약

| 항목   | framer-motion | motion           |
| ------ | ------------- | ---------------- |
| 구조   | 단일 패키지   | 모듈 분리        |
| 번들   | 상대적으로 큼 | 더 가벼움        |
| 범용성 | React 중심    | 다양한 환경 지원 |

---

## 6. 프로젝트 적용 구조

### 6.1 Lenis (Global)

- 위치: `layout.tsx`
- 역할: 전체 스크롤 감도 제어

모든 애니메이션의 기반

---

### 6.2 GSAP (Section 단위)

- 위치: 각 Section 컴포넌트

예:

- HeroSection
- AboutSection
- ProjectsSection

스크롤 흐름 설계

---

### 6.3 Motion (Component 단위)

- 위치: UI 컴포넌트

예:

- Button
- Card
- Modal

사용자 인터랙션 처리

---

## 7. 실무 핵심 규칙

### Rule 1. 한 요소 = 한 엔진

(X) GSAP + Motion 동시에 같은 요소 제어

문제:

- transform 충돌
- jitter 발생

---

### Rule 2. 역할 분리

권장 구조

```
GSAP → 컨테이너 이동
Motion → 내부 요소 인터랙션
```

---

### Rule 3. Motion으로 스크롤 구현 금지

(X) useScroll로 ScrollTrigger 대체

이유:

- pin 없음
- timeline 약함
- 복잡도 증가

스크롤은 GSAP이 담당

---

## 8. 실제 업계 구조

```
스크롤 감각 → Lenis / Locomotive
스크롤 연출 → GSAP
UI 인터랙션 → Motion or CSS
```

---

## 9. 도구 선택 기준

### Motion 사용

- hover
- 클릭
- 상태 변화
- 레이아웃 전환

---

### GSAP 사용

- 스크롤 연출
- pin
- scrub
- timeline

---

### Lenis 사용

- 전체 스크롤 품질 향상

---

## 10. 결론

> 이 조합은 과한 게 아니라 “표준 구조”

---

### 최종 요약

- Lenis → 감각
- GSAP → 구조
- Motion → 디테일

---

### 한 줄 정리

> **Lenis로 도로를 만들고, GSAP으로 흐름을 설계하고, Motion으로 UX를 완성한다**
