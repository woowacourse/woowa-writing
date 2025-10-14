# 문제 상황

유효하지 않은 토큰을 가진채로 토큰이 필요한 api를 요청시 401에러가 뜬다.

이때 리프레시 토큰을 가지고 토큰을 재발급해주는 `reissue` api 요청을 한다.

하지만 `reissue` api의 요청이 여러번 보내면 나중에 보내진 요청이 401이 뜬다.

![요청 오류나는 사진](image.png)

<details>
  <summary>apiClient 코드</summary>

```tsx
    async get<T>({
      endpoint,
      searchParams,
      withCredentials,
    }: ApiClientGetType): Promise<T> {
      const url = new URL(`${this.#baseUrl}${endpoint}`);
      url.search = new URLSearchParams(searchParams).toString();

      const options = {
        method: 'GET',
        headers: {
          accept: 'application/json',
          'Content-Type': 'application/json',
        },
        credentials: withCredentials
          ? 'include'
          : ('same-origin' as RequestCredentials),
      };

      const sendRequest = async () => {
        const response = await fetch(url, options);
        if (!response.ok) {
          const data = await response.json();
          throw new ApiError(data.message, response.status);
        }
        console.log('!성공!', endpoint);

        return response.json();
      };

      try {
        return await sendRequest();
      } catch (error) {
        if (error instanceof ApiError) {
          console.log('error.message', endpoint, error.message);
          console.log(error);
          if (error.status === 401 || error.status === 403) {
            try {
              console.log('#재발급#');
              await postReissue(); // 토큰 재발급
            } catch (error) {
              console.error('토큰 재발급 실패', error);
              // 재발급 실패 시 로그아웃 처리 등 추가 가능
              throw error;
            }

            try {
              return await sendRequest(); // 재발급 성공 후 재요청
            } catch (error) {
              console.error('재요청 실패', error);
              // 재발급 실패 시 로그아웃 처리 등 추가 가능
              throw error;
            }
          }
        }

        throw error;
      }
    }
```

</details>

<br />

# 문제 원인

문제의 원인은 토큰 재발급시 토큰뿐 아니라 리프레시 토큰도 함께 재발행되었기 때문이었다.

## 문제되는 상황 시나리오

1. 여러 요청이 동시에 401을 받음
2. 401을 받은 api의 catch문에서 `reissue` api 요청을 보낸다.

- 이때, n번의 요청이 가게 된다.
- n번의 요청 모두 똑같은 리프레시 토큰과 액세스 토큰을 가지고 요청을 보내게 된다.

3. n번의 요청 중 먼저 도착한 `reissue` 요청을 브라우저 쿠키에 저장된 토큰들과 함께 요청한다. 이 요청으로 인해 액세스 토큰이 재발급 되어서 브라우저의 쿠키에 저장된다.

- 이때 리프레시 토큰도 재발급 된다.
- 서버에 리프레시 토큰이 갱신된다.
- 브라우저 쿠키에 저장된다.

4. 리프레시 토큰을 가지고 나중에 도착한 `reissue` 요청을한다.
5. 서버에서 갱신된 리프레시 토큰과 이미 이전값이 되어버린 리프레시 토큰은 일치하지 않기에 401 에러가 뜬다.

이렇게 여러 요청중 첫번째 요청만이 유효한 리프레시 토큰으로 서버에 접근하게 되고, 나머지 후에 요청들은 폐기된 토큰값으로 접근하게 되어 문제가 되는 것이다.

# 해결

첫 401 응답에서만 reissue를 호출해 refreshPromise를 만들고, 다른 요청들은 그 프로미스가 끝날 때까지 await해서 브라우저 쿠키에 저장된 **최신 토큰**으로 재시도하게 한다.

---

### **1) fetchRefresh - 토큰 재발급 요청**

- 실제로 reissue(postReissue) API를 호출하고 실패 시 적절히 에러를 던짐.

```tsx
const fetchRefresh = async () => {
  try {
    await postReissue();
  } catch (error) {
    console.error("토큰 갱신 실패", error);
    if (error instanceof ApiError) {
      // 호출자에서 처리할 수 있도록 같은 타입의 에러를 던진다
      throw new ApiError("토큰 갱신 실패", error.status);
    }
    throw error;
  }
};
```

postReissue()의 성공/실패를 캡슐화해서 호출부가 단순하게 await만 하게 만든다.

---

### **2) ensureRefreshed - 중복 리프레시 토큰 발급 방지**

- 여러 요청이 동시에 401을 받더라도, 실제 postReissue() 요청은 **단 한 번만** 발생하도록 제어한다.
- 한 요청이 이미 리프레시를 시작했다면, 나머지 요청은 **그 Promise가 끝날 때까지 기다리기만 한다.**

```tsx
let refreshPromise: Promise<void> | null = null;

