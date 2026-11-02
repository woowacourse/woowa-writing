`Webpack`에 대해 이해하려면 우선 **번들**과 **번들러**의 개념을 알아야 한다.

**번들(Bundle)**은 여러 파일을 하나 또는 여러 개의 파일로 묶어 놓은 결과물을 의미한다.

**번들러(Bundler)**는 JavaScript, CSS, 이미지 등 흩어져있는 파일들을 분석하고, 하나 또는 최소의 파일로 결합해주는 도구이다.

`Webpack`은 이러한 번들러 중 하나이다.

> npm create vite@latest

React를 사용한다면 한 번쯤 사용해봤을 명령어이다.

이 명령어는 `Vite`를 기반으로 프로젝트를 생성한다.

`Vite`와 `Webpack` 뭐가 다른 걸까?

## Vite vs Webpack

두 도구의 가장 큰 차이점은 **개발 환경에서 모듈을 처리하는 방식**에 있다.

### Webpack은 번들링을 먼저 수행한다

`Webpack`은 여러 파일을 분석하고 번들로 묶는 방식으로 동작한다. 개발 서버를 실행하면 필요한 파일을 분석하고 번들링한 뒤 브라우저에 제공한다.

### Vite는 ESM을 활용한다

반면 `Vite`는 브라우저가 기본적으로 지원하는 ESM을 활용한다. 

> ESM: JavaScript에서 `import`와 `export`를 사용해 모듈을 내보내고 가져오는 표준 모듈 시스템

`Vite`는 개발 환경에서 소스 코드를 미리 하나의 번들로 묶지 않는다. 대신 브라우저가 요청한 모듈을 필요한 시점에 변환하여 제공한다.

이러한 방식 덕분에 `Vite`는 개발 서버를 빠르게 실행할 수 있다.

### Webpack을 왜 사용할까

이 말만 들었을 때는 `Webpack`을 `Vite` 대신 사용할 이유가 없어보인다.

`Vite`를 사용하면 기본 설정을 자동으로 해주고, 개발 서버도 빠르게 띄워준다. 또한 프로덕션 빌드에서도 번들링을 수행한다.

총총에서도 `Vite`를 사용할 수 있었다. 그럼에도 `Webpack`을 선택한 것은 **코드가 브라우저에 전달되기까지의 과정을 직접 구성하고, 요구사항에 맞게 관리하고 싶었기 때문이다.**

`Vite`는 번들러의 내부 동작을 자세히 알지 못하더라도 프로젝트를 쉽게 시작할 수 있도록 해준다.
이는 큰 장점이지만, 반대로 생각해봤을 때 개발 환경이 어떤 과정을 거쳐 구성되는지 이해하지 못한 채 사용만 할 수 있다.

`Webpack`을 처음부터 설정하다 보면 JavaScript와 TypeScript 파일이 어떻게 변환되는지, 개발 서버와 프로덕션 빌드는 어떤 차이가 있는지 직접 확인할 수 있다.

또한 성능을 개선하려면 코드뿐 아니라 실제로 브라우저에 전달되는 결과물의 크기와 로딩 방식도 살펴봐야 한다고 생각했다. 빌드 설정부터 결과물까지 직접 다루면서, 필요한 최적화를 적용할 수 있는 기반을 만들고자 했다.

이런 이유들로 인해 총총에 `Webpack`을 사용했다.


## 바닥부터 쌓아올리기

먼저, 총총은 `React`와 `TypeScript`를 사용한다. 해당 문법은 브라우저가 실행할 수 없기 때문에 브라우저가 실행할 수 있는 `JavaScript`로 변환해야 한다.

또한, 진입점을 설정해줘야 한다. `main.tsx`가 자동으로 진입점이 되는 것이 아니라 설정을 해줘야 한다.

그럼 `Webpack`이 해야할 일들을 유추해낼 수 있다.

1. 진입점을 정한다.
2. 해당 파일이 사용하는 다른 파일을 찾는다.
3. 여러 파일(tsx, ts)을 브라우저가 읽을 수 있도록 변환한다.
4. 실행에 필요한 여러 코드를 번들링한다.
5. HTML에서 생성된 번들을 불러오도록 연결한다.

