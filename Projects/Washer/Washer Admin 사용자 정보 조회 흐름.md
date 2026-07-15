# Washer Admin 사용자 정보 조회 흐름

## `useGetMyInfo`부터 `AdminLayout` 권한 검사까지

## 1. 학습 목표

이 글에서는 `entities/user`에 있는 `useGetMyInfo`가 실제로 어떻게 실행되고, 조회한 사용자 정보가 `AdminLayout`의 권한 검사까지 어떻게 전달되는지 정리한다.

핵심적으로 이해해야 할 내용은 다음과 같다.

- API 응답 타입과 실제 데이터 타입의 차이
- TanStack Query의 `queryKey` 역할
- `staleTime`과 `gcTime`의 차이
- `useQuery`가 반환하는 데이터와 상태
- 구조 분해 할당과 별칭
- 조건부 Query 실행
- Layout에서 공통 권한을 검사하는 이유

---

# 2. 전체 구조

관련 파일은 크게 `entities`, `shared`, `widgets`로 나뉜다.

```text
src/
├── entities/
│   └── user/
│       ├── model/
│       │   └── types.ts
│       ├── api/
│       │   └── useGetMyInfo.ts
│       └── index.ts
│
├── shared/
│   ├── api/
│   │   ├── queryKeys.ts
│   │   ├── urls.ts
│   │   ├── types.ts
│   │   └── http.ts
│   │
│   └── constants/
│       └── queryOptions.ts
│
└── widgets/
    └── layout/
        └── admin-layout/
            └── ui/
                └── AdminLayout.tsx
```

실제 요청 흐름은 다음과 같다.

```text
AdminLayout
→ useGetMyInfo
→ useQuery
→ queryKey 확인
→ queryFn 실행
→ GET 요청
→ 서버 응답
→ Query 캐시에 저장
→ AdminLayout 재렌더링
→ 사용자 role 검사
```

---

# 3. 사용자 데이터 타입 정의

현재 로그인한 사용자의 실제 데이터는 `MyInfoType`으로 정의한다.

```ts
export type UserRole = "ADMIN" | "USER" | "DORMITORY_COUNCIL";

export interface MyInfoType {
  id: number;
  name: string;
  studentId: string;
  roomNumber: string;
  grade: number;
  floor: number;
  penaltyCount: number;
  createdAt: string;
  updatedAt: string;
  canReserve: boolean;
  penaltyExpiresAt: string;
  role: UserRole;
}
```

`MyInfoType`은 서버 응답 전체를 나타내는 타입이 아니다.

응답 안에 들어 있는 **실제 사용자 데이터 부분**만 표현한다.

```json
{
  "id": 1,
  "name": "김봄",
  "studentId": "20260001",
  "roomNumber": "301",
  "role": "ADMIN"
}
```

## 개념: 도메인 타입

`MyInfoType`은 `user` 도메인의 데이터를 표현한다.

```text
User 도메인
├── id
├── name
├── studentId
├── roomNumber
└── role
```

따라서 `entities/user`에서 관리하는 것이 자연스럽다.

---

# 4. 공통 API 응답 타입

백엔드는 사용자 데이터만 바로 반환하지 않고, 공통 응답 객체 안에 데이터를 감싸서 전달한다.

```ts
export interface BaseResponseType<T> {
  status: string;
  code: number;
  message: string;
  data: T;
}
```

`T`에는 API마다 다른 실제 데이터 타입이 들어간다.

내 정보 조회 API에서는 다음과 같다.

```ts
BaseResponseType<MyInfoType>;
```

타입을 실제 구조로 풀면 다음과 같다.

```ts
interface MyInfoResponse {
  status: string;
  code: number;
  message: string;
  data: MyInfoType;
}
```

서버 응답 예시는 다음과 같다.

```json
{
  "status": "SUCCESS",
  "code": 200,
  "message": "내 정보 조회에 성공했습니다.",
  "data": {
    "id": 1,
    "name": "김봄",
    "studentId": "20260001",
    "roomNumber": "301",
    "role": "ADMIN"
  }
}
```

