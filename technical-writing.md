# 들어가며

현재 커피빵 프로젝트에 참여하고 있습니다.

저는 기능 구현도 중요하지만, 서비스 운영에서 가장 중요한 것은 **사용자에게 안정적인 경험을 제공하는 것**이라고 생각합니다.

만약 에러 상황이 발생했을 때, 에러를 그대로 노출시키는 서비스가 있다면 어떨까요?

<img src="./images/1.png"/>

저는 개인적으로 당황스럽고 해당 서비스에 대한 신뢰도가 떨어지게 되는데요.

<br/>

이에 대한 고민을 하던 중 **커피빵**에서는 현재 에러 핸들링에 대한 규칙이 없다는 것을 깨달았습니다.

예상치 못한 에러가 발생했을 때 사용자에게 어떤 화면을 보여줄지, 개발자는 어떻게 에러를 추적하고 대응할지에 대한 일관된 전략이 필요하다고 느꼈습니다.

따라서 오늘은 커피빵 프로젝트에 적용할 **전역 에러 핸들링 전략**을 수립하고, 이를 통해 더 나은 사용자 경험을 제공하는 방법에 대해 정리해보려고 합니다.

<br/>

# 기존 에러 처리의 방식의 한계

기존에 커피빵에서 에러를 처리하는 방식을 살펴보면, 아래와 같은 문제가 있었습니다.

### 1. `api` 요청 과정에서 발생한 에러 핸들링 방식이 모두 다릅니다.

기존 커피빵의 코드를 보면, 아래와 같습니다.

먼저, 미니 게임에 대한 정보를 **get** 요청으로 받아올 때의 코드입니다.

```jsx
 useEffect(() => {
    (async () => {
      try {
        setLoading(true);
        const _miniGames = await api.get<MiniGameType[]>('/rooms/minigame');
        setMiniGames(_miniGames);
      } catch (error) {
        if (error instanceof ApiError) {
          setError(error.message);
        } else if (error instanceof NetworkError) {
          setError('네트워크 연결을 확인해주세요');
        } else {
          setError('알 수 없는 오류가 발생했습니다');
        }
      } finally {
        setLoading(false);
      }
    })();
  }, []);
```

반면 커피 메뉴를 받는 코드는 아래와 같습니다.

```jsx
useEffect(() => {
    (async () => {
      const menus = await api.get<Menu[]>(`/menu-categories/${selectedCategory.id}/menus`);
      setMenus(menus);
    })();
  }, [selectedCategory]);
```

어떤 곳은 **에러 처리를 전혀 하지 않고**, 어떤 곳은 **상태로 관리하며**, 또 다른 곳은 **다른 방식으로 처리**하는 등 일관성이 없습니다.

만약 이 모든 에러 처리를 한 곳에서 통합적으로 관리할 수 있다면, 코드의 응집성과 유지보수성이 크게 향상될 것입니다.

<br/>

### 2. 예상하지 못한 에러에 대한 핸들링 방안이 없다.

`try-catch`문을 통해서 `api` 요청에 대한 에러 핸들링은 어느 정도 진행해 주고 있지만, 막상 렌더링 과정이나 그 외 예상치 못한 에러에 대한 핸들링을 진행해 주지 않고 있었습니다.

따라서, 예상치 못한 에러 상황에서 사용자가 아래와 같은 에러 코드를 그대로 보게 되는 상황이 실제로 종종 발생했습니다.

<img src="./images/2.png" width="300px"/>

이는 사용자 경험을 크게 해치는 요소가 됩니다.

<br/>

### 3. 에러 발생 시 사용자에게 일관된 경험을 제공하지 못한다.

동일한 API 에러가 발생해도 화면마다 다른 방식으로 표현되고 있었습니다.

- A 페이지: 토스트 메시지로 표시
- B 페이지: 에러 텍스트로 표시
- C 페이지: 아무런 피드백 없음

이러한 불일치는 사용자에게 혼란을 주고, 서비스의 완성도를 떨어뜨립니다. 따라서 서비스 전체에서 일관된 에러 UX가 필요합니다.

따라서, 이러한 문제점들을 해결하기 위해 **전역 에러 핸들링 체계**를 구축하고자 했습니다. 모든 에러를 공통된 방식으로 처리함으로써 개발자는 에러 처리 로직을 반복해서 작성할 필요가 없어지고, 사용자는 **어디서든 일관된 경험**을 받을 수 있게 됩니다.

