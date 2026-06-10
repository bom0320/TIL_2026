# FSD를 참고한 레이어 구조 정리

## FSD를 참고한 레이어 구조 정리

Washer는 Next.js App Router 기반의 관리자 웹으로, 세탁기, 예약, 신고, 사용자 등 여러 도메인의 데이터를 다룬다.

페이지 컴포넌트 안에서 API 호출, 서버 상태 관리, 필터링, UI 렌더링을 모두 처리하면 기능이 늘어날수록 코드 위치를 예측하기 어려워진다. 또한 팀원마다 API 호출 방식, queryKey 작성 방식, 데이터 가공 기준이 달라질 수 있다.

이를 줄이기 위해 FSD를 참고해 `app`, `widgets`, `features`, `entities`, `shared` 계층으로 역할을 나누었다.

```txt
src/
  app/
  widgets/
  features/
  entities/
  shared/
  lib/
```

| Layer      | 역할                                                     |
| ---------- | -------------------------------------------------------- |
| `app`      | Next.js App Router 기반 라우팅 진입점                    |
| `widgets`  | 페이지 단위 UI 조합                                      |
| `features` | 로그인, 예약 생성, 상태 변경처럼 사용자 행동 중심의 기능 |
| `entities` | machine, reservation, report, user 등 도메인 데이터 관리 |
| `shared`   | 공통 API wrapper, endpoint, queryKey, 공통 타입 관리     |
| `lib`      | 프로젝트 전역 설정성 로직                                |

---

## entities: 도메인별 API와 서버 상태 관리

`entities` 계층은 도메인별 API 함수, 타입, query hook을 관리한다.
예를 들어 report 도메인은 다음과 같이 구성한다.

```txt
entities/report/
  api/
    getMalfunctionReports.ts
    useGetMalfunctionReports.ts
    index.ts
  model/
    types.ts
  ui/
  index.ts
```

`getMalfunctionReports.ts`는 실제 API 요청만 담당한다.

```ts
export const getMalfunctionReports = async (
  params?: ReportParamsType
): Promise<BaseResponseType<ReportResponseType>> => {
  const response = await get<BaseResponseType<ReportResponseType>>(
    reportUrl.getMalfunctionReports(),
    {
      params,
    }
  );

  return response;
};
```

이 함수는 다음 역할만 수행한다.

- `reportUrl`에서 endpoint를 가져온다.
- 요청 params를 전달한다.
- 응답 타입을 `BaseResponseType<ReportResponseType>`로 지정한다.
- 서버 응답을 그대로 반환한다.

즉, API 함수는 서버 요청과 raw response 반환만 담당한다.
상태값 변환, 시간 포맷팅, 필드명 변경 같은 UI 가공 로직은 API 함수에 포함하지 않는다.

`useGetMalfunctionReports.ts`는 API 함수를 React Query와 연결하는 역할을 담당한다.

```ts
export const useGetMalfunctionReports = (
  params?: ReportParamsType,
  initialData?: BaseResponseType<ReportResponseType>
) => {
  const queryKey = reportQueryKeys.getMalfunctionReports(params || {});

  return useQuery({
    queryKey,
    queryFn: () => fetchMalfunctionReports(params),
    initialData,
  });
};
```

이 hook은 다음 역할을 담당한다.

- queryKey를 설정한다.
- API 함수를 queryFn으로 연결한다.
- initialData를 처리한다.
- loading, error, refetch 등 서버 상태를 컴포넌트에 제공한다.

이를 통해 API 함수와 React Query hook의 책임을 분리한다.

```txt
API function
  → 서버 요청
  → raw response 반환

Query hook
  → queryKey 설정
  → caching
  → loading/error 관리
  → refetch 제공
```

---

## shared: 공통 API 기반 관리

`shared/api`에서는 프로젝트 전체에서 공통으로 사용하는 API 기반 코드를 관리한다.

```txt
shared/api/
  apiUrls.ts
  http.ts
  queryKeys.ts
  types.ts
```

각 파일의 역할은 다음과 같다.