### 1. 진입점을 정한다

진입점을 설정할 때는 `entry`를 사용한다. `entry`는 `Webpack`이 의존성 분석을 시작하는 파일이다.

총총의 진입점은 `main.tsx`이다. 따라서 `entry: './main.tsx'`으로 설정했다.

여기서 착각할 수 있는 점은 `Webpack`이 프로젝트 폴더 내의 모든 파일을 하나로 합치는 것은 아니라는 점이다.

진입점으로 설정한 파일에서부터 시작해 의존성을 분석하고, 코드 분할 등의 설정에 따라 최종 결과물을 구성한다.

진입점을 설정했으니 결과물을 만들 곳도 설정해줘야한다. 이땐 `output`을 사용한다. 


```js
// 공통 설정(webpack.common.js)
output: {
  clean: true,
  path: path.resolve(__dirname, 'dist'),
  publicPath: '/',
},
```

```js
// webpack.local.js
output: {
  filename: 'bundle.js',
  chunkFilename: '[name].js',
},
```

`path`는 빌드 결과물이 저장되는 파일 시스템상의 경로이고, `publicPath`는 브라우저가 요청할 때 사용하는 URL의 기준 경로이다.

이 설정에서는 `frontend` 폴더 아래의 `dist`에 결과물을 만든다.

`filename`은 진입점에서 생성되는 JavaScript 번들의 이름을,  `chunkFilename`은 동적 로딩 등으로 별도로 생성되는 청크의 파일 이름을 정한다.
-> 이 설정은 뒤에서 다시 다룬다.

`clean: true`는 새 빌드를 만들 때 이전 결과물을 정리하는 설정이다. 파일 이름이 달라지는 빌드가 반복되더라도 오래된 결과물이 계속 쌓이는 것을 막는 데 도움이 된다.

### 2. 해당 파일이 사용하는 다른 파일을 찾는다

진입점과 결과물을 정했다면 그 파일에서 사용하는 다른 파일을 찾아야 한다.

React 코드를 작성하다 보면 다음과 같이 확장자를 생략한 채 파일을 가져오는 경우가 많다.
```jsx
import Button from './Button';
```
하지만 실제 파일 이름은 `Button.tsx`다.

Webpack은 어떻게 `./Button`이 어떤 파일을 의미하는지 알 수 있을까?

이때 사용하는 설정이 `resolve`다. 그중 `resolve.extensions`는 확장자가 생략된 경로를 해석할 때 **시도할 확장자와 순서를 지정**한다.

총총에서는 다음과 같이 설정했다.

```js
resolve: {
  extensions: ['.ts', '.js', '.tsx'],
},
```

위 설정을 기준으로 파일 후보 탐색을 단순화하면 다음과 같다.

```js
./Button.ts
./Button.js
./Button.tsx
```

만약 `Button.ts`와 `Button.tsx`가 같은 위치에 있다면, `.ts`가 먼저 선택된다.
확장자를 생략했을 때 어떤 파일을 가져올지는 Webpack의 규칙에 따라 결정된다.

### 3. 여러 파일을 브라우저가 읽을 수 있도록 변환한다

파일을 찾았다면 그 내용을 처리해야 한다.

총총에서는 `.ts` `.tsx` 파일을 변환하기 위해 `babel-loader`를 사용한다.

```js
module: {
  rules: [
    {
      test: /\.(ts|tsx)$/,
      use: 'babel-loader',
      exclude: /node_modules/,
    },
  ],
},
```
  
이 설정을 문장으로 풀면 다음과 같다.

> `.ts` 또는 `.tsx`로 끝나는 파일은 `babel-loader`로 처리하되, `node_modules` 내부의 파일은 이 규칙에서 제외한다.

`test`는 규칙을 적용할 파일의 조건이다.
위 정규표현식은 파일 경로가 `.ts` 또는 `.tsx`로 끝나는지 확인한다.

`use`는 해당 파일을 처리할 로더를 지정한다.