지금부터 커피빵 프로젝트에 적용할 구체적인 에러 핸들링 전략을 살펴보겠습니다.

<br/>

# 핸들링 해야 하는 에러

먼저 현재 프로젝트를 진행하면서 핸들링해야 하는 에러는 어떤게 있을지 생각해 보았습니다.

저는 많은 에러들 중에서 현재 저희 서비스에서 필수적으로 핸들링 해야 하는 에러는 아래 2가지가 있다고 생각했습니다.

1. **HTTP 통신 과정에서 발생한 에러**
2. **예상치 못한 에러**

따라서 이와 관련된 에러를 핸들링하는 과정을 다룰 예정입니다.

그 외 다른 에러(웹소켓 연결/통신이나 비즈니스 로직과 관련된 에러)들은 공통적으로 핸들링하기 어려운 부분이며, 이미 서비스 내부에서 개별적으로 처리되어 있다고 판단했기 때문에 이번 전략에서는 제외했습니다.

<br/>

# 에러 핸들링 전략

따라서 각각의 에러에 유형에 대해 아래의 전략을 세웠습니다.

### **1. `HTTP API`와 관련된 에러**

**1-1) `GET` 메서드를 통헤 데이터 패칭에 실패한 경우**

`GET` 요청으로 데이터를 받아오는 과정에서 에러가 발생하면, 이는 **데이터 패칭에 실패**했다는 의미입니다.

<img src="./images/3.png" width="300px"/>

예를 들어 위의 화면에서 에러가 발생한다면, 사용자들은 당연히 데이터를 다시 받아오고 싶을 것입니다.

**모든 `get` 요청에 대해 동일하다**고 생각하기 때문에, `get` 메서드를 통해 데이터를 패칭하는데 실패했다면, **지역적인 `Fallback UI`를 제공하여 데이터를 다시 받아올 수 있게끔 재시도**를 할 수 있도록 하고자 합니다.

<br/>

**1-2) get 이외의 메서드에서 요청이 실패 한 경우**

이는 `get` 이외의 메서드 (`post`, `delete`, `patch`)의 요청이 실패한 경우입니다.

<img src="./images/4.png" width="300px"/>

예를 들어, 위의 화면의 경우에 **방 만들러 가기**를 눌렀을 때, `post` 요청이 되는데요.

이때 사용자들은 에러가 나면 **내가 어떤 이유로 에러를 만났는지**를 궁금하게 될 텐데요. 따라서, 이때는 **`Toast`를 통해 에러의 원인을 명확하게 전달**하면 될 것 같습니다.

`GET`과 달리 재시도를 위한 별도의 `Fallback UI`는 제공하지 않습니다. 사용자는 에러 원인을 확인한 후 필요하다면 동일한 액션을 다시 시도할 수 있기 때문입니다.

<br/>

### 2. 예상치 못한 에러

예상치 못한 에러는 아래와 같이 분류할 수 있습니다.

**2-1. 렌더링 관련 에러**

컴포넌트 렌더링 중 발생할 수 있는 에러로는 `TypeError`, `ReferenceError` 등이 대표적입니다. 이는 다음과 같은 상황에서 발생할 수 있습니다.

- API 응답 구조가 예상과 다를 때
- 필수 데이터가 누락되었을 때
- 잘못된 데이터 타입을 참조할 때

이러한 경우를 핸들링 하기 위해 `ErrorBoundary`를 설정하고 해당 경우에 보여줄 `Fallback UI`를 설정하여 해당 `UI`를 보여주려고 합니다.

**2-2. 그 외 예상치 못한 에러**

실제 프로덕션 환경에서는 개발 단계에서 예측할 수 없는 다양한 에러가 발생할 수 있습니다.

- 사용자가 개발자 도구로 DOM을 직접 수정하는 경우
- 외부 서비스(CDN, 서드파티 라이브러리)의 장애
- 특정 브라우저 환경에서만 발생하는 이슈

이러한 경우에도 대응하기 위해 `Sentry`로 에러를 로깅하고 `ErrorBoundary`에서 `Fallback UI`를 보여줍니다.

