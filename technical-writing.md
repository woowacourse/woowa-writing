# 예측 불가능성에 대비하기: 에러 핸들링 A to Z

![image1](./images/image1.png)

## 0. 서문

### 'Happy Path'를 넘어, 실패에 대비하기

프론트엔드 개발은 흔히 ‘Happy Path’, 즉 모든 것이 계획대로 작동하는 경로를 따라 시작됩니다. 하지만 이 경로를 벗어났을 때, 즉 예측 불가능한 실패와 마주했을 때, 서비스의 완성도가 결정되곤 합니다.

에러 핸들링은 단순히 버그를 잡는 보조 작업이 아닙니다. 사용자의 신뢰, 개발 생산성, 그리고 서비스의 안정성을 좌우하는 핵심 설계 요소입니다.
사용자가 마주하는 깨진 화면, 개발자가 보는 불명확한 로그는 모두 “에러를 다루는 방식”의 결과입니다.

### 체계적인 에러 핸들링을 위한 로드맵

이 글은 프론트엔드 에러 핸들링을 체계적으로 접근하고자 하는 개발자들을 위한 포괄적인 안내서입니다. 에러와 예외의 근본적인 개념을 명확히 하는 것부터 시작하여, 다양한 유형의 에러를 분류하고 각 상황에 맞는 방안을 소개하고자 합니다.
나아가 React와 같은 프레임워크에서 선언적이고 우아하게 에러를 다루는 방법에 이르기까지, 에러 핸들링의 전 과정을 다룰 예정입니다.

이 글은 프론트엔드 에러 핸들링을 ‘운에 맡기지 않는 기술’로 다루기 위한 가이드입니다. 에러와 예외의 개념부터 시작해, 유형별 처리 전략, React에서의 선언적 에러 처리까지 단계적으로 살펴봅니다. 즉, 단순히 “에러가 나면 어떻게 할까”가 아니라, “에러가 나도 흔들리지 않는 시스템을 어떻게 설계할까”에 대해 이야기해보고자 합니다.

![image2](./images/image2.png)

## 1. 기본 다지기: 에러(Error)와 예외(Exception)

'에러'와 '예외', 두 용어는 종종 혼용되지만, JavaScript의 동작 방식 내에서 미묘하지만 중요한 차이를 가집니다.

### 에러(Error)와 예외(Exception)의 관계

JavaScript에서 에러(Error) 는 런타임에 발생하는 문제를 나타내는 객체입니다. 이 `Error` 객체는 일반적으로 세 가지 핵심 프로퍼티를 가집니다.

- `name`:

  에러의 유형을 나타내는 이름 (예: 'TypeError')

- `message`:

  개발자가 제공하거나 엔진이 생성한 에러에 대한 설명

- `stack`:

  에러가 발생한 지점까지의 함수 호출 스택을 보여주는 문자열. 디버깅에 있어 가장 중요한 정보입니다.

반면, 예외(Exception) 는 코드의 정상적인 실행 흐름을 중단시키는 이벤트 또는 행위를 의미합니다. 이 예외는 `throw` 키워드를 사용하여 발생시킬 수 있습니다. `throw` 문 뒤에는 어떤 값이든 올 수 있습니다. 예를 들어, 문자열이나 숫자도 `throw` 할 수 있습니다.

```js
try {
  // throw "문제가 발생했습니다!"; // 문자열을 예외로 던질 수 있음
  throw new Error('문제가 발생했습니다!'); // Error 객체를 던지는 것이 모범 사례
} catch (e) {
  console.log(e.message); // "문제가 발생했습니다!"
  console.log(e.stack); // 문자열을 던지면 stack 프로퍼티가 없어 undefined가 될 수 있음
}
```

여기서 핵심은, `throw`의 대상으로 `Error` 객체를 사용하는 것이 보편적인 모범 사례라는 점입니다. 왜냐하면 `Error` 객체만이 문제의 근원을 추적하는 데 필수적인 `stack` 정보를 담고 있기 때문입니다. 따라서 "예외를 던진다"는 말은 실질적으로 "`Error` 객체를 `throw` 한다"는 의미로 이해할 수 있습니다.

### 예외 처리의 기본 문법 : `try...catch...finally`

JavaScript는 발생한 예외를 처리하기 위한 구조로 `try...catch...finally` 문을 제공합니다.  

- `try` 블록:

  예외가 발생할 가능성이 있는 코드를 감싸는 구간입니다. 이 블록 안의 코드가 실행되다가 예외가 발생하면, 즉시 실행을 멈추고 제어 흐름이 `catch` 블록으로 넘어갑니다.

- `catch (e)` 블록:

  `try` 블록에서 예외가 발생했을 때만 실행됩니다. 매개변수 `e`는 `throw` 된 값(주로 `Error` 객체)을 받습니다. 이 블록에서는 에러를 기록하거나, 사용자에게 알림을 표시하거나, 대체 동작을 수행하는 등의 에러 처리 로직을 구현합니다.

