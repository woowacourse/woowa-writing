## moment 서비스를 위한 최적의 Webpack 개발 환경 구축기
안녕하세요! moment 서비스 개발자입니다. 저희 팀은 사용자들이 하루의 소중한 순간을 감성적으로 공유하는 서비스를 만들고 있습니다. 이번 글에서는 저희 서비스의 특성에 맞춰 Webpack 기반의 프론트엔드 개발 환경을 어떻게 구축하고 최적화했는지 그 여정을 공유하고자 합니다.

<img width="1440" height="779" alt="image" src="https://github.com/user-attachments/assets/d5a8cd9b-1ca9-4660-ac90-2844d64e075b" />


### 서론: moment 서비스를 위한 최적의 개발 환경을 찾아서
'moment'는 사용자의 감성을 자극하는 이미지와 짧은 글을 공유하는 서비스입니다. 따라서 사용자에게 빠르고 안정적인 로딩 속도와 매끄러운 인터랙션을 제공하는 것이 무엇보다 중요했습니다. 이러한 기술적 우선순위를 달성하기 위해 저희는 개발 환경부터 탄탄하게 구축하기로 했습니다.

프로젝트 초기, 저희에게는 Webpack을 사용해야 한다는 기술적 요구사항이 있었습니다. 사실 요즘 많은 프로젝트가 Vite와 같이 개발 서버 구동이 빠르고 설정이 간편한 도구를 선호하는 추세입니다. 그래서 처음에는 Webpack이라는 제약이 조금 아쉽게 느껴지기도 했습니다. 하지만 주어진 환경 안에서 최고의 개발 경험과 결과물을 만들어내는 것 또한 개발자의 중요한 역량이라고 생각했습니다. 그래서 우리는 Webpack을 깊이 있게 이해하고, 이를 적극적으로 활용하여 우리 서비스에 최적화된 개발 환경을 구축하는 여정을 시작했습니다.

### Webpack 깊이 이해하기
본격적인 환경 구축에 앞서, 우리는 왜 Webpack을 사용해야만 했는지 그 본질을 이해하는 것부터 시작했습니다. 이를 위해 최근 많이 사용되는 Vite, Parcel과 같은 다른 번들러들과 Webpack을 비교하며 핵심 철학과 장단점을 분석했습니다.

**Vite, Parcel과의 비교를 통해 본 Webpack의 핵심 철학**
- Vite: Vite는 개발 환경에서 네이티브 ES 모듈(ESM)을 사용하여 소스 코드를 브라우저에 직접 제공합니다. 이 덕분에 개발 서버 구동 속도가 매우 빠르다는 장점이 있습니다. 하지만 프로덕션 빌드 시에는 Rollup을 사용하여 번들링하므로, 개발 환경과 프로덕션 환경 간에 미묘한 차이가 발생할 수 있습니다.

- Parcel: Parcel은 'zero-config'를 지향하는 번들러로, 별도의 설정 파일 없이도 바로 사용할 수 있어 진입 장벽이 매우 낮습니다. 파일들을 분석하여 필요한 변환과 플러그인을 자동으로 적용해주기 때문에 매우 편리하지만, 복잡하고 세밀한 설정을 하기에는 Webpack에 비해 유연성이 떨어집니다.

- Webpack: Webpack의 핵심 철학은 **"모든 것을 모듈로 본다"**는 것입니다. JavaScript 파일뿐만 아니라 CSS, 이미지, 폰트 등 모든 자원을 모듈로 관리하고, 이들 간의 의존성을 파악하여 하나의 (또는 여러 개의) 결과물로 합쳐줍니다. 이 과정에서 로더(Loader)와 플러그인(Plugin)을 통해 다양한 자원을 원하는 대로 가공할 수 있는 강력한 확장성과 세밀한 제어 능력을 제공합니다. 이는 복잡한 요구사항을 가진 대규모 프로젝트에서 빛을 발하는 장점입니다.

이러한 비교 분석을 통해 저희는 Webpack이 주어진 이유가 바로 이 **'높은 자유도와 확장성'**에 있음을 깨달았습니다. 우리 서비스가 성장함에 따라 발생할 수 있는 다양한 기술적 요구사항에 유연하게 대응하기 위해서는 Webpack의 강력한 생태계와 세밀한 설정 기능이 필수적이었던 것입니다. 우리는 이 장점을 최대한 활용하여 우리만의 최적화된 개발 환경을 만들기로 했습니다.

### 왜 Babel을 선택했는지
최신 JavaScript 문법과 TypeScript를 모든 브라우저에서 문제없이 실행하기 위해서는 트랜스파일러(Transpiler)가 필수적입니다. 최근에는 Rust 기반의 SWC나 esbuild처럼 매우 빠른 속도를 자랑하는 도구들이 주목받고 있습니다. 저희 역시 빌드 속도 향상을 위해 이들을 검토했지만, 최종적으로는 Babel을 선택했습니다. 그 이유는 다음과 같습니다.