**정리하면**, `GET` 요청 실패는 재시도 `UI`를, 그 외 `HTTP` 에러는 `Toast`를, 예상치 못한 에러는 `ErrorBoundary`와 `Sentry`를 활용하여 처리합니다.

<br/>
<br/>

# 에러 핸들링의 전체적인 구조

위에서 정의한 다양한 에러 케이스들을 어떻게 공통적으로 처리할 수 있을까 고민한 끝에, `ErrorBoundary`를 활용하여 통합적으로 핸들링하기로 결정했습니다.

<br/>

## ErrorBoundary란?

<aside>

하위 컴포넌트 트리의 어디에서든 **자바스크립트 에러를 기록**하며 깨진 컴포넌트 트리 대신 `Fallback UI`를 보여주는 컴포넌트

</aside>

```tsx
import * as React from 'react';

class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false };
  }

	// => 자식 컴포넌트에서 오류가 발생했을 때 호출
  static getDerivedStateFromError(error) {
    // state를 업데이트하여 다음 렌더링에 fallback UI가 표시되도록 합니다.
    return { hasError: true };
  }

  // => render 이후의 side effects를 다루는 메서드
  componentDidCatch(error, info) {
    // 에러 기록
    logErrorToMyService(
      ...
    );
  }

  render() {
    if (this.state.hasError) {
      // 사용자 지정 fallback UI를 렌더링할 수 있습니다.
      return this.props.fallback;
    }

    return this.props.children;
  }
}
```

`React`의 `ErrorBoundary`는 클래스 컴포넌트에서 제공하는 기능으로, 각 메서드는 아래와 같은 역할을 합니다.

<br/>

1. **getDerivedStateFromError**

```tsx
static getDerivedStateFromError(error) {
  // 다음 렌더링에서 Fallback UI를 표시하도록 상태 업데이트
  return { hasError: true, error };
}
```

- **실행 시점** : `render` 단계
- **특징**: 순수 함수로 관리되어야 하며, 부수 효과(side effect)를 포함할 수 없음
- **역할**: 자식 컴포넌트에서 에러가 발생했을 때 호출되어, Fallback UI를 렌더링하도록 상태를 업데이트

<br/>

2. **componentDidCatch**

```tsx
componentDidCatch(error, errorInfo) {
  // 에러 로깅 서비스에 에러 정보 전송
  logErrorToService(error, errorInfo);
}
```

- **실행 시점**: `commit` 단계
- **특징**: 부수 효과(side effect) 허용
- **역할**: 에러 로그를 외부 서비스(Sentry 등)에 전송하거나, 에러 정보를 기록할 때 사용

<br/>

3. **render**

```tsx
render() {
  if (this.state.hasError) {
    // 에러 발생 시 Fallback UI 렌더링
    return <ErrorFallbackUI error={this.state.error} />;
  }

  // 정상 상태에서는 자식 컴포넌트 렌더링
  return this.props.children;
}
```

- **역할**: 에러 상태에 따라 Fallback UI 또는 정상 컴포넌트를 렌더링
- **동작**: `hasError` 상태가 `true`이면 에러 화면을 보여주고, `false`이면 자식 컴포넌트를 그대로 렌더링

<br/>
<br/>

# 커피빵의 에러 핸들링 구조 설계하기

커피빵에서는 에러를 효과적으로 처리하기 위해 **2개의 ErrorBoundary**를 계층적으로 사용하기로 했습니다.

각각의 역할과 구조를 설명하겠습니다.

<br/>

## 전체 에러 처리의 흐름

그 전에, 먼저 에러가 발생되고 처리되는 전체적인 흐름은 아래와 같습니다.

<br/>

### 1. API 요청 단계

**apiRequest 함수에서 에러 발생**

`API` 요청 과정에서 `apiRequest` 함수에서 발생하는 에러를 세분화하여 처리합니다.

<img src="./images/5.png" width="400px"/>

<br/>

에러 분류 기준은 아래와 같습니다.

- **API 에러**: 서버에서 반환된 에러 응답 → `ApiError` 객체
- **네트워크 에러**: `TypeError` 또는 "Failed to fetch" 포함 → `NetworkError` 객체
- **그 외**: 예상치 못한 에러 → `Error` 객체

