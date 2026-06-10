# API 호출 구조와 서버 상태 관리 흐름

## 핵심 흐름

Washer의 API 호출 구조는 다음 흐름이다.

```
apiUrls → http wrapper → API function → query hook → component
```

각 계층의 역할을 분리해서 컴포넌트가 API 요청 세부 구현에 직접 의존하지 않도록 만든 구조이다.

---

## apiUrls

`apiUrls.ts`는 API endpoint 주소를 모아두는 파일이다.

```tsx
exportconstreportUrl= {
  get MalfunctionReports: () =>"/api/v2/admin/malfunction-reports",
}asconst;
```

API 함수에서 문자열 URL을 직접 작성하지 않고, `reportUrl.getMalfunctionReports()`처럼 가져다 쓰는 방식이다.

이렇게 하면 endpoint가 변경되었을 때 API 함수마다 찾아서 수정하지 않아도 되고, 도메인별 API 주소를 한 곳에서 확인할 수 있다.

---

## http wrapper

`http.ts`는 `axiosInstance`를 감싼 공통 요청 함수이다.

```tsx
export const get = async <T,>(...args: Parameters<typeofaxiosInstance.get>) =>
  awaitaxiosInstance.get<T, T>(...args);
```

API 함수에서 `axiosInstance.get()`을 직접 호출하지 않고, `get()` 함수를 통해 요청하도록 만든 구조이다.

이 wrapper는 HTTP method별 요청 방식을 통일하기 위한 역할을 한다.

```
get
post
patch
put
del
```

즉, API 함수는 axios 설정을 직접 알 필요 없이 공통 요청 함수만 사용하면 된다.

---

## API function

`getMalfunctionReports.ts`는 고장 신고 목록을 조회하는 API function이다.

```tsx
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

API function은 서버 요청만 담당하는 계층이다.

이 함수가 하는 일은 다음과 같다.

```
1. 어떤 URL로 요청할지 정한다.
2. 어떤 params를 보낼지 정한다.
3. 어떤 response type을 받을지 정한다.
4. 서버 요청을 실행한다.
```

여기서는 `reportUrl.getMalfunctionReports()`로 endpoint를 가져오고, `get()` wrapper를 통해 GET 요청을 보낸다.

React Query의 `queryKey`, `loading`, `error`, `cache` 같은 상태는 여기서 다루지 않는다.

---

## query hook

`useGetMalfunctionReports.ts`는 React Query와 API function을 연결하는 hook이다.

```tsx
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

query hook은 서버 상태 관리를 담당하는 계층이다.

이 hook이 하는 일은 다음과 같다.

```
1. queryKey를 만든다.
2. queryFn에 API function을 연결한다.
3. initialData를 설정할 수 있게 한다.
4. React Query가 loading, error, data, refetch 상태를 관리하게 한다.
```

여기서 `queryFn`은 실제 데이터를 가져오는 함수이고, 내부에서 `fetchMalfunctionReports(params)`를 호출한다.

즉, API 요청 자체는 `getMalfunctionReports`가 담당하고, 요청 결과를 React Query로 관리하는 일은 `useGetMalfunctionReports`가 담당한다.

---

## params 흐름

`params`는 API 요청과 queryKey에 모두 사용된다.

```tsx
const queryKey = reportQueryKeys.getMalfunctionReports(params || {});
```

```tsx
queryFn: () => fetchMalfunctionReports(params);
```

즉, params는 다음 두 가지 역할을 한다.

```
1. 서버에 필터 조건으로 전달된다.
2. React Query 캐시를 구분하는 기준이 된다.
```

예를 들어 status 필터가 다르면 서로 다른 queryKey가 만들어진다.

```tsx
getMalfunctionReports({ status: "PENDING" });
getMalfunctionReports({ status: "RESOLVED" });
```

이렇게 하면 서로 다른 조건의 데이터가 같은 캐시에 섞이지 않는다.

---

## component

컴포넌트는 query hook을 통해 서버 데이터를 가져온다.

```tsx
const { data, isLoading, isError } = useGetMalfunctionReports(params);
```

컴포넌트는 axios나 endpoint를 직접 알 필요가 없다.

컴포넌트가 알면 되는 것은 다음 정도이다.

```
- 어떤 hook을 호출할지
- 어떤 params를 넘길지
- data/loading/error 상태를 어떻게 화면에 보여줄지
```

---

## 전체 구조 정리

```
apiUrls
- endpoint 주소를 관리한다.

http wrapper
- axiosInstance 기반 공통 요청 함수를 제공한다.

API function
- URL, params, response type을 기준으로 서버 요청만 담당한다.

query hook
- queryKey, queryFn, loading/error, caching을 담당한다.

component
- query hook을 통해 서버 상태를 가져와 화면에 렌더링한다.
```

---

## 이 구조의 핵심

이 구조의 핵심은 **서버 요청 책임과 화면 책임을 분리하는 것**이다.

컴포넌트에서 직접 axios를 호출하면 화면 코드가 API 세부 구현에 의존하게 된다.

반면 query hook을 기준으로 데이터를 가져오면 컴포넌트는 화면 구성에 집중할 수 있다.

최종 흐름은 다음과 같다.

```
component
  → query hook
  → API function
  → http wrapper
  → apiUrls
```

이 구조는 API 요청 흐름을 예측 가능하게 만들고, 도메인별 서버 상태 관리를 정리하기 위한 구조이다.
