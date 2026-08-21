# Safari와 Chrome의 Intrinsic Sizing 차이로 발생한 Life Motion 중첩 문제

## 문제 상황

Life Motion 영역은 여러 이미지를 위·아래 두 행으로 나누고, 각 행의 이미지 그룹을 복제하여 가로로 이어지는 형태로 구현했습니다.

```
TOP
[1][3][5][7][9] | [1][3][5][7][9]

BOTTOM
[2][4][6][8][10] | [2][4][6][8][10]
```

Chrome에서는 의도한 대로 렌더링됐지만 Safari에서는 **Original Group과 Clone Group의 이미지가 서로 겹쳐 보이는 문제**가 발생했습니다.

스크롤 기반 인터랙션 영역이었기 때문에 처음에는 GSAP의 초기 `transform`, ScrollTrigger의 progress, Safari의 GPU compositing 등을 의심했습니다.

실제로 Safari에서 스크롤 위치와 GSAP이 주입한 `transform`, `opacity` 값을 확인하고 `mask-image`, `filter`, `backface-visibility` 등 렌더링에 영향을 줄 수 있는 속성을 하나씩 제거해 비교했지만 동일한 문제가 유지됐습니다.

이를 통해 애니메이션 자체보다는 **DOM 구조와 브라우저의 레이아웃 계산 과정**에 문제가 있다고 판단했습니다.

---

## 기존 구조를 다시 이해하며 발견한 반복 로직

문제를 추적하면서 Life Motion이 실제로 어떤 방식으로 데이터를 만들고 렌더링하고 있는지 다시 살펴봤습니다.

기존에는 다음과 같이 반복 횟수를 별도의 상수로 관리하고 있었습니다.

```tsx
const REPEAT_IN_GROUP = 2;
```

처음에는 이 값을 **위·아래 두 행을 만들기 위해 필요한 반복 횟수**라고 이해했습니다.

하지만 실제 데이터를 만드는 코드를 추적해보니 위·아래 행은 이미 다음 로직에서 분리되고 있었습니다.

```tsx
const topBaseItems = items.filter((_, index) => index % 2 === 0);
const bottomBaseItems = items.filter((_, index) => index % 2 === 1);
```

즉 전체 데이터가:

```
1 2 3 4 5 6 7 8 9 10
```

이라면 이 시점에서 이미:

```
TOP
1 3 5 7 9

BOTTOM
2 4 6 8 10
```

으로 두 행이 완성됩니다.

`REPEAT_IN_GROUP`은 행의 개수를 결정하는 값이 아니라 **각 행 내부의 이미지 묶음을 몇 번 반복할 것인지 결정하는 값**이었습니다.

그런데 렌더링 단계에서도 Original Group과 동일한 Clone Group을 이미 한 번 더 생성하고 있었습니다.

```
ROW
├─ Original Group
│  └─ 1 3 5 7 9
│
└─ Clone Group
   └─ 1 3 5 7 9
```

따라서 `REPEAT_IN_GROUP = 2`일 경우 실제로는:

```
Original Group
1 3 5 7 9 | 1 3 5 7 9

Clone Group
1 3 5 7 9 | 1 3 5 7 9
```

처럼 **데이터 생성 단계와 렌더링 단계에서 반복이 중복되어 있었습니다.**

이 구조를 정확히 이해한 뒤 DOM을 다시 확인하면서, 문제가 단순히 이미지가 여러 번 렌더링돼서 발생하는 것이 아니라 **Safari에서 두 Group의 레이아웃 너비 자체가 다르게 계산되고 있다는 점**을 발견했습니다.

---

# 문제가 발생한 핵심 원인

문제 상황은 WebKit(Safari)과 Blink(Chrome/Edge) 엔진 간에 **Intrinsic Sizing(고유 크기 계산) 과정에서 `flex-basis`를 처리하는 방식의 차이**가 드러난 경우였습니다.

## 1. 부모의 Intrinsic Sizing

부모 요소가:

```css
width: max-content;
```

또는 `fit-content`처럼 콘텐츠 기반의 크기를 사용할 경우, 부모는 자신의 너비를 바로 알 수 없습니다.

브라우저는 먼저:

> “내부 자식들이 차지하는 전체 크기가 얼마인가?”

를 계산한 뒤 그 값을 기반으로 부모의 너비를 결정합니다.

즉 `max-content`는 단순한 고정 너비가 아니라 **자식 콘텐츠의 크기를 기준으로 계산되는 Intrinsic Size**입니다.

기존 Life Motion의 Row도 다음과 같은 구조였습니다.

```css
.life-motion__row {
  display: flex;
  width: max-content;
}
```

따라서 Row의 전체 너비를 결정하기 위해서는 내부 Group과 Item의 크기를 먼저 알아야 했습니다.

---

## 2. `flex-basis`와 Intrinsic Width의 충돌

문제는 각 카드의 크기를 실제 `width`가 아니라 `flex-basis`에만 의존하고 있었다는 점입니다.

```css
.life-motion__item {
  flex: 0 0 clamp(240px, 20vw, 360px);
}
```

이를 풀어쓰면:

