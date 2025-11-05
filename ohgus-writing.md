# TypeScript 제네릭 실전 가이드

## API 타입 안전성과 코드 중복을 동시에 해결하는 4가지 패턴

## 1. 개요

### 1.1 글의 목적

본 글은 TypeScript를 이미 사용하고 있으나, 제네릭(Generic)을 **막연히 어렵고 복잡한 기능**으로만 인식하고 있는 웹 개발자를 대상으로 한다. 특히 다음과 같은 문제의식을 가진 독자를 상정한다.

- `fetch().json()`의 반환 타입이 항상 `any`로 추론되는 상황이 불편한 경우
- API Client·유틸 함수·공통 컴포넌트를 구현하면서
  **여기를 더 타입 안전하게 만들 수 있을 것 같은데…** 라는 생각을 해본 경우
- 코드 중복과 타입 안전성을 **동시에 개선할 수 있는 실전 패턴**을 찾고 있는 경우

이 글의 목표는 다음과 같다.

1. **제네릭의 기본 개념을 직관적으로 이해**하고
2. 실제 프로젝트에서 **API 타입 안전성과 코드 중복 제거에 활용 가능한 4가지 패턴**을 정리하며
3. 팀/서비스 코드베이스에 **바로 적용할 수 있는 기준과 예시**를 제공하는 것

### 1.2 대상 독자

- API Client, 데이터 레이어, 공통 유틸 구현을 담당하는 개발자
- 코드 품질과 타입 안전성에 관심이 높은 주니어 레벨 개발자

---

## 2. 배경: any와 코드 중복 문제

다음과 같은 코드는 실제 서비스 코드베이스에서도 쉽게 발견된다.

```tsx
// 사용자 정보 가져오기
async function fetchUser() {
  const response = await fetch('/api/user');
  const data = await response.json();
  return data;
}

// 게시글 정보 가져오기
async function fetchPost() {
  const response = await fetch('/api/post');
  const data = await response.json();
  return data;
}

// 댓글 정보 가져오기
async function fetchComment() {
  const response = await fetch('/api/comment');
  const data = await response.json();
  return data;
}
```

표면적으로 보면 **URL만 다르고 구현은 거의 동일**하다.
문제는 이 코드가 반환하는 값의 타입이다.

```tsx
const response = await fetch('/api/user');
const data = await response.json(); // data: any
```

기본적으로 `response.json()`의 반환 타입은 `any`로 추론된다. 이로 인해 다음 문제가 발생한다.

- **자동완성(IDE 인텔리센스)** 를 제대로 활용할 수 없다.
- 프로퍼티 이름에 **오타가 있어도 컴파일 타임에 감지되지 않는다.**
- 런타임에서 `undefined` 등의 값이 튀어나온 뒤에야 문제를 인지하게 된다.
- API 응답 구조를 리팩토링할 때, **타입 시스템이 전혀 도움을 주지 못한다.**

```tsx
const user = await fetchUser();

console.log(user.nmae); // 오타지만 컴파일 에러 없음
// 실제 실행 후에야 문제 인지
```

또한 엔드포인트마다 거의 동일한 함수가 반복되면서:

- 공통 에러 처리 로직을 수정할 때 **여러 함수를 동시에 수정**해야 하고
- 응답 구조가 변경되면, **각 함수의 반환 타입과 사용처를 일일이 추적**해야 한다.

이 글에서는 이러한 문제를 **TypeScript 제네릭을 활용해 구조적으로 해결**하는 방법을 다룬다.

---

## 3. 제네릭 기본 개념

### 3.1 제네릭이란 무엇인가?

**제네릭(Generic)** 은 한 문장으로 다음과 같이 정의할 수 있다.

> **타입을 매개변수로 받는 함수 또는 타입 정의**

일반 함수와 비교하면 개념이 보다 명확해진다.

```tsx
// 일반 함수: 값(value)을 매개변수로 받음
function echo(value: string): string {
  return value;
}

echo('hello'); // ✅
echo(42); // ❌ number는 허용되지 않음
```

```tsx
// 제네릭 함수: 타입(type)을 매개변수로 받음
function echo<T>(value: T): T {
  return value;
}

const str = echo('hello'); // T = string
const num = echo(42); // T = number
const obj = echo({ x: 1 }); // T = { x: number }
```

여기서 `<T>`는 **타입 매개변수(Type Parameter)** 이며 다음과 같은 특징이 있다.

