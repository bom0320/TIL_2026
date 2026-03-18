# Troubleshooting: GSAP 전환 후 요소가 사라지는 문제

## 문제 상황

프로젝트 섹션에서 스크롤 기반 전환 애니메이션을 구현하던 중, 스크롤할 때는 요소가 잠깐 정상적으로 보이지만 스크롤이 멈추면 텍스트와 이미지 같은 실제 콘텐츠가 사라지는 문제가 발생했다.

### 증상

- 전체 레이아웃은 유지됨
- 모니터 프레임, 섹션 제목 같은 고정 요소는 보임
- 실제 프로젝트 텍스트, 이미지, 디테일 콘텐츠만 사라짐
- 다시 스크롤하면 다음 전환 중에는 잠깐 보였다가, 멈추면 또 사라짐

즉, 레이아웃 자체가 깨진 것이 아니라 **콘텐츠 레이어만 투명해진 상태**처럼 보였다.

---

## 초기 의심

처음에는 다음과 같은 가능성을 의심했다.

- `position: absolute` / `stack` 구조 문제
- `z-index` 충돌
- `sticky` 동작 문제
- 조건부 렌더링 타이밍 문제
- GSAP 타임라인 종료 시점 문제

하지만 화면을 보면 프레임과 레이아웃은 멀쩡하게 남아 있었고,

실제 내용물만 안 보이는 점에서 CSS 레이아웃 붕괴보다는 **애니메이션 스타일 잔존 문제** 쪽이 더 유력했다.

---

## 원인 분석

프로젝트 전환 애니메이션은 `current` 요소를 사라지게 하고, `next` 요소를 나타나게 하는 방식으로 구성되어 있었다.

예를 들어 텍스트 전환은 아래와 같았다.

```scss
.to(currentText, { opacity:0, y:-20 },0)
.to(nextText, { opacity:1, y:0 },0.12)
```

이미지 전환도 비슷하게 구성되어 있었다.

```scss
.to(currentImage, { opacity:0, scale:0.98 },0)
.to(nextImage, { opacity:1, scale:1 },0.1)
```

이 구조 자체는 맞다. 문제는 **전환이 끝난 뒤** 발생했다.

### 실제로 일어난 일

1. 전환 중 `current` 요소는 `opacity: 0`, `transform: ...` 상태가 됨
2. `next` 요소는 보이게 됨
3. 전환 완료 후 상위 상태가 바뀌면서 `nextProject`는 사라지고, `nextProject`가 `currentProject`가 됨
4. 이때 React가 DOM을 완전히 새로 만들지 않고 기존 `current` 레이어를 재사용할 수 있음
5. 그러면 이전 GSAP이 남긴 inline style이 그대로 유지됨

예를 들어 이런 스타일이 남아 있을 수 있다.

```scss
opacity: 0;
transform: translateY(-20px);
```

결국 데이터는 새 프로젝트로 바뀌었는데,

그 데이터를 담는 current 레이어는 여전히 투명한 상태라서 화면상으로는 **내용이 사라진 것처럼 보이게 된다.**

---

## 왜 스크롤 중에는 잠깐 보였는가

전환 중에는 `isTransitioning === true` 상태라서 `next` 레이어가 함께 렌더링된다.

즉, 화면에 실제로 보이는 것은 새 current가 아니라 **애니메이션 중인 next 레이어**다.

하지만 스크롤이 멈춰 전환이 끝나면:

- `next` 레이어는 제거되고
- 남아야 할 `current` 레이어는 이미 `opacity: 0` 상태

이 때문에 화면이 텅 빈 것처럼 보였다.

---

## 핵심 원인

**GSAP이 적용한 inline style이 전환 완료 후에도 남아 있었고, React가 기존 DOM을 재사용하면서 그 스타일이 다음 current 요소에 그대로 이어졌다.**

즉 문제는 스크롤 자체가 아니라,

**전환 후 스타일 복구가 이루어지지 않았다는 점**이었다.

---

## 해결 방법

이번 문제는 아래 두 가지를 우선 적용하는 방식으로 접근했다.

---

### 1. `key`를 부여해서 current / next 레이어 강제 remount

React가 기존 DOM을 재사용하지 않고, 프로젝트가 바뀔 때 새로운 노드로 갈아끼우도록 `key`를 부여했다.

> ProjectFrameText.tsx, ProjectFrameVisual.tsx, ProjectFrameDetail.tsx 수정

예를 들어 텍스트 current 레이어는 다음처럼 수정할 수 있다.

```tsx
<div
key={`current-${currentProject.id}`}
ref={currentTextRef}
className="project-frame__info project-frame__info--current"
>
```

