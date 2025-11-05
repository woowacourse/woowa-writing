# 성능 최적화, 프론트엔드 개발에서 어떻게 접근할 것인가

## “성능 최적화”란 무엇일까 - 프론트엔드 관점에서 본 정의

프론트엔드에서 말하는 성능은 로딩 성능과 렌더링 성능으로 구성됩니다.

로딩 성능은 사용자가 요청한 페이지가 얼마나 빨리 화면에 나타나는지를,
렌더링 성능은 화면이 표시된 후 인터랙션이나 애니메이션이 얼마나 부드럽게 동작하는지를 의미합니다.

즉, 사용자가 첫 화면을 얼마나 빠르게 볼 수 있는지, 사용성이 얼마나 매끄럽게 이루어지는지가
프론트엔드 성능을 결정짓는 주요 요소입니다.

결국 최적화란 이러한 두 성능 요소를 개선하여
사용자가 더 빠르고 쾌적하게 웹 서비스를 이용할 수 있도록 만드는 과정이라 할 수 있습니다.

## 성능 최적화가 필요한 이유

아래의 결과는 2017년 Think with Google에서
페이지 로딩 속도에 따라 사용자가 이탈할 확률이 얼마나 달라지는지를 조사한 결과입니다.

![alt text](image.png)

연구에 따르면, 페이지 로딩 시간이 1초에서 3초로 늘어날 경우
사용자가 페이지를 이탈할 확률은 32% 증가합니다.

1초에서 5초로 늘어나면 90%, 1초에서 10초로 늘어나면 123% 증가합니다.

이는 로딩 속도가 늘어날수록 사용자의 서비스 이용 의지가 떨어진다는 것을 보여줍니다.

일상적으로 생각해보아도, 유튜브 첫 화면이 5초 이상 걸린다면
대부분의 사용자는 느리다고 인식할 것입니다.

즉, 이러한 사용자의 체감 속도는 실제 서비스 유지율에 직접적인 영향을 미칩니다.

따라서 사용자에게 빠르고 쾌적한 경험을 제공하기 위해서는
로딩 성능의 최적화가 사용자 유지 측면에서 매우 중요합니다.

## 그렇다면 어떤 부분부터 최적화를 진행하면 되는걸까?

처음 최적화를 시도한다면 “도대체 어디부터 손대야 하지?”라는 생각이 들 수 있습니다.
이를 도와주는 도구가 바로 Lighthouse입니다.

Lighthouse는 구글이 개발한 자동화 성능 측정 도구로,
웹사이트의 성능을 점수화하고 개선 방향을 구체적으로 제안해줍니다.
크롬 개발자 도구(DevTools)에서 바로 실행할 수 있습니다.

아래는 Lighthouse가 보여주는 Performance 리포트 예시입니다.

여기서 우리가 집중해서 볼 부분은 Performance 탭입니다.

이는 실제 사용자가 페이지를 사용할 때의 체감 성능을 수치화한 결과입니다.

![alt text](image-1.png)

#### Core Web Vitals

구글의 Core Web Vitals는 웹 페이지의 “사용자 경험 품질”을 수치로 표현한 세 가지 핵심 지표입니다.

- LCP (Largest Contentful Paint) ≤ 2.5초 — 주요 콘텐츠가 표시되는 속도를 나타내고 로딩 성능을 평가합니다.

- INP (Interaction to Next Paint) ≤ 200ms — 사용자의 입력 후 화면이 반응하는 속도를 나타내고 상호작용 성능을 평가합니다.

- CLS (Cumulative Layout Shift) ≤ 0.1 — 예기치 않은 화면 이동 정도를 나타내고 시각적 안정성을 평가합니다.

이 세 가지를 기준으로 페이지의 전반적인 사용자 경험 품질을 평가할 수 있습니다.
즉, 이 지표들이 낮을수록 “사용자가 체감하는 빠름” 이 향상된다는 뜻입니다.

#### Lighthouse의 결과 해석하기

Lighthouse는 성능 점수 외에도 두 가지 섹션을 제공합니다.

- Insights (개선 포인트)

→ “무엇을 고치면 성능이 향상되는가?”를 정량적으로 보여줍니다.

