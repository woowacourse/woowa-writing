# [개선] TanStack Query 도입을 통한 픽잇 서버 상태 관리

픽잇의 데이터 처리 과정은 단순히 `fetch`를 통해 데이터를 불러와서 사용하고 있었다.
초기에는 큰 문제가 없었지만, 프로젝트가 커지면서 중복 요청, 재요청 관리 등 여러 복잡한 이슈가 드러나기 시작했다.

결국 **서버 상태를 효과적으로 선언적으로 관리**하기 위해 TanStack Query를 도입하게 되었고, 이는 프로젝트 구조의 단순화와 효율적인 데이터 흐름을 만드는 계기가 되었다.

이번 글에서는 TanStack Query를 도입하며 겪은 서버 상태 관리의 변화와 이를 통해 픽잇 프로젝트의 데이터 관리 방식이 어떻게 개선되었는지를 중심으로 이야기해보려 한다.

---

## 📍1. 서버 상태 관리의 중요성 및 TanStack Query 도입 배경

### 1.1. 서버 상태 관리 라이브러리 검토

프론트엔드 개발에서 서버와의 데이터 통신은 매우 중요한 역할을 하며, 데이터가 복잡할수록 **효율적인 서버 상태 관리**가 필수적이다.
대표적인 서버 상태 관리 라이브러리는 `SWR`, `TanStack Query`, `Apollo Client` 등이 있다.
픽잇 프로젝트는 GraphQL을 사용하지 않으므로 `Apollo Client`는 제외하였고, npm trends를 통해 `TanStack Query`가 높은 인기를 보임을 확인했다.

