# Washer Admin 리팩토링 정리

## 목적

Washer Admin 프로젝트의 API 계층과 React Query 구조를 통일하고,
도메인별로 일관된 패턴을 적용하기 위한 리팩토링.

특히 Reports 도메인을 기준으로:

- API URL 관리
- API 함수 분리
- React Query Hook 분리
- Query Key 중앙 관리

구조를 정리하는 것이 목적이다.

---

# 공통 API 관리 패턴

## shared/apiUrls.ts

역할:

- API 엔드포인트 주소만 관리

예시

```tsx
export const reportUrl = {
  getMalfunctionReports: () => "/api/v2/admin/malfunction-reports",
};

export const userUrl = {
  getUsers: () => "/api/v2/admin/users",
};
```

즉,

```tsx
reportUrl.getMalfunctionReports();
```

를 호출하면 URL 문자열만 반환한다.

```
/api/v2/admin/malfunction-reports
```

실제 요청은 보내지 않는다.

---

## shared/api.ts

역할:

- axios wrapper
- 실제 HTTP 요청 수행

예시

```tsx
get<T>(url, options);
post<T>(url, body);
patch<T>(url, body);
delete (<T>url);
```

예시

```tsx
const response = await get<UserResponse>(userUrl.getUsers());
```

이때 실제 GET 요청이 발생한다.

---

## 관계

```
apiUrls.ts
    ↓
URL 제공

shared/api.ts
    ↓
실제 HTTP 요청

도메인 API 함수
    ↓
데이터 반환
```

---

# Query Key 관리

## shared/queryKeys.ts

역할:

- React Query의 queryKey 중앙 관리

예시

```tsx
export const userQueryKeys = {
  getUsers: (params?: UserParamsType) => ["users", "list", params] as const,

  getMyInfo: () => ["users", "my"] as const,
};
```

---

## 사용하는 이유

### 1. 키 이름 통일

잘못된 예

```tsx
["user"]["users"]["user-list"];
```

이런 실수를 방지.

---

### 2. invalidateQueries 편리

```tsx
queryClient.invalidateQueries({
  queryKey: userQueryKeys.getUsers(),
});
```

---

### 3. Params 포함 Query 일관화

```tsx
["users", "list", params];
```

패턴 유지 가능.

---

# API 함수와 React Query Hook의 역할 차이

많이 헷갈리는 부분.

---

## getMalfunctionReports.ts

역할

```
어떻게 데이터를 가져올 것인가?
```

담당:

- 어떤 URL 호출
- 어떤 params 전달
- 어떤 응답 타입 반환

---

예시

```tsx
export const getMalfunctionReports = async (params?: ReportParamsType) => {
  return get<BaseResponseType<ReportResponseType>>(
    reportUrl.getMalfunctionReports(),
    {
      params,
    }
  );
};
```

---

### 책임

- API 호출
- params 전달
- 응답 타입 지정

---

### 책임 아님

- React Query
- 캐시
- 로딩 상태
- 에러 상태

---

즉

```
Raw 데이터를 받아오는 함수
```

이다.

---

# useGetMalfunctionReports.ts

역할

```
API 호출을 React Query와 연결
```

---

예시

```tsx
export const useGetMalfunctionReports = (params?: ReportParamsType) => {
  return useQuery({
    queryKey: reportQueryKeys.getMalfunctionReports(params),

    queryFn: () => getMalfunctionReports(params),
  });
};
```

---

### 담당

- queryKey 설정
- useQuery 연결
- 캐시 관리
- refetch 정책
- staleTime
- enabled

---

즉

```
서버 상태 관리 계층
```

이다.

---

# 실제 요청 흐름

## 1단계

컴포넌트

```tsx
const { data } = useGetMalfunctionReports({
  status: "PENDING",
});
```

---

## 2단계

Hook 실행

```tsx
useQuery({
  queryKey: reportQueryKeys.getMalfunctionReports(params),

  queryFn: () => getMalfunctionReports(params),
});
```

---

## 3단계

API 함수 실행

```tsx
getMalfunctionReports(params);
```

---

## 4단계

URL 생성

```tsx
reportUrl.getMalfunctionReports();
```

결과

```
/api/v2/admin/malfunction-reports
```

---

## 5단계

HTTP 요청

```tsx
get<BaseResponseType<ReportResponseType>>("/api/v2/admin/malfunction-reports", {
  params,
});
```

