# Webpack 기반의 리액트, 타입스크립트 프로젝트 초기 개발 환경 구축

> 레벨 3, 4 팀 프로젝트 기간 동안 Webpack 기반으로 `리뷰미` 서비스의 초기 개발 환경을 구축한 경험을 녹여낸 글입니다.

## 1. Webpack 기반 프로젝트 설정 배경

<img src='./images/webpack.png'>

### Webpack의 역할

webpack은 모듈 번들러로, 프로젝트 내의 다양한 자원(HTML, CSS, JavaScript, 이미지, 폰트 등)을 하나의 번들로 묶어줍니다. 프론트엔드 개발에서 사용하는 다양한 파일 형식은 브라우저가 직접 이해할 수 없는 경우가 많기 때문에, 이들을 적절히 변환해서 브라우저가 이해할 수 있는 번들의 형태로 제공하는 것이 핵심 기능입니다.

### Vite, CRA 등 다른 도구와의 차별점

Webpack 이외에도 Vite, CRA, Parcel 등 여러 빌드 도구들이 있습니다. 그 중에서도 Vite는 ES 모듈(JavaScript의 공식 표준 모듈 시스템)을 기반으로 하여 빠른 개발 서버와 빠른 빌드 속도를 자랑합니다. 그럼에도 Webpack을 사용한 이유는 다음과 같습니다.

**1. 프로젝트 요구 사항**

- 팀 프로젝트 요구 사항에 따라 Webpack 이외의 빌드 도구 및 보일러 플레이트는 제한됩니다.
- Vite 등 다른 도구를 도입하는 경우가 늘고 있으나, 아직까지는 프로덕션에서 Webpack 기반으로 구성된 프로젝트가 많습니다. 따라서 Webpack을 사용해 빌드 환경을 직접 구성했습니다. 

**2. 확장성**

- Webpack은 다양한 플러그인과 로더를 지원하기 때문에 변화하는 요구사항에 빠르게 대처할 수 있습니다.

**3. 넓은 생태계와 문서화**

- Webpack은 2012년부터 프론트엔드 개발의 주요 번들러로 자리 잡아왔습니다. 커뮤니티 지원이 강력하며, 공식 문서가 잘 정리되어 있어 신뢰할 수 있는 자료가 풍부합니다.

## 2. Webpack 5 업데이트와 주요 기능

2020년 10월에 Webpack 5가 출시된 이후로 여러 가지 성능 향상과 새로운 기능이 추가되었습니다. 특히 새로운 기능들은 번들 크기 축소와 빌드 속도 향상에 중점을 두고 있습니다. 주요 업데이트 및 기능들은 다음과 같습니다.

- `모듈 연합(Module Federation)`: 서로 다른 Webpack 빌드들이 독립적인 번들로 존재하면서 동적으로 모듈 공유가 가능해졌습니다.
- 빌드 결과를 영구적으로 캐싱해서 재빌드 시간이 대폭 단축되었습니다.
- 트리 쉐이킹(사용되지 않는 코드를 제거하여 최종 번들 크기를 줄이는 최적화 기법)이 개선되었습니다.

## 3. 세팅하기에 앞서

가장 먼저 프로젝트에 관련된 정보와 패키지를 관리하는 `package.json`을 생성합니다. 리뷰미에서는 패키지 관리자로 yarn을 채택했기 때문에 이 글에서는 모두 yarn 명령어를 사용하겠습니다.

```bash
yarn init -y
```

`yarn init` 명령어를 실행하면 프로젝트 이름, 버전, 설명 등을 물어보며 사용자가 입력한 값으로 프로젝트를 초기화합니다. 하지만 뒤에 `-y` 옵션을 붙여 모든 질문에 일일이 답하지 않고 자동으로 기본값이 입력된 파일로 초기화했습니다.

그 결과, 다음과 같이 파일이 생성되었습니다.

```json
{
  "name": "frontend",
  "version": "1.0.0",
  "main": "index.js",
  "license": "MIT"
}
```

## 4. Webpack 세팅

