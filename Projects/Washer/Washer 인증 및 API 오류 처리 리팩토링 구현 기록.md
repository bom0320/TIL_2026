# Washer 인증 및 API 오류 처리 리팩 구현 기록

## 작업 요약

기존 Washer 관리자 페이지는 내 정보 API 요청이 실패하면 오류 원인을 확인하지 않고 모든 오류를 인증 만료로 처리했다.

이 때문에 서버 오류, 네트워크 오류, Zod 응답 검증 오류처럼 인증과 관계없는 문제에서도 Access Token과 Refresh Token이 삭제되고 로그인 페이지로 이동할 수 있었다.

이를 해결하기 위해 Axios와 Zod에서 발생하는 오류를 프로젝트 공통 오류 형태인 `AppError`로 정규화하고, 실제 인증 오류에서만 세션을 정리하도록 오류 처리 흐름을 개선했다.

---

## 1. 문제 상황

관리자 페이지 진입 시 `AdminLayout`에서 `useGetMyInfo`를 호출해 사용자 정보와 관리자 권한을 확인하고 있었다.

기존에는 요청 실패 여부만 확인했다.

```tsx
if (isMyInfoError) {
  queryClient.clear();

  deleteCookie(COOKIE_KEYS.ACCESS_TOKEN);
  deleteCookie(COOKIE_KEYS.REFRESH_TOKEN);

  window.location.href = "/sign-in";
}
```

이 구조는 `isError`가 발생한 이유를 구분하지 않았다.

따라서 다음 오류가 모두 로그인 만료로 처리될 수 있었다.

- Access Token 및 Refresh Token 만료
- 서버 내부 오류
- 네트워크 연결 실패
- 요청 타임아웃
- API 응답 타입 불일치
- Zod Schema 검증 실패
- 기타 알 수 없는 오류

---

## 2. 문제가 드러난 계기

주요 조회 API 응답에 Zod 런타임 검증을 적용하면서 서버 응답과 Schema가 일치하지 않을 때 `ZodError`가 발생하도록 개선했다.

하지만 `getMyInfo`의 Zod 검증이 실패하면 해당 오류도 TanStack Query의 `isError` 상태로 전달됐다.

`AdminLayout`은 오류 종류를 확인하지 않았기 때문에 Zod 검증 오류까지 인증 만료로 판단했다.

```
서버 응답과 Schema 불일치
→ ZodError 발생
→ TanStack Query isError
→ 로그인 만료로 오판
→ Query Cache 초기화
→ 인증 토큰 삭제
→ 로그인 페이지 이동
```

데이터 계약 오류가 인증 오류로 잘못 해석되면서, 인증과 관계없는 문제로 사용자가 강제 로그아웃될 수 있는 구조였다.

---

## 3. 원인 분석

### `isError`는 오류의 존재만 나타냄

TanStack Query의 `isError`는 Query 함수 실행이 실패했다는 사실만 알려준다.

다음과 같은 오류 원인은 직접 구분해주지 않는다.

```
401 인증 오류
403 권한 오류
500 서버 오류
네트워크 오류
Zod 검증 오류
일반 JavaScript 오류
```

따라서 `isError`만으로 인증 만료를 판단하면 안 된다.

### 서로 다른 라이브러리가 다른 형태의 오류를 반환함

API 요청 실패 시 Axios는 `AxiosError`를 반환하고, Schema 검증 실패 시 Zod는 `ZodError`를 반환한다.

각 UI 컴포넌트에서 이 오류들을 직접 확인하면 다음 문제가 생긴다.

- Axios와 Zod에 대한 의존성이 UI까지 전달됨
- 화면마다 오류 분류 코드가 반복됨
- 같은 오류가 화면마다 다르게 처리될 수 있음
- 새로운 오류 유형을 추가할 때 여러 컴포넌트를 수정해야 함

### 세션 정리 책임이 UI에 포함됨

`AdminLayout`과 Header에서 다음 동작을 직접 수행하고 있었다.

- Query Cache 초기화
- Access Token 삭제
- Refresh Token 삭제
- 페이지 이동

화면 렌더링과 인증 데이터 정리 책임이 함께 존재했고, 로그아웃 흐름에서도 코드가 중복됐다.

---

## 4. 설계 방향

오류 처리 흐름을 다음과 같이 나눴다.