```css
.life-motion__item {
  flex-grow: 0;
  flex-shrink: 0;
  flex-basis: clamp(240px, 20vw, 360px);
}
```

입니다.

이때 명시적인 `width`는 존재하지 않기 때문에 실질적으로:

```css
width: auto
flex-basis: clamp(...)
```

상태입니다.

`flex-basis`는 단순히 요소의 `width`를 지정하는 속성이 아니라, **Flexbox가 Main Axis 방향의 크기를 계산할 때 사용하는 기준 크기**입니다.

따라서 기존 구조에서는 부모의 Intrinsic Size를 계산하는 과정과 Flexbox가 자식의 실제 크기를 계산하는 과정이 서로 중첩되어 있었습니다.

```markdown
ITEM
width: auto
flex-basis: clamp(...)
↓
GROUP
자식의 크기를 기반으로 자신의 너비 계산
↓
ROW
GROUP의 크기를 기반으로 max-content 계산
```

즉 부모는:

> “자식들이 실제로 얼마나 넓은지 알아야 내 너비를 결정할 수 있다.”

는 상황인데,

정작 자식의 크기도 명시적인 `width`가 아니라 **Flexbox의 계산 결과에 의존**하고 있었습니다.

---

# Blink와 WebKit의 계산 결과 차이

Chrome과 Edge는 Blink, Safari는 WebKit이라는 서로 다른 렌더링 엔진을 사용합니다.

이번처럼:

```
Nested Flex Container
+
flex-basis
+
width: auto
+
max-content
+
clamp()
+
vw
```

와 같이 여러 크기 계산 규칙이 중첩된 경우, 두 엔진에서 Intrinsic Sizing 결과가 다르게 나타났습니다.

### Blink (Chrome)

Chrome에서는 부모의 Intrinsic 크기를 계산할 때 자식의 `flex-basis`를 기반으로 한 크기가 상위 Group의 너비 계산에도 의도한 형태로 반영됐습니다.

따라서 첫 번째 Group의 카드가 차지하는 전체 영역만큼 Group의 너비가 계산되고, 그 다음 위치에서 Clone Group이 시작했습니다.

```
Original Group
|--------------------------------|

[1] [3] [5] [7] [9]
                                [1] [3] [5] [7] [9]
                                |------ Clone Group ------
```

즉:

```
Safari가 아니라 Chrome에서는

Group의 계산된 너비
≈
실제 카드가 차지하는 전체 너비
```

가 일치하고 있었습니다.

---

### WebKit (Safari)

Safari에서는 부모의 Intrinsic 크기를 계산할 때 자식에 명시적인 `width`나 `min-width`가 없는 상태에서 `flex-basis`에만 크기를 의존하자, Group의 계산된 너비와 Flexbox 배치 후 카드들이 실제 차지하는 너비가 다르게 나타났습니다.

개념적으로 보면:

```
Safari가 계산한 Group Box
|------------------|

실제 카드가 차지하는 영역
[1] [3] [5] [7] [9]
```

처럼 **부모가 생각하는 자신의 너비보다 실제 자식 카드들이 더 넓은 공간을 차지하는 상태**가 된 것입니다.

그리고 Original Group 다음에 Clone Group이 배치되기 때문에 Clone의 시작 위치는 **Safari가 계산한 Original Group의 너비**를 기준으로 결정됩니다.

```
Original
[1] [3] [5] [7] [9]

Safari가 계산한 Group 종료 지점
                  ↓
                  [1] [3] [5] [7] [9]
                  ↑
                Clone 시작
```

그 결과 첫 번째 Group의 카드가 아직 끝나지 않았는데 두 번째 Group이 시작하면서 서로 같은 공간을 사용하게 되었습니다.

화면에서는 이것이:

> 같은 이미지가 한 겹 더 렌더링된 것처럼 보이는 현상

으로 나타났습니다.

---

# 해결: Intrinsic Sizing 단계에서 명확한 크기 정보 제공

Safari가 부모의 크기를 계산하는 과정에서 `flex-basis`만을 근거로 자식의 크기를 추론하지 않도록, 카드에 실제 `width`를 명시했습니다.

### Before

```css
.life-motion__item {
  flex: 0 0 clamp(240px, 20vw, 360px);
}
```

### After

```css
.life-motion__item {
  flex: 0 0 auto;

  width: clamp(240px, 20vw, 360px);
  height: clamp(220px, 28vw, 320px);
}
```

`flex: 0 0 auto`를 풀어쓰면:

```css
flex-grow: 0;
flex-shrink: 0;
flex-basis: auto;
```

가 됩니다.

따라서 카드 크기의 기준을:

```
flex-basis에 직접 지정된 값
```

에서:

```
명시적인 width
```

로 변경했습니다.

Flexbox는 `flex-basis: auto` 상태에서 요소에 설정된 `width`를 기준으로 카드의 기본 크기를 결정할 수 있습니다.

결과적으로 기존:

```
flex-basis
    ↓
item 크기 계산
    ↓
group 크기 추론
    ↓
row intrinsic width 추론
```

구조에서:

```
item의 명시적인 width
        ↓
group의 max-content width
        ↓
row의 max-content width
```