- 관습적으로 `T`를 사용하지만, `TData`, `TResponse` 등 **의미 있는 이름**을 사용하는 것이 권장된다.
- 대다수의 경우, 제네릭 타입 인자는 **함수 호출 시 자동으로 추론**된다.
- 동일한 로직을 여러 타입에 대해 재사용할 수 있도록 해준다.

### 3.2 배열 유틸 함수 예시

간단한 예로, 배열의 첫 번째 요소를 반환하는 함수에 제네릭을 적용할 수 있다.

```tsx
function getFirst<T>(arr: T[]): T | undefined {
  return arr[0];
}

const numbers = [1, 2, 3];
const firstNum = getFirst(numbers);
// firstNum: number | undefined

const users = [
  { id: 1, name: 'John' },
  { id: 2, name: 'Jane' },
];
const firstUser = getFirst(users);
// firstUser: { id: number; name: string } | undefined

if (firstUser) {
  console.log(firstUser.name); // 타입 안전하게 접근 가능
}
```

이처럼 제네릭을 사용하면 **배열의 요소 타입에 상관없이 동일한 로직을 재사용하면서도**,
각 요소의 **구체적인 타입 정보를 유지**할 수 있다.

---

## 4. 실전 패턴 ① – 제네릭 fetch 래퍼로 any 제거하기

가장 먼저 해결해야 할 문제는 `response.json()`의 `any` 타입이다.
이를 위해 제네릭을 사용한 `fetchData` 래퍼 함수를 정의할 수 있다.

```tsx
// 제네릭을 사용한 fetch 래퍼 함수
async function fetchData<T>(url: string): Promise<T> {
  const response = await fetch(url);

  if (!response.ok) {
    throw new Error(`HTTP ${response.status}`);
  }

  // 반환 타입은 any가 아닌 T
  return response.json();
}
```

사용 시에는 **반환 타입을 제네릭으로 지정**한다.

```tsx
interface User {
  id: number;
  name: string;
  email: string;
}

const user = await fetchData<User>('/api/user');

console.log(user.name); // string, 자동완성 및 타입 체크 지원
// console.log(user.nmae); // 컴파일 에러: 오타 즉시 감지
```

### 4.1 개선 효과 요약

**제네릭 도입 전/후를 비교하면 다음과 같다.**

<table>
  <thead>
    <tr>
      <th>항목</th>
      <th>제네릭 도입 전</th>
      <th>제네릭 도입 후</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>함수 개수</td>
      <td>엔드포인트마다 별도 함수 정의</td>
      <td>공통 제네릭 함수 1개로 통합</td>
    </tr>
    <tr>
      <td>반환 타입</td>
      <td>
        <code>any</code>
      </td>
      <td>
        <code>User</code>, <code>Post</code> 등 구체적인 타입
      </td>
    </tr>
    <tr>
      <td>자동완성</td>
      <td>제한적</td>
      <td>IDE 자동완성 적극 활용 가능</td>
    </tr>
    <tr>
      <td>에러 발견 시점</td>
      <td>런타임</td>
      <td>컴파일 타임</td>
    </tr>
    <tr>
      <td>리팩토링 안정성</td>
      <td>응답 구조 변경 누락 위험</td>
      <td>타입 에러로 변경 누락 즉시 감지</td>
    </tr>
    <tr>
      <td>유지보수</td>
      <td>공통 로직 수정 시 여러 함수 수정 필요</td>
      <td>공통 함수 하나만 수정하면 반영</td>
    </tr>
  </tbody>
</table>

> **핵심 포인트**
>
> - `fetchData<T>` 하나로 **모든 GET 요청을 통합**할 수 있다.
> - API 응답 타입은 **사용하는 쪽에서 제네릭으로 명시**한다.
> - `any` 대신 T를 사용함으로써 **타입 시스템의 도움을 극대화**할 수 있다.

---

## 5. 실전 패턴 ② – 공통 API 응답 타입 제네릭으로 통합하기

실제 서비스의 API는 대체로 비슷한 응답 구조를 갖는다. 예를 들어:

```jsonc
{
  "success": true,
  "data": {
    /* 실제 데이터 */
  },
  "error": null
}
```

이를 제네릭으로 추상화하면 다음과 같다.

```tsx
// 공통 API 응답 타입
interface ApiResponse<T> {
  success: boolean;
  data: T;
  error?: string;
}
```

사용 예시는 아래와 같다.