여기서 `useFetch`는 실제 `api` 요청 함수인 `apiRequest`를 래핑하는 함수이며, 컴포넌트 단에서 `api`를 요청할 때 사용되는 커스텀 훅입니다.

`useFetch`를 호출하는 컴포넌트에서 핸들링하지 않기 때문에 자연스럽게 **상위 컴포넌트로 에러가 전파**됩니다.

<img src="./images/6.png" width="500px"/>

<br/>

### 2. ErrorBoundary에서 캐치

이렇게 분류된 에러는 두 계층의 `ErrorBoundary`에서 처리됩니다.

앞서 봤던 `MiniGameSection`에서 에러가 발생한다면, 각 조건에 맞게 `LocalErrorBoundary` 또는 `GlobalErrorBoundary`에서 처리하게 될 것입니다.

<br/>
<img src="./images/7.png" width="300px"/>

<br/>
<br/>

## 1. GlobalErrorBoundary

<aside>

**최상단에서 에러를 잡아주는 에러 바운더리**입니다.

</aside>

아래와 같이 `App`의 최상단에 적용하도록 하면, 알 수 없는 에러나 런타임 에러에 대해 내부적으로 설정한 `Fallback UI`가 렌더링되도록 설계했습니다.

```tsx
const App = () => {
  return (
    <ThemeProvider theme={theme}>
      <IdentifierProvider>
        <ParticipantsProvider>
          <WebSocketProvider>
            <PlayerTypeProvider>
              <ProbabilityHistoryProvider>
                <ToastProvider>
                  <ModalProvider>
                    **
                    <GlobalErrorBoundary>
                      **
                      <Suspense fallback={<div>Loading...</div>}>
                        <Outlet />
                      </Suspense>
                      **
                    </GlobalErrorBoundary>
                    **
                  </ModalProvider>
                </ToastProvider>
              </ProbabilityHistoryProvider>
            </PlayerTypeProvider>
          </WebSocketProvider>
        </ParticipantsProvider>
      </IdentifierProvider>
    </ThemeProvider>
  );
};

export default App;
```

`App` 컴포넌트의 최상단에 배치하여, 애플리케이션 전체에서 발생하는 예상치 못한 에러를 최종적으로 캐치합니다.

<br/>

### 에러 처리 전략

`GlobalErrorBoundary`는 에러의 종류와 상황에 따라 다음과 같이 처리합니다.

**1) Toast 메시지로 처리**

- `GET` 요청 에러이면서 `display: Toast`로 설정된 경우
- `GET` 이외의 메서드(`POST`, `PATCH`, `DELETE`)에서 발생한 `API` 에러
  - `GET` 이외의 메서드는 무조건 `Toast`를 띄우도록 했습니다. 그 이유는 보통 특정 액션에 의해서 발생하는 요청이기 때문에 굳이 재시도를 할 `UI`를 보여줄 필요가 없다고 생각했습니다.

**2) Fallback UI로 처리**

- 렌더링 중 발생한 예상치 못한 에러
- 타입 에러, 참조 에러 등 알 수 없는 에러

**3) Modal로 처리**

- 현재는 사용하지 않지만, 향후 확장 가능

그럼 여기서 **모든 에러를 처리하면 되지 않느냐?** 라고 생각할 수 있는데요.

저는 해당 에러 바운더리 말고 **별도의 에러 바운더리를 하나 더 두어 에러 핸들링을 진행**해주었습니다.

<br/>

## 2. LocalErrorBoundary

<aside>

지역적인 에러를 잡기 위한 에러 바운더리입니다.

</aside>

### 필요한 이유?

`GlobalErrorBoundary`에는 한 가지 문제점이 있는데요.

바로 하위 컴포넌트에서 에러를 던지면 `GlobalErrorBoundary`를 감싸는 모든 컴포넌트에 대해서 `Fallback UI`로 대체하게 된다는 점입니다. 예를 들어, 사이드바의 작은 컴포넌트에서 에러가 발생했을 때 전체 페이지가 에러 화면으로 바뀌는 것은 자연스럽지 않습니다. 이런 경우 **에러가 발생한 부분만 Fallback UI로 대체**하는 것이 더 나은 사용자 경험을 제공한다고 생각했습니다.

