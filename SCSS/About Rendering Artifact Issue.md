# About Rendering Artifact Issue_SVG로 전환

### 렌더링 아티팩트(Rendering Artifact)란?

**렌더링 아티팩트** 란 의도한 디자인과 다르게, 렌더링 과정에서 발생하는 **비정상적인 시각적 흔적**을 의미한다.

대표적인 예로는:

- 경계에 생기는 미세한 선
- 픽셀 깨짐
- 색 번짐
- 불균일한 두께 변화 등이 있다.
  이번 bomWaveTitle과 About me 타이틀에서 발생한 현상은 **텍스트 내부에 생긴 헤어라인 형태의 렌더링 아티팩트** 에 해당한다.

## 배경

포트폴리오 Hero 섹션의 `bomWaveTitle.tsx` 를 구현하면서 큰 사이즈의 타이틀 텍스트에 **외곽선(text-stroke) + 내부 fill 애니메이션** 을 적용했다.

초기에는 HTML 텍스트에 `-webkit-text-stroke` 를 사용해 스트로크 효과를 주는 방식으로 시작했다.

### `-webkit-text-stroke` 란?

**텍스트의 "윤곽선(외곽선)"을 그려주는 CSS 속성이다.**

원래 텍스트는 색(color)만 있고, 선("stroke")이라는 개념이 없다. 그런데`webkit-text-stroke` 는 **글자를 하나의 도형처럼 취급해서 테두리를 그려주는 속성** 이다.

---

## 발생한 문제

폰트 사이즈가 커지고, 굵은 웨이트를 사용할수록 텍스트 내부에 의도하지 않은 선(헤어라인)이 생기는 현상이 발생했다.

