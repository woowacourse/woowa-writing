# TanStack Query는 왜 등장했는가

## 예상 독자

- React 기반 웹 애플리케이션을 개발하는 프론트엔드 개발자
- 비동기 데이터 처리와 상태 관리 패턴에 관심 있는 개발자
- useEffect/Redux의 한계를 경험하고 더 나은 해결책을 찾는 개발자

## 개요

React 애플리케이션에서 외부 API 데이터를 관리하는 것은 생각보다 복잡합니다. 로딩 상태, 에러 처리, 캐싱, 데이터 동기화 등 고려해야 할 사항이 많습니다. 이 글에서는 전통적인 `useEffect` 방식과 Redux 미들웨어의 한계를 분석하고, TanStack Query가 이를 어떻게 해결하는지 살펴봅니다.

## 문제 상황

React 애플리케이션에서 Todo 리스트를 외부 API로부터 가져와 화면에 표시하는 기능을 구현한다고 가정해봅시다. 가장 직관적인 방법은 `useEffect`와 `useState`를 조합하는 것입니다.

```jsx
import React, { useState, useEffect } from "react";
import axios from "axios";

const App = () => {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    const fetchData = async () => {
      setLoading(true);
      try {
        const response = await axios.get("http://localhost:4000/todos");
        setData(response.data);
      } catch (error) {
        setError(error);
      } finally {
        setLoading(false);
      }
    };
    fetchData();
  }, []);

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;
  // ... (데이터 렌더링)
};
```

이 코드는 동작하지만, 다음과 같은 문제가 발생합니다.

### 주요 문제점

**1. 상태 관리의 복잡성**

- 하나의 API 호출을 위해 세 가지 상태(data, loading, error)를 직접 관리해야 합니다
- 컴포넌트가 늘어날수록 반복적인 보일러플레이트 코드가 증가합니다

**2. 중복 요청 문제**

- 동일한 데이터를 여러 컴포넌트에서 필요로 할 때, 각 컴포넌트가 독립적으로 API를 호출합니다
- 네트워크 비용이 불필요하게 증가하고 서버 부하가 커집니다

**3. 데이터 동기화 이슈**

- 한 컴포넌트에서 데이터를 수정하면, 다른 컴포넌트의 데이터는 자동으로 갱신되지 않습니다
- 수동으로 모든 관련 컴포넌트를 찾아 리렌더링을 트리거해야 합니다

**4. 비즈니스 로직의 혼재**

- UI 렌더링 로직과 데이터 패칭 로직이 한 컴포넌트에 섞여 있습니다
- 코드의 가독성이 떨어지고 테스트가 어려워집니다

## 해결 시도: Redux 미들웨어

위 문제들을 해결하기 위해 많은 개발자들이 Redux와 미들웨어(Redux Thunk, Redux Saga)를 도입했습니다.

### Redux Thunk 방식

Redux Thunk는 액션 크리에이터에서 함수를 반환하여 비동기 로직을 처리합니다.

```jsx
// Action Creator
export const fetchTodos = () => async (dispatch) => {
  dispatch({ type: 'FETCH_TODOS_REQUEST' });

  try {
    const response = await axios.get('/todos');
    dispatch({
      type: 'FETCH_TODOS_SUCCESS',
      payload: response.data
    });
  } catch (error) {
    dispatch({
      type: 'FETCH_TODOS_FAILURE',
      error: error.message
    });
  }
};

// Reducer
const todosReducer = (state = initialState, action) => {
  switch (action.type) {
    case 'FETCH_TODOS_REQUEST':
      return { ...state, loading: true };
    case 'FETCH_TODOS_SUCCESS':
      return { ...state, loading: false, data: action.payload };
    case 'FETCH_TODOS_FAILURE':
      return { ...state, loading: false, error: action.error };
    default:
      return state;
  }
};
```

### Redux의 장점

**1. 중앙 집중식 상태 관리**

- 모든 데이터가 하나의 스토어에서 관리됩니다
- 상태 변화를 예측 가능하게 추적할 수 있습니다

**2. 비즈니스 로직 분리**

- 데이터 패칭 로직을 액션 크리에이터로 분리할 수 있습니다
- 컴포넌트는 UI 렌더링에만 집중합니다

**3. 일관된 에러 처리**

- 로딩 상태와 에러 상태를 중앙에서 관리합니다

### Redux의 한계

하지만 Redux도 완벽한 해결책은 아니었습니다.

**1. 과도한 보일러플레이트**

```jsx
// 하나의 API 호출을 위해 필요한 코드들
- Action Types (3개: REQUEST, SUCCESS, FAILURE)
- Action Creators (3개 이상)
- Reducer (switch-case 로직)
- 컴포넌트 연결 코드 (useDispatch, useSelector)

```

**2. 서버 상태 관리의 비효율성**