## 타입 흐름

```text
BaseResponseType<MyInfoType>
├── status
├── code
├── message
└── data
    └── MyInfoType
```

따라서 각 값의 타입은 다음과 같다.

```ts
response;
// BaseResponseType<MyInfoType>

response.data;
// MyInfoType

response.data.role;
// UserRole
```

## 개념: 제네릭

`BaseResponseType<T>`의 `T`는 아직 정해지지 않은 타입 자리다.

```ts
BaseResponseType<MyInfoType>;
BaseResponseType<UserListType>;
BaseResponseType<MachineType>;
```

공통 응답 구조는 유지하면서 내부 `data` 타입만 바꿀 수 있다.

---

# 5. QueryKey 정의

사용자 관련 QueryKey는 한곳에서 관리한다.

```ts
export const userQueryKeys = {
  all: ["users"] as const,

  getUsers: (params?: UserParamsType) => ["users", "list", params] as const,

  getMyInfo: () => ["users", "my"] as const,
} as const;
```

내 정보 조회에서는 다음 함수를 사용한다.

```ts
userQueryKeys.getMyInfo();
```

반환값은 다음과 같다.

```ts
["users", "my"];
```

## QueryKey란

QueryKey는 API URL이 아니다.

TanStack Query가 데이터를 구분하고 저장하기 위해 사용하는 **캐시 식별자**다.

```text
["users", "my"]
→ 현재 로그인한 사용자 정보

["users", "list", { page: 1 }]
→ 사용자 목록 1페이지

["users", "list", { page: 2 }]
→ 사용자 목록 2페이지
```

TanStack Query의 캐시를 단순화하면 다음과 같다.

```text
Query Cache
├── ["users", "my"]
│   └── 현재 사용자 정보
│
├── ["users", "list", { page: 1 }]
│   └── 사용자 목록 1페이지
│
└── ["users", "list", { page: 2 }]
    └── 사용자 목록 2페이지
```

## 왜 함수로 관리하는가

컴포넌트마다 배열을 직접 작성하면 규칙이 달라질 수 있다.

```ts
["user", "my"][("users", "me")][("users", "myInfo")];
```

같은 데이터를 조회하면서 서로 다른 키를 사용하면 캐시가 분리된다.

그래서 QueryKey 생성 규칙을 한곳에 모은다.

```ts
queryKey: userQueryKeys.getMyInfo();
```

## `as const`의 역할

```ts
["users", "my"] as const;
```

일반 배열이 아니라 수정할 수 없는 읽기 전용 튜플로 추론하게 한다.

```ts
string[]
```

이 아니라 다음과 같이 더 구체적인 타입이 된다.

```ts
readonly[("users", "my")];
```

TanStack Query에서 QueryKey의 타입을 안정적으로 유지하는 데 도움이 된다.

---

# 6. staleTime 정책

서버 데이터별 신선도 기준을 공통 상수로 관리한다.

```ts
const SECOND = 1000;
const MINUTE = 60 * SECOND;

export const STALE_TIME = {
  MACHINE: 30 * SECOND,
  RESERVATION: MINUTE,
  REPORT: 3 * MINUTE,
  USER: 3 * MINUTE,
  MY_INFO: 30 * MINUTE,
} as const;
```

내 정보 조회에서는 다음 값을 사용한다.

```ts
STALE_TIME.MY_INFO;
```

즉, 30분이다.

## staleTime이란

데이터를 서버에서 가져온 뒤 얼마 동안 해당 데이터를 최신 상태인 `fresh`로 판단할지 정하는 값이다.

```text
내 정보 조회 성공
→ Query 캐시에 저장
→ 30분 동안 fresh 상태
→ 같은 QueryKey 사용 시 캐시 데이터 재사용
```

30분이 지나면 데이터가 즉시 삭제되는 것은 아니다.

