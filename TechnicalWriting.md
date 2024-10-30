# TanStack Query 시작하기

# 도입부

API 요청으로 받아오는 서버 상태를 관리하는 데 어려움을 겪지 않으셨나요? 아래의 예시 코드처럼 해당 상태를 관리하기 위하여 data 상태뿐만 아니라 loading 상태와 error 상태를 따로 두어 관리하는 탓에 장황해진 코드를 보며 불편함을 느끼지는 않으셨나요? 또한 해당 코드가 상태별로 필요함에 따라 같은 형태의 코드가 반복되는 것을 보고서 불편함을 느끼지는 않으셨나요? 이를 해결하기 위하여 TanStack Query를 배우려고 했을 겁니다.

```tsx
import React, { useState, useEffect } from 'react';

function UserComponent({ userId }) {
  const [user, setUser] = useState(null);
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState(null);

  useEffect(() => {
    const fetchUser = async () => {
      setIsLoading(true);
      setError(null);
      try {
        const response = await fetch(`https://api.example.com/users/${userId}`);
        if (!response.ok) {
          throw new Error('Failed to fetch user');
        }
        const data = await response.json();
        setUser(data);
      } catch (err) {
        setError(err.message);
      } finally {
        setIsLoading(false);
      }
    };

    fetchUser();
  }, [userId]);

  if (isLoading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;
  if (!user) return null;

  return <div>{user.name}</div>;
```

그러나 초보자의 경우 코드를 읽는 것도 힘든데 해당 문서가 영어로 되어있는 탓에 심리적 장벽이 높아 학습하는데 거부감이 들었을 겁니다. 또한, 문서를 봐도 와닿지 않는 용어들이 많고 훅이나 상태들이 너무 많아 무엇이 중요한지에 대해 구분해야 하는 점에서 어려움을 겪었을 겁니다. 실제 본인 또한 학습하는데 해당 어려움을 겪으며 긴 시간이 걸렸었고 그때마다 중요한 부분들에 대해서 한글로 작성된 친절한 문서가 있다면 얼마나 좋을까? 라는 생각을 하곤 했습니다. 이러한 생각을 바탕으로 TanStack Query를 처음 써보기 위하여 학습하는 개발자들에게 도움을 주기 위하여 문서를 작성했습니다.

# **TanStack Query란?**

---

## 소개

> `TanStack Query (FKA React Query)` is often described as the missing data-fetching library for web applications, but in more technical terms, it makes **`fetching, caching, synchronizing and updating server state`** in your web applications a breeze.

공식 문서에서 TanStack Query에 대해서 서버 상태를 보다 쉽고 효율적으로 가져오고, 캐싱하며, 동기화 및 업데이트한다고 설명합니다.

> While most traditional state management libraries are great for working with client state, they are **`not so great at working with async or server state**.`

반면에, TanStack Query 외 대다수 상태 관리 라이브러리는 클라이언트 상태 관리에는 효과적이지만, 비동기 혹은 서버 상태 관리의 경우는 그다지 효과적이지 않다고 설명하며 이러한 이유로 TanStack Query가 등장하게 된 배경을 설명하고 있습니다.

위 설명을 살펴보면 `서버 상태`라는 용어가 반복적으로 등장하는데 해당 용어가 생소할 수 있습니다. 그렇기에 본 TanStack Query에 대한 설명에 앞서 서버 상태에 대해서 알아보겠습니다.

## **서버 상태란?**

> Is persisted remotely in a location you may not control or own
> Requires asynchronous APIs for fetching and updating
> Implies shared ownership and can be changed by other people without your knowledge
> Can potentially become "out of date" in your applications if you're not careful

공식 문서에서 서버 상태란 다음 4가지 특징을 가지는 상태로 설명합니다.

- 직접 관리하거나 소유하지 않은 원격 저장소에 보관됨.
- fetching 과 updating을 위해서는 비동기 API 요청이 필요.
- 여러 사용자가 공유하는 데이터로, 내가 모르는 사이에 다른 사람에 의해 변경 가능.
- 주의하지 않으면 앱에서 사용 중인 데이터가 최신 상태를 유지하지 않을 수 있음.

해당 설명에 대해서 이해를 돕기 위하여 구체적인 예시와 클라이언트 상태와의 비교를 통해서 설명하겠습니다.

### 클라이언트 상태

: 클라이언트 측(브라우저)에서 로컬로 관리하며 해당 클라이언트에만 존재하는 상태를 의미합니다.

예시: UI 상태(모달 열림/닫힘), 폼 입력값, 페이지 스크롤 위치 등

```tsx
const Modal = (isOpen: boolean) => {
  return isOpen ? <div>모달</div> : null;
};

const Page = () => {
  const [isOpen, setIsOpen] = useState(false);

  const handleToggleModal = () => {
    setIsOpen((prev) => !prev);
  };

  return (
    <>
      <button onClick={handleToggleModal}>버튼</button>
      <Modal isOpen={isOpen} />
    </>
  );
};
```

### 서버 상태

: 서버(데이터베이스)에 저장되고 관리되는 데이터로, 클라이언트가 필요할 때 요청하여 받아오는 데이터를 의미합니다.

예시: API에서 가져온 사용자 프로필, 상품 목록, 뉴스 피드 등

```tsx
export interface Product {
  id: number;
  name: string;
  price: number;
  category: string;
}

async function fetchProducts(): Promise<Product[]> {
  const response = await fetch("http://simorimi.com/products");
  return await response.json();
}
```

둘의 차이 중 핵심은 `관리 주체`입니다. 해당 상태에 대해서 관리 주체가 서버라면 서버 상태가, 클라이언트라면 클라이언트 상태가 됩니다.

## **서버 상태 관리 시 발생하는 어려움들**

> Caching... (possibly the hardest thing to do in programming)
> Deduping multiple requests for the same data into a single request
> Updating "out of date" data in the background
> Knowing when data is "out of date"
> Reflecting updates to data as quickly as possible
> Performance optimizations like pagination and lazy loading data
> Managing memory and garbage collection of server state
> Memoizing query results with structural sharing

공식 문서에서 서버 상태 관리 시 어려움으로 다음과 같이 언급합니다.

- 캐싱
  : 여기서 캐싱이란 파일 복사본을 캐시 또는 임시 저장 위치에 저장하여 보다 빠르게 액세스할 수 있도록 하는 프로세스를 의미합니다.
- 같은 데이터에 대한 중복 요청을 단일 요청 통합
- 백그라운드에서 오래된 데이터 업데이트
- 데이터가 얼마나 오래되었는지를 파악
- 데이터 업데이트를 가능한 빠르게 반영
- 페이지 네이션 및 데이터 지연 로드와 같은 성능을 최적화
- 서버 상태의 메모리 및 가비지 수집 관리
- 구조 공유를 사용하여 쿼리 결과를 메모화
  : 여기서 구조 공유란 데이터 구조의 변경되지 않은 부분은 그대로 유지하며 변경된 부분만 새로 생성하는 것을 말합니다.

또 다른 어려움으로 서버 상태를 관리하다 보면 보일러플레이트 코드를 발생시킵니다. 여기서 보일러플레이트 코드란 여러 가지 상황에서 거의 또는 전혀 변경하지 않고 재사용할 수 있는 컴퓨터 언어 텍스트를 의미합니다. 아래의 예시와 같이 서버 상태를 관리하기 위해서 데이터, 로딩 그리고 에러를 관리하기 위하여 useState와 useEffect가 반복해서 등장하게 됩니다.

```tsx
import React, { useState, useEffect } from "react";

function UserComponent({ userId }) {
  const [user, setUser] = useState(null);
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState(null);

  useEffect(() => {
    const fetchUser = async () => {
      setIsLoading(true);
      setError(null);
      try {
        const response = await fetch(`https://api.example.com/users/${userId}`);
        if (!response.ok) {
          throw new Error("Failed to fetch user");
        }
        const data = await response.json();
        setUser(data);
      } catch (err) {
        setError(err.message);
      } finally {
        setIsLoading(false);
      }
    };

    fetchUser();
  }, [userId]);

  if (isLoading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;
  if (!user) return null;

  return <div>{user.name}</div>;
}
```

이처럼 하나의 상태를 관리하는 길고 복잡한 보일러플레이트 코드가 각각의 서버 상태마다 존재하게 됩니다.

## **서버 상태 관리 시 발생하는 어려움들을 해결하는 TanStack query**

서버 관리 시 발생하는 어려움을 효율적으로 해결하기 위하여 등장한 것이 TanStack Query 입니다. TanStack Query는 앞서 살펴본 어려움들을 모두 기능적으로 지원합니다.

- 캐싱
  : TanStack Query는 쿼리 키를 기반으로 메모리 내 캐싱을 수행합니다. 각 쿼리 결과는 고유한 쿼리 키와 연결되어 저장됩니다.
- 같은 데이터에 대한 중복 요청을 단일 요청 통합
  : 동일한 쿼리 키로 여러 요청이 동시에 발생하면, TanStack Query는 이를 단일 네트워크 요청으로 통합합니다.
- 백그라운드에서 오래된 데이터 업데이트
  : `refetchInterval` 옵션을 사용하여 주기적으로 백그라운드에서 데이터를 자동으로 업데이트할 수 있습니다.
- 데이터가 얼마나 오래되었는지를 파악
  : `staleTime` 옵션을 통해 데이터가 "오래된(stale)" 상태가 되는 시점을 설정할 수 있습니다. 이를 통해 데이터가 얼마나 오래되었는지를 파악합니다.
- 데이터 업데이트를 가능한 빠르게 반영
  : React의 상태 관리와 통합되어 있어, 데이터가 업데이트되면 자동으로 컴포넌트가 리렌더링 됩니다.
- 페이지네이션 및 데이터 지연 로드와 같은 성능을 최적화
  - 페이지네이션: `useInfiniteQuery` 훅을 제공하여 무한 스크롤이나 "더 보기" 기능을 쉽게 구현할 수 있습니다.
  - 데이터 지연 로드: `enabled` 옵션을 사용하여 조건부로 쿼리를 실행할 수 있습니다.
- 서버 상태의 메모리 및 가비지 수집 관리
  : `gcTime` 옵션을 통해 비활성 쿼리가 가비지 수집 관리의 대상이 되어 메모리를 관리합니다.
- 구조 공유를 사용하여 쿼리 결과를 메모화
  : 내부적으로 구조 공유(structural sharing)를 사용하여 불필요한 리렌더링을 방지하고 메모리 사용을 최적화합니다.

> Help you remove many lines of complicated and misunderstood code from your application and replace with just a handful of lines of React Query logic.

또한, TanStack Query로 서버 상태를 관리하게 되면 서버 상태 관리할 때 복잡해질 수 있는 코드들을 단 몇 줄로 줄일 수 있습니다. 아래에 예시는 어려움에서 소개했던 예시 코드를 TanStack Query로 마이그레이션 한 코드입니다. 두 코드를 비교하면 반복적인 useState와 useEffect가 사라졌으며 길고 복잡했던 코드가 단 몇 줄로 줄었음을 알 수 있습니다.

```tsx
import React from 'react';
import { useQuery } from 'react-query';

const fetchUser = async (userId) => {
  const response = await fetch(`https://api.example.com/users/${userId}`);
  if (!response.ok) {
    throw new Error('Failed to fetch user');
  }
  return response.json();
};

function UserComponent({ userId }) {
  const { data: user, isLoading, error } = useQuery(
    ['user', userId],
    () => fetchUser(userId),
  );

  if (isLoading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;
  if (!user) return null;

  return <div>{user.name}</div>;
```

> Powerful asynchronous state management for TS/JS, React, Solid, Vue, Svelte and Angular

이러한 특징으로 인하여 TanStack Query가 서버 상태를 관리하는데 강력한 도구임을 알 수 있습니다.

# 기본 설정

---

## **QueryClient**

> The QueryClient can be used to interact with a cache

QueryClient를 사용하면 캐시와 상호작용을 할 수 있습니다. 또한 defaultOptions를 이용하면 모든 Query 또는 Mutation에 기본 옵션을 추가할 수 있습니다. 대표적인 예로는 staleTime 혹은 gcTime 설정이 있습니다. 물론 defaultOptions가 없어도 괜찮습니다.

QueryClient는 다양한 메서드를 제공합니다. 이 중 대표적인 것으로 쿼리를 무효화하고 최신화를 진행하는 queryClient.invalidateQueries 메서드가 있습니다. 이에 대해서는 핵심 개념 설명에서 자세히 다루겠습니다.

이외 다른 메서드들은 [공식 문서](https://tanstack.com/query/v5/docs/reference/QueryClient#queryclient) 를 참고하면 됩니다.

```tsx
import { QueryClient } from "@tanstack/react-query";

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: Infinity,
    },
  },
});
// const queryClient = new QueryClient();