```tsx
async function fetchUser(): Promise<ApiResponse<User>> {
  return fetchData<ApiResponse<User>>('/api/user');
}

const res = await fetchUser();

if (res.success) {
  // res.data: User
  console.log(res.data.name);
} else {
  console.error(res.error);
}
```

### 5.1 페이지네이션 응답 확장

페이지네이션이 있는 리스트 API는 다음과 같이 확장할 수 있다.

```tsx
interface PaginatedResponse<T> extends ApiResponse<T[]> {
  pagination: {
    page: number;
    totalPages: number;
  };
}

async function fetchUsers(page: number): Promise<PaginatedResponse<User>> {
  return fetchData(`/api/users?page=${page}`);
}

const res = await fetchUsers(1);

if (res.success) {
  res.data.forEach((user) => {
    console.log(user.name); // User[] 타입으로 안전
  });

  console.log(res.pagination.totalPages);
}
```

**이 패턴의 장점은 다음과 같다.**

- 다양한 API를 **일관된 응답 구조**로 다룰 수 있다.
- 응답 스펙이 변경될 때, **공통 타입 정의만 수정해도 전체 코드에 변경이 전파**된다.
- 에러 처리, 로깅, 모니터링 등 **공통 로직을 통합**하기 용이하다.

---

## 6. 실전 패턴 ③ – `extends`와 `keyof`로 타입 제약 강화하기

제네릭을 사용할 때, 모든 타입을 무제한으로 허용하는 것은 오히려 안전성을 떨어뜨릴 수 있다.
이를 방지하기 위해 `extends`와 `keyof`를 활용해 **타입 제약 조건**을 추가할 수 있다.

### 6.1 `extends`를 통한 최소 구조 보장

```tsx
interface BaseResponse {
  success: boolean;
}

// T는 반드시 BaseResponse 형태를 포함해야 함
async function fetchApi<T extends BaseResponse>(url: string): Promise<T> {
  const response = await fetch(url);
  if (!response.ok) throw new Error(`HTTP ${response.status}`);
  return response.json();
}

// 정상: success 필드 포함
fetchApi<ApiResponse<User>>('/api/user');

// 컴파일 에러: success 필드 없음
// fetchApi<{ data: User }>('/api/user');
```

**포인트는 다음과 같다.**

- `T extends BaseResponse`를 통해
  **모든 API 응답은 최소한 `success`를 포함한다** 는 계약을 타입 수준에서 보장할 수 있다.

### 6.2 `keyof`와 `Pick`을 이용한 타입 안전한 키 선택

객체에서 일부 속성만 추출하는 `pick` 유틸을 타입 안전하게 구현하면 다음과 같다.

```tsx
function pick<T, K extends keyof T>(obj: T, keys: K[]): Pick<T, K> {
  const result = {} as Pick<T, K>;

  keys.forEach((key) => {
    result[key] = obj[key];
  });

  return result;
}

const user = {
  id: 1,
  name: 'John',
  email: 'john@example.com',
  password: 'secret',
};

const publicUser = pick(user, ['id', 'name', 'email']);
// publicUser: { id: number; name: string; email: string }

// 컴파일 에러: 'wrong'은 user의 key가 아님
// const invalid = pick(user, ['id', 'wrong']);
```

여기서 사용된 요소는 다음과 같다.

- `keyof T` → `T`의 모든 키를 유니온 타입으로 표현
- `K extends keyof T` → `K`는 `T`의 키 중 하나(또는 그 유니온)여야 함
- `Pick<T, K>` → `T`에서 `K`에 해당하는 키만 추려낸 타입

여기에서 `const result = {} as Pick<T, K>;` 구문은 **TS가 객체를 단계적으로 채워 넣는 패턴을 정확히 추론하지 못하기 때문에 사용하는 관용적인 단언 패턴**이다.
함수 시그니처가 `Pick<T, K>`를 반환하고, 루프에서 실제로 해당 키만 복사하기 때문에 외부 관점에서 구조적으로 일치하는 안전한 단언으로 볼 수 있다.
단언이 부담스럽다면, `Object.fromEntries` 기반 구현이나 검증 로직을 추가하는 방식으로 대체할 수도 있다.

이 패턴은 DTO 변환, 공개/비공개 데이터 분리, 응답 필드 제한 등 다양한 상황에서 활용 가능하다.

---

## 7. 실전 패턴 ④ – 실제 API Client 리팩토링

