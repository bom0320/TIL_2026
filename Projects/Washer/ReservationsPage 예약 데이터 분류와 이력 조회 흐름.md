# ReservationsPage 예약 데이터 분류와 이력 조회 흐름

## 핵심 흐름

ReservationsPage는 현재 활성화된 예약 현황을 보여주고, 특정 기기의 예약 이력을 모달로 확인하는 페이지이다.

전체 흐름은 다음과 같다.

```txt
useGetReservations
  → 예약 목록 조회

mapReservations
  → 서버 DTO를 화면용 ReservationItem으로 변환

ReservationsPage
  → WASHER / DRYER 기준으로 데이터 분리

ReservationStatusPanel
  → 예약 현황 표시

ReservationRow
  → 개별 예약 row 표시
  → 이력 열기
  → 예약 강제 취소

ReservationHistoryModal
  → machineName 기준 예약 이력 조회

useGetMachineReservationHistory
  → 특정 기기의 예약 이력 서버 상태 관리
```

ReservationsPage의 핵심은 예약 데이터를 기기 타입별로 분류하고, 사용자가 특정 기기의 이력을 열었을 때 `machineName` 기준으로 이력 데이터를 조회하는 것이다.

---

## ReservationsPage의 상태

ReservationsPage는 예약 이력 모달 상태를 관리한다.

```ts
type HistoryOverlayState = {
  machineName: string;
  side: "left" | "right";
} | null;
```

```ts
const [historyOverlay, setHistoryOverlay] = useState<HistoryOverlayState>(null);
```

`historyOverlay`는 현재 열려 있는 이력 모달 정보를 나타낸다.

```txt
machineName
- 이력을 조회할 기기 이름이다.

side
- 모달이 어느 패널 옆에 열릴지 결정하는 값이다.
- left 또는 right 값을 가진다.

null
- 모달이 닫힌 상태이다.
```

즉, 이력 모달은 단순히 열림/닫힘만 관리하는 것이 아니라, 어떤 기기의 이력을 어느 방향에 표시할지도 함께 관리한다.

---

## 예약 목록 조회

예약 목록은 `useGetReservations`로 조회한다.

```ts
const { data, isLoading, isError } = useGetReservations();
```

이 hook은 예약 목록 서버 상태를 관리한다.

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

`useGetReservations`는 내부적으로 `getReservations` API function을 호출한다.

```ts
export async function getReservations(
  params?: ReservationParamsType
): Promise<ReservationItem[]> {
  const response = await get<BaseResponseType<ReservationResponseType>>(
    reservationUrl.getReservations(),
    {
      params,
    }
  );

  return mapReservations(response.data.reservations);
}
```

여기서 중요한 점은 API 응답을 그대로 반환하지 않고 `mapReservations`를 거쳐 `ReservationItem[]`으로 변환한다는 것이다.

즉, ReservationsPage는 서버 DTO가 아니라 화면에서 사용하기 좋은 형태의 예약 데이터를 받는다.

---

## loading / error 처리

ReservationsPage는 예약 목록을 가져오는 동안 loading 상태를 보여준다.

```ts
if (isLoading) {
  return <div>불러오는 중...</div>;
}
```

요청이 실패하면 error 상태를 보여준다.

```ts
if (isError) {
  return <div>데이터를 불러오지 못했습니다.</div>;
}
```

이 페이지에서는 예약 목록이 핵심 데이터이므로, 목록 조회가 끝나기 전에는 패널을 렌더링하지 않는다.

---

## 예약 데이터 기본값 처리

조회가 끝나면 data를 reservations로 사용한다.

```ts
const reservations = data ?? [];
```

`data`가 없을 경우 빈 배열을 기본값으로 사용한다.

이렇게 하면 이후 `filter`, `map`을 사용할 때 undefined 에러를 방지할 수 있다.

---

## WASHER / DRYER 기준 데이터 분리

예약 데이터는 기기 타입 기준으로 분리한다.

```ts
const dryerReservations = reservations.filter((item) => item.type === "DRYER");

const washerReservations = reservations.filter(
  (item) => item.type === "WASHER"
);
```

`item.type`은 서버 DTO에 그대로 있던 값이 아니라 `mapReservation`에서 만들어진 화면용 값이다.

```txt
ReservationDTO
  ↓
mapReservation
  ↓
ReservationItem.type
```

이렇게 변환된 `type`을 기준으로 건조기와 세탁기 데이터를 나눈다.