`package.json` 파일을 생성했고, 다음으로는 Webpack을 설치하겠습니다.

다음 순서인 리액트와 타입스크립트를 Webpack보다 먼저 설치해도 차이는 없습니다. 하지만 설정의 편의성과 논리적인 흐름을 고려할 때, Webpack을 먼저 설정하는 것이 바람직합니다. 프로젝트의 빌드 도구와 환경을 먼저 설정한 뒤에 사용할 라이브러리를 얹는 것이 더 자연스럽기 때문입니다.

### 설치

```bash
yarn add -D webpack webpack-dev-server webpack-cli
```

설치한 라이브러리의 역할은 다음과 같습니다.

- `webpack`: 모듈 번들러입니다.
- `webpack-dev-server`: 개발 서버를 실행하여 코드 변경 사항을 실시간으로 브라우저에 반영해주는 도구입니다. HMR(Hot Module Replacement, 코드가 수정될 때 전체 페이지를 새로고침하지 않고 변경된 부분만을 업데이트하는 기능)을 지원합니다.
- `webpack-cli`: Webpack을 커맨드라인 인터페이스(CLI)에서 사용할 수 있게 해주는 도구입니다. Webpack은 자체적으로 CLI를 제공하지 않기 때문에, 해당 라이브러리를 함께 설치해야 명령어를 터미널에서 실행할 수 있습니다.

### 설정 파일 작성

Webpack 설정 파일인 `webpack.config.js`를 작성했습니다. 각각의 설정이 무엇을 의미하는지 주석으로 설명하겠습니다.

```js
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');
const StylelintPlugin = require('stylelint-webpack-plugin');
const { CleanWebpackPlugin } = require('clean-webpack-plugin');

module.exports = (env, argv) => {
  const isProduction = argv.mode === 'production'; // 배포 환경인지 개발 환경인지 판단

  return {
    mode: isProduction ? 'production' : 'development', // 배포 환경인 경우 Webpack 기본 제공 플러그인 실행
    entry: './src/index.tsx', // 번들링을 시작할 최초 진입점 설정
    output: {
      path: path.join(__dirname, 'dist'), // 번들 파일이 저장될 디렉토리
      filename: 'bundle.js',
      clean: true, // 빌드할 때마다 기존의 번들 파일을 삭제해서 폴더가 정리되도록 설정
      publicPath: '/',
    },
    // Webpack이 파일을 해석할 때 사용할 확장자와 경로 별칭을 설정하는 부분
    resolve: {
      extensions: ['.ts', '.tsx', '.js'],
      alias: {
        '@': path.resolve(__dirname, './src'), // src 폴더 내부에서 절대 경로 사용
      },
    },
    module: {
      // Webpack이 파일을 처리하는 방식(로더 설정)을 정의하는 부분
      rules: [
        {
          test: /\.(ts|tsx|js)$/,
          exclude: /node_modules/, // 파일 처리에서 제외할 경로
          use: [
            {
              loader: 'babel-loader', // 최신 자바스크립트 코드를 브라우저에 맞게 변환하는 babel-loader
            },
          ],
        },
        {
          test: /\.(png|jpeg|jpg)$/,
          type: 'asset/resource',
        },
      ],
    },
    // Webpack이 번들링 과정에서 사용하는 플러그인 설정
    plugins: [
      // HTML 파일에 JavaScript 번들을 자동으로 주입하는 플러그인
      new HtmlWebpackPlugin({
        template: './public/index.html',
        minify: isProduction // 배포 모드에서만 html을 최적화하도록 설정
          ? {
              collapseWhitespace: true,
              removeComments: true,
            }
          : false,
        hash: true,
      }),
    ],
    devtool: isProduction ? 'hidden-source-map' : 'eval', // 배포 환경에서는 소스 코드 노출 최소화를 위해 hidden-source-map, 개발 환경에서는 빠른 디버깅을 위해 eval 방식 사용
    devServer: {
      historyApiFallback: true, // 개발 환경에서 404 에러가 발생하는 경우 index.html로 이동하도록 설정
      port: 3000, // 프로젝트를 실행시킬 localhost 포트 번호
      hot: true, // HMR 활성화
    },
  };
};
```