실제 요청

```
GET
/api/v2/admin/malfunction-reports?status=PENDING
```

---

## 6단계

서버 응답

```json
{
  "status": "OK",
  "code": 200,
  "message": "OK",
  "data": {
    "reports": [],
    "totalCount": 0
  }
}
```

---

## 7단계

응답 반환

```tsx
return response;
```

---

## 최종 흐름

```
Component
    ↓
useGetMalfunctionReports
    ↓
getMalfunctionReports
    ↓
reportUrl.getMalfunctionReports
    ↓
get()
    ↓
Axios
    ↓
Server
    ↓
Response
    ↓
React Query Cache
    ↓
Component
```

---

# Reservation 리팩토링 방향

Reservation도 Reports와 동일한 패턴 적용.

---

## 맞춰야 하는 것

### API 호출

기존

```tsx
axiosInstance.get(...)
```

변경

```tsx
get(...)
```

---

### URL 하드코딩 제거

기존

```tsx
"/reservations";
```

변경

```tsx
reservationUrl.getReservations();
```

---

### Query Key 하드코딩 제거

기존

```tsx
["reservations"];
```

변경

```tsx
reservationQueryKeys.getReservations();
```

---

# 반드시 똑같이 맞출 필요는 없는 것

## 응답 데이터 구조

도메인 특성에 따라 다를 수 있다.

---

### Reports

현재는 서버 응답을 거의 그대로 사용

```tsx
Promise<BaseResponseType<ReportResponseType>>;
```

Raw Response 유지

---

### Reservations

이미 UI 중심 구조가 많음

예시

```tsx
mapReservations();
mapMachineReservations();
mapReservationHistory();
```

---

즉

```
Server Response
    ↓
Mapper
    ↓
UI Model
```

구조가 자연스럽다.

---

# Mapper가 필요한 경우

서버 응답이 UI와 다를 때.

예시

```json
{
  "status": "PENDING"
}
```

↓

```tsx
{
  label: "예약 대기",
  color: "yellow",
}
```

---

또는

```json
{
  "createdAt": "2026-06-10T12:00:00"
}
```

↓

```tsx
{
  createdDate: "2026.06.10";
}
```

---

이런 경우 Mapper를 둔다.

---

# Mapper가 필요 없는 경우

서버 응답을 그대로 사용해도 문제없는 경우.

예시

```tsx
Promise<BaseResponseType<ReportResponseType>>;
```

---

현재 Reports는 여기에 가까움.

---

# Path Alias 정리

## tsconfig

```json
{
  "paths": {
    "@repo/auth": ["..."],
    "@repo/storage": ["..."]
  }
}
```

---

## Turborepo

```
turbo.json
```

설정과 연결.

---

## 사용하는 이유

상대경로 제거

기존

```tsx
import Auth from "../../../../auth";
```

변경

```tsx
import Auth from "@repo/auth";
```

---

가독성과 유지보수 향상.

---

# 현재 정리된 원칙

## 1

API URL은

```tsx
apiUrls.ts;
```

에서 관리한다.

---

## 2

실제 HTTP 요청은

```tsx
shared / api.ts;
```

가 담당한다.

---

## 3

도메인 API 함수는

```
어떤 URL을 호출할지
어떤 Params를 전달할지
어떤 타입을 반환할지
```

만 담당한다.

---

## 4

React Query Hook은

```
queryKey
cache
staleTime
refetch
enabled
```

등 서버 상태 관리만 담당한다.

---

## 5

Query Key는

```tsx
shared / queryKeys.ts;
```

에서 중앙 관리한다.

---

## 6

서버 응답을 그대로 사용한다면

```
도메인 모델 유지
```

---

## 7

필드 의미 변경,
상태 변환,
날짜 포맷,
UI 전용 데이터 생성이 많아질 경우

```
DTO
 ↓
Mapper
 ↓
UI Model
```

구조로 분리한다.

---

# 지금 해야 할 작업

Reports 도메인을 기준 구조로 삼아

- Reservation API
- Reservation Query Key
- Reservation Hook

을 동일한 패턴으로 통일한다.

단,

Reservation의 Mapper 계층은 유지하고,

무조건 Raw Response로 맞추려고 하지는 않는다.

핵심은

```
호출 방식 통일
데이터 구조 강제 통일 X
```

이다.