따라서 `LocalErrorBoundary`는 지역적으로 에러가 발생하게 될 경우, 지역 `Fallback`을 띄워주기 위해 별도로 구현하게 되었습니다. 아래와 같이 지역 `Fallback`을 주입하면, 해당 컴포넌트를 렌더링하는 부분만 `Fallback UI`를 보여주기 위함입니다.

예를 들어, 아래와 같이 `LocalErrorBoundary`를 설정하고, `Fallback UI`를 주입하면

<img src="./images/8.png" width="500px"/>

해당 컴포넌트 내부에서 요청하는 `api` 내부에서 오류가 발생하면, 화면 전체가 `Fallback UI`로 갈아끼워지는 것이 아닌, 부분적인 `Fallback UI`가 들어가게 됩니다.

<img src="./images/9.png" width="300px"/>

### 에러 처리 전략

`LocalErrorBoundary`는 다음과 같은 경우에 지역 `Fallback UI`를 표시합니다.

**1) 지역 Fallback UI 표시**

- GET 요청 에러이면서 `display: Fallback`으로 설정된 경우
- 네트워크 에러가 발생한 경우

```tsx
<LocalErrorBoundary
  fallback={
    <ErrorFallback
      message="데이터를 불러오는데 실패했습니다"
      onRetry={refetch}
    />
  }
>
  <MiniGameList />
</LocalErrorBoundary>
```

**2) 그 외는 상위로 `throw`**

- 위 조건에 해당하지 않는 에러는 상위 `ErrorBoundary`(`GlobalErrorBoundary`)로 전파

이를 통해 지역적으로 처리할 수 없는 심각한 에러는 `GlobalErrorBoundary`에서 최종적으로 처리됩니다.

### 전체 구조 정리

```tsx
GlobalErrorBoundary (최상위)
  ├─ Toast: GET 이외 메서드 에러, display: Toast인 경우
  ├─ Fallback UI: 알 수 없는 에러, 렌더링 에러
    │
    └─ LocalErrorBoundary (지역)
        ├─ 지역 Fallback: GET + display: Fallback, 네트워크 에러
        └─ 상위로 throw: 그 외 모든 에러
```

<br/>
<br/>

# 커피빵에 적용해보기

앞서 수립한 에러 핸들링 전략을 실제 코드로 구현하는 과정을 단계별로 정리했습니다.

## **1. LocalErrorBoundary 구현**

**1-1. 기본 사용 방법**

`LocalErrorBoundary`는 특정 컴포넌트를 감싸서 **지역적인 에러를 처리**합니다.

```tsx
<LocalErrorBoundary>
  <MiniGameSection
    selectedMiniGames={selectedMiniGames}
    handleMiniGameClick={handleMiniGameClick}
  />
</LocalErrorBoundary>
```

**1-2. `API` 요청에 `displayMode` 추가**

에러가 발생했을 때 어떻게 표시할지 설정할 수 있도록 `displayMode`를 추가했습니다.

**ApiRequestOptions 타입 수정**

```tsx
export type ApiRequestOptions<TData> = {
  method?: Method;
  headers?: Record<string, string>;
  body?: TData;
  retry?: {
    count: number;
    delay: number;
  };
  **displayMode?: ErrorDisplayMode; // 추가**
};
```

**apiRequest 함수에서 displayMode 처리**

```tsx
if (!response.ok) {
  // GET은 기본적으로 fallback, 나머지는 toast
  const display = displayMode || (method === "GET" ? "fallback" : "toast");
  const apiError = new ApiError(
    response.status,
    errorMessage,
    errorData,
    display
  );
  throw apiError;
}
```

**1-3. LocalErrorBoundary 클래스 구현**

에러 타입과 `displayMode`에 따라 적절히 처리하는 `LocalErrorBoundary`를 구현했습니다.