마지막으로, 실제 서비스 코드에서 API Client를 제네릭 기반으로 리팩토링한 사례를 살펴본다.

### 7.1 리팩토링 전: Response 기반 얇은 래퍼

```tsx
// 개선 전: Response만 반환
const createApiMethod = (method: string) => async (url: string) => {
  const response = await fetch(url, { method });

  if (!response.ok) {
    throw new Error('API 요청 실패');
  }

  return response; // 이후 json()은 any
};

export const apiClient = {
  get: createApiMethod('GET'),
  post: createApiMethod('POST'),
  patch: createApiMethod('PATCH'),
  delete: createApiMethod('DELETE'),
};

// 사용 예시
const response = await apiClient.get('/api/routie');
const data = await response.json(); // data: any
```

이 구조에서는:

- API Client는 단순히 `fetch`를 얇게 감싼 수준이며
- 실제로 중요한 `json()` 결과는 **어디에서도 타입으로 표현되지 않는다.**

### 7.2 리팩토링 후: 제네릭 기반 API Client

```tsx
// T를 반환 타입으로 받는 제네릭 팩토리
const createApiMethod =
  <T,>(method: string) =>
  async (url: string): Promise<T> => {
    const response = await fetch(url, { method });

    if (!response.ok) throw new Error(`HTTP ${response.status}`);

    return response.json();
  };

export const apiClient = {
  get: createApiMethod('GET'),
  post: createApiMethod('POST'),
  patch: createApiMethod('PATCH'),
  delete: createApiMethod('DELETE'),
};

interface RoutieResponse {
  routiePlaces: Array<{ sequence: number; placeId: number }>;
  routes: Array<{ fromSequence: number; toSequence: number }>;
}

const data = await apiClient.get<RoutieResponse>('/api/routie');
// data: RoutieResponse
```

### 7.3 개선 효과

**API Client 리팩토링 전/후 비교**

<table>
  <thead>
    <tr>
      <th>항목</th>
      <th>리팩토링 전</th>
      <th>리팩토링 후</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>
        <code>json()</code> 타입
      </td>
      <td>
        <code>any</code>
      </td>
      <td>
        구체적인 응답 타입(<code>RoutieResponse</code> 등)
      </td>
    </tr>
    <tr>
      <td>자동완성</td>
      <td>제한적</td>
      <td>필드 이름까지 정확하게 제안</td>
    </tr>
    <tr>
      <td>리팩토링 안정성</td>
      <td>응답 구조 변경 누락 가능성 존재</td>
      <td>타입 에러를 통해 변경 누락 즉시 감지</td>
    </tr>
    <tr>
      <td>추상화 수준</td>
      <td>Response 래핑 수준</td>
      <td>타입 정보를 포함하는 도메인 수준 추상화</td>
    </tr>
  </tbody>
</table>

### 7.4 요청/응답 타입을 동시에 다루는 변형

POST/PATCH/DELETE 요청처럼 **요청 바디와 응답 타입을 모두 중요하게 다뤄야 하는 경우**는 다음과 같이 구현할 수 있다.

```tsx
// TResponse를 먼저, TRequest를 나중에 배치
// → "결과(응답) → 입력(요청)" 순서로, useMutation 패턴과 유사한 형태
const createMutationMethod =
  <TResponse, TRequest>(method: 'POST' | 'PATCH' | 'DELETE') =>
  async (url: string, body?: TRequest): Promise<TResponse> => {
    const res = await fetch(url, {
      method,
      headers: { 'Content-Type': 'application/json' },
      body: body ? JSON.stringify(body) : undefined,
    });

    if (!res.ok) throw new Error(`HTTP ${res.status}`);

    return res.json();
  };

type CreateUserRequest = { name: string; email: string };
type CreateUserResponse = {
  id: number;
  name: string;
  email: string;
  createdAt: string;
};

const createUser = createMutationMethod<CreateUserResponse, CreateUserRequest>(
  'POST'
);

const newUser = await createUser('/api/users', {
  name: 'John',
  email: 'john@example.com',
});
```

여기서 **타입 매개변수 순서를 `TResponse → TRequest`로 두는 이유**는 다음과 같다.<br/>
사용자가 가장 관심 있는 것은 보통 **무엇이 반환되는가(응답)** 이기 때문에 앞에 둔다.<br/>
`useMutation<TData, TError, TVariables>`처럼 널리 쓰이는 패턴과 유사해,
**다른 라이브러리와 함께 사용할 때 인지 부하가 줄어든다.**