데이터 상태가 `fresh`에서 `stale`로 바뀔 뿐이다.

```text
fresh
→ 아직 최신이라고 판단하는 상태

stale
→ 오래됐을 가능성이 있다고 판단하는 상태
```

## staleTime과 gcTime의 차이

### staleTime

데이터를 언제 오래된 것으로 판단할지 정한다.

```text
staleTime
→ 데이터의 신선도
```

### gcTime

어떤 컴포넌트에서도 사용하지 않는 Query 캐시를 언제 메모리에서 제거할지 정한다.

```text
gcTime
→ 사용하지 않는 캐시의 보관 시간
```

예를 들어 다음과 같이 설정할 수 있다.

```text
staleTime: 30분
gcTime: 30분
```

이 경우 데이터는 조회 후 30분 동안 fresh 상태이며, Query를 사용하는 컴포넌트가 사라진 뒤 최대 30분 동안 캐시에 남는다.

## 데이터마다 staleTime이 다른 이유

데이터의 변경 빈도가 다르기 때문이다.

```text
세탁기 상태
→ 사용 직후 자주 변경됨
→ 짧은 staleTime

예약 정보
→ 비교적 자주 변경됨
→ 중간 staleTime

현재 사용자 정보와 권한
→ 세션 중 자주 변경되지 않음
→ 긴 staleTime
```

---

# 7. `useGetMyInfo` Query Hook

현재 사용자 정보를 조회하는 Hook은 다음과 같다.

```ts
import { useQuery } from "@tanstack/react-query";
import { get, userQueryKeys, userUrl } from "@/shared/api";
import type { BaseResponseType } from "@/shared/api/types";
import { STALE_TIME } from "@/shared/constants/queryOptions";
import type { MyInfoType } from "../model/types";

export const useGetMyInfo = () => {
  return useQuery({
    staleTime: STALE_TIME.MY_INFO,
    queryKey: userQueryKeys.getMyInfo(),
    queryFn: () => get<BaseResponseType<MyInfoType>>(userUrl.getMyInfo()),
  });
};
```

---

# 8. `useQuery`가 하는 일

```ts
return useQuery({
```

`useQuery`는 단순히 API 응답만 반환하지 않는다.

서버 요청 결과와 요청 상태를 함께 담은 **Query 결과 객체**를 반환한다.

```ts
{
  data,
  error,
  isLoading,
  isPending,
  isFetching,
  isError,
  isSuccess,
  refetch,
  ...
}
```

현재 Hook은 이 Query 결과 객체를 그대로 반환한다.

```ts
return useQuery({ ... });
```

설명용으로 풀어쓰면 다음과 같다.

```ts
export const useGetMyInfo = () => {
  const queryResult = useQuery({
    queryKey: userQueryKeys.getMyInfo(),
    queryFn: () => get<BaseResponseType<MyInfoType>>(userUrl.getMyInfo()),
    staleTime: STALE_TIME.MY_INFO,
  });

  return queryResult;
};
```

여기서 `queryResult`는 이해를 위해 임시로 만든 변수명이다.

실제 코드에서는 `useQuery` 결과를 바로 반환한다.

---

# 9. queryKey의 실제 사용

```ts
queryKey: userQueryKeys.getMyInfo(),
```

실제로는 다음 값이 전달된다.

```ts
["users", "my"];
```

TanStack Query는 이 키를 기준으로 다음 작업을 수행한다.

1. 같은 키의 캐시가 존재하는지 확인
2. 데이터가 fresh인지 stale인지 확인
3. 요청 결과를 해당 키에 저장
4. 여러 컴포넌트에서 같은 데이터를 공유
5. 나중에 캐시 무효화 대상을 찾음

따라서 QueryKey는 선언만 해두는 값이 아니라 `useQuery` 안에서 실제 캐시 주소로 사용된다.

---

# 10. queryFn의 역할

```ts
queryFn: () =>
  get<BaseResponseType<MyInfoType>>(
    userUrl.getMyInfo(),
  ),
```

