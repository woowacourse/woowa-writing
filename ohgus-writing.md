# TypeScript 제네릭 실전 가이드: API 타입 안전성과 코드 중복 한 번에 해결하기

## 제네릭, 5분 만에 이해하기

### 왜 제네릭을 배워야 하는가

다음과 같은 코드를 작성한 경험이 있을 것이다.

```tsx
// 사용자 정보 가져오기
async function fetchUser() {
  // API 호출
  const response = await fetch('/api/user');
  // JSON으로 파싱 (문제: 타입이 any!)
  const data = await response.json();
  return data;
}

// 게시글 정보 가져오기 (위와 거의 동일)
async function fetchPost() {
  const response = await fetch('/api/post');
  const data = await response.json(); // 또 any 타입!
  return data;
}

// 댓글 정보 가져오기 (또 동일한 코드)
async function fetchComment() {
  const response = await fetch('/api/comment');
  const data = await response.json(); // 계속 any...
  return data;
}
```

URL만 다를 뿐 나머지는 완전히 동일한 코드다. 복사 붙여넣기로 함수를 3개나 만든 것이다.

더 큰 문제가 있다. VSCode에서 `data`에 마우스를 올려보면 다음과 같이 표시된다.

```tsx
const data: any;
```

`any` 타입이다. 자동완성도 작동하지 않으며, 오타를 내도 에러가 발생하지 않는다.

```tsx
const user = await fetchUser();
console.log(user.nmae); // 오타인데 에러가 발생하지 않음
// 런타임에 undefined가 출력되고 나서야 문제를 알 수 있음
```

런타임에 실행해봐야만 에러를 발견할 수 있다.

### 제네릭으로 해결하기

제네릭을 사용하면 이 모든 문제가 해결된다.

```tsx
// 하나의 함수로 통합
// <T>: 타입을 매개변수로 받음
async function fetchData<T>(url: string): Promise<T> {
  const response = await fetch(url);
  // T 타입으로 반환 (any가 아님)
  return response.json();
}

// 타입 정의
interface User {
  id: number;
  name: string;
  email: string;
}

interface Post {
  id: number;
  title: string;
  content: string;
}

// 사용 - 타입 안전하게
// <User>를 전달하여 반환 타입을 명시
const user = await fetchData<User>('/api/user');
console.log(user.name); // ✅ string 타입, 자동완성 작동

// <Post>를 전달하여 Post 타입으로 추론
const post = await fetchData<Post>('/api/post');
console.log(post.title); // ✅ string 타입, 타입 체크 완벽
```

이제 `user.nmae`처럼 오타를 내면 **컴파일 시점에 에러**가 발생한다. VSCode가 빨간 줄로 표시해준다.

**개선 효과를 표로 정리하면 다음과 같다.**

| 항목        | 제네릭 없이        | 제네릭 사용            |
| ----------- | ------------------ | ---------------------- |
| 함수 개수   | 타입마다 따로 작성 | **하나로 통합**        |
| 타입 안전성 | `any` (위험)       | **명확한 타입**        |
| 자동완성    | ❌ 작동하지 않음   | ✅ **완벽하게 작동**   |
| 에러 발견   | 런타임 (늦음)      | **컴파일 타임 (빠름)** |
| 유지보수    | 여러 곳 수정       | **한 곳만 수정**       |

### 제네릭이란 무엇인가

제네릭은 **타입을 매개변수로 받는 함수**다.

일반 함수와 비교해보자.

```tsx
// 일반 함수: 값을 매개변수로 받음
function echo(value: string): string {
  // 문자열만 받을 수 있음
  return value;
}

echo('hello'); // ✅ 작동
echo(42); // ❌ 에러! number는 불가능
```

```tsx
// 제네릭 함수: 타입을 매개변수로 받음
// <T>가 타입 매개변수
function echo<T>(value: T): T {
  // 어떤 타입이든 받을 수 있음
  // 들어온 타입 그대로 반환
  return value;
}

// T = string으로 결정됨
const str = echo('hello');
// str의 타입: string

// T = number로 결정됨
const num = echo(42);
// num의 타입: number

// T = { x: number }로 결정됨
const obj = echo({ x: 1 });
// obj의 타입: { x: number }
```

`<T>`가 바로 타입 매개변수다. 함수를 호출할 때 타입이 결정된다.

**`<T>`의 의미**

- `T`는 Type의 약자다 (관습적 명명)
- 다른 이름도 사용 가능하다 (`U`, `V`, `TData`, `TResponse` 등)
- 함수 호출 시 TypeScript가 자동으로 타입을 추론한다

### 더 실용적인 예제

