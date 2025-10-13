# MSW + Storybook 현명하게 활용하기

## 개요

MSW(Mock Service Worker)는 Service Worker를 활용해 서버의 API를 모킹(mocking)할 수 있는 도구입니다. 즉, 백엔드 API가 아직 완성되지 않았더라도, 미리 응답 구조를 정의해 두고 프론트엔드 개발을 진행할 수 있게 도와줍니다. 이 자체로도 충분히 유용하지만, Storybook과 함께 사용하면 훨씬 강력한 시너지를 발휘합니다.

## MSW 세팅 톺아보기

### mockServiceWorker.js

MSW (Mock Service Worker) 라이브러리의 핵심 구성 요소 중 하나로, Service Worker 스크립트입니다. 이 파일은 브라우저에서 작동하며, 실제 네트워크 요청을 가로채고 가짜(mocked) 응답을 반환하는 역할을 합니다.

```bash
npx msw init public/ --save
```

환경에 따라 Browser과 Server 두 가지로 나누어서 세팅을 해야 합니다.

### Browser 설정

```javascript
// src/mocks/browser.js
import { setupWorker } from "msw/browser";
import { handlers } from "./handlers";

export const worker = setupWorker(...handlers);
```

**setupWorker**: 브라우저 환경에서 네트워크 요청을 가로채기 위한 함수

**(1) 인자를 넘기지 않을 경우**

- 빈 핸들러 리스트로 초기화
- `worker.use()`로 후에 동적으로 핸들러를 추가할 수 있음

**(2) 인자를 넘길 경우**

- 인자로 전달한 핸들러는 초기 핸들러로 등록
- Service Worker 등록은 비동기적으로 이루어지기 때문에, `worker.start()`를 호출할 때 Promise를 반환
- 호출부에서 꼭 해당 작업을 비동기로 처리해야 함

### Server 설정

```javascript
// src/mocks/server.js
import { setupServer } from "msw/node";
import { handlers } from "./handlers";

export const server = setupServer(...handlers);
```

#### 왜 server와 browser 환경을 따로 세팅 해줘야 하나요?

- **browser 환경**: Service Worker API를 이용해 실제 네트워크 요청을 가로채는 방식으로 동작 (브라우저의 네트워크 계층)
- **server 환경**: 브라우저가 없기 때문에 Service Worker를 사용할 수 없고, 대신 node 환경에서의 요청 함수를 직접 patching해서 모킹

즉, 각 실행환경의 네트워크 계층에 맞게 동작해야 하므로 나누어 세팅해야 합니다.

> **패칭(Patching)**: 기존 기능의 동작을 덮어씌워서 바꾸는 것 (가짜 함수로 교체)

### Handler 설정

```javascript
import { http, HttpResponse } from "msw";
import cartItems from "./cartItems.json";

const END_POINT = "*/cart-items";

export const handlers = [
  // 정상 응답
  http.get(END_POINT, async () => {
    return HttpResponse.json(cartItems);
  }),

  // 401 Unauthorized 에러
  http.get(`${END_POINT}/unauthorized`, async () => {
    return HttpResponse.json({ message: "Unauthorized" }, { status: 401 });
  }),

  // 403 Forbidden 에러
  http.get(`${END_POINT}/forbidden`, async () => {
    return HttpResponse.json({ message: "Forbidden" }, { status: 403 });
  }),

  // 404 Not Found 에러
  http.get(`${END_POINT}/not-found`, async () => {
    return HttpResponse.json({ message: "Not Found" }, { status: 404 });
  }),

  // 500 Server Error
  http.get(`${END_POINT}/server-error`, async () => {
    return HttpResponse.json(
      { message: "Internal Server Error" },
      { status: 500 }
    );
  }),

  // 네트워크 에러 시뮬레이션
  http.get(`${END_POINT}/network-error`, async () => {
    return HttpResponse.error();
  }),
];
```

CartPage에서 실제로 fetch 요청을 보낼 때, 다른 엔드포인트로 테스트해볼 수 있도록 handler에 명시적으로 path를 지정했습니다.

### main.tsx 설정

```javascript
import React from "react";
import ReactDOM from "react-dom/client";
import App from "./App";

async function enableMocking() {
  const { worker } = await import("./mocks/browser");

  return worker.start({
    onUnhandledRequest: "warn",
    serviceWorker: {
      url: "/mockServiceWorker.js",
    },
  });
}

enableMocking()
  .then(() => {
    const rootElement = document.getElementById("root");
    if (rootElement) {
      ReactDOM.createRoot(rootElement).render(
        <React.StrictMode>
          <App />
        </React.StrictMode>
      );
    }
  })
  .catch((error) => {
    console.error("Failed to initialize MSW:", error);
  });
```

