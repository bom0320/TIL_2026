# API 요청 실패와 Proxy 경로 예외 처리

## 핵심 흐름

Washer Admin 개발 중 프론트엔드에서 백엔드 API를 호출했을 때 데이터가 정상적으로 받아와지지 않는 문제가 있었다.

초기에는 API 요청 경로, 인증서, 프록시 설정 문제를 의심했다.
하지만 확인해보니 Next.js proxy에서 페이지 접근 제어와 API 요청 흐름이 함께 처리될 수 있는 구조가 문제에 영향을 줄 수 있었다.

따라서 `/api`, `/sign-in`, 정적 리소스 경로는 proxy 인증 검사 대상에서 제외하고, accessToken이 없는 일반 페이지 요청만 `/sign-in`으로 redirect하도록 수정했다.

---

## 문제 상황

프론트엔드에서 API를 호출했을 때 관리자 페이지에 필요한 데이터가 정상적으로 표시되지 않았다.

예를 들어 고장 신고, 예약, 기기 목록 같은 데이터는 React Query를 통해 요청하고 있었지만, 요청 흐름이 정상적으로 이어지지 않아 서버 연동이 실패한 것처럼 보였다.

초기에는 다음 문제를 의심했다.

```txt
- API endpoint 경로 문제
- Axios baseURL 설정 문제
- 프록시 설정 문제
- 인증서 검증 문제
- React Query queryFn 문제
```

하지만 API 함수와 query hook 구조를 확인했을 때, 요청 로직 자체보다는 proxy의 인증 처리 흐름과 API 요청 경로가 분리되지 않은 것이 문제에 영향을 줄 수 있었다.

---

## 원인 분석

Next.js proxy는 요청이 들어올 때 pathname과 cookie를 확인한다.

```ts
const { pathname } = request.nextUrl;
const accessToken = request.cookies.get(COOKIE_KEYS.ACCESS_TOKEN)?.value;
```

이때 모든 요청을 같은 방식으로 검사하면 문제가 생길 수 있다.

예를 들어 API 요청도 일반 페이지 요청처럼 accessToken 검사 대상이 되면, API 응답을 받아야 하는 요청이 `/sign-in`으로 redirect될 수 있다.

```txt
/api/v2/admin/malfunction-reports
  ↓
proxy 인증 검사
  ↓
accessToken 없음
  ↓
/sign-in redirect
```

이 경우 프론트엔드에서는 JSON API 응답을 기대하지만, 실제로는 redirect 응답이 돌아올 수 있다.
결과적으로 React Query에서는 요청 실패로 처리되고, 화면에는 데이터가 표시되지 않는다.

즉, 문제의 핵심은 다음과 같다.

```txt
페이지 보호 로직과 API 요청 흐름이 명확히 분리되어야 한다.
```

---

## 해결 코드

수정한 proxy 코드는 다음과 같다.

```ts
import { type NextRequest, NextResponse } from "next/server";
import { COOKIE_KEYS } from "@/shared/constants/cookies";

export async function proxy(request: NextRequest) {
  const { pathname } = request.nextUrl;
  const accessToken = request.cookies.get(COOKIE_KEYS.ACCESS_TOKEN)?.value;

  if (pathname.startsWith("/api/") || pathname === "/sign-in") {
    return NextResponse.next();
  }

  if (!accessToken) {
    return NextResponse.redirect(new URL("/sign-in", request.url));
  }

  return NextResponse.next();
}

export const config = {
  matcher: [
    "/((?!api|_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp|ico)$).*)",
  ],
};
```

---

## 코드 흐름

proxy의 동작 흐름은 다음과 같다.

```txt
요청 발생
  ↓
pathname 확인
  ↓
/api/* 요청인지 확인
  ↓
/sign-in 요청인지 확인
  ↓
위 조건에 해당하면 그대로 통과
  ↓
그 외 페이지 요청은 accessToken 확인
  ↓
accessToken이 없으면 /sign-in으로 redirect
  ↓
accessToken이 있으면 통과
```

즉, `/api` 요청은 인증 redirect 대상이 아니다.
인증이 필요한 것은 관리자 페이지 접근이고, API 요청 자체는 proxy에서 로그인 페이지로 redirect하지 않도록 분리한다.

---

## /api 경로 예외 처리

가장 중요한 부분은 이 조건이다.

```ts
if (pathname.startsWith("/api/") || pathname === "/sign-in") {
  return NextResponse.next();
}
```

이 코드는 `/api/*` 요청과 `/sign-in` 페이지 요청을 proxy 인증 검사에서 제외한다.

