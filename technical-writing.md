# TanStack Query v5로 무한스크롤 끝장내기 🚀

스크롤만 내려도 콘텐츠가 끊기지 않고 자연스럽게 이어지면, 사용자는 ‘진짜 빠르네’라고 느낍니다. 하지만 실제로 이런 무한 스크롤을 구현하는 건 생각보다 까다롭습니다. 중복 호출을 막고, 커서도 정확히 관리해야 하죠. 또, 로딩과 에러 상태도 구분해서 표시해야 하고, 스크롤 트리거가 제대로 제어되는지도 신경 써야 합니다.

> 이번 글에서는 React와 TypeScript 환경에서 TanStack Query v5, 그리고 브라우저의 `IntersectionObserver`만 활용해서 부드럽고 믿을 수 있는 무한 스크롤을 만드는 실전 노하우를 정리했습니다.

#### 이 글에서 다루는 내용

- `infiniteQueryOptions`로 이전하는 방법과 `getNextPageParam` 설계 팁
- `useInfiniteQuery`의 `fetchNextPage`, `isFetchingNextPage` 상태 구분
- 커서 기반 페이지네이션과 `resetQueries`를 활용한 데이터 갱신 전략
- 스크롤 트리거 제어와 스켈레톤 UI를 늦게 보여주는 방식
- 여러 곳에서 쓸 수 있는 `useInfiniteScroll` 커스텀 훅 예시

#### 다루지 않는 내용

- 백엔드 API 설계나 DB 인덱스 전략
- SSR, ISR 등 프레임워크별 캐시 무효화 심화 내용
- `react-virtual` 같은 가상 스크롤 라이브러리의 깊은 활용법
- 접근성 전체에 대한 가이드(핵심 원칙 정도만 언급)

#### 이런 분께 추천합니다

- React 18 이상과 TypeScript를 이미 쓰고 있고, TanStack Query v5도 기본적으로 다뤄 본 프론트엔드 개발자라면 프레임워크가 Next.js든 CRA든 상관없이 도움이 될 겁니다.

> 이 글을 읽으면 복잡한 상태도 쿼리 옵션과 커스텀 훅으로 깔끔하게 정리하고, 로딩 때문에 사용자가 답답함을 느끼지 않도록 UI를 설계하는 방법을 배울 수 있습니다.

![image4](/assets/bottom-thumbnail.svg)

## 1. TanStack Query v5 소개

TanStack Query는 React 애플리케이션에서 서버 상태를 관리하는 데 유용한 라이브러리입니다. `데이터 패칭, 캐싱, 동기화 및 업데이트를 간편하게 처리`할 수 있도록 도와줍니다. 특히 v5에서는 성능과 사용성을 더욱 개선하여 개발자들이 더욱 쉽게 사용할 수 있도록 설계되었습니다.

![image0](/assets/react-query.png)

## 2. 무한 스크롤의 필요성

무한 스크롤은 사용자가 스크롤을 내릴 때 자동으로 추가 콘텐츠를 로드하는 기능입니다. 이는 페이지네이션보다 더 매끄러운 사용자 경험을 제공하며, 특히 모바일 환경에서 유용합니다. 사용자는 버튼을 클릭할 필요 없이 콘텐츠를 계속해서 탐색할 수 있습니다.

아래 GIF는 페이지네이션 대비 자연스러운 스크롤 경험을 시각적으로 보여줍니다.
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

설치가 완료되면, React 애플리케이션의 최상위 컴포넌트에서 `QueryClientProvider`로 감싸줍니다.

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

다음 다이어그램은 관찰 대상 영역과 트리거 지점의 관계를 간단히 나타냅니다.
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

## 6. `infiniteQueryOptions`로 마이그레이션

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
        lastPage.hasNext ? lastPage.nextCursor : null,
      initialPageParam: null,
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

아래 GIF는 실제 구현 결과로, 스크롤 하단 진입 시 다음 페이지가 자연스럽게 이어지는 모습을 보여줍니다.
![image3](/assets/result.gif)

## 9. 성능 최적화 및 팁

무한 스크롤을 구현할 때 성능을 최적화하는 것이 중요합니다. 다음은 몇 가지 팁입니다.