```tsx
import { Component, ReactNode } from "react";
import { ApiError, NetworkError } from "@/apis/rest/error";
import LocalErrorFallback from "./LocalErrorFallback";

interface Props {
  children: ReactNode;
  fallback?: (error: Error, retry: () => void) => ReactNode;
}

interface State {
  error: Error | null;
}

class LocalErrorBoundary extends Component<Props, State> {
  constructor(props: Props) {
    super(props);
    this.state = { error: null };
  }

  static getDerivedStateFromError(error: Error): State {
    // GET 요청이 아닌 API 에러는 상위로 전파
    if (error instanceof ApiError) {
      if (error.method !== "GET") {
        throw error;
      }

      // displayMode가 fallback인 경우만 처리
      if (error.displayMode === "fallback") {
        return { error: error as ApiError };
      }
    }

    // 네트워크 에러는 처리
    if (error instanceof NetworkError) {
      return { error: error as NetworkError };
    }

    // 그 외는 상위로 전파
    throw error;
  }

  handleRetry = (): void => {
    this.setState({ error: null });
  };

  render(): ReactNode {
    if (this.state.error) {
      if (this.props.fallback) {
        return this.props.fallback(this.state.error, this.handleRetry);
      }

      return (
        <LocalErrorFallback
          error={this.state.error}
          handleRetry={this.handleRetry}
        />
      );
    }

    return this.props.children;
  }
}

export default LocalErrorBoundary;
```

<br/>

### 구현 결과

실제로 개발자 도구에서 `network` → `offline`으로 설정한 후 실행해 보았더니, 아래와 같이 지역 `Fallback` 화면이 잘 뜨는 것을 볼 수 있었습니다.

<img src="./images/10.png" width="300px"/>

<br/>

## 2. **GlobalErrorBoundary 구현**

**2-1. 클래스 컴포넌트에서 Hook 사용 문제**

`GlobalErrorBoundary`에서 `Toast`를 띄우려면 `useToast hook`을 사용해야 하는데, **클래스 컴포넌트에서는 `hook`을 직접 사용할 수 없습니다**.

**왜 클래스 컴포넌트에서 Hook을 사용할 수 없을까?**

**1. Hook은 함수 컴포넌트의 렌더링 사이클에 의존합니다**

```tsx
// 함수 컴포넌트
function MyComponent() {
  const [state, setState] = useState(0);
  // React는 함수가 호출될 때마다:
  // 1. Hook 호출 순서를 추적
  // 2. 해당 컴포넌트 인스턴스의 상태를 관리
  // 3. 리렌더링 시 같은 순서로 Hook 호출

  return <div>{state}</div>;
}

// 클래스 컴포넌트
class MyComponent extends Component {
  render() {
    const { showToast } = useToast(); // 에러!
    // render는 메서드이지, 함수 컴포넌트가 아님

    return <div />;
  }
}
```

1. **Hook의 순서 보장**

```tsx
function MyComponent() {
  const [name, setName] = useState("");
  const [age, setAge] = useState(0);

  // React는 Hook 호출 순서로 상태를 구분함
  // 첫 번째 useState = name
  // 두 번째 useState = age
}
```

클래스 컴포넌트는 메서드가 여러 번, 다른 순서로 호출될 수 있어서 이 보장이 불가능합니다.

<br/>

**2-2. 해결 방법: Context API 직접 사용**

클래스 컴포넌트에서 `Context`를 직접 사용하여 `Toast` 기능을 활용할 수 있습니다.

```tsx
import React, { Component, ReactNode } from "react";
import { ApiError, NetworkError } from "../rest/error";
import { ToastContext } from "@/components/@common/Toast/ToastContext";
import GlobalErrorFallback from "./GlobalErrorFallback";

interface Props {
  children: ReactNode;
  fallback?: ReactNode;
}

interface State {
  error: Error | null;
}

class GlobalErrorBoundary extends Component<Props, State> {
  // Context를 사용하기 위한 설정
  static contextType = ToastContext;
  declare context: React.ContextType<typeof ToastContext>;

  constructor(props: Props) {
    super(props);
    this.state = { error: null };
  }

  static getDerivedStateFromError(error: Error): State {
    // Toast로 처리할 에러는 상태에 저장하지 않음
    if (error instanceof ApiError && error.displayMode === "toast") {
      return { error: null };
    }

    if (error instanceof NetworkError && error.displayMode === "toast") {
      return { error: null };
    }

    // Fallback UI를 보여줄 에러만 상태에 저장
    return { error };
  }

  componentDidCatch(error: Error) {
    // Context를 통해 Toast 표시
    if (error instanceof ApiError && error.displayMode === "toast") {
      this.context?.showToast({
        type: "error",
        message: error.message,
      });
    }

    if (error instanceof NetworkError && error.displayMode === "toast") {
      this.context?.showToast({
        type: "error",
        message: "네트워크 오류가 발생했습니다. 다시 시도해주세요.",
      });
    }
  }

  render(): ReactNode {
    if (this.state.error) {
      return (
        this.props.fallback || <GlobalErrorFallback error={this.state.error} />
      );
    }

    return this.props.children;
  }
}

export default GlobalErrorBoundary;
```

