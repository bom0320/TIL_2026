# Amplitude 핵심 정리

## Amplitude가 정확히 뭐야?

Amplitude는 **사용자가 웹사이트 안에서 어떤 행동을 했는지 기록하고 분석하는 도구** 이다.

예를 들면 내 포트폴리오에서 사용자가

```
사이트에 들어옴
→ 프로젝트 섹션까지 스크롤함
→ Washer 프로젝트를 클릭함
→ GitHub 링크를 클릭함
→ 이메일 버튼을 클릭함
```

이런 행동을 Amplitude 서버로 보내면, 나중에 Amplitude 화면에서 다음처럼 볼 수 있다.

```
방문자 100명 중
프로젝트 섹션 도달: 58명
프로젝트 클릭: 24명
GitHub 클릭: 9명
연락 버튼 클릭: 3명
```

즉, Amplitude는 단순 방문자 카운터가 아니라 **사용자 행동의 흐름을 분석하는 제품 분석 도구** 이다. SDK가 보낸 이벤트 데이터를 바탕으로 퍼널, 행동 차트, 사용자 그룹 등을 만들 수 있다.

### 추가로 내가 설치한 건 Amplitude 자체가 아님

터미널에서 실행한 명령어는

```bash
pnpm add @amplitude/analytics-browser
```

이 명령어로 설치한 것은 **Amplitude Browser SDK** 이다.

**SDK가 뭔데?**

> 외부 서비스를 내 프로젝트에서 사용할 수 있도록 준비된 코드 묶음

Amplitude 웹사이트에 직접 요청을 만드는 코드는 내가 처음부터 작성하려면 Amplitude 서버 주소, 이벤트 전송 형식, 사용자 식별, 세션 관리 등을 직접 처리해야한다. 이걸 SDK가 이 작업을 대신 해준다.

그래서 개발자는 단순히

```ts
amplitude.init(...)
amplitude.track(...)
```

만 호출하면 된다.

### 전체 흐름부터 이해하기

지금 작성한 코드는 다음 흐름으로 움직인다.

```
.env.local
Amplitude 프로젝트 API Key 보관
        ↓
amplitude.ts
SDK 초기화 함수와 이벤트 전송 함수 정의
        ↓
AmplitudeProvider
사이트가 실행될 때 초기화 함수 호출
        ↓
layout.tsx
Provider를 전체 사이트에 적용
        ↓
사용자 행동 발생
        ↓
trackAmplitudeEvent 호출
        ↓
Amplitude 서버로 이벤트 전송
        ↓
Amplitude 대시보드에서 분석
```

## 주요 용어

### API Key

이벤트를 어느 Amplitude 프로젝트로 전송할지 구분하는 값

```env
NEXT_PUBLIC_AMPLITUDE_API_KEY=발급받은_API_KEY
```

`NEXT_PUBLIC_` 이 붙은 이유는 Amplitude SDK 가 브라우저에서 실행되기 때문이ㅏㄷ.

Secret Key와는 다르며, Secret Key는 프론트엔드에 넣으면 안 된다.

### SDK 초기화

```ts
amplitude.init(apiKey, undefined, {
  autocapture: false,
});
```

각 값의 의미는 다음과 같다.

- **apiKey**
  - 어느 Amplitude 프로젝트와 연결할지 지정
- **undefined**
  - 로그인 사용자 ID를 별도로 지정하지 않음
- **autocapture: false**
  - 자동 수집을 끄고 직접 정의한 이벤트만 전송

### 이벤트 전송

```ts
amplitude.track("project_clicked", {
  project_name: "washer-admin",
});
```

사용자가 수행한 행동과 해당 행도으이 추가 정보를 Amplitude로 전송한다.

## 이벤트와 이벤트 속성

### 이벤트

사용자가 무엇을 했는지 나타낸다.

```
portfolio_viewed
section_viewed
project_clicked
external_link_clicked
```

### 이벤트 속성