`exclude`는 이 규칙에서 제외할 대상을 지정한다.

일반적으로 외부 패키지(node_modules)는 이미 변환을 거친 상태로 제공된다. 이런 코드까지 모두 같은 규칙으로 변환하면 불필요한 작업이 늘어날 수 있다.

그렇다면 Babel은 구체적으로 무엇을 변환할까?

총총에서는 다음 프리셋을 사용한다.

```js
presets: [
  ['@babel/preset-env', { targets: 'defaults, not dead' }],
  [
    '@babel/preset-react',
    {
      runtime: 'automatic',
      importSource: '@emotion/react',
    },
  ],
  '@babel/preset-typescript',
],
```

프리셋은 특정 목적에 필요한 Babel 플러그인을 묶어놓은 설정이다.

각 프리셋은 서로 다른 부분을 담당한다.

> @babel/preset-typescript : TypeScript 문법 변환
> @babel/preset-react : JSX 문법 변환
> @babel/preset-env : 대상 환경에 필요한 JavaScript 문법 변환 선택


먼저 다음 TypeScript 코드를 살펴보자.

```ts
const count: number = 3;
```

타입 표기는 개발 중 코드를 검사할 때 사용되며, 실행할 JavaScript에는 필요하지 않다.

이 코드는 다음과 같이 변환된다.

```js
const count = 3;
```

타입이 없어진 것을 확인할 수 있다. 

다음은 JSX다.

```jsx
const element = <h1>총총</h1>;
```

JSX는 JavaScript 안에서 UI 구조를 표현하기 위한 문법이다. 브라우저는 이 문법을 그대로 실행할 수 없으므로 JavaScript 함수 호출 형태로 바꿔야 한다.

```js
{
  runtime: 'automatic',
  importSource: '@emotion/react',
}
```

`runtime: 'automatic'`으로 JSX 변환에 필요한 런타임 함수를 자동으로 가져온다. `importSource`는 그 함수를 가져올 패키지를 지정한다.

총총은 Emotion을 사용하므로 `@emotion/react`를 지정했다.

마지막으로 `preset-env`는 대상 브라우저를 고려한다.

```js
['@babel/preset-env', { targets: 'defaults, not dead' }]
```

`targets`에 설정한 브라우저를 기준으로, 해당 브라우저에서 사용할 수 있도록 JavaScript 문법을 변환한다.

이렇게 정리하면 각 도구의 역할이 구분된다.

```
Webpack
→ 의존성을 분석하고 번들을 구성한다.

Babel
→ 소스 코드의 문법을 변환한다.

babel-loader
→ Webpack의 처리 과정에 Babel을 연결한다.
```

여기서 한 가지 더 확인해야 한다.

Babel이 TypeScript를 변환했다는 사실이 타입 검사까지 통과했다는 의미는 아니다.

다음 코드를 보자.

```ts
const count: number = 'hello';
```

number 타입으로 선언한 변수에 문자열을 넣었으므로 타입 오류가 발생한다.

하지만 Babel은 타입을 검사하지 않는다. 타입 표기가 제거되면 다음과 같이 된다.

```js
const count = 'hello';
```

이 코드는 JavaScript에서 정상이다. 따라서 Babel 변환만으로는 원래 코드에 있던 타입 오류를 발견하지 못한다.

이를 위해 타입 검사 명령을 별도로 제공한다.

```bash
pnpm type-check

```
이 명령은 TypeScript 컴파일러인 tsc를 실행한다.

실행용 코드의 변환과 번들 생성은 Babel과 Webpack이 담당하므로, TypeScript 컴파일러는 타입 검사에 집중하도록 역할을 나눈 것이다.

현재 총총에서는 CI에서 타입 검사를 별도 작업으로 수행한다. 빌드 명령 자체에 타입 검사가 포함된 구조는 아니므로, 번들 생성 성공과 타입 검사 성공은 각각 확인해야 한다.

### 4. 실행에 필요한 여러 코드를 번들링한다

```js
// webpack.local.js
output: {
  filename: 'bundle.js',
  chunkFilename: '[name].js',
},
```

