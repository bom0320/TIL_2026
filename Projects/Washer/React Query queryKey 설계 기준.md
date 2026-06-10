# React Query queryKey 설계 기준

## 핵심 흐름

Washer에서는 React Query의 `queryKey`를 각 query hook 안에서 문자열로 직접 작성하지 않고, `shared/api/queryKeys.ts`에서 도메인별로 중앙 관리한다.

```ts
export const reportQueryKeys = {
  all: ["reports"] as const,
  getMalfunctionReports: (params?: ReportParamsType) =>
    ["reports", "list", params] as const,
  getReportById: (id: number) => ["reports", "detail", id] as const,
} as const;
```

이 구조는 서버 상태의 캐시 기준을 일관되게 관리하기 위한 구조이다.

React Query는 `queryKey`를 기준으로 데이터를 캐싱한다.
따라서 같은 데이터를 조회하는데 queryKey가 달라지면 서로 다른 캐시로 인식하고, 반대로 다른 조건의 데이터인데 queryKey가 같으면 캐시가 섞일 수 있다.

그래서 queryKey를 도메인별로 정리하고, 데이터 성격과 params를 key에 포함하는 방식으로 설계한다.

---

## queryKey를 중앙 관리하는 이유

queryKey를 각 hook이나 component 안에서 직접 작성하면 다음 문제가 생길 수 있다.

```txt
- 같은 데이터를 서로 다른 key로 호출할 수 있다.
- 오타가 발생해도 쉽게 찾기 어렵다.
- mutation 이후 invalidateQueries 범위를 정하기 어렵다.
- list, detail, history 같은 데이터 성격이 코드에서 명확히 드러나지 않는다.
```

예를 들어 여러 파일에서 직접 아래처럼 작성하면 관리가 어려워진다.

```ts
useQuery({
  queryKey: ["reports"],
  queryFn: ...
});
```

```ts
useQuery({
  queryKey: ["report", "list"],
  queryFn: ...
});
```

둘 다 신고 목록을 의미하더라도 React Query는 서로 다른 key로 인식한다.

그래서 Washer에서는 queryKey를 `shared/api/queryKeys.ts`에 모아두고, 각 query hook에서는 정의된 key 함수만 사용한다.

---

## 도메인별 queryKey 분리

현재 queryKey는 다음 도메인으로 분리되어 있다.

```ts
export const reportQueryKeys = {
  all: ["reports"] as const,
  getMalfunctionReports: (params?: ReportParamsType) =>
    ["reports", "list", params] as const,
  getReportById: (id: number) => ["reports", "detail", id] as const,
} as const;

export const machineQueryKeys = {
  all: ["machines"] as const,
  getMachines: (params: { floor?: number } = {}) =>
    ["machines", "list", params] as const,
} as const;

export const userQueryKeys = {
  all: ["users"] as const,
  getUsers: (params?: UserParamsType) => ["users", "list", params] as const,
  getMyInfo: () => ["users", "my"] as const,
} as const;

export const reservationQueryKeys = {
  all: ["reservations"] as const,
  getReservations: (params?: ReservationParamsType) =>
    ["reservations", "list", params] as const,
  getMachineReservationHistory: (machineName: string | null) =>
    ["reservations", "history", machineName] as const,
} as const;
```

도메인별로 queryKey를 분리하면 어떤 서버 상태가 어떤 도메인에 속하는지 명확해진다.

```txt
reports       → 고장 신고 데이터
machines      → 세탁기/건조기 데이터
users         → 사용자 데이터
reservations  → 예약 데이터
```

이 방식은 FSD 구조에서 `entities`를 도메인 기준으로 분리한 것과도 연결된다.
API 함수와 query hook이 도메인별로 나뉘어 있기 때문에 queryKey도 같은 기준으로 나누는 것이 자연스럽다.

---

## all prefix

각 도메인에는 `all` key가 있다.

```ts
export const reportQueryKeys = {
  all: ["reports"] as const,
  ...
} as const;
```

```ts
export const reservationQueryKeys = {
  all: ["reservations"] as const,
  ...
} as const;
```

`all`은 해당 도메인의 최상위 key 역할을 한다.

예를 들어 `reports` 도메인의 queryKey는 다음처럼 시작한다.

```txt
["reports"]
["reports", "list", params]
["reports", "detail", id]
```

예약 도메인은 다음처럼 시작한다.

```txt
["reservations"]
["reservations", "list", params]
["reservations", "history", machineName]
```

이렇게 최상위 prefix를 맞춰두면 mutation 이후 invalidate 범위를 잡기 쉽다.

예를 들어 신고 상태 변경 이후 신고 관련 데이터를 다시 가져와야 한다면 `reportQueryKeys.all` 기준으로 무효화할 수 있다.

```ts
queryClient.invalidateQueries({
  queryKey: reportQueryKeys.all,
});
```

이렇게 하면 `reports`로 시작하는 관련 query를 한 번에 갱신하는 기준을 만들 수 있다.