그 행동이 **어떤 대상과 조건에서 발생했는지** 나타낸다.

```ts
{
  project_name: "washer-admin",
  location: "project_section",
  viewport_width: 390,
}
```

프로젝트마다 별도 이벤트를 만드는 것보다 하나의 이벤트와 속성으로 구분하는 편이 관리하기 쉽다.

```
비추천
washer_clicked
hyoit_clicked
nova_clicked

추천
project_clicked
+ project_name 속성
```

## `amplitude.ts` 역할

```ts
import * as amplitude from "@amplitude/analytics-browser";
```

설치한 SDK 를 가져온다.

```ts
const apiKey = process.env.NEXT_PUBLIC_AMPLITUDE_API_KEY;
```

환경변수에 저장한 API Key를 읽는다.

```ts
let isInitialized = false;
```

SDK가 중복 초기화되는 것을 방지한다.

```ts
if (typeof window === "undefined") {
  return;
}
```

서버가 아닌 브라우저에서만 실행되도록 막는다.

```ts
amplitude.init(apiKey.undefined, {
  autocapture: false,
});
```

Amplitude SDK를 사용할 준비를 한다.

```ts
amplitude.track(eventName, properties);
```

## Provider 역할

```ts
"use client";
```

Amplitude는 `window`, `document` 같은 브라우저 기능을 사용하므로 Client Component에서 실행해야 한다.

```ts
useEffect(() => {
  initAmplitude();
});
```

사이트가 브라우저에 처음 실행됐을 때 SDK를 초기화한다.

```ts
<AmplitudeProvider>{children}</AmplitudeProvider>
```

루트 레이아웃에서 전체 포트폴리오를 감싸 사이트 전체에서 Amplitude를 사용할 수 있게 한다.

### 현재 구현한 범위

```
Amplitude 계정 및 프로젝트 생성
API Key 발급
Browser SDK 설치
환경변수 연결
SDK 초기화 함수 작성
루트 Provider 연결
portfolio_viewed 이벤트 전송
Live Events 수신 확인
```

아직 추가할 항목은 다음과 같다.

```
섹션 도달 이벤트
프로젝트 클릭 이벤트
이력서·GitHub 클릭 이벤트
연락 수단 클릭 이벤트
캐러셀 조작 이벤트
퍼널과 차트 구성
```

### 포트폴리오 핵심 이벤트

```ts
portfolio_viewed;
section_viewed;
project_viewed;
project_clicked;
external_link_clicked;
carousel_interacted;
```

주요 속성:

```
section_name
project_name
project_order
link_type
location
device_type
viewport_width
referrer
```

## 자동 수집과 직접 수집

### 자동 수집

Amplitude가 기본 행동을 자동으로 기록한다.

```
페이지뷰
세션
클릭
파일 다운로드
유입 정보
기기 정보
```

### 직접 수집

서비스에서 의미 있는 행동을 개발자가 직접 정의한다.

```
프로젝트 섹션 도달
특정 프로젝트 클릭
이력서 클릭
연락 버튼 클릭
```

포트폴리오에서는 기본 데이터는 자동 수집하고, 중요한 사용자 흐름은 커스텀 이벤트로 직접 추적하는 방식이 적합하다.

## 면접 답변

> Amplitude는 사용자가 서비스 안에서 어떤 행동을 했는지 이벤트 단위로 수집하고 분석하는 제품 분석 도구입니다. 포트폴리오에서는 Browser SDK를 설치하고, 루트 Provider에서 SDK를 한 번 초기화했습니다. 이후 공통 이벤트 전송 함수를 만들어 이벤트 이름과 속성을 Amplitude로 보내도록 구성했습니다. 현재는 포트폴리오 방문 이벤트가 정상적으로 수집되는 것을 확인했고, 이후 섹션 도달과 프로젝트 클릭, 외부 링크 클릭 이벤트를 추가해 사용자 흐름을 분석할 예정입니다.
