# Raw Response와 Mapper 적용 기준

## 핵심 흐름

Washer에서는 모든 API 응답에 mapper를 강제하지 않는다.

서버 응답 구조를 화면에서 그대로 사용할 수 있는 도메인은 raw response를 유지하고, 화면 표시를 위해 상태 변환이나 날짜 포맷팅이 필요한 도메인에만 mapper를 적용한다.

즉, 기준은 다음과 같다.

```txt
서버 응답을 그대로 사용해도 되는 경우
→ raw response 유지

UI 표시를 위해 계산/변환이 필요한 경우
→ mapper 또는 util 적용
```

이 기준을 두는 이유는 불필요한 데이터 변환을 줄이고, 필요한 도메인에서만 UI 친화적인 데이터 구조를 만들기 위해서이다.

---

## Report 도메인: raw response 유지

Report 도메인은 서버 응답 구조를 거의 그대로 화면에서 사용할 수 있다.

```ts
export type ReportStatusType = "PENDING" | "IN_PROGRESS" | "RESOLVED";

export interface ReportItemType {
  id: number;
  machineId: number;
  machineName: string;
  reporterId: number;
  reporterName: string;
  description: string;
  status: ReportStatusType;
  reportedAt: string;
  processingStartedAt: string | null;
  resolvedAt: string | null;
  createdAt: string;
  updatedAt: string;
}

export interface ReportResponseType {
  reports: ReportItemType[];
  totalCount: number;
  totalPages: number;
  currentPage: number;
}
```

Report 데이터는 다음 값들을 그대로 화면에서 사용할 수 있다.

```txt
machineName
reporterName
description
status
reportedAt
resolvedAt
```

따라서 Report 도메인에는 별도의 mapper를 만들지 않는다.

API 함수도 서버 응답을 받아서 그대로 반환한다.

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

여기서 API 함수는 데이터를 변환하지 않는다.
서버 요청과 응답 반환만 담당한다.

---

## Report에서 mapper를 적용하지 않는 이유

Report 데이터는 서버 응답 필드와 UI에서 필요한 필드가 크게 다르지 않다.

예를 들어 고장 신고 목록에서는 신고자 이름, 기기명, 설명, 신고 상태, 신고 시각을 그대로 사용하면 된다.

따라서 아래와 같은 변환이 필요하지 않다.

```txt
machineName → machine
status → badgeStatus
reservedAt → reserveAt
machineAvailability → deviceStatus
```

이런 변환이 필요하지 않기 때문에 Report에 mapper를 적용하면 오히려 불필요한 계층이 생긴다.

Report 도메인은 raw response를 유지하는 것이 더 단순하다.

---

## status label/style은 mapper가 아니라 map으로 처리한다

Report나 Reservation에서 상태에 따라 색상이나 라벨을 보여줘야 할 때가 있다.

이때 서버 응답 데이터 자체를 바꾸는 것이 아니라, UI 표시용 map을 따로 둔다.

```ts
export const statusColorMap = {
  사용중: "bg-[#4D83F6]",
  예약중: "bg-[#4D83F6]",
  확인필요: "bg-[#EF4B4F]",

  "사용 완료": "bg-[#4D83F6]",
  취소됨: "bg-[#EF4B4F]",
} as const;
```

`statusColorMap`은 API 응답을 변환하는 mapper가 아니다.
상태값에 맞는 UI 스타일을 찾기 위한 표시용 map이다.

즉, 데이터 구조 변환과 UI 스타일 매핑은 구분한다.

```txt
mapper
- 서버 응답 구조를 UI에서 쓰기 좋은 데이터 구조로 바꾼다.

statusColorMap
- 이미 정리된 상태값에 맞는 스타일 클래스를 반환한다.
```

---

## Reservation 도메인: UI 계산이 필요하므로 mapper 적용

Reservation 도메인은 서버 응답 구조와 화면에서 필요한 구조가 다르다.

서버에서 내려오는 예약 데이터는 `ReservationDTO` 형태이다.