const ensureRefreshed = async () => {
  if (!refreshPromise) {
    refreshPromise = fetchRefresh().finally(() => {
      refreshPromise = null;
    });
  }
  return refreshPromise;
};
```

동작 원리

- **처음 401을 받은 요청만** fetchRefresh()를 호출해 postReissue() API를 보낸다.
  이 시점에서 refreshPromise는 fetchRefresh()의 Promise 객체로 채워진다.
- **이후에 들어온 요청들은** refreshPromise가 이미 존재하므로
  if (!refreshPromise) 블록을 건너뛰고, 기존 Promise를 그대로 return 받는다.
  즉, 이들은 **postReissue를 새로 호출하지 않고**, 단지 await refreshPromise로
  “첫 번째 리프레시가 끝날 때까지 기다리는” 역할만 수행한다.
- 첫 번째 요청의 postReissue()가 완료되면, 쿠키에 새로운 토큰이 저장되고,
  finally 블록에서 refreshPromise가 다시 null로 초기화된다.

---

### **3) sendRequest - 실제 API 호출(재시도 가능)**

- 단일 요청 로직을 함수로 분리. 이 함수는 요청을 보내고, 실패 시 ApiError를 던짐.

```tsx
const sendRequest = async <T,>(url: URL, options: RequestInit): Promise<T> => {
  const response = await fetch(url, options);
  if (!response.ok) {
    const data = await response.json().catch(() => ({}));
    throw new ApiError((data as any).message ?? "요청 실패", response.status);
  }
  return response.json();
};
```

---

<details>
  <summary>get메서드 전체 코드</summary>

```tsx
async get<T>({ endpoint, searchParams, withCredentials }: ApiClientGetType): Promise<T> {
const url = new URL(`${this.#baseUrl}${endpoint}`);
url.search = new URLSearchParams(searchParams).toString();

const options: RequestInit = {
  method: 'GET',
  headers: {
    accept: 'application/json',
    'Content-Type': 'application/json',
  },
  credentials: withCredentials ? 'include' : ('same-origin' as RequestCredentials),
};

try {
  // 0) 이미 리프레시가 진행 중이면 기다린다 (다른 요청들이 최신 토큰으로 요청하도록)
  if (refreshPromise) {
    await refreshPromise;
  }

  // 1) 실제 요청 시도
  return await sendRequest<T>(url, options);
} catch (error) {
  // 2) 401/403이면 토큰 재발급을 시도하고, 재발급 완료 후 재요청
  if (error instanceof ApiError && (error.status === 401 || error.status === 403)) {
    try {
      await ensureRefreshed(); // 내부에서 중복 방지
    } catch (refreshError) {
      // 리프레시 자체 실패: 호출자에게 에러 전파
      console.error('토큰 재발급 중 오류', refreshError);
      throw refreshError;
    }

    try {
      // 3) 리프레시가 완료된 뒤, 최신 토큰을 사용해 다시 요청
      return await sendRequest<T>(url, options);
    } catch (retryError) {
      console.error('재요청 실패', retryError);
      if (retryError instanceof ApiError) {
        throw new ApiError('재요청 실패', retryError.status);
      }
      throw retryError;
    }
  }

  // 4) 401/403이 아닌 다른 에러는 그대로 던진다
  throw error;
}
}
```

</details>

<br />

# **리프레시 동작 흐름 정리**

1. **첫 실패 요청(A)이 401을 받고 ensureRefreshed() 진입**
   - 아직 refreshPromise가 null이므로 fetchRefresh()를 실행하고, 그 Promise를 refreshPromise에 저장한다.
   - 내부에서 postReissue() API가 호출되어 실제 토큰 재발급이 시작된다.
   - 이 시점부터 다른 요청들은 “리프레시 중” 상태를 공유하게 된다.
2. **이후 요청들(B, C)이 거의 동시에 401을 받음**
   - 이제 refreshPromise는 이미 존재한다.
   - 따라서 B, C는 if (!refreshPromise) 블록을 건너뛰고 곧바로 return refreshPromise로 기존 Promise를 받는다.
   - 즉, **B와 C는 새로운 reissue를 시도하지 않고**,
     “A의 리프레시가 끝날 때까지 기다리는 역할”만 한다.
   - await이 없다면 이벤트 루프(Task Queue)에 대기하지 않고,
     갱신 전의 만료된 토큰으로 sendRequest()를 바로 재실행해 다시 401이 발생한다.
     → 결국 **await이 리프레시 완료까지의 정확한 비동기 대기 흐름을 보장**한다.
3. **A의 리프레시(postReissue)가 완료됨**
   - 서버가 새 토큰을 발급하고, 쿠키에 최신 토큰이 저장된다.
   - Promise가 resolve되며 finally에서 refreshPromise = null로 초기화된다.
   - 즉, “리프레시 작업이 끝났다”는 신호를 보낸 셈이다.
4. **대기 중이던 요청(B, C)이 순차적으로 깨어남**
   - await refreshPromise가 끝나자마자, B와 C는 다음 라인으로 이동한다.
   - 이미 저장된 **최신 토큰으로 sendRequest()를 재요청**해 정상 응답을 받는다.
5. **다음 사이클에서도 같은 패턴이 반복**
   - 다음에 또 여러 요청이 동시에 401을 받아도,
     **A처럼 첫 요청만 리프레시를 실행**하고 나머지는 그 Promise를 기다린다.
   - 덕분에 **리프레시 요청은 항상 단 한 번만 발생**하며,
     모든 요청이 안전하게 최신 토큰으로 재시도된다.