```txt
DRYER
→ 건조기 예약 현황 패널

WASHER
→ 세탁기 예약 현황 패널
```

---

## 화면 구조

ReservationsPage는 두 개의 예약 현황 패널을 렌더링한다.

```tsx
<div className="admin-page-grid">
  <div className="relative admin-page-item">
    <ReservationStatusPanel
      title="건조기 예약 현황"
      icon={<Waves size={18} className="translate-y-px text-[#A4A4AA]" />}
      reservations={dryerReservations}
      onOpenHistory={(machineName) =>
        setHistoryOverlay({ machineName, side: "right" })
      }
    />

    {historyOverlay?.side === "right" && (
      <ReservationHistoryModal
        machineName={historyOverlay.machineName}
        side="right"
        onClose={() => setHistoryOverlay(null)}
      />
    )}
  </div>

  <div className="relative admin-page-item">
    <ReservationStatusPanel
      title="세탁기 예약 현황"
      icon={<Droplet size={18} className="translate-y-px text-[#A4A4AA]" />}
      reservations={washerReservations}
      onOpenHistory={(machineName) =>
        setHistoryOverlay({ machineName, side: "left" })
      }
    />

    {historyOverlay?.side === "left" && (
      <ReservationHistoryModal
        machineName={historyOverlay.machineName}
        side="left"
        onClose={() => setHistoryOverlay(null)}
      />
    )}
  </div>
</div>
```

건조기 패널과 세탁기 패널은 같은 `ReservationStatusPanel`을 재사용한다.
차이는 title, icon, reservations, modal side 값이다.

---

## 건조기 예약 현황 패널

건조기 예약 현황 패널에는 `dryerReservations`를 전달한다.

```tsx
<ReservationStatusPanel
  title="건조기 예약 현황"
  icon={<Waves size={18} className="translate-y-px text-[#A4A4AA]" />}
  reservations={dryerReservations}
  onOpenHistory={(machineName) =>
    setHistoryOverlay({ machineName, side: "right" })
  }
/>
```

건조기 패널에서 이력을 열면 `side`는 `"right"`로 저장된다.

```txt
건조기 패널에서 이력 열기
  ↓
setHistoryOverlay({ machineName, side: "right" })
```

이 값에 따라 모달이 건조기 패널 오른쪽에 열린다.

---

## 세탁기 예약 현황 패널

세탁기 예약 현황 패널에는 `washerReservations`를 전달한다.

```tsx
<ReservationStatusPanel
  title="세탁기 예약 현황"
  icon={<Droplet size={18} className="translate-y-px text-[#A4A4AA]" />}
  reservations={washerReservations}
  onOpenHistory={(machineName) =>
    setHistoryOverlay({ machineName, side: "left" })
  }
/>
```

세탁기 패널에서 이력을 열면 `side`는 `"left"`로 저장된다.

```txt
세탁기 패널에서 이력 열기
  ↓
setHistoryOverlay({ machineName, side: "left" })
```

이 값에 따라 모달이 세탁기 패널 왼쪽에 열린다.

---

## ReservationStatusPanel의 역할

ReservationStatusPanel은 예약 현황 목록을 표시하는 컴포넌트이다.

```tsx
export default function ReservationStatusPanel({
  title,
  icon,
  reservations,
  onOpenHistory,
}: ReservationStatusPanelProps) {
  return (
    <StatusPanelShell title={title} icon={icon}>
      {reservations.length === 0 ? (
        <div className="flex h-full items-center justify-center text-lg text-[#9A9AA0]">
          현재 활성화된 예약이 없습니다.
        </div>
      ) : (
        reservations.map((item) => (
          <ReservationRow
            key={item.id}
            item={item}
            onOpenHistory={onOpenHistory}
          />
        ))
      )}
    </StatusPanelShell>
  );
}
```

이 컴포넌트는 다음 책임을 가진다.

```txt
- 패널 제목과 아이콘 표시
- 예약 목록이 없을 때 empty message 표시
- 예약 목록이 있을 때 ReservationRow 렌더링
```

ReservationStatusPanel은 데이터를 직접 조회하지 않는다.
이미 분류된 `reservations` 배열을 props로 받아서 표시한다.

---

## ReservationRow의 역할

ReservationRow는 예약 하나를 표시하는 컴포넌트이다.