queryClient.invalidateQueries({ queryKey: ["todos"] });
```

## Query Client Provider

> Use the QueryClientProvider component to connect and provide a QueryClient to your application

TanStack Query를 사용하기 위해서 `QueryClientProvider`를 애플리케이션 최상단에 감싸주고 `QueryClient` 인스턴스를 client props로 넣어주면 됩니다. 이를 통해 QueryClient를 애플리케이션에 연결하고 제공합니다.

```tsx
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";

const queryClient = new QueryClient();

function App() {
  return <QueryClientProvider client={queryClient}>...</QueryClientProvider>;
}
```

# 캐싱 라이프 사이클

---

## staleTime 과 gcTime

본격적인 캐싱 라이플 사이클을 이해하기에 앞서 필요한 개념인 staleTime과 gcTime에 대해서 설명하겠습니다.

### staleTime

> Stale queries are refetched automatically in the background when:
>
> - New instances of the query mount
> - The window is refocused
> - The network is reconnected
> - The query is optionally configured with a refetch interval

`staleTime`에서 stale이라는 단어는 fresh의 반대말로 썩은, 오래된 이라는 의미로 staleTime은 데이터가 fresh에서 stale 상태로 변경되는데 걸리는 시간을 의미합니다. fresh 상태일 때는 쿼리 인스턴스가 새롭게 mount 즉, 스크린에서 사용되어도 네트워크 요청(fetch)이 일어나지 않는 데 반해 stale인 경우는 새롭게 mount 될 때 네트워크 요청이 일어납니다. 기본값은 0입니다.

### gcTime

> Query results that have no more active instances of useQuery, useInfiniteQuery or query observers are labeled as "inactive" and remain in the cache in case they are used again at a later time.
>
> - By default, "inactive" queries are garbage collected after **5 minutes**.

gcTime은 Garbage Collection Time의 줄임말로 쿼리 인스턴스가 unmount 되면 즉, 스크린에서 더이상 사용되지 않으면 데이터는 inactive 상태로 변경됩니다. inactive 상태가 된 캐시는 `gcTime`만큼 유지되다 가비지컬렉터의 수집 대상이 됩니다. 기본값은 5분입니다.

## 캐싱 라이프 사이클

> At its core, TanStack Query manages query caching for you based on query keys.

TanStack Query는 쿼리 키를 바탕으로 쿼리 캐싱을 진행합니다.

> This caching example illustrates the story and lifecycle of:
>
> - Query Instances with and without cache data
> - Background Refetching
> - Inactive Queries
> - Garbage Collection

이제 캐싱 라이프 사이클의 이해를 돕기위하여 예를 들어 설명하겠습니다. staleTime(0)과 gcTime(5분)은 기본값입니다.

> A new instance of useQuery({ queryKey: ['todos'], queryFn: fetchTodos }) mounts.

[’todos’] 쿼리 키를 최초로 사용합니다. 이때, fetchTodos 쿼리 함수로 데이터를 요청하여 가져옵니다. 가져온 데이터는 해당 쿼리 키 아래에 캐싱됩니다. staleTime이 0이기에 이 데이터는 즉시 stale 상태가 됩니다.

> A second instance of useQuery({ queryKey: ['todos'], queryFn: fetchTodos }) mounts elsewhere.

['todos'] 쿼리 키는 이미 사용했기에 해당 쿼리 키 아래에 데이터를 즉시 가져옵니다. 그러나 해당 데이터는 stale 상태이기에 fetchTodos 쿼리 함수로 새롭게 데이터를 요청하여 가져옵니다. 가져온 데이터로 해당 쿼리 키 아래 캐싱된 데이터를 업데이트 합니다. 이로 인하여 해당 쿼리 키를 사용하는 모든 인스턴스는 새로운 데이터로 업데이트 됩니다.

> Both instances of the useQuery({ queryKey: ['todos'], queryFn: fetchTodos }) query are unmounted and no longer in use.

['todos'] 쿼리 키를 더 이상 사용하지 않으면 inactive 상태가 됩니다. inactive 상태로 설정한 gcTime이 지나게 되면 해당 쿼리 키 아래에 데이터는 삭제되고 가비지 컬렉터 대상이 됩니다.

> Before the cache timeout has completed, another instance of useQuery({ queryKey: ['todos'], queryFn: fetchTodos }) mounts. The query immediately returns the available cached data while the fetchTodos function is being run in the background. When it completes successfully, it will populate the cache with fresh data.

gcTime이 지나기 전에 해당 쿼리 키를 사용하는 인스턴스가 다시 mount 된다면 fetchTodos 쿼리 함수를 통해 새롭게 데이터를 요청하고 가져옵니다. 가져온 데이터로 쿼리 키 아래 캐싱된 데이터를 업데이트 합니다. 해당 경우 다시 active 상태가 되어 gcTime은 영향을 미치지 않습니다.

# **핵심 개념 3가지**

---

## **Queries(useQuery)**

> A query is a declarative dependency on an asynchronous source of data that is tied to a **`unique key`**. A query can be used with any Promise based method (including GET and POST methods) to fetch data from a server. If your method modifies data on the server, we recommend using [**Mutations**](https://tanstack.com/query/latest/docs/framework/react/guides/mutations) instead.

쿼리는 고유한 키와 연결된 비동기 데이터 소스에 대한 선언적 의존성입니다. 이 말을 쉽게 풀어서 얘기하면 `queryKey`를 기반으로 `쿼리 캐싱`을 관리한다는 것을 의미합니다. 또한 쿼리는 서버로부터 데이터를 가져오기 위해 Promise 기반의 모든 메서드(GET과 POST 메서드 포함)와 함께 사용될 수 있습니다. 만약 서버의 데이터를 수정하고자 한다면, 쿼리 대신 뮤테이션 사용을 권합니다.

### 옵션

queries에 해당하는 useQuery 훅을 사용하기 위해서는 queryKey와 queryFn이 필수입니다.

```tsx
const result = useQuery({
  queryKey: ["todos", 5, { preview: true }],
  queryFn: fetchTodos,
});
```

queryKey

: queryKey는 `배열`로 지정해 줘야 하며 이는 단일 문자열만 포함된 배열이 될 수도 있고, 여러 문자열과 중첩된 객체로 구성된 복잡한 형태일 수도 있습니다.

queryFn

: queryFn는 데이터를 성공적으로 가져오면 데이터를 resolved 하는, 실패하면 에러를 throw 하는 `Promise`를 반환하는 함수여야 합니다.

이 외에는 선택입니다. 선택 옵션으로는 gcTime, staleTime 등 수많은 옵션이 존재합니다. 다른 옵션들은 [공식 문서](https://tanstack.com/query/latest/docs/framework/react/reference/useQuery)를 참고하시길 바랍니다.

```tsx
const result = useQuery({
  queryKey: ["todos"],
  queryFn: fetchTodos,
  staleTime: 0,
  gcTime: 5 * 60 * 1000,
  // ... options
});
```

### 주요 반환 값

`data`

> Defaults to undefined.
>
> The last successfully resolved data for the query.

기본값은 undefined이며 가장 최근에 쿼리 요청 성공했을 때의 resolved된 data를 가집니다.

`error`

> Defaults to null
>
> The error object for the query, if an error was thrown.

기본값은 null이며 쿼리 요청이 실패했을 때 throw된 error 객체를 가집니다.

`status`

> The result object contains a few very important states you'll need to be aware of to be productive. A query can only be in one of the following states at any given moment:
>
> - isPending or status === 'pending' - The query has no data yet
> - isError or status === 'error' - The query encountered an error
> - isSuccess or status === 'success' - The query was successful and data is available

쿼리는 status 값으로 다음 세 가지 상태 중 하나만 가질 수 있습니다.

- status === ‘pending’(isPending)

: 캐싱 된 데이터가 없고, 쿼리 시도가 아직 완료되지 않은 상태를 의미합니다.

- status === 'error'(isError)

: 쿼리 요청 중 에러가 발생한 상태를 의미합니다.

- status === 'success'(isSuccess)

: 쿼리 요청이 성공했고 데이터를 이용할 수 있는 상태를 의미합니다.

```tsx
function Todos() {
  const { isPending, isError, data, error, ...} = useQuery({
    queryKey: ['todos'],
    queryFn: fetchTodos,
  })

  if (isPending) {
    return <span>Loading...</span>
  }

  if (isError) {
    return <span>Error: {error.message}</span>
  }

  return (
    <ul>
      {data.map((todo) => (
        <li key={todo.id}>{todo.title}</li>
      ))}
    </ul>
  )
}
```

이 외에도 다양한 반환 값들이 있습니다. 그중 자주 사용하는 몇 가지에 대해서 소개하겠습니다.

`fetchStatus`

> fetchStatus === 'fetching' - The query is currently fetching.
>
> fetchStatus === 'paused' - The query wanted to fetch, but it is paused. Read more about this in the [**Network Mode**](https://tanstack.com/query/latest/docs/framework/react/guides/network-mode) guide.
>
> fetchStatus === 'idle' - The query is not doing anything at the moment.

- fetchStatus === 'fetching'(isFetching)

: fetching 중인 상태를 의미합니다. 캐싱 된 데이터가 있더라도 fetching 여부에 따라 true/false를 반환합니다.

- fetchStatus === 'paused'(isPaused)

: 쿼리는 fetch를 원하나 멈춘 상태를 의미합니다. 해당 값은 네트워크 연결이 끊어졌을 때 예외 처리 시 유용합니다.

- fetchStatus === 'idle'(isIdle)

: 쿼리가 fetch를 하지 않는 상태를 의미합니다.

status가 존재하는데 왜 fetchStatus가 필요할까 하는 의문이 들 수 있습니다. 그 이유는 같은 status 값이라도 다른 fetchStatus 조합이 나올 수 있기 때문입니다.

> a query in success status will usually be in idle fetchStatus, but it could also be in fetching if a background refetch is happening.
>
> a query that mounts and has no data will usually be in pending status and fetching fetchStatus, but it could also be paused if there is no network connection.

예를 들어, status가 success인 경우 일반적으로는 fetchStatus가 idle 값을 가지겠지만, 이미 데이터를 가진 상태에서 refetch를 하는 경우에는 fetchStatus가 fetching 값을 가질 수 있습니다. 또한, status가 pending인 경우 fetching을 진행하여 fetchStatus 값이 fetching일 수 있지만, 네트워크가 끊겼다면 fetchStatus는 paused 값을 가질 수 있습니다. 이러한 이유로 status와 fetchStatus 각각을 따로 두어 관리하는 것입니다.

이 외 isPending과 비슷한 isLoading도 있습니다.

- isLoading(isFetching && isPending)

: 캐싱 된 데이터가 없을 때, 쿼리 함수가 실행되면 로딩 여부에 따라 true/false를 반환합니다.

isLoading과 isPending, isFetching과 isPending이 헷갈릴 수 있습니다. 해당 의미의 명확한 구분을 위하여 각 값을 비교해 보겠습니다.

- isLoading vs isPending
  - isLoading은 캐싱 된 데이터가 없을 때, fetching의 경우에만 true 값을 가질 수 있습니다.
  - isPending은 캐싱 된 데이터가 없을 때는 fetching 전이나 중일 때 언제든 true 값을 가질 수 있습니다.
- isFetching vs isPending
  - isFetching은 캐싱 된 데이터가 있어도 fetching 여부에 따라 true 값을 가질 수 있습니다.
  - isPending은 캐싱 된 데이터가 있으면 항상 false 값만 가집니다.

이 외 다양한 반환 값들이 있습니다. 다른 반환 값들은 [공식 문서](https://tanstack.com/query/latest/docs/framework/react/reference/useQuery)를 참고하시길 바랍니다.

## **Mutations(useMutation)**

> Unlike queries, mutations are typically used to create/update/delete data or perform server side-effects. For this purpose, TanStack Query exports a useMutation hook.

mutations의 경우 queries와 달리 데이터를 create/update/delete 하거나 서버에 사이드 이펙트를 일으키는 경우 사용합니다. HTTP 메서드로는 post, patch, delete 요청 시 주로 mutations를 사용합니다.

### 의문

여기까지 학습하면 왜 GET의 경우에는 queries를 써야 하고 이 외의 경우는 mutations를 사용해야하는지 궁금할 것입니다. 해당 의문의 답은 `캐싱의 차이`입니다. queries의 경우 데이터를 받아오면 옵션에서 설정한 쿼리 키를 바탕으로 해당 쿼리 키 아래에 캐싱을 진행합니다. 반면, mutations의 경우 옵션으로 쿼리 키를 가지지 않습니다. 즉, 쿼리 키를 바탕으로 캐싱을 진행하지 않습니다. 이렇기에 데이터를 가져오는 작업은 queries를 서버에 사이드 이펙트를 일으키는 작업에는 mutations를 사용합니다. mutations의 경우 뮤테이션 키를 가지지만 해당 키는 캐싱을 위한 키가 아닙니다. 해당 키는 mutation의 defaults를 설정하기 위하여 사용되는 값이기에 캐싱과는 무관합니다.

### 옵션

옵션의 경우 promise를 반환하는 함수인 mutationFn이 필수입니다.

```tsx
const mutation = useMutation({
  mutationFn: createTodo,
  onMutate() {
    /* ... */
  },
  onSuccess(data) {
    console.log(data);
  },
  onError(err) {
    console.log(err);
  },
  onSettled() {
    /* ... */
  },
  // ... options
});
```

이 외에는 선택입니다. 그중 자주 사용하는 것 몇 가지를 소개하겠습니다.

- onMutate

: mutationFn이 실행되기 전에 실행되는 함수로 mutation 함수가 받을 동일한 변수를 받습니다. 이는 주로 낙관적 업데이트를 구현할 때 많이 사용됩니다.

- onSuccess

: mutationFn이 성공했을 때 실행되는 함수로 mutation의 결과를 받습니다. mutation 결과의 한 예로 post의 경우는 새로 생성된 데이터를 의미합니다.

- onError

: mutationFn이 실패했을 때 실행되는 함수로 error 객체를 받습니다.

- onSettled

: try…catch…finally 구문의 finally처럼 요청의 성공 여부와 관계없이 실행되는 함수를 의미합니다.

이 외 다양한 옵션이 있습니다. 다른 옵션들은 [공식 문서](https://tanstack.com/query/latest/docs/framework/react/reference/useMutation)를 참고하시길 바랍니다.

### 주요 반환 값

`data`

> Defaults to undefined.
>
> The last successfully resolved data for the mutation.

기본값은 undefined이며 가장 최근에 뮤테이션 요청 성공했을 때의 resolved된 data를 가집니다.

`error`

> Defaults to null
>
> The error object for the query, if an error was encountered.

기본값은 null이며 뮤테이션 요청이 실패했을 때 error 객체를 가집니다.

`status`

> A mutation can only be in one of the following states at any given moment:
>
> - isIdle or status === 'idle' - The mutation is currently idle or in a fresh/reset state
> - isPending or status === 'pending' - The mutation is currently running
> - isError or status === 'error' - The mutation encountered an error
> - isSuccess or status === 'success' - The mutation was successful and mutation data is available

뮤테이션의 경우 fetchStatus는 따로 없고 status만 가집니다.

- status === 'idle'(isIdle)

: mutation이 아직 실행되지 않았거나, 이전 실행 후 리셋된 상태를 의미합니다.

- status === 'pending'(isPending)

: mutation이 진행 중인 상태를 의미합니다.

- status === 'error'(isError)

: mutation 중 에러가 발생한 상태를 의미합니다.

- status === 'success'(isSuccess)

: mutation이 성공하고 데이터가 이용할 수 있는 상태를 의미합니다.

```tsx
const {data, error, isIdle, isPending, isError, isSuccess ...} = useMutation({
  mutationFn: createTodo,
});

  if (isPending) {
    return <span>Loading...</span>
  }

  if (isError) {
    return <span>Error: {error.message}</span>
  }

  return (
    <ul>
      {data.map((todo) => (
        <li key={todo.id}>{todo.title}</li>
      ))}
    </ul>
  )
