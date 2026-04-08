# 왜 interceptor에서 response.data를 반환하면 타입이 꼬일까

## 핵심 결론

`axios.get<T>()`의 제네릭 `<T>`는 **`response.data`의 타입**이다.

하지만 axios는 기본적으로 **`AxiosResponse<T>` 전체를 반환한다.**

문제는 interceptor에서 이렇게 바꿨을 때 발생한다:

```tsx
axiosInstance.interceptors.response.use((response) => {
  returnresponse.data;
});
```

이 순간부터:

- **런타임 반환값** → `T`
- **TypeScript가 기대하는 값** → `AxiosResponse<T>`

이 둘이 어긋나면서 타입이 꼬인다.

---

## 1. axios 기본 동작

```tsx
constresponse = awaitaxios.get<GetUsersResponse>("/v2/admin/users");
```

TypeScript는 이렇게 이해한다:

```tsx
response: AxiosResponse<GetUsersResponse>;
```

따라서 접근은 이렇게 된다:

```tsx
response.data; // GetUsersResponse
response.data.data.users;
```

---

## 2. interceptor에서 구조를 바꿔버림

```tsx
axiosInstance.interceptors.response.use((response) => {
  returnresponse.data;
});
```

이건 단순한 가공이 아니라 **반환 구조 자체를 변경하는 것**이다.

### 원래 응답

```tsx
{
data: {
status:"OK",
data: {users: [...] }
  },
status:200
}
```

### interceptor 이후

```tsx
{
status:"OK",
data: {users: [...] }
}
```

즉 이제:

```tsx
response === GetUsersResponse;
```

---

## 3. 충돌 지점

```tsx
constresponse = await axiosInstance.get<GetUsersResponse>("/v2/admin/users");
```

### 실제 런타임

```tsx
response: GetUsersResponse;
response.data.users;
```

### TypeScript의 기대

```tsx
response: AxiosResponse<GetUsersResponse>;
response.data.data.users;
```

**타입 시스템 vs 실제 값이 다름**

---

## 4. 왜 `<T>`로 해결이 안 되는가

```tsx
axiosInstance.get<T>();
```

axios 타입 정의는 기본적으로 이렇게 되어 있다:

```tsx
get<T>() =>Promise<AxiosResponse<T>>
```

하지만 interceptor 이후 실제 동작은:

```tsx
get<T>() =>Promise<T>
```

문제는:

> TypeScript는 interceptor 내부 구현까지 분석해서 반환 타입을 자동으로 바꿔주지 않는다.

---

## 5. 정리 (중요)

| 구분                             | 반환 타입          |
| -------------------------------- | ------------------ |
| interceptor 없음                 | `AxiosResponse<T>` |
| `response.data` 반환 interceptor | `T`                |

이 차이 때문에 타입이 꼬인다.

---

## 6. 해결 방법

### 1) 타입 단언 (가장 단순)

```tsx
constresponse= (await axiosInstance.get("/v2/admin/users"))asGetUsersResponse;
```

- 빠름
- 하지만 타입 안전성 낮음

---

### 2) 래퍼 함수

```tsx
exportasyncfunctiongetData<T>(url:string):Promise<T> {
returnaxiosInstance.get(url)asPromise<T>;
}
```

사용:

```tsx
constresponse = awaitgetData<GetUsersResponse>("/v2/admin/users");
```

- 반복 제거
- 구조 정리됨

---

### 3) interceptor 제거 (가장 타입 친화적)

```tsx
axiosInstance.interceptors.response.use((response) => response);
```

사용:

```
response.data.data.users
```

- 타입 안정성 최고
- 대신 접근이 번거로움

---

## 7. 실무 기준 추천

너처럼:

- interceptor에서 `data` 벗기고
- 사용처 간단하게 쓰고 싶으면
- **“래퍼 함수 방식”이 가장 균형 좋다**

---

## 한 줄 요약

> interceptor에서 `response.data`를 반환하면
>
> axios의 기본 타입(`AxiosResponse<T>`)과 실제 반환값(`T`)이 달라져서 타입 추론이 깨진다.