```tsx
export default function ReservationRow({
  item,
  onOpenHistory,
}: ReservationRowProps) {
  const { mutate: deleteReservation, isPending } = useDeleteReservation();

  const remainText = useRemainingTime(item.expectedCompletionTime);
  const expiredText = useRemainingTime(item.startTime);

  const handleHistory = () => {
    onOpenHistory(item.machine);
  };

  const handleDelete = () => {
    const confirmed = window.confirm("이 예약을 강제 취소하시겠습니까?");
    if (!confirmed) return;

    deleteReservation(item.id);
  };

  return (...)
}
```

ReservationRow는 다음 흐름을 담당한다.

```txt
- 예약 상태별 정보 표시
- 남은 시간 계산
- 예약 만료 시간 계산
- 이력 버튼 클릭 처리
- 예약 강제 취소 처리
```

---

## 예약 상태별 표시

ReservationRow는 `badgeStatus`에 따라 다른 정보를 보여준다.

### 사용중

```tsx
{
  item.badgeStatus === "사용중" && (
    <>
      <p className="mt-1 text-sm text-[#969696]">남은 시간: {remainText}</p>
      {item.deviceStatus && (
        <p className="mt-1 text-sm text-[#969696]">
          기기 상태: {item.deviceStatus}
        </p>
      )}
    </>
  );
}
```

사용중 상태에서는 남은 시간과 기기 상태를 보여준다.

```txt
사용중
- 남은 시간
- 기기 상태
```

### 예약중

```tsx
{
  item.badgeStatus === "예약중" && (
    <>
      <p className="mt-1 text-sm text-[#969696]">예약 시간: {item.reserveAt}</p>
      <p className="mt-1 text-sm text-[#EA3B42]">
        예약 만료까지: {expiredText}
      </p>
    </>
  );
}
```

예약중 상태에서는 예약 시간과 예약 만료까지 남은 시간을 보여준다.

```txt
예약중
- 예약 시간
- 예약 만료까지 남은 시간
```

### 확인필요

```tsx
{
  item.badgeStatus === "확인필요" && (
    <p className="mt-1 text-sm text-[#EA3B42]">
      기기를 현재 사용할 수 없습니다.
    </p>
  );
}
```

확인필요 상태에서는 기기를 사용할 수 없다는 안내를 보여준다.

```txt
확인필요
- 기기를 현재 사용할 수 없음
```

---

## useRemainingTime

ReservationRow에서는 `useRemainingTime`을 사용한다.

```ts
const remainText = useRemainingTime(item.expectedCompletionTime);
const expiredText = useRemainingTime(item.startTime);
```

각 값의 의미는 다르다.

```txt
expectedCompletionTime
- 사용중인 예약의 예상 완료 시간이다.
- 남은 시간을 계산하는 기준이다.

startTime
- 예약된 사용 시작 시간이다.
- 예약 만료까지 남은 시간을 계산하는 기준이다.
```

즉, 같은 hook을 사용하지만 상태에 따라 다른 시간 기준을 넣는다.

---

## 이력 열기 흐름

ReservationRow에서 이력 버튼을 누르면 `handleHistory`가 실행된다.

```ts
const handleHistory = () => {
  onOpenHistory(item.machine);
};
```

`item.machine`은 mapper에서 `machineName`을 화면용으로 정리한 값이다.

이 값이 상위 컴포넌트로 전달된다.

```txt
ReservationRow
  ↓
onOpenHistory(item.machine)
  ↓
ReservationStatusPanel
  ↓
ReservationsPage
  ↓
setHistoryOverlay({ machineName, side })
```

결과적으로 ReservationsPage의 `historyOverlay` 상태가 설정되고, 해당 위치에 `ReservationHistoryModal`이 열린다.

---

## 예약 강제 취소 흐름

ReservationRow에서는 예약 강제 취소도 처리한다.

```ts
const { mutate: deleteReservation, isPending } = useDeleteReservation();
```

삭제 버튼을 누르면 먼저 confirm을 띄운다.

```ts
const handleDelete = () => {
  const confirmed = window.confirm("이 예약을 강제 취소하시겠습니까?");
  if (!confirmed) return;

  deleteReservation(item.id);
};
```

사용자가 확인하면 `deleteReservation(item.id)`를 실행한다.

```txt
삭제 버튼 클릭
  ↓
confirm
  ↓
deleteReservation(item.id)
  ↓
DELETE /api/v2/admin/reservations/{id}
```

삭제 요청이 진행 중일 때는 `isPending`을 기준으로 액션을 비활성화한다.

