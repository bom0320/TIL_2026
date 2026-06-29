# MVC란

## 한줄 정의

MVC는 애플리케이션을 `Model`, `View`, `Controller`로 나누어 각 영역의 책임을 분리하는 설계 패턴(디자인 패턴)이다.

## 왜 사용하는가?

데이터 처리, 화면 출력, 사용자 요청 로직이 한 곳에 섞이면 코드 수정과 유지보수가 어려워진다.
MVC는 역할을 분리하여 코드 구조를 명확하게 만들기 위해 사용한다.

## 핵심 개념

### Model

**데이터와 비즈니스 로직을 담당**한다.

예시:

- 사용자 데이터 조회
- 게시글 저장
- 주문 금액 계산
- 데이터베이스 접근

### View

**사용자에게 보여지는 화면을 담당**한다.
Model에서 처리한 데이터를 화면에 출력한다.

### Controller

사용자의 요청을 받아 적절한 Model을 호출하고, 그 결과를 View에 전달한다.

## 동작 흐름

![](https://mblogthumb-phinf.pstatic.net/MjAxODEyMDVfMTA1/MDAxNTQ0MDE3MDQ3OTE3.g5uFx4CwmDcihBhk4Yc3DXwMrLE1FtQkRc5dQ3awkZkg.vNLkkt-Xyalvg4AwO65vf3b3bvwVaR0hnM9c639Mftog.JPEG.jhc9639/%25EB%25B8%2594%25EB%25A1%259C%25EA%25B7%25B81.jpg?type=w966)

사용자 요청 -> Controller가 요청을 받음 -> Controller가 Model호출 -> Model이 데이터 처리 -> Controller가 결과를 View에 전달 -> View가 화면 출력

### 장점

- 역할과 책임이 명확해진다.
- 유지보수가 쉬워진다.
- 화면과 비즈니스 로직을 분리할 수 있다.
- 여러 개발자가 역할을 나누어 작업하기 좋다.

### 주의할 점

- 작은 프로젝트에 적용하면 구조가 과도하게 복잡해질 수 있다.
- Controller에 로직이 몰리면 비대해질 수 있다.
- 프레임워크마다 MVC 구현 방식이 다를 수 있다.

## 면접 답변

MVC는 애플리케이션을 Model, View, Controller로 나누는 설계패턴입니다. Model은 데이터와 비즈니스 로직, View는 화면, Controller는 사용자 요청과 전체 흐름을 담당합니다.
각 영역의 책임을 분리하여 유지보수성과 확장성을 높이기 위해 사용합니다.

### 관련 개념

- 디자인 패턴
- 관심사의 분리
- MVVM
- 계층형 아키텍처
