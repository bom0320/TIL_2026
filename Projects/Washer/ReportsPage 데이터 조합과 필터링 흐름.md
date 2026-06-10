# ReportsPage 데이터 조합과 필터링 흐름

## 핵심 흐름

ReportsPage는 고장 신고 목록을 보여주는 페이지이다.

이 페이지에서는 단순히 report 데이터만 조회하지 않고, machine 데이터도 함께 조회한다.
그 이유는 `floor` 필터링을 처리하기 위해서이다.

전체 흐름은 다음과 같다.

```txt
useGetMalfunctionReports
  → 고장 신고 목록 조회

useGetMachines
  → 층별 기기 목록 조회

reports + machines
  → floor 기준으로 데이터 조합

search
  → reporterName 기준 클라이언트 필터링

ReportsPanel
  → 신고 목록 표시

ReportFilterPanel
  → 필터 입력 UI
```

---

## ReportsPage의 상태

ReportsPage는 필터 상태를 직접 관리한다.

```ts
const [status, setStatus] = useState<ReportStatusType | undefined>();
const [search, setSearch] = useState("");
const [floor, setFloor] = useState<number | undefined>();
```

각 상태의 역할은 다음과 같다.

```txt
status
- 고장 신고 처리 상태 필터이다.
- PENDING, IN_PROGRESS, RESOLVED 값을 가진다.
- 서버 요청 params로 전달된다.

search
- 신고자 이름 검색어이다.
- 클라이언트에서 reporterName 기준으로 필터링한다.

floor
- 층 필터이다.
- useGetMachines의 params로 전달된다.
- 가져온 machine 목록과 report.machineId를 비교해 필터링한다.
```

이 페이지에서는 필터를 전부 같은 방식으로 처리하지 않는다.
각 필터의 성격에 따라 서버 요청 params로 처리할지, 클라이언트에서 처리할지 나눈다.

---

## status 필터링

status는 `useGetMalfunctionReports`에 params로 전달된다.

```ts
const {
  data: reportsData,
  isLoading: isReportsLoading,
  isError: isReportsError,
  refetch: refetchReports,
} = useGetMalfunctionReports({
  status,
});
```

이 구조에서 status는 서버 요청 조건이다.

즉, status가 바뀌면 `useGetMalfunctionReports`의 queryKey도 바뀌고, API 요청 params도 바뀐다.

```txt
status 변경
  ↓
useGetMalfunctionReports({ status })
  ↓
queryKey 변경
  ↓
고장 신고 목록 재조회
```

status 필터는 서버에서 처리할 수 있는 조건이기 때문에 클라이언트에서 전체 목록을 받은 뒤 필터링하지 않고 API params로 전달한다.

---

## useGetMalfunctionReports

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

이 hook은 params를 queryKey와 queryFn에 모두 사용한다.

```txt
params → queryKey
params → API 요청 params
```

따라서 status가 달라지면 캐시도 분리된다.

```txt
["reports", "list", { status: "PENDING" }]
["reports", "list", { status: "RESOLVED" }]
```

이 구조는 상태별 신고 목록이 같은 캐시에 섞이지 않도록 하기 위한 구조이다.

---

## floor 필터링

floor는 report API에 직접 전달하지 않고, machine API에 전달한다.

```ts
const { data: machinesData, isLoading: isMachinesLoading } = useGetMachines({
  floor,
});
```

`useGetMachines`는 다음 구조이다.

```ts
export const useGetMachines = (params: { floor?: number } = {}) => {
  return useQuery({
    queryKey: machineQueryKeys.getMachines(params),
    queryFn: () =>
      get<BaseResponseType<MachineResponseType>>(machineUrl.getMachines(), {
        params,
      }),
  });
};
```

즉, floor가 선택되면 해당 층의 machine 목록을 가져온다.

```txt
floor 선택
  ↓
useGetMachines({ floor })
  ↓
해당 층에 있는 machine 목록 조회
```

---

## report와 machine 데이터 조합

ReportsPage에서는 report 데이터와 machine 데이터를 각각 꺼낸다.

```ts
const reports = reportsData?.data.reports ?? [];
const machines = machinesData?.data.machines ?? [];
```

report에는 `machineId`가 있고, machine에는 `id`가 있다.

```txt
report.machineId
machine.id
```