```tsx
<StatusRowActions
  badge={<ReservationStatusBadge label={item.badgeStatus} />}
  onHistory={handleHistory}
  onDelete={handleDelete}
  disabled={isPending}
/>
```

---

## useDeleteReservation

예약 삭제 mutation hook은 다음과 같다.

```ts
export const useDeleteReservation = () => {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: deleteReservation,
    onSuccess: () => {
      queryClient.invalidateQueries({
        queryKey: reservationQueryKeys.all,
      });
    },
  });
};
```

삭제 API function은 다음과 같다.

```ts
export async function deleteReservation(id: number) {
  await del(reservationUrl.deleteReservation(id));
}
```

삭제 성공 이후에는 `reservationQueryKeys.all` 기준으로 예약 도메인의 query를 invalidate한다.

```txt
예약 삭제 성공
  ↓
reservationQueryKeys.all invalidate
  ↓
예약 목록 / 예약 이력 관련 데이터 재조회 대상
```

예약 삭제는 현재 목록뿐 아니라 이력 데이터에도 영향을 줄 수 있으므로 reservation 도메인 전체를 갱신하는 방식이다.

---

## ReservationHistoryModal 렌더링 조건

ReservationsPage에서는 `historyOverlay.side`에 따라 모달을 렌더링한다.

```tsx
{
  historyOverlay?.side === "right" && (
    <ReservationHistoryModal
      machineName={historyOverlay.machineName}
      side="right"
      onClose={() => setHistoryOverlay(null)}
    />
  );
}
```

```tsx
{
  historyOverlay?.side === "left" && (
    <ReservationHistoryModal
      machineName={historyOverlay.machineName}
      side="left"
      onClose={() => setHistoryOverlay(null)}
    />
  );
}
```

즉, 모달은 항상 하나만 열리고, 현재 선택된 패널의 side에 따라 위치가 달라진다.

```txt
side === "right"
→ 오른쪽 방향 모달

side === "left"
→ 왼쪽 방향 모달
```

---

## ReservationHistoryModal의 역할

ReservationHistoryModal은 특정 기기의 예약 이력을 보여주는 컴포넌트이다.

```tsx
export default function ReservationHistoryModal({
  machineName,
  onClose,
  side = "right",
}: ReservationHistoryModalProps) {
  const isOpen = Boolean(machineName);
  const panelRef = useRef<HTMLDivElement>(null);

  const { data, isLoading, isError } = useGetMachineReservationHistory(
    machineName ? { machineName } : undefined,
  );

  useOutsideClick(panelRef, onClose, isOpen);

  if (!isOpen || !machineName) return null;

  ...
}
```

이 컴포넌트의 핵심 책임은 다음과 같다.

```txt
- machineName 기준 예약 이력 조회
- 모달 열림 여부 판단
- side 기준 위치 클래스 결정
- outside click으로 닫기 처리
- loading/error/empty/success 상태별 UI 표시
```

---

## machineName 기준 이력 조회

이력 조회는 `machineName`을 params로 전달한다.

```ts
const { data, isLoading, isError } = useGetMachineReservationHistory(
  machineName ? { machineName } : undefined
);
```

`machineName`이 있으면 이력 조회를 실행하고, 없으면 undefined를 넘긴다.

```txt
machineName 있음
→ useGetMachineReservationHistory({ machineName })

machineName 없음
→ useGetMachineReservationHistory(undefined)
```

---

## useGetMachineReservationHistory

예약 이력 조회 hook은 다음과 같다.

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

이 hook은 `machineName`을 queryKey에 포함한다.

```txt
["reservations", "history", machineName]
```

따라서 기기별 예약 이력이 서로 다른 캐시로 관리된다.

```txt
["reservations", "history", "WASHER-1"]
["reservations", "history", "DRYER-1"]
```

또한 `enabled` 옵션이 있다.

```ts
enabled: Boolean(machineName);
```

`machineName`이 없으면 query가 실행되지 않는다.
이력 모달은 특정 기기를 선택해야 의미가 있으므로, 선택된 기기가 없을 때 API 요청을 막는 구조이다.

---

## getMachineReservationHistory

예약 이력 API function은 다음과 같다.

```ts
export async function getMachineReservationHistory(
  params?: MachineReservationHistoryParamsType
): Promise<MachineReservationHistory | null> {
  const response = await get<
    BaseResponseType<MachineReservationHistoryResponseType>
  >(reservationUrl.getMachineReservationHistory(), {
    params,
  });

  const machine = response.data.machines[0];

  if (!machine) return null;

  return mapMachineReservationHistory(machine);
}
```

