# 예약 API 응답 런타임 검증 리팩토링

## 1. 리팩토링 배경

기존 예약 조회 코드는 서버 응답 타입을 다음과 같이 제네릭으로 지정했다.

```ts
get<BaseResponseType<ReservationResponseType>>(...)
```

기존 세탁기와 건조기 예약을 각각 요청하고, 응답을 `reservations` 배열을 Mapper에 전달해 UI 모델로 변환한 뒤 합치는 구조였다.

```
WASHER 예약 API 요청
DRYER 예약 API 요청
        ↓
ReservationResponseType이라고 간주
        ↓
mapReservations
        ↓
ReservationItem[]
        ↓
두 배열 병합
```

구조 자체는 다음 책임이 분리되어 있었다.

- `ReservationDTO` : 서버에서 전달되는 예약 한 건의 원본 구조
- `ReservationItem` : 화면에서 사용하는 예약 데이터 구조
- `mapReservation` : DTO를 UI 모델로 변환
- `getReservations` : API 요청과 데이터 병합
- `useGetReservations`: TanStack Query 캐시와 요청 상태 관리

문제는 **서버에서 전달된 실제 값이 `ReservationResponseType` 과 일치하는지 확인하는 단계가 없었다는 점** 이다.

## 2. 기존 코드에서 헷갈렸던 부분

### 2.1) TS 타입을 지정하면 실제 응답도 검사되는가?

```ts
const response =
  await get<BaseResponseType<ReservationResponseType>>(...);
```

처음에는 이 코드가 서버 응답의 구조를 실제로 검사한다고 생각하기 쉬웠다.

하지만 제네릭 타입은 서버 데이터를 검사하는 것이 아니다.

이 코드는 TS에게 다음과 같이 알려주는 역할이다.

> 이 요청의 response.data는 ReservationResponseType이라고 가정하고 사용한다.

즉, 서버에서 실제로 아래처럼 잘못된 값이 와도 TS가 런타임에 검사하지 않는다.

```json
{
  "reservations": [
    {
      "id": "1",
      "status": "WAITING"
    }
  ],
  "totalCount": "1"
}
```

문제점:

- id가 number가 아닌 string
- status가 정의되지 않은 "WAITING"
- totalCount가 number가 아닌 string
- 필수 필드 누락

TS는 빌드 과정에서 코드의 타입만 검사하고, 서버가 보내는 JSON까지 검사하지 않는다.

### 2.2) `ReservationDTO`와 `ReservationResponseType` 의 차이

기존 타입은 두 단계로 구성되어 있었다.

#### `ReservationDTO`

예약 한 건의 서버 응답 구조다.

```ts
export type ReservationDTO = {
  id: number;
  userId: number;
  userName: string;
  machineId: number;
  machineName: string;
  status: ReservationDTOStatus;
  machineAvailability: MachineAvailabilityStatus;
};
```

#### `ReservationResponseType`

예약 목록과 페이지 정보를 포함한 `response.data` 전체 구조다.

```ts
export interface ReservationResponseType {
  reservations: ReservationDTO[];
  totalCount: number;
  totalPages: number;
  currentPage: number;
}
```

```
ReservationResponseType
├─ reservations: ReservationDTO[]
├─ totalCount
├─ totalPages
└─ currentPage
```

처음에는 DTO와 목록 응답 타입이 같은 파일에 섞여 있어, 어떤 타입이 서버 응답 한 건이고, 어떤 타입이 목록 전체인지 구분하기 어려웠다.

### 2.3) `ReservationDTO` 와 `ReservationItem` 의 차이

두 타입 모두 예약 한 건을 표현하지만 용도가 다르다.

#### `ReservationDTO`

백엔드가 보내는 원본 데이터이다.

```ts
{
  machineName: "WASHER-1",
  status: "RESERVED",
  machineAvailability: "RESERVED",
  reservedAt: "2026-07-15T10:00:00"
}
```

#### ReservationItem

Mapper를 거쳐 화면에서 사용하기 쉽게 만든 데이터이다.