```ts
export type ReservationDTO = {
  id: number;
  userId: number;
  userName: string;
  userRoomNumber: string;
  userStudentId: string;
  machineId: number;
  machineName: string;
  reservedAt: string;
  startTime: string;
  expectedCompletionTime: string;
  actualCompletionTime: string | null;
  cancelledAt: string | null;
  status: ReservationDTOStatus;
  machineAvailability: MachineAvailabilityStatus;
};
```

하지만 화면에서 예약 현황 카드에 필요한 데이터는 `ReservationItem` 형태이다.

```ts
export interface ReservationItem {
  id: number;
  machineId: number;
  machine: string;
  type: ReservationMachineType;
  badgeStatus: ReservationStatusLabel;
  reserveAt?: string;
  deviceStatus?: string;
  expectedCompletionTime?: string;
  startTime?: string;
}
```

즉, 서버 응답에는 있지만 화면에 필요 없는 값도 있고, 반대로 화면 표시를 위해 새로 계산해야 하는 값도 있다.

예를 들어 다음 값들은 서버 응답을 그대로 쓰는 것이 아니라 가공해서 만든다.

```txt
machineName → machine
machineName → type
status + machineAvailability → badgeStatus
reservedAt → reserveAt
machineAvailability → deviceStatus
```

그래서 Reservation에는 mapper를 적용한다.

---

## Reservation mapper 흐름

예약 데이터 변환은 `mapReservation`에서 처리한다.

```ts
export function mapReservation(dto: ReservationDTO): ReservationItem {
  const badgeStatus = mapBadgeStatus(dto);

  return {
    id: dto.id,
    machineId: dto.machineId,
    machine: dto.machineName,
    type: getMachineType(dto.machineName),
    badgeStatus,
    reserveAt:
      badgeStatus === "예약중" ? formatDateTime(dto.reservedAt) : undefined,
    deviceStatus:
      badgeStatus === "사용중"
        ? mapAvailabilityDeviceStatus(dto.machineAvailability)
        : undefined,
    expectedCompletionTime:
      badgeStatus === "사용중" ? dto.expectedCompletionTime : undefined,
    startTime: badgeStatus === "예약중" ? dto.startTime : undefined,
  };
}
```

이 mapper는 서버 응답인 `ReservationDTO`를 화면에서 사용할 `ReservationItem`으로 변환한다.

---

## machineName으로 기기 타입을 판단한다

Reservation 데이터에서는 `machineName`을 기준으로 기기 타입을 판단한다.

```ts
function getMachineType(machineName: string): ReservationMachineType {
  const upper = machineName.toUpperCase();

  if (upper.startsWith("D") || upper.includes("DRYER")) {
    return "DRYER";
  }

  return "WASHER";
}
```

화면에서는 세탁기와 건조기를 나누어 보여줘야 한다.

```txt
WASHER
DRYER
```

그래서 mapper 단계에서 `machineName`을 기준으로 `type` 값을 만든다.

이렇게 하면 페이지 컴포넌트에서는 이미 변환된 `type` 값을 기준으로 데이터를 분류할 수 있다.

---

## badgeStatus를 UI 기준으로 변환한다

서버 상태값은 다음과 같다.

```ts
export type ReservationDTOStatus =
  | "RESERVED"
  | "RUNNING"
  | "COMPLETED"
  | "CANCELLED";
```

기기 사용 가능 상태는 다음과 같다.

```ts
export type MachineAvailabilityStatus =
  | "IN_USE"
  | "RESERVED"
  | "AVAILABLE"
  | "UNAVAILABLE";
```

하지만 화면에서 표시하는 예약 상태 라벨은 다음과 같다.

```ts
export type ReservationStatusLabel = "예약중" | "사용중" | "확인필요";
```

즉, 서버 상태값과 UI 상태 라벨이 다르다.

그래서 `mapBadgeStatus`에서 UI 표시용 상태로 변환한다.