서버 응답은 `machines` 배열로 내려온다.
하지만 모달에서는 선택된 기기 하나의 이력만 필요하다.

그래서 첫 번째 machine을 꺼낸다.

```ts
const machine = response.data.machines[0];
```

기기 데이터가 없으면 `null`을 반환한다.

```ts
if (!machine) return null;
```

기기 데이터가 있으면 `mapMachineReservationHistory`로 UI에서 사용할 형태로 변환한다.

```ts
return mapMachineReservationHistory(machine);
```

---

## 모달 위치 처리

ReservationHistoryModal은 `side` 값에 따라 위치 클래스를 결정한다.

```ts
const overlayPositionClass =
  side === "right"
    ? "absolute left-[calc(100%+16px)] top-0 h-full w-full"
    : "absolute right-[calc(100%+16px)] top-0 h-full w-full";
```

```txt
side === "right"
→ 현재 패널의 오른쪽에 모달 표시

side === "left"
→ 현재 패널의 왼쪽에 모달 표시
```

이 구조는 건조기 패널과 세탁기 패널이 같은 모달 컴포넌트를 사용하면서도, 각각 다른 방향으로 열릴 수 있게 한다.

---

## outside click 처리

모달은 바깥 영역을 클릭하면 닫힌다.

```ts
const panelRef = useRef<HTMLDivElement>(null);

useOutsideClick(panelRef, onClose, isOpen);
```

`panelRef`는 모달 내부 패널을 가리킨다.
`useOutsideClick`은 열려 있는 상태에서 패널 외부를 클릭했을 때 `onClose`를 실행한다.

```txt
모달 열림
  ↓
패널 외부 클릭
  ↓
onClose()
  ↓
setHistoryOverlay(null)
```

---

## ReservationHistoryModal 상태별 렌더링

이력 모달도 query 상태에 따라 다른 UI를 보여준다.

### loading

```tsx
{
  isLoading && <p className="py-8 text-sm text-[#9A9AA0]">불러오는 중...</p>;
}
```

이력 데이터를 불러오는 중이면 loading 문구를 보여준다.

### error

```tsx
{
  isError && (
    <p className="py-8 text-sm text-[#EA3B42]">
      히스토리를 불러오지 못했습니다.
    </p>
  );
}
```

이력 조회에 실패하면 error 문구를 보여준다.

### empty

```tsx
{
  !isLoading && !isError && (!data || data.reservations.length === 0) && (
    <p className="py-8 text-sm text-[#9A9AA0]">히스토리가 없습니다.</p>
  );
}
```

요청은 성공했지만 데이터가 없으면 empty 상태를 보여준다.

### success

```tsx
{
  !isLoading && !isError && data && data.reservations.length > 0 && (
    <div className="flex flex-col gap-4">
      {data.reservations.map((item, index) => (
        <ReservationHistoryCard
          key={`${item.roomNumber}-${item.reservedAt}-${index}`}
          machineName={data.machineName}
          item={item}
        />
      ))}
    </div>
  );
}
```

이력 데이터가 있으면 `ReservationHistoryCard` 목록을 렌더링한다.

---

## ReservationHistoryCard의 역할

ReservationHistoryCard는 예약 이력 하나를 표시한다.

```tsx
export default function ReservationHistoryCard({
  machineName,
  item,
}: ReservationHistoryCardProps) {
  return (
    <div className="rounded-xl border border-[#D4D4D8] bg-white px-4 py-4">
      <div className="mb-3 flex items-center justify-between">
        <p className="text-[15px] font-medium text-[#4A4A4F]">{machineName}</p>
        <ReservationStatusBadge label={item.status} />
      </div>

      <div className="border-t border-[#ECECEC] pt-3">
        <div className="grid grid-cols-[90px_1fr] gap-y-2 text-[14px] text-[#555]">
          <span>예약호실</span>
          <span className="text-right text-[#8B8B8B]">{item.roomNumber}호</span>

          <span>예약시간</span>
          <span className="text-right text-[#8B8B8B]">{item.reservedAt}</span>

          <span>{item.status === "취소됨" ? "취소 시간" : "완료 시간"}</span>
          <span className="text-right text-[#8B8B8B]">
            {item.actionAt ?? "-"}
          </span>
        </div>
      </div>
    </div>
  );
}
```