예: 이미지 크기 최적화, 불필요한 JS 제거 등

- Diagnostics (진단 정보)

→ “왜 느린가?”를 데이터 기반으로 분석합니다.

예: 메인 스레드 차단 시간, Third-party 리소스 비중, DOM 크기 등

따라서 최적화를 진행할 때는

1️⃣ Insights로 우선순위를 정하고

2️⃣ Diagnostics로 원인을 파악하는 흐름으로 접근하는 것이 효과적입니다.

## 그렇다면 최적화 어떻게 해볼 수 있을까?

앞서 살펴본 것처럼 웹의 성능은 크게 로딩 성능(Loading Performance) 과 렌더링 성능(Rendering Performance) 으로 나눌 수 있습니다.

로딩 성능은 “얼마나 빨리 화면을 볼 수 있는가”,
렌더링 성능은 “화면을 본 이후 얼마나 부드럽게 반응하는가”의 문제입니다.

이 두 성능은 서로 독립적이면서도, 사용자 경험 전체를 구성하는 한 몸입니다.

로딩이 빠르더라도 인터랙션이 끊기면 사용자는 불편함을 느끼고,
렌더링이 아무리 부드러워도 첫 화면이 늦게 나타난다면 “느린 서비스”로 인식합니다.

이번 섹션에서는 이 두 성능을 각각 개념적으로 분석하고,
실제 프로젝트에서 어떻게 적용할 수 있는지를 함께 살펴보겠습니다.

### 1) 로딩 성능 최적화

목표: “사용자가 첫 화면을 보기까지 걸리는 시간”을 단축하기

로딩 과정에서 브라우저는 다음 단계를 거칩니다.

1️⃣ HTML 파싱 → 2️⃣ CSS 파싱 및 렌더트리 생성 → 3️⃣ JS 파싱 및 실행 → 4️⃣ 이미지 등 리소스 로드 → 5️⃣ 첫 화면 렌더링.

이 중 어느 하나라도 지연되면 사용자에게 “빈 화면”이 더 오래 보이게 됩니다.

즉, 로딩 성능 최적화는 이 Critical Rendering Path(핵심 렌더링 경로) 를 단축하는 과정입니다.

그 핵심은 리소스의 크기, 수량, 우선순위를 조정하는 데 있습니다.

**(1) 이미지 최적화**

이미지는 대부분의 웹 페이지에서 가장 큰 네트워크 부하를 차지합니다.

실제로 구글의 HTTP Archive에 따르면, 웹 페이지 리소스 중 약 60~70%가 이미지입니다.

따라서 이미지 최적화는 가장 직접적이면서 체감 효과가 큰 최적화 포인트입니다.

① 이미지 포맷 최적화

JPEG나 PNG 대신 WebP, AVIF 같은 차세대 포맷을 사용하면
같은 화질을 유지하면서 파일 크기를 30~70% 줄일 수 있습니다.

프로젝트에서는 Webpack 빌드 단계에서
image-minimizer-webpack-plugin￼
을 활용하여 자동 변환 및 압축을 수행할 수 있습니다.

```typescript
new ImageMinimizerPlugin({
  generator: [
    {
      implementation: ImageMinimizerPlugin.sharpGenerate,
      type: 'asset',
      filter: (_, sourcePath) => /\.png$/i.test(sourcePath),
      filename: 'static/[name].webp',
      options: {
        encodeOptions: { webp: { quality: 40 } },
        resize: { width: 1920 },
      },
    },
  ],
});
```

이 설정은 PNG 파일을 WebP로 변환하고, 동시에 리사이즈 및 품질 압축을 수행합니다.

즉, “이미지를 업로드할 때마다 개발자가 수동으로 압축하지 않아도 되는 자동화된 빌드 파이프라인”이 완성됩니다.

② 이미지 로딩 우선순위 관리

이미지가 많을수록 브라우저는 네트워크 요청을 병렬로 수행하더라도 시간이 걸립니다.

따라서 “무엇을 먼저 보여줄 것인가” 를 명시하는 것이 중요합니다.