```

`isPaused`

> will be true if the mutation has been paused
> see [**Network Mode**](https://tanstack.com/query/latest/docs/framework/react/guides/network-mode) for more information.

: 뮤테이션을 원하나 멈춘 상태를 의미합니다. 해당 값은 네트워크 연결이 끊어졌을 때 예외 처리 시 유용합니다.

이 외 다양한 반환 값들이 있습니다. 다른 반환 값들은 [공식 문서](https://tanstack.com/query/latest/docs/framework/react/reference/useMutation)를 참고하시길 바랍니다.

## **QueryInvalidation(invalidateQueries)**

> The QueryClient has an invalidateQueries method that lets you intelligently mark queries as stale and potentially refetch them too!

> When a query is invalidated with invalidateQueries, two things happen:
>
> - It is marked as stale. This stale state overrides any staleTime configurations being used in useQuery or related hooks
> - If the query is currently being rendered via useQuery or related hooks, it will also be refetched in the background

invalidateQueries는 queryClient의 메서드로 useQueryClient를 통해 호출합니다. invalidateQueries는 쿼리를 무효화 및 최신화합니다. 이를 이용하여 쿼리를 무효화 하게 되면 어떠한 staleTime 설정보다 우선시되어 해당 쿼리를 stale 상태로 만듭니다. 그리고 해당 쿼리가 이미 렌더링 중이라면 백그라운드에서 자동으로 새로운 데이터를 가져옵니다. 이를 통해 화면을 최신상태로 유지합니다. invalidateQueries에 옵션이 없는 경우에는 캐시 안에 있는 모든 쿼리를 무효화 및 최산화합니다. 옵션으로 쿼리 키를 넣어주면 해당 쿼리 키를 가진 모든 쿼리를 무효화 및 최신화합니다.

```tsx
import { useQuery, useQueryClient } from "@tanstack/react-query";