배열의 첫 번째 요소를 가져오는 함수를 만들어보자.

```tsx
// T[]: T 타입의 배열
// 반환: T | undefined (배열이 비어있을 수 있으므로)
function getFirst<T>(arr: T[]): T | undefined {
  // 첫 번째 요소 반환
  return arr[0];
}

// 숫자 배열
const numbers = [1, 2, 3];
const firstNum = getFirst(numbers);
// firstNum의 타입: number | undefined

// 객체 배열
const users = [
  { id: 1, name: 'John' },
  { id: 2, name: 'Jane' },
];
const firstUser = getFirst(users);
// firstUser의 타입: { id: number, name: string } | undefined

// 이제 안전하게 사용 가능
if (firstUser) {
  console.log(firstUser.name); // ✅ 타입 체크 완벽
}
```

제네릭 덕분에 배열의 타입이 무엇이든 첫 번째 요소의 타입을 정확하게 추론한다.

### 인터페이스에도 제네릭 사용하기

함수뿐만 아니라 인터페이스에도 제네릭을 사용할 수 있다.

```tsx
// 어떤 타입이든 담을 수 있는 박스
// <T>가 인터페이스의 타입 매개변수
interface Box<T> {
  value: T; // T 타입의 값을 담음
}

// 숫자를 담는 박스
const numBox: Box<number> = { value: 42 };
// numBox.value는 number 타입

// 문자열을 담는 박스
const strBox: Box<string> = { value: 'hello' };
// strBox.value는 string 타입

// 사용자 객체를 담는 박스
const userBox: Box<User> = {
  value: { id: 1, name: 'John', email: 'john@example.com' },
};
// userBox.value는 User 타입
```

API 응답 타입도 제네릭으로 만들 수 있다.

```tsx
// 모든 API 응답의 공통 구조
interface ApiResponse<T> {
  success: boolean; // 성공 여부
  data: T; // 실제 데이터 (타입은 T)
  error?: string; // 에러 메시지 (선택)
}

// 사용자 API 응답
const userResponse: ApiResponse<User> = {
  success: true,
  data: { id: 1, name: 'John', email: 'john@example.com' },
  // data는 User 타입
};

// 게시글 목록 API 응답
const postsResponse: ApiResponse<Post[]> = {
  success: true,
  data: [
    { id: 1, title: '첫 글', content: '내용' },
    { id: 2, title: '둘째 글', content: '내용2' },
  ],
  // data는 Post[] 타입
};
```

하나의 `ApiResponse` 타입으로 모든 API 응답을 표현할 수 있다.

### 정리

**제네릭을 한 문장으로 정리하면**

> 타입을 매개변수로 받아서, 여러 타입에서 재사용 가능하면서도 타입 안전성을 유지하는 기능

**언제 사용하는가**

- 같은 로직을 여러 타입에서 사용할 때
- 타입을 미리 정할 수 없을 때
- API 응답, 유틸리티 함수, 재사용 가능한 컴포넌트 등

**핵심만 기억하자**

```tsx
// 이 3줄만 이해하면 90% 이해한 것이다

// 1. 제네릭 함수
function 함수이름<T>(매개변수: T): T {}

// 2. 제네릭 인터페이스
interface 인터페이스이름<T> {
  value: T;
}

// 3. 사용
const result = 함수<타입>(값);
```

다음 장에서는 이 제네릭을 실전 프로젝트에 어떻게 적용하는지 알아볼 것이다. API 호출 코드를 완전히 타입 안전하게 만들어보자.

## 실전 - API 타입 안전하게 만들기

### 문제 상황: response.json()이 any 타입

실제 프로젝트에서 자주 보는 코드를 살펴보자.

```tsx
// 일반적인 fetch 사용
const response = await fetch('/api/user');
const data = await response.json(); // any 타입
console.log(data.name); // 오타가 나도 알 수 없음
```

VSCode에서 `data`에 마우스를 올려보면 `any` 타입이라고 표시된다.

**문제점:**

- 자동완성이 작동하지 않음
- 오타를 내도 컴파일 에러가 발생하지 않음
- 런타임에 실행해봐야 에러를 발견할 수 있음
- 리팩토링 시 타입 체크가 불가능

### 해결 1: 제네릭 fetch 함수

타입을 명시하는 fetch 래퍼 함수를 만들어보자.