- **스켈레톤 지연**: 스크롤이 트리거 되었다고 해서, 무조건 스켈레톤 UI를 먼저 표시하고 추가 콘텐츠를 표시하지 않아도 됩니다. 추가 콘텐츠가 매우 빨리 도착했을 때, 스켈레톤 UI가 오히려 화면을 번쩍거리게 만들어 UX를 저해시키는 경우도 있습니다. 이런 경우 오히려 스켈레톤 UI를 표시하지 않고, 추가 콘텐츠를 바로 표시하는 것이 더 낫습니다.
- **데이터 캐싱** : TanStack Query는 데이터를 자동으로 캐싱하므로, 불필요한 API 호출을 줄일 수 있습니다.
- **로딩 상태 관리** : 로딩 상태를 적절히 관리하여 사용자에게 피드백을 제공하는 것이 중요합니다.
- **에러 처리** : API 호출 중 에러가 발생할 경우, 사용자에게 적절한 메시지를 제공해야 합니다.

## 10. 마무리 및 추가 자료

### 핵심 요약

- TanStack Query v5의 `infiniteQueryOptions`와 `useInfiniteQuery`를 활용해 커서 기반 무한 스크롤 구현 과정을 간단하게 만들었습니다.
- 브라우저의 `IntersectionObserver`와 `useInfiniteScroll` 훅을 적용해, 스크롤 트리거 역할을 분리했습니다.
- `isLoading`과 `isFetchingNextPage`를 따로 관리해, 초기 로딩과 추가 데이터 로딩 UX를 좀 더 세밀하게 조정할 수 있습니다.
- 스켈레톤을 지연 표시해 화면이 갑자기 바뀌는 현상을 줄이고, 사용자가 느끼는 속도를 높였습니다.
- 상황별 가이드: 추가 데이터가 필요할 땐 `fetchNextPage`, 전체 리셋이나 필터 변경 시엔 `resetQueries`가 적합합니다.

### 적용 체크리스트

- [ ] `QueryClientProvider`로 앱 전체를 감쌉니다.
- [ ] `infiniteQueryOptions`에서 `getNextPageParam`과 `initialPageParam`을 꼭 설정합니다.
- [ ] `useInfiniteQuery`로 받은 데이터를 `pages.flatMap`으로 한 번에 펼칩니다.
- [ ] 센티널 역할을 하는 DOM 요소와 `useInfiniteScroll` 훅을 연결합니다.
- [ ] 로딩, 에러, 빈 상태, 스켈레톤 지연 표시를 모두 고려해 UI를 구성합니다.
- [ ] `hasNext`와 `scrollEnabled` 같은 조건으로 중복 호출을 막습니다.

### 다음 단계

- 나중엔 가상 스크롤(예: `react-virtual`)까지 적용해 렌더링 비용을 더 줄일 수 있습니다.
- 접근성도 중요하니, `aria-live` 속성 추가, 포커스 이동, 키보드 탐색 방식도 함께 고민해 보세요.
- 에러 로그 수집과 재시도(backoff) 전략도 도입할 만합니다.
- SSR/ISR 환경에서 사용한다면, 초기 데이터와 캐시 정책부터 꼼꼼히 설계해야 합니다.
- 실제 사용자 경험은 성능 모니터링(예: Web Vitals, Tracing) 도구를 써서 지속적으로 측정해 주세요.

### 참고 자료

- [React-Query(tanstack query v5)로 만들어보는 무한스크롤](https://velog.io/@fromjjong/React-Querytanstack-query-v5%EB%A1%9C-%EB%A7%8C%EB%93%A4%EC%96%B4%EB%B3%B4%EB%8A%94-%EB%AC%B4%ED%95%9C%EC%8A%A4%ED%81%AC%EB%A1%A4)
- [Next.js TanStack Query V5 무한 스크롤 구현](https://velog.io/@white0_0/Next.js-TanStack-Query-V5-%EB%AC%B4%ED%95%9C-%EC%8A%A4%ED%81%AC%EB%A1%A4-%EA%B5%AC%ED%98%84)
- [Tanstack query를 활용한 무한 스크롤 UI 구현하기](https://ji-hoon.github.io/blog/implement-infinite-scroll-w-tanstack-query)
- [Tanstack-Query(React-Query) v5 와 IntersectionObserver를 이용한 무한스크롤(useInfiniteQuery)](https://sikk.tistory.com/282)
- [useInfiniteQuery | TanStack Query React Docs](https://tanstack.com/query/v4/docs/framework/react/reference/useInfiniteQuery)

무한 스크롤을 통해 사용자에게 더 나은 경험을 제공해보세요!
