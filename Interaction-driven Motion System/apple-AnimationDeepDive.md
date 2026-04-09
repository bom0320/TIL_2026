# Apple-Animation-DeepDive

여기선 Apple이 애니메이션을 어떻게 사고하고 설계하는지를 파보도록 하자.

몇개를 구글링 해본 결과, 스크롤-동기화 장면은 일반적인 video `currenTime` 제어보다 이미지 시퀸스/캔버스 쪽이 더 정밀하다는 관점, 그리고 Hero/배경/아이콘/UI 요소를 서로 다른 기술로 섞는 하이브리드 접근은 꽤 타당한 관찰이였다.

## 1. 먼저 구분해야 할 것: Apple "웹 애니메이션" vs. Apple "플랫폼 애니메이션"

이 둘은 닮았지만 같은 이야기는 아니다.

### Apple 플랫폼 애니메이션

Apple 공식 문서와 WWDC 는 주로 ios/macOS/visionOS 앱 쪽의 모션 철학과 렌더링 사고방식을 설명한다. 여기서 Apple은 모션을 장식이 아니라, **상태 전달, 피드백, 방향 안내**를 위한 수단으로 정의하고, 부드럽고 유연한 움직임이 인터페이스를 이해하게 만든다고 말한다.
또 swiftUI 애니메이션 세션은 프레임마다 뷰가 어떻게 갱신되고, 어떤 값이 시간에 따라 보간되며, 애니메이션 문맥이 어떻게 전파되는지까지 설명한다.

