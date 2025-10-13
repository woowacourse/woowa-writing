# TanStack Query v5로 무한스크롤 끝장내기 🚀

최근 웹 개발에서 무한 스크롤은 사용자 경험을 향상시키는 중요한 요소로 자리 잡았습니다. 특히 React와 TypeScript를 사용하는 개발자들에게는 더욱 매력적인 기능입니다. `TanStack Query v5를 활용하여 무한 스크롤을 구현하는 방법`에 대해 자세히 알아보겠습니다.

## 1. TanStack Query v5 소개

TanStack Query는 React 애플리케이션에서 서버 상태를 관리하는 데 유용한 라이브러리입니다. `데이터 패칭, 캐싱, 동기화 및 업데이트를 간편하게 처리`할 수 있도록 도와줍니다. 특히 v5에서는 성능과 사용성을 더욱 개선하여 개발자들이 더욱 쉽게 사용할 수 있도록 설계되었습니다.

![image0](/assets/react-query.png)

## 2. 무한 스크롤의 필요성

무한 스크롤은 사용자가 스크롤을 내릴 때 자동으로 추가 콘텐츠를 로드하는 기능입니다. 이는 페이지네이션보다 더 매끄러운 사용자 경험을 제공하며, 특히 모바일 환경에서 유용합니다. 사용자는 버튼을 클릭할 필요 없이 콘텐츠를 계속해서 탐색할 수 있습니다.

![image1](/assets/infinite-scroll.gif)

## 3. TanStack Query v5 설치하기

먼저, TanStack Query를 설치해야 합니다. 다음 명령어 중 하나를 사용하여 설치할 수 있습니다.

```bash
npm install @tanstack/react-query
```

```bash
pnpm add @tanstack/react-query
```

```bash
yarn add @tanstack/react-query
```

설치가 완료되면, React 애플리케이션의 최상위 컴포넌트에서 QueryClientProvider로 감싸줍니다.

```tsx
// App.tsx
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";

const queryClient = new QueryClient();

function App() {
  return <QueryClientProvider client={queryClient}>...</QueryClientProvider>;
}
```

## 4. 스켈레톤 UI 준비

스크롤을 내려 자동으로 추가 콘텐츠를 로드하는 동안, 사용자에게 로딩 상태를 인지시키기 위해 스켈레톤 UI를 표시해야 합니다. 이는 UX적으로도 중요한 요소입니다.

```tsx
// CardSkeletonList.tsx
import CardSkeleton from "./CardSkeleton/CardSkeleton";

const SKELETON_COUNT = 20;

function CardSkeletonList() {
  return (
    <ul aria-label="프로젝트 목록 로딩 스켈레톤">
      {Array.from({ length: SKELETON_COUNT }, () => (
        <CardSkeleton key={crypto.randomUUID()} />
      ))}
    </ul>
  );
}

export default CardSkeletonList;
```

## 5. 스크롤 감지 커스텀 훅 구현

> 외부 라이브러리 사용을 최소화 하기 위해 `react-intersection-observer` 라이브러리 대신 기본 웹 API인 `IntersectionObserver`를 활용하였습니다.

`IntersectionObserver`를 활용하여, 스크롤이 되었음을 감지하는 커스텀 훅을 작성하였습니다. 무한스크롤 기능이 여러 곳에서 재사용될 수 있기 때문에, `useInfiniteScroll` 커스텀 훅에서는 스크롤이 감지되면, 외부에서 주입받은 refetch 함수를 호출하는 역할만 담당하고 있습니다.

![image2](/assets/area.png)

```ts
// useInfiniteScroll.ts
import { useCallback, useEffect, useRef } from "react";

interface UseInfiniteScrollProps {
  hasNext: boolean;
  nextCursor?: string;
  refetch: () => void;
  enabled?: boolean;
}

const useInfiniteScroll = ({
  hasNext,
  nextCursor,
  refetch,
  enabled = true,
}: UseInfiniteScrollProps) => {
  const targetRef = useRef<HTMLDivElement>(null);
  const observerRef = useRef<IntersectionObserver | null>(null);

  const handleIntersect = useCallback(
    (entries: IntersectionObserverEntry[]) => {
      const [entry] = entries;

      if (entry.isIntersecting && hasNext && nextCursor && enabled) {
        refetch();
      }
    },
    [hasNext, nextCursor, refetch, enabled]
  );

  useEffect(() => {
    const target = targetRef.current;
    if (!target) return;

    observerRef.current = new IntersectionObserver(handleIntersect, {
      threshold: 0.1,
      rootMargin: "20px",
    });

    observerRef.current.observe(target);

    return () => {
      if (observerRef.current) {
        observerRef.current.disconnect();
      }
    };
  }, [handleIntersect]);

  return { targetRef };
};

export default useInfiniteScroll;
```

## 6. `infiniteQueryOptions`로의 쿼리 옵션 수정

기존의 `queryOptions` 방식에서 커서(`getNextPageParam`)를 포함한 `infiniteQueryOptions`로 마이그레이션 하였습니다.

```ts
// project.queries.ts - useQuery
import { queryOptions } from "@tanstack/react-query";
import getProjects from "./getProjects";

export const projectQueries = {
  all: ["projects"] as const,
  fetchList: (cursor?: string) =>
    queryOptions({
      queryKey: projectQueries.all,
      queryFn: () => getProjects(cursor),
    }),
};
```