- `finally` 블록:

  `try` 블록의 성공 여부나 `catch` 블록의 실행 여부와 관계없이 항상 실행되는 코드 블록입니다. 주로 파일 핸들이나 네트워크 연결을 닫는 등 리소스 정리 작업에 사용되어 코드의 안정성을 높입니다.

### JavaScript의 내장 에러 유형

JavaScript 엔진은 특정 상황에 맞는 여러 종류의 내장 `Error` 생성자를 제공합니다. 이러한 내장 에러 타입을 이해하면 `catch` 블록에서 보다 구체적인 조건으로 에러를 처리할 수 있습니다.

- `TypeError`:

  변수나 매개변수가 예상된 타입이 아닐 때 발생합니다. (예: null.toUpperCase())

- `ReferenceError`:

  존재하지 않는 변수를 참조할 때 발생합니다. (예: 선언되지 않은 변수 사용)

- `SyntaxError`:

  문법적으로 유효하지 않은 코드를 실행하려고 할 때 발생합니다. (주로 코드 파싱 단계에서 잡히지만, eval() 사용 시 런타임에 발생 가능)

- `RangeError`:

  값이 허용된 범위를 벗어났을 때 발생합니다. (예: new Array(-1))

- `URIError`:

  encodeURI()나 decodeURI() 함수에 잘못된 인자가 전달되었을 때 발생합니다.

`catch` 블록 내에서 `instanceof` 연산자를 사용하면 발생한 에러의 종류를 식별하여 각각 다른 방식으로 대응할 수 있습니다.

```js
try {
  // 어떤 작업을 수행
  null.method();
} catch (e) {
  if (e instanceof TypeError) {
    console.error('타입 에러가 발생했습니다:', e.message);
  } else if (e instanceof ReferenceError) {
    console.error('참조 에러가 발생했습니다:', e.message);
  } else {
    console.error('알 수 없는 에러:', e);
  }
}
```

### 더 나은 설계를 위한 Custom Error

내장 에러만으로는 애플리케이션의 복잡한 도메인 특화적인 실패 상황을 표현하기에 부족합니다. 예를 들어, 'API 통신 실패'나 '사용자 입력값 검증 실패'와 같은 에러는 JavaScript 내장 타입으로 표현할 수 없습니다. 이때 `Error` 클래스를 상속받아 Custom Error를 만드는 패턴을 사용할 수 있습니다.

Custom Error를 만들면 에러 객체에 `statusCode`, `errorCode`와 같은 추가적인 컨텍스트 정보를 담을 수 있습니다. 이는 에러 처리 로직을 훨씬 더 깔끔하고 명확하게 만들어 줍니다. 예를 들어, API 요청 실패 시 `RequestError`라는 커스텀 에러를 생성하여, 에러를 어떻게 UI에 표시할지(`toast` 또는 `fallback`)에 대한 정보를 담아 처리하는 정교한 패턴을 만들 수 있습니다.

```js
// Custom Error 정의
class NetworkError extends Error {
  constructor(message, statusCode) {
    super(message);
    this.name = 'NetworkError';
    this.statusCode = statusCode;
  }
}

// Custom Error 사용
async function fetchData() {
  const response = await fetch('/api/data');
  if (!response.ok) {
    throw new NetworkError('데이터를 가져오는 데 실패했습니다.', response.status);
  }
  return response.json();
}

// Custom Error 처리
try {
  await fetchData();
} catch (e) {
  if (e instanceof NetworkError) {
    console.error(`네트워크 에러 발생 (상태 코드: ${e.statusCode}): ${e.message}`);
    // 상태 코드에 따라 다른 UI를 보여줄 수 있음
  } else {
    console.error('알 수 없는 에러:', e);
  }
}
```

이처럼 일반적인 `catch (e)` 블록에서 에러 메시지 문자열을 파싱하여 분기 처리하는 방식은 매우 불안정하고 유지보수가 어렵습니다. 반면, Custom Error와 `instanceof`를 활용하는 것은 애플리케이션 내에서 에러가 어떻게 소통되어야 하는지에 대한 명확한 약속을 하는 것과 같습니다. 이는 이후에 다룰 모든 고급 에러 처리 전략의 기초가 되는 설계 원칙입니다.

## 2. 프론트엔드 에러의 유형별 분류와 접근법

에러는 발생 지점에 따라 성격이 다르며, 그에 따라 처리 전략도 달라집니다.  
프론트엔드에서는 보통 다음 세 가지로 구분할 수 있습니다.

| 구분           | 주요 원인              | 처리 핵심                 |
| -------------- | ---------------------- | ------------------------- |
| JS 런타임 에러 | 코드 결함              | 앱 중단 방지 + 로그 전송  |
| UI 렌더링 에러 | 컴포넌트/상태 불일치   | 에러 격리 (ErrorBoundary) |
| 네트워크 에러  | 외부 시스템, 통신 실패 | 재시도 및 복구 전략       |

![image3](./images/image3.png)

### 2-1. JavaScript 런타임 에러