```ts
{
  machine: "WASHER-1",
  badgeStatus: "예약중",
  type: "WASHER",
  reserveAt: "2026. 07. 15. 10:00"
}
```

정리하면

```
ReservationDTO
→ 서버의 데이터 구조

ReservationItem
→ 화면의 데이터 구조
```

### 2.4) `확인필요` 와 API 오류의 차이

기존 Mapper에는 다음 fallback이 있었다.

```ts
return "확인필요";
```

처음에는 정의되지 안은 상태가 들어와도 `"확인필요"` 로 보여주면 충분하지 않을까라고 생각했다.

하지만 두 상황의 성격은 달랐다.

#### 정상적인 업무 상태

```ts
{
  status: "RUNNING",
  machineAvailability: "UNAVAILABLE"
}
```

- `RUNNING`은 정의된 예약 상태
- `UNAVAILABLE`도 정의된 기기 상태
- 데이터 계약은 정상
- 관리자 판단이 필요함

따라서 처리는 `확인 필요`가 옳음

#### API 계약 오류

```ts
{
  status: "WAITING";
}
```

- `"WAITING" 은 프론트와 백엔드가 정의한 상태에 없음
- 데이터 구조 또는 API 스펙이 깨진 상태

이 값을 `"확인필요"`로 처리하면 서버 계약 오류가 정상적인 업무 상태처럼 화면에 표시된다.

따라서 역할을 분리했다.

- **Zod**

  - API 응답이 계약에 맞는지 검사

- **Mapper의 확인필요**
  - 계약에 맞는 데이터 중 관리자의 확인이 필요한 상태 처리

## 3. Zod를 채택한 이유

### 3.1) 이미 DTO와 Mapper 구조가 존재했기 때문

기존 코드에는 이미 다음 구조가 있었다.

```
서버 응답
→ ReservationDTO
→ Mapper
→ ReservationItem
→ UI
```

다만 서버 응답이 실제로 `ReservationDTO` 구조인지 검사하는 단계가 없었다.

Zod는 기존 구조를 바꾸는 기술이 아니라, 비어 있던 검증 단계를 채우는 역할을 한다.

```
서버 응답
→ Zod 검증
→ ReservationDTO
→ Mapper
→ ReservationItem
→ UI
```

### 3.2) TS와 런타임 검증을 함께 사용하기 위해

TS는 개발 중 코드의 타입 오류를 잡는다.

- TS는 컴파일 시점 검사

Zod는 서버에서 들어온 실제 값을 검사한다.

- Zod는 실행 시점 검사

둘은 대체 관계가 아니라 역할이 다르다.

```
TypeScript
→ 코드 안에서 잘못된 값을 사용하지 않도록 방지

Zod
→ 코드 밖에서 들어오는 데이터가 올바른지 검사
```

### 3.3) Schema와 TS 타입의 중복을 줄이기 위해

기존에는 타입을 직접 작성했다

```ts
export type ReservationDTOStatus =
  | "RESERVED"
  | "RUNNING"
  | "COMPLETED"
  | "CANCELLED";
```

Zod를 적용 후에는 Schema를 기준으로 타입을 추론한다.

```ts
export const reservationStatusSchema = z.enum([
  "RESERVED",
  "RUNNING",
  "COMPLETED",
  "CANCELLED",
]);

export type ReservationDTOStatus = z.infer<typeof reservationStatusSchema>;
```

이제 상태가 추가될 경우 Schema만 수정하면 타입에도 자동으로 반영된다.

```
기존
- 타입 규칙과 검증 규칙을 각각 관리