```ts
// project.queries.ts - useInfiniteQuery
import { infiniteQueryOptions } from "@tanstack/react-query";
import getProjects from "./getProjects";

export const projectQueries = {
  all: ["projects"] as const,
  fetchList: () =>
    infiniteQueryOptions({
      queryKey: projectQueries.all,
      queryFn: ({ pageParam }) => getProjects(pageParam),
      getNextPageParam: (lastPage) =>
        lastPage.hasNext ? lastPage.nextCursor : "",
      initialPageParam: "",
    }),
};
```

## 7. `useInfiniteQuery`를 활용한 데이터 쿼링

- 기존의 `useQuery` 방식에서 커서(`fetchNextPage`, `isFetchingNextPage`)를 포함한 `infiniteQueryOptions`로 마이그레이션 하였습니다.
- 최초 로딩 상태(`isLoading`)인지, 스크롤 되어 추가 콘텐츠 로딩 상태(`isFetchingNextPage`)인지를 구분합니다.
- `data`가 page별로 구분되어 2차원 배열로 return 되기 때문에, `flatMap` 고차 함수를 사용하여, 1차원 배열로 만들어 줍니다.
- 여러 로딩 상태와 조건들을 조합하여 스크롤 트리거를 방지하기도 하고, 스켈레톤 UI를 표시할지 말지도 결정합니다.

```ts
// useProjectList.ts
import { useInfiniteQuery, useQueryClient } from "@tanstack/react-query";
import { projectQueries } from "@/apis/projects/project.queries";
import useDelayedVisibility from "@/shared/hooks/useDelayedVisibility";

const useProjectList = () => {
  const queryClient = useQueryClient();
  const { data, isLoading, fetchNextPage, isFetchingNextPage } =
    useInfiniteQuery(projectQueries.fetchList());

  const projects = data?.pages.flatMap((page) => page.contents);

  const totalCount = data?.pages[0]?.totalCount ?? 0;

  const hasNext = data?.pages[data.pages.length - 1]?.hasNext ?? false;
  const nextCursor = data?.pages[data.pages.length - 1]?.nextCursor ?? "";

  const refetch = async () => {
    await queryClient.resetQueries({
      queryKey: projectQueries.all,
    });
  };

  const scrollEnabled = !isLoading && hasNext && !isFetchingNextPage;

  const showInitialSkeleton = useDelayedVisibility(isLoading);
  const showNextSkeleton = useDelayedVisibility(isFetchingNextPage);
  const showSkeleton = showInitialSkeleton || showNextSkeleton;

  return {
    projects,
    totalCount,
    nextCursor,
    fetchNextPage,
    refetch,
    hasNext,
    scrollEnabled,
    showSkeleton,
    isLoading,
  };
};

export default useProjectList;
```

## 8. 무한 스크롤 구현하기

이제 모든 준비가 완료되었습니다. 위의 코드를 바탕으로 무한 스크롤을 구현할 수 있습니다. 사용자가 스크롤을 내릴 때마다 새로운 데이터를 자동으로 로드하게 됩니다. 이로 인해 사용자 경험이 크게 향상됩니다.

![image3](/assets/result.gif)

## 8. 성능 최적화 및 팁

무한 스크롤을 구현할 때 성능을 최적화하는 것이 중요합니다. 다음은 몇 가지 팁입니다.

- **스켈레톤 지연**: 스크롤이 트리거 되었다고 해서, 무조건 스켈레톤 UI를 먼저 표시하고 추가 콘텐츠를 표시하지 않아도 됩니다. 추가 콘텐츠가 매우 빨리 도착했을 때, 스켈레톤 UI가 오히려 화면을 번쩍거리게 만들어 UX를 저해시키는 경우도 있습니다. 이런 경우 오히려 스켈레톤 UI를 표시하지 않고, 추가 콘텐츠를 바로 표시하는 것이 더 낫습니다.
- **데이터 캐싱** : TanStack Query는 데이터를 자동으로 캐싱하므로, 불필요한 API 호출을 줄일 수 있습니다.
- **로딩 상태 관리** : 로딩 상태를 적절히 관리하여 사용자에게 피드백을 제공하는 것이 중요합니다.
- **에러 처리** : API 호출 중 에러가 발생할 경우, 사용자에게 적절한 메시지를 제공해야 합니다.

## 9. 마무리 및 추가 자료

TanStack Query v5를 활용한 무한 스크롤 구현 방법에 대해 알아보았습니다. 이 기술을 통해 사용자 경험을 향상시키고, 더 나은 웹 애플리케이션을 개발할 수 있습니다. 추가적으로, 다음 링크들을 참고하시면 더 많은 정보를 얻을 수 있습니다.

- [React-Query(tanstack query v5)로 만들어보는 무한스크롤](https://velog.io/@fromjjong/React-Querytanstack-query-v5%EB%A1%9C-%EB%A7%8C%EB%93%A4%EC%96%B4%EB%B3%B4%EB%8A%94-%EB%AC%B4%ED%95%9C%EC%8A%A4%ED%81%AC%EB%A1%A4)
- [Next.js TanStack Query V5 무한 스크롤 구현](https://velog.io/@white0_0/Next.js-TanStack-Query-V5-%EB%AC%B4%ED%95%9C-%EC%8A%A4%ED%81%AC%EB%A1%A4-%EA%B5%AC%ED%98%84)
- [Tanstack query를 활용한 무한 스크롤 UI 구현하기](https://ji-hoon.github.io/blog/implement-infinite-scroll-w-tanstack-query)
- [Tanstack-Query(React-Query) v5 와 IntersectionObserver를 ...](https://sikk.tistory.com/282)

무한 스크롤을 통해 사용자에게 더 나은 경험을 제공해보세요!

![image4](/assets/bottom-thumbnail.svg)