| 파일           | 역할                                                  |
| -------------- | ----------------------------------------------------- |
| `apiUrls.ts`   | 도메인별 endpoint 중앙 관리                           |
| `http.ts`      | axiosInstance 기반 get/post/patch/delete wrapper 제공 |
| `queryKeys.ts` | React Query queryKey 중앙 관리                        |
| `types.ts`     | 백엔드 공통 응답 타입 관리                            |

### apiUrls.ts

`apiUrls.ts`는 API endpoint를 도메인별로 관리한다.

```ts
export const reportUrl = {
  getMalfunctionReports: () => "/api/v2/admin/malfunction-reports",
} as const;
```

API 함수에서 endpoint 문자열을 직접 작성하지 않고, `reportUrl.getMalfunctionReports()`처럼 URL 생성 함수를 참조한다.
이를 통해 endpoint 하드코딩을 줄이고, API 경로 변경 시 수정 위치를 예측하기 쉽게 만든다.

### http.ts

`http.ts`는 axiosInstance를 감싼 공통 HTTP wrapper를 제공한다.

```ts
export const get = async <T>(...args: Parameters<typeof axiosInstance.get>) =>
  await axiosInstance.get<T, T>(...args);
```

컴포넌트나 API 함수에서 axiosInstance를 직접 호출하지 않고, `get`, `post`, `patch`, `delete` 같은 공통 요청 함수를 사용하도록 한다.

### queryKeys.ts

`queryKeys.ts`는 React Query의 queryKey를 도메인별로 관리한다.

```ts
export const reportQueryKeys = {
  all: ["reports"] as const,
  getMalfunctionReports: (params?: ReportParamsType) =>
    ["reports", "list", params] as const,
} as const;
```

queryKey를 문자열로 직접 작성하지 않고, 도메인별 함수로 관리한다.
이를 통해 캐싱 기준을 통일하고, mutation 이후 invalidate 범위를 예측하기 쉽게 만든다.

### types.ts

`types.ts`는 백엔드 공통 응답 타입을 관리한다.

```ts
export type BaseResponseType<T> = {
  status: string;
  code: number;
  message: string;
  data: T;
};
```

백엔드 응답의 공통 구조는 `BaseResponseType<T>`로 관리하고, 실제 `data`에 들어가는 도메인별 응답 타입은 제네릭으로 분리한다.

---

## API 호출 흐름

Washer의 API 호출 흐름은 다음과 같이 구성한다.

```txt
component
  ↓
query hook
  ↓
API function
  ↓
http wrapper
  ↓
apiUrls
  ↓
API server
```

예를 들어 고장 신고 목록 조회는 다음 흐름으로 실행된다.

```txt
ReportsPage
  ↓
useGetMalfunctionReports(params)
  ↓
getMalfunctionReports(params)
  ↓
get<BaseResponseType<ReportResponseType>>(...)
  ↓
reportUrl.getMalfunctionReports()
```

이 구조를 통해 컴포넌트는 axiosInstance나 endpoint 문자열에 직접 의존하지 않는다.
컴포넌트는 도메인별 query hook을 통해 필요한 서버 상태만 가져온다.

---

## widgets: 페이지 단위 데이터 조합

`widgets` 계층은 페이지 단위 UI를 조합한다.
예를 들어 `ReportsPage`에서는 report와 machine 도메인의 query hook을 조합해 신고 목록과 필터 데이터를 구성한다.

```tsx
const {
  data: reportsData,
  isLoading: isReportsLoading,
  isError: isReportsError,
  refetch: refetchReports,
} = useGetMalfunctionReports({
  status,
});

const { data: machinesData, isLoading: isMachinesLoading } = useGetMachines({
  floor,
});
```

페이지에서는 직접 API를 호출하지 않고, 도메인별 query hook을 통해 서버 상태를 가져온다.

`ReportsPage`는 다음 역할을 담당한다.