- Redux는 클라이언트 상태 관리에 최적화되어 있습니다
- 캐싱, 자동 리페칭, 백그라운드 업데이트 등의 기능이 없습니다
- 이런 기능들을 직접 구현하면 코드가 더욱 복잡해집니다

**3. 테스트의 복잡도**

- 비동기 액션 크리에이터를 테스트하려면 모킹이 필요합니다
- 다양한 응답 시나리오를 시뮬레이션하기 어렵습니다

## 해결 방향: TanStack Query의 등장

### 핵심 인사이트

Redux를 사용하면서 개발자들은 중요한 사실을 깨달았습니다.

> 서버에서 가져온 데이터는 클라이언트 상태와 근본적으로 다르다.
> 

클라이언트 상태(예: UI 토글, 폼 입력값)와 달리, 서버 상태는 다음과 같은 특성이 있습니다:

- **비동기적**: 네트워크를 통해 가져오므로 로딩 시간이 필요합니다
- **공유됨**: 여러 컴포넌트에서 동일한 데이터를 필요로 합니다
- **시간에 민감**: 시간이 지나면 오래된(stale) 데이터가 됩니다
- **소유권이 없음**: 다른 사용자나 프로세스가 언제든 변경할 수 있습니다

TanStack Query는 이러한 서버 상태의 특성에 최적화된 라이브러리입니다.

### TanStack Query의 핵심 기능

| 기능 | 설명 | 전통적 방식과의 차이 |
| --- | --- | --- |
| **자동 캐싱** | 동일한 데이터는 한 번만 가져오고 재사용 | useEffect는 매번 새로 요청 |
| **자동 리페칭** | 데이터가 오래되면 자동으로 갱신 | 수동으로 갱신 트리거 필요 |
| **백그라운드 업데이트** | 사용자 경험을 해치지 않고 데이터 갱신 | 로딩 화면이 반복 표시됨 |
| **쿼리 무효화** | 데이터 변경 시 관련 쿼리 자동 갱신 | 수동으로 모든 관련 상태 업데이트 필요 |

## TanStack Query 기본 사용법

### 1. 초기 설정

먼저 애플리케이션 최상단에 QueryClientProvider를 설정합니다.

```jsx
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { ReactQueryDevtools } from "@tanstack/react-query-devtools";

const queryClient = new QueryClient();

ReactDOM.createRoot(document.getElementById("root")).render(
  <QueryClientProvider client={queryClient}>
    <App />
    <ReactQueryDevtools initialIsOpen={false} />
  </QueryClientProvider>
);

```

### 2. 데이터 조회: useQuery

기존의 복잡한 `useEffect` 코드가 간단해집니다.

**Before (useEffect)**

```jsx
const [data, setData] = useState(null);
const [loading, setLoading] = useState(true);
const [error, setError] = useState(null);

useEffect(() => {
  const fetchData = async () => {
    setLoading(true);
    try {
      const response = await axios.get("/todos");
      setData(response.data);
    } catch (error) {
      setError(error);
    } finally {
      setLoading(false);
    }
  };
  fetchData();
}, []);

```

**After (useQuery)**

```jsx
const { data, isPending, isError } = useQuery({
  queryKey: ["todos"],
  queryFn: async () => {
    const response = await axios.get("/todos");
    return response.data;
  },
});

if (isPending) return <div>Loading...</div>;
if (isError) return <div>Error occurred</div>;

```

### 3. 데이터 변경: useMutation

데이터를 생성, 수정, 삭제할 때는 `useMutation`을 사용합니다.

```jsx
const queryClient = useQueryClient();

const { mutate } = useMutation({
  mutationFn: async (newTodo) => {
    return await axios.post("/todos", newTodo);
  },
  onSuccess: () => {
    // todos 쿼리를 무효화하여 자동 리페칭 트리거
    queryClient.invalidateQueries(["todos"]);
  },
});

// 사용
const handleSubmit = (content) => {
  mutate({ content });
};

```

### 4. 자동 데이터 동기화

`invalidateQueries`의 강력함은 여러 컴포넌트에서 드러납니다.

```jsx
// 컴포넌트 A: Todo 목록 표시
function TodoList() {
  const { data: todos } = useQuery({
    queryKey: ["todos"],
    queryFn: fetchTodos,
  });
  // ...
}

// 컴포넌트 B: Todo 추가 폼
function TodoForm() {
  const queryClient = useQueryClient();

  const { mutate } = useMutation({
    mutationFn: addTodo,
    onSuccess: () => {
      // 이 한 줄로 모든 컴포넌트의 todos가 자동 갱신됨
      queryClient.invalidateQueries(["todos"]);
    },
  });
  // ...
}

```

## 핵심 개념: 쿼리 생명주기

TanStack Query의 효율성은 정교한 생명주기 관리에서 나옵니다.