이 컴포넌트는 이미 mapper에서 정리된 이력 데이터를 받아서 표시한다.

```txt
roomNumber
reservedAt
actionAt
status
```

`status`가 `"취소됨"`이면 `취소 시간`으로 표시하고, 그 외에는 `완료 시간`으로 표시한다.

```txt
status === "취소됨"
→ 취소 시간

그 외
→ 완료 시간
```

---

## 책임 분리

ReservationsPage 관련 파일의 책임은 다음과 같다.

| 파일                                                          | 역할                                                   |
| ------------------------------------------------------------- | ------------------------------------------------------ |
| `widgets/reservations-page/index.tsx`                         | 예약 목록 조회, WASHER/DRYER 분류, 이력 모달 상태 관리 |
| `widgets/reservations-page/ui/ReservationStatusPanel.tsx`     | 예약 현황 패널 표시                                    |
| `widgets/reservations-page/ui/ReservationRow.tsx`             | 개별 예약 row 표시, 이력 열기, 삭제 요청 연결          |
| `widgets/reservations-page/ui/ReservationHistoryModal.tsx`    | machineName 기준 이력 조회 및 모달 표시                |
| `widgets/reservations-page/ui/ReservationHistoryCard.tsx`     | 예약 이력 item 표시                                    |
| `entities/reservation/api/useGetReservations.ts`              | 예약 목록 서버 상태 관리                               |
| `entities/reservation/api/useGetMachineReservationHistory.ts` | 특정 기기 예약 이력 서버 상태 관리                     |
| `entities/reservation/api/useDeleteReservation.ts`            | 예약 삭제 mutation 및 query invalidate                 |
| `entities/reservation/lib/mapReservation.ts`                  | 예약 DTO를 화면용 item으로 변환                        |
| `entities/reservation/lib/mapReservationHistory.ts`           | 예약 이력 DTO를 화면용 item으로 변환                   |

---

## 데이터 흐름 정리

예약 목록 화면의 데이터 흐름은 다음과 같다.

```txt
getReservations
  ↓
GET /api/v2/admin/reservations
  ↓
ReservationDTO[]
  ↓
mapReservations
  ↓
ReservationItem[]
  ↓
ReservationsPage
  ↓
WASHER / DRYER 분리
  ↓
ReservationStatusPanel
  ↓
ReservationRow
```

예약 이력 모달의 데이터 흐름은 다음과 같다.

```txt
ReservationRow 이력 버튼 클릭
  ↓
onOpenHistory(item.machine)
  ↓
setHistoryOverlay({ machineName, side })
  ↓
ReservationHistoryModal
  ↓
useGetMachineReservationHistory({ machineName })
  ↓
GET /api/v2/admin/reservations/machines/history
  ↓
mapMachineReservationHistory
  ↓
ReservationHistoryCard[]
```

예약 삭제 흐름은 다음과 같다.

```txt
ReservationRow 삭제 버튼 클릭
  ↓
confirm
  ↓
useDeleteReservation
  ↓
DELETE /api/v2/admin/reservations/{id}
  ↓
reservationQueryKeys.all invalidate
  ↓
예약 관련 데이터 갱신
```

---

## 정리

ReservationsPage는 예약 데이터를 조회한 뒤, `WASHER`와 `DRYER` 기준으로 화면을 분리한다.

특정 기기의 이력을 확인할 때는 `machineName`과 `side`를 상태로 저장하고, `ReservationHistoryModal`에서 `machineName` 기준으로 이력 데이터를 조회한다.

핵심은 다음과 같다.

```txt
useGetReservations로 예약 데이터를 가져온다.
getReservations에서 mapReservations를 통해 화면용 데이터로 변환한다.
ReservationsPage에서 type이 DRYER인 데이터와 WASHER인 데이터를 분리한다.
ReservationStatusPanel은 예약 현황 목록을 표시한다.
ReservationRow는 개별 예약 표시, 이력 열기, 삭제 액션을 담당한다.
사용자가 이력을 열면 machineName과 side를 historyOverlay 상태에 저장한다.
ReservationHistoryModal은 machineName을 기준으로 예약 이력을 조회한다.
useGetMachineReservationHistory는 machineName이 있을 때만 query를 실행한다.
예약 삭제 성공 후 reservationQueryKeys.all을 invalidate한다.
```

이 구조는 예약 현황 표시, 기기별 이력 조회, 예약 삭제 후 데이터 갱신 흐름을 분리해서 관리하는 방식이다.