`queryFn`은 실제 서버 요청을 수행하는 함수다.

실행 흐름은 다음과 같다.

```text
userUrl.getMyInfo()
→ 내 정보 API URL 반환

get<T>()
→ 해당 URL로 GET 요청

BaseResponseType<MyInfoType>
→ 서버 응답 타입 지정
```

`queryFn`이 반환하는 것은 API 요청 결과다.

```ts
BaseResponseType<MyInfoType>;
```

`isLoading`, `isError` 같은 상태는 `queryFn`이 직접 반환하지 않는다.

TanStack Query가 `queryFn` 실행 과정을 관찰해 자동으로 생성한다.

```text
queryFn 실행 시작
→ 로딩 상태

queryFn 성공
→ data 저장
→ 성공 상태

queryFn 실패
→ error 저장
→ 에러 상태
```

---

# 11. `useGetMyInfo`의 반환 타입

`useGetMyInfo()`의 반환값은 개념적으로 다음과 같다.

```ts
{
  data: BaseResponseType<MyInfoType> | undefined;
  error: Error | null;
  isLoading: boolean;
  isPending: boolean;
  isFetching: boolean;
  isError: boolean;
  isSuccess: boolean;
  refetch: () => Promise<unknown>;
}
```

`data`에 `undefined`가 포함되는 이유는 최초 렌더링 시 아직 요청이 완료되지 않았을 수 있기 때문이다.

```text
최초 렌더링
→ data: undefined
→ 요청 진행

요청 성공 후
→ data: BaseResponseType<MyInfoType>
```

---

# 12. `isLoading`, `isPending`, `isFetching` 차이

TanStack Query 버전에 따라 자주 헷갈리는 부분이다.

## isPending

아직 성공한 데이터가 없는 초기 상태를 의미한다.

```text
데이터 없음
→ isPending: true
```

## isLoading

TanStack Query v5에서는 일반적으로 다음 상태를 의미한다.

```text
isPending && isFetching
```

즉, 처음 데이터를 가져오는 중인 상태다.

## isFetching

실제로 queryFn이 실행 중인지 나타낸다.

초기 조회뿐 아니라 백그라운드 재조회에서도 `true`가 된다.

```text
첫 요청
→ isPending: true
→ isFetching: true

기존 데이터가 있는 상태에서 재조회
→ isPending: false
→ isFetching: true
```

따라서 화면 전체 로딩은 `isLoading` 또는 `isPending`, 작은 갱신 표시에는 `isFetching`을 사용할 수 있다.

---

# 13. AdminLayout에서 Hook 호출

`AdminLayout`에서는 다음과 같이 Hook을 사용한다.

```ts
const {
  data: myInfoData,
  isLoading: isMyInfoLoading,
  isError: isMyInfoError,
} = useGetMyInfo();
```

이 코드는 구조 분해 할당과 별칭을 사용한다.

원래 반환 객체의 속성명은 다음과 같다.

```ts
data;
isLoading;
isError;
```

AdminLayout에는 다른 Query도 존재하기 때문에 의미가 명확하도록 이름을 바꾼다.

```ts
data: myInfoData;
isLoading: isMyInfoLoading;
isError: isMyInfoError;
```

풀어쓰면 다음 코드와 같다.

```ts
const queryResult = useGetMyInfo();

const myInfoData = queryResult.data;
const isMyInfoLoading = queryResult.isLoading;
const isMyInfoError = queryResult.isError;
```

## 개념: 구조 분해 할당 별칭

```ts
const { data: myInfoData } = queryResult;
```

이 코드에서:

```text
data
→ 원래 객체의 속성 이름

myInfoData
→ 현재 코드에서 사용할 변수 이름
```

객체의 속성명이 바뀌는 것이 아니라, 꺼내온 값의 변수명만 바뀐다.

---

# 14. Query 데이터와 실제 사용자 데이터

```ts
const myInfo = myInfoData?.data;
```