이 패턴을 통해

- 요청/응답 스펙을 **타입 수준에서 명시**할 수 있고
- 실제 구현과 API 문서 간 불일치가 발생할 경우, **타입 에러로 빠르게 감지**할 수 있다.

---

## 8. 제네릭 사용 시 자주 발생하는 실수

제네릭은 강력한 도구이지만, 잘못 사용하면 코드 가독성과 안정성이 모두 떨어질 수 있다.
아래는 현업에서 자주 등장하는 실수 패턴들이다.

### 8.1 불필요한 제네릭 사용

```tsx
// 불필요한 제네릭 사용
function add<T>(a: number, b: number): number {
  return a + b;
}

// 제네릭이 필요 없는 경우
function add(a: number, b: number): number {
  return a + b;
}
```

> **제네릭 도입 기준**
>
> - 여러 타입에서 동일한 로직을 재사용해야 하는가?
> - 입력과 출력 타입 사이에 **의미 있는 관계**를 표현해야 하는가?

이 두 기준을 만족하지 않는다면, 제네릭 없이 **단순 타입을 사용하는 편이 더 낫다.**

### 8.2 의미 없는 타입 매개변수 이름

```tsx
// 의미가 모호한 타입 매개변수
function fetch<T, T2, T3>(/* ... */) {}

// 역할이 드러나는 이름
function fetch<TData, TError, TVariables>(/* ... */) {}
```

제네릭 매개변수의 개수가 많아질수록 **이름에서 역할이 드러나야 가독성이 유지**된다.

### 8.3 `extends` 제약 조건 누락

```tsx
// length 프로퍼티가 있다고 가정하지만 타입으로 보장되지 않음
function getLength<T>(value: T) {
  // @ts-expect-error: T에는 length가 없을 수도 있음
  return value.length;
}

// length 프로퍼티를 가진 타입으로 제한
function getLength<T extends { length: number }>(value: T) {
  return value.length;
}
```

### 8.4 `any`로 되돌아가는 구현

```tsx
// 제네릭의 의미가 사라진 구현
function process<T>(data: any): any {
  return data;
}

// 입력과 출력을 T로 연결
function process<T>(data: T): T {
  return data;
}
```

### 8.5 타입 추론을 과도하게 무시

```tsx
// 불필요한 명시적 타입 인자
const result = echo<string>('hello');

// 타입 추론을 신뢰하는 편이 더 간결하다
const result = echo('hello'); // result: string
```

TypeScript는 일반적으로 강력한 타입 추론을 제공한다.
**명시가 필요한 지점에서만 제네릭 타입 인자를 표기**하는 것이 좋다.

### 8.6 Try-Catch와 에러 타입 다루기

제네릭으로 **데이터 타입**을 안전하게 만들면서도, 에러 쪽은 여전히 `any`나 `Error` 한 가지 타입으로만 취급하는 경우가 많다. 에러까지 타입 안전하게 가져가려면 두 가지 방향을 고려할 수 있다.

1. **반환 타입에 에러를 포함하는 패턴 (Result 타입)**

```tsx
// 성공/실패를 하나의 유니온 타입으로 정의
type ApiResult<TData, TError = string> =
  | { ok: true; data: TData }
  | { ok: false; error: TError };

async function safeFetch<T, E = string>(url: string): Promise<ApiResult<T, E>> {
  try {
    const res = await fetch(url);
    if (!res.ok) {
      return { ok: false, error: `HTTP ${res.status}` as E };
    }
    const data = await res.json();
    return { ok: true, data };
  } catch (e) {
    return { ok: false, error: 'NETWORK_ERROR' as E };
  }
}
```

2. **try-catch 내부에서 에러 분기 로직을 명시적으로 두고, 호출부에서는 `ok` 여부에 따라 분기**

```tsx
const result = await safeFetch<User>('/api/user');

if (!result.ok) {
  // result.error는 TError 타입
  console.error(result.error);
} else {
  // result.data는 User 타입
  console.log(result.data.name);
}
```

이 글의 본문에서는 데이터 타입에 초점을 맞추기 위해 에러 타입을 깊게 파고들지는 않았지만,
실제 팀 도입 시에는 다음과 같은 기준을 함께 고민하는 것을 추천한다.