```
API 함수
→ Axios 요청
→ Zod 응답 검증
→ 오류 발생 시 AppError로 정규화

Query 또는 Mutation Hook
→ API 함수 실행
→ 성공 시 캐시 갱신
→ 실패 시 정규화된 오류 전달

UI
→ 분류된 오류를 기준으로 메시지·재시도·라우팅 결정
```

핵심 기준은 다음과 같다.

1. API 요청 실패와 인증 만료를 동일하게 취급하지 않음
2. 오류 분류는 API 경계에서 수행
3. UI는 오류 원인을 다시 해석하지 않고 사용자 동작만 결정
4. 실제 인증 오류에서만 세션 정리
5. 원본 오류를 보존해 디버깅 가능성 유지

---

## 5. 공통 오류 타입 설계

프로젝트 내부에서 사용할 오류 형식을 `AppError`로 통일했다.

```tsx
exportconstAPP_ERROR_TYPE= {
  AUTHENTICATION:"AUTHENTICATION",
  FORBIDDEN:"FORBIDDEN",
  BAD_REQUEST:"BAD_REQUEST",
  NOT_FOUND:"NOT_FOUND",
  CONFLICT:"CONFLICT",
  SERVER:"SERVER",
  NETWORK:"NETWORK",
  VALIDATION:"VALIDATION",
  UNKNOWN:"UNKNOWN",
}asconst;
```

```tsx
exportclassAppErrorextendsError {readonly type:AppErrorType;readonly status?:number;readonly cause?:unknown;

  constructor({ type, message, status, cause }:AppErrorOptions) {super(message);this.name="AppError";this.type=type;this.status=status;this.cause=cause;
  }
}
```

### 각 필드의 역할

- `type`: 애플리케이션에서 구분한 오류 종류
- `message`: 사용자 안내 또는 로그에 사용할 메시지
- `status`: HTTP 상태 코드
- `cause`: 원본 `AxiosError`, `ZodError` 등

`AppError`가 기본 `Error`를 확장하기 때문에 TanStack Query의 오류 흐름과 자연스럽게 연결할 수 있다.

---

## 6. 오류 정규화 함수 구현

`normalizeApiError`는 여러 형태의 오류를 `AppError`로 변환한다.

```
AxiosError ─┐
ZodError   ─┼→ normalizeApiError → AppError
Error      ─┤
기타 값    ─┘
```

### 오류 분류 기준

| 발생 조건          | AppError 타입    |
| ------------------ | ---------------- |
| 401                | `AUTHENTICATION` |
| 403                | `FORBIDDEN`      |
| 400, 422           | `BAD_REQUEST`    |
| 404                | `NOT_FOUND`      |
| 409                | `CONFLICT`       |
| 500 이상           | `SERVER`         |
| 서버 응답 없음     | `NETWORK`        |
| Zod 검증 실패      | `VALIDATION`     |
| 분류되지 않은 오류 | `UNKNOWN`        |

서버 오류 응답에 `message`가 포함된 경우 해당 메시지를 유지하고, 없을 때만 클라이언트 기본 메시지를 사용했다.

이미 `AppError`로 변환된 오류가 다시 들어오면 그대로 반환해 중복 변환도 방지했다.

---

## 7. API 함수의 오류 처리 표준화

조회와 변경 API 함수에서 요청, 검증, Mapper 실행을 `try-catch`로 묶었다.

```tsx
exportasyncfunctiongetSomething() {try {constresponse=awaitget<BaseResponseType<unknown>>(...);constparsedData=responseSchema.parse(response.data);returnmapData(parsedData);
  }catch (error) {thrownormalizeApiError(error);
  }
}
```

적용한 주요 조회 API:

- 내 정보 조회
- 사용자 목록 조회
- 기기 목록 조회
- 예약 목록 조회
- 예약 히스토리 조회
- 대시보드 조회
- 고장 신고 목록 조회

적용한 주요 Mutation API:

- 기기 삭제
- 기기 상태 변경
- 예약 삭제
- 사용자 패널티 해제
- 신고 상태 변경

Mutation Hook 내부에서 직접 HTTP 요청을 실행하던 코드도 별도 API 함수로 분리했다.