위에서 `output`을 사용해 결과물을 만들었다.

여기서 **청크**라는 개념이 등장한다.

청크는 Webpack이 코드를 묶고 로딩하는 단위로 이해할 수 있다. JavaScript 청크는 출력 과정에서 별도의 파일로 만들어질 수 있다.

그렇다면 여러 파일을 묶기 위해 번들러를 사용하는데, 왜 다시 여러 개의 청크로 나눌까?

모든 코드를 하나의 파일로 만들면 첫 화면에서 필요하지 않은 코드까지 포함되기 때문이다.

예를 들어 랜딩 페이지를 보기 위해 다른 모든 페이지의 코드까지 받아야 한다면 초기 로딩에 불필요한 비용이 발생할 수 있다.

총총의 main.tsx에는 다음 코드가 있다.

```tsx
{
  path: '/',
  lazy: async () => ({
    Component: (await import('./src/features/landing/App')).default,
  }),
},
```

여기서 사용하는 동적 import()는 모듈을 비동기로 가져온다. Webpack은 이를 코드 분할 지점으로 사용해 별도 청크를 생성할 수 있다.

코드를 나누면 초기 다운로드를 줄일 수 있지만 추가 요청이 발생한다. 따라서 파일을 무조건 하나로 합치거나 최대한 잘게 나누는 것보다, 사용자가 어떤 코드를 언제 필요로 하는지 고려해야 한다.

운영 환경에서는 번들 파일 이름도 다르게 설정한다.

```js
// webpack.prod.js
output: {
  filename: '[name].[contenthash:16].js',
  chunkFilename: '[name].[contenthash:16].js',
},
```
  
예를 들어 다음과 같은 이름이 만들어질 수 있다.

```
main.a1b2c3d4e5f67890.js
```
  
`[name]`은 청크의 이름이고, `[contenthash:16]`은 내용에 기반한 해시를 16자리로 표현한다.

이 해시는 캐시 갱신과 관련이 있다.
브라우저가 이전에 다운로드한 파일을 재사용하면 네트워크 비용을 줄일 수 있다. 하지만 코드가 바뀌었다면 새로운 파일을 받아야 한다.
파일 내용이 달라질 때 이름도 달라지면 브라우저는 새로운 URL로 파일을 요청할 수 있다.

운영 환경에서는 `production` 모드, 로컬 환경에서는 `development` 모드를 사용한다.
mode는 단순한 환경 이름이 아니라 Webpack의 기본 최적화 설정에 영향을 준다.

이러한 차이를 관리하기 위해 설정을 나누었다.

```
설정 파일				용도
-----------------------------------------
webpack.common.js	환경에 공통으로 적용할 설정
webpack.local.js	로컬 개발 서버
webpack.dev.js		개발 환경 배포용 빌드
webpack.prod.js		운영 환경 배포용 빌드
```

공통 설정을 만든 뒤 `webpack-merge`로 환경별 설정을 합친다.

```js
export default merge(commonConfig, {
  mode: 'production',
  // 필요한 설정
});
```

이 구조에서는 공통으로 필요한 설정을 반복해서 작성하지 않고, 환경별로 달라지는 부분을 따로 관리할 수 있다.

### 5. HTML에서 생성된 번들을 불러오도록 연결한다

번들을 생성했다면 브라우저에서 해당 파일을 불러와야 한다.

예를 들어 JavaScript 번들의 이름이 bundle.js라면 HTML에 다음과 같은 연결이 필요하다.

```html
<script src="/bundle.js"></script>
```

하지만 `index.html`에는 번들을 가져오는 태그를 직접 작성하지 않고 `HtmlWebpackPlugin`을 사용할 수 있다.

```js
new HtmlWebpackPlugin({
  template: './index.html',
  filename: 'index.html',
  inject: true,
}),
```

`template`은 결과 HTML을 만들 때 사용할 원본 파일이고, `filename`은 생성할 HTML의 이름이다.

`inject: true`는 생성된 번들을 불러오는 태그를 HTML에 자동으로 추가하도록 한다.