안정성과 생태계: Babel은 오랜 기간 수많은 프로젝트에서 검증된, 가장 성숙하고 안정적인 트랜스파일러입니다. 방대한 플러그인 생태계를 갖추고 있어 필요한 기능을 쉽게 확장할 수 있으며, 특히 저희가 사용하고자 했던 Emotion 라이브러리와의 호환성이 뛰어났습니다. babel-plugin-emotion은 CSS-in-JS의 개발 경험을 크게 향상시키는 핵심적인 플러그인이었습니다.

세밀한 폴리필(Polyfill) 관리: @babel/preset-env는 core-js와 함께 사용하여 타겟 브라우저에 필요한 폴리필만 선별적으로 추가해주는 강력한 기능을 제공합니다. 이를 통해 불필요한 코드의 양을 줄여 최종 번들 크기를 최적화할 수 있었습니다. 이는 사용자에게 빠른 로딩 속도를 제공해야 하는 저희 서비스에 매우 중요한 요소였습니다.

팀의 학습 곡선: 팀원 대부분이 Babel 사용 경험이 있었기 때문에, 새로운 도구를 학습하는 데 드는 시간을 절약하고 빠르게 개발에 집중할 수 있다는 현실적인 장점도 고려했습니다.

물론 Babel의 트랜스파일링 속도가 다른 최신 도구에 비해 느리다는 단점은 명확합니다. 하지만 저희는 현재 프로젝트 규모에서는 빌드 속도가 개발 경험을 심각하게 저해할 정도는 아니라고 판단했으며, 안정성, 확장성, 그리고 번들 최적화 측면에서 Babel이 제공하는 이점이 더 크다고 결론 내렸습니다.

### 서비스 맞춤형 환경 설정
앞서 내린 결정과 분석을 바탕으로, 저희는 webpack.common.js, webpack.dev.js, webpack.prod.js 세 개의 파일로 설정을 분리하여 관리했습니다. 각 파일의 핵심 코드를 통해 저희 서비스의 요구사항이 어떻게 반영되었는지 보여드리겠습니다.

1. 공통 설정 (webpack.common.js)
이 파일에는 개발 환경과 프로덕션 환경 모두에 적용되는 핵심 설정들이 담겨 있습니다.
```javascript
// webpack.common.js

// ... (import 구문 생략)

const config = {
  entry: './src/index.tsx', // 진입점 설정
  resolve: {
    extensions: ['.ts', '.tsx', '.js'], // 확장자 자동 해석
    alias: {
      '@': path.resolve(__dirname, 'src'), // 경로 별칭 설정
    },
    // ...
  },
  module: {
    rules: [
      {
        test: /\.(ts|tsx)$/,
        exclude: /node_modules/,
        use: 'babel-loader', // TypeScript/React 코드를 Babel로 트랜스파일
      },
      {
        test: /\.css$/,
        use: [
          // 프로덕션 환경에서는 CSS 파일을 분리하고, 개발 환경에서는 style-loader 사용
          process.env.NODE_ENV === 'production' ? MiniCssExtractPlugin.loader : 'style-loader',
          'css-loader',
        ],
      },
      {
        test: /\.(png|jpe?g|gif|svg|webp)$/i,
        type: 'asset', // 8KB 미만 이미지는 inline, 초과 시 파일로 분리
        parser: {
          dataUrlCondition: {
            maxSize: 8 * 1024,
          },
        },
        // ...
      },
    ],
  },
  plugins: [
    new HtmlWebpackPlugin({ template: './public/index.html' }), // HTML 자동 생성
    new ForkTsCheckerWebpackPlugin(), // 타입스크립트 타입 체크를 별도 프로세스에서 수행하여 빌드 속도 향상
    new webpack.DefinePlugin({
      // 환경 변수 주입
      'process.env.REACT_APP_BASE_URL': JSON.stringify(/* ... */),
      // ...
    }),
  ],
};

export default config;
```

- `babel-loader`: TypeScript와 JSX 문법을 JavaScript로 변환하는 역할을 합니다.

- `이미지 처리`: asset 모듈 타입을 사용하여 작은 이미지는 데이터 URL로 변환하여 네트워크 요청을 줄이고, 큰 이미지는 파일로 분리하여 캐싱 효율을 높였습니다. 이는 이미지 중심의 'moment' 서비스에서 로딩 성능을 최적화하는 데 중요한 설정입니다.

- `ForkTsCheckerWebpackPlugin`: TypeScript의 타입 검사와 Webpack의 번들링을 별개의 프로세스에서 동시에 진행시켜 전체 빌드 시간을 단축했습니다.

2. 개발 환경 설정 (webpack.dev.js)
개발 환경에서는 빠른 피드백과 디버깅의 편의성에 중점을 두었습니다.
```javascript
// webpack.dev.js

// ...

export default merge(common, {
  mode: 'development',
  devtool: 'eval-source-map', // 빠르고 상세한 소스맵 제공
  output: {
    // ...
    publicPath: '/',
  },
  devServer: {
    port: 3000,
    open: true, // 서버 실행 시 브라우저 자동 열기
    hot: true, // HMR(Hot Module Replacement) 활성화
    historyApiFallback: true, // SPA 라우팅을 위한 설정
    // ...
  },
  plugins: [
    // 필요 시 번들 분석 도구 활성화
    ...(process.env.ANALYZE_BUNDLE === 'true' ? [new BundleAnalyzerPlugin()] : []),
  ],
});
```

