# Washer v2 컴포넌트 설계 기준

## 1. 설계 배경

Washer v1의 관리자 화면은 하나의 `AdminPage` 안에 다음 책임이 함께 들어가 있었습니다.

- 기기·예약·신고·사용자 데이터 관리
- API 호출과 상태 변경
- 필터 및 모달 상태
- 목록과 상태 배지 렌더링
- 사용자 액션 처리

관리자 페이지가 1,600줄 이상으로 커지면서 기능별 수정 위치를 파악하기 어렵고, 유사한 UI가 화면마다 직접 작성돼 공통 스타일이나 동작을 변경할 때 여러 영역을 함께 수정해야 했습니다.

v2에서는 단순히 파일을 작게 나누는 것이 아니라, **컴포넌트의 변경 이유와 사용자 동작을 기준으로 공통화 범위를 결정**했습니다.

---

# 핵심 분리 원칙

> 외형과 동작 구조가 같으면 공통화하고, 데이터 처리와 업무 흐름이 다르면 도메인 컴포넌트로 유지한다.

컴포넌트를 다음 세 가지 경우로 구분했습니다.

1. 별도 공통 컴포넌트로 추출
2. 하나의 컴포넌트 안에서 조건부 렌더링
3. 도메인별 별도 컴포넌트로 유지

---

## 2. 외부 레이아웃이 같으면 공통 컴포넌트로 추출

예약·기기·신고·사용자 관리 패널은 모두 다음 구조를 공유합니다.

- 패널 외부 영역
- 제목
- 아이콘
- 본문 영역

데이터와 기능은 다르지만 외부 레이아웃은 동일했기 때문에 `StatusPanelShell`로 분리했습니다.

```tsx
<StatusPanelShell title={title} icon={icon}>
  {도메인별 내용}
</StatusPanelShell>
```

`StatusPanelShell`은 패널의 외형만 담당하고, 실제 목록과 액션은 `children`으로 전달받습니다.

### 판단 기준

> 데이터와 사용자 동작은 달라도 외부 레이아웃과 헤더 구조가 동일하면 공통화한다.

### 적용 대상

- `MachineStatusPanel`
- `ReservationStatusPanel`
- `ReportsPanel`
- `UserStatusPanel`

---

## 3. 구조가 같고 값만 다르면 props로 주입

세탁기와 건조기 화면은 다음 요소만 다릅니다.

- 기기 데이터
- 제목
- 아이콘
- 모달이 열리는 방향

반면 다음 흐름은 동일합니다.

- 기기 목록 표시
- 상태 배지 표시
- 예약 이력 조회
- 기기 관리 기능

따라서 세탁기와 건조기용 컴포넌트를 각각 만들지 않고 동일한 `MachineStatusPanel`을 재사용했습니다.

```tsx
const dryerMachines = machines.filter((item) => item.type === "DRYER");

const washerMachines = machines.filter((item) => item.type === "WASHER");
```

```tsx
<MachineStatusPanel
  title="건조기 상태"
  icon={<Waves />}
  machines={dryerMachines}
/>

<MachineStatusPanel
  title="세탁기 상태"
  icon={<Droplet />}
  machines={washerMachines}
  side="left"
/>
```

### 판단 기준

> 레이아웃과 기능 흐름은 같고 데이터·제목·아이콘처럼 값만 다르면 같은 컴포넌트를 사용하고 차이를 props로 전달한다.

예약 화면도 같은 방식으로 전체 예약 데이터를 `WASHER`, `DRYER` 타입으로 분류한 뒤 동일한 `ReservationStatusPanel`에 전달합니다.

---

## 4. 화면 배치에 따른 차이는 명시적인 union type으로 표현

화면이 두 개의 패널로 나뉘어 있어 모달은 패널의 위치에 따라 반대 방향으로 열려야 했습니다.

이를 다음처럼 `side` prop으로 표현했습니다.

```tsx
side?: "left" | "right";
```

```tsx
const overlayPositionClass =
  side === "right"
    ? "absolute left-[calc(100%+16px)]"
    : "absolute right-[calc(100%+16px)]";
```

`isLeft`와 같은 boolean 대신 `"left" | "right"` union type을 사용해 prop의 목적을 코드에서 바로 확인할 수 있도록 했습니다.

### 판단 기준

> 두 가지 이상의 명확한 UI 상태가 존재하면 boolean보다 의미를 직접 드러내는 union type으로 표현한다.

---

## 5. 선택 항목과 UI 위치가 함께 변하면 하나의 상태로 관리

예약 이력 모달은 다음 정보가 함께 필요합니다.

- 어떤 기기의 이력인지
- 어느 방향에 모달을 표시할지

이를 각각의 state로 분리하지 않고 하나의 상태 객체로 관리했습니다.