---

## list / detail / history / my 구분

queryKey에는 데이터 성격을 나타내는 값이 들어간다.

```txt
list     → 목록 데이터
detail   → 상세 데이터
history  → 이력 데이터
my       → 내 정보 데이터
```

예를 들어 신고 도메인은 목록과 상세를 구분한다.

```ts
getMalfunctionReports: (params?: ReportParamsType) =>
  ["reports", "list", params] as const,

getReportById: (id: number) =>
  ["reports", "detail", id] as const,
```

예약 도메인은 목록과 이력을 구분한다.

```ts
getReservations: (params?: ReservationParamsType) =>
  ["reservations", "list", params] as const,

getMachineReservationHistory: (machineName: string | null) =>
  ["reservations", "history", machineName] as const,
```

사용자 도메인은 사용자 목록과 내 정보를 구분한다.

```ts
getUsers: (params?: UserParamsType) =>
  ["users", "list", params] as const,

getMyInfo: () =>
  ["users", "my"] as const,
```

이렇게 나누는 이유는 같은 도메인 안에서도 데이터의 목적이 다르기 때문이다.

예를 들어 `reservations` 도메인 안에도 현재 예약 목록과 특정 기기의 예약 이력이 있다.
두 데이터는 같은 예약 도메인에 속하지만 화면에서 쓰이는 목적과 조회 조건이 다르기 때문에 key를 분리한다.

---

## params를 queryKey에 포함하는 이유

목록 조회 queryKey에는 params가 포함된다.

```ts
getMalfunctionReports: (params?: ReportParamsType) =>
  ["reports", "list", params] as const,
```

```ts
getReservations: (params?: ReservationParamsType) =>
  ["reservations", "list", params] as const,
```

```ts
getMachines: (params: { floor?: number } = {}) =>
  ["machines", "list", params] as const,
```

params를 queryKey에 포함하는 이유는 필터 조건별로 캐시를 분리하기 위해서이다.

예를 들어 신고 목록을 상태별로 조회한다고 하면 다음 queryKey는 서로 다른 캐시로 관리된다.

```txt
["reports", "list", { status: "PENDING" }]
["reports", "list", { status: "RESOLVED" }]
```

만약 params를 queryKey에 넣지 않으면 두 요청이 같은 key로 인식될 수 있다.

```txt
["reports", "list"]
```

그러면 `PENDING` 데이터를 가져온 뒤 `RESOLVED` 조건으로 조회해도 React Query가 같은 캐시라고 판단할 수 있다.
결과적으로 필터 조건과 맞지 않는 데이터가 화면에 보일 가능성이 생긴다.

따라서 서버 요청에 사용되는 params는 queryKey에도 포함하는 것이 기준이다.

---

## useGetMalfunctionReports의 queryKey 흐름

고장 신고 목록 조회 hook은 다음과 같다.

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

여기서 params는 두 곳에 사용된다.

```txt
1. queryKey 생성
2. API 요청 params
```

```ts
const queryKey = reportQueryKeys.getMalfunctionReports(params || {});
```

이 코드는 현재 조회 조건을 포함한 queryKey를 만든다.

```ts
queryFn: () => fetchMalfunctionReports(params);
```

이 코드는 같은 params를 실제 API 요청에도 전달한다.

즉, React Query 캐시 기준과 서버 요청 조건이 같은 params를 기준으로 맞춰지는 구조이다.

---

## useGetReservations의 queryKey 흐름

예약 목록 조회 hook도 같은 구조이다.

```ts
export const useGetReservations = (
  params?: ReservationParamsType,
  initialData?: ReservationItem[]
) => {
  const queryKey = reservationQueryKeys.getReservations(params || {});

  return useQuery({
    queryKey,
    queryFn: () => fetchReservations(params),
    initialData,
  });
};
```

예약 목록은 `reservationQueryKeys.getReservations(params || {})`를 사용한다.

```txt
["reservations", "list", params]
```

예약 목록도 params 조건에 따라 캐시가 분리된다.

예를 들어 기기 타입이나 상태 조건이 params에 들어간다면, 조건별로 다른 예약 목록 캐시가 만들어진다.

---

## useGetMachineReservationHistory의 queryKey 흐름

예약 이력 조회 hook은 `machineName`을 기준으로 queryKey를 만든다.

```ts
export const useGetMachineReservationHistory = (
  params?: MachineReservationHistoryParamsType,
  initialData?: MachineReservationHistory | null
) => {
  const machineName = params?.machineName ?? null;
  const queryKey =
    reservationQueryKeys.getMachineReservationHistory(machineName);

  return useQuery({
    queryKey,
    queryFn: () => fetchMachineReservationHistory(params),
    initialData,
    enabled: Boolean(machineName),
  });
};
```

예약 이력 queryKey는 다음 형태이다.

```ts
getMachineReservationHistory: (machineName: string | null) =>
  ["reservations", "history", machineName] as const,
```

