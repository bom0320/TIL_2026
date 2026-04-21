# API Architecture & Data Handling Strategy

> 이 문서는 API 호출 구조와 데이터 처리 기준을 정의한다.

---

## 1. 목적

- API 호출 구조를 일관되게 유지한다
- 서버 데이터 처리 기준을 명확하게 정의한다
- 불필요한 데이터 가공을 방지한다
- 유지보수성과 확장성을 확보한다

---

## 2. 핵심 원칙

### 2.1 Raw Response 우선

서버 응답은 가능한 한 그대로 사용한다.

- 필드 의미 변경 (X)
- 상태 변환 (X)
- 시간 포맷팅 (X)
- 임의 데이터 가공 (X)

```
API → Raw Response → UI
```

### 2.2 필요한 경우에만 Mapper 사용

다음 조건에 해당할 때만 mapper를 사용한다.

- UI 요구사항이 복잡한 경우
- 상태 변환이 필요한 경우
- 시간 계산/포맷이 필요한 경우
- 서버 응답을 그대로 사용하기 어려운 경우

```
API → Raw Response → Mapper → UI
```

---

### 2.3 API는 “데이터 가져오기”만 담당

API 레이어에서는 데이터 가공을 수행하지 않는다.

- API 함수는 반드시 raw 데이터를 반환한다
- 비즈니스 로직은 포함하지 않는다

---

## 3. Shared API 구조

### 3.1 api.ts

HTTP 요청을 담당하는 axios wrapper

```tsx
get<T>(url,options)
post<T>(...)
```

역할:

- 요청 전송
- 공통 설정 처리
- 응답 반환

---

### 3.2 apiUrls.ts

API endpoint를 중앙에서 관리한다.

```tsx
export const reportUrl = {
  getMalfunctionReports: () => "/api/v2/admin/malfunction-reports",
};
```

역할:

- URL 문자열 관리
- 도메인별 endpoint 분리

---

### 3.3 types.ts

공통 응답 타입 정의

```tsx
export type BaseResponseType<T> = {
  status: string;
  code: number;
  message: string;
  data: T;
};
```

---

### 3.4 queryKeys.ts

React Query key를 중앙에서 관리한다.

```tsx
export const queryKeys= {
  users: {
    list: (params) => ["users","list",params]asconst,
    my: () => ["users","my"]asconst,
  },
};
```

역할:

- queryKey 구조 통일
- 캐싱 안정성 확보
- invalidateQueries 대응

---

## 4. API 레이어 구조

### 4.1 API 함수 (getXXX.ts)

API 호출만 담당한다.

```tsx
export const getMalfunctionReports = async (
  params?: ReportParamsType
): Promise<BaseResponseType<ReportResponseType>> => {
  constresponse = awaitget<BaseResponseType<ReportResponseType>>(
    reportUrl.getMalfunctionReports(),
    {
      params,
    }
  );

  returnresponse;
};
```

특징:

- URL 호출
- params 전달
- 타입 지정
- ❌ 데이터 가공 없음

---

### 4.2 React Query Hook (useXXX.ts)

서버 상태 관리를 담당한다.

```tsx
export const useGetMalfunctionReports = (params) => {
  returnuseQuery({
    queryKey: queryKeys.reports.list(params),
    queryFn: () => getMalfunctionReports(params),
  });
};
```

역할:

- queryKey 설정
- 캐싱 관리
- loading / error 상태 관리

---

## 5. 데이터 흐름

```
Component
  ↓
useQuery (React Query)
  ↓
API 함수 (getXXX)
  ↓
axios wrapper (get)
  ↓
API 서버
```

---

## 6. 데이터 처리 전략

### 6.1 Case 1: Raw 사용 (Report)

조건:

- 서버 응답을 그대로 사용 가능

특징:

- 단일 타입 유지
- mapper 사용 ❌

```tsx
Promise<BaseResponseType<ReportResponseType>>;
```

---

### 6.2 Case 2: Mapper 사용 (Reservation)

조건:

- UI 요구사항이 복잡함

예:

- 상태 변환 필요
- 시간 계산 필요
- 추가 필드 생성 필요

```tsx
mapReservation(dto)→ ReservationItem
```

---

## 7. 판단 기준

| 기준                         | 처리 방식   |
| ---------------------------- | ----------- |
| 서버 데이터 그대로 사용 가능 | Raw 유지    |
| UI 요구 복잡                 | Mapper 사용 |
| 상태/시간 가공 필요          | Mapper 사용 |

---

## 8. 통일해야 하는 것

- axios instance 사용
- apiUrls 구조
- queryKeys 구조
- API 호출 패턴

---

## 9. 통일하지 않아도 되는 것

- 데이터 가공 방식
- mapper 사용 여부
- entity별 처리 방식

---

## 10. Anti-Pattern

다음은 피해야 한다.

- API 레이어에서 데이터 가공
- 모든 API에 mapper 강제 적용
- queryKey를 문자열로 직접 작성
- endpoint를 하드코딩

---

## 11. 최종 정리

```
API는 데이터를 가져오기만 한다.
데이터 가공은 필요할 때만 수행한다.
```