여기서 `data`가 두 번 등장하기 때문에 처음 보면 헷갈릴 수 있다.

두 `data`는 서로 다른 계층이다.

```text
첫 번째 data
→ TanStack Query 결과의 data
→ API 응답 전체

두 번째 data
→ API 응답 객체 내부의 data
→ 실제 사용자 정보
```

전체 구조는 다음과 같다.

```text
useGetMyInfo() 반환값
├── data
│   └── BaseResponseType<MyInfoType>
│       ├── status
│       ├── code
│       ├── message
│       └── data
│           └── MyInfoType
├── isLoading
├── isError
└── isFetching
```

타입 흐름으로 보면 다음과 같다.

```ts
const myInfoData = queryResult.data;
// BaseResponseType<MyInfoType> | undefined

const myInfo = myInfoData?.data;
// MyInfoType | undefined

const role = myInfo?.role;
// UserRole | undefined
```

---

# 15. Optional Chaining

```ts
const myInfo = myInfoData?.data;
```

최초 렌더링에서는 요청이 끝나지 않았기 때문에 다음 상태일 수 있다.

```ts
myInfoData === undefined;
```

이때 아래처럼 접근하면 오류가 발생한다.

```ts
const myInfo = myInfoData.data;
```

`undefined`에서 `data`를 읽으려고 하기 때문이다.

따라서 optional chaining을 사용한다.

```ts
myInfoData?.data;
```

의미는 다음과 같다.

```text
myInfoData가 존재하면
→ myInfoData.data 반환

myInfoData가 undefined 또는 null이면
→ undefined 반환
```

---

# 16. 인증 실패 처리

AdminLayout에서는 사용자 정보 조회 실패를 로그인 만료 또는 잘못된 인증 상태로 처리한다.

```ts
useEffect(() => {
  if (isMyInfoError) {
    toast.error(
      "로그인이 만료되었거나 유효하지 않습니다. 다시 로그인해주세요."
    );

    queryClient.clear();

    deleteCookie(COOKIE_KEYS.ACCESS_TOKEN);
    deleteCookie(COOKIE_KEYS.REFRESH_TOKEN);

    window.location.href = "/sign-in";
    return;
  }
}, [isMyInfoError, queryClient]);
```

실행 흐름은 다음과 같다.

```text
내 정보 요청 실패
→ isMyInfoError = true
→ 사용자에게 오류 메시지 표시
→ Query 캐시 삭제
→ 인증 쿠키 삭제
→ 로그인 페이지 이동
```

## 왜 Query 캐시를 지우는가

로그아웃이나 인증 만료 후 이전 사용자의 서버 데이터가 캐시에 남아 있으면 안 되기 때문이다.

```ts
queryClient.clear();
```

를 사용하면 Query Client에 저장된 캐시를 전체 제거한다.

---

# 17. 사용자 역할 검사

사용자 정보 조회에 성공하면 역할을 검사한다.

```ts
if (myInfo && myInfo.role === "USER") {
  toast.error("관리자만 접근 가능합니다.");

  queryClient.clear();

  deleteCookie(COOKIE_KEYS.ACCESS_TOKEN);
  deleteCookie(COOKIE_KEYS.REFRESH_TOKEN);

  window.location.href = "/app-download";
}
```

조건을 풀어보면 다음과 같다.

```text
myInfo가 존재하고
role이 USER라면
관리자 페이지 접근 차단
```

실행 흐름은 다음과 같다.

```text
내 정보 요청 성공
→ myInfo 추출
→ role 확인
→ USER인 경우 접근 차단
→ 캐시와 토큰 제거
→ 앱 다운로드 페이지 이동
```

현재 코드에서 차단하는 역할은 다음 하나다.

```text
USER
```

따라서 실제로 허용되는 역할은 다음과 같다.

```text
ADMIN
DORMITORY_COUNCIL
```

즉, 코드상으로는 “ADMIN만 허용”이 아니라 “일반 USER만 차단”하는 구조다.

