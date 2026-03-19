# ProjectSection 구조 리팩토링 정리

## 문제 상황

현재 ProjectsSection 의 전환 구조는 다음과 같은 문제를 가지고 있다.

### 1. 상태가 여러 컴포넌트에 분산됨

- `displayIndex`, `incomingIndex`. `isTransitioning` 이 상위에서 관리되지만 하위 컴포넌트들도 각각 `current/ next` 상태를 따로 처리

**결과 :**

- 상태 흐름 추적 어려움
- 디버깅 난이도 상승

### 2. current / next 구조가 중복됨

각 컴포넌트가 모두 이 구조를 가짐

- `ProjectFrameText`
- `ProjectFrameVisual`
- `ProjectFrameDetail`

**결과 :**

- ref 2배 증가
- 애니메이션 로직 중복
- clearProps / cleanup 문제 발생

### 3. 애니메이션 완료 타이밍이 분산됨

- Text/ Visual/ Detail 각각 timeline 이 존재
- onComplete 위치가 여러 군데

**결과 :**

- 전환 완료 시점 불명확
- UI 끊김 / 겹침 발생

## 리팩토링 목표

> "전환 상태는 한 곳에서 관리하고,
> 컴포넌트는 오직 렌더링만 담당하게 만든다."

## 구조 개선 방향

### 기존 구조 (문제 있는 구조)

```
ProjectFrame
 ├─ Text (current / next)
 ├─ Visual (current / next)
 └─ Detail (current / next)
```

모든 컴포넌트가 전환 상태를 알고 있음

### 개선 구조 (추천)

```
ProjectFrame
 ├─ ProjectLayer (current)
 │   └─ ProjectCard
 │       ├─ Text
 │       ├─ Visual
 │       └─ Detail
 │
 └─ ProjectLayer (next)
     └─ ProjectCard
         ├─ Text
         ├─ Visual
         └─ Detail
```

current / next는 상위에서만 관리

## 핵심 개념

### 1. Layer 기반 구조

- current layer
- next layer

두 개를 겹쳐놓고 애니메이션 처리

---

### 2. 하위 컴포넌트는 “project 하나만 받기”

```
interface Props {
  project: ProjectItem;
}
```

(X) 기존 (문제)

```
currentProject
nextProject
isTransitioning
```

완전히 제거

---

### 3. 애니메이션은 Frame에서만

- opacity
- transform
- background

**한 timeline에서 처리**

---

## 🔄 데이터 흐름

### Before

```
ProjectsSection
 → ProjectFrame
   → Text / Visual / Detail
      → 각각 current/next 관리
```

### After

```
ProjectsSection (상태 관리)
 → ProjectFrame (레이어 관리)
   → ProjectLayer (current / next)
     → ProjectCard (단일 project)
```

---

## 역할 분리

### ProjectsSection

- scroll → index 계산
- 전환 시작
- 전환 완료 처리

---

### ProjectFrame

- current / next 레이어 렌더링
- 애니메이션 실행

---

### ProjectCard

- 하나의 프로젝트 UI

---

### Text / Visual / Detail

- 순수 렌더링 컴포넌트
- 상태/애니메이션 모름

---

## 기존 문제의 근본 원인

- 상태와 표현이 섞여 있음
- 애니메이션이 분산되어 있음
- current / next 개념이 여러 군데 존재

👉 “전환 로직이 분해되어 있음”

---

## 리팩토링 효과

- 상태 흐름 단순화
- 애니메이션 충돌 제거
- 코드 가독성 증가
- 디버깅 난이도 감소
- 확장성 향상

---

## 다음 작업 순서

1. Text / Visual / Detail → `project` 단일 props로 변경
2. ProjectCard 컴포넌트 생성
3. ProjectLayer(current / next) 구조 도입
4. ProjectFrame에서만 애니메이션 처리
5. 기존 GSAP 분산 로직 제거

---

## 한 줄 정리

> 전환은 “레이어” 단위로 처리하고,
>
> 컴포넌트는 “데이터 하나만” 렌더링하게 만든다.
