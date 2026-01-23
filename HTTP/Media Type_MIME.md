# 미디어 타입

인터넷은 수천 가지 데이터 타입을 다루기 때문에, HTTP는 웹에서 전송되는 객체 각각에 신중하게 MIME 타입이라는 데이터 포맷 라벨을 붙인다.

MIME(Multipurpose Internet Mail Extensions, 다목적 인터넷 메일 확장)은 원래 각기 다른 전자메일 시스템 사이에서 메시지가 오갈 때 겪는 문제점을 해결하기 위해 설계되었다. MIME은 이메일에서 워낙 잘 모당작했기 때문에 HTTP에서도 멀티미디어 콘텐츠를 기술하고 라벨을 붙이기 위해 채택되었다.

즉, 쉽게 말하면 이 데이터가 어떤 종류의 데이터인지 알려주는 라벨이라고 할 수 있다. 예를 들면 서버가 클라이언트에게 "이건 HTML이야, 이건 JSON이야.."이런식으로 말하는 것이다.

웹 서버는 모든 HTT 객체 데이터에 MIME 타입을 붙인다.

```bash
Content-type: image/jpeg
Content-length: 12984
```

웹 브라우저는 서버로부터 객체를 돌려받을 때, 다룰 수 있는 객체인지 MIME 타입을 통해 확인한다. 대부분의 웹 브라우저는 잘 알려진 객체 타입 수백 가지를 다룰 수 있다. 이미지 파일을 보여주고, HTML 파일을 분석하거나 포맷팅하고, 오디오 파일을 컴퓨터의 스피커를 통해 재생하고, 특별한 포맷의 파일을 다루기 위해 외부 플러그인 소프트웨어를 실행한다.

MIME은 **사선(/)으로 구분된 주 타입(primary object type)**과 **부 타입(specific subtype)**으로 이루어진 **문자열 라벨**이다. 예를 들면 다음과 같다.

- HTML로 작성된 텍스트 문서는 text/html 라벨이 부는다.
- plain ASCII 텍스트 문서는 text/plain 라벨이 붙는다
- JPEG 이미지는 image/jpeg가 붙는다.
- GIF 이미지는 image/gif 가 된다.
- 애플 퀵타입 동영상은 video/quicktime이 붙는다.
- 마이크로소프트 파워포인트 프레젠테이션은 application/vnd.ms-powerpoint가 붙는다.

즉, 수백가지의 잘 알려진 MIME 타입과, 급다 더 많은 실험용 혹은 특정용도의 MIME 타입이 존재한다. MIME 전체 목록은 구글링 하면 바로 찾을 수 있다.

### + Content-Type은 MIME(미디어) 타입을 명시하는 HTTP 헤더이다

ex)

```http
Content-Type: application/json
```