[Apple Developer](https://developer.apple.com/kr/design/human-interface-guidelines/motion?utm_source=chatgpt.com)

### Apple 웹 애니메이션

반면 제품 페이지는 브라우저 환경에서 돌아가는 별도 문제다. 여기는 SwiftUI 가 아니라 스크롤, 레이어링, 캔버스, 미디어, 렌더링 비용의 문제이다. 다만 철학은 같다.
핵심은

- 장면은 이미 정돈돼 있고
- 모션은 의미를 전달하고
- 스크롤은 단순 이동이 아니라 상태 제어 입력이 된다.

## 2. Apple식 애니메이션의 본질은 "무엇을 움직일지" 보다, "무엇을 고정할지"에 있다.

Apple 스타일이 고급스럽게 보이는 이유는 화려한 기술보다 **기준점(ancor)** 이 강하기 때문이다. 큰 타이틀, 제품의 중심 실루엣, 주요 레이아웃 축은 쉽게 흔들리지 않는다. 대신 주변 레이어, 깊이감, 반응형 요소, 전환 요소만 움직인다. 그래서 사용자는 "움직임을 본다"기 보다, "질서가 유지된 상태에서 상태가 변한다"고 느낀다. 이건 Apple의 HIG에서 말하는 모션의 역할, 즉, 상태 전달과 방향 안내라는 원칙과 맞닿아 있다.

[Human Interface Guide](https://developer.apple.com/design/human-interface-guidelines?utm_source=chatgpt.com)

[Designing Fluid Interfaces](https://prod.liveshare.vsengsaas.visualstudio.com/join?B54029143AF9D9CBD106DD5A2BE86DB67C4D)

예를 들어, 내 Hero 페이지에 대입하면 이런 구조가 나온다.

- `BOM's PORTFOLIO` = 기준점
- 캐릭터 = 생명감 담당
- 스크롤 = 다음 상태로 넘기는 입력
- 나머지 요소 = 장면의 안정성 유지
  이건 단순히 취향 문제가 아니라, Apple식 장면 설꼐 문법에 가깝다.

## 3. Apple이 집착하는 건 "부드러움"이 아니라 "연속성"이다.

사람들이 Apple 모션을 보면 "부드럽다"고 말하는데, 더 정확히는 **끊김 없이 이어진다**가 맞다.

Apple 공식 성능 자료는 최대 120Hz 까지 갱신될 수 있고, 포그라운드 앱은 그 예산 안에서 입력 처리, 레이아웃, 렌더링을 끝내야 한다고 설명한다. 또 UI 애니메이션 hitch 자료는 스크롤/애니메이션이 왜 끊기는지, 렌더 루프에서 어떤 병목이 발생하는지를 분석하는 관점을 준다. 이건 단순히 "ease를 잘 쓰자" 수준이 아니라, **렌더링 파이프라인 전체를 의식하라**는 이야기이다.

[Improving app responsiveness](https://developer.apple.com/documentation/xcode/improving-app-responsiveness?utm_source=chatgpt.com)

[Explore UI animation hitches and the render loop](https://developer.apple.com/videos/play/tech-talks/10855/?utm_source=chatgpt.com)

[Demystify SwiftUI performance](https://developer.apple.com/videos/play/wwdc2023/10160/?utm_source=chatgpt.com)

그래서 Apple식 애니메이션을 파고 싶으면 질문을 이렇게 바꿔야 하다.

- 이 장면은 왜 예쁜가?
  - 절반만 맞아
- 이 장면은 **왜 끊기지 않고** 인지적으로 자연스러운가?
  - 이게 핵심!

## 4. 웹에서 Apple식 구현을 깊게 파면 결국 세 갈래이다.

앞에서도 설명했지만, 구조를 더 명확하게 잡아보면 이렇다.

### A. 스크롤-동기화 장면: 이미지 시퀸스 / 캔버스 계열

제품이 회전하거나 내부가 열리는 장면처럼, **스크롤 위치와 프레임이 거의 1:1로 묶여야 하는 경우** 엔 이미지 시퀸스 기반 사고가 강하다. 사용자의 스크롤이 입력값이 되고, 현재 프레임은 그 입력값의 직접적인 함수가 된다.

이 방식의 장점은 **정밀 제어** 이다. 비디오 seek의 지연이나 압축 artifacts 문제에서 좀 더 자유롭다. 이건 내가 찾아본 [커뮤니티](https://www.reddit.com/r/Frontend/comments/1cw2mk2/what_animation_library_does_the_apple_website_uses/) 에서의 대화에서의 핵심 주장과도 맞닿아 있었다. 다만 특정 내부 엔진명이나 압축 방식은 공식 검증 없이 단정하면 안 된다.

### B. Hero 계열: 자동 재생 비디오 / 제한된 모션

반대로 Hero처럼 사용자가 페이지에 진입했을 때 장면의 분위기만 잡아주면 되는 부분은 꼭 스크롤-프레임 동기화일 필요가 없다. 그래서 업계에서는 종종 비디오, SVG/Lottie, 혹은 가벼운 루프 모션을 쓴다. 커뮤에서도 hero hero는 직접적인 scroll-tied 가 아닐 수도 있다는 식으로 말한다.

### C. 인터랙티브 레이어링: sticky / fixed / masking / depth

이건 내 포폴에 가장 직접적으로 중요하다. Apple 제품 페이지는 물체 하나를 화면 중앙에 오래 붙들어 두고, 그 위나 뒤로 정보 레이어가 지나가게 만든다. 이런 구조는 "와, 애니메이션 화려하다" 보다 "이 제품을 차분히 설명받고 있다"는 느낌을 만든다. 공식 SwiftUI 세션에서도 scroll effect, custom transitions, visual effects 같은 주제를 다루는데, 본질은 레이어와 상태 변화의 관계를 설계하는 것이다.

## 5. 내가 정말로 파야 하는 건 "정체성" 자체보다, "정체성이 움직임 규칙으로 번역되는 방식"이다

Apple의 애니메이션에 파다보니 "정체성(Identity)"에 대한 이야기가 많이 나왔는데, 여기서 흔히 실수하는게 정체성을 추상어로만 붙드는 것이다. 예를 들면:

- 정밀함
- 차분함
- 미래적
- 부드러움
  이렇게 끝내면 별 도움이 안 된다.
  정체성은 **모션 규칙** 으로 바뀌어야 한다.

예를 들어 네 사이트가 주고 싶은 인상이:

### "정제된 프론트엔드 개발자"

라면 모션 규칙은 이렇게 번역된다.

- 큰 타이틀은 거의 안 움직임
- 전환은 빠르지 않고 정확함
- 한번에 여러 요소가 반응하지 않음
- 반복 루프가 티 나지 않음
- 다음 섹션은 "등장"이 아니라 "이어짐"

이 단계까지 내려와야 구현 얘기가 의미가 생긴다.
그래서 내가 고민해야하는 건 "정체성을 더 파야 하나"가 아니라

> **정체성을 장면 규칙과 상태 전환 규칙으로 번역했나**

이 질문이다

## 6. Apple식 자료를 심도 있게 보려면, 순서를 잘 잡아야한다.

막연히 "애플 애니메이션 영상 추천" 보다 아래 순서가 낫다.

### 1단계: 철학

가장 먼저 볼 건 Apple HIG의 Motion이다.
여기서 Apple은 모션의 목적을 장식이 아니라, 상태 전달, 피드백, 안내로 둔다. 이걸 안 보고 기술부터 파면 결국 "예쁜 GSAP" 만 남는다.

[Movement](https://developer.apple.com/kr/design/human-interface-guidelines/motion?utm_source=chatgpt.com)

### 2단계: 인터랙션 감각

그 다음은 WWDC 2018의 Designing Fluid Interfaces다.

[Designing Fluid Interfaces](https://developer.apple.com/videos/play/wwdc2018/803/?utm_source=chatgpt.com)

이건 지금 봐도 가치가 크다. "유체적 인터페이스"가 뭔지, 제스처와 모션이 왜 붙어 다니는지, 사용자가 왜 움직임을 '느끼는지'를 이해하는 데 좋다.

### 3단계: 애니메이션 엔진의 사고방식

그 다음은 Explore Swift animation이다.
웹 구현을 하더라도 이걸 보면 "상태가 바뀔 때 어떤 값이 보간되는가"라는 사고가 잡힌다. 라이브러리 문법보다 훨씬 중요하다.

[Explore SwiftUI animation](https://developer.apple.com/videos/play/wwdc2023/10156/?utm_source=chatgpt.com)

[Animation](https://developer.apple.com/documentation/swiftui/animations?utm_source=chatgpt.com)

### 4단계: 성능

그 다음은 Demystify SwiftUI performance, Explore UI animation hitches and the render loop, 그리고 responsiveness 문서다.
이 단계부터는 "애니매이션이 왜 예쁘지?" 가 아니라 "왜 어떤 모션은 갑자기 싼 티가 나지?"에 답하게 된다. 대부분 프레임 예산, 메인 스레드 부하, 레이아웃/페인트 비용 같은 현실 문제이다.

[Demystify SwiftUI performance](https://developer.apple.com/videos/play/wwdc2023/10160/?utm_source=chatgpt.com)

[Explore UI animation hitches and the render loop](https://developer.apple.com/videos/play/tech-talks/10855/?utm_source=chatgpt.com)

[Improving app responsiveness](https://developer.apple.com/documentation/xcode/improving-app-responsiveness?utm_source=chatgpt.com)

### 5단계: 최신 시각 효과

그 다음은 Create custom visual effects with SwiftUI같은 최신 세션이다.
이건 직접 웹에 쓰는 내용이 아니더라도, Apple이 요즘 시각 효과를 어떻게 레이어로 쪼개고, scroll effect나 custom transition을 어떻게 설계하는지 보는 데 좋다.

### 6단계: 접근성

마지막으로 Reduced Motion 기준을 꼭 같이 봐라.
Apple은 고급 모션을 만들면서도, deep simulation이나, parallax, blur 같은 효과가 멀미나 불편을 유발할 수 있으면 줄이거나 바꾸라고 한다. 진짜 상위권 구현은 멋진 것과 안전한 것을 동시에 챙긴다.

## 7. 깊게 보면 좋은 질문 7개

이건 그냥 자료 소비 말고, 진짜 분석 질문이다.

1. 이 장면에서 절대 안 움직이는 기준점은 무엇인가?
2. 움직이는 요소는 왜 그 요소여야 하지?
3. 이 모션은 상태를 설명하고 있나, 아니면 그냥 자식인가?
4. 이 전환은 "등장/퇴장"인가, 아니면 "이전 상태에서 다음 상태로 이어짐"인가?
5. 사용자의 입력과 애니메이션 진행률은 어느 정도 결합돼 있나?
6. 렌더링 비용이 큰 지점은 어디고, 왜 끊길 수 있나?
7. Reduced Motion이 켜졌을 때, 이 장면은 어떻게 단순화돼야 하나?
   이 7개 질문으로 Apple 제품 페이지를 뜯어보면 그냥 "예쁘다"가 아니라 **설계 문법**이 보이기 시작한다.

## 8. 내 포트폴리오 웹사이트에 연결하면

예를 들어 Hero 기준에서 Apple식 깊은 분석은 이렇게 번역된다.

### 장면 구성

- 타이틀 = 제품명 같은 앵커
- 캐릭터 = 제품 실루엣 같은 핵심 오브젝트
- 설명 텍스트 = 보조 정보 레이어
- 다음 섹션 = 스크롤 기반 스토리 확장

### 모션 철학

- 초기 장면은 거의 완성된 상태여야 함
- 캐릭터만 제한적으로 살아있어야 함
- 전환은 사라짐이 아니라 압축/인계여야 함
- 스크롤은 단순 페이지 이동이 아니라 상태 제어여야 함

## 내 상황에선, "구현법" 보다 "분석 프레임"이 먼저

왜냐면 구현은 내가 어느정도 해놨음 -> 문제는 **무엇을 구현할지 판단하는 기준** 이다.

Apple식 하이엔드 모션을 깊게 파고 싶으면

1. HIG Motion 읽기
2. Designing Fluid Interfaces 보기
3. Explore SwiftUI animation 보기
4. Demystify SwiftUI performance + animation hitches 보기
5. Apple 제품 페이지를 직접 뜯으면서

- 기준점
- 스크롤 결합도
- 레이어 구조
- 전환 논리
- 접근성 대응
  위와 같이 공부해보도록 하자
