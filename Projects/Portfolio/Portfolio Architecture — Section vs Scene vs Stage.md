# Portfolio Architecture — Section vs Scene vs Stage

## 1. 핵심 개념 차이

| 개념   | Section           | Scene                          |
| ------ | ----------------- | ------------------------------ |
| 역할   | 페이지 구획       | 연출 장면                      |
| 기준   | 문서 구조         | 타임라인/스크롤 흐름           |
| 독립성 | 독립 레이아웃     | 다른 장면과 연결됨             |
| 목적   | 정보 전달         | 경험/전환 연출                 |
| 스크롤 | 자연스럽게 지나감 | pinned/sticky/transition 가능  |
| 예시   | About, Contact    | Hero intro, reveal, transition |

---

# 2. Section이란?

Section은:

> “페이지의 정보 단위”

이다.

즉:

- 독립된 콘텐츠를 가짐
- 혼자 존재 가능함
- 문서 흐름 기반
- 일반적인 웹사이트 구조

예시:

```
AboutSection
ProjectsSection
ContactSection
```

특징:

- 다음 섹션 없어도 의미 성립
- 각각 자체 목적 존재
- semantic/document 구조 중심

즉 CRUD 사이트나 일반 랜딩페이지 구조에 가까움.

---

# 3. Scene이란?

Scene은:

> “스크롤 기반 연출 장면”

이다.

특징:

- 다른 Scene과 연결됨
- transition 전제
- sticky/pin 사용 가능
- timeline 기반
- 독립 콘텐츠보다 흐름이 중요

예시:

```
HeroScene
↓
LifeMotionScene
```

이건 단순 섹션이 아니라:

> “스크롤 영화의 장면 전환”

에 가까움.

---

# 4. 왜 Hero + LifeMotion은 Scene인가?

현재 구조:

```tsx
<sectionclassName="hero-life-transition">
<Hero/>
<LifeMotion/>
</section>
```

이건:

```
독립 섹션 2개
```

가 아니라:

```
하나의 sticky stage 안에서
두 장면이 이어지는 구조
```

이다.

핵심이:

```
콘텐츠
```

가 아니라

```
전환 경험
```

이기 때문.

즉:

- Hero가 끝나며
- LifeMotion이 등장하고
- 서로 transition으로 연결됨

→ Scene 개념이 맞다.

---

# 5. 왜 Naming이 중요한가?

처음 헷갈렸던 이유:

```
"왜 HeroSection이 LifeMotion까지 침범하지?"
```

하지만 Scene 관점으로 보면:

```
"원래 같은 무대 위 장면인데?"
```

가 된다.

즉 구조 이해 자체가 달라진다.

---

# 6. Stage 개념

Stage는:

> “여러 Scene을 담는 하나의 연출 무대”

이다.

구조 예시:

```
IntroStage
 ├─ HeroScene
 └─ LifeMotionScene
```

즉:

- Scene = 장면
- Stage = 장면 묶음

이다.

---

# 7. 현재 포트폴리오 구조 기준

```
HomePage
 ├─ IntroStage
 │   ├─ HeroScene
 │   └─ LifeMotionScene
 │
 ├─ AboutStage
 │   ├─ AboutHeroScene
 │   ├─ InterviewScene
 │   └─ SkillsScene
 │
 ├─ ProjectsStage
 │   ├─ ShowcaseScene
 │   ├─ MonitorScene
 │   └─ DetailScene
 │
 └─ ContactStage
```

이 구조가 현재 방향성과 가장 잘 맞는다.

---

# 8. 왜 네 포트폴리오는 Stage 중심 구조인가?

현재 지향점:

- Apple style storytelling
- WWDC 느낌
- Scroll cinematic UX
- pinned transitions
- GSAP timeline flow
- scroll-driven narrative

이런 구조는 대부분:

```
Stage
 ├─ Scene
 ├─ Scene
 └─ Scene
```

방식으로 설계된다.

반면 일반 CRUD 사이트는:

```
Page
 ├─ Section
 ├─ Section
 └─ Section
```

구조를 사용한다.

---

# 9. Stage 판단 기준

## IntroStage

```
Hero → LifeMotion
```

- 강한 transition 존재
- 하나의 cinematic flow
- sticky/pinned 기반

→ Stage가 맞음

---

## AboutStage

```
Hero → Interview → Skills
```

- 순차적 reveal
- narrative 흐름 존재
- 스크롤 기반 연출 가능

→ Stage 가능

---

## ProjectsStage

```
Pinned monitor showcase
```

- activeIndex 기반 전환
- timeline 흐름
- monitor reveal

→ Stage가 맞음

---

## ContactStage

```
Footer CTA reveal
```

- reveal 중심 연출이면 Stage 가능
- 단일 static section이면 Section도 가능

---

# 10. Scene Naming은 아껴 써라

Scene은:

> 내부 장면 흐름이 실제로 존재할 때만 사용

하는 게 좋다.

좋은 예시:

```
HeroScene
LifeMotionScene
InterviewScene
SkillsScene
```

애매한 예시:

```
ContactScene
```

Contact가 단일 정적 화면이면:

```
ContactSection
```

정도가 더 자연스럽다.

---

# 11. 최종 정리

## Section

```
정보 단위
문서 구조
독립 콘텐츠
```

---

## Scene

```
연출 장면
transition 중심
timeline 기반
```

---

## Stage

```
여러 Scene을 담는 연출 무대
스크롤 서사 단위
```

---

# 12. 네 포트폴리오의 본질

현재 내 포트폴리오는:

```
정보형 웹사이트
```

가 아니라

```
스크롤 기반 인터랙티브 스토리텔링 경험
```

에 가깝다.

그래서:

```
Section 중심 사고
```

보다

```
Stage → Scene 흐름 사고
```

가 훨씬 자연스럽다.