이 유형의 에러는 브라우저의 JavaScript 엔진이 코드를 실행하는 도중에 발생하는 문제들입니다. 대부분 개발자의 코드에 논리적인 결함이 있을 때 발생하며, 순수 JavaScript 문법과 규칙에 기반합니다.

- `TypeError`:

  예상치 못한 타입의 값에 대해 연산을 시도할 때 발생합니다. 가장 흔한 예는 `null`이나 `undefined`인 값의 프로퍼티에 접근하려는 경우입니다. 예를 들어, API 응답이 비어있을 가능성을 고려하지 않고 `response.data.name`에 접근하면 `TypeError: Cannot read properties of null (reading 'name')`가 발생할 수 있습니다.

- `ReferenceError`:

  선언되지 않은 변수를 참조하거나, 스코프 밖의 변수에 접근하려 할 때 발생합니다. 변수명의 오타나 잘못된 스코프 이해가 주된 원인입니다.

- `SyntaxError`:

  엄밀히 말해 코드가 실행되기 전, 파싱 단계에서 잡히는 문법 오류입니다. 하지만 eval() 함수나 동적으로 스크립트를 로드하는 경우 런타임에도 발생할 수 있습니다.

- `RangeError`:

  숫자 값이 허용된 범위를 벗어났을 때 발생합니다. 예를 들어, 배열의 길이를 음수로 설정하려고 시도하는 경우입니다 (`new Array(-1)`).  

이러한 런타임 에러는 대부분 코드의 결함을 의미합니다. 따라서 사용자 경험 보호(에러가 전체 앱을 멈추지 않게)와 개발자 피드백(정확한 로그 수집)의 두 축을 중심으로 대응 전략을 세워야 합니다.

### 2-2. UI 렌더링 에러

이 유형의 에러는 React, Vue, Svelte 등 프레임워크의 렌더링 사이클이나 상태 관리 흐름에서 발생하는 문제입니다. 대부분은 상태(state)와 UI의 불일치 혹은 컴포넌트 생명주기 규칙 위반에서 비롯됩니다.

- 렌더 함수 내부의 에러:

  React 컴포넌트의 렌더링 중 TypeError와 같은 일반적인 JavaScript 에러가 발생하면, React는 해당 컴포넌트 트리의 렌더링을 중단합니다. 이 에러를 처리하지 않으면 전체 애플리케이션이 흰 화면으로 멈춰버리는 현상이 발생할 수 있습니다.

  ```js
  function Profile({ user }) {
    // user가 null인 경우 → TypeError
    return <p>{user.name}</p>;
  }
  ```

- 잘못된 상태 업데이트:

  useEffect 안에서 의존성 배열 없이 setState를 호출해 무한 루프를 유발하거나, 언마운트된 컴포넌트에서 상태를 업데이트하려는 경우가 대표적입니다.

- 프레임워크 규칙 위반:

  React의 “Hook은 조건문 안에서 호출하지 않는다”와 같은 규칙을 어길 때 발생하는 에러입니다.

렌더링 에러는 UI의 한 구역에서 시작되지만, 그대로 두면 애플리케이션 전체로 전파됩니다. 따라서 핵심 전략은 “에러를 격리하고 나머지 영역을 보호하는 것”입니다. 다시 말해, 오염 전파 차단(Isolation)라고 말할 수 있습니다.

React의 ErrorBoundary는 이러한 격리를 위해 만들어진 장치입니다. 문제가 발생한 컴포넌트를 격리하고, 대신 미리 준비된 Fallback UI를 보여줍니다. 이로써 사용자는 앱 전체가 멈추지 않고 “일부 기능만 잠시 사용할 수 없음” 정도로 인식하게 됩니다.

```jsx
<ErrorBoundary fallback={<p>위젯을 불러오는 중 문제가 발생했습니다.</p>}>
  <Widget />
</ErrorBoundary>
```

### 2-3. 네트워크 및 API 통신 에러

프론트엔드 애플리케이션은 대부분 외부 API와의 통신에 의존합니다. 이 통신 과정에서 발생하는 에러는 애플리케이션 외부의 요인에 의해 발생하는 경우가 많아 가장 예측하기 어렵고 다루기 까다로운 유형입니다.

- HTTP 에러 상태 코드:

  - 서버는 클라이언트의 요청에 대한 처리 결과를 HTTP 상태 코드로 응답합니다. 서버의 응답을 단순히 “에러”로만 처리하기보다는, 상태 코드별로 의미를 구분해 다르게 대응해야 합니다.
  - 4xx: 클라이언트 요청 문제 (예: 401 Unauthorized, 404 Not Found)
  - 5xx: 서버 문제 (예: 500 Internal Server Error, 503 Service Unavailable)

- 네트워크 연결 문제:

  사용자의 인터넷 연결이 끊기거나 불안정할 때, 또는 DNS 조회에 실패하는 경우 요청 자체가 서버에 도달하지 못하고 실패할 수 있습니다.

- CORS (Cross-Origin Resource Sharing) 에러:

  보안상의 이유로 브라우저가 다른 출처(origin)로의 요청을 차단할 때 발생합니다.

- 타임아웃:

  서버가 정해진 시간 내에 응답하지 않을 때 발생하는 에러입니다.