### 쿼리 상태

| 상태 | 설명 | 발생 시점 |
| --- | --- | --- |
| fresh | 데이터가 최신 상태 | staleTime이 지나지 않음 |
| stale | 데이터가 오래됨, 리페칭 필요 | staleTime이 경과 |
| fetching | 데이터를 가져오는 중 | API 호출 진행 중 |
| inactive | 사용되지 않는 쿼리 | 컴포넌트 언마운트 |
| deleted | 캐시에서 제거됨 | gcTime 경과 후 |

### 데이터 흐름 시나리오

**시나리오 1: 최초 데이터 로드**

1. 컴포넌트 마운트 → `useQuery` 실행
2. 캐시 확인 → 데이터 없음
3. `queryFn` 실행 (`isPending = true`)
4. 데이터 수신 → 캐시 저장 → 리렌더링
5. UI에 데이터 표시

**시나리오 2: 동일 데이터 재요청 (SWR 전략)**

1. 다른 컴포넌트에서 동일 `queryKey`로 `useQuery` 호출
2. 캐시 데이터 즉시 반환 (빠른 UI 표시)
3. 동시에 백그라운드에서 `queryFn` 실행 (데이터 최신화)
4. 새 데이터 도착 → 캐시 갱신 → 필요시 리렌더링

**시나리오 3: 데이터 변경 후 동기화**

1. `mutate` 함수 호출 (데이터 추가/수정/삭제)
2. 서버 요청 성공 → `onSuccess` 콜백 실행
3. `invalidateQueries` 호출
4. 해당 쿼리를 사용하는 모든 컴포넌트 자동 리페칭
5. 최신 데이터로 UI 갱신

## 필수 설정 옵션

### 시간 관련 옵션

```jsx
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 60 * 1000, // 1분 동안 fresh 상태 유지
      gcTime: 5 * 60 * 1000, // 5분 후 캐시 삭제
    },
  },
});
```

**staleTime vs gcTime**

| 옵션 | 기본값 | 의미 | 용도 |
| --- | --- | --- | --- |
| `staleTime` | 0ms | 데이터가 fresh한 시간 | 불필요한 리페칭 방지 |
| `gcTime` | 5분 | 캐시 보관 시간 | 메모리 관리 |

### 자동 리페칭 제어

```jsx
useQuery({
  queryKey: ["todos"],
  queryFn: fetchTodos,
  refetchOnMount: true,        // 컴포넌트 마운트 시
  refetchOnWindowFocus: true,  // 윈도우 포커스 시
  refetchOnReconnect: true,    // 네트워크 재연결 시
});
```

### 에러 처리

```jsx
useQuery({
  queryKey: ["todos"],
  queryFn: fetchTodos,
  retry: 3, // 실패 시 3번 재시도
  retryDelay: (attemptIndex) => Math.min(1000 * 2 ** attemptIndex, 30000),
});
```

## 고급 기능

### 1. 조건부 쿼리 실행

```jsx
function UserProjects({ userId }) {
  // userId가 있을 때만 쿼리 실행
  const { data } = useQuery({
    queryKey: ["projects", userId],
    queryFn: () => fetchProjects(userId),
    enabled: !!userId, // userId가 falsy면 쿼리 실행 안 함
  });
}
```

### 2. 종속 쿼리

```jsx
// 1. 먼저 사용자 정보 가져오기
const { data: user } = useQuery({
  queryKey: ["user"],
  queryFn: fetchUser,
});

// 2. 사용자 정보가 있으면 프로젝트 가져오기
const { data: projects } = useQuery({
  queryKey: ["projects", user?.id],
  queryFn: () => fetchProjects(user.id),
  enabled: !!user?.id, // user 데이터가 있을 때만 실행
});
```

### 3. 데이터 변형 (select)

```jsx
const { data: username } = useQuery({
  queryKey: ["user"],
  queryFn: fetchUser,
  select: (user) => user.name, // 원본 데이터에서 name만 추출
});

// 캐시에는 전체 user 객체가 저장되고
// 컴포넌트에는 name만 전달됨
```

### 4. Optimistic Updates

사용자 경험을 극대화하기 위해 서버 응답 전에 UI를 먼저 업데이트합니다.

```jsx
const { mutate } = useMutation({
  mutationFn: updateTodo,
  onMutate: async (newTodo) => {
    // 진행 중인 리페칭 취소
    await queryClient.cancelQueries(["todos"]);

    // 이전 데이터 백업
    const previousTodos = queryClient.getQueryData(["todos"]);

    // Optimistic Update
    queryClient.setQueryData(["todos"], (old) => [...old, newTodo]);

    return { previousTodos };
  },
  onError: (err, newTodo, context) => {
    // 에러 발생 시 롤백
    queryClient.setQueryData(["todos"], context.previousTodos);
  },
  onSettled: () => {
    // 성공/실패 관계없이 최종 동기화
    queryClient.invalidateQueries(["todos"]);
  },
});
```