floor 필터가 선택된 경우, 현재 floor에 해당하는 machine id 목록을 만든다.

```ts
const machineIdsOnFloor = new Set(machines.map((m) => m.id));
```

그 다음 report의 `machineId`가 해당 Set에 포함되는지 확인한다.

```ts
result = result.filter((report) => machineIdsOnFloor.has(report.machineId));
```

즉, floor 필터링 흐름은 다음과 같다.

```txt
선택된 floor로 machine 목록 조회
  ↓
machine.id 목록을 Set으로 변환
  ↓
report.machineId가 Set에 포함되는지 확인
  ↓
해당 층의 신고만 남김
```

Report API 응답만으로는 층 정보를 직접 판단하기 어렵기 때문에 machine 데이터와 조합하는 방식이다.

---

## search 필터링

search는 클라이언트에서 처리한다.

```ts
if (search) {
  result = result.filter((report) =>
    report.reporterName.toLowerCase().includes(search.toLowerCase())
  );
}
```

검색 기준은 `reporterName`이다.

```txt
search 입력값
  ↓
report.reporterName과 비교
  ↓
포함되는 신고만 필터링
```

`toLowerCase()`를 사용해서 대소문자 차이로 검색이 실패하지 않도록 처리한다.

search는 사용자가 입력할 때마다 즉시 반영되는 화면 필터이다.
따라서 API 요청을 매번 보내기보다, 이미 조회된 신고 목록에서 클라이언트 필터링으로 처리한다.

---

## useMemo를 사용하는 이유

필터링 결과는 `useMemo`로 계산한다.

```ts
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

`filteredReports`는 다음 값이 바뀔 때만 다시 계산된다.

```txt
reports
machines
floor
search
```

필터링은 렌더링 때마다 새로 계산할 필요가 없고, 관련 데이터나 필터 상태가 바뀔 때만 다시 계산하면 된다.

그래서 `useMemo`를 사용한다.

---

## loading 상태 조합

ReportsPage에서는 report 요청과 machine 요청이 동시에 사용된다.

```ts
const isLoading = isReportsLoading || isMachinesLoading;
```

신고 목록을 표시하려면 report 데이터가 필요하고, floor 필터링까지 정확히 적용하려면 machine 데이터도 필요하다.

따라서 둘 중 하나라도 loading 상태이면 전체 패널을 loading으로 처리한다.

```txt
report loading
machine loading
  ↓
isLoading = true
```

---

## error 처리

error 상태는 report 요청을 기준으로 처리한다.

```ts
isError = { isReportsError };
onRetry = { refetchReports };
```

ReportsPanel에는 `isReportsError`와 `refetchReports`를 전달한다.

```tsx
<ReportsPanel
  title="고장 신고 관리"
  reports={filteredReports}
  variant="detail"
  isLoading={isLoading}
  isError={isReportsError}
  onRetry={refetchReports}
/>
```

고장 신고 목록 자체를 불러오지 못하면 화면에 표시할 핵심 데이터가 없기 때문에 오류 상태를 보여준다.

재시도 버튼을 누르면 `refetchReports`를 실행한다.

---

## reset 흐름

필터 초기화는 `handleReset`에서 처리한다.

```ts
const handleReset = () => {
  setStatus(undefined);
  setSearch("");
  setFloor(undefined);
};
```

초기화 대상은 세 가지이다.

```txt
status
search
floor
```

초기화 버튼을 누르면 status, search, floor가 모두 기본값으로 돌아간다.

```txt
status → undefined
search → ""
floor → undefined
```

그 결과 status와 floor 조건이 없는 전체 목록 기준으로 다시 조회/필터링된다.

---

## ReportFilterPanel의 역할

ReportFilterPanel은 필터 입력 UI를 담당한다.

```tsx
<ReportFilterPanel
  status={status}
  onStatusChange={setStatus}
  search={search}
  onSearchChange={setSearch}
  floor={floor}
  onFloorChange={setFloor}
  onReset={handleReset}
/>
```

이 컴포넌트는 상태를 직접 소유하지 않는다.
상태 값과 변경 함수를 props로 받아서 사용한다.

```txt
status
onStatusChange

search
onSearchChange

floor
onFloorChange