수정 후
- Zod Schema를 기준으로 타입 자동 추론
```

#### 3.4) 기존 의존성을 실제로 활용하기 위해

프로젝트에는 이미 `zod`가 설치되어 있었지만 예약 API 응답 검증에는 사용되지 않고 있었다.

따라서 새로운 라이브러리를 무작정 추가한 것이 아니라, 이미 포함된 도구를 API 경계 검증에 적용했다.

## 4. 기존 코드

```ts
export async function getReservations(
  params?: ReservationParamsType
): Promise<ReservationItem[]> {
  const [washersResponse, dryersResponse] = await Promise.all([
    get<BaseResponseType<ReservationResponseType>>(
      reservationUrl.getReservations(),
      {
        params: {
          ...params,
          machineType: "WASHER",
        },
      }
    ),

    get<BaseResponseType<ReservationResponseType>>(
      reservationUrl.getReservations(),
      {
        params: {
          ...params,
          machineType: "DRYER",
        },
      }
    ),
  ]);

  const washers = mapReservations(washersResponse.data.reservations);

  const dryers = mapReservations(dryersResponse.data.reservations);

  return [...washers, ...dryers];
}
```

### 기존 코드의 의미

#### 1. 세탁기와 건조기 예약을 병렬 요청

```ts
Promise..all([
    WASHER 요청,
    DRYER 요청,
])
```

두 요청은 서로 의존하지 않으므로 동시에 실행했다.

#### 2. 응답을 `ReservationResponseType` 으로 간주

```ts
get<BaseResponseType<ReservationResponseType>>;
```

실제 검증은 없고, 응답이 해당 타입이라고 TS에게 알려줬다.

#### 3. 각각 Mapper 적용

```ts
mapReservations(washersResponse.data.reservations);
mapReservations(dryersResponse.data.reservations);
```

서버 DTO 배열을 UI 모델 배열로 변환했다.

#### 4. 두 배열 병합

```ts
return [...washers, ...dryers];
```

세탁기와 건조기 예약을 하나의 목록으로 반환했다.

## 5. 수정 코드

```ts
async function getReservationsByMachineType(
  machineType: ReservationMachineType,
  params?: ReservationParamsType
): Promise<ReservationItem[]> {
  const response = await get<BaseResponseType<unknown>>(
    reservationUrl.getReservations(),
    {
      params: {
        ...params,
        machineType,
      },
    }
  );

  const parsedData = reservationResponseSchema.parse(response.data);

  return mapReservations(parsedData.reservations);
}

export async function getReservations(
  params?: ReservationParamsType
): Promise<ReservationItem[]> {
  const [washers, dryers] = await Promise.all([
    getReservationsByMachineType("WASHER", params),
    getReservationsByMachineType("DRYER", params),
  ]);

  return [...washers, ...dryers];
}
```

## 6. 기존 코드와 수정 코드의 구체적인 차이

### 6.1) 응답 타입을 신뢰하는 방식 변경

#### 기존

```ts
get<BaseResponseType<ReservationResponseType>>;
```

response.data는 ReservationResponseType이라고 가정

#### 수정

```ts
get<BaseResponseType<unknown>>;
```

response.data는 아직 검증하지 않았기 때문에 알 수 없는 값

이후 직접 검증한다.

```ts
reservationResponseSchema.parse(response.data);
```

### 6.2) 실제 런타임 검증 단계 추가

#### 기존

서버 응답 -> 바로 Mapper

#### 수정

서버 응답 -> Zod 검증 -> Mapper

검증 내용

- reservations 가 배열인지
- 배열의 각 요소가 예약 DTO 구조인지
- id가 숫자인지
- useName이 문자열인지
- status가 정의된 enum 인지
- machineAvailability 가 정의된 enum 인지
- 필수 필드가 존재하는지
- nullable 필드만 `null`을 허용하는지
- 페이지 정보가 숫자인지

### 6.3) 타입 선언 방식 변경

#### 기존

```ts
export type ReservationDTO = {
  id: number;
  userId: number;
  // ...
};
```

#### 수정

```ts
export const reservationDTOSchema = z.object({
  id: z.number(),
  userId: z.number(),
  // ...
});