프로젝트 환경이 리액트, 타입스크립트로 확정되었기 때문에 리액트, 타입스크립트 관련 라이브러리가 설치되어 있다는 가정 하에 구성했습니다.

**output.clean과 CleanWebpackPlugin의 차이점**

모두 Webpack의 출력 디렉토리를 정리하는 데 사용됩니다.
Webpack 5 이전에는 output.clean을 지원하지 않았기 때문에 CleanWebpackPlugin을 사용해서 빌드 전에 수동으로 디렉토리를 정리했습니다.
더 세밀하게 디렉토리를 정리하고 싶은 경우에는 Webpack 5에서도 사용 가능하지만, 리뷰미에서는 output.clean으로도 충분하다 판단하여 사용하지 않았습니다.

## 5. 리액트 세팅

리액트와 관련 라이브러리를 설치합니다.

### 설치

```bash
yarn add react react-dom react-router-dom
```

각 라이브러리의 역할은 다음과 같습니다.

- `react-dom`: 리액트 컴포넌트를 브라우저의 실제 DOM에 렌더링합니다.
- `react-router-dom`: 클라이언트 측 라우팅을 구현합니다. 리뷰미에서는 라우터를 사용하기 때문에 함께 설치했습니다.

## 6. 타입스크립트 세팅

타입스크립트와 관련 라이브러리를 설치합니다.

### 설치

```bash
yarn add -D typescript ts-loader @types/react @types/react-dom
```

각 라이브러리의 역할은 다음과 같습니다.

- `typescript`: 타입스크립트 컴파일러입니다. 타입스크립트로 작성된 코드를 자바스크립트로 변환합니다.
- `ts-loader`: 타입스크립트 파일을 Webpack에서 번들링할 수 있게 해주는 로더입니다.
- `@types/react`: 리액트 라이브러리에 대한 타입 정의 파일입니다. 타입스크립트에서 리액트 컴포넌트와 관련된 타입을 정의하고 검사할 수 있게 해줍니다.
- `@types/react-dom`: `react-dom` 라이브러리에 대한 타입 정의 파일입니다. 리액트 컴포넌트가 DOM에 렌더링되는 과정에서 필요한 타입을 제공합니다.

### 설정 파일 작성

타입스크립트 설정 파일인 `tsconfig.json`을 작성했습니다. 각각의 설정이 무엇을 의미하는지 주석으로 설명하겠습니다.

```json
{
  "compilerOptions": {
    "target": "es6",
    "lib": ["dom", "dom.iterable", "esnext"],
    "esModuleInterop": true, // 모듈 간의 호환성 유지를 도움
    "module": "esnext", // 타입스크립트가 컴파일할 때 모듈 시스템, esnext는 ES 모듈을 사용
    "moduleResolution": "node", // Node.js 방식의 모듈 해석 방식 지정
    "resolveJsonModule": true, // JSON 파일을 타입스크립트에서 import할 수 있게 허용
    "forceConsistentCasingInFileNames": true, // 파일과 디렉터리 이름에서 대소문자 불일치 오류 검출
    "strict": true, // 타입스크립트의 엄격한 타입 검사 모드 활성화
    "skipLibCheck": true,
    "jsx": "react-jsx",
    "outDir": "./dist", // 컴파일된 파일이 저장될 출력 디렉터리
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"] // 절대 경로 사용
    }
  },
  "include": ["src"],
  "exclude": ["node_modules", "dist"]
}
```

## 7. Emotion CSS 세팅

리뷰미는 CSS-in-JS 방식의 `Emotion CSS`를 사용합니다.

### 설치

```bash
yarn add @emotion/react
yarn add @emotion/styled
```

각 라이브러리의 역할은 다음과 같습니다.