```tsx
// 제네릭을 사용한 fetch 래퍼
// <T>: 반환할 데이터의 타입
async function fetchData<T>(url: string): Promise<T> {
  // API 호출
  const response = await fetch(url);

  // HTTP 에러 처리
  if (!response.ok) {
    throw new Error(`HTTP ${response.status}`);
  }

  // T 타입으로 반환 (any가 아님)
  return response.json();
}

// 사용
interface User {
  id: number;
  name: string;
  email: string;
}

// <User>를 전달하여 타입 명시
const user = await fetchData<User>('/api/user');
console.log(user.name); // string 타입, 자동완성 작동
console.log(user.nmae); // 컴파일 에러 - 오타 즉시 발견
```

이제 타입 안전하게 API를 호출할 수 있다.

### 해결 2: API 응답 통합 타입

대부분의 API는 비슷한 응답 구조를 가진다. 이를 제네릭으로 통합해보자.

```tsx
// 공통 API 응답 타입
// <T>: 실제 데이터의 타입
interface ApiResponse<T> {
  success: boolean; // 성공 여부
  data: T; // 실제 데이터 (타입은 T)
  error?: string; // 에러 메시지 (선택적)
}

// 사용
async function fetchUser(): Promise<ApiResponse<User>> {
  return fetchData<ApiResponse<User>>('/api/user');
}

// 안전하게 사용
const response = await fetchUser();
if (response.success) {
  // response.data는 User 타입
  console.log(response.data.name); // 타입 안전
} else {
  console.error(response.error);
}
```

**페이지네이션 응답도 간단히 처리할 수 있다.**

```tsx
// 페이지네이션 응답 타입
// extends로 ApiResponse를 확장
interface PaginatedResponse<T> extends ApiResponse<T[]> {
  pagination: {
    page: number; // 현재 페이지
    totalPages: number; // 전체 페이지 수
  };
}

// 사용
async function fetchUsers(page: number): Promise<PaginatedResponse<User>> {
  return fetchData(`/api/users?page=${page}`);
}

// 타입 안전하게 사용
const response = await fetchUsers(1);
if (response.success) {
  // response.data는 User[] 타입
  response.data.forEach((user) => {
    console.log(user.name); // 자동완성 완벽
  });

  // pagination 정보도 타입 안전
  console.log(response.pagination.totalPages); // number 타입
}
```

### 제약 조건: extends로 타입 제한하기

더 안전하게 만들려면 `extends`로 제약 조건을 추가할 수 있다.

```tsx
// 기본 응답 인터페이스
interface BaseResponse {
  success: boolean;
}

// API 응답은 반드시 success 필드를 가져야 함
async function fetchApi<T extends BaseResponse>(url: string): Promise<T> {
  const response = await fetch(url);
  if (!response.ok) {
    throw new Error(`HTTP ${response.status}`);
  }
  return response.json();
}

// 올바른 사용 (success 필드 있음)
fetchApi<ApiResponse<User>>('/api/user');

// 컴파일 에러 (success 필드 없음)
fetchApi<{ data: User }>('/api/user');
// Error: Type '{ data: User }' does not satisfy constraint 'BaseResponse'
```

`extends`를 사용하면 잘못된 타입 사용을 컴파일 시점에 방지할 수 있다.

### keyof로 객체 키 안전하게 다루기

객체의 특정 속성만 추출하는 함수를 만들어보자.

```tsx
// 타입 안전한 pick 함수
// T: 원본 객체 타입
// K: T의 키들 (keyof T로 제한)
function pick<T, K extends keyof T>(obj: T, keys: K[]): Pick<T, K> {
  // 결과 객체 (Pick<T, K> 타입)
  const result = {} as Pick<T, K>;

  // 선택한 키들만 복사
  keys.forEach((key) => {
    result[key] = obj[key];
  });

  return result;
}

// 사용 예시
const user = {
  id: 1,
  name: 'John',
  email: 'john@example.com',
  password: 'secret123',
};

// 비밀번호를 제외한 공개 정보만 추출
const publicUser = pick(user, ['id', 'name', 'email']);
// publicUser의 타입: { id: number, name: string, email: string }

// 컴파일 에러 - 'wrong'은 user의 키가 아님
const invalid = pick(user, ['id', 'wrong']);
// Error: Type '"wrong"' is not assignable to type 'keyof { id: number, name: string, ... }'
```

`keyof`를 사용하면 존재하지 않는 키를 사용하는 것을 방지할 수 있다.

### 실전 적용: API Client 개선 사례

실제 프로젝트의 API Client를 제네릭으로 개선한 사례를 살펴보자.

#### Before: 타입이 불명확한 코드