export type ReservationDTO = z.infer<typeof reservationDTOSchema>;
```

기존에는 타입이 기준이였다.

수정 후에는 Schema가 기준이고, TS 타입은 Schema에서 추론된다.

### 6.4) 요청 중복 제거

#### 기존

WASHER 요청과 DRYER 요청 코드가 각각 존재했다.

```ts
get(...machineType: "WASHER");
get(...machineType: "DRYER");
```

Zod 검증을 그대로 추가하면 다음 코드도 두 번 작성해야 했다.

```ts
reservationResponseSchema.parse(...);
```

#### 수정

공통 함수를 만들었다.

```ts
getReservationsByMachineType(machineType, params);
```

이 함수가 다음을 모두 담당한다.

- 기기 타입별 API 요청
  - 응답 검증
  - DTO 배열 추출
  - Mapper 적용
  - ReservationItem[] 반환

따라서 WASHER 와 DRYER는 값만 다르게 전달한다.

```ts
getReservationsByMachineType("WASHER", params);
getReservationsByMachineType("DRYER", params);
```

### 6.5 `washers` 변수의 의미 변경

#### 기존

```ts
const washersResponse = await get(...);
```

`washersResponse` 는 서버 전체 응답 객체였다.

```ts
BaseResponseType<ReservationResponseType>;
```

그 다음 Mapper를 적용해 별도 변수로 만들었다.

```ts
const washers = mapReservations(washersResponse.data.reservations);
```

#### 수정

```ts
const [washers, dryers] = await Promise.all([
    getReservationByMachineType(...),
])
```

`getReservationsByMachineType` 내부에서 이미 검증과 Mapper를 끝낸다.
따라서 수정 코드의 `washers` 는 처음부터 다음 타입이다.

```ts
ReservationItem[]
```

즉, Mapper 코드가 사라진 것이 아니라 공통 함수 안으로 이동했다.

```ts
return mapReservations[parseData.reservations];
```

## 7. Zod Schema 구성

### 예약 상태

```ts
export const reservationStatusSchema = z.enum([
  "RESERVED",
  "RUNNING",
  "COMPLETED",
  "CANCELLED",
]);
```

서버 예약 상태가 네 값 중 하나인지 검사한다.

### 기기 상태

```ts
export const machineAvailabilityStatusSchema = z.enum([
  "IN_USE",
  "RESERVED",
  "AVAILABLE",
  "UNAVAILABLE",
]);
```

기기의 사용 가능 상태가 정의된 값인지 검사한다.

### 예약 한 건

```ts
export const reservationDTOSchema = z.object({
  id: z.number(),
  userId: z.number(),
  userName: z.string(),
  // ...
});
```

기존 `ReservationDTO` 와 대응한다.

`ReservationDTO ↔ reservationDTOSchema`

- ReservationDTO
  - TS 타입
- reservationDTOSchema
  - 실제 값 검증 규칙

### 예약 목록 응답

```ts
export const reservationResponseSchema = z.object({
  reservations: z.array(reservationDTOSchema),
  totalCount: z.number(),
  totalPages: z.number(),
  currentPage: z.number(),
});
```

기존 `ReservationResponseType` 과 대응한다.

`ReservationResponseType ↔ reservationResponseSchema`

특히,

```ts
reservations: z.array(reservationDTOSchema);
```

는 TS의 다음 코드와 같은 구조를 나타낸다.

```ts
reservations: ReservationDTO[];
```

즉, 예약 배열 안에 있는 모든 요소가 `reservationDTOSchema`를 통과해야한다.

## 8. 수정 후 데이터 흐름

```
useGetReservations
        ↓
getReservations
        ↓
Promise.all
├─ getReservationsByMachineType("WASHER")
└─ getReservationsByMachineType("DRYER")
        ↓
API 응답을 BaseResponseType<unknown>으로 수신
        ↓
reservationResponseSchema.parse(response.data)
        ↓
검증된 ReservationDTO[]
        ↓
mapReservations
        ↓
ReservationItem[]
        ↓
WASHER·DRYER 배열 병합
        ↓
TanStack Query Cache
        ↓