구조로 변경되어 브라우저가 추론해야 하는 단계를 줄였습니다.

---

## Group과 Row에도 Intrinsic Hint 제공

카드뿐만 아니라 상위 Group과 Row도 내부 콘텐츠의 전체 크기를 명확하게 유지하도록 변경했습니다.

```css
.life-motion__group {
  display: flex;
  flex: none;

  width: max-content;
  min-width: max-content;
}

.life-motion__row {
  display: flex;
  flex-wrap: nowrap;

  width: max-content;
  min-width: max-content;
}
```

`width: max-content`를 통해:

> 자식 카드 전체가 필요로 하는 공간을 Group의 너비로 사용한다.

는 것을 명확히 했고,

```css
min-width: max-content;
```

를 통해 Flex Layout 과정에서도 콘텐츠가 필요로 하는 너비보다 작아지지 않도록 했습니다.

즉 WebKit이 부모의 Intrinsic Size를 계산할 때 참고할 수 있는 **명시적인 크기 힌트**를 제공한 것입니다.

이를 통해 Safari에서도:

```
Original Group
|--------------------------------|

[1] [3] [5] [7] [9]
                                [1] [3] [5] [7] [9]
                                |------ Clone Group ------
```

처럼 첫 번째 Group이 실제 카드 전체의 너비를 차지하고 난 뒤 Clone Group이 시작하도록 수정했습니다.

---

# 문제를 이해한 뒤 반복 구조까지 리팩토링

Safari 문제를 해결하는 과정에서 기존 코드의 데이터 구조까지 다시 이해하게 되었기 때문에, 단순히 CSS만 수정하고 끝내지 않았습니다.

기존에는:

```
createLifeMotionGroups
repeatItems
REPEAT_IN_GROUP
topGroupItems
bottomGroupItems
```

처럼 **행을 나누는 책임과 데이터를 반복하는 책임이 섞여 있었습니다.**

특히 `REPEAT_IN_GROUP`이 행의 수를 의미하는 것처럼 오해할 수 있었고, 실제 렌더링 단계에서도 Original + Clone Group을 이미 생성하고 있었기 때문에 데이터 반복 로직은 불필요했습니다.

이를:

```
LifeMotionScene
│
├─ createLifeMotionRows
│  ├─ topItems
│  └─ bottomItems
│
├─ LifeMotionRow(top)
│  ├─ Original Group
│  └─ Clone Group
│
└─ LifeMotionRow(bottom)
   ├─ Original Group
   └─ Clone Group
```

으로 정리했습니다.

`createLifeMotionRows`는:

> 어떤 데이터가 Top / Bottom Row에 속하는가

만 담당하고,

`LifeMotionRow`는:

> 하나의 Row에서 Original과 Clone을 어떻게 렌더링할 것인가

만 담당하도록 책임을 분리했습니다.

그 결과 더 이상 필요하지 않은:

```
REPEAT_IN_GROUP
repeatItems
repeatCount
```

도 제거했습니다.

---

# 결과와 배운 점

Safari에서 Original Group과 Clone Group이 겹쳐 렌더링되던 문제를 해결했고, Chrome과 Safari에서 동일한 레이아웃을 유지할 수 있게 됐습니다.

이번 문제를 해결하면서 단순히 Safari 전용 CSS를 추가하는 것이 아니라 다음 순서로 원인을 좁혔습니다.

```
GSAP / ScrollTrigger 상태 확인
        ↓
mask / filter / compositing 영향 배제
        ↓
DOM 반복 구조 확인
        ↓
Row / Group / Item의 실제 Box Size 확인
        ↓
flex-basis와 max-content의 관계 확인
        ↓
Blink / WebKit의 Intrinsic Sizing 결과 차이 확인
        ↓
명시적인 width 기반으로 크기 계산 단순화
        ↓
기존 반복 구조 리팩토링
```

이를 통해 `flex-basis`와 `width`가 최종 화면에서 비슷한 크기를 만들더라도 **브라우저의 레이아웃 계산 과정에서는 서로 다른 역할을 가진다는 점**을 이해했습니다.

특히 `max-content`처럼 부모가 자식의 크기를 기반으로 자신의 Intrinsic Size를 결정하는 구조에서 자식의 크기까지 `flex-basis`에만 의존하면, **부모의 Intrinsic Sizing과 자식의 Flexbox Sizing이 서로 의존하는 복합적인 계산 구조**가 만들어질 수 있습니다.

Chrome의 Blink에서는 의도한 결과가 나왔지만 Safari의 WebKit에서는 같은 구조에서 다른 크기 계산 결과가 나타났고, 이를 해결하기 위해 특정 브라우저의 추론 결과에 의존하기보다 **카드의 실제 `width`와 Group/Row의 콘텐츠 크기를 명시하는 방향으로 레이아웃을 재설계했습니다.**

또한 디버깅 과정에서 기존의 `row`, `group`, `repeat`, `clone`의 역할을 정확하게 이해하게 되었고, 문제 해결에서 끝내지 않고 **그 이해를 바탕으로 불필요한 반복 로직과 모호한 추상화까지 제거했습니다.**