```tsx
// 개선 전 코드 (문제: Response만 반환 → json()이 any)
const createApiMethod = (method: string) => async (url: string) => {
  const response = await fetch(url, { method });
  if (!response.ok) {
    throw new Error('API 요청 실패');
  }
  return response; // 아래에서 any 발생
};

// API Client 생성
export const apiClient = {
  get: createApiMethod('GET'),
  post: createApiMethod('POST'),
  patch: createApiMethod('PATCH'),
  delete: createApiMethod('DELETE'),
};

// 사용하는 곳
const response = await apiClient.get('/api/routie');
const data = await response.json(); // any 타입
```

**문제점**

- `response.json()`의 반환 타입이 `any`
- 자동완성이 작동하지 않음
- 타입 에러를 런타임에 발견

#### After: 제네릭으로 타입 안전하게

```tsx
// 개선 후 코드
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
```

**개선 효과**

| 항목            | Before     | After             |
| --------------- | ---------- | ----------------- |
| 반환 타입       | `any`      | **명확한 타입**   |
| 에러 발견 시점  | 런타임     | **컴파일 타임**   |
| VSCode 자동완성 | 작동 안 함 | **완벽하게 작동** |
| 리팩토링 안전성 | 낮음       | **높음**          |
| 타입 에러 수    | 많음       | **0개**           |

### HTTP 메서드별 타입 정의

```tsx
// 요청/응답 타입을 동시에 다루는 간결한 예시
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

### 정리

**이번 장에서 배운 것**

1. **제네릭 fetch 함수**: `any` 타입을 명확한 타입으로
2. **API 응답 통합 타입**: 모든 API를 하나의 타입으로
3. **extends 제약 조건**: 타입 안전성 강화
4. **keyof 활용**: 객체 키를 안전하게
5. **실전 적용**: 실제 프로젝트 개선 사례

**핵심 패턴:**

```tsx
// 1. 기본 제네릭 API 함수
async function fetchData<T>(url: string): Promise<T>;

// 2. 통합 응답 타입
interface ApiResponse<T> {
  success: boolean;
  data: T;
}

// 3. 제약 조건
async function fetchApi<T extends BaseResponse>(url: string): Promise<T>;

// 4. keyof 활용
function pick<T, K extends keyof T>(obj: T, keys: K[]): Pick<T, K>;
```

## 전체 정리

### 핵심 요약

**제네릭 3줄 정리:**

1. **제네릭 = 타입을 매개변수로 받는 함수**

   ```tsx
   function fetchData<T>(url: string): Promise<T>;
   ```

2. **코드 중복 제거 + 타입 안전성 확보**
   - 3개 함수 → 1개 제네릭 함수
   - `any` 타입 → 명확한 타입
3. **실전 활용**
   - API 함수: `response.json()`의 `any` 제거

### 자주 하는 실수

#### 1. 제네릭 남발

```tsx
// ❌ 불필요한 제네릭
function add<T>(a: number, b: number): number {
  return a + b;
}

// ✅ 간단하게
function add(a: number, b: number): number {
  return a + b;
}
```

**제네릭이 필요한 경우:**

- 여러 타입에서 재사용할 때
- 입력과 출력의 타입 관계를 유지해야 할 때

#### 2. 타입 매개변수 이름 혼란

```tsx
// ❌ 의미 없는 이름
function fetch<T, T2, T3>(...)

// ✅ 명확한 이름
function fetch<TData, TError, TVariables>(...)
```

**관습적 명명:**

- `T`: 기본 타입
- `TData`: 데이터 타입
- `TError`: 에러 타입
- `TVariables`: 변수 타입

#### 3. extends 제약 조건 누락

```tsx
// ❌ 런타임 에러 가능
function getLength<T>(value: T) {
  return value.length; // T에 length가 없을 수 있음
}

// ✅ 안전
function getLength<T extends { length: number }>(value: T) {
  return value.length;
}
```

#### 4. any 타입으로 회귀

```tsx
// ❌ 제네릭의 의미 없음
function process<T>(data: any): any {
  return data;
}

// ✅ 제대로 활용
function process<T>(data: T): T {
  return data;
}
```

#### 5. 타입 추론 무시

```tsx
// ❌ 불필요한 타입 명시
const result = identity<string>('hello');

// ✅ 자동 추론 활용
const result = identity('hello'); // TypeScript가 string으로 추론
```

### 마치며

제네릭은 TypeScript의 핵심 기능이다. 처음에는 어렵게 느껴질 수 있지만, 실전에서 몇 번 사용해보면 그 강력함을 체감할 수 있다.

**기억할 점:**

- 완벽하게 이해하려 하지 말고 일단 사용해보자
- 작은 유틸 함수부터 시작하자
- 실수하면서 배우는 것이 가장 빠르다

당신의 프로젝트에 제네릭을 적용하고, 타입 안전성과 생산성 향상을 경험해보길 바란다.