#### onUnhandledRequest는 왜 필요한가요?

MSW에 정의되지 않은 네트워크 요청이 발생했을 때 어떻게 처리할지 설정하는 옵션입니다.

백엔드에서 API 개발이 진행되는 과정 중, 일부 API가 실제로 구현되면 기존에 사용하던 모킹(mocking) 코드를 제거하고, 실 API를 사용할 수 있도록 설정을 전환하게 됩니다.

**옵션 종류:**

- **'warn' (기본값)**: 콘솔에 경고 메시지를 출력 (예: `[MSW] Warning: captured a request …`)
- **'bypass'**: 경고 없이 요청을 그대로 원래 목적지로 전달 (실제 서버로 요청)
- **'error'**: 오류를 throw하여 테스트나 개발 환경에서 실패하도록 유도. 핸들러 누락을 빠르게 발견하는 데 유용

## MSW는 그래서 언제 쓰는 것일까요?

### API 모킹용

서버의 다양한 응답 코드에 대한 UI 테스트

### 💡 MSW 사용: 다양한 응답 코드에 대비

handler에서 특정 path로 향할 때 어떤 응답 코드를 반환할지 설정 (401, 404 등 다양한 !response.ok 상황을 가정)

```javascript
useEffect(() => {
  const loadCartItems = async () => {
    try {
      const result = await fetchCartItems(
        "http://localhost:5173",
        "/unauthorized"
      );
      setCartItems(result.content);
      setError(null);
    } catch (err) {
      if (err instanceof Error) {
        const statusMatch = err.message.match(/HTTP Error: (\d+)/);
        const status = statusMatch ? parseInt(statusMatch[1]) : undefined;
        // ...
      }
    }
  };
});
```

아래 코드처럼 `fetchCartItems` 할 때 path를 넘겨줌으로써(`"/unauthorized"`) 수동으로 테스트 가능합니다.

## MSW + Storybook

Storybook의 경우 컴포넌트를 독립적으로 개발하고 테스트할 수 있는 도구입니다. 문서화 용도로도 많이 사용되고 시각적 회귀(Visual Regression) 테스트에도 많이 활용되고 있습니다.

실제로 Storybook 공식 문서에서도 라이브러리를 지원해주며 함께 사용하는 예시가 나와있습니다.

🔗 [https://storybook.js.org/addons/msw-storybook-addon](https://storybook.js.org/addons/msw-storybook-addon)

### 주요 함수

**initialize()**

- MSW 초기화하는 함수
- 내부에서 `worker.start()` 해주는 역할

**mswLoader**

- 각 스토리마다 해당하는 핸들러를 불러올 수 있도록 도와주는 스토리 로더 함수
- 각 Story에서 `msw.handlers`를 설정 ⇒ 해당 Story가 렌더링될 때만 적용되는 핸들러로 MSW가 동작

### 각 스토리마다 msw 핸들러가 있다?

아래 예시처럼 Story마다 handler 직접 설정 가능합니다. 즉, 어떤 응답 코드에서 어떤 storybook 컴포넌트를 보여줘야 하는지 설정하는 역할을 합니다.

```typescript
// CartPage.stories.tsx

export const MockedSuccess: Story = {
  parameters: {
    msw: {
      handlers: [
        http.get(END_POINT, async () => {
          return HttpResponse.json(mockData);
        }),
      ],
    },
  },
};

export const UnauthorizedError: Story = {
  parameters: {
    msw: {
      handlers: [
        http.get(END_POINT, async () => {
          return HttpResponse.json(
            { message: "Unauthorized" },
            { status: 401 }
          );
        }),
      ],
    },
  },
};
```

## 결론

MSW를 사용하면 우리가 대응해야 하는 다양한 HTTP 에러 코드(401, 403, 404, 500 등)에 대해 UI 단에서 직접 에러 핸들링 로직을 테스트하고 시뮬레이션할 수 있습니다.

브라우저의 fetch 단계에서 이러한 에러를 인위적으로 발생시키는 것은 번거롭고 비효율적이기 때문에, MSW의 핸들러를 활용해 현실적인 테스트 환경을 구성하는 것이 훨씬 효과적입니다.

또한 단순한 수동 테스트를 넘어, MSW + Jest로 유닛 테스트를 수행하거나, Storybook에서 스토리마다 서로 다른 핸들러를 적용해 UI 컴포넌트별 상태 테스트도 가능합니다.

**즉, MSW는 프론트엔드 개발자가 백엔드 의존 없이 안정적인 UI를 설계하고 검증할 수 있게 해주는 강력한 도구입니다.**
