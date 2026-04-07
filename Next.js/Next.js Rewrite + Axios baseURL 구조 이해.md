# Next.js Rewrite + Axios baseURL 구조 이해

#

## 1. 문제 상황

Next.js 환경에서 API 요청 구조를 구성하면서 다음과 같은 의문이 생겼다.

- axios의 `baseURL: "/api"`는 실제 어디를 가리키는가?
- `next.config.ts`의 `rewrites()`는 언제 동작하는가?
- axios baseURL과 백엔드 URL이 중복되는 것 아닌가?

---

## 2. 전체 구조

현재 프로젝트는 다음과 같은 구조로 API 요청을 처리한다.

### Axios 설정

```tsx
// shared/api/client.ts

import axios from "axios";

export const api = axios.create({
  baseURL: "/api",
  timeout: 10000,
});
```

### 📦 Next.js rewrite 설정

```tsx
// next.config.ts

async rewrites() {
  return [
    {
      source: "/api/:path*",
      destination: `${process.env.NEXT_PUBLIC_API_BASE_URL}/:path*`,
    },
  ];
}
```

---

## 3. 요청 흐름

### 1. 프론트에서 API 호출

```tsx
api.get("/v2/admin/malfunction-reports");
```

실제 요청:

```
/api/v2/admin/malfunction-reports
```

---

### 2. 브라우저 → Next 서버

```
http://localhost:3000/api/v2/admin/malfunction-reports
```

여기서 중요한 점:

- `/api`는 **백엔드 주소가 아니라 Next 서버의 경로**

---

### 3. Next.js rewrite 적용

```tsx
source: "/api/:path*"
→ destination: "https://backend-server/:path*"
```

내부적으로 변환:

```
/api/v2/admin/malfunction-reports
→ https://backend-server/v2/admin/malfunction-reports
```

---

### 4. 백엔드 요청 및 응답

- Next가 대신 백엔드로 요청
- 응답을 다시 브라우저로 전달

---

## 4. 핵심 개념 정리

### 1. baseURL: "/api"의 의미

> 백엔드 URL이 아니라
>
> **Next 서버로 보내는 프록시 진입점**

---

### 2. rewrites()의 역할

> `/api`로 들어온 요청을
>
> **실제 백엔드 주소로 매핑하는 규칙**

---

### 3. 둘의 관계

| 구분          | 역할          |
| ------------- | ------------- |
| axios baseURL | 프론트 → Next |
| rewrite       | Next → 백엔드 |

- 역할이 완전히 다르기 때문에 충돌하지 않음

---

## 5. 왜 이 구조를 사용하는가

### 1. CORS 문제 해결

브라우저에서 직접 백엔드 요청 시 발생하는 CORS 문제를 방지

---

### 2. 환경 분리 용이

```tsx
baseURL: "/api";
```

→ 개발 / 운영 환경에서 코드 변경 없이 유지 가능

---

### 3. 백엔드 주소 숨김

- 실제 API 서버 URL이 노출되지 않음

---

### 4. 구조 안정성

- 프론트는 `/api`만 알면 됨
- 백엔드 주소 변경 시 Next 설정만 수정

---

## 6. 주의할 점

### (X) 잘못된 방식

```tsx
baseURL: process.env.NEXT_PUBLIC_API_BASE_URL;
```

문제:

- rewrite 우회
- CORS 발생 가능
- 구조 깨짐

---

### (O) 올바른 방식

```tsx
baseURL: "/api";
```

항상 Next를 거쳐서 요청

---

## 7. 한 줄 정리

> 프론트는 `/api`로 요청하고,
>
> Next가 그 요청을 받아 실제 백엔드로 전달하는 **프록시 구조**다.

---

## 8. 추가 인사이트

- `next.config.ts`는 실행되는 코드가 아니라
  **Next 서버의 라우팅 규칙 설정 파일**
- axios가 직접 rewrite를 호출하는 것이 아니라
  **브라우저 요청을 Next가 처리하면서 자동 적용**

---

## 9. 아키텍처 요약

```
[Browser]
   ↓
/api/...
   ↓
[Next (rewrite)]
   ↓
[Backend API]
   ↓
[Next]
   ↓
[Browser]
```

---

## 10. 결론

현재 구조는:

- CORS 해결
- 환경 분리
- 유지보수성 확보

까지 모두 고려된 **표준적인 Next.js API 프록시 패턴**이다.