```txt
/api/*
→ API 요청이므로 그대로 통과

/sign-in
→ 로그인 페이지이므로 그대로 통과
```

만약 `/sign-in`을 예외 처리하지 않으면, 로그인 페이지에 접근할 때도 accessToken이 없다는 이유로 다시 `/sign-in`으로 redirect되는 흐름이 생길 수 있다.

만약 `/api/*`를 예외 처리하지 않으면, API 요청이 로그인 페이지 redirect로 바뀌어 데이터 요청이 실패할 수 있다.

---

## accessToken 기준 페이지 보호

API 요청과 로그인 페이지를 제외한 나머지 요청은 accessToken을 기준으로 보호한다.

```ts
if (!accessToken) {
  return NextResponse.redirect(new URL("/sign-in", request.url));
}
```

이 조건은 로그인하지 않은 사용자가 관리자 페이지에 접근하는 것을 막는다.

```txt
accessToken 없음
  ↓
보호된 페이지 접근
  ↓
/sign-in으로 이동
```

반대로 accessToken이 있으면 요청을 통과시킨다.

```ts
return NextResponse.next();
```

즉, proxy의 역할은 API 호출을 막는 것이 아니라, 인증되지 않은 사용자의 페이지 접근을 제한하는 것이다.

---

## matcher 설정

matcher는 proxy가 적용될 경로를 제한한다.

```ts
export const config = {
  matcher: [
    "/((?!api|_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp|ico)$).*)",
  ],
};
```

이 설정은 다음 경로를 proxy 대상에서 제외한다.

```txt
/api
/_next/static
/_next/image
favicon.ico
이미지 파일(svg, png, jpg, jpeg, gif, webp, ico)
```

정적 리소스나 이미지 파일까지 인증 검사 대상이 되면, 페이지 렌더링에 필요한 파일 요청이 redirect될 수 있다.

따라서 matcher에서 정적 파일과 Next 내부 리소스를 제외한다.

---

## 이중 예외 처리 구조

현재 코드는 `/api` 요청을 두 번 보호한다.

첫 번째는 함수 내부 조건이다.

```ts
if (pathname.startsWith("/api/") || pathname === "/sign-in") {
  return NextResponse.next();
}
```

두 번째는 matcher 설정이다.

```ts
matcher: [
  "/((?!api|_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp|ico)$).*)",
],
```

즉, `/api` 요청은 matcher 단계에서 proxy 대상에서 제외되고, 혹시 proxy 함수에 들어오더라도 함수 내부에서 다시 통과된다.

이 구조는 API 요청이 인증 redirect 흐름에 섞이지 않도록 방어적으로 처리한 방식이다.

---

## 해결 결과

수정 이후 API 요청과 페이지 보호 로직이 분리되었다.

```txt
API 요청
→ proxy redirect 대상에서 제외
→ 정상적으로 서버 데이터 요청

페이지 요청
→ accessToken 확인
→ 없으면 /sign-in redirect
→ 있으면 페이지 접근 허용
```

이후 고장 신고, 예약, 기기 목록 등 관리자 페이지에서 필요한 데이터를 React Query를 통해 정상적으로 받아올 수 있게 되었다.

---

## 정리

이 문제는 단순히 API 함수나 React Query 코드의 문제가 아니라, Next.js proxy에서 인증 보호 대상 경로를 명확히 분리하지 않으면 발생할 수 있는 요청 흐름 문제이다.

핵심은 다음과 같다.

```txt
API 요청 실패 원인을 endpoint, Axios, React Query, proxy 흐름으로 나누어 확인했다.
Next proxy가 API 요청까지 redirect 대상으로 처리하지 않도록 /api 경로를 예외 처리했다.
로그인 페이지인 /sign-in도 redirect 대상에서 제외했다.
정적 리소스와 이미지 파일도 matcher에서 제외했다.
accessToken이 없는 일반 페이지 요청만 /sign-in으로 redirect하도록 수정했다.
결과적으로 API 요청 흐름과 페이지 보호 로직을 분리했다.
```

---

## 포트폴리오용 요약 문장

API 요청이 정상적으로 처리되지 않는 문제가 발생해 endpoint, Axios, React Query, proxy 흐름을 나누어 원인을 확인했다.
Next.js proxy에서 API 요청과 페이지 보호 로직이 함께 처리될 수 있는 구조를 정리하고, `/api`, `/sign-in`, 정적 리소스 경로를 인증 redirect 대상에서 제외했다.
이후 accessToken이 없는 일반 페이지 요청만 `/sign-in`으로 이동하도록 수정해 API 요청 흐름과 인증 보호 로직을 분리했다.