```
기존
Mutation Hook
→ HTTP 요청
→ 캐시 갱신
→ 오류 UI 처리

개선
API 함수
→ HTTP 요청 및 오류 정규화

Mutation Hook
→ API 함수 실행
→ 캐시 갱신 및 사용자 안내
```

이를 통해 API 요청과 TanStack Query 상태 관리의 책임을 분리했다.

---

## 8. 예약 히스토리 응답 검증 추가

기존 예약 히스토리 조회는 TypeScript 타입만 지정하고 실제 서버 응답을 검증하지 않았다.

이를 `BaseResponseType<unknown>`으로 수신한 뒤 Zod Schema로 검증하도록 변경했다.

```tsx
constresponse=awaitget<BaseResponseType<unknown>>(...);constparsedData=machineReservationHistoryResponseSchema.parse(response.data);
```

검증된 데이터만 Mapper에 전달하도록 구성했다.

```
API 응답
→ unknown
→ Zod 검증
→ 검증된 데이터
→ Mapper
→ UI
```

---

## 9. 인증 오류와 일반 오류 처리 분리

`AdminLayout`에서 `isError`만 확인하지 않고 실제 `AppError.type`을 확인하도록 변경했다.

```tsx
if (
  myInfoErrorinstanceofAppError &&
  myInfoError.type === APP_ERROR_TYPE.AUTHENTICATION
) {
  clearAuthSession(queryClient);
  window.location.replace("/sign-in");
}
```

### 인증 오류

```
AUTHENTICATION
→ Query Cache 초기화
→ Access Token 삭제
→ Refresh Token 삭제
→ 로그인 페이지 이동
```

### 일반 오류

```
SERVER / NETWORK / VALIDATION / UNKNOWN
→ 인증 토큰 유지
→ 오류 안내
→ 다시 시도 버튼 제공
```

### 권한이 없는 일반 사용자

일반 사용자의 관리자 페이지 접근은 인증 실패가 아니므로 토큰을 삭제하지 않고 앱 다운로드 페이지로 이동시켰다.

```
role === USER
→ 세션 유지
→ /app-download 이동
```

---

## 10. 세션 정리 로직 공통화

중복되던 Query Cache와 인증 쿠키 정리 로직을 `clearAuthSession`으로 분리했다.

```tsx
exportconstclearAuthSession = (queryClient: QueryClient): void => {
  queryClient.clear();
  deleteCookie(COOKIE_KEYS.ACCESS_TOKEN);
  deleteCookie(COOKIE_KEYS.REFRESH_TOKEN);
};
```

다음 흐름에서 공통으로 사용한다.

- 인증 만료 처리
- Header 로그아웃

페이지 이동은 함수에 포함하지 않았다.

세션 정리 함수는 인증 데이터 정리만 담당하고, 로그인 페이지나 다른 화면으로 이동할지는 호출한 UI가 결정하도록 책임을 분리했다.

---

## 11. TanStack Query 재시도 정책

프로젝트의 기존 전역 설정을 확인한 결과 Query 재시도는 이미 비활성화돼 있었다.

```tsx
queries: {retry:false,
}
```

인증 오류나 Zod 검증 오류가 자동으로 반복 요청되는 문제는 없었기 때문에 기존 정책을 유지했다.

서버 및 네트워크 오류에서는 자동 재시도 대신 사용자가 직접 실행할 수 있는 `다시 시도` 버튼을 제공했다.

---

## 12. 검증한 시나리오

### 정상 관리자 접근

- 내 정보 조회 성공
- 관리자 페이지 정상 렌더링
- 대시보드 및 주요 목록 조회 확인

### 일반 사용자 접근

- 관리자 페이지 진입 차단
- 인증 토큰 유지
- `/app-download`로 이동

### 인증 만료

- Access Token 재발급 실패
- Query Cache 및 인증 토큰 삭제
- `/sign-in`으로 이동

### Zod 응답 검증 실패

- `VALIDATION` 오류로 분류
- 강제 로그아웃되지 않음
- 사용자 세션 유지
- 오류 화면과 다시 시도 버튼 표시

### 로그아웃

- Query Cache 초기화
- Access Token 및 Refresh Token 삭제
- `/sign-in`으로 이동

### Query 및 Mutation 오류

- API 함수에서 `AppError`로 정규화
- Hook의 `error` 또는 `onError`로 전달
- 정규화된 메시지 출력 확인

