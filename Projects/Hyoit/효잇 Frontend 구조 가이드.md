# 효잇 Frontend 구조 가이드

## 1. 기본 방향

효잇은 현재 `apps/mobile` 중심으로 개발되는 모바일 앱이다.

모노레포 구조를 사용하고 있지만, 모든 공통 코드를 무조건 `packages`로 분리하지 않는다.
코드의 성격과 변경 이유에 따라 다음처럼 구분한다.

```txt
packages
→ 앱이 돌아가기 위한 기반 코드

apps/mobile/src/shared
→ 모바일 앱 내부에서 부모/자녀가 함께 쓰는 화면/기능 규칙

apps/mobile/src/parent
→ 부모 플로우 전용 코드

apps/mobile/src/child
→ 자녀 플로우 전용 코드
```

핵심 기준은 다음이다.

```txt
앱 없어도 설명 가능한 코드인가?
→ packages

모바일 화면 맥락이 있어야 설명 가능한 코드인가?
→ apps/mobile/src/shared 또는 parent/child
```

여기서 “앱 없어도 설명 가능한 코드”란, 효잇 홈 화면이나 부모/자녀 플로우를 몰라도 역할이 설명되는 코드를 말한다.

예를 들어 `Button`, `Card`, `UserRole`, `ApiResponse`, `useAuthStore`, `storage`는 특정 화면을 몰라도 설명할 수 있다.
반대로 `HomeStatus`, `HOME_STATUS_LABEL`, `ConnectedChildCard`, `CheckinReplyButton`은 효잇 모바일 화면 맥락이 있어야 설명할 수 있다.

---

## 2. 전체 구조

```txt
hyoit-FE/
  apps/
    mobile/
      src/
        parent/
        child/
        shared/

  packages/
    auth/
    storage/
    types/
    ui/
```

---

## 3. `packages`의 역할

`packages`는 특정 화면이나 플로우에 묶이지 않는 프로젝트 기반 코드를 관리한다.

즉, `Home`, `Profile`, `Log`, `Parent`, `Child` 같은 화면 맥락을 몰라도 설명 가능한 코드만 둔다.

```txt
packages
= 앱 기반 코드
= 화면과 무관하게 재사용 가능한 코드
= 앱 없어도 역할이 설명되는 코드
```

---

## 4. “앱 없어도 설명 가능한 코드” 예시

### 4-1. `Button`

```tsx
<Button variant="primary">확인</Button>
```

설명:

```txt
사용자가 누를 수 있는 기본 버튼 컴포넌트
```

이건 효잇 홈, 부모, 자녀 플로우를 몰라도 설명 가능하다.

위치:

```txt
packages/ui/Button
```

---

### 4-2. `Card`

```tsx
<Card>
  <Text>내용</Text>
</Card>
```

설명:

```txt
내용을 감싸는 공통 카드 UI
```

특정 화면과 무관한 순수 UI 컴포넌트다.

위치:

```txt
packages/ui/Card
```

---

### 4-3. `UserRole`

```ts
export type UserRole = "parent" | "child";
```

설명:

```txt
사용자의 역할을 나타내는 타입
```

홈 화면 전용 타입이 아니라, 인증, 온보딩, 라우팅, API에서도 쓰일 수 있다.

위치:

```txt
packages/types
```

---

### 4-4. `ApiResponse`

```ts
export type ApiResponse<T> = {
  data: T;
  message: string;
};
```

설명:

```txt
서버 응답의 공통 형식
```

특정 화면이 아니라 API 전체 규칙이다.

위치:

```txt
packages/types
```

---

### 4-5. `useAuthStore`

```ts
const role = useAuthStore((state) => state.role);
```

설명:

```txt
로그인 여부, 사용자 역할, 온보딩 상태를 관리하는 인증 store
```

홈 화면이 없어도 필요한 인증 기반 코드다.

위치:

```txt
packages/auth
```

---