```ts
function mapBadgeStatus(dto: ReservationDTO): ReservationStatusLabel {
  if (dto.machineAvailability === "UNAVAILABLE") {
    return "확인필요";
  }

  if (dto.status === "RESERVED") {
    return "예약중";
  }

  if (dto.status === "RUNNING") {
    return "사용중";
  }

  return "확인필요";
}
```

이 함수는 서버의 `status`와 `machineAvailability`를 조합해서 화면에서 사용할 `badgeStatus`를 만든다.

---

## 조건부 필드 처리

Reservation mapper에서는 상태에 따라 필요한 필드만 내려준다.

```ts
reserveAt:
  badgeStatus === "예약중" ? formatDateTime(dto.reservedAt) : undefined,
```

예약중일 때만 예약 시간이 필요하다.

```ts
deviceStatus:
  badgeStatus === "사용중"
    ? mapAvailabilityDeviceStatus(dto.machineAvailability)
    : undefined,
```

사용중일 때만 기기 상태 표시가 필요하다.

```ts
expectedCompletionTime:
  badgeStatus === "사용중" ? dto.expectedCompletionTime : undefined,
```

사용중일 때만 예상 완료 시간이 필요하다.

즉, Reservation mapper는 단순히 이름만 바꾸는 것이 아니라, 화면 상태에 따라 필요한 값만 정리하는 역할을 한다.

---

## 날짜 포맷팅도 mapper에서 처리한다

예약 시간은 서버에서 문자열로 내려온다.

```ts
reservedAt: string;
```

화면에서는 그대로 보여주기보다 포맷팅된 날짜가 필요하다.

```ts
reserveAt:
  badgeStatus === "예약중" ? formatDateTime(dto.reservedAt) : undefined,
```

그래서 `formatDateTime`을 사용해 화면 표시용 시간 문자열로 바꾼다.

이런 처리는 UI 컴포넌트마다 반복해서 작성하지 않고, mapper에서 한 번 처리하는 것이 더 적절하다.

---

## mapReservations

목록 데이터는 배열 형태이기 때문에 `mapReservations`에서 개별 mapper를 반복 적용한다.

```ts
export function mapReservations(dtos: ReservationDTO[]): ReservationItem[] {
  return dtos.map(mapReservation);
}
```

즉, 흐름은 다음과 같다.

```txt
ReservationDTO[]
  ↓
mapReservations
  ↓
ReservationItem[]
```

이렇게 변환된 데이터는 화면에서 바로 사용하기 좋은 구조가 된다.

---

## Reservation History mapper

예약 이력 데이터도 서버 응답과 UI 표시 형태가 다르다.

서버 응답 item은 다음 구조이다.

```ts
export type MachineReservationHistoryItemDTO = {
  roomNumber: string;
  reservedAt: string;
  actualCompletionTime: string | null;
  cancelledAt: string | null;
  status: "COMPLETED" | "CANCELLED" | "RESERVED" | "IN_USE";
};
```

UI에서 사용하는 이력 item은 다음 구조이다.

```ts
export interface ReservationHistoryItem {
  roomNumber: string;
  reservedAt: string;
  actionAt?: string;
  status: ReservationHistoryStatus;
}
```

여기서 `actionAt`은 상태에 따라 다른 값을 사용한다.

```txt
취소됨
→ cancelledAt

사용 완료
→ actualCompletionTime
```

그래서 이력 데이터에도 mapper를 적용한다.

---

## 이력 상태 변환

서버 상태값은 영어 enum이다.

```ts
"COMPLETED" | "CANCELLED" | "RESERVED" | "IN_USE";
```

하지만 화면에서는 한글 상태로 보여준다.

```ts
export type ReservationHistoryStatus = "사용 완료" | "취소됨";
```

이를 위해 `mapHistoryStatus`를 사용한다.