onReset
```

즉, ReportFilterPanel은 controlled component처럼 동작한다.

---

## status FilterChip

상태 필터는 `FilterChip`으로 구성되어 있다.

```tsx
<FilterChip
  label="대기"
  minWidthClass="min-w-[62px]"
  active={status === "PENDING"}
  onClick={() => onStatusChange(status === "PENDING" ? undefined : "PENDING")}
/>
```

동작 방식은 다음과 같다.

```txt
현재 선택된 status와 같은 chip 클릭
→ undefined로 변경
→ 필터 해제

현재 선택되지 않은 chip 클릭
→ 해당 status로 변경
→ 필터 적용
```

예를 들어 `"PENDING"` chip을 다시 누르면 필터가 해제된다.

```txt
PENDING 선택 상태
  ↓
PENDING 다시 클릭
  ↓
status = undefined
```

이 구조는 같은 필터를 다시 눌렀을 때 토글되도록 만든 방식이다.

---

## floor Filter

floor 필터는 공통 필터 컴포넌트인 `FloorGenderFilters`를 사용한다.

```tsx
<FloorGenderFilters selectedFloor={floor} onFloorChange={onFloorChange} />
```

ReportsPage에서는 floor 값을 직접 관리하고, ReportFilterPanel은 그 값을 `FloorGenderFilters`에 전달한다.

```txt
ReportsPage
  ↓
ReportFilterPanel
  ↓
FloorGenderFilters
```

floor가 변경되면 `setFloor`가 실행되고, `useGetMachines({ floor })`가 새로운 조건으로 동작한다.

---

## ReportsPanel의 역할

ReportsPanel은 신고 목록을 화면에 표시하는 컴포넌트이다.

```tsx
<ReportsPanel
  title="고장 신고 관리"
  reports={filteredReports}
  variant="detail"
  isLoading={isLoading}
  isError={isReportsError}
  onRetry={refetchReports}
/>
```

ReportsPanel은 필터링 로직을 알지 않는다.
이미 필터링된 `reports` 배열을 받아서 렌더링한다.

ReportsPanel이 담당하는 것은 다음과 같다.

```txt
- 패널 제목 표시
- loading 상태 표시
- error 상태 표시
- 빈 목록 상태 표시
- 신고 목록 렌더링
- retry 버튼 처리
```

데이터 조회나 필터 계산은 ReportsPage가 담당하고, ReportsPanel은 표시 역할만 담당한다.

---

## ReportsPanel의 상태별 렌더링

ReportsPanel은 상태에 따라 다른 UI를 보여준다.

### loading

```tsx
{
  isLoading && (
    <div className="flex h-[200px] items-center justify-center">
      <p className="text-sm text-gray-500 font-medium animate-pulse">
        데이터를 불러오는 중...
      </p>
    </div>
  );
}
```

데이터를 불러오는 중이면 loading 문구를 보여준다.

### error

```tsx
{
  isError && (
    <div className="flex h-[200px] flex-col items-center justify-center gap-4">
      <p className="text-sm text-red-500 font-medium">
        데이터를 불러오는 중 오류가 발생했습니다.
      </p>
      {onRetry && (
        <Button variant="outline" size="sm" onClick={onRetry}>
          다시 시도
        </Button>
      )}
    </div>
  );
}
```

오류가 발생하면 에러 문구와 다시 시도 버튼을 보여준다.

### empty

```tsx
{
  !isLoading && !isError && reports.length === 0 && (
    <div className="flex h-[200px] items-center justify-center">
      <p className="text-sm text-gray-500 font-medium">신고 내역이 없습니다.</p>
    </div>
  );
}
```

요청은 성공했지만 표시할 신고가 없으면 빈 상태를 보여준다.

### success

```tsx
{
  !isLoading && !isError && reports.length > 0 && (
    <div className="sidebar-scrollbar max-h-full overflow-y-auto divide-y divide-[#E9E9EE]">
      {reports.map((item) => (
        <ReportRow key={item.id} item={item} variant={variant} />
      ))}
    </div>
  );
}
```

데이터가 있으면 신고 목록을 렌더링한다.

---

## ReportRow의 역할

ReportRow는 신고 하나를 표시한다.

```tsx
const ReportRow = ({
  item,
  variant,
}: {
  item: ReportItemType;
  variant: "summary" | "detail";
}) => {
  return (
    <div className="flex items-start justify-between gap-4 py-4 first:pt-0 last:pb-0">
      ...
    </div>
  );
};
```

`variant`에 따라 표시하는 정보가 달라진다.

```txt
summary
- 신고자 이름
- 신고 시간

