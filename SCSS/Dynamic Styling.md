# 동적 스타일링(Dynamic Styling) 이란

이는 동적 스타일링 또는 데이터 기반 스타일링이라고 부른다.

핵심은 **"변하지 않는 틀(CSS 클래스)"** 과 **"데이터에 따라 변하는 값(인라인 스타일)"** 을 분리해서 관리하는 기술적 패턴이다.

## 1. 관심사 분리 (Separation of Concerns)

1. **className**
   - 레이아웃, 크기, 간격 등 **정적인 디자인**을 담당한다.
2. **style**
   - 사용자가 설정한 테마색, 배경 이미지 등 **서버 데이터나 상태(state)** 에 따라 바뀌는 값을 담당한다.

## 2. CSS 변수 주입 (CSS Variable Injection)

1. `--project-accent` 같이 인라인으로 넘기는 것은 아주 세련된 방식이다.
2. 부모 요소에서 이 변수를 한 번만 선언해두면, 자식 요소들은 별도의 Props 전달 없이도 CSS 파일에서 `color: var(--project-accent);` 처럼 이 값들을 공유해 쓸 수 있다.

## 3. 타입 안정성 (Type Assertion)

1. `as React.CSSProperties` 는 타입스크립트에게 "이 객체는 리액트 스타일 객체 표준을 따르니 안심해" 라고 알려주는 장치이다 커스텀 CSS 변수를 사용할 때 발생하는 에러를 막아준다.

```ts
<div
    className={`project-frame__hero project-frame__hero--${project.tone}`}
        style={
          {
            "--project-accent": project.themeColor,
            backgroundColor: project.background,
          } as React.CSSProperties
        }
      >
```

결론적으로, 이 패턴은 **디자인 시스템은 유지하되 특정 데이터(컬러, 배경 등)만 유연하게 갈아 끼우고 싶을 때** 사용하는 표준적인 리액트 구현 방식이다.