**처리 흐름**

1. `getDerivedStateFromError`: Toast로 처리할 에러는 `null` 반환 (UI 유지), Fallback으로 처리할 에러는 상태에 저장
2. `componentDidCatch`: Context를 통해 Toast 표시
3. `render`: 에러 상태에 따라 Fallback UI 또는 children 렌더링

**2-3. GlobalErrorFallback 구현**
전역 에러는 **앱 전체가 동작할 수 없는 심각한 상황**이므로, 재시도가 아닌 **메인 페이지로 이동**하도록 구현했습니다.

<img src="./images/11.png" width="300px"/>

<br/>

**2-4. 왜 GlobalErrorFallback은 메인으로, LocalErrorFallback은 다시 시도인가?**

**GlobalErrorFallback**의 상황

- 앱 전체를 감싸고 있어서, 여기까지 온 에러는 **심각한 에러**
- LocalErrorBoundary를 통과한 에러
- 앱 전체가 동작할 수 없는 상황
- 재시도해도 복구가 어려운 경우

**LocalErrorFallback**의 상황

- 특정 컴포넌트에서 발생한 **지역적인 에러**
- 데이터 로딩 실패 등 **재시도로 해결 가능한 에러**
- 앱의 다른 부분은 정상 동작 중

<br/>

# 개선 효과

체계적인 에러 핸들링 시스템을 구축한 후, 커피빵 프로젝트에 여러 개선된 변화가 있었습니다.

### 문제 1. API 요청 에러 핸들링 방식의 불일치 해결

**Before: 제각각인 에러 처리**

```tsx
// 컴포넌트 A: 상세한 에러 처리
useEffect(() => {
  (async () => {
    try {
      setLoading(true);
      const data = await api.get("/rooms/minigame");
      setData(data);
    } catch (error) {
      if (error instanceof ApiError) {
        setError(error.message);
      } else if (error instanceof NetworkError) {
        setError("네트워크 연결을 확인해주세요");
      } else {
        setError("알 수 없는 오류가 발생했습니다");
      }
    } finally {
      setLoading(false);
    }
  })();
}, []);

// 컴포넌트 B: 에러 처리 누락
useEffect(() => {
  (async () => {
    const menus = await api.get("/menu-categories/1/menus");
    setMenus(menus);
  })();
}, []);
```

**문제점**

- 어떤 컴포넌트는 15줄의 에러 처리 코드, 어떤 컴포넌트는 0줄
- 새로운 API 호출할 때마다 "이번엔 에러를 어떻게 처리하지?"에 대한 고민 반복

**After: 통합된 에러 처리**

```tsx
// 모든 컴포넌트에서 동일한 패턴
const { data, error } = useFetch({
  endpoint: "/rooms/minigame",
  displayMode: "fallback", // 또는 'toast'
});

if (error) throw error;
```

**개선 효과**

- 모든 API 호출이 동일한 방식으로 에러 처리
- `displayMode` 하나만 결정하면 나머지는 자동 처리

### 문제 2: 예상치 못한 에러의 노출 방지

**Before: 사용자에게 그대로 노출되는 에러**

```tsx
Uncaught TypeError: Cannot read property 'map' of undefined
    at MiniGameSection.tsx:24
    at renderWithHooks
    ...
```

**문제점**

- 렌더링 중 발생한 에러를 처리하지 못함
- 사용자가 개발자용 에러 메시지를 그대로 봄
- "이 서비스 괜찮은 건가?" 하는 불신 발생

**After: ErrorBoundary로 안전하게 처리**

```tsx
<GlobalErrorBoundary>
  <App />
</GlobalErrorBoundary>
```

**개선 효과**

- 모든 예상치 못한 에러를 `ErrorBoundary`가 캐치
- 사용자에게는 친화적인 에러 화면 표시
- 개발자는 콘솔에서 상세한 에러 정보 확인 가능

<br/>

### DX 향상

에러 핸들링 시스템을 구축하면서 예상하지 못했던 부분이 하나 있었습니다. 바로 개발자 경험의 개선이었습니다.