- 에러를 **예외(`throw`)로 처리할지, Result 타입으로 다룰지**
- 공통 에러 타입(`ApiError`)을 정의할지, 문자열/HTTP 코드 수준에서만 다룰지
- 로깅/모니터링 시스템(`Sentry` 등)과 어떻게 연동할지

데이터 타입을 제네릭으로 정리한 뒤, 다음 단계로 **에러 타입 설계까지 확장**하면 API 계층 전체의 안정성을 한 단계 더 끌어올릴 수 있다.

---

## 9. 결론 및 적용 가이드

### 9.1 요약

본 글에서는 다음과 같은 내용을 다루었다.

- **배경**

  - API 호출 코드에서 `any` 남용과 코드 중복이 발생하는 근본적인 이유

- **제네릭 기본 개념**

  - “타입을 매개변수로 받는 함수/타입”으로서의 제네릭 이해

- **실전 패턴 4가지**

  1. `fetchData<T>` 제네릭 래퍼로 `any` 제거
  2. `ApiResponse<T>`, `PaginatedResponse<T>`로 공통 응답 구조 통합
  3. `extends`, `keyof`, `Pick`을 활용한 타입 제약 강화
  4. 실제 API Client를 제네릭 기반으로 리팩토링하는 패턴

- **자주 발생하는 실수와 권장 패턴**

  - 불필요한 제네릭 남용, 의미 없는 타입 매개변수 이름, `any` 회귀 등

### 9.2 실무 적용 순서 제안

실제 코드베이스에 제네릭 기반 API 패턴을 적용할 때는, 한 번에 모든 곳을 고치기보다는 **작은 단계부터 점진적으로 확장하는 방식**을 추천한다.

1. **1단계 – `fetch().json()`의 `any` 제거 (`fetchData<T>` 도입)**

   - `fetch().json()`을 직접 사용하는 지점을 한두 곳 선정한다.
   - `fetchData<T>` 제네릭 래퍼로 치환하고, 구체적인 응답 타입을 정의한다.
   - 가장 영향 범위가 좁은 API부터 적용해 보면서, 팀 내 합의와 사용 패턴을 맞춰간다.

2. **2단계 – 공통 응답 구조 도입 (`ApiResponse<T>`, `PaginatedResponse<T>`)**

   - 여러 API가 공유하는 공통 응답 구조를 정리한다.
   - `ApiResponse<T>`, `PaginatedResponse<T>` 같은 제네릭 응답 타입을 도입해,  
     `response.success`, `response.data` 형태로 사용하는 패턴을 통일한다.
   - 신규 API나 변경 예정 API부터 먼저 적용하고, 기존 엔드포인트는 점진적으로 마이그레이션한다.

3. **3단계 – API Client 제네릭화 (패턴 ④ 적용)**

   - 공통 API Client 레이어(`apiClient`)를 제네릭 기반으로 재설계한다.
   - GET/Mutation 메서드에 `<TResponse, TRequest>` 패턴을 도입해,  
     각 엔드포인트의 요청/응답 타입이 함수 시그니처에 드러나도록 만든다.
   - 도메인(유저, 결제, 게시글 등) 단위로 나누어, 가장 변경이 쉬운 영역부터 차례대로 마이그레이션한다.

4. **4단계 – 에러 타입 및 에러 처리 패턴 정리 (선택 적용)**
   - `throw Error` 중심에서 벗어나, `ApiResult<TData, TError>`와 같은 Result 패턴 도입 여부를 검토한다.
   - 공통 에러 타입(`ApiError`)을 정의하거나, HTTP 코드/에러 코드 기준으로 분류하는 규칙을 정리한다.
   - 로깅·모니터링 시스템(Sentry 등)과 연동 전략을 함께 논의하면서, 핵심 API부터 적용해 나간다.

“데이터 타입 → 응답 구조 → API Client → 에러 타입” 순으로 확장해 나가면,  
각 단계마다 영향을 받는 범위를 명확히 나눌 수 있고, 팀 내부 합의도 단계별로 맞춰가기 수월하다.

---

## 10. 참고 자료

- [TypeScript Handbook – Generics](https://www.typescriptlang.org/docs/handbook/2/generics.html)

- [TypeScript Handbook – Utility Types (`Pick`, `Omit` 등)](https://www.typescriptlang.org/docs/handbook/utility-types.html)

- [TypeScript Handbook – Functions](https://www.typescriptlang.org/docs/handbook/2/functions.html)

- [MDN Web Docs – Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API)