### 4-6. `storage`

```ts
storage.set("accessToken", token);
storage.get("role");
```

설명:

```txt
앱 저장소에 값을 저장하고 읽는 공통 래퍼
```

특정 화면과 무관한 저장소 기반 코드다.

위치:

```txt
packages/storage
```

---

## 5. 모바일 화면 맥락이 있어야 설명되는 코드

이런 코드는 `packages`보다 `apps/mobile/src` 안에 둔다.

### 5-1. `HomeStatus`

```ts
export type HomeStatus = "received" | "empty" | "multiple" | "sent" | "checked";
```

설명:

```txt
효잇 모바일 홈에서 안부 상태를 어떻게 보여줄지 나타내는 타입
```

이 타입은 효잇 모바일 홈 화면 맥락이 있어야 설명된다.

위치:

```txt
apps/mobile/src/shared/model/homeStatus.ts
```

---

### 5-2. `HOME_STATUS_LABEL`

```ts
export const HOME_STATUS_LABEL = {
  parent: {
    received: "새 안부",
    empty: "확인함",
    multiple: "자녀 확인 전",
    sent: "자녀 확인함",
    checked: "확인함",
  },
  child: {
    received: "새 안부",
    empty: "확인함",
    multiple: "부모님 확인 전",
    sent: "부모님 확인함",
    checked: "확인함",
  },
} as const;
```

설명:

```txt
부모/자녀 홈에서 상태별로 보여줄 문구
```

이건 서버 계약도 아니고, 순수 UI 컴포넌트도 아니다.
모바일 홈 화면의 역할별 표시 규칙이다.

위치:

```txt
apps/mobile/src/shared/model/homeStatus.ts
```

---

### 5-3. `ConnectedChildCard`

설명:

```txt
부모 홈에서 연결된 자녀 정보를 보여주는 카드
```

부모 홈 화면이 있어야 설명되는 컴포넌트다.

위치:

```txt
apps/mobile/src/parent/widgets/connected-child-card
```

---

### 5-4. `CheckinReplyButton`

설명:

```txt
자녀가 부모님의 안부에 답장할 때 누르는 버튼
```

자녀 플로우가 있어야 설명되는 기능이다.

위치:

```txt
apps/mobile/src/child/features/reply-checkin
```

---

## 6. `packages/auth`

인증과 사용자 역할 상태를 관리한다.

### 포함 대상

```txt
로그인 상태
사용자 role
온보딩 여부
토큰 상태
auth store
logout
```

### 예시

```ts
import { useAuthStore } from "@hyoit/auth";
```

### 넣지 않는 것

```txt
부모 홈 상태
자녀 홈 상태
안부 문구
홈 카드 UI 상태
```

`auth`는 인증만 알아야 한다.
홈 화면이나 안부 화면의 규칙을 알면 안 된다.

---

## 7. `packages/storage`

앱 저장소 접근 로직을 관리한다.

### 포함 대상

```txt
AsyncStorage wrapper
토큰 저장/삭제
온보딩 여부 저장
role 저장
```

특정 화면의 상태가 아니라, 앱 전체에서 사용하는 저장소 기반 로직만 둔다.

---

## 8. `packages/types`

프로젝트 전체에서 공유하는 타입을 관리한다.

### 포함 대상

```txt
User
UserRole
AuthUser
ApiResponse
서버 API 응답 타입
push payload 타입
서버와 맞춰야 하는 계약 타입
```

### 예시

```ts
export type UserRole = "parent" | "child";

export type ApiResponse<T> = {
  data: T;
  message: string;
};
```

### 판단 기준

```txt
서버/API/push와 직접 맞춰야 하는 타입인가?
→ packages/types
```

예를 들어 push payload가 다음처럼 온다면:

```ts
export type PushEventType =
  | "CHECKIN_RECEIVED"
  | "CHECKIN_CHECKED"
  | "CHECKIN_REPLIED";

export type PushPayload = {
  type: PushEventType;
  checkinId: string;
  senderRole: "parent" | "child";
};
```