관리자만 허용하려면 다음처럼 작성하는 편이 더 명확하다.

```ts
if (myInfo && myInfo.role !== "ADMIN") {
  // 접근 차단
}
```

기숙사 자치회까지 허용하려면 허용 역할을 명시할 수 있다.

```ts
const allowedRoles: UserRole[] = ["ADMIN", "DORMITORY_COUNCIL"];

const hasAdminAccess = myInfo && allowedRoles.includes(myInfo.role);
```

허용 역할을 명시하는 방식은 새로운 역할이 추가될 때 의도하지 않은 접근 허용을 막는 데 유리하다.

---

# 18. 권한 확인 전 로딩 처리

AdminLayout에서는 권한 확인이 끝나기 전까지 관리자 UI를 렌더링하지 않는다.

```tsx
if (
  isMyInfoLoading ||
  isMyInfoError ||
  (myInfo && myInfo.role === "USER") ||
  (!myInfo && !isMyInfoError)
) {
  return <LoadingSpinner />;
}
```

각 조건의 의미는 다음과 같다.

## 요청 중

```ts
isMyInfoLoading;
```

```text
아직 사용자 role을 알 수 없음
→ 관리자 화면 렌더링 중단
```

## 요청 실패

```ts
isMyInfoError;
```

```text
useEffect의 리다이렉트가 실행되기 전까지
→ 관리자 화면 숨김
```

## 일반 사용자

```ts
myInfo && myInfo.role === "USER";
```

```text
접근 권한 없음
→ 리다이렉트 전까지 관리자 화면 숨김
```

## 사용자 정보가 아직 없음

```ts
!myInfo && !isMyInfoError;
```

```text
에러는 아니지만 사용자 데이터가 준비되지 않음
→ 권한 확인 전이므로 화면 숨김
```

이 처리는 권한이 없는 사용자가 관리자 UI를 짧게라도 보는 화면 깜빡임을 방지한다.

---

# 19. 조건부 Query 실행

관리자 권한이 확인된 뒤에만 대시보드 데이터를 조회한다.

```ts
const { data, isLoading, isError } = useQuery({
  queryKey: ["dashboard", "summary"],
  queryFn: getDashboardSummary,

  enabled: !!myInfo && myInfo.role !== "USER" && !isMyInfoError,
});
```

핵심은 `enabled` 옵션이다.

```ts
enabled: !!myInfo && myInfo.role !== "USER" && !isMyInfoError;
```

조건을 풀어보면 다음과 같다.

```text
myInfo가 존재하고
role이 USER가 아니며
내 정보 요청에 실패하지 않았을 때
대시보드 Query 실행
```

## `!!myInfo`의 의미

`myInfo` 값을 boolean으로 변환한다.

```ts
!!undefined;
// false

!!null;
// false

!!{ id: 1 };
// true
```

따라서 다음과 같은 의미다.

```ts
!!myInfo;
// 사용자 정보가 존재하는가
```

## enabled의 역할

기본적으로 `useQuery`는 컴포넌트가 렌더링되면 바로 실행된다.

하지만 `enabled: false`이면 Query를 자동 실행하지 않는다.

```text
enabled: false
→ Query 대기

enabled: true
→ queryFn 실행
```

현재 구조에서는 요청 순서가 다음처럼 된다.

```text
1. 현재 사용자 정보 조회
2. 사용자 role 확인
3. 접근 가능한 사용자라면
4. 대시보드 정보 조회
```

이를 **의존적인 Query** 또는 **Dependent Query**라고 볼 수 있다.

두 번째 Query가 첫 번째 Query의 결과에 의존하기 때문이다.

---

# 20. 전체 실행 흐름

관리자 페이지에 접근했을 때의 전체 흐름은 다음과 같다.