네트워크 에러는 일시적인 문제일 가능성이 높습니다. 예를 들어, 서버가 잠시 과부하 상태이거나 사용자의 네트워크가 순간적으로 불안정했을 수 있습니다. 따라서 이 유형의 에러 처리 전략은 단순히 실패를 알리는 것을 넘어 회복탄력성(Resilience)을 구축하는 데 초점을 맞춰야 합니다. 실패한 요청을 자동으로 재시도하거나, 서버 상태를 확인한 후 요청을 보내는 등의 전략이 필요합니다.

### 에러 유형에 따라 다른 대응 전략

이처럼 에러의 발생 위치와 성격에 따라 필요한 대응 전략은 판이하게 달라집니다. JavaScript 런타임 에러는 개발자가 수정해야 할 버그 신호이며, UI 렌더링 에러는 격리가 필요한 UI 오염이고, 네트워크 에러는 시스템이 스스로 회복을 시도해야 하는 외부 요인입니다. 이 분류 체계를 이해하고 각 에러에 맞는 맞춤형 전략을 적용하는 것이 견고한 에러 핸들링의 첫걸음입니다.

## 3. 통신 계층: API 요청 에러 핸들링 대응 전략

프론트엔드 애플리케이션에서 가장 빈번하고 예측 불가능한 에러의 원천은 바로 서버와의 통신 계층입니다. 사용자의 네트워크 상태, 서버의 안정성, API 응답 형식 등 수많은 변수가 존재하기 때문입니다. 따라서 이 통신 계층에 견고한 방어선을 구축하는 것은 서비스 안정성의 핵심입니다. 이번 글에서는 `axios` 라이브러리를 중심으로, 반복적인 코드를 줄이고 시스템의 회복탄력성을 높이는 고급 API 에러 핸들링 전략을 자세하게 다루고자합니다.

### Interceptor를 활용한 중앙 집중식 에러 처리

모든 API 요청 함수마다 `try...catch` 블록이나 `.catch()` 체인을 추가하여 에러를 처리하는 방식은 심각한 문제를 낳습니다.

- 코드 중복:

  에러 로깅, 사용자 알림, 특정 상태 코드(예: 401 Unauthorized)에 대한 공통 처리 로직이 모든 요청 함수에 흩어져 유지보수를 어렵게 만듭니다.

- 일관성 부재:

  개발자마다 에러를 처리하는 방식이 달라져 사용자 경험의 일관성이 깨지고, 잠재적인 버그를 유발할 수 있습니다.

이 문제의 해결책은 `axios`의 인터셉터(Interceptors)를 사용하는 것입니다. 인터셉터는 모든 요청이 서버로 전송되기 전(`request interceptor`)이나, 모든 응답이 애플리케이션에 도달하기 전(`response interceptor`)에 가로채서 공통 로직을 수행할 수 있는 강력한 미들웨어입니다. 특히 `response interceptor`는 API 에러를 한 곳에서 중앙 집중적으로 관리할 수 있습니다.  

```js
import axios from 'axios';

const apiClient = axios.create({
  baseURL: 'https://api.example.com',
  timeout: 5000,
});

// 응답 인터셉터 설정
apiClient.interceptors.response.use(
  (response) => {
    // 2xx 범위의 상태 코드에 대한 성공 처리
    return response;
  },
  async (error) => {
    // 2xx 범위를 벗어나는 상태 코드에 대한 실패 처리
    const { config, response } = error;

    // response가 없는 네트워크 에러 등의 경우
    if (!response) {
      // 네트워크 연결 오류 메시지 표시
      console.error('Network Error:', error.message);
      return Promise.reject(error);
    }

    const { status } = response;

    if (status === 401) {
      // 401 Unauthorized: 토큰 재발급 로직
      try {
        const newAccessToken = await refreshAuthToken();
        config.headers.Authorization = `Bearer ${newAccessToken}`;
        // 원래 요청을 새로운 토큰으로 재시도
        return axios(config);
      } catch (refreshError) {
        // 재발급 실패 시 로그아웃 처리
        console.error('Token refresh failed', refreshError);
        // logout();
        return Promise.reject(refreshError);
      }
    }

    if (status >= 500) {
      // 5xx 서버 에러: 사용자에게 일반적인 서버 오류 메시지 표시
      console.error('Server Error:', status);
    }

    // 처리된 에러는 애플리케이션의 다른 부분으로 전파하지 않고,
    // 여기서 Promise.reject를 통해 통일된 에러 객체를 전달
    return Promise.reject(error);
  },
);

export default apiClient;
```

이처럼 인터셉터를 사용하면 인증 토큰 재발급 , 특정 에러 코드에 대한 전역 알림, 서버 응답 에러를 일관된 형식의 Custom Error 객체로 변환하는 등의 로직을 단 한 곳에서 관리할 수 있습니다. 이는 코드의 재사용성을 높이고 애플리케이션의 예측 가능성을 크게 향상시킵니다.

### 자동 재시도를 통한 회복탄력성 확보