이런 타입은 화면 표현용 타입이 아니라 서버/push와 맞춰야 하는 통신 계약 타입이므로 `packages/types`에 둔다.

### 넣지 않는 것

```txt
HomeStatus
HomeBannerVariant
LogFilterTab
ModalStep
ProfileCardType
```

이런 타입은 모바일 화면 맥락이 강하므로 `apps/mobile/src/shared` 또는 각 역할 폴더에 둔다.

---

## 9. `packages/ui`

순수 UI 컴포넌트를 관리한다.

### 포함 대상

```txt
Button
Text
Card
Input
Modal
Badge
Avatar
IconButton
colors
typography
spacing
```

### 판단 기준

```txt
이 컴포넌트가 효잇의 도메인을 몰라도 되는가?
→ packages/ui
```

예를 들어 `Button`, `Card`, `Badge`는 부모/자녀/안부 상태를 몰라도 사용할 수 있으므로 `packages/ui`에 둔다.

### 넣지 않는 것

```txt
HomeStatusCard
ConnectedChildCard
ConnectedParentCard
CheckinReplyCard
ParentHomeBanner
ChildHomeBanner
```

이런 컴포넌트는 효잇의 도메인, 역할, 홈 화면 맥락을 알고 있으므로 `packages/ui`가 아니라 `apps/mobile/src` 안에 둔다.

---

## 10. `apps/mobile/src/shared`

`apps/mobile/src/shared`는 모바일 앱 내부에서 부모/자녀가 함께 사용하는 코드를 관리한다.

```txt
apps/mobile/src/shared
= 모바일 앱 안에서만 의미 있는 공통 코드
= parent/child가 함께 쓰지만 화면 맥락이 있는 코드
```

즉, 앱 밖에서도 독립적으로 설명 가능한 기반 코드는 `packages`에 두고, 모바일 화면 맥락이 있어야 설명 가능한 공통 코드는 `src/shared`에 둔다.

---

### 포함 대상

```txt
홈 상태 타입
홈 상태 문구
역할별 라벨
모바일 전용 util
모바일 전용 hook
부모/자녀가 함께 쓰는 화면 규칙
```

### 예시 구조

```txt
apps/mobile/src/shared/
  model/
    homeStatus.ts
  constants/
    homeLabels.ts
  lib/
    formatDate.ts
  hooks/
    useModal.ts
  ui/
    HomeStatusBadge.tsx
```

---

## 11. `parent`

`apps/mobile/src/parent`는 부모 플로우 전용 코드를 관리한다.

```txt
apps/mobile/src/parent/
  pages/
  widgets/
  features/
  entities/
```

### 포함 대상

```txt
부모 홈 화면
부모 프로필 화면
부모 로그 화면
자녀 상태 카드
안부 보내기 기능
부모 기준 확인 상태 UI
```

### 판단 기준

```txt
부모 사용자 경험 때문에 바뀌는가?
→ parent
```

부모와 자녀 화면이 비슷하게 생겼더라도, 문구나 액션 흐름이 부모 기준으로 바뀐다면 `parent`에 둔다.

### 예시

```txt
apps/mobile/src/parent/widgets/home-status-card
apps/mobile/src/parent/widgets/connected-child-card
apps/mobile/src/parent/features/send-checkin
apps/mobile/src/parent/features/confirm-checkin
apps/mobile/src/parent/entities/child
```

---

## 12. `child`

`apps/mobile/src/child`는 자녀 플로우 전용 코드를 관리한다.

```txt
apps/mobile/src/child/
  pages/
  widgets/
  features/
  entities/
```

### 포함 대상

```txt
자녀 홈 화면
자녀 프로필 화면
자녀 로그 화면
부모님 상태 카드
안부 답장 기능
자녀 기준 확인 상태 UI
```

