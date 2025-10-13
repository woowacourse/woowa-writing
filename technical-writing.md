# 기존 에러 처리의 방식의 문제점

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

따라서, 예상치 못한 에러 상황에서 사용자가 에러 코드를 그대로 보게 되는 상황이 실제로 종종 발생했습니다.

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

예를 들어 데이터를 받아오는 화면에서 에러가 발생한다면, 사용자들은 당연히 데이터를 다시 받아오고 싶을 것입니다.

**모든 `get` 요청에 대해 동일하다**고 생각하기 때문에, `get` 메서드를 통해 데이터를 패칭하는데 실패했다면, **지역적인 `Fallback UI`를 제공하여 데이터를 다시 받아올 수 있게끔 재시도**를 할 수 있도록 하고자 합니다.

**1-2) get 이외의 메서드에서 요청이 실패 한 경우**

이는 `get` 이외의 메서드 (`post`, `delete`, `patch`)의 요청이 실패한 경우입니다.

예를 들어, 대다수의 경우 특정 버튼을 눌렀을 때, `post` 요청이 되는데요.

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

# 에러 핸들링의 전체적인 구조

위에서 정의한 다양한 에러 케이스들을 어떻게 공통적으로 처리할 수 있을까 고민한 끝에, `ErrorBoundary`를 활용하여 통합적으로 핸들링하기로 결정했습니다.

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

### 1. API 요청 단계

**apiRequest 함수에서 에러 발생**

`API` 요청 과정에서 `apiRequest` 함수에서 발생하는 에러를 세분화하여 처리합니다.

에러 분류 기준은 아래와 같습니다.

- **API 에러**: 서버에서 반환된 에러 응답 → `ApiError` 객체
- **네트워크 에러**: `TypeError` 또는 "Failed to fetch" 포함 → `NetworkError` 객체
- **그 외**: 예상치 못한 에러 → `Error` 객체

여기서 `useFetch`는 실제 `api` 요청 함수인 `apiRequest`를 래핑하는 함수이며, 컴포넌트 단에서 `api`를 요청할 때 사용되는 커스텀 훅입니다.

`useFetch`를 호출하는 컴포넌트에서 핸들링하지 않기 때문에 자연스럽게 **상위 컴포넌트로 에러가 전파**됩니다.

### 2. ErrorBoundary에서 캐치

이렇게 분류된 에러는 두 계층의 `ErrorBoundary`에서 처리됩니다.

앞서 봤던 `MiniGameSection`에서 에러가 발생한다면, 각 조건에 맞게 `LocalErrorBoundary` 또는 `GlobalErrorBoundary`에서 처리하게 될 것입니다.

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