---

## 13. 개선 결과

### 사용자 관점

- 인증과 무관한 오류로 강제 로그아웃되는 문제 방지
- 일시적인 서버·네트워크 오류에서 로그인 상태 유지
- 오류 상황에 맞는 안내와 재시도 흐름 제공
- 일반 사용자의 유효한 인증 상태를 불필요하게 제거하지 않음

### 개발 관점

- AxiosError와 ZodError를 `AppError` 하나로 통일
- UI에서 외부 라이브러리 오류 형태를 직접 판단하지 않도록 개선
- API 함수와 Query/Mutation Hook 책임 분리
- 서버 응답 메시지와 HTTP 상태 코드 보존
- 원본 오류를 `cause`에 유지해 원인 추적 가능
- 조회 및 변경 API에 동일한 오류 처리 기준 적용
- 인증 세션 정리 코드 중복 제거

---

## 14. 한계와 후속 개선

현재 TanStack Query의 전역 재시도 정책은 `retry: false`다.

서비스 운영 데이터가 축적되면 네트워크 또는 일부 서버 오류에만 제한적인 자동 재시도를 적용할 수 있다.

예시:

```tsx
retry: (failureCount, error) => {
  if (!errorinstanceofAppError) returnfalse;
  return (
    failureCount < 2 &&
    (error.type === APP_ERROR_TYPE.NETWORK ||
      error.type === APP_ERROR_TYPE.SERVER)
  );
};
```

또한 현재 오류는 화면 안내와 원본 `cause` 보존까지만 처리하고 있으므로, 이후 Sentry와 같은 오류 추적 도구를 도입하면 다음 정보를 수집할 수 있다.

- API URL
- HTTP 상태 코드
- 오류 유형
- Zod 검증 실패 경로
- 사용자 행동 흐름
- 오류 발생 빈도

---

# 이력서용 소재

## 문제

내 정보 조회 실패를 모두 로그인 만료로 처리해 서버·네트워크·Zod 응답 검증 오류에도 사용자 세션이 삭제될 수 있었다.

## 수행

- Axios와 Zod 오류를 프로젝트 공통 `AppError`로 정규화
- HTTP 상태와 오류 발생 지점에 따라 인증·권한·요청·충돌·서버·네트워크·응답 검증 오류 분류
- 조회 및 Mutation API의 오류 처리 흐름 통일
- 인증 오류에서만 Query Cache와 인증 토큰을 정리하도록 변경
- 비인증 오류에는 세션 유지 및 재시도 UI 제공
- 중복된 세션 정리 로직을 공통 함수로 분리

## 결과

- 인증과 관계없는 오류로 인한 불필요한 강제 로그아웃 방지
- 서버·네트워크·API 계약 오류에서도 사용자 세션 유지
- AxiosError와 ZodError의 처리 기준 통일
- 오류 원인에 맞는 사용자 안내와 복구 흐름 제공
- API 함수와 Query/Mutation Hook의 책임 분리
- 원본 오류와 HTTP 상태를 보존해 원인 추적 가능

---

# 이력서 문장 초안

> 내 정보 조회의 모든 실패를 인증 만료로 처리해 서버·네트워크·응답 검증 오류에도 세션이 삭제되던 문제를 분석하고, Axios·Zod 오류를 공통 `AppError`로 정규화했습니다. 실제 인증 오류에서만 Query Cache와 인증 토큰을 정리하도록 변경하고, 주요 조회·변경 API의 오류 처리 기준과 재시도 흐름을 통일했습니다.

더 짧게 쓰면:

> Axios·Zod 오류를 공통 `AppError`로 정규화하고 인증·권한·서버·네트워크·응답 검증 오류를 분리해, 비인증 오류로 인한 강제 로그아웃을 방지하고 주요 API의 오류 처리 흐름을 표준화했습니다.

---

# 면접 답변용 핵심 흐름

```
Zod 런타임 검증 도입
→ 응답 계약 오류가 명확하게 드러남
→ 모든 Query 오류를 로그인 만료로 처리하던 기존 구조 발견
→ AxiosError와 ZodError를 AppError로 정규화
→ 실제 인증 오류에서만 세션 정리
→ 비인증 오류에는 세션 유지와 재시도 제공
→ 주요 Query·Mutation API까지 동일 기준 확장
```
