# Uniform Resource Identifier(URL)

```scss
URI
 ├─ URL (Locator)
 └─ URN (Name)
```

URI(Uniform Resource Identifier)는 웹 상의 자원을 식별하기 위한 상위 개념이며, URL과 URN은 URI의 하위 개념이다. 좀 더 자세히 알아보도록 하자.

## 1. URI

웹 서버 리소스는 각자 이름을 갖고 있기 때문에, 클라이언트는 관심 있는 리소스를 지목할 수 있다.
서버 리소스 이름은 통합 자원 식별자(uniform resource identifier), 혹은 URI로 불린다. URI는 인터넷의 우편물 주소 같은 것으로, **정보 리소스를 고유하게 식별하고 위치를 지정**할 수 있다.
'죠의 컴퓨터 가게'의 웹 서버에 있는 이미지 리소스에 대한 URI라면 이런 식이다.

```
http://www.joes-hardware.com/specials/saw-blade.gif

```

1. **`http://`** : HTTP 프로토콜을 사용하라
2. **`www.joes-hardware.com`** : www.joes-hardware.com으로 이동하라
3. **`specials/saw-blade.gif`** : /specials/saw-blade.gif라고 불리는 리소스를 가져와라

![](https://media.discordapp.net/attachments/1308318171299840002/1464136915061182516/IMG_4736.jpg?ex=69745f42&is=69730dc2&hm=a003b58d80b2f09eccd2d0c420afda6395cb2ce9f25d66b8f113ec09cf463485&=&format=webp&width=2928&height=1204)
위의 그림과 같이 죠의 컴퓨터 가게 서버에 GIF 형식의 톱날 그림 리소스에 대한 URI가 HTTP 프로토콜에서 어떻게 해석되는지 보여준다. HTTP는 주어진 URI 객체를 찾아온다. URI에는 두 가지가 있는데, URL과 URN이라는 것이다. 이 두 종류의 자원 식별자에 대해 살펴보도록 하자

## 2. URL

통합 자원 지시자(uniform resource locator, URL) 는 리소스 식별자의 가장 흔한 형태이다. URL은 특정 서버의 한 리소스에 대한 **구체적인 위치를 서술** 한다. URL 은 리소스가 정확히 어디에 있고 어떻게 접근할 수 있는지 분명히 알려준다. 위의 그림에선 어떻게 URL로 리소스이 정확한 위치와 접근 방법을 표현하는지 보여준다. 아래 표는 URL의 몇 가지 예이다.

| URL                                                           | 설명                                                                           |
| ------------------------------------------------------------- | ------------------------------------------------------------------------------ |
| `http://www.oreilly.com/index.html`                           | 오라일리 출판사 홈페이지의 URL                                                 |
| `http://www.yahoo.com/images/logo.gif`                        | 야후! 웹 사이트 로고의 URL                                                     |
| `http://www.joes-hardware.com/inventory-check.cgi?item=12731` | 물품 **#12731**의 재고가 있는지 확인하는 프로그램에 대한 URL                   |
| `ftp://joe:tools4u@ftp.joes-hardware.com/locking-pliers.gif`  | 비밀번호로 보호되는 FTP를 통해 `locking-pliers.gif` 이미지 파일에 접근하는 URL |

대부분의 URL은 세 부분으로 이루어진 **표준 포맷** 을 따른다.

- URL의 첫번째 부분은 **스킴(scheme)** 이라고 불리는데, 리소스에 접근하기 위해 사용되는 프로토콜을 서술한다. 보통 **HTTP 프로토콜(http://)** 이다.
- 두 번째 부분은 서버의 인터넷 주소를 제공한다. (예: ww.joes-hardware.com).
- 마지막은 웹 서버의 리소스를 가리킨다. (예: /specials/saw-blade.gif)

그리고 오늘날 **대부분의 URI는 URL**이다.

## 3. URN

URI의 두 번째 종류는 유니폼 리소스 이름(uniform resource name, URN)이다. URN은 콘텐츠를 이루는 한 리소스에 대해, 그 리소스의 위치에 영향 받지 않는 유일무이한 이름 역할을 한다. 이 위치 독립적인 URN은 리소스를 여기저기로 옮기더라도 문제 없이 동작한다. 리소스가 그 이름을 변하지 않게 유지하는 한, 여러 종류의 네트워크 접속 프로토콜로 접근해도 문제 없다.

예를 들어, 다음의 URN은 인터넷 표준 문서 'RFC 2141'가 어디에 있거나 상관없이(심지어 여러 군데에 복사되었더라도) 그것을 지칭하기 위해 사용할 수 있다.

```
urn:ietf:rfc:2141
```

URN은 여전히 실험 중인 상태고, 아직 널리 채택되지 않았다. 효율적인 동작을 위해 URN은 리소스 위치를 분석하기 위한 인프라 지원이 필요한데, 그러한 인프라가 부재하기에 URN 채택이 더 늦춰지고 있다. 그러나 URN의 전망은 분명 밝다.