### 판단 기준

```txt
자녀 사용자 경험 때문에 바뀌는가?
→ child
```

부모 화면과 비슷해 보여도 역할 의미가 다르면 분리한다.

### 예시

```txt
apps/mobile/src/child/widgets/home-status-card
apps/mobile/src/child/widgets/connected-parent-card
apps/mobile/src/child/features/reply-checkin
apps/mobile/src/child/features/confirm-checkin
apps/mobile/src/child/entities/parent
```

---

## 13. FSD 레이어 기준

효잇의 `parent`, `child` 내부에서는 다음 기준으로 코드를 나눈다.

---

### 13-1. `pages`

라우트와 직접 연결되는 화면 단위이다.

```txt
parent/pages/home
child/pages/home
```

화면 전체를 조합하는 역할을 한다.

---

### 13-2. `widgets`

화면을 구성하는 큰 UI 블록이다.

```txt
home-header
home-status-card
connected-child-card
recent-log-section
profile-info-card
```

여러 `features`, `entities`, `shared/ui`, `packages/ui`를 조합할 수 있다.

---

### 13-3. `features`

사용자 행동 단위이다.

```txt
send-checkin
reply-checkin
confirm-checkin
connect-family
edit-profile
```

버튼, 모달, mutation hook, action handler가 포함될 수 있다.

---

### 13-4. `entities`

도메인 데이터 단위이다.

```txt
user
parent
child
checkin
connection
notification
```

타입, mapper, query hook, 도메인 util 등을 둘 수 있다.

---

### 13-5. `shared`

도메인 맥락이 약하거나, 모바일 내부에서 공통으로 쓰는 코드이다.

```txt
formatDate
homeStatus
roleLabel
useModal
HomeStatusBadge
```

---

## 14. 위치 결정 규칙

코드 위치는 다음 순서로 판단한다.

```txt
1. 앱 없어도 설명 가능한 기반 코드인가?
   예: auth, storage, UserRole, Button, ApiResponse
   → packages

2. 모바일 앱 안에서 부모/자녀가 함께 쓰는 화면 규칙인가?
   예: HomeStatus, HOME_STATUS_LABEL, formatDateForLog
   → apps/mobile/src/shared

3. 부모 사용자 경험 때문에 바뀌는가?
   예: ParentHomeStatusCard, ConnectedChildCard
   → apps/mobile/src/parent

4. 자녀 사용자 경험 때문에 바뀌는가?
   예: ChildHomeStatusCard, ConnectedParentCard
   → apps/mobile/src/child
```

---

## 15. 공통화 기준

비슷해 보인다는 이유만으로 공통화하지 않는다.

공통화 기준은 다음이다.

```txt
같이 쓰는가? 보다
같이 바뀌는가? 를 기준으로 판단한다.
```

---

### 15-1. 분리해야 하는 경우

다음에 해당하면 공통화하지 않는다.

```txt
문구가 다르다.
사용자 역할이 다르다.
액션 흐름이 다르다.
요구사항이 따로 바뀔 가능성이 높다.
```

예를 들어 부모 홈 카드와 자녀 홈 카드는 비슷해 보여도 역할 의미가 다르므로 분리한다.

```txt
apps/mobile/src/parent/widgets/home-status-card
apps/mobile/src/child/widgets/home-status-card
```

---

### 15-2. 공통화해도 되는 경우

다음에 해당하면 공통화할 수 있다.

```txt
타입이 같다.
상태 라벨 규칙이 같다.
날짜 포맷이 같다.
역할 구분 기준이 같다.
부모/자녀가 같은 이유로 변경된다.
```

예:

```txt
apps/mobile/src/shared/model/homeStatus.ts
apps/mobile/src/shared/lib/formatDate.ts
```

---

## 16. push / API / 화면 상태 구분

부모와 자녀가 push로 값을 주고받는다고 해서, 무조건 같은 UI 코드로 관리해야 하는 것은 아니다.