- `devServer.hot`: HMR(Hot Module Replacement) 기능은 코드를 수정했을 때 전체 페이지를 새로고침하지 않고 변경된 모듈만 교체하여 상태를 유지한 채로 빠르게 변경 사항을 확인할 수 있게 해줍니다. 이는 개발 생산성을 극대화하는 핵심 기능입니다.

- `devtool`: 'eval-source-map': 빠르고 정확한 소스맵을 생성하여 디버깅을 용이하게 합니다.

3. 프로덕션 환경 설정 (webpack.prod.js)
프로덕션 환경의 목표는 단 하나, 최적화입니다. 사용자에게 가장 빠르고 가벼운 결과물을 제공하기 위해 다양한 플러그인과 설정을 추가했습니다.
```javascript
// webpack.prod.js

// ...

export default merge(common, {
  mode: 'production',
  output: {
    filename: 'js/[name].[contenthash].js', // 콘텐츠 변경 시에만 파일 이름이 바뀌도록 하여 캐싱 효율 극대화
    chunkFilename: 'js/[name].[contenthash].chunk.js',
    path: path.resolve(__dirname, 'dist'),
    publicPath: '/',
    clean: true, // 빌드 시 이전 결과물 삭제
  },
  devtool: false,
  optimization: {
    minimize: true,
    minimizer: [
      new TerserPlugin({ // JavaScript 코드 난독화 및 압축
        terserOptions: {
          compress: {
            drop_console: true, // console.log 제거
          },
        },
        // ...
      }),
      new CssMinimizerPlugin(), // CSS 코드 압축
    ],
    splitChunks: { // 코드 스플리팅
      chunks: 'all',
      // ... (캐시 그룹 설정)
    },
    runtimeChunk: 'single',
  },
  plugins: [
    new MiniCssExtractPlugin({ // CSS 파일을 별도로 추출
      filename: 'css/[name].[contenthash].css',
    }),
    new CompressionPlugin({ // 파일을 gzip으로 압축
      algorithm: 'gzip',
      // ...
    }),
  ],
});
```
- `output.filename`: [contenthash]를 사용하여 파일 내용이 변경될 때만 해시값이 바뀌도록 했습니다. 이를 통해 사용자는 변경되지 않은 파일을 다시 다운로드할 필요 없이 브라우저 캐시를 최대한 활용할 수 있습니다.

- `optimization.minimizer`: TerserPlugin과 CssMinimizerPlugin을 사용하여 JavaScript와 CSS 코드에서 불필요한 공백, 주석 등을 제거하고 변수명을 줄여 파일 크기를 최소화했습니다.

- `optimization.splitChunks`: 공통으로 사용되는 라이브러리(예: react, react-dom)를 'vendor' 청크로 분리하고, 여러 곳에서 중복으로 사용되는 코드를 별도의 파일로 추출하여 초기 로딩에 필요한 코드의 양을 줄였습니다.

- `CompressionPlugin`: 빌드 결과물을 gzip으로 압축하여 서버에서 클라이언트로 전송되는 데이터의 크기를 획기적으로 줄였습니다.

### 결론: 좋은 개발 환경이란 무엇인가
이번 프로젝트를 통해 Webpack 환경을 직접 구성하고 최적화하면서, '좋은 개발 환경'이란 단순히 최신 기술을 사용하거나 빌드 속도가 빠른 것만을 의미하지 않는다는 것을 깨달았습니다.

좋은 개발 환경이란, 우리 서비스의 특성과 기술적 목표를 명확히 이해하고, 주어진 도구의 철학을 깊이 파고들어 그 장점을 극대화하며, 마주한 문제를 능동적으로 해결해 나가는 과정 그 자체에서 만들어지는 것이었습니다. Webpack이라는 '제약'은 오히려 저희에게 번들링의 원리를 더 깊게 학습하고, 웹 성능 최적화에 대해 진지하게 고민할 기회를 주었습니다.

물론 현재의 구성이 완벽하지는 않습니다. 프로젝트 규모가 더 커지면 Babel의 빌드 속도가 발목을 잡을 수도 있고, 더 효율적인 코드 스플리팅 전략이 필요해질 수도 있습니다. 앞으로 서비스가 성장함에 따라 빌드 캐시를 도입하거나, SWC와 같은 빠른 트랜스파일러로 점진적으로 전환하는 등의 개선 방향을 계속해서 고민하고 적용해 나갈 것입니다.

이번 경험은 저에게 기술을 수동적으로 받아들이는 것이 아니라, 그 본질을 이해하고 능동적으로 활용하는 개발자로 성장하는 소중한 밑거름이 되었습니다. 이 글이 Webpack 환경을 구축하며 고민하고 있는 다른 개발자분들께 작은 도움이 되기를 바랍니다.