- `emotion`: 핵심 CSS-in-JS 기능을 제공하며, css 함수와 CSS prop을 사용하여 동적 스타일링을 할 수 있게 해줍니다.
- `emotion/styled`: styled API를 사용해 재사용 가능한 styled 컴포넌트를 정의하고, 컴포넌트 단위로 스타일링을 할 수 있게 해줍니다.

두 패키지를 모두 사용해서 기본적으로는 styled 컴포넌트로 스타일링하되, 필요한 경우 css를 사용해 동적으로 스타일링합니다.

### 설정 파일 작성

emotion 설치 이후에, `tsconfig.json`에 다음을 추가합니다. Emotion 라이브러리와 JSX를 함께 사용할 때 필요한 설정입니다.

```json
// tsconfig.json
{
  // ...
  "jsxImportSource": "@emotion/react"
  // ...
}
```

### 트러블슈팅

Emotion 라이브러리를 사용하는 컴포넌트마다 `/** @jsxImportSource @emotion/react */` pragma(전처리 명령)를 파일 최상단에 입력해야 한다는 불편함이 있었습니다.

해당 전처리 명령은 JSX 코드를 변환할 때, Babel 트랜스파일러에게 React의 `createElement`함수가 아닌 Emotion의 `jsx` 함수를 사용하라고 알려주는 역할을 합니다. Babel 설정 파일을 수정하여 Emotion을 자동으로 처리하도록 구성했습니다.

먼저 필요한 Babel 패키지를 설치합니다.

```bash
yarn add -D @babel/core @babel/preset-env @babel/preset-react @babel/preset-typescript babel-loader
```

`babel.config.json`을 작성합니다.

```json
{
  "presets": [
    "@babel/preset-env",
    ["@babel/preset-react", { "runtime": "automatic", "importSource": "@emotion/react" }],
    "@babel/preset-typescript"
  ],
  "plugins": ["@emotion"]
}
```

pragma를 매번 입력하지 않아도 Emotion을 사용할 수 있게 되었습니다.

## 8. ESLint, Prettier 세팅

ESLint는 코드의 문법 오류를 찾아내고, 일관성 있는 코드 스타일을 유지하는 도구입니다.

Prettier는 코드의 일관성을 유지하고, 개발자 간의 스타일 차이를 줄이는 도구입니다. 여러 명의 개발자가 함께 작업하기 때문에 사용했습니다.

ESLint 관련 플러그인 설정 과정에서 패키지끼리 버전이 호환되지 않는 문제가 발생했습니다. 호환성을 위해 ESLint 라이브러리의 버전을 9에서 8로 다운그레이드 했습니다. 아래의 설정 파일은 ESLint 버전 8 기준으로 작성되었습니다.

### 설치

```bash
yarn add -D eslint @typescript-eslint/parser @typescript-eslint/eslint-plugin eslint-plugin-react eslint-plugin-react-hooks eslint-plugin-import eslint-plugin-react-refresh
```

```bash
yarn add -D prettier eslint-config-prettier eslint-plugin-prettier
```

각 라이브러리의 역할은 다음과 같습니다.

- `eslint`: 코드의 스타일을 일관되게 유지하고, 잠재적인 오류를 찾는 린팅 도구입니다.
- `@typescript-eslint/parser`: 타입스크립트 코드를 분석하고, ESLint가 타입스크립트 문법을 이해할 수 있도록 도와주는 파서(parser)입니다.
- `@typescript-eslint/eslint-plugin`: 타입스크립트 코드에 대해 구체적인 ESLint 규칙을 정의하고 적용할 수 있게 해주는 플러그인입니다.
- `eslint-plugin-react`: 리액트 코드에 대해 권장되는 스타일과 규칙을 적용할 수 있도록 도와주는 리액트 전용 ESLint 플러그인입니다.
- `eslint-plugin-react-hooks`: 리액트 훅(Hooks) 사용 시, 권장되는 규칙을 강제하여 리액트 훅의 올바른 사용을 보장하는 플러그인입니다.
- `eslint-plugin-import`: 모듈 import/export 문에 대해 일관된 코드 스타일과 최적화를 지원하는 플러그인입니다.
- `eslint-plugin-react-refresh`: 리액트의 Fast Refresh 기능을 지원하는 ESLint 플러그인으로, 주로 개발 환경에서 코드 변경 시 컴포넌트를 빠르게 업데이트할 수 있도록 도와줍니다.