```text
1. 관리자 Route 접근

2. AdminLayout 렌더링

3. useGetMyInfo 호출

4. useQuery가 QueryKey 확인
   ["users", "my"]

5. Query 캐시 확인

6-1. fresh 캐시가 존재함
   → 서버 요청 없이 캐시 사용

6-2. 캐시가 없거나 재조회가 필요함
   → queryFn 실행
   → userUrl.getMyInfo()
   → get<BaseResponseType<MyInfoType>>()
   → 서버 요청

7. 최초 요청 중
   → 로딩 화면 렌더링

8. 요청 성공
   → API 응답을 Query 캐시에 저장

9. AdminLayout 재렌더링

10. myInfoData?.data 실행
    → 실제 사용자 정보 추출

11. myInfo.role 검사

12-1. USER
    → 접근 차단
    → 캐시 및 인증 쿠키 제거
    → /app-download 이동

12-2. ADMIN 또는 DORMITORY_COUNCIL
    → 대시보드 Query enabled = true

13. 대시보드 요약 정보 조회

14. AdminLayout 공통 UI 렌더링

15. children 위치에 현재 관리자 페이지 렌더링
```

---

# 21. 타입 변화 정리

이 흐름에서 타입이 어떻게 바뀌는지만 따로 보면 다음과 같다.

```ts
get<BaseResponseType<MyInfoType>>(...)
```

반환 타입:

```ts
BaseResponseType<MyInfoType>;
```

`useQuery` 결과 안에서는:

```ts
queryResult.data;
// BaseResponseType<MyInfoType> | undefined
```

구조 분해 할당 후:

```ts
myInfoData;
// BaseResponseType<MyInfoType> | undefined
```

실제 사용자 데이터 추출 후:

```ts
myInfoData?.data;
// MyInfoType | undefined
```

사용자 역할 접근:

```ts
myInfo?.role;
// UserRole | undefined
```

한 줄로 정리하면 다음과 같다.

```text
BaseResponseType<MyInfoType>
→ .data
→ MyInfoType
→ .role
→ UserRole
```

---

# 22. QueryKey가 사용되는 위치

QueryKey는 주로 두 가지 상황에서 사용된다.

## 데이터를 조회할 때

```ts
useQuery({
  queryKey: userQueryKeys.getMyInfo(),
  queryFn: getMyInfo,
});
```

```text
QueryKey 기준으로
→ 캐시 조회
→ 데이터 저장
→ 데이터 공유
```

## 데이터를 무효화할 때

```ts
queryClient.invalidateQueries({
  queryKey: userQueryKeys.all,
});
```

`userQueryKeys.all`은 다음 값이다.

```ts
["users"];
```

이 키를 기준으로 사용자 관련 Query를 무효화할 수 있다.

```text
["users", "my"]
["users", "list", params]
```

즉 QueryKey는 다음 두 역할을 한다.

```text
useQuery
→ 캐시 조회와 저장 기준

invalidateQueries
→ 캐시 무효화 대상 검색
```

## invalidateQueries란

캐시 데이터를 바로 삭제하는 것이 아니라 오래된 데이터로 표시하고, 활성 상태인 Query라면 다시 요청하도록 만든다.

```ts
queryClient.invalidateQueries({
  queryKey: userQueryKeys.all,
});
```

```text
사용자 관련 데이터가 변경됨
→ 기존 사용자 캐시 stale 처리
→ 활성 Query 재조회
→ 최신 서버 데이터 반영
```

---

# 23. 왜 Query Hook으로 감싸는가

컴포넌트에서 직접 `useQuery`를 작성할 수도 있다.

```ts
useQuery({
  queryKey: ["users", "my"],
  queryFn: () => get("/users/my"),
  staleTime: 30 * 60 * 1000,
});
```

하지만 이렇게 작성하면 컴포넌트가 너무 많은 세부 구현을 알아야 한다.

```text
컴포넌트가 알아야 하는 내용
├── API URL
├── HTTP 요청 함수
├── QueryKey
├── 응답 타입
└── staleTime
```

Query Hook으로 감싸면 컴포넌트에서는 다음 코드만 사용하면 된다.