"네트워크 오류가 발생했습니다. 다시 시도해주세요."라는 메시지를 사용자에게 보여주는 것은 책임을 사용자에게 떠넘기는 것입니다. 많은 네트워크 관련 에러는 일시적입니다. 서버가 순간적으로 바쁘거나, 사용자의 Wi-Fi 연결이 잠시 끊겼을 수 있습니다. 이런 경우, 애플리케이션이 자동으로 요청을 몇 번 더 재시도한다면 사용자는 아무런 문제도 인지하지 못한 채 서비스를 계속 이용할 수 있습니다. 이것이 바로 회복탄력성(Resilience)의 핵심입니다.

이러한 자동 재시도 로직을 직접 구현하는 것은 복잡할 수 있지만, `axios-retry`와 같은 라이브러리를 사용하면 손쉽게 구현할 수 있습니다.

효과적인 API 에러 핸들링은 단순히 실패를 감지하는 것을 넘어, 시스템이 스스로 회복할 수 있는 능력을 갖추도록 설계하는 것입니다. 인터셉터를 통한 중앙 집중식 관리와 `axios-retry`를 활용한 재시도 전략은, 예측 불가능한 통신 환경 속에서도 사용자에게 안정적이고 끊김 없는 경험을 제공하는 견고한 프론트엔드 애플리케이션의 핵심 요소입니다.

## 4. 선언적으로 에러 다루기: React에서의 우아한 에러 처리

명령형 프로그래밍의 `try/catch` 구문은 여전히 유효하고 강력한 도구지만, 컴포넌트 기반의 선언적 UI 패러다임을 지향하는 React에서는 더 우아하고 직관적인 에러 처리 방식이 필요합니다. React는 렌더링 과정에서 발생하는 에러를 컴포넌트 레벨에서 선언적으로 처리할 수 있는 `ErrorBoundary`와, 비동기 데이터 로딩 상태를 관리하는 `Suspense`를 도입하여 에러 및 로딩 상태 관리했습니다. 이 두 가지를 조합하면, 복잡한 조건부 렌더링 로직을 컴포넌트에서 분리하여 코드의 가독성과 유지보수성을 향상시킬 수 있습니다.

### `ErrorBoundary`로 렌더링 에러 격리하기

ErrorBoundary는 자신의 자식 컴포넌트 트리에서 발생하는 JavaScript 에러를 잡아내고, 에러가 발생했을 때 UI가 완전히 깨지는 대신 미리 준비된 대체 UI(Fallback UI)를 보여주는 React 컴포넌트입니다. 이는 마치 컴포넌트를 위한 `try/catch` 블록처럼 동작합니다.

클래스 컴포넌트가 `static getDerivedStateFromError()` 또는 `componentDidCatch()` 생명주기 메서드 중 하나 이상을 정의하면 에러 경계가 됩니다.

- `static getDerivedStateFromError(error)`:

  에러가 발생한 후 폴백 UI를 렌더링하기 위해 사용됩니다. 이 메서드는 state를 업데이트하는 객체를 반환해야 합니다.

- `componentDidCatch(error, errorInfo)`:

  발생한 에러와 에러 정보를 로깅 서비스(예: Sentry)에 전송하는 등의 부수 효과(side effect)를 수행하기 위해 사용됩니다.

```js
import React from 'react';

class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false };
  }

  static getDerivedStateFromError(error) {
    // 다음 렌더링에서 폴백 UI가 보이도록 state를 업데이트합니다.
    return { hasError: true };
  }

  componentDidCatch(error, errorInfo) {
    // 에러 리포팅 서비스에 에러를 기록합니다.
    logErrorToMyService(error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      // 직접 폴백 UI를 렌더링할 수 있습니다.
      return <h1>Something went wrong.</h1>;
    }

    return this.props.children;
  }
}

// 사용 예시
<ErrorBoundary>
  <MyWidget />
</ErrorBoundary>;
```

`ErrorBoundary`의 가장 큰 장점은 에러의 영향 범위를 국소화하는 것입니다. `MyWidget` 컴포넌트에서 렌더링 에러가 발생하더라도, 애플리케이션 전체가 멈추는 대신 `ErrorBoundary`가 이를 감지하고 대체 UI를 보여줌으로써 나머지 부분은 정상적으로 작동하도록 유지합니다.

### `ErrorBoundary`의 한계

`ErrorBoundary`는 매우 강력하지만 만능은 아닙니다. 이것이 처리할 수 없는 에러의 종류를 명확히 이해하는 것이 중요합니다.  

- 이벤트 핸들러:

  이벤트 핸들러 내부의 코드는 React의 렌더링 단계에서 실행되지 않습니다. 따라서 여기서 발생하는 에러는 `ErrorBoundary`가 감지할 수 없습니다. 이벤트 핸들러 내의 에러는 전통적인 `try/catch` 문을 사용해야 합니다.

- 비동기 코드:

  `setTimeout`, `requestAnimationFrame` 콜백이나 Promise의 `.then()` 블록 내부에서 발생하는 에러 또한 렌더링 주기와는 무관하게 발생하므로 ErrorBoundary의 감지 범위를 벗어납니다.