전환 중 next 레이어도 동일하게 처리한다.

```tsx
<div
key={`next-${nextProject.id}`}
ref={nextTextRef}
className="project-frame__info project-frame__info--next"
>
```

이 방식은 **GSAP이 남긴 스타일이 다음 프로젝트 current 레이어에 전염되는 가능성**을 줄여준다.

---

### 2. 애니메이션 종료 후 `clearProps`로 inline style 초기화

> animations/project.ts 파일일의 함수들 수정

이건 더 근본적인 해결책이다.

GSAP은 애니메이션 과정에서 `opacity`, `x`, `y`, `scale`, `transform` 같은 값을 요소에 inline style로 남긴다.

전환이 끝난 뒤 이 값들을 지워주지 않으면 다음 렌더링에서 문제가 발생할 수 있다.

그래서 timeline 마지막에 `clearProps`를 추가해 스타일을 정리했다.

**텍스트 전환 예시**

```tsx
exportfunctioncreateProjectHeroTextTransition({
  currentText,
  nextText,
}:ProjectHeroTextTransitionParams) {
gsap.set(nextText, { opacity:0, y:20 });

returngsap
.timeline({
      defaults: {
        duration:0.45,
        ease:"power2.out",
      },
    })
.to(currentText, { opacity:0, y:-20 },0)
.to(nextText, { opacity:1, y:0 },0.12)
.set([currentText,nextText], { clearProps:"all" });
}
```

**이미지 전환 예시**

```tsx
exportfunctioncreateProjectHeroVisualTransition({
  currentImage,
  nextImage,
}:ProjectHeroVisualTransitionParams) {
gsap.set(nextImage, { opacity:0, scale:1.04 });

returngsap
.timeline({
      defaults: {
        duration:0.5,
        ease:"power2.out",
      },
    })
.to(currentImage, { opacity:0, scale:0.98 },0)
.to(nextImage, { opacity:1, scale:1 },0.1)
.set([currentImage,nextImage], { clearProps:"all" });
}
```

**detail 전환 예시**

```tsx
exportfunctioncreateProjectDetailTransition({
  currentKeyword,
  nextKeyword,
  currentOverview,
  nextOverview,
  currentThumbs,
  nextThumbs,
  onComplete,
}:ProjectDetailTransitionParams) {
gsap.set(nextKeyword, { opacity:0, y:8 });
gsap.set(nextOverview, { opacity:0, y:10 });
gsap.set(nextThumbs, { opacity:0, x:24 });

returngsap
.timeline({
      defaults: {
        duration:0.35,
        ease:"power2.out",
      },
      onComplete,
    })
.to(currentKeyword, { opacity:0, y:-8 },0)
.to(currentOverview, { opacity:0, y:-10 },0)
.to(currentThumbs, { opacity:0, x:-24 },0)
.to(nextKeyword, { opacity:1, y:0 },0.12)
.to(nextOverview, { opacity:1, y:0 },0.16)
.to(nextThumbs, { opacity:1, x:0 },0.12)
.set(
      [
currentKeyword,
nextKeyword,
currentOverview,
nextOverview,
currentThumbs,
nextThumbs,
      ],
      { clearProps:"all" }
    );
}
```

---

### 3. **보조 안전장치: 비전환 상태에서 current 레이어 강제 초기화**

> ProjectFrameText.tsx 수정

이건 보조 안전장치로 사용할 수 있다.

전환이 아닐 때 `current` 요소에 남아 있는 inline style을 강제로 비워서,

혹시라도 남아 있는 상태값 때문에 화면이 숨겨지지 않도록 방어할 수 있다.

**1. 기존 코드**

```tsx
useLayoutEffect(() => {
  if (!isTransitioning || !nextProject) return;

  constcurrentText = currentTextRef.current;
  constnextText = nextTextRef.current;

  if (!currentText || !nextText) return;

  const tl = ProjectAnimation.createProjectHeroTextTransition({
    currentText,
    nextText,
  });

  return () => {
    tl.kill();
  };
}, [isTransitioning, nextProject]);
```

이 코드는 **전환이 시작됐을 때만 애니메이션을 실행하는 코드**다.

즉 역할은 딱 하나다:

> `isTransitioning === true` 이고 `nextProject`가 있을 때
>
> current → next 애니메이션을 시작한다.

**이 코드의 특징**

- 전환 중일 때만 동작
- 전환이 아닐 때는 아무 것도 안 함
- current에 남아 있는 스타일을 정리하진 않음
- 즉 **애니메이션 실행 담당**