### 설정 파일 작성

ESLint 설정 파일인 `.eslintrc.cjs`를 작성합니다.

```cjs
module.exports = {
  root: true,
  env: { browser: true, es2020: true },
  extends: [
    'eslint:recommended',
    'plugin:@typescript-eslint/recommended',
    'plugin:react-hooks/recommended',
    'plugin:import/recommended',
    'plugin:react/recommended',
    'prettier',
  ],
  ignorePatterns: ['dist', '.eslintrc.cjs', 'webpack.config.js'], // ESLint가 검사하지 않을 파일이나 폴더
  parser: '@typescript-eslint/parser',
  plugins: ['react-refresh'],
  rules: {
    'react/react-in-jsx-scope': 'off',
    'react/no-unknown-property': ['error', { ignore: ['css'] }],
    'import/order': [
      // import 순서를 강제하여 모듈을 가져오는 순서를 명확하게 정의
      'error',
      {
        'newlines-between': 'always',
        groups: [['builtin', 'external'], 'internal', 'parent', 'sibling', 'index'],
        pathGroups: [
          {
            pattern: 'next',
            group: 'builtin',
          },
          {
            pattern: 'react',
            group: 'builtin',
          },
          {
            pattern: '@MyDesignSystem/**',
            group: 'internal',
          },
          {
            pattern: 'src/**',
            group: 'internal',
          },
        ],
        pathGroupsExcludedImportTypes: ['src/**', '@MyDesignSystem/**'],
        alphabetize: {
          order: 'asc',
          caseInsensitive: true,
        },
      },
    ],
  },
  settings: {
    'import/resolver': {
      // import 해석 규칙 정의
      node: {
        extensions: ['.js', '.jsx', '.ts', '.tsx'],
      },
    },
  },
};
```

prettier의 설정 파일인 `.prettierrc`도 작성합니다.

```json
{
  "endOfLine": "auto",
  "singleQuote": true,
  "semi": true,
  "useTabs": false,
  "tabWidth": 2,
  "trailingComma": "all", // 함수 인수, 배열, 객체 리터럴의 마지막 요소 뒤에 콤마 추가
  "printWidth": 120,
  "bracketSpacing": true,
  "arrowParens": "always", // 화살표 함수의 매개변수에 괄호 강제
  "prefer-const": true // 재할당이 없는 변수를 선언할 때 let 대신 const 사용
}
```

## 9. StyleLint 세팅

CSS-in-JS 환경에서 스타일 코드를 자동으로 포맷팅하기 위해 `prettier-plugin-css-order`를 고려했습니다. 하지만 prettier 최신 버전에서는 postcss 오류가 발생하여 작동하지 않았습니다. 대안으로 Stylelint를 적용하여 이 문제를 해결했습니다.

StyleLint는 CSS 코드 스타일을 검사하고, 일관성 있는 CSS 포맷팅 규칙을 적용할 수 있는 도구입니다.

### 설치

```bash
yarn add -D stylelint stylelint-order stylelint-config-clean-order @stylelint/postcss-css-in-js
```

각 라이브러리의 역할은 다음과 같습니다.

- `stylelint-order`: CSS 속성의 순서를 강제하는 플러그인입니다. 함께 설치한 clean-order 설정을 기반으로 하게 됩니다.
- `stylelint-config-clean-order`: `stylelint-order` 플러그인과 함께 사용되는 속성 순서 규칙입니다.
- `@stylelint/postcss-css-in-js`: CSS-in-JS 코드에서 Stylelint를 적용할 수 있도록 해줍니다. Emotion에서도 동작합니다.

### 설정 파일 작성