## 상태 비교: isPending vs isFetching

이 두 상태의 차이를 이해하는 것이 중요합니다.

| 상황 | isPending | isFetching | 설명 |
| --- | --- | --- | --- |
| 최초 로딩 | `true` | `true` | 캐시 없음, 데이터 가져오는 중 |
| 백그라운드 리페칭 | `false` | `true` | 캐시 있음, 업데이트 중 |
| 데이터 표시 | `false` | `false` | 데이터 사용 가능 |

```jsx
function Todos() {
  const { data, isPending, isFetching } = useQuery({
    queryKey: ["todos"],
    queryFn: fetchTodos,
  });

  // 최초 로딩
  if (isPending) return <div>Loading...</div>;

  return (
    <div>
      {/* 백그라운드 업데이트 표시 */}
      {isFetching && <span>Updating...</span>}

      {/* 데이터 표시 */}
      {data.map(todo => <TodoItem key={todo.id} todo={todo} />)}
    </div>
  );
}
```

## 실전 예제: 완전한 Todo 앱

```jsx
import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query";
import axios from "axios";

// API 함수들
const fetchTodos = async () => {
  const { data } = await axios.get("/api/todos");
  return data;
};

const addTodo = async (content) => {
  const { data } = await axios.post("/api/todos", { content });
  return data;
};

const deleteTodo = async (id) => {
  await axios.delete(`/api/todos/${id}`);
};

function TodoApp() {
  const queryClient = useQueryClient();

  // 데이터 조회
  const { data: todos, isPending, isError } = useQuery({
    queryKey: ["todos"],
    queryFn: fetchTodos,
    staleTime: 30 * 1000, // 30초 동안 fresh
  });

  // 추가 뮤테이션
  const addMutation = useMutation({
    mutationFn: addTodo,
    onSuccess: () => {
      queryClient.invalidateQueries(["todos"]);
    },
  });

  // 삭제 뮤테이션
  const deleteMutation = useMutation({
    mutationFn: deleteTodo,
    onSuccess: () => {
      queryClient.invalidateQueries(["todos"]);
    },
  });

  const handleAdd = (content) => {
    addMutation.mutate(content);
  };

  const handleDelete = (id) => {
    deleteMutation.mutate(id);
  };

  if (isPending) return <div>Loading...</div>;
  if (isError) return <div>Error loading todos</div>;

  return (
    <div>
      <TodoForm onSubmit={handleAdd} />
      <TodoList
        todos={todos}
        onDelete={handleDelete}
        isDeleting={deleteMutation.isPending}
      />
    </div>
  );
}
```

## 결론

### 배운 점

**1. 서버 상태는 클라이언트 상태와 다르다**

- 비동기적이고, 공유되며, 시간에 민감한 특성을 가집니다
- 이를 일반 상태 관리 도구로 다루면 복잡도가 증가합니다

**2. 적절한 도구 선택의 중요성**

- useEffect: 간단한 일회성 요청
- Redux: 복잡한 클라이언트 상태 관리
- TanStack Query: 서버 상태 관리

### TanStack Query를 사용해야 하는 경우

- 여러 컴포넌트에서 동일한 API 데이터를 공유해야 하는 경우
- 자주 변경되거나 실시간성이 중요한 데이터를 다뤄야 하는 경우 (예: 주기적 갱신)
- 서버 데이터 캐싱을 통해 네트워크 요청을 최소화하고 성능을 개선하고 싶은 경우
- 복잡한 데이터 동기화 로직을 단순화하고 싶은 경우
- Optimistic Update(낙관적 업데이트)가 필요한 경우
- 네트워크 상태나 포커스 변화에 따른 자동 재요청/재시도가 필요한 경우

### 마이그레이션 전략

기존 프로젝트에 TanStack Query를 도입할 때는 점진적 접근이 효과적입니다.

1. 가장 복잡한 데이터 패칭 로직부터 적용
2. 자주 사용되는 공유 데이터에 적용
3. 나머지 API 호출을 점진적으로 마이그레이션

### 성능 고려사항

```jsx
// 너무 짧은 staleTime
staleTime: 0 // 매번 리페칭 (기본값)

// 데이터 특성에 맞는 staleTime 설정
staleTime: 5 * 60 * 1000 // 자주 변하지 않는 데이터 (5분)
staleTime: 30 * 1000      // 자주 변하는 데이터 (30초)
```

TanStack Query는 단순한 데이터 패칭 도구를 넘어, 서버 상태 관리의 모범적인 패턴을 제공합니다. 복잡한 비동기 처리에 얽매이지 않고, 핵심 비즈니스 로직에 집중할 수 있도록 도와줍니다.