push나 API로 주고받는 값은 통신 계약이다.
화면에서 그 값을 어떻게 보여줄지는 역할별 UX이다.

---

### 16-1. 서버/API/push 계약 타입

```txt
packages/types
```

예:

```ts
export type PushEventType =
  | "CHECKIN_RECEIVED"
  | "CHECKIN_CHECKED"
  | "CHECKIN_REPLIED";

export type PushPayload = {
  type: PushEventType;
  checkinId: string;
  senderRole: "parent" | "child";
};
```

---

### 16-2. 모바일 화면 상태

```txt
apps/mobile/src/shared/model
```

예:

```ts
export type HomeStatus = "received" | "empty" | "multiple" | "sent" | "checked";
```

---

### 16-3. 역할별 화면 해석

```txt
apps/mobile/src/parent
apps/mobile/src/child
```

예:

```txt
부모 화면:
자녀 확인 전
자녀 확인함

자녀 화면:
부모님 확인 전
부모님 확인함
```

---

## 17. 판단 예시

| 코드                   | 설명                         | 위치                              |
| ---------------------- | ---------------------------- | --------------------------------- |
| `Button`               | 기본 버튼                    | `packages/ui`                     |
| `Card`                 | 기본 카드                    | `packages/ui`                     |
| `Badge`                | 기본 뱃지                    | `packages/ui`                     |
| `useAuthStore`         | 인증 상태 store              | `packages/auth`                   |
| `storage`              | 저장소 wrapper               | `packages/storage`                |
| `UserRole`             | 사용자 역할 타입             | `packages/types`                  |
| `ApiResponse`          | 서버 응답 공통 타입          | `packages/types`                  |
| `PushPayload`          | push 데이터 계약 타입        | `packages/types`                  |
| `PushEventType`        | push 이벤트 타입             | `packages/types`                  |
| `HomeStatus`           | 모바일 홈 상태 표현 타입     | `apps/mobile/src/shared/model`    |
| `HOME_STATUS_LABEL`    | 홈 상태 문구                 | `apps/mobile/src/shared/model`    |
| `formatDateForLog`     | 로그용 날짜 포맷             | `apps/mobile/src/shared/lib`      |
| `HomeStatusBadge`      | 홈 상태를 보여주는 모바일 UI | `apps/mobile/src/shared/ui`       |
| `ParentHomeStatusCard` | 부모 홈 상태 카드            | `apps/mobile/src/parent/widgets`  |
| `ChildHomeStatusCard`  | 자녀 홈 상태 카드            | `apps/mobile/src/child/widgets`   |
| `ConnectedChildCard`   | 부모 화면의 연결 자녀 카드   | `apps/mobile/src/parent/widgets`  |
| `ConnectedParentCard`  | 자녀 화면의 연결 부모 카드   | `apps/mobile/src/child/widgets`   |
| `send-checkin`         | 부모의 안부 보내기 기능      | `apps/mobile/src/parent/features` |
| `reply-checkin`        | 자녀의 안부 답장 기능        | `apps/mobile/src/child/features`  |

---

## 18. 최종 기준

효잇에서는 다음 기준으로 구조를 잡는다.

```txt
packages
= 앱 기반 코드
= 앱 없어도 설명 가능한 코드
= auth, storage, types, ui

apps/mobile/src/shared
= 모바일 앱 내부 공통 코드
= parent/child가 함께 쓰지만 화면 맥락이 있는 코드

apps/mobile/src/parent
= 부모 플로우 전용 코드

apps/mobile/src/child
= 자녀 플로우 전용 코드
```

가장 단순한 판단 기준은 다음이다.

```txt
앱 없어도 설명 가능하면 packages
모바일 화면이 있어야 설명 가능하면 apps/mobile/src/shared
부모 경험 때문에 바뀌면 parent
자녀 경험 때문에 바뀌면 child
```