![](https://velog.velcdn.com/images/minji2219/post/664a0394-cc5d-4666-bf28-e82bff1da19e/image.png)

### 1.2. TanStack Query의 주요 이점

`TanStack Query`는 단순 데이터 요청 이상의 기능을 제공하며, 캐싱, 자동 갱신, 중복 요청 방지 등으로 개발 업무를 수월하게 만들어준다. 픽잇에서 **기존 데이터 통신의 불편함**을 경험하며 `TanStack Query` 도입을 결정하게 됐다.

---

## 📥2. 픽잇의 기존 데이터 통신 방식

### 2.1. 기존 방식(React Suspense, use, fetch)

픽잇은 React `Suspense`와 `use` 훅, `fetch` 함수를 조합해 데이터를 페칭하며 `Error Boundary`를 통해 에러를 처리하고 있다.
`Suspense`는 데이터 로드 완료 전까지 로딩 UI를 표시해 로딩 상태 관리 부담 감하고,
`Error Boundary`는 컴포넌트 내 발생 예외를 잡아 대체 UI를 제공한다.

```jsx
function RoomDetail() {
  const getWish = useMemo(() => wishlist.getWishTemplates(), []);
  return (
    <ErrorBoundary>
      <Suspense fallback={<LoadingSpinner />}>
        <TemplatesTab wishGroup={getWishGroup} />
      </Suspense>
    </ErrorBoundary>
  );
}

function TemplatesTab({ wishGroup }: Props) {
  const wishes = use(wishGroup);
  return (
    <S.Container>
      <S.Description>픽잇 찜({wishes?.length})</S.Description>
      {/* ... 위시리스트 맵핑 */}
    </S.Container>
  );
}
```

### 2.2. 기존 방식의 장점 및 한계

기존 방법에서도 `useEffect` 없이 데이터 페칭이 가능했고, `error`/`data`/`loading` 상태 정의가 불필요했다.
그러나 데이터 캐싱이 존재하지 않았기 때문에, **불필요한 데이터 재요청** 발생했다.
또한, **폴링·리페치·낙관적 업데이트** 등 수동 구현의 번거로움이 존재했다.

`TanStack Query`를 사용하는 이유가 단순 상태 정의 해소 목적에는 부적합했지만, 복잡한 비동기 로직 해결을 위해 `TanStack Query`를 도입하기로 결정했다.

---

## 👀3. TanStack Query 도입 필요성

### 3.1. 데이터 캐싱을 통한 리소스 낭비 해소

- 위시 상세 모달을 열 때마다 이미지 포함 데이터가 반복적으로 `fetch`가 된다.
- 위시리스트는 **변동이 적은 공용 데이터**로 클라이언트 캐시 재사용이 효율적이라고 생각이 됐다.
- `TanStack Query` **캐시 기능** 활용하여 해결하는 것을 기대했다.

![](https://velog.velcdn.com/images/minji2219/post/53e99761-0a03-4189-b39a-e37aa27047ed/image.gif)

### 3.2. 폴링 로직 간결화

- 함께 투표를 진행하므로 (유사)실시간성을 Polling을 통해 구현했다.
- 프로젝트 내에서 3초 주기로 직접 fetch Polling 구현했다.
- `TanStack Query`의 `refetchInterval` 옵션을 사용 시 코드가 간결해지고, 유지보수가 용이해질 것을 기대했다.

```jsx
// 기존 Polling 방식
export function usePolling<T>(
  fetcher: () => Promise<T>,
  {
    onData,
    interval = 3000,
    immediate = false,
    enabled = true,
    errorHandler = (error: Error) => {
      console.error('Polling error:', error.message);
    },
  }
) {
  const isUnmounted = useRef(false);

  useEffect(() => {
    if (!enabled) return;

    isUnmounted.current = false;

    const run = async () => {
      try {
        const result = await fetcher();
        if (!isUnmounted.current) onData(result);
      } catch (err) {
        if (isUnmounted.current) return;
        if (err instanceof Error) {
          errorHandler(err);
          return;
        }
        errorHandler(new Error(String(err)));
      }
    };

    // interval 시간 동안 폴링
    const intervalId = setInterval(run, interval);

    if (immediate) run();

    return () => {
      isUnmounted.current = true;
      clearInterval(intervalId);
    };
  }, []);
}
```

### 3.3. 수동 상태 갱신 감소: 쿼리 무효화 (Invalidation)

- POST 요청 후 UI 반영 위해 별도 리페치 호출 및 로컬 상태 관리가 필요했다.
- `TanStack Query` `useMutation`과 `invalidateQueries()`로 수동 상태 관리를 제거하기로 했다.

```jsx
// 기존 무효화 방식
const { error, wishlistData, handleGetWish } = useManageWishlist(wishId);
const handleCreateWish = () => {
  handleGetWish(); // 무효화
  handleUnmountModal();
};
```

```jsx
export const useManageWishlist = (wishId: number) => {
  // 상태 관리
  const [wishlistData, setWishlistData] = useState<Wishes[]>([]);
  const [error, setError] = useState<boolean>(false);
  const showToast = useShowToast();

  const handleGetWish = async () => {
    try {
         const response = await wishlist.get(wishId);
		 if (response) setWishlistData(response);
    } catch {
	     showToast({
	        mode: 'ERROR',
	        message: '찜 목록을 불러오던 중 에러가 발생했습니다.',
	     });
	     setError(true);
	  }
  };

  useEffect(() => {
    handleGetWish();
  }, []);

```

### 3.4. 사용자 경험 향상: 낙관적 업데이트 (Optimistic Update)

- 현재 투표 좋아요 버튼에서 서버 응답 기다리지 않고 즉시 UI를 반영하도록 **낙관적 업데이트**가 구현되어 있다.
- `TanStack Query`로 낙관적 업데이트를 간결하게 선언적 구현하고, 에러 발생 시 쉽게 롤백하는 효과를 기대했다.

### 3.5. 우아한 공통 에러 처리 로직

- QueryClient 설정으로 조회 쿼리는 Error Boundary로 `mutate`에러는 토스트 메시지로 처리하는 등 공통 에러 처리 및 UX를 개선하고자 했다.

---

## 📑4. TanStack Query 사용법

### 4.1. 설치 및 기본 설정

[[참고]](https://github.com/ssi02014/react-query-tutorial?tab=readme-ov-file)

**설치**

```
npm i @tanstack/react-query
```

**기본 설정**

```jsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

const queryClient = new QueryClient({
  defaultOptions: {
    // ...
  },
});

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <div>블라블라</div>
    </QueryClientProvider>
  );
}
```

### QueryClient

- QueryClient를 사용하여 캐시와 상호 작용할 수 있다.
- QueryClient에서 모든 query 또는 mutation에 **기본 옵션**을 추가할 수 있다.
- TanStack Query를 사용하기 위해서는 `QueryClientProvider`를 최상단에서 감싸주고 QueryClient 인스턴스를 `client props`로 넣어 애플리케이션에 연결해야 한다.

### 4.2. 개발 도구 (Devtools)

```jsx
npm i @tanstack/react-query-devtools
```

devtools를 사용하면 React Query의 모든 내부 동작을 시각화하는 데 도움이 되며 문제가 발생하면 디버깅 시간을 절약할 수 있다.

```jsx
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      {/* The rest of your application */}
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  );
}
```

### 4.3. 데이터 조회: useQuery

- `queryKey`(배열)와 `queryFn` (Promise 반환 함수) 필수.
- 반환 데이터: `data`, `error`, `status`, `isLoading`, `refetch` 등.

```jsx
const { data, isLoading } = useQuery({
  queryKey: ['fetchKey'],
  queryFn: fetchFn,
  // ...options ex) gcTime, staleTime, select, ...
});
```

### queryKey

- `queryKey`는 배열로 지정해 줘야 한다.
  이는 단일 문자열만 포함된 배열이 될 수도 있고, 여러 문자열과 중첩된 객체로 구성된 복잡한 형태일 수도 있다.
- queryKey를 기반으로 쿼리 캐싱을 관리하는 것이 핵심이다.
- 만약, 쿼리가 특정 변수에 의존한다면 배열에다 이어서 줘야 한다.
  ex: `["super-hero", heroId, ...]`

### **queryFn**

- `queryFn`는 `Promise`를 반환하는 함수를 넣어야 한다.

> ### 반환데이터

- `data`: 쿼리 함수가 리턴한 Promise에서 resolved된 데이터
- `error`: 쿼리 함수에 오류가 발생한 경우, 쿼리에 대한 오류 객체
- `status`: data, 쿼리 결과값에 대한 상태를 표현하는 status는 문자열 형태로 3가지의 값이 존재한다.
  - pending: 쿼리 데이터가 없고, 쿼리 시도가 아직 완료되지 않은 상태.
  - error: 에러 발생했을 때 상태
  - success: 쿼리 함수가 오류 없이 요청 성공하고 데이터를 표시할 준비가 된 상태.
- `isLoading`: 캐싱 된 데이터가 없을 때 즉, 처음 실행된 쿼리일 때 로딩 여부에 따라 true/false로 반환된다.
  - 이는 캐싱 된 데이터가 있다면 로딩 여부에 상관없이 false를 반환한다.
- `isSuccess`: 쿼리 요청이 성공하면 true
- `isError`: 쿼리 요청 중에 에러가 발생한 경우 true
- `refetch`: 쿼리를 수동으로 다시 가져오는 함수.

### useQuery 주요 옵션: **staleTime / gcTime**

#### staleTime: (number | Infinity)

- `staleTime`은 데이터가 `fresh`에서 `stale` 상태로 변경되는 데 걸리는 시간이다.
- **`fresh` 상태**일 때는 쿼리 인스턴스가 새롭게 mount 되어도 **네트워크 요청(fetch)이 일어나지 않는다**.
- 참고로, `staleTime`의 기본값은 0이기 때문에 일반적으로 fetch 후에 바로 stale이 된다.

**gcTime: (number | Infinity)**

- 쿼리 인스턴스가 unmount 되면 데이터는 `inactive` 상태로 변경되며 캐시는 `gcTime`만큼 유지된다.
- `gcTime`이 지나면 가비지 콜렉터로 수집된다.
- `gcTime`이 지나기 전에 쿼리 인스턴스가 다시 mount 되면, 데이터를 fetch 하는 동안 캐시 데이터를 보여준다.
- `gcTime`은 `staleTime`과 관계없이 무조건 `inactive` 된 시점을 기준으로 캐시 데이터 삭제를 결정한다.
- `gcTime`의 기본값은 5분이다. SSR 환경에서는 Infinity이다.

### 4.4. 데이터 변경: useMutation

- 만약 서버의 data를 `post`, `patch`, `put`, `delete`와 같이 수정하고자 한다면 이때는 `useMutation`을 이용한다.

```jsx
const { mutate } = useMutation({
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
});

const onCreateTodo = (e) => {
  mutate({ title });
};
```

- `useMutation`의 반환 값인 mutation 객체의 `mutate` 메서드를 이용해서 요청 함수를 호출할 수 있다.
- mutate는 `onSuccess`, `onError` 메서드를 통해 성공했을 시, 실패했을 시 response 데이터를 핸들링할 수 있다.
- `onMutate`는 mutation 함수가 실행되기 전에 실행되고, mutation 함수가 받을 동일한 변수가 전달된다. (낙관적 업데이트 등에 이용됨)
- `onSettled`는 `try...catch...finally` 구문의 `finally`처럼 요청이 성공하든 에러가 발생하든 상관없이 마지막에 실행된다.

---

## 🧀5. 픽잇에서 사용된 주요 기능

### 5.1. 폴링

- `refetchInterval`: 주기적 데이터 갱신
- `refetchIntervalInBackground`: 백그라운드 상태에서도 refetch

```jsx
export const restaurantsQuery = {
  useSuspenseGet: (pickeatCode: string, option?: QueryOption) => {
    const { pollingInterval = 0, ...restOption } = option ?? {};

    return useSuspenseQuery({
      queryKey: [RESTAURANTS_BASE_PATH, pickeatCode, restOption],
      queryFn: async () => restaurants.get(pickeatCode, restOption),
      refetchInterval: pollingInterval, // 리페치 주기 설정
    });
  },
};
```

### 5.2. 쿼리 무효화

- 데이터 변경 후 화면 최신화 필요 시 사용
- 쿼리 키 기준 빠른 리페치 실행

```jsx
usePostMember: (roomId: number) => {
    const showToast = useShowToast();

    return useMutation({
      mutationFn: ({ userIds }: { userIds: number[] }) =>
        room.postMember(roomId, userIds),
      onSuccess() {
        showToast({
          mode: 'SUCCESS',
          message: '초대 성공!',
        });
      // 쿼리 무효화
        queryClient.invalidateQueries({ queryKey: ['includeMembers', roomId] });
      },
      onError() {
        showToast({
          mode: 'ERROR',
          message: '초대에 실패했습니다. 다시 시도해 주세요.',
        });
      },
    });
  },
```

### 5.3. 낙관적 업데이트

- mutate 전 낙관적 업데이트가 필요한 데이터를 `onMustate`에서 업데이트
- Polling을 통해 3초 뒤에 데이터를 다시 `refetch` 해오기 때문에,
  이 때 실패시 롤백 성공시 업데이트 되도록 구현

```jsx
usePatchLike: (pickeatCode: string) => {
    const showToast = useShowToast();

    return useMutation({
      mutationFn: async (id: number) => await restaurant.patchLike(id),
      // 낙관적 업데이트 onMutate에서 진행
      onMutate: async (id: number, context) => {
        context.client.setQueryData(
          [RESTAURANTS_BASE_PATH, pickeatCode, { isExcluded: 'false' }],
          (oldData: Restaurant[] | undefined) => {
            return [
              ...(oldData || []).map(restaurant => {
                if (restaurant.id === id) {
                  return {
                    ...restaurant,
                    // 좋아요 상태와 좋아요 개수를 낙관적으로 업데이트
                    isLiked: true,
                    likeCount: restaurant.likeCount + 1,
                  };
                }
                return restaurant;
              }),
            ];
          }
        );
      },
      onError: () => {
        showToast({
          mode: 'ERROR',
          message: '좋아요 요청에 실패하였습니다.',
        });
      },
    });
  },
```

### 5.4. 기존 React 환경과의 통합: Error Boundary와 Suspense

#### 5.4.1 Error Boundary 및 에러 리셋

- `useQueryErrorResetBoundary`와 `ErrorBoundary` 결합해 선언적 에러 처리
- 페이지 전체 새로고침 없이 컴포넌트 단위 에러 복구 제공

> `useQueryErrorResetBoundary`를 사용하지 않으면?
> `useQueryErrorResetBoundary` 를 사용하지 않고 `queryClient` 옵션에 `{ throwOnError: true }` 를 적용하여 에러를 상위로 던질 수는 있으나, 에러 리셋과 재시도를 깔끔하게 구현하려면 `useQueryErrorResetBoundary`와 `QueryErrorResetBoundary` 컴포넌트를 사용하는 것이 권장된다.
> 에러 바운더리 내에서 에러 발생 시 사용자에게 새로고침 버튼을 제공하고, 버튼 클릭 시 `window.location.reload()`를 호출하여 페이지 전체를 새로고침한다면 전체 앱을 초기 상태로 돌려보내므로, 에러 상태 뿐 아니라 모든 상태와 캐시가 초기화되어 자연스럽게 에러가 해제된다.
> 다만, 이렇게 전체 페이지를 새로고침하는 UX는 사용자 입장에서 다소 불편할 수 있고, 특히 SPA의 장점인 빠르고 끊김 없는 사용자 경험이 저하될 수 있다. `useQueryErrorResetBoundary`와 함께 쓰면 사용자는 페이지 전체를 새로고침하지 않고도 컴포넌트 단위에서 에러 상태만 편리하게 초기화하며 재시도를 할 수 있어 더 나은 UX를 제공할 수 있다.

```jsx
import { useQueryErrorResetBoundary } from '@tanstack/react-query';
import { ErrorBoundary } from 'react-error-boundary';

interface Props {
  children: ReactNode;
}

const QueryErrorBoundary = ({ children }: Props) => {
  const { reset } = useQueryErrorResetBoundary();

  return (
    <ErrorBoundary
      onReset={reset}
      fallbackRender={({ resetErrorBoundary }) => (
        <div>
          Error!!
          <button onClick={() => resetErrorBoundary()}>Try again</button>
        </div>
      )}
    >
      {children}
    </ErrorBoundary>
  );
};

export default QueryErrorBoundary;
```

**5.4.2 Suspense**

- v5부터 안정화된 `useSuspenseQuery` 활용
- 최상단에 `QueryErrorBoundary`와 `Suspense` 조합해서 사용

```jsx
const { data } = useSuspenseQuery({
  queryKey: ['groups'],
  queryFn: fetchGroups,
});
```

```jsx
function App() {
  return (
    <QueryErrorBoundary>
      <Suspense fallback={<Loader />}>{/* 하위 컴포넌트들 */}</Suspense>
    </QueryErrorBoundary>;
  );
}
```

---

## ✨6. 도입 후 효과 및 느낀 점

### 6.1. 도입 후 효과

- **리소스 낭비 최소화 및 성능 향상**
  공용 위시리스트처럼 변동 낮은 데이터를 효율적으로 캐시 및 재사용하여 네트워크 요청과 로딩 시간 감소했다.

- **선언적 폴링 구현 및 코드 간결화**
  기존 수동 폴링 로직을 `refetchInterval` 옵션으로 대체하여 코드 양 감소 및 가독성이 향상됐다.
  **63줄에서 7줄로 약 84% 코드 감소**

- **상태 갱신 로직 단순화 (쿼리 무효화)**
  Mutation 후 별도의 수동 리페치 없이 `invalidateQueries`로 최신 상태를 유지했다.
  결과적으로 불필요한 상태 변수(data, loading, error) 3개를 제거하고, `useEffect` 의존 로직도 사라졌다.
  POST 요청 후 GET 요청으로 이어지는 로직이 한줄로 간소화됐다.

- **최적의 사용자 경험 제공 (낙관적 업데이트)**
  서버 응답 대기 없이 UI를 즉시 반영하여 복잡한 롤백도 표준화된 방식으로 안정적 처리했다.
  낙관적 업데이트를 처리하는 로직이 확연하게 줄어들었다.
  **133줄에서 32줄로 약 78% 코드 감소**

도입 결과 코드량은 평균적으로 70% 감소했고, 로딩 성능 향상 및 유지 보수성이 개선되었다.

### 6.2. 도입을 통해 느낀 점

처음에는 기존 React `Suspense`와 `use` 조합 방식의 장점을 충분히 누리고 있었기에 `TanStack Query`를 꼭 도입해야 할까 하는 의문이 있었다. 직접 사용해보기 전까지는, 익숙한 방식에서 벗어나는 것이 오히려 불필요한 변화로 느껴졌던 것 같다.

하지만 프로젝트가 진행될수록 점점 복잡해지는 비동기 데이터 로직에 직면하게 되었고, 그 과정에서 `TanStack Query`의 추상화와 선언적인 옵션의 강력함을 실제로 체감하게 됐다. 반복적인 폴링, 낙관적 업데이트, 리페치 등 다양한 수동 관리 코드를 간결한 선언형 옵션으로 바꿀 수 있었던 점이 가장 큰 변화였다.

`TanStack Query`는 단순한 데이터 페칭 라이브러리가 아닌 클라이언트-서버 데이터의 수명 주기 전반을 체계적으로 관리해주는 강력한 매니저라는 점을 프로젝트와 함께 깨닫게 됐다.

픽잇 프로젝트에는 총 36개의 API가 있었는데 이를 전면적으로 `TanStack Query`로 리팩토링하는 과정은 생각보다 까다로웠다. 중간에 `queryKey`를 잘못 지정해서 데이터 동기화에 문제가 생긴 적도 있었지만, devtool을 활용해 원인을 빠르게 찾아낼 수 었다.

이번 경험을 통해 새로운 기술을 무작정 도입하는 것보다 **직접 문제를 겪고 그 필요성을 스스로 깨닫는 과정**이 훨씬 더 가치 있다는 점을 깨달았다.
`TanStack Query` 역시 처음엔 단순히 "좋다고 하니 써볼까?"라는 마음이었지만, 프로젝트가 복잡해질수록 이 도구가 왜 필요한지를 이해하게 됐다.
앞으로도 단순한 도입이 아닌 **“어떤 문제를 해결하기 위한 도입”**를 선택할 수 있는 개발자가 되고 싶다.
