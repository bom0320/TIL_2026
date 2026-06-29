# Concept Notes

프론트엔드 개발에 필요한 핵심 개념을 주제별로 정리한 공간입니다.

단순히 개념의 정의만 기록하는 것이 아니라, 해당 개념이 필요한 이유와 동작 원리, 실제 예시, 주의할 점까지 이해하는 것을 목적으로 합니다.

## 문서 작성 형식

각 개념 노트는 다음 순서로 작성합니다.

1. 한 줄 정의
2. 왜 사용하는가?
3. 핵심 개념
4. 동작 흐름
5. 예시
6. 장점
7. 주의할 점
8. 면접 답변
9. 관련 개념

---

## Design Patterns

개발 과정에서 반복적으로 발생하는 설계 문제를 해결하기 위한 소프트웨어 디자인 패턴을 정리합니다.

### 포함되는 개념

- [MVC](./Design-Patterns/mvc.md)
- Observer Pattern
- Factory Pattern
- Strategy Pattern
- Singleton Pattern
- Adapter Pattern

---

## Architecture

애플리케이션 전체의 구조와 책임 분리, 코드 구성 및 모듈 간 관계를 설계하는 방식에 관한 개념을 정리합니다.

### 포함되는 개념

- 계층형 아키텍처
- 클린 아키텍처
- FSD
- 모놀리식 아키텍처와 마이크로서비스 아키텍처
- 헥사고날 아키텍처

---

## Network

클라이언트와 서버의 통신 방식, HTTP 프로토콜, 인증 및 API 설계에 관한 개념을 정리합니다.

### 포함되는 개념

- [REST API](./Network/rest-api.md)
- HTTP
- HTTPS
- CORS
- 쿠키와 세션
- JWT
- HTTP 상태 코드

---

## Framework

프레임워크의 역할과 특징, 웹 애플리케이션의 렌더링 및 라우팅 방식에 관한 개념을 정리합니다.

### 포함되는 개념

- [Next.js를 사용하는 이유](./Framework/why-nextjs.md)
- 라이브러리와 프레임워크의 차이
- CSR, SSR, SSG, ISR
- App Router
- 서버 컴포넌트와 클라이언트 컴포넌트

---

## Browser

브라우저가 웹 문서를 해석하고 화면을 그리는 과정과 브라우저 내부 동작에 관한 개념을 정리합니다.

### 포함되는 개념

- 브라우저 렌더링 과정
- DOM과 CSSOM
- 이벤트 루프
- 리플로우와 리페인트
- 웹 스토리지

---

## JavaScript

JavaScript 언어의 동작 원리와 실행 환경, 객체 및 타입 시스템에 관한 개념을 정리합니다.

### 포함되는 개념

- 실행 컨텍스트
- 클로저
- 호이스팅
- 프로토타입
- `this`
- 타입 변환

---

## React

React의 렌더링 원리와 상태 관리, 컴포넌트 동작 방식에 관한 개념을 정리합니다.

### 포함되는 개념

- Virtual DOM
- 상태와 렌더링
- `useEffect`
- 제어 컴포넌트와 비제어 컴포넌트
- 서버 상태와 클라이언트 상태

---

## Development Process

기능 개발 이후 테스트와 검수, 협업, 배포까지 이어지는 개발 업무 과정에 관한 개념을 정리합니다.

### 포함되는 개념

- [QA](./Development-Process/qa.md)
- [배포 프로세스](./Development-Process/deployment-process.md)
- CI/CD
- 코드 리뷰
- 테스트
- 애자일

---

## 폴더 구조

```text
Concept/
├─ Design-Patterns/
├─ Architecture/
├─ Network/
├─ Framework/
├─ Browser/
├─ JavaScript/
├─ React/
├─ Development-Process/
└─ README.md
```

새로운 개념을 학습할 때는 개념의 성격에 맞는 폴더에 Markdown 파일을 추가합니다.

```text
Concept/Design-Patterns/mvc.md
Concept/Architecture/fsd.md
Concept/Network/rest-api.md
Concept/Framework/why-nextjs.md
Concept/Browser/browser-rendering.md
Concept/JavaScript/execution-context.md
Concept/React/virtual-dom.md
Concept/Development-Process/ci-cd.md
```

## 분류 기준

- 반복되는 설계 문제의 해결 방식은 `Design-Patterns`
- 애플리케이션 전체 구조와 모듈 구성 방식은 `Architecture`
- 클라이언트와 서버의 통신은 `Network`
- 프레임워크의 기능과 사용 방식은 `Framework`
- 브라우저 내부 동작은 `Browser`
- JavaScript 언어의 동작 원리는 `JavaScript`
- React의 렌더링과 컴포넌트 구조는 `React`
- 개발, 테스트, 협업, 배포 과정은 `Development-Process`