- 주요 콘텐츠(LCP 이미지)는 <link rel="preload"> 또는 fetchpriority="high" 속성으로 우선 로드
- 뷰포트 밖 이미지는 <img loading="lazy"> 속성으로 지연 로드(lazy-loading)
- 시각적으로 작은 아이콘은 SVG나 CSS Sprite로 대체

이렇게 하면 사용자가 가장 먼저 봐야 하는 이미지를 신속하게 보여주면서,
불필요한 리소스 요청을 뒤로 미뤄 네트워크 혼잡을 줄일 수 있습니다.

**(2) 리소스 코드 전송 최적화**

웹 애플리케이션은 HTML·CSS·JS뿐 아니라 다양한 번들 파일로 구성됩니다.

이 중 자바스크립트(JS)는 실행 전 파싱·컴파일·실행 과정이 필요하기 때문에
전송량이 많을수록 렌더링 차단(blocking)이 발생합니다.

① 번들 크기 줄이기

Webpack이나 Vite 같은 번들러는 mode: 'production' 설정 시
자동으로 코드 압축(Terser) 과 트리 셰이킹(Tree Shaking) 을 수행합니다.

- Terser → 공백·주석 제거, 식별자 단축, 사용되지 않는 코드 제거
- Tree Shaking → 실제로 import되지 않은 모듈 자동 제거

여기에 추가로 gzip / brotli 압축을 적용하면
네트워크 전송량을 최대 70%까지 감소시킬 수 있습니다.

이러한 압축은 대부분의 CDN에서 자동으로 지원하지만,
Nginx나 CloudFront 등 서버 설정을 직접 다룰 때도 쉽게 적용 가능합니다.

② 코드 스플리팅 (Code Splitting)

코드 스플리팅은 “필요한 시점에 필요한 코드만 불러오기” 전략입니다.

초기 번들에 모든 페이지 코드를 포함하는 대신,
라우트 단위로 분리하여 사용자가 접근할 때마다 로드하는 방식입니다.

React에서는 React.lazy와 Suspense로 간단히 구현할 수 있습니다.

```typescript
const Search = lazy(() => import('./pages/Search/Search'));

<Routes>
  <Route path="/" element={<Home />} />
  <Route
    path="/search"
    element={
      <Suspense fallback={<div>Loading...</div>}>
        <Search />
      </Suspense>
    }
  />
</Routes>;
```

이렇게 분리하면 초기 번들 크기가 줄고,
LCP(Largest Contentful Paint) 개선 효과를 직접적으로 얻을 수 있습니다.

**(3) 폰트 전략**

폰트는 종종 간과되지만, 실제로 텍스트 표시 지연(FOIT)과 레이아웃 점프(CLS) 의 주요 원인입니다.

또한 정적 리소스(JS/CSS/이미지/폰트)의 장기 캐싱은 재방문 속도에 큰 영향을 미칩니다.

아래는 프로젝트에서 바로 적용할 수 있는 구체적인 방법입니다.

핵심 목표는 두 가지입니다.

1. 첫 텍스트 표시 시점 단축: 폰트 로딩을 기다리지 않고 먼저 읽히게 만들기
2. 시각적 안정성 확보(CLS 방지): 폴백 폰트와 실제 폰트의 메트릭을 맞춰 레이아웃 점프 최소화

구체적으로는 다음 순서를 권장합니다.

① WOFF2 서브셋(Subset) + self-host

- 사용 글리프만 포함한 서브셋 폰트 생성(영문/숫자/기호/국문 분리 가능) → 파일 크기 급감
- CDN/자체 서버에 self-host(Google Fonts URL 의존 줄이고 캐시/정책을 우리가 통제)

예시) PretendardSubsetKR.woff2, PretendardSubsetLatin.woff2 로 분리

② Preconnect & Preload로 폰트 우선순위 올리기

```html
<!-- 폰트를 같은 도메인에서 서비스한다면 생략 가능 -->
<link rel="preconnect" href="https://static.example-cdn.com" crossorigin />

<link
  rel="preload"
  href="https://static.example-cdn.com/fonts/PretendardSubsetKR.woff2"
  as="font"
  type="font/woff2"
  crossorigin
/>
```

- preconnect는 TCP/TLS 핸드셰이크를 앞당겨 첫 바이트까지의 지연을 줄여줍니다.