- 서버 사이드 렌더링(SSR):

  서버 환경에서 렌더링하는 동안 발생하는 에러는 `ErrorBoundary`로 처리할 수 없습니다.

- `ErrorBoundary` 자신에게서 발생하는 에러:

  `ErrorBoundary` 컴포넌트 자체의 렌더링 로직에서 에러가 발생하면, 그 에러는 자신의 catch 로직이 아닌, 상위의 다른 `ErrorBoundary`로 전파됩니다.

따라서 실용적인 접근법은 하이브리드 모델을 채택하는 것입니다. 렌더링과 관련된 에러는 `ErrorBoundary`에 위임하고, 이벤트 핸들러나 비동기 로직 내에서는 try/catch를 사용하여 명시적으로 에러를 처리해야 합니다.

```js
function MyComponent() {
  const = useState(null);

  const handleClick = async () => {
    try {
      const result = await fetchData();
      setData(result);
    } catch (error) {
      // 비동기 에러는 try/catch로 직접 처리
      console.error("Failed to fetch data:", error);
      // 사용자에게 에러 상태를 알리는 state 업데이트
    }
  };

  return (
    <div>
      <button onClick={handleClick}>Fetch Data</button>
      {/* data가 잘못된 형식일 경우 렌더링 에러가 발생하며,
          이는 상위의 ErrorBoundary가 처리 */}
      <p>{data.name}</p>
    </div>
  );
}
```

### `Suspense`와 `ErrorBoundary`의 조합

`Suspense`는 컴포넌트가 아직 렌더링될 준비가 되지 않았을 때(예: 코드 스플리팅으로 컴포넌트 코드를 로딩 중이거나, 데이터를 비동기적으로 가져오는 중일 때) 로딩 상태를 선언적으로 관리할 수 있게 해주는 기능입니다.

`React Query`나 `SWR`과 같은 최신 데이터 페칭 라이브러리들은 `Suspense` 모드를 지원합니다. 이 모드를 활성화하면, 데이터 페칭이 진행 중일 때 컴포넌트는 렌더링을 '일시 중단(suspend)'하고, React는 가장 가까운 상위 <Suspense> 컴포넌트의 fallback prop을 렌더링합니다.  

여기서 중요한 점은, 만약 `Suspense` 모드를 사용하는 데이터 페칭 `Promise`가 성공적으로 `resolve`되지 않고 `reject`된다면, React는 이를 에러로 취급하여 throw합니다. 렌더링 중에 발생한 이 throw는 가장 가까운 상위 ErrorBoundary에 의해 잡히게 됩니다.  

이 패턴은 선언적 상태 관리의 핵심입니다.

```js
import { Suspense } from 'react';
import { ErrorBoundary } from 'react-error-boundary';
import { useSuspenseQuery } from '@tanstack/react-query';

// 데이터를 가져오는 컴포넌트
function UserProfile({ userId }) {
  // useSuspenseQuery는 데이터 로딩 중에는 컴포넌트를 일시 중단시키고,
  // 에러 발생 시에는 에러를 throw합니다.
  const { data } = useSuspenseQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
  });

  // 이 컴포넌트는 오직 '성공' 상태만 신경 쓰면 됩니다.
  return (
    <div>
      <h1>{data.name}</h1>
      <p>{data.email}</p>
    </div>
  );
}

// 컴포넌트를 사용하는 곳
function App() {
  return (
    <div>
      <h1>My Application</h1>
      <ErrorBoundary
        FallbackComponent={({ error }) => (
          <div>
            <h2>Oops, something went wrong.</h2>
            <p>{error.message}</p>
          </div>
        )}
      >
        <Suspense fallback={<h2>Loading profile...</h2>}>
          <UserProfile userId={123} />
        </Suspense>
      </ErrorBoundary>
    </div>
  );
}
```

위 예시에서 `UserProfile` 컴포넌트는 `isLoading`, `isError`, `error`와 같은 상태를 전혀 관리하지 않습니다. 오직 데이터를 성공적으로 가져왔을 때의 UI 렌더링에만 집중합니다. 로딩 상태는 상위의 `<Suspense>`가, 에러 상태는 상위의` <ErrorBoundary>`가 각각 책임지고 처리합니다.

이처럼 `Suspense`와 `ErrorBoundary`의 조합은 데이터 페칭과 관련된 상태 관리 로직을 컴포넌트로부터 완전히 분리하여, '관심사의 분리(Separation of Concerns)' 원칙을 구현합니다. 이는 컴포넌트를 훨씬 더 단순하고, 예측 가능하며, 재사용하기 쉽게 만들어 줄 수 있습니다.

## 5. 에러 처리와 사용자 경험(UX)의 상관관계