앞서 운영 번들에는 내용에 따라 달라지는 해시를 파일 이름에 넣었다.

만약 HTML의 `<script>` 태그를 직접 관리한다면, 빌드 결과의 파일 이름이 바뀔 때마다 HTML도 수정해야 한다.

`HtmlWebpackPlugin`은 빌드 결과를 기준으로 태그를 생성하므로 이러한 작업을 자동화한다.

따라서 전체 흐름은 다음과 같다.

```
브라우저가 HTML을 받는다.
→ HTML에 연결된 JavaScript 번들을 가져온다.
→ 번들에 포함된 애플리케이션 코드가 실행된다.
→ React가 root 요소에 화면을 렌더링한다.
```

여기까지가 소스 코드에서 브라우저 화면까지 이어지는 기본 흐름이다.

### 개발 중에는 이 과정을 어떻게 반복할까

하지만 개발할 때는 파일을 계속 수정한다. 그때마다 빌드 명령을 실행하고 결과를 확인하는 작업은 번거롭다.

이를 위해 `webpack-dev-server`를 사용한다.

개발 서버는 파일 변경을 감지하고 재컴파일하면서 브라우저에 애플리케이션을 제공한다.

번들링 과정을 생략하는 것이 아니라, 개발자가 직접 반복하던 작업을 자동으로 수행하는 것이다.

```js
devServer: {
  port: 3005,
  open: true,
  hot: true,
  historyApiFallback: true,
  client: {
    overlay: true,
  },
},
```

`port`는 개발 서버의 포트, `open`은 실행 후 브라우저를 여는 설정이다. `client.overlay`는 컴파일 오류 등을 브라우저 화면에 표시하는 기능을 제어한다.

`hot: true`는 HMR을 활성화한다. HMR은 실행 중인 모듈을 교체할 수 있도록 하는 기능이다.

`historyApiFallback`은 `React Router`처럼 브라우저에서 경로를 처리하는 SPA와 관련이 있다.

예를 들어 애플리케이션 안에서 `/login`으로 이동하면 `React Router`가 로그인 화면을 표시할 수 있다.

반면 `/login`에서 새로고침하면 브라우저가 개발 서버에 해당 경로를 직접 요청한다.

```js
GET /login
```

서버가 `/login`이라는 파일만 찾는다면 404가 발생할 수 있다. React가 아직 실행되지 않았으므로 라우터도 화면을 선택할 수 없다.

`historyApiFallback: true`는 이러한 경로 요청에 `index.html`을 반환할 수 있도록 한다.

```
/login 요청
→ index.html 반환
→ 번들 실행
→ React Router가 /login 경로 처리
```

### 후기

`Webpack`을 직접 설정하면서 가장 크게 달라진 점은 **설정에 문제가 생겼을 때 어디를 확인해야 할지 구분할 수 있게 되었다는 것**이다.

이전에는 프로젝트가 정상적으로 실행되는지에만 집중했다. 오류가 발생하면 필요한 설정을 찾아 추가했지만, 그 설정이 어떤 과정에 영향을 주는지 설명하기는 어려웠다.

이번에는 진입점에서 의존성을 찾고, 소스를 변환하고, 번들을 생성해 HTML에 연결하는 과정을 하나씩 구성했다. 이를 통해 파일을 찾지 못하는 문제와 문법 변환 문제, 타입 오류가 서로 다른 단계에서 발생한다는 것을 이해했다.

또한 **작성한 코드와 브라우저에 전달되는 결과물은 다르다**는 점도 배웠다. 코드가 정상적으로 동작하는 것뿐 아니라 어떤 코드가 하나의 번들에 포함되는지, 언제 별도 청크를 요청하는지, 변경된 파일을 브라우저가 어떻게 다시 받아오는지도 살펴봐야 했다.

설정을 직접 관리하는 만큼 초기 구성이 오래걸렸다. 다만 앞으로 `Vite`처럼 기본 설정을 제공하는 도구를 사용하더라도, 문제가 발생한 단계를 먼저 파악하고 필요한 설정의 역할을 확인하며 접근할 수 있을 것이다.