```ts
function mapHistoryStatus(
  status: MachineReservationHistoryItemDTO["status"]
): ReservationHistoryStatus {
  switch (status) {
    case "CANCELLED":
      return "취소됨";
    case "COMPLETED":
      return "사용 완료";
    default:
      return "사용 완료";
  }
}
```

이 함수는 서버 상태값을 UI 라벨로 변환한다.

---

## 이력 item 변환

개별 이력 item은 `mapHistoryItem`에서 변환한다.

```ts
function mapHistoryItem(
  dto: MachineReservationHistoryItemDTO
): ReservationHistoryItem {
  const status = mapHistoryStatus(dto.status);

  return {
    roomNumber: dto.roomNumber,
    reservedAt: formatDateTime(dto.reservedAt) ?? "-",
    actionAt:
      status === "취소됨"
        ? formatDateTime(dto.cancelledAt)
        : formatDateTime(dto.actualCompletionTime),
    status,
  };
}
```

이 mapper는 다음 처리를 담당한다.

```txt
reservedAt 날짜 포맷팅
status 한글 라벨 변환
상태에 따른 actionAt 선택
```

이런 가공을 모달 컴포넌트에서 직접 처리하지 않고 mapper에서 처리한다.

---

## mapMachineReservationHistory

기기별 예약 이력 전체 구조는 `mapMachineReservationHistory`에서 변환한다.

```ts
export function mapMachineReservationHistory(
  dto: MachineReservationHistoryDTO
): MachineReservationHistory {
  return {
    machineName: dto.machineName,
    reservations: dto.reservations.map(mapHistoryItem),
  };
}
```

흐름은 다음과 같다.

```txt
MachineReservationHistoryDTO
  ↓
mapMachineReservationHistory
  ↓
MachineReservationHistory
```

이렇게 하면 `ReservationHistoryModal`은 이미 가공된 `reservations` 배열을 받아 화면에 표시할 수 있다.

---

## Machine mapper

Machine 도메인도 서버 응답 상태와 화면 표시 상태가 다르기 때문에 mapper를 적용한다.

```ts
export function mapMachine(dto: AdminMachineDTO): MachineItem {
  return {
    id: dto.id,
    name: dto.name,
    type: dto.type,
    status: mapMachineStatusLabel(dto.status, dto.availability),
    condition: dto.status,
    availability: dto.availability,
    deviceStatus: mapAvailabilityDeviceStatus(dto.availability),
  };
}
```

Machine mapper는 다음 변환을 담당한다.

```txt
서버 condition + availability
→ 화면용 status label

availability
→ deviceStatus
```

기기 상태는 단순히 하나의 값만 보고 판단하기 어렵다.
예를 들어 기기 자체가 고장 상태라면 availability보다 condition을 우선해서 `고장`으로 보여줘야 한다.

```ts
function mapMachineStatusLabel(
  condition: MachineConditionStatusDTO,
  availability: MachineAvailabilityStatusDTO
): MachineStatusLabel {
  if (condition === "MALFUNCTION") {
    return "고장";
  }

  switch (availability) {
    case "IN_USE":
      return "사용중";
    case "RESERVED":
      return "예약";
    case "AVAILABLE":
      return "미사용";
    case "UNAVAILABLE":
      return "사용 정지";
    default:
      return "확인필요";
  }
}
```

이런 상태 판단은 UI 컴포넌트보다 mapper에 두는 것이 적절하다.

---

## Dashboard mapper

Dashboard는 서버 요약 데이터를 화면 카드 배열로 바꾸기 위해 mapper를 사용한다.

```ts
export function mapDashboard(summary: DashboardSummaryDTO): DashboardItem[] {
  return [
    { label: "총 기기수", value: `${summary.totalMachines}대` },
    { label: "예약 활성화", value: `${summary.activeReservations}대` },
    {
      label: "고장 신고",
      value: `${
        summary.pendingMalfunctionReports + summary.processingMalfunctionReports
      }건`,
    },
    { label: "세탁정지", value: `${summary.suspendedStudents}명` },
  ];
}
```

