# Next.js rewrites vs Axios Direct Call 구조 변화

## 1. 기존 구조 (Next.js rewrites 사용)

```
브라우저
   ↓
/api/v2/admin/users
   ↓
Next 서버 (rewrites)
   ↓
https://백엔드/api/v2/admin/users
```

### 특징

- 브라우저는 실제 백엔드 주소를 모름
- Next 서버가 중간에서 **프록시 역할** 수행
- 요청이 한 번 더 서버를 거침

---

## 2. 변경된 구조 (Axios Direct Call)

```
브라우저
   ↓
https://백엔드/api/v2/admin/users
```

### 특징

- 브라우저가 백엔드에 **직접 요청**
- Next 서버를 거치지 않음
- rewrites 설정 불필요

---

## 3. 핵심 차이

- 이전: Next가 대신 요청 (프록시 구조)
- 현재: 브라우저가 직접 요청 (Direct Call 구조)

---

## 4. 구조를 변경한 이유

### Self-signed certificate 문제

#### 기존

```
Next 서버(Node) → 백엔드
```

- Node 환경에서 인증서 검증 수행
- self-signed certificate 오류 발생

#### 변경 후

```
브라우저 → 백엔드
```

- 브라우저가 인증서를 처리
- 문제 우회 가능

---

## 5. 구조 변경으로 인한 영향

### 1) rewrites 제거

- 더 이상 Next 서버 프록시 필요 없음

---

### 2) baseURL 변경

```ts
baseURL: process.env.NEXT_PUBLIC_API_URL;
```

- 실제 백엔드 주소를 직접 사용

---

### 3) CORS 영향 발생

#### 기존

- Next 서버가 요청 → CORS 영향 없음

#### 현재

- 브라우저가 직접 요청 → **CORS 필수**

---

### 4) interceptor는 그대로 유지

Axios interceptor 역할:

- Authorization 헤더 자동 추가
- 토큰 만료 시 refresh 처리
- 응답 데이터 가공

* 구조가 바뀌어도 **인증 로직은 그대로 필요**

---

## 6. 개념 정리

### rewrites

- URL을 다른 서버로 전달하는 프록시 역할
- 요청 경로만 변경
- 인증/로직 처리 없음

---

### interceptor

- 요청/응답을 가공하는 로직 레이어
- 토큰 처리, 에러 처리 담당

---

## 7. 비유

- 이전: 배달 대행(Next)이 음식 가져다줌
- 현재: 직접 음식점 가서 가져옴

---

## 8. 현재 상태 요약

- rewrites 제거
- axios baseURL → 백엔드 직접 연결
- interceptor 유지

* **프록시 구조 → Direct Call 구조로 전환 완료**

---

## 9. 다음 단계

- 404 → API 경로 문제 확인 (`/api` 포함 여부)
- 403 → 인증/권한 문제 확인 (토큰, Authorization)

---

## 한 줄 정리

> Next 프록시를 제거하고, 브라우저가 백엔드에 직접 요청하는 구조로 변경했다.