- preload는 해당 폰트가 반드시 필요함을 명시해 우선 로드하게 합니다.

### 2) 렌더링 성능 최적화

목표: “화면이 그려진 이후, 얼마나 부드럽게 반응하는가” 개선하기

렌더링 성능은 브라우저의 메인 스레드가 얼마나 효율적으로 작동하는가에 달려 있습니다.

브라우저는 한 프레임을 그릴 때마다 다음 과정을 수행합니다.

1️⃣ 자바스크립트 실행
2️⃣ 스타일 계산 (Recalculate Style)
3️⃣ 레이아웃 계산 (Reflow)
4️⃣ 페인트 (Paint)
5️⃣ 합성 (Compositing)

이 중 하나라도 지연되면 프레임 드롭이 발생해 화면이 ‘끊기는’ 듯한 느낌을 줍니다.

(1) 애니메이션 최적화

애니메이션은 사용자의 체감 품질에 큰 영향을 줍니다.

그러나 잘못 구현된 애니메이션은 오히려 렌더링 병목을 일으킬 수 있습니다.

① Reflow / Repaint 최소화

- Reflow는 DOM 구조나 크기, 위치가 변경될 때 전체 레이아웃을 다시 계산하는 과정입니다.
- Repaint는 색상이나 그림자 등의 시각적 속성만 바뀔 때 발생합니다.

Reflow는 비용이 훨씬 크므로, 가능하면 DOM 구조 변경을 최소화해야 합니다.

```typescript
// 🔴 반복문 안에서 매번 DOM에 접근하고 스타일 변경
const items = document.querySelectorAll('.list-item');

for (let i = 0; i < items.length; i++) {
  items[i].style.width = items[i].offsetWidth + 10 + 'px'; // offsetWidth 접근 시 즉시 reflow 발생
  items[i].style.marginLeft = i * 5 + 'px';
}
```

문제점

- offsetWidth를 읽는 시점에 브라우저는 최신 레이아웃 정보를 알아야 합니다.
  -> 이때 즉시 Reflow(레이아웃 재계산) 이 발생하며, 모든 요소의 배치를 다시 계산하게 됩니다.

- 이어서 스타일이 변경되면, 브라우저는 다시 레이아웃을 검증해야 합니다.
  -> 이 과정이 layout → style → layout → style 형태로 반복되면서
  프레임마다 연속적인 Reflow가 발생하고, 결과적으로 CPU 사용량이 증가하며 화면이 끊겨 보이는 현상이 나타납니다.

```typescript
const items = document.querySelectorAll('.list-item');

// 🟢 DOM 접근은 한 번에 묶기 (읽기 → 계산 → 쓰기 순서로 분리)
const widths = Array.from(items).map((item) => item.offsetWidth);

for (let i = 0; i < items.length; i++) {
  const newWidth = widths[i] + 10;
  // 🟢 스타일 변경은 메모리 상에서만 조작 후 한 번에 반영
  items[i].style.transform = `translateX(${i * 5}px)`;
  items[i].style.width = `${newWidth}px`;
}
```

개선 포인트

- DOM 읽기(offsetWidth)와 쓰기(style)를 분리하여,
  “읽기 단계 → 계산 단계 → 쓰기 단계” 순으로 명확히 구분합니다.
  이렇게 하면 브라우저가 중간에 불필요하게 레이아웃을 다시 계산하지 않아
  Reflow 발생 빈도를 크게 줄일 수 있습니다.

- 또한, 위치나 크기 변경이 필요한 경우
  top, left와 같은 레이아웃 관련 속성 대신 transform 속성을 사용하는 것이 좋습니다.
  transform은 GPU 레벨에서 처리되므로 레이아웃 계산에 영향을 주지 않아
  부드러운 애니메이션과 안정적인 렌더링 성능을 확보할 수 있습니다.

② GPU 가속 활용

transform과 opacity 속성은 GPU 합성 레이어에서 처리됩니다.

즉, CPU 기반의 레이아웃 계산을 거치지 않아도 되기 때문에 부드러운 애니메이션을 만들 수 있습니다.