detail
- 신고자 이름
- 신고 사유
```

summary 모드는 대시보드 같은 요약 영역에서 사용할 수 있고, detail 모드는 고장 신고 관리 페이지에서 사용할 수 있다.

---

## ReportMachineIcon의 역할

ReportMachineIcon은 기기 이름을 기준으로 세탁기/건조기 아이콘을 선택한다.

```tsx
const ReportMachineIcon = ({ machineName }: { machineName: string }) => {
  const isWasher = machineName.toUpperCase().includes("WASHER");
  const src = isWasher ? "/icons/washer-drop.svg" : "/icons/dryer-wave.svg";

  return (
    <div className="flex h-10 w-10 shrink-0 items-center justify-center translate-y-0.5">
      <Image
        src={src}
        alt={isWasher ? "WASHER" : "DRYER"}
        width={28}
        height={28}
      />
    </div>
  );
};
```

기기명이 `"WASHER"`를 포함하면 세탁기 아이콘을 사용하고, 그렇지 않으면 건조기 아이콘을 사용한다.

```txt
machineName includes WASHER
→ washer-drop.svg

그 외
→ dryer-wave.svg
```

이 로직은 신고 데이터에 기기 타입 필드가 없거나, 화면에서 이름 기준으로 간단히 분기할 수 있을 때 사용하는 방식이다.

---

## ReportStatusBadge

ReportRow에서는 신고 상태를 `ReportStatusBadge`로 표시한다.

```tsx
<ReportStatusBadge status={item.status} />
```

ReportRow는 상태값을 직접 label이나 색상으로 변환하지 않는다.
상태 표시 책임은 `ReportStatusBadge`가 가진다.

이렇게 하면 row 컴포넌트가 상태 스타일 매핑까지 알 필요가 없다.

---

## 책임 분리

ReportsPage 관련 컴포넌트의 책임은 다음과 같다.

| 파일                                              | 역할                                                      |
| ------------------------------------------------- | --------------------------------------------------------- |
| `widgets/reports-page/index.tsx`                  | 데이터 조회, 필터 상태 관리, report + machine 데이터 조합 |
| `widgets/reports-page/ui/ReportFilterPanel.tsx`   | 검색, 상태, 층 필터 입력 UI                               |
| `widgets/reports-page/ui/ReportsPanel.tsx`        | 신고 목록 표시, loading/error/empty 처리                  |
| `entities/report/api/useGetMalfunctionReports.ts` | 신고 목록 서버 상태 관리                                  |
| `entities/machine/api/useGetMachines.ts`          | 기기 목록 서버 상태 관리                                  |

---

## 필터 처리 기준 정리

ReportsPage의 필터는 다음 기준으로 나뉜다.

| 필터   | 처리 위치                            | 이유                                                                  |
| ------ | ------------------------------------ | --------------------------------------------------------------------- |
| status | 서버 params                          | 신고 API에서 상태 기준 조회가 가능함                                  |
| floor  | machine API params + 클라이언트 조합 | report 데이터에는 floor 기준이 직접 없고 machine 데이터와 조합해야 함 |
| search | 클라이언트 필터링                    | reporterName 기준 즉시 검색 처리에 적합함                             |

---

## 정리

ReportsPage는 고장 신고 목록을 query hook으로 조회하고, machine 데이터와 조합해 floor 필터링을 처리하는 구조이다.

핵심은 다음과 같다.

```txt
useGetMalfunctionReports로 신고 데이터를 가져온다.
useGetMachines로 층별 machine 데이터를 가져온다.
status는 신고 API params로 전달한다.
floor는 machine 데이터의 id와 report.machineId를 비교해 필터링한다.
search는 reporterName 기준으로 클라이언트에서 필터링한다.
ReportsPanel은 신고 목록 표시를 담당한다.
ReportFilterPanel은 필터 입력 UI를 담당한다.
```

이 구조는 서버에서 처리할 필터와 클라이언트에서 처리할 필터를 분리하고, report 데이터와 machine 데이터를 조합해 화면 요구사항에 맞게 목록을 구성하는 방식이다.