1. **반복 작업에서 해방**

이전에는 새로운 API를 호출할 때마다 똑같은 고민을 반복해야 했습니다. "에러는 어떻게 처리하지?", "네트워크 에러는 따로 처리해야 하나?" 같은 질문들이 그 예시입니다.

하지만 이제는 `displayMode`만 결정하면 나머지는 에러 바운더리에서 처리해주게 되었습니다. 따라서 반복되는 작업을 줄이고, 팀원들은 비즈니스 로직에 집중할 수 있게 되었습니다.

2. **심리적 안정감**

기존에는 “혹시 내가 에러 처리를 빠뜨린 거 아닐까?”, “배포 후 에러가 나면 어쩌지?”와 같은 불안감이 있었습니다. 하지만 `ErrorBoundary`라는 최종 안전망이 있고, 팀 전체가 일관된 패턴을 사용한다는 확신이 생겼기 때문에 심리적 안정감이 올라가게 되었습니다.

<br/>

# **학습한 점**

이번 에러 핸들링 시스템을 구축하면서 여러 가지를 배울 수 있었습니다.

---

**1. ErrorBoundary의 한계와 비동기 처리**

ErrorBoundary는 렌더링 단계에서 발생한 에러만 캐치할 수 있다는 중요한 제약이 있었습니다. 비동기 작업 내부에서 던진 에러는 캐치되지 않기 때문에, 에러를 상태로 관리한 후 렌더링 시점에 `throw`하는 패턴이 필요했습니다.

**2. 클래스 컴포넌트와 Hook의 제약**

ErrorBoundary를 구현하려면 클래스 컴포넌트를 사용해야 하는데, 클래스 컴포넌트에서는 Hook을 직접 사용할 수 없었습니다. 이를 해결하기 위해 `Context API`를 직접 활용하는 방법을 배웠습니다. React 팀이 함수형 `ErrorBoundary`를 제공하지 않는 이유와 그 한계를 이해할 수 있었습니다.

**3. 에러의 계층적 처리**

모든 에러를 한 곳에서 처리하는 것보다, **에러의 성격과 범위에 따라 계층적으로 처리**하는 것이 더 효과적이라는 것을 깨달았습니다. 지역적인 데이터 로딩 실패는 해당 영역만 Fallback UI로 대체하고, 심각한 에러는 전역에서 처리하는 구조가 사용자 경험 측면에서 훨씬 자연스러웠습니다.

**4. 사용자 관점에서의 에러 처리**

단순히 에러를 잡는 것이 아니라, **사용자가 무엇을 원할지 고민**하는 것이 중요했습니다. 데이터 조회 실패 시에는 재시도 버튼을, 액션 실패 시에는 원인을 알려주는 Toast를, 심각한 에러 시에는 안전한 곳으로 안내하는 것처럼 상황에 맞는 UX를 제공하는 것이 중요하다는 것을 느낄 수 있었습니다.

<br/>

# 아쉬운 점

현재는 재시도 버튼만 고정으로 제공하는 수준이지만, 더 고차원적인 복구 전략이 필요할 것 같습니다.

또한 개발 환경과 프로덕션 환경의 구분이 없어서, 모든 환경에서 동일한 에러 메세지를 보여주지만, 개발 환경에서는 상세한 스택 트레이스를 보고 싶을 수 있습니다. 따라서 이를 구분하는 작업이 필요할 것 같습니다.

<br/>

# 마무리

완벽한 서비스는 에러가 발생하지 않는 서비스가 아니라, **에러가 발생해도 사용자가 당황하지 않고 다음 행동을 취할 수 있는 서비스**입니다.

이번 에러 핸들링 시스템 구축을 통해 **커피빵**은 더욱 안정적인 서비스가 되었고, 에러를 추적하고 개선할 수 있는 환경을 마련했습니다. 무엇보다 **사용자가 신뢰할 수 있는 서비스**를 만드는 데 한 걸음 더 나아갔다는 점에서 의미가 있었습니다.

에러는 피할 수 없지만, 어떻게 대응하느냐는 우리가 선택할 수 있습니다. 여러분의 프로젝트에서도 체계적인 에러 핸들링을 통해 더 나은 사용자 경험을 제공하시길 바랍니다.