```json
{
  "extends": ["stylelint-config-clean-order"],
  "plugins": ["stylelint-order"],
  "customSyntax": "@stylelint/postcss-css-in-js", // CSS-in-JS 스타일을 검사할 수 있게 설정
  "rules": {
    "declaration-empty-line-before": [
      "never", // CSS 선언 사이에 빈 줄을 허용하지 않음
      {
        "ignore": ["after-declaration"] // 다른 선언 뒤에 빈 줄 무시
      }
    ]
  }
}
```

이제 스크립트를 실행하여 코드에 StyleLint를 적용할 수 있게 되었습니다. 만약 코드를 저장할 때마다 자동으로 적용되게 하고 싶다면 다음과 같이 VSCode 추가 설정을 할 수 있습니다.

1. VSCode에서 StyleLint 확장 프로그램을 설치합니다.
2. `.vscode/setting.json`에서 다음의 설정을 추가합니다.

```json
{
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": true,
    "source.fixAll.stylelint": true
  },
  "stylelint.validate": ["css", "scss", "javascript", "typescript", "javascriptreact", "typescriptreact"]
}
```

이제 타입스크립트로 작성된 스타일 파일도 자동으로 포맷팅 할 수 있게 되었습니다.

## 10. MSW 세팅

백엔드에서 API를 제공하기 전까지 프론트엔드 단에서 개발을 이어나가기 위해 MSW를 설치합니다.

### 설치

```bash
yarn add -D msw
```

## 11. 테스트 환경 세팅

유틸 함수와 리액트 훅을 테스트하기 위해 `jest`와 `RTL`(React Testing Library)를 설치합니다.

### 설치

```bash
yarn add -D jest @types/jest jest-environment-jsdom
```

```bash
yarn add -D @testing-library/react @testing-library/jest-dom @testing-library/user-event @testing-library/dom
```

각 라이브러리의 역할은 다음과 같습니다.

- `jest`: 자바스크립트 테스트 프레임워크입니다.
- `@types/jest`: Jest API의 타입스크립트 타입을 정의합니다.
- `jest-environment-jsdom`: 브라우저 환경에서 DOM을 테스트할 수 있는 jsdom 환경을 제공합니다.
- `@testing-library/react`: 리액트 컴포넌트의 렌더링 및 동작을 테스트합니다.
- `@testing-library/jest-dom`: DOM 관련 matcher를 추가하여 DOM 상태를 쉽게 테스트할 수 있게 합니다.
- `@testing-library/user-event`: 클릭, 입력과 같은 사용자 상호작용을 시뮬레이션 할 수 있게 합니다.
- `@testing-library/dom`: DOM 조작을 위한 쿼리와 DOM 상태를 테스트합니다.

## 12. 시작 파일 작성

마지막으로 시작 파일을 작성해서 개발 환경이 잘 구축되었는지 확인해보겠습니다.
먼저, 클라이언트가 직접 접근할 수 있는 public 폴더를 만들고 `index.html`을 작성합니다.

```html
<!DOCTYPE html>
<html lang="ko">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>리뷰미</title>
  </head>
  <body>
    <div id="root"></div>
  </body>
</html>
```

다음으로 src 폴더를 만들고 `App.tsx`와 `index.tsx`를 작성합니다.

```tsx
const App = () => {
  return <div>안녕하세요, 리뷰미 입니다.</div>;
};

export default App;
```

```tsx
import App from '@/App';
import React from 'react';
import ReactDOM from 'react-dom/client';

const root = ReactDOM.createRoot(document.getElementById('root') as HTMLElement);
root.render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
```

스크립트를 실행하여 개발 서버가 정상적으로 동작하는 것을 확인할 수 있습니다.

## 마치며

지금까지 리뷰미 팀의 프론트엔드 초기 개발 환경 구축 과정을 소개했습니다. Webpack 기반으로 개발 환경을 구축해야 하는 개발자 분들에게 이 글이 도움이 되었으면 좋겠습니다.

## 참고

https://gong-check.github.io/dev-blog/FE/%EC%98%A8%EC%8A%A4%ED%83%80/emotion%20%EC%A0%81%EC%9A%A9%EA%B8%B0/