- 신고 목록 데이터를 가져온다.
- 세탁기 데이터를 가져온다.
- 신고 상태, 검색어, 층수 필터 상태를 관리한다.
- machine 데이터와 report 데이터를 조합해 층수 기준 필터링을 수행한다.
- 필터링된 데이터를 `ReportsPanel`에 전달한다.
- 필터 UI 상태를 `ReportFilterPanel`에 전달한다.

```tsx
const filteredReports = useMemo(() => {
  let result = reports;

  if (floor !== undefined) {
    const machineIdsOnFloor = new Set(machines.map((m) => m.id));
    result = result.filter((report) => machineIdsOnFloor.has(report.machineId));
  }

  if (search) {
    result = result.filter((report) =>
      report.reporterName.toLowerCase().includes(search.toLowerCase())
    );
  }

  return result;
}, [reports, machines, floor, search]);
```

이 코드는 report 데이터만으로는 처리하기 어려운 층수 필터링을 machine 데이터와 조합해 처리한다.
즉, widget은 여러 도메인의 데이터를 페이지 요구사항에 맞게 조합하는 역할을 담당한다.

`ReservationsPage`에서도 reservation query hook으로 데이터를 가져온 뒤, 화면 기준에 맞게 세탁기와 건조기 예약 현황을 분리한다.

```tsx
const { data, isLoading, isError } = useGetReservations();

const reservations = data ?? [];

const dryerReservations = reservations.filter((item) => item.type === "DRYER");

const washerReservations = reservations.filter(
  (item) => item.type === "WASHER"
);
```

`ReservationsPage`는 다음 역할을 담당한다.

- 예약 데이터를 가져온다.
- 세탁기 예약과 건조기 예약을 분리한다.
- 예약 현황 패널에 데이터를 전달한다.
- 예약 이력 모달의 open/close 상태를 관리한다.
- 선택한 machineName을 모달에 전달한다.

이처럼 widget은 도메인 데이터를 가져와 페이지 화면에 맞게 조합하고, API 요청 세부 구현은 `entities`와 `shared` 계층에 위임한다.

---

## 데이터 처리 기준

API 함수는 서버 응답을 가져오는 역할만 담당한다.

```txt
API function
  → URL 호출
  → params 전달
  → 응답 타입 지정
  → raw response 반환
```

API 함수에서는 다음을 처리하지 않는다.

- 상태값 한글 변환
- 시간 포맷팅
- 필드명 변경
- UI 전용 데이터 생성

서버 응답을 그대로 사용할 수 있는 도메인은 raw response를 유지한다.
예를 들어 report 도메인은 단순 조회 중심의 데이터이며, status label과 style 변환도 별도 map으로 처리할 수 있기 때문에 mapper를 강제하지 않는다.

반대로 reservation처럼 상태 변환, 남은 시간 계산, 예약 이력 표시 등 UI 요구사항이 복잡한 도메인은 mapper 또는 별도 util을 통해 데이터를 가공한다.

즉, 모든 API 응답에 mapper를 강제하지 않고, 도메인별 UI 요구사항에 따라 데이터 가공 여부를 판단한다.

---

## 정리

이 구조를 통해 다음 기준을 만든다.

- `app`은 라우팅 진입점으로 사용한다.
- `widgets`는 페이지 단위 UI와 데이터 조합을 담당한다.
- `features`는 로그인, 예약 생성, 상태 변경처럼 사용자 행동 중심의 기능을 담당한다.
- `entities`는 도메인별 API 함수, 타입, query hook을 관리한다.
- `shared`는 공통 HTTP wrapper, endpoint, queryKey, 공통 타입을 관리한다.
- API 함수는 raw response 반환만 담당한다.
- React Query hook은 queryKey, caching, loading/error, refetch 등 서버 상태 관리를 담당한다.
- 페이지 컴포넌트는 직접 axios를 호출하지 않고 도메인별 query hook을 사용한다.
- widget은 여러 도메인의 데이터를 페이지 요구사항에 맞게 조합한다.
- 데이터 가공은 모든 API에 강제하지 않고, 필요한 도메인에만 적용한다.