```tsx
type HistoryOverlayState = {
  machineName: string;
  side: "left" | "right";
} | null;
```

```tsx
setHistoryOverlay({
  machineName,
  side: "right",
});
```

### 판단 기준

> 동시에 생성되고 동시에 초기화되는 상태는 하나의 상태 객체로 묶어 잘못된 상태 조합을 방지한다.

예를 들어 `isOpen`, `machineName`, `side`를 따로 관리하면 모달은 열렸지만 기기명이 없거나, 이전 방향값이 남아 있는 상태가 만들어질 수 있습니다.

---

## 6. UI 배치가 같고 실행 로직만 다르면 콜백으로 공통화

예약 행과 기기 행의 오른쪽 영역은 다음 구조가 동일합니다.

- 상태 배지
- 예약 이력 버튼
- 취소 또는 관리 버튼

이 구조를 `StatusRowActions`로 분리했습니다.

```tsx
interface StatusRowActionsProps {
  badge: ReactNode;
  onHistory?: () => void;
  onDelete?: () => void;
  disabled?: boolean;
}
```

예약 화면에서는 예약 상태 배지와 예약 취소 로직을 전달합니다.

```tsx
<StatusRowActions
  badge={<ReservationStatusBadge label={item.badgeStatus} />}
  onHistory={handleHistory}
  onDelete={handleDelete}
  disabled={isPending}
/>
```

기기 화면에서는 기기 상태 배지와 기기 관리 모달을 여는 로직을 전달합니다.

```tsx
<StatusRowActions
  badge={<MachineStatusBadge status={machine.status} />}
  onHistory={onHistory}
  onDelete={onManage}
/>
```

### 책임 분리

- `StatusRowActions`: 버튼의 배치와 외형
- `ReservationRow`: 예약 이력 조회와 예약 취소
- `MachineRow`: 예약 이력 조회와 기기 관리
- Badge 컴포넌트: 도메인별 상태 표현

### 판단 기준

> 표시되는 UI 구조는 같고 클릭 후 실행되는 로직만 다르면 ReactNode와 callback을 주입해 공통화한다.

---

## 7. 전체 구조가 유지되고 일부 내용만 바뀌면 내부에서 조건 분기

예약 행은 예약 상태에 따라 부가 정보가 달라집니다.

### 사용 중

```tsx
{
  item.badgeStatus === "사용중" && (
    <>
      <p>남은 시간: {remainText}</p>
      <p>기기 상태: {item.deviceStatus}</p>
    </>
  );
}
```

### 예약 중

```tsx
{
  item.badgeStatus === "예약중" && (
    <>
      <p>예약 시간: {item.reserveAt}</p>
      <p>예약 만료까지: {expiredText}</p>
    </>
  );
}
```

이외에도 `확인필요`, `사용 완료`, `취소됨` 상태에 맞는 안내 문장을 표시합니다.

상태마다 별도 행 컴포넌트를 만들지 않은 이유는 다음 구조가 모두 같기 때문입니다.

- 기기 아이콘
- 기기명
- 사용자 방 번호
- 상태 배지
- 이력 및 취소 버튼

달라지는 것은 중간의 설명 영역뿐입니다.

### 판단 기준

> 전체 레이아웃과 액션은 유지되고 일부 텍스트나 부가 정보만 달라지면 하나의 컴포넌트 안에서 조건부 렌더링한다.

---

## 8. 버튼 구성과 업무 흐름이 다르면 별도 컴포넌트로 유지

사용자 관리 화면은 `StatusRowActions`를 재사용하지 않고 `UserRowActions`를 별도로 구성했습니다.

사용자 상태에 따라 버튼의 구성 자체가 달라지기 때문입니다.

```tsx
if (isRestrictedCase) {
  return (
    <>
      <button>연장</button>
      <button>해제</button>
    </>
  );
}

return <button>세탁 정지</button>;
```

기기·예약 화면은 동일한 아이콘 버튼 구조를 사용하지만, 사용자 화면은 다음 요소가 모두 다릅니다.

- 버튼 개수
- 버튼 형태
- 버튼 의미
- 실행되는 업무 흐름

### 판단 기준

> 단순히 클릭 이벤트만 다른 것이 아니라 버튼 구성과 사용자 업무 흐름 자체가 달라지면 별도 컴포넌트로 유지한다.

이를 공통 컴포넌트로 합치면 다음과 같은 조건 props가 계속 늘어날 수 있습니다.

```tsx
<RowActions
  showHistory={false}
  showDelete={false}
  showStopLaundry
  showExtend={isRestricted}
  showRelease={isRestricted}
/>
```

이런 구조는 공통 컴포넌트가 여러 도메인의 조건을 모두 알아야 하므로 오히려 변경 비용이 증가합니다.

---

## 9. 외형이 비슷해도 API와 업무 책임이 다르면 합치지 않음