지금까지 우리는 에러를 기술적으로 어떻게 감지하고, 분류하며, 시스템 내부에서 처리할 것인지에 초점을 맞추었습니다. 하지만 이 모든 노력은 최종적으로 사용자에게 어떻게 전달되는지에 따라 그 가치가 결정됩니다. 기술적으로 완벽하게 처리된 에러라 할지라도, 사용자에게 혼란스럽고 불친절한 방식으로 표현된다면 그것은 실패한 에러 핸들링입니다. 에러 처리는 단순한 기술적 과제를 넘어, 사용자와의 신뢰를 구축하는 중요한 소통 과정이자 제품 디자인의 일부입니다.

### 절대 원시 에러(Raw Error)를 노출하지 마라

사용자에게 `TypeError: Cannot read properties of undefined`와 같은 원시 에러 메시지나 스택 트레이스를 그대로 보여주는 것은 최악의 안티패턴입니다. 여기에는 두 가지 중요한 이유가 있습니다.

- 사용자 경험 저해:
  비개발자에게 이러한 메시지는 아무런 의미가 없는 외계어일 뿐입니다. 사용자는 자신이 무엇을 잘못했는지, 혹은 시스템에 무슨 일이 일어났는지 전혀 이해할 수 없으며, 이는 불편함으로 이어집니다.

- 보안 취약점 노출:

  에러 메시지나 스택 트레이스에는 시스템의 내부 구조, 사용된 라이브러리 버전, 데이터베이스 스키마 등 공격자에게 유용한 정보가 포함될 수 있습니다. 이러한 정보를 외부에 노출하는 것은 잠재적인 보안에 위험이 있습니다.

따라서 서버에서 전달된 에러 메시지라 할지라도, 프론트엔드는 이를 사용자가 이해할 수 있는 언어로 번역하고 정제하여 보여줄 최종적인 책임이 있습니다.

### 에러의 맥락에 맞는 UI/UX 전략

모든 에러를 동일한 방식으로 사용자에게 알리는 것은 효과적이지 않습니다. 에러의 심각성, 사용자의 현재 작업 흐름에 미치는 영향 등을 고려하여 상황에 맞는 UI를 제공해야 합니다.

- 토스트 / 스낵바 (Toast / Snackbar):

  사용자의 현재 작업을 방해하지 않는 비판적이지 않고 일시적인 에러에 적합합니다. 예를 들어, '자동 저장에 실패했습니다. 잠시 후 재시도합니다.'와 같은 알림은 사용자가 하던 일을 계속할 수 있도록 하면서 시스템의 상태를 알려줍니다.  

- 모달 (Modal):

  사용자의 명시적인 확인이나 행동이 필요한 중요한 에러에 사용됩니다. 예를 들어, '세션이 만료되었습니다. 다시 로그인해주세요.'와 같이 현재 작업을 더 이상 진행할 수 없고 사용자의 조치가 필요한 경우에 적합합니다. 모달은 사용자의 시선을 강제하여 문제를 확실히 인지시킵니다.  

- 인라인 메시지 (Inline Message):

  특정 입력 필드와 관련된 유효성 검사 에러에 가장 효과적입니다. '올바른 이메일 형식이 아닙니다.'와 같은 메시지를 해당 입력 필드 바로 아래에 보여줌으로써 사용자가 즉시 문제를 파악하고 수정할 수 있도록 돕습니다.

- 전체 페이지 폴백 (Full-Page Fallback):

  페이지 전체 또는 애플리케이션의 핵심 기능이 동작하지 않는 치명적인 에러에 사용됩니다. 이는 주로 최상위 `ErrorBoundary`가 렌더링하는 UI입니다. 이 화면에서는 단순히 '오류가 발생했습니다'라고 보여주는 대신, 친근한 일러스트와 함께 '홈으로 돌아가기'나 '새로고침'과 같은 명확한 다음 행동을 제시해야 합니다.

![image4](./images/image4.png)

### 행동을 유도하는 좋은 에러 메세지 작성 방법

좋은 에러 메시지는 사용자가 문제 해결의 행동을 유도하도록 만들어야 합니다. 효과적인 에러 메시지는 다음 세 가지 요소를 포함해야 합니다.

1. 무슨 일이 일어났는가 (What):

- 전문 용어를 피하고 쉽고 간결한 언어로 문제 상황을 설명합니다. (예: "로그인에 실패했습니다.")

2. 왜 일어났는가 (Why):

- 가능하다면 문제의 원인을 간략하게 설명하여 사용자의 이해를 돕습니다. (예: "비밀번호가 일치하지 않습니다.")

3. 무엇을 해야 하는가 (What to do next):

- 사용자에게 다음에 취할 수 있는 구체적인 행동을 안내합니다. (예: "비밀번호를 다시 확인하시거나 '비밀번호 찾기'를 이용해주세요.")

더 나아가, 시스템이 스스로 해결할 수 없는 문제(예: 500 서버 에러)의 경우, 사용자에게 문제 해결을 돕도록 요청할 수도 있습니다. "문제가 계속되면 고객센터로 문의해주세요. (에러 코드: a1b2-c3d4)"와 같이 고유한 상관관계 ID(Correlation ID)를 제공하면, 사용자는 자신이 겪는 문제를 정확하게 전달할 수 있고, 지원팀과 개발자는 이 ID를 통해 서버 로그를 신속하게 추적하여 문제의 근본 원인을 파악할 수 있습니다.  