```css
.element {
  transform: translateY(20px);
  will-change: transform;
}
```

여기서 will-change는 브라우저에게 “이 속성이 곧 바뀔 것”임을 미리 알려
GPU 레이어를 사전 생성하게 함으로써 첫 프레임 지연을 줄여줍니다.

단, 너무 많은 요소에 적용하면 오히려 메모리 사용량이 증가할 수 있으므로
정말 필요한 애니메이션 요소에만 사용해야 합니다.

(2) 불필요한 리렌더링 최소화 (React 기준)

React의 렌더링 병목은 대부분 상태 변경이 불필요하게 많은 컴포넌트에서 발생합니다.

즉, 데이터는 변하지 않았는데 상위 컴포넌트가 리렌더링되면서
자식들도 연쇄적으로 다시 그려지는 경우입니다.

이를 줄이기 위한 전략은 다음과 같습니다.

① 컴포넌트 분리

하나의 큰 컴포넌트가 많은 상태를 관리하면, 작은 변화에도 전체가 리렌더링됩니다.

상태가 달라지는 부분을 작은 단위로 쪼개고, 각 컴포넌트가 자신에게 필요한 상태만 구독하도록 구조를 분리합니다.

② 메모이제이션 (Memoization)

- React.memo → props가 바뀌지 않으면 리렌더링 방지
- useMemo, useCallback → 계산 비용이 큰 값이나 함수를 캐싱

단, 모든 컴포넌트에 적용하면 오히려 메모리 사용량이 늘고 유지보수가 복잡해질 수 있습니다.

따라서 렌더링 비용이 크거나 자주 리렌더링되는 컴포넌트에만 선택적으로 적용하는 것이 좋습니다.

③ 전역 상태 최소화

전역 Context는 하위 모든 컴포넌트를 리렌더링시킬 수 있습니다.

useSyncExternalStore나 zustand 같은 외부 스토어를 활용하여
필요한 데이터만 구독하게 하면, 불필요한 렌더링을 획기적으로 줄일 수 있습니다.

(3) 메인 스레드 부하 줄이기

렌더링 성능은 결국 메인 스레드가 얼마나 자주 쉬는가에 달려 있습니다.

JS 연산이 길어지면 렌더링이 차단되어 사용자는 “버벅임”을 느낍니다.

이런 경우 다음과 같은 방식으로 부하를 분산시킬 수 있습니다.

- Web Worker: 무거운 연산(예: 데이터 파싱, 이미지 변환)을 별도 스레드로 이동
- requestIdleCallback: 사용자의 인터랙션이 없는 틈새 시간에 비필수 작업 수행
- Debounce / Throttle: 스크롤, 입력 이벤트를 일정 주기로 묶어 처리

## 마무리하며

성능 최적화는 단순히 “점수 올리기”가 아니라,
사용자가 체감하는 경험의 품질을 높이는 일입니다.

빠른 로딩과 부드러운 상호작용은 결국 사용자의 신뢰와 만족도로 이어집니다.

따라서 성능 최적화는 개발 과정의 마지막 단계가 아니라,
기획·디자인·개발 전 과정에 걸친 사용자 중심 사고로 접근해야 합니다.

## 참고 자료

- [프론트엔드 최적화? 토스에서는 이렇게 합니다!](https://toss.tech/article/firesidechat_frontend_9)
- [Think With Google - Mobile Page Speed](https://www.thinkwithgoogle.com/intl/en-emea/marketing-strategies/app-and-mobile/find-out-how-you-stack-new-industry-benchmarks-mobile-page-speed/)
- [Web.dev - Rendering Performance](https://web.dev/articles/rendering-performance?hl=ko)
- [Web.dev - Optimize Web Fonts](https://web.dev/articles/optimize-webfont-loading?hl=ko)
- [Web Vitals](https://web.dev/articles/vitals?hl=ko)
- [MDN Web Docs - will-change](https://developer.mozilla.org/en-US/docs/Web/CSS/will-change)
- [Webpack - Image Minimizer Plugin](https://github.com/webpack-contrib/image-minimizer-webpack-plugin)
- [Google Fonts Knowledge - Font performance](https://fonts.google.com/knowledge)