즉, 특정 기기의 예약 이력은 `machineName` 기준으로 캐시가 분리된다.

```txt
["reservations", "history", "세탁기 1"]
["reservations", "history", "세탁기 2"]
```

이렇게 하면 서로 다른 기기의 예약 이력이 같은 캐시에 섞이지 않는다.

---

## enabled 옵션

`useGetMachineReservationHistory`에는 `enabled` 옵션이 있다.

```ts
enabled: Boolean(machineName),
```

이 옵션은 `machineName`이 있을 때만 query를 실행한다는 뜻이다.

예약 이력 모달은 사용자가 특정 기기를 선택한 뒤에 열리는 구조이다.
따라서 아직 선택된 기기가 없다면 이력 조회 API를 호출할 필요가 없다.

```txt
machineName 없음 → query 실행 안 함
machineName 있음 → query 실행
```

이 구조는 불필요한 API 요청을 막기 위한 처리이다.

---

## initialData

각 query hook은 `initialData`를 받을 수 있다.

```ts
initialData?: BaseResponseType<ReportResponseType>
```

```ts
initialData?: ReservationItem[]
```

```ts
initialData?: MachineReservationHistory | null
```

`initialData`는 React Query에서 초기 데이터로 사용할 값이다.

서버에서 미리 받은 데이터가 있거나, 이전 상태를 초기값으로 넣어야 할 때 사용할 수 있다.
현재 구조에서는 query hook이 초기 데이터 주입 가능성을 열어둔 형태이다.

---

## mutation 이후 invalidate 기준

queryKey를 도메인별로 정리해두면 mutation 이후 invalidate 기준을 잡기 쉽다.

예를 들어 예약을 삭제한 뒤에는 예약 목록과 예약 이력이 바뀔 수 있다.

이때 예약 도메인의 최상위 key를 기준으로 invalidate할 수 있다.

```ts
queryClient.invalidateQueries({
  queryKey: reservationQueryKeys.all,
});
```

또는 더 좁은 범위로 목록만 갱신할 수도 있다.

```ts
queryClient.invalidateQueries({
  queryKey: reservationQueryKeys.getReservations(params),
});
```

이렇게 queryKey를 체계적으로 구성하면 다음 기준을 선택할 수 있다.

```txt
전체 도메인 갱신
- reservationQueryKeys.all

목록만 갱신
- reservationQueryKeys.getReservations(params)

특정 기기 이력만 갱신
- reservationQueryKeys.getMachineReservationHistory(machineName)
```

즉, mutation 이후 어떤 데이터를 다시 가져와야 하는지 예측 가능해진다.

---

## as const를 사용하는 이유

queryKey 객체에는 `as const`가 붙어 있다.

```ts
export const reservationQueryKeys = {
  all: ["reservations"] as const,
  getReservations: (params?: ReservationParamsType) =>
    ["reservations", "list", params] as const,
  getMachineReservationHistory: (machineName: string | null) =>
    ["reservations", "history", machineName] as const,
} as const;
```

`as const`는 배열 안의 값을 일반 string 배열이 아니라, 고정된 tuple 타입으로 추론하게 만든다.

예를 들어 다음 값은 단순한 `string[]`이 아니라 고정된 queryKey 형태로 추론된다.

```ts
["reservations", "history", machineName] as const;
```

이렇게 하면 queryKey의 구조가 더 명확해지고, TypeScript가 key 형태를 더 좁은 타입으로 추론할 수 있다.

---

## 전체 구조 정리

현재 queryKey 구조는 다음 기준을 따른다.

```txt
도메인
  ↓
데이터 성격
  ↓
식별자 또는 params
```

예시는 다음과 같다.

```txt
["reports", "list", params]
["reports", "detail", id]

["machines", "list", params]

["users", "list", params]
["users", "my"]

["reservations", "list", params]
["reservations", "history", machineName]
```

이 구조는 React Query 캐시를 도메인별로 관리하기 위한 기준이다.

---

## 정리

Washer의 queryKey 설계 기준은 다음과 같다.

```txt
queryKey는 문자열로 직접 작성하지 않는다.
shared/api/queryKeys.ts에서 중앙 관리한다.
report, machine, user, reservation 도메인별로 분리한다.
각 도메인은 all prefix를 가진다.
목록은 list, 상세는 detail, 이력은 history, 내 정보는 my로 구분한다.
서버 요청 params는 queryKey에도 포함한다.
식별자가 필요한 데이터는 id나 machineName을 queryKey에 포함한다.
mutation 이후 invalidate 범위를 예측 가능하게 만든다.
```

결국 queryKey 설계의 핵심은 **서버 상태를 어떤 기준으로 캐싱하고 갱신할지 명확히 정하는 것**이다.

API 요청 params와 queryKey 기준을 맞춰두면 필터 조건별 캐시가 섞이지 않고, mutation 이후 필요한 데이터를 안정적으로 다시 가져올 수 있다.