서버 응답은 통계값 중심의 객체이지만, 화면에서는 카드 리스트 형태가 필요하다.

그래서 Dashboard mapper는 객체를 배열로 변환한다.

```txt
DashboardSummaryDTO
  ↓
mapDashboard
  ↓
DashboardItem[]
```

또한 값에 단위를 붙이는 처리도 mapper에서 담당한다.

```txt
totalMachines → 10대
pendingMalfunctionReports + processingMalfunctionReports → 3건
suspendedStudents → 2명
```

---

## 공통 util: mapAvailabilityDeviceStatus

기기 사용 가능 상태를 화면 문구로 바꾸는 로직은 여러 도메인에서 사용할 수 있다.

```ts
export function mapAvailabilityDeviceStatus(
  availability: AvailabilityStatus
): string | undefined {
  switch (availability) {
    case "IN_USE":
      return "사용 중";
    case "RESERVED":
      return "예약됨";
    case "AVAILABLE":
      return "사용 가능";
    case "UNAVAILABLE":
      return "사용 불가";
    default:
      return undefined;
  }
}
```

이 함수는 Reservation과 Machine에서 모두 사용할 수 있는 공통 변환 로직이다.

따라서 특정 도메인 mapper 내부에 두지 않고 `shared/lib`에 둔다.

기준은 다음과 같다.

```txt
특정 도메인에만 필요한 변환
→ entities/{domain}/lib 또는 mapper

여러 도메인에서 재사용 가능한 변환
→ shared/lib
```

---

## Raw Response와 Mapper 판단 기준

이 프로젝트에서 mapper 적용 여부를 판단하는 기준은 다음과 같다.

```txt
1. 서버 응답 필드와 UI 사용 필드가 거의 같으면 raw response를 유지한다.
2. 서버 상태값을 UI 라벨로 바꿔야 하면 mapper를 적용한다.
3. 날짜 포맷팅이 반복되면 mapper에서 처리한다.
4. 여러 서버 필드를 조합해 하나의 UI 상태를 만들어야 하면 mapper를 적용한다.
5. 화면에서 필요한 배열/카드 구조가 서버 응답과 다르면 mapper를 적용한다.
6. 단순한 색상/스타일 매핑은 mapper가 아니라 별도 map으로 처리한다.
```

---

## 도메인별 적용 기준 정리

| 도메인              | 처리 방식         | 이유                                                      |
| ------------------- | ----------------- | --------------------------------------------------------- |
| Report              | raw response 유지 | 서버 응답 구조를 화면에서 그대로 사용할 수 있음           |
| Reservation         | mapper 적용       | 상태 라벨, 기기 타입, 날짜, 조건부 필드 변환이 필요함     |
| Reservation History | mapper 적용       | 상태 라벨, actionAt, 날짜 포맷팅이 필요함                 |
| Machine             | mapper 적용       | condition과 availability를 조합해 화면 상태를 만들어야 함 |
| Dashboard           | mapper 적용       | 요약 객체를 카드 배열로 바꿔야 함                         |

---

## 정리

Washer에서는 API 응답을 무조건 mapper로 변환하지 않는다.

Report처럼 서버 응답 구조를 그대로 사용할 수 있는 도메인은 raw response를 유지한다.
Reservation, Machine, Dashboard처럼 화면 표시를 위해 상태 변환, 날짜 포맷팅, 데이터 구조 변경이 필요한 도메인에는 mapper를 적용한다.

최종 기준은 다음과 같다.

```txt
API function
- 서버 요청과 raw response 반환을 담당한다.

mapper
- 서버 응답을 UI에서 사용하기 좋은 구조로 변환한다.

status map
- 상태값에 맞는 UI 스타일을 매핑한다.

shared util
- 여러 도메인에서 재사용되는 변환 로직을 담당한다.
```

이 구조의 핵심은 데이터 가공을 무조건 적용하는 것이 아니라, 도메인별 UI 요구사항에 따라 필요한 곳에만 적용하는 것이다.