**2. 보조 안전장치 포함 코드**

```tsx
useLayoutEffect(() => {
  if (!isTransitioning) {
    if (currentTextRef.current) {
      gsap.set(currentTextRef.current, { clearProps: "all" });
    }
    return;
  }

  if (!nextProject) return;

  constcurrentText = currentTextRef.current;
  constnextText = nextTextRef.current;

  if (!currentText || !nextText) return;

  const tl = ProjectAnimation.createProjectHeroTextTransition({
    currentText,
    nextText,
  });

  return () => {
    tl.kill();
  };
}, [isTransitioning, nextProject, currentProject.id]);
```

이건 역할이 두 개다.

- **역할 1**
  - 전환 중일 때 애니메이션 실행
  - 기존 코드와 동일
- **역할 2**
  - 전환이 아닐 때 current 요소에 남아 있을 수 있는 inline style 정리
  - `clearProps: "all"`로 초기화

즉 이건:

> **애니메이션 실행 + 비전환 상태 복구**

를 둘 다 담당한다.

즉, 정리해보면

`clearProps`를 timeline 마지막에 추가하는 것만으로도 대부분 해결되지만,

상태 전환 타이밍이나 DOM 재사용 방식에 따라 current 레이어에 이전 inline style이 남아 있을 가능성을 완전히 배제할 수는 없다.

이 경우를 대비해, **전환이 아닐 때 current 요소의 inline style을 한 번 더 초기화하는 보조 effect**를 둘 수 있다.

이 방식은 전환 중에는 기존과 동일하게 애니메이션을 실행하고,

전환이 끝난 뒤에는 `currentProject.id` 변경을 감지해 새 current 레이어의 스타일을 다시 초기화한다.

즉 이 effect는 단순히 애니메이션을 실행하는 역할만 하는 것이 아니라,

**비전환 상태에서 화면이 항상 정상 스타일을 유지하도록 보정하는 안전장치** 역할도 한다.

다만 이 방식은 어디까지나 보조책에 가깝고,

우선적으로는 다음 두 가지가 먼저 적용되어야 한다.

- current / next 레이어에 `key`를 부여해 DOM 재사용 가능성 줄이기
- timeline 마지막에 `clearProps: "all"`을 추가해 GSAP inline style 정리하기

---

## 왜 `kill()`만으로는 해결되지 않았는가

처음에는 cleanup에서 `tl.kill()`을 호출하고 있었기 때문에, 타임라인 정리만으로 충분할 수 있다고 생각했다.

하지만 `kill()`은 **애니메이션 실행 자체를 중단하거나 정리하는 역할**이고,

이미 요소에 적용된 inline style을 자동으로 복구하지는 않는다.

즉:

- `kill()` → 애니메이션 엔진 정리
- `clearProps` → 요소 스타일 원복

둘은 역할이 다르다.

---

## 적용 우선순위

이번 문제에서는 아래 순서로 접근하는 것이 가장 현실적이었다.

### 1순위

current / next 래퍼 요소에 `key`를 부여해서 DOM 재사용 가능성 낮추기

### 2순위

timeline 마지막에 `clearProps: "all"` 추가해서 GSAP inline style 초기화

이 두 가지 조합만으로도 대부분의 경우 문제가 해결될 가능성이 높다는 결론이 나왔다.

---

## 배운 점

이번 이슈를 통해 단순히 “애니메이션이 보이느냐”만 보는 것이 아니라,

**애니메이션이 끝난 뒤 DOM이 어떤 상태로 남는지까지 관리해야 한다**는 점을 배웠다.

특히 React와 GSAP를 함께 사용할 때는 다음을 항상 의식해야 한다.

- React는 DOM을 재사용할 수 있다
- GSAP은 요소에 inline style을 직접 남긴다
- 따라서 전환 후 스타일 초기화가 없으면 다음 렌더링에서 문제가 생길 수 있다

즉, 전환 애니메이션은 “보이는 순간”만 구현하는 것이 아니라

**끝난 뒤 상태를 어떻게 정리할지까지 포함해서 설계해야 한다.**

---

## 정리

이번 문제의 핵심은 다음 한 줄로 요약할 수 있다.

> GSAP 전환 과정에서 current 요소에 남은 inline style이, React의 DOM 재사용과 겹치면서 다음 current 레이어를 계속 `opacity: 0` 상태로 만들고 있었다.

그리고 해결의 핵심은 다음 두 가지였다.

- `key`를 통해 current / next 레이어를 명확히 분리하기
- `clearProps`를 통해 전환 후 GSAP inline style을 정리하기