UI
```

## 9. 문제

기존 예약 조회 API는 응답을 `ReservationResponseType`으로 지정하고 있었지만, 이는 TS의 정적 타입 정보일 뿐 실제 서버 응답을 검사하지 않았다.

따라서 서버에서 필드가 누락되거나 자료형이 달라지거나 정의되지 않은 enum 값이 전달돼도 API 진입 단계에서 이를 감지할 수 없었따.

또한 Mapper의 `"확인필요"` fallback이 존재해, 잘못된 API 상태값이 정상적인 운영 예외처럼 처리될 가능성이 있었다.

## 10. 설계 원칙

- 외부에서 들어오는 서버 응답은 검증 전까지 신뢰하지 않는다.
- API 응답은 내부 모델로 진입하기 전에 런타임 검증을 통과해야 한다.
- API 계약 오류와 업무상 확인이 필요한 상태를 구분한다.
- Schema를 단일 기준으로 두고 TS 타입은 Schema에서 추론한다.
- UI는 검증된 DTO를 Mapper로 변환한 모델만 사용한다.
- 기기 타입만 다른 반복 요청, 검증 흐름은 공통 함수로 관리한다.

# 11. 구조적 해결

## API 응답을 `unknown`으로 수신

```tsx
get<BaseResponseType<unknown>>;
```

서버 응답이 특정 타입이라고 미리 가정하지 않도록 변경했다.

## Zod Schema를 통한 런타임 검증

```tsx
reservationResponseSchema.parse(response.data);
```

예약 목록, 예약 DTO, 상태 enum, 페이지 정보를 실제 실행 시점에 검사했다.

## Schema 기반 타입 추론

```tsx
typeReservationDTO = z.infer<typeofreservationDTOSchema>;
```

Schema와 타입을 별도로 관리하지 않도록 변경했다.

## DTO와 UI 모델 경계 유지

검증된 DTO만 기존 Mapper로 전달했다.

```tsx
returnmapReservations(parsedData.reservations);
```

Zod가 데이터 계약을 검증하고, Mapper가 화면 표현을 담당하도록 책임을 분리했다.

## 기기 타입별 요청 공통화

```tsx
getReservationsByMachineType(machineType, params);
```

WASHER와 DRYER의 요청·검증·매핑 흐름을 하나의 함수로 통합했다.

---

# 12. 결과

- 필드 누락, 타입 불일치, 정의되지 않은 enum 값을 API 진입 단계에서 감지
- 검증되지 않은 외부 데이터가 Mapper와 UI로 전달되는 흐름 차단
- API 계약 오류와 관리자의 확인이 필요한 업무 상태를 분리
- Zod Schema와 TypeScript 타입의 중복 선언 제거
- 상태값 변경 시 Schema와 타입 간 불일치 가능성 감소
- WASHER와 DRYER 요청·검증·Mapper 코드의 중복 축소
- 기존 DTO–Mapper–UI Model 구조를 유지하면서 외부 데이터 경계를 강화

---

# 13. 이번 리팩토링의 한계

이번 작업은 성능 개선이 아니라 **데이터 안정성과 API 경계 개선**이다.

또한 Zod의 `parse`가 실패하면 예약 Query 전체가 오류 상태가 된다. 이후 운영 단계에서는 다음 개선도 고려할 수 있다.

- `safeParse`를 사용한 에러 변환
- API 응답 검증 실패 로그 수집
- Sentry 등 오류 추적 도구 연동
- 사용자에게 표시할 공통 에러 UI
- OpenAPI 기반 Schema 동기화
- API 계약 테스트 추가

---

# 14. 핵심 역량 문장

> TypeScript 타입만으로 신뢰하던 예약 API 응답을 `unknown`으로 수신하고 Zod Schema로 런타임 검증한 뒤, 검증된 DTO만 Mapper를 통해 UI 모델로 변환하도록 외부 데이터 진입 경계를 설계했습니다.

조금 더 구조 중심으로 적으면:

> 예약 API에 Zod 기반 런타임 검증 계층을 추가하고 Schema에서 TypeScript 타입을 추론하도록 구성해, API 계약 오류와 업무상 예외 상태를 분리하고 검증된 데이터만 내부 UI 모델로 전달하도록 데이터 흘므을 재설계 했습니다.