const queryClient = useQueryClient();

queryClient.invalidateQueries();
queryClient.invalidateQueries({ queryKey: ["todos"] });
```

# 마무리

지금까지 TanStack Query에 대해서 알아보았습니다. TanStack Query를 통해 서버 상태를 보다 쉽고 효율적으로 가져오고, 캐싱하며, 동기화 및 업데이트한다는 것을 알게 되었습니다. 또한, 핵심 개념 3가지에 대해 살펴보며 해당 개념의 속성 및 반환값에 대해서 알아보았습니다. 이번 기회를 통해 TanStack Query를 처음 써보는 개발자들이 더 쉽게 시작하는 계기가 되었으면 합니다. 이후 작업하면서 필요한 부분에 대해서는 공식 문서를 찾아보기를 권합니다. 기본적인 이해를 바탕으로 공식 문서를 살펴보면 처음 시작할 때보다 훨씬 쉽게 시작할 수 있을 것으로 기대합니다. 끝으로 TanStack Query의 핵심으로 마무리하겠습니다. `TanStack Query는 리액트 등에서 비동기(서버) 상태를 관리하는 강력한 도구이다.`

![alt text](image.png)

### 참고자료

---

https://tanstack.com/query/latest/docs/framework/react/overview

https://tanstack.com/query/v5/docs/reference/QueryClient#queryclient

https://tanstack.com/query/v5/docs/framework/react/reference/QueryClientProvider
https://tanstack.com/query/v5/docs/framework/react/devtools

https://tanstack.com/query/v5/docs/framework/react/guides/important-defaults

https://tanstack.com/query/v5/docs/framework/react/guides/caching

https://tanstack.com/query/latest/docs/framework/react/guides/queries

https://tanstack.com/query/latest/docs/framework/react/reference/useQuery

https://tanstack.com/query/latest/docs/framework/react/guides/mutations

https://tanstack.com/query/latest/docs/framework/react/reference/useMutation

https://tanstack.com/query/latest/docs/framework/react/guides/query-invalidation