결론적으로, 프론트엔드 개발자는 기술과 디자인의 교차점에서 사용자와 소통하는 중요한 역할을 담당합니다. 에러 핸들링을 단순히 예외를 잡는 기술적 행위로만 보지 않고, 사용자의 감정을 고려하고 명확한 가이드를 제공하는 제품 디자인의 과정으로 접근할 때, 실패 상황에서조차 긍정적인 사용자 경험을 만들어낼 수 있습니다.

## 7. 마무리: 장애는 당연합니다, 중요한 것은 회복 탄력성!

해당 글을 통해 프론트엔드 에러 핸들링이 단순히 try/catch 구문을 추가하는 행위를 넘어, 사용자 경험, 시스템 아키텍처, 그리고 운영 철학까지 아우르는 깊이 있는 분야임을 알 수 있었습니다. 소프트웨어 개발에서 장애는 피할 수 없는 현실입니다. 완벽하게 에러 없는 코드를 작성하겠다는 목표는 비현실적일 뿐만 아니라, 오히려 변화와 혁신을 저해할 수 있습니다. 진정한 목표는 에러가 발생하지 않는 시스템이 아니라, 에러가 발생하더라도 우아하게 실패하고(fail gracefully), 신속하게 복구하며(recover quickly), 그 과정에서 명확한 교훈을 얻어 더 나은 방향으로 진화하는 회복탄력성(Resilience) 있는 시스템을 구축하는 것입니다.

이 글에서 살펴본 회복탄력성있는 서비스를 구축하기 위한 핵심 원칙은 아래와 같습니다.

- 에러 분류

  모든 에러는 동등하지 않습니다. JavaScript 런타임 에러, UI 렌더링 에러, 네트워크 통신 에러로 나누어 각각의 성격에 맞는 최적의 대응 전략을 적용해야 합니다.

- 선언적으로 다루기

  React의 `ErrorBoundary`와 `Suspense`를 조합하여 로딩과 에러 상태를 UI 로직에서 분리함으로써, 더 깨끗하고 예측 가능한 컴포넌트를 만들 수 있습니다.

- 사용자 우선

  기술적인 에러를 사용자가 이해할 수 있는 언어로 번역하고, 명확하고 행동 가능한 피드백을 제공하여 실패의 순간마저 긍정적인 경험으로 전환해야 합니다.

- 모든 것을 관찰하기

  운영 환경은 미지의 영역입니다. Sentry와 같은 모니터링 도구, 소스맵, 그리고 구조화된 로깅을 통해 데이터를 기반으로 한 의사결정을 내려야 합니다.

에러 핸들링은 일회성 작업이 아닌, 지속적인 개선과 학습의 과정입니다. 새로운 기능이 추가될 때마다 새로운 실패 시나리오가 생겨나고, 우리는 그에 맞춰 방어 체계를 끊임없이 발전시켜야 합니다. 오늘 당장 여러분의 코드베이스에 흩어져 있는 catch 블록을 검토하고, 최상위 레벨에 ErrorBoundary를 추가하며, 빌드 파이프라인에 소스맵 업로드를 설정하는 작은 행동 하나하나가 모여 사용자가 신뢰하고 완성도 높은 서비스가 될 것입니다. 장애를 두려워하지 말고, 그것을 더 나은 제품을 만들기 위한 소중한 신호로 받아들이는 문화가 정착되기를 바랍니다.

## 참고

- [Error - JavaScript | MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Error)
- [Errors vs Exceptions in JS | stackoverflow](https://stackoverflow.com/questions/75636288/errors-vs-exceptions-in-js-are-there-exceptions-in-javascript)
- [Exception | MDN](https://developer.mozilla.org/en-US/docs/Glossary/Exception)
- [Error Boundaries](https://legacy.reactjs.org/docs/error-boundaries.html)
- [Axios - Handling Errors and Using Retry Mechanisms](https://www.joinplank.com/articles/axios-handling-errors-and-using-retry-mechanisms)
- [토스 SLASH21 프론트엔드 웹 서비스에서 우아하게 비동기 처리하기](https://velog.io/@devjeenie/%EC%9A%B0%EC%95%84%ED%95%98%EA%B2%8C-%EB%B9%84%EB%8F%99%EA%B8%B0-%EC%B2%98%EB%A6%AC%ED%95%98%EA%B8%B0)
- [프론트엔드 에러 핸들링 전략](https://blog.review-me.page/blog/fe-error-handleing)
- [백엔드 개발자의 웹 프론트엔드 개발기| 우아한형제들 기술 블로그](https://techblog.woowahan.com/2683/)
- [선언적으로 에러처리하기 | velog](https://velog.io/@mmmdo21/%EC%84%A0%EC%96%B8%EC%A0%81%EC%9C%BC%EB%A1%9C-%EC%97%90%EB%9F%AC%EC%B2%98%EB%A6%AC%ED%95%98%EA%B8%B0react-error-boundary)