![](https://private-user-images.githubusercontent.com/167315197/536123660-f19cdd96-a485-4079-9270-56b6712242a4.png?jwt=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3Njg0NjcwOTEsIm5iZiI6MTc2ODQ2Njc5MSwicGF0aCI6Ii8xNjczMTUxOTcvNTM2MTIzNjYwLWYxOWNkZDk2LWE0ODUtNDA3OS05MjcwLTU2YjY3MTIyNDJhNC5wbmc_WC1BbXotQWxnb3JpdGhtPUFXUzQtSE1BQy1TSEEyNTYmWC1BbXotQ3JlZGVudGlhbD1BS0lBVkNPRFlMU0E1M1BRSzRaQSUyRjIwMjYwMTE1JTJGdXMtZWFzdC0xJTJGczMlMkZhd3M0X3JlcXVlc3QmWC1BbXotRGF0ZT0yMDI2MDExNVQwODQ2MzFaJlgtQW16LUV4cGlyZXM9MzAwJlgtQW16LVNpZ25hdHVyZT0zNTAxZGQ5YjNjMjAzNGYzYmMyZTg3NGVjMzBjYWI4NDFjNzU1ZmExZTYyODM4NjA3ZDExZTBjMWI4OWFlNGE3JlgtQW16LVNpZ25lZEhlYWRlcnM9aG9zdCJ9.WANww32EYtxlr63TPNlEAZs1lqA3BAUVkvYXaKGUPug)

이 문제는 특히

- 다크 배경에서 스트로크 텍스트를 메인 비주얼로 사용할 때 매우 눈에 띄었고, UI 완성도를 해쳤다.
  그리고 이러한 문제는 단순한 스타일 오류가 아니라, 브라우저마다 **미묘하게 다르게 나타나는 렌더링 문제라는 점도 확인** 하였다.
  > 특히 크롬에서 자주 뜨고, 확대/축소(줌)나 GPU 합성 상태에 따라 더 눈에 띄기도 함.

## 원인 분석\_왜 text-stroke에서 이런 문제가 생겼을까?

`-webkit-text-stroke` 는 CSS레벨에서 텍스트에 외곽선을 그리는 방식이지만, 실제 렌더링은 블우저 **폰트 렌더링(헌팅, 안티앨리어싱)** 파이프라인을 그대로 따른다.

이 과정에서는:

- 폰트 헌팅(Font Hinting)
- 안티앨리어싱(Anti-aliasing)
- 서브픽셀 렌더링
  같은 단계들이 포함된다.

굵은 폰트 + 큰 사이즈, fill과 stroke 경계가 픽셀 단위로 정확히 맞지 않으며 내부에 미세한 선이나 경계 깨짐 같은 **아티팩트** 가 발생할 수 있다. 그리고 이 문제는 CSS 속성 조정만으로는 **완전히 제어하기 어려다는** 한계가 있다.

즉, 이는 단순히 css 오류가 아닌, 브라우저 텍스트 렌더링 특성에서 비롯된 구조적인 한계라고 볼 수 있음.

### 처음에 시도해본 방법

text-stroke 기반 텍스트에서도 아티팩트를 완화하기 위한 방법들을 시도해보았다.

1. **stroke 두께 줄이고 대비로 보완**
   굵은 stroke일수록 내부 헤어라인이 더 잘보이는 경우가 많다.

```css
-webkit-text-stroke: 1px rgba(255, 255, 255, 0.75);
color: rgba(255, 255, 255, 0.1);
```

- 두께는 줄이고, opacity 대비로 외곽선을 강조
- 일부 상황에서는 개선되지만 완전한 해결책은 아니였음

2. **translateZ(0) /will-change로 합성 고정**
   렌더링 과정에서 발생하는 떨림이나 경계 불안정이 조금 줄어드는 경우가 있다.

```css
.about-title {
  transform: translateZ(0);
  will-change: transform;
}
```

- GPU 합성 단계로 넘기기 위한 트릭
- 환경에 따라 효과가 있기도, 없기도 하다.

3. **텍스트를 한겹 더 만들어 "가짜 stroke" 구현**
   `text-shadow`를 여러 번 사용해 외곽선을 흉내내는 방식이다.

```css
text-shadow: 1px 0 0 #fff, -1px 0 0 #fff, 0 1px 0 #fff, 0 -1px 0 #fff;
```

- 내부 헤어라인이 줄어드는 경우도 있음
- 하지만 큰 사이즈에선 한계 명확

---

## 해결-SVG path 기반 타이틀로 전환

위와 같은 방법들을 여러번 시도해보았지만 결국 해결되진 않았다. 즉, 대형 타이틀 \_ 스트로크 중심 디자인에선 한계가 존재

그래서 bomWaveTitle에서는 텍스트를 HTML 텍스트가 아닌 SVG Path로 처리하였다.

SVG로 전환하면서:

- 텍스트가 폰트가 아닌 **백터 경로**로 렌더링된다.
  - 폰트 헌팅, 안티앨리어싱 영향에서 벗어남
- stroke/ fill을 **정확하게 분리 제어 가능**
- 물결 애니메이션(bomWave)처럼 마스킹 이동 애니메이션을 안정적으로 적용 가능

결과적으로 텍스트 내부에 생기던 렌더리 아티팩트를 제거하기 위해선 SVG를 사용하는 방법도 하나의 해결책이 될 수 있다.

## 정리

이번 경험을 통해 "텍스트 UI"라도 상황에 따라 **그래픽 요소처럼 다뤄야 할 때** 가 있다는 걸 배웠다.

특히,

- 대형 타이틀
- 스트로크 기반 디자인
- 시각적 완성도가 중요한 메인 비주얼
  이런 경우에는 CSS text-stroke 보다 **SVG 지향 접근이 더 안정적인 선택** 이였다.

#### 한 줄 요약

text-stroke는 렌더링 아티팩트를 완전히 제어하기 어렵고, bomWaveTitle에서느 SVG path 기반 텍스트로 전환해 문제르 근본적으로 해결하였다.
