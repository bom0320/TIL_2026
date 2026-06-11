# Self-Signed 인증서로 인한 Next.js Rewrites 프록시 요청 실패

## 문제 상황

Washer Admin은 DataGSM OAuth 로그인을 통해 accessToken을 발급받고, 관리자 페이지에서 사용자, 예약, 고장 신고 데이터를 조회하는 구조로 개발되었다.

프론트엔드에서는 React Query를 사용해 서버 상태를 관리하고 있었으며, API 요청은 Next.js rewrites를 통해 백엔드 서버로 프록시되고 있었다.

```txt
Client
  ↓
React Query
  ↓
/api/*
  ↓
Next.js Rewrites
  ↓
Backend Server
```

개발 도중 관리자 페이지에 진입하면 데이터가 정상적으로 표시되지 않는 문제가 발생했다.

특히 다음 API들이 모두 실패하고 있었다.

```txt
GET /v2/admin/users
GET /v2/admin/reservations
GET /v2/admin/malfunction-reports
```

화면에서는 데이터가 비어 있었고 React Query도 정상적인 응답을 받지 못하고 있었다.

---

## 초기 가설

처음에는 일반적인 API 연동 문제를 의심했다.

### 1. API Endpoint 오류

```ts
userUrl.getUsers();
reservationUrl.getReservations();
reportUrl.getMalfunctionReports();
```

잘못된 경로를 호출하고 있는지 확인했다.

---

### 2. Axios Base URL 문제

```ts
axiosInstance;
```

또는

```ts
get();
post();
patch();
```

wrapper 설정이 잘못된 것은 아닌지 확인했다.

---

### 3. React Query 문제

```ts
useGetUsers();
useGetReservations();
useGetMalfunctionReports();
```

queryFn이 정상적으로 실행되는지 확인했다.

---

### 4. Next.js Rewrites 문제

```ts
rewrites();
```

설정이 올바르게 적용되고 있는지 확인했다.

---

## 로그 확인

브라우저와 서버 로그를 확인하던 중 공통적으로 다음 에러가 발생하는 것을 확인했다.

```txt
DEPTH_ZERO_SELF_SIGNED_CERT
self-signed certificate
```

실제 로그

```txt
Failed to proxy https://gsmsv-1.yujun.kr:26528/v2/admin/users

Error:
DEPTH_ZERO_SELF_SIGNED_CERT
```

```txt
Failed to proxy https://gsmsv-1.yujun.kr:26528/v2/admin/reservations

Error:
DEPTH_ZERO_SELF_SIGNED_CERT
```

```txt
Failed to proxy https://gsmsv-1.yujun.kr:26528/v2/admin/malfunction-reports

Error:
DEPTH_ZERO_SELF_SIGNED_CERT
```

모든 API 요청이 동일한 에러로 실패하고 있었다.

---

## 원인 분석

백엔드 개발 서버는 self-signed 인증서를 사용하고 있었다.

일반적인 HTTPS 통신은 다음과 같다.

```txt
Client
  ↓
인증서 확인
  ↓
공인 CA 인증서
  ↓
신뢰
  ↓
통신
```

하지만 현재 서버는 공인 인증기관(CA)이 발급한 인증서가 아닌 self-signed 인증서를 사용하고 있었다.

```txt
Client
  ↓
인증서 확인
  ↓
Self-Signed 인증서
  ↓
신뢰 불가
  ↓
연결 차단
```

Node.js는 기본적으로 self-signed 인증서를 신뢰하지 않는다.

따라서 rewrites가 백엔드 서버로 요청을 전달하는 과정에서 TLS 인증서 검증에 실패하고 있었다.

실제 요청 흐름은 다음과 같았다.

```txt
Browser
  ↓
Next.js Server
  ↓
Rewrites Proxy
  ↓
Backend Server
  ↓
TLS 인증서 검증 실패
```

즉 API 코드 자체는 정상적으로 실행되고 있었지만, 네트워크 계층에서 요청이 차단되고 있었다.

---

## 시도한 해결 방법

### NODE_TLS_REJECT_UNAUTHORIZED

가장 먼저 개발 환경에서 인증서 검증을 비활성화하는 방법을 시도했다.

```json
{
  "scripts": {
    "dev:insecure": "NODE_TLS_REJECT_UNAUTHORIZED=0 next dev"
  }
}
```

실행

```bash
pnpm dev:insecure
```

실행 로그

```txt
Warning:
NODE_TLS_REJECT_UNAUTHORIZED=0
```

환경 변수는 정상적으로 적용되었다.

하지만 API 요청은 여전히 실패했다.

```txt
DEPTH_ZERO_SELF_SIGNED_CERT
```

동일한 에러가 계속 발생했다.

---

## 추가 분석

팀원 환경에서는 동일한 설정으로 정상 동작하고 있었지만, 내 환경에서는 지속적으로 실패했다.

이 과정에서 다음 가능성을 검토했다.

```txt
Node.js 버전 차이
macOS 환경 차이
Next.js 버전 차이
Turbopack 영향
Rewrites 내부 동작 차이
```

특히 Next.js 16 + Rewrites + Self-Signed 인증서 조합에서 인증서 검증 우회가 정상적으로 적용되지 않는 사례들을 확인했다.

이를 통해 단순한 Axios 설정 문제가 아니라 Rewrites 프록시 계층에서 발생하는 TLS 문제라는 점을 확인할 수 있었다.

---

## 검토한 해결 방향

### 1. CA 인증서 신뢰 추가

백엔드 서버의 CA 인증서를 전달받아 Node가 신뢰하도록 구성

```bash
NODE_EXTRA_CA_CERTS=./backend.pem pnpm dev
```

가장 올바른 해결 방법이다.

---

### 2. Route Handler 기반 프록시

Next.js rewrites 대신 직접 API Route를 구성

```txt
Browser
  ↓
Route Handler
  ↓
Backend Server
```

프록시 로직을 직접 제어할 수 있다.

---

### 3. 개발용 HTTP Endpoint 사용

개발 서버에서만 TLS를 제거한 별도 Endpoint를 사용하는 방법도 검토했다.

---

## 추가 정리

이후 DataGSM 로그인 흐름을 정리하면서 proxy 보호 범위도 함께 수정했다.

```ts
if (pathname.startsWith("/api/") || pathname === "/sign-in") {
  return NextResponse.next();
}
```

```txt
/api
/sign-in
정적 리소스
```

경로는 인증 redirect 대상에서 제외하고,

```txt
관리자 페이지
```

만 accessToken 검사 대상으로 분리했다.

이를 통해

```txt
로그인 진입
API 요청
페이지 보호
```

세 가지 책임을 명확하게 분리할 수 있었다.

---

## 배운 점

이번 문제는 React Query나 Axios 코드의 문제가 아니었다.

처음에는 프론트엔드 로직 문제로 접근했지만,

```txt
Endpoint
↓
Axios
↓
React Query
↓
Rewrites
↓
TLS 인증서
```

순서로 요청 흐름을 추적하면서 실제 원인이 네트워크 계층에 있다는 점을 확인할 수 있었다.

이를 통해 단순히 API 호출 코드만 보는 것이 아니라,

- 브라우저 요청 흐름
- Next.js Rewrites 동작 방식
- TLS 인증서 검증 과정
- Proxy 계층의 역할

까지 함께 고려해야 한다는 점을 경험할 수 있었다.