`ReservationHistoryModal`과 `MachineStatusModal`은 외형상 다음 요소가 비슷합니다.

- 좌우 위치
- 닫기 버튼
- 외부 클릭 닫기
- 패널 스타일
- 제목 영역

하지만 내부 책임은 다릅니다.

### `ReservationHistoryModal`

- 기기명을 기준으로 예약 이력 조회
- 로딩·에러·빈 상태 처리
- 예약 이력 목록 렌더링

### `MachineStatusModal`

- 기기 상태 변경
- 기기 삭제
- mutation pending 처리
- 삭제 확인 화면
- 성공·실패 알림

두 모달을 하나로 합치면 `mode`, `showDelete`, `onStatusUpdate`, `reservations` 등의 조건이 하나의 범용 컴포넌트에 집중됩니다.

### 판단 기준

> 외형이 비슷하더라도 사용하는 API, mutation, 사용자 업무 흐름이 다르면 별도 도메인 컴포넌트로 유지한다.

다만 모달의 위치 계산과 외부 패널 프레임은 중복되므로 이후 다음 정도의 공통화는 가능합니다.

```tsx
<SidePanel side={side}>{children}</SidePanel>
```

전체 기능을 합치는 것이 아니라, 정말 동일한 외부 레이아웃만 추출하는 방식입니다.

---

# 최종 컴포넌트 분기 기준

## 공통 컴포넌트로 추출하는 경우

- 외형과 레이아웃이 동일함
- 사용자 동작의 형태가 동일함
- 차이를 props 또는 callback으로 표현할 수 있음
- 같은 이유로 함께 변경될 가능성이 높음

### 예시

- `StatusPanelShell`
- `StatusRowActions`
- 세탁기·건조기의 `MachineStatusPanel`
- 세탁기·건조기의 `ReservationStatusPanel`

---

## 하나의 컴포넌트 안에서 조건 분기하는 경우

- 전체 구조는 동일함
- 일부 텍스트나 부가 정보만 달라짐
- 상태를 명확한 union 값으로 표현할 수 있음

### 예시

```tsx
badgeStatus === "사용중";
badgeStatus === "예약중";
badgeStatus === "확인필요";
badgeStatus === "사용 완료";
badgeStatus === "취소됨";
```

---

## 별도 컴포넌트로 유지하는 경우

- 사용하는 API 또는 mutation이 다름
- 사용자 업무 흐름이 다름
- 버튼의 개수나 의미가 다름
- 공통화할 경우 boolean prop이나 조건문이 증가함
- 도메인마다 변경되는 이유가 다름

### 예시

- `UserRowActions`
- `MachineStatusModal`
- `ReservationHistoryModal`
- 각 도메인의 StatusPanel

---

# 기준별 처리 방식

| 차이가 발생하는 기준          | 처리 방식                                 |
| ----------------------------- | ----------------------------------------- |
| 외부 레이아웃과 헤더만 동일   | 공통 레이아웃 컴포넌트로 추출             |
| 제목과 아이콘만 다름          | `title`, `icon` props 전달                |
| 세탁기·건조기 데이터가 다름   | `type`으로 필터링 후 동일 컴포넌트에 전달 |
| 패널 위치가 다름              | `side: "left" \| "right"` prop 사용       |
| 선택된 기기가 다름            | 기기 객체 또는 기기명을 상태로 관리       |
| 일부 표시 내용만 다름         | 같은 컴포넌트 내부에서 조건부 렌더링      |
| 상태 배지의 종류만 다름       | `ReactNode`로 주입                        |
| 클릭 후 실행 로직만 다름      | callback으로 주입                         |
| 버튼 구성과 업무 흐름이 다름  | 별도 액션 컴포넌트 유지                   |
| API와 mutation이 다름         | 별도 도메인 컴포넌트 유지                 |
| 로딩·에러·빈 목록 상태가 다름 | 도메인 컴포넌트 내부에서 상태별 렌더링    |

---

# 설계의 핵심

Washer v2에서는 반복되는 UI를 화면 모양만 보고 하나로 합치지 않았습니다.

- 외형이 같은 부분은 공통화
- 값만 다른 부분은 props로 주입
- 일부 내용만 다른 경우에는 조건부 렌더링
- API와 업무 흐름이 다른 경우에는 도메인 컴포넌트로 유지

즉, **변경 이유가 같은 코드는 함께 관리하고 변경 이유가 다른 코드는 분리하는 기준**으로 컴포넌트를 설계했습니다.

이 구조를 통해 v1의 단일 관리자 페이지에서 여러 위치에 흩어져 있던 반복 UI의 수정 지점을 공통 컴포넌트로 모으고, 도메인별 기능 변경이 다른 화면으로 전파되는 범위를 줄였습니다.