```ts
useGetMyInfo();
```

세부 구현은 `entities/user` 내부에 숨겨진다.

```text
AdminLayout
→ 현재 사용자 정보가 필요하다는 것만 알고 있음

useGetMyInfo
→ 실제 조회 방법과 캐싱 정책을 알고 있음
```

이를 관심사 분리라고 볼 수 있다.

---

# 24. 왜 `entities/user`에 두는가

현재 사용자 정보 조회는 로그인 버튼을 누르는 행동이 아니라 `user` 도메인의 서버 데이터를 가져오는 작업이다.

```text
로그인 폼 제출
→ 사용자 행동
→ features/auth/sign-in

현재 로그인한 사용자 정보 조회
→ user 도메인 데이터
→ entities/user
```

FSD 관점에서 `entities`는 비즈니스에서 사용되는 핵심 데이터와 관련 로직을 관리한다.

```text
entities/
├── user
├── machine
├── reservation
└── report
```

따라서 사용자 타입, 사용자 API 함수, 사용자 Query Hook을 `entities/user`에서 관리하는 것이 자연스럽다.

---

# 25. 왜 AdminLayout에서 권한을 검사하는가

관리자 페이지는 모두 공통으로 `AdminLayout`을 통과한다.

```text
AdminLayout
├── MachinesPage
├── ReservationsPage
├── ReportsPage
└── UsersPage
```

각 페이지에서 권한 검사를 작성하면 같은 코드가 반복된다.

```text
MachinesPage에서 권한 검사
ReservationsPage에서 권한 검사
ReportsPage에서 권한 검사
UsersPage에서 권한 검사
```

AdminLayout에서 한 번만 검사하면 하위 관리자 페이지 전체에 동일한 접근 정책을 적용할 수 있다.

```text
AdminLayout에서 권한 확인
→ 허용되면 children 렌더링
→ 허용되지 않으면 리다이렉트
```

이 방식의 장점은 다음과 같다.

- 권한 검사 코드 중복 감소
- 관리자 페이지별 정책 불일치 방지
- 권한 로직 수정 위치 단일화
- 하위 페이지 렌더링 전 접근 차단

---

# 26. 클라이언트 권한 검사의 한계

AdminLayout의 권한 검사는 사용자 경험을 위한 클라이언트 측 보호다.

하지만 이것만으로 서버 데이터까지 완전히 보호할 수는 없다.

사용자가 직접 관리자 API를 요청할 수도 있기 때문이다.

따라서 실제 보안은 백엔드에서도 반드시 검사해야 한다.

```text
프론트엔드 권한 검사
→ 관리자 화면 노출 방지
→ 잘못된 페이지 접근 차단

백엔드 권한 검사
→ 관리자 API 접근 자체를 차단
→ 실제 데이터 보호
```

즉, 프론트엔드 권한 검사는 보안의 최종 방어선이 아니라 UI 접근 제어 역할에 가깝다.

# 27. 최종 흐름 요약

```text
AdminLayout 렌더링
→ useGetMyInfo 호출
→ ["users", "my"] 캐시 확인
→ 필요하면 서버 요청
→ BaseResponseType<MyInfoType> 응답 저장
→ myInfoData?.data로 사용자 정보 추출
→ 사용자 role 검사
→ 권한이 있으면 대시보드 Query 활성화
→ 관리자 공통 UI와 하위 페이지 렌더링
```

---

# 28. 최종 한 문장

`useGetMyInfo`는 `["users", "my"]` QueryKey와 30분의 staleTime을 기준으로 현재 로그인한 사용자 정보를 조회하고 캐싱한다. `AdminLayout`은 TanStack Query가 반환한 데이터와 요청 상태를 받아 API 응답 내부의 실제 사용자 정보를 추출한 뒤 역할을 검사하며, 권한 확인이 완료된 경우에만 관리자 공통 데이터와 하위 페이지를 렌더링한다.
