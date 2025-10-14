# 브라우저에서 보이지 않는 최적화

## 들어가며

웹 개발을 하다 보면 "왜 이 페이지는 이렇게 느릴까?"라는 질문을 자주 하게 됩니다. 단순히 코드를 작성하는 것을 넘어서 브라우저가 어떻게 동작하는지 이해하면 더 나은 최적화 전략을 세울 수 있습니다.

이 글은 프론트엔드 초급 개발자를 대상으로 작성하였습니다. 깊이 있는 기술 분석보다는 "이런 것이 있구나"를 이해하고, 실무에서 어떻게 적용할 수 있을지 생각해볼 수 있도록 구성했습니다.

## 1. 브라우저는 페이지를 어떻게 그릴까

### 1.1 렌더링의 기본 흐름

브라우저가 HTML 파일을 받으면 다음과 같은 과정을 거칩니다.

1. HTML을 읽어서 DOM 트리를 만듭니다
2. CSS를 읽어서 CSSOM을 만듭니다
3. DOM과 CSSOM을 합쳐서 렌더 트리를 구성합니다
4. 각 요소의 위치와 크기를 계산합니다 (레이아웃)
5. 화면에 그립니다 (페인트)

이 과정에서 중요한 점은 `<script>` 태그를 만나면 HTML 파싱이 멈춘다는 것입니다. JavaScript가 DOM을 조작할 수 있기 때문에 브라우저는 스크립트 실행이 끝날 때까지 기다려야 합니다.

```html
<head>
  <script src="heavy-script.js"></script>
  <!-- 여기서 파싱이 멈춤 -->
</head>
<body>
  <h1>안녕하세요</h1>
  <!-- 스크립트 로딩이 끝나야 보임 -->
</body>
```

### 1.2 왜 스크립트 위치가 중요한가

처음 웹 개발을 배울 때 "스크립트는 body 끝에 넣으세요"라는 말을 많이 듣습니다. 그 이유가 바로 이것입니다. HTML을 먼저 파싱해서 화면에 보여주고, 그 다음에 스크립트를 실행하는 것이 사용자 경험에 좋기 때문입니다.

하지만 매번 body 끝에 넣는 것만이 답은 아닙니다. `async`와 `defer` 속성을 사용하면 더 유연하게 제어할 수 있습니다.

```html
<!-- async: 다운로드 즉시 실행 (순서 보장 안됨) -->
<script src="analytics.js" async></script>

<!-- defer: HTML 파싱 끝나고 실행 (순서 보장됨) -->
<script src="main.js" defer></script>
```

저는 대부분의 경우 `defer`를 사용하는 것을 선호합니다. 스크립트 실행 순서가 보장되고, 사용자는 콘텐츠를 빨리 볼 수 있기 때문입니다.

## 2. 리소스 로딩 우선순위 제어하기

### 2.1 중요한 것부터 로드하기

모든 리소스를 동시에 다운로드할 수는 없습니다. 브라우저는 네트워크 대역폭이 제한적이기 때문에 우선순위를 정해서 로드합니다. 개발자는 이 우선순위를 조정할 수 있습니다.

**preload로 미리 가져오기**

```html
<link
  rel="preload"
  href="main-font.woff2"
  as="font"
  type="font/woff2"
  crossorigin
/>
```

preload는 "이 리소스를 최대한 빨리 다운로드하세요"라고 브라우저에게 알려줍니다. 저는 주로 폰트 파일에 사용합니다. 폰트는 CSS 파일 안에 숨어있어서 늦게 발견되는 경우가 많기 때문입니다.

**fetchpriority로 우선순위 명시하기**

```html
<img src="hero-banner.jpg" fetchpriority="high" alt="메인 배너" />
```

첫 화면에 보이는 중요한 이미지에는 `fetchpriority="high"`를 설정합니다. 반대로 덜 중요한 이미지는 `fetchpriority="low"`를 사용할 수 있습니다.

**loading="lazy"로 나중에 로드하기**

```html
<img src="footer-image.jpg" loading="lazy" alt="하단 이미지" />
```

페이지 하단에 있는 이미지는 바로 필요하지 않습니다. `loading="lazy"`를 사용하면 이미지가 화면에 가까워질 때만 다운로드됩니다. 이 기능을 사용하면 초기 로딩 시간이 크게 줄어듭니다.

저는 이 세 가지 기법을 조합해서 사용합니다. 중요한 리소스는 preload로 미리 가져오고, 화면 상단 이미지는 fetchpriority를 높이고, 하단 이미지는 lazy loading을 적용하는 식입니다.

### 2.2 LCP 개선하기

LCP(Largest Contentful Paint)는 "가장 큰 콘텐츠가 화면에 표시되는 시간"을 측정합니다. 구글은 2.5초 이내를 권장합니다.

제 경험상 LCP를 개선하는 가장 효과적인 방법은 다음과 같습니다.

1. **메인 이미지 최적화하기**

   - WebP 포맷 사용 (JPEG보다 30% 정도 작음)
   - 적절한 크기로 리사이징
   - fetchpriority="high" 설정

2. **서버 응답 속도 개선하기**

   - CDN 사용
   - 캐싱 적용

3. **불필요한 CSS 제거하기**
   - 사용하지 않는 스타일 정리
   - Critical CSS만 인라인으로 포함

실제로 프로젝트에서 메인 이미지를 2.5MB에서 180KB로 줄이고 fetchpriority를 적용했더니 LCP가 4.2초에서 1.8초로 개선된 적이 있습니다. 이미지 최적화만으로도 큰 효과를 볼 수 있습니다.

## 3. JavaScript 엔진은 코드를 어떻게 실행할까

### 3.1 단순히 한 줄씩 읽는 것이 아니다

JavaScript 엔진(V8, SpiderMonkey 등)은 코드를 그냥 읽는 것이 아니라 최적화 과정을 거칩니다.

1. 소스 코드를 파싱해서 AST(추상 구문 트리)로 만듭니다
2. AST를 바이트코드로 변환합니다
3. 인터프리터가 바이트코드를 실행합니다
4. 자주 실행되는 코드는 JIT 컴파일러가 최적화합니다

여기서 흥미로운 점은 4번 과정입니다. 엔진은 코드를 실행하면서 "이 함수는 자주 실행되네?"라고 판단하면, 그 함수를 더 빠르게 실행할 수 있도록 최적화합니다.

### 3.2 엔진이 좋아하는 코드 패턴

**일관된 타입 사용하기**

```javascript
function add(a, b) {
  return a + b;
}

add(1, 2); // 숫자 덧셈
add(5, 10); // 숫자 덧셈
add(100, 200); // 숫자 덧셈
```

엔진은 "아, 이 함수는 항상 숫자를 받는구나"라고 학습합니다. 그러면 타입 체크를 건너뛰고 바로 덧셈 연산을 수행하는 최적화된 코드를 생성합니다.

하지만 이렇게 하면 어떻게 될까요?

```javascript
add('hello', 'world'); // 문자열 연결
```

엔진은 "어? 타입이 바뀌었네?"라고 판단하고 최적화를 해제합니다. 다시 범용 코드를 사용해야 하므로 성능이 떨어집니다.

저는 이런 이유로 함수가 여러 타입을 처리하도록 만들지 않으려고 합니다. 타입별로 함수를 분리하는 것이 좋습니다.

**객체 구조를 일정하게 유지하기**

```javascript
// 엔진이 싫어하는 패턴
const user = { name: 'Alice' };
user.age = 25; // 나중에 속성 추가
delete user.name; // 속성 삭제

// 엔진이 좋아하는 패턴
const user = {
  name: 'Alice',
  age: 25,
};
```

엔진은 객체의 구조(shape)를 기억합니다. 같은 구조를 가진 객체들은 같은 최적화를 받습니다. 동적으로 속성을 추가하거나 삭제하면 구조가 바뀌어서 최적화가 깨집니다.

### 3.3 성능을 위한 간단한 팁

1. **반복문에서 불변 값은 밖으로 빼기**

```javascript
// 개선 전
for (let i = 0; i < array.length; i++) {
  process(array[i]);
}

// 개선 후
const length = array.length;
for (let i = 0; i < length; i++) {
  process(array[i]);
}
```

2. **작은 함수로 나누기**
   작은 함수는 엔진이 인라인 처리하기 쉽습니다. 함수 호출 오버헤드를 없앨 수 있습니다.

3. **try-catch는 필요한 곳에만**
   try-catch 블록은 최적화를 방해할 수 있습니다. 꼭 필요한 곳에만 사용하세요.

## 4. 메모리는 어떻게 관리될까

### 4.1 가비지 컬렉션의 기본 원리

JavaScript는 메모리를 자동으로 관리합니다. 가비지 컬렉터(GC)가 더 이상 사용되지 않는 메모리를 찾아서 해제합니다.

GC는 주로 Mark-Sweep-Compact 알고리즘을 사용합니다.

1. **Mark**: 사용 중인 객체를 표시합니다
2. **Sweep**: 표시되지 않은 객체를 제거합니다
3. **Compact**: 메모리를 정리해서 빈 공간을 모읍니다

```javascript
let obj = { name: 'Alice' };
obj = null; // 이제 { name: 'Alice' }를 참조하는 것이 없음
// GC가 다음 실행 때 이 메모리를 해제함
```

### 4.2 메모리 누수를 방지하는 방법

메모리 누수는 더 이상 필요하지 않은 객체가 여전히 참조되어 GC가 회수하지 못하는 상황입니다. 제가 실무에서 자주 보는 메모리 누수 패턴은 다음과 같습니다.

**이벤트 리스너를 정리하지 않는 경우**

```javascript
// 문제가 있는 코드
button.addEventListener('click', handleClick);
button.remove(); // 리스너가 메모리에 남아있음

// 올바른 코드
button.addEventListener('click', handleClick);
button.removeEventListener('click', handleClick);
button.remove();
```

**타이머를 정리하지 않는 경우**

```javascript
// 문제가 있는 코드
setInterval(() => {
  console.log('실행');
}, 1000); // 페이지를 닫기 전까지 계속 실행

// 올바른 코드
const timerId = setInterval(() => {
  console.log('실행');
}, 1000);

// 필요 없어지면
clearInterval(timerId);
```

저는 React를 사용할 때 useEffect의 cleanup 함수에서 이벤트 리스너와 타이머를 항상 정리합니다. 이것만으로도 대부분의 메모리 누수를 방지할 수 있습니다.

**WeakMap 활용하기**

일반 Map은 객체를 강하게 참조합니다. WeakMap은 약하게 참조해서 GC가 자유롭게 메모리를 회수할 수 있게 합니다.

```javascript
// 일반 Map
const map = new Map();
let obj = { data: 'value' };
map.set(obj, 'info');
obj = null; // 하지만 Map이 여전히 참조

// WeakMap
const weakMap = new WeakMap();
let obj2 = { data: 'value' };
weakMap.set(obj2, 'info');
obj2 = null; // GC가 회수 가능
```

WeakMap은 캐싱이나 DOM 요소에 메타데이터를 저장할 때 유용합니다.

## 5. 페이지 생명주기 이벤트

### 5.1 페이지를 떠날 때의 처리

사용자가 페이지를 닫거나 다른 페이지로 이동할 때 발생하는 이벤트를 잘 다뤄야 합니다.

**beforeunload: 경고 표시하기**

```javascript
window.addEventListener('beforeunload', (event) => {
  if (hasUnsavedChanges) {
    event.preventDefault();
    event.returnValue = ''; // 브라우저가 경고창을 표시
  }
});
```

저장되지 않은 변경사항이 있을 때 사용자에게 경고할 수 있습니다. 하지만 남용하면 사용자 경험을 해치므로 정말 필요한 경우에만 사용해야 합니다.

**pagehide: 안정적인 선택**

```javascript
window.addEventListener('pagehide', (event) => {
  if (event.persisted) {
    console.log('페이지가 bfcache에 저장됨');
  } else {
    console.log('페이지가 닫힘');
  }

  saveImportantData();
});
```

`pagehide`는 모든 브라우저에서 안정적으로 동작합니다. `unload` 이벤트보다 이것을 사용하는 것을 추천합니다.

### 5.2 CLS 문제 해결하기

CLS(Cumulative Layout Shift)는 페이지 로드 중 레이아웃이 예상치 못하게 움직이는 정도를 측정합니다. 사용자가 읽고 있던 내용이 갑자기 밀려나거나, 클릭하려던 버튼이 움직이면 짜증나는 경험을 하게 됩니다.

**이미지 크기를 미리 지정하기**

```html
<!-- 나쁜 예 -->
<img src="photo.jpg" alt="사진" />

<!-- 좋은 예 -->
<img src="photo.jpg" alt="사진" width="800" height="600" />
<!-- 또는 -->
<img src="photo.jpg" alt="사진" style="aspect-ratio: 16/9; width: 100%;" />
```

브라우저가 이미지 크기를 미리 알면 공간을 예약해두므로 레이아웃 이동이 발생하지 않습니다.

**동적 콘텐츠 위치 고려하기**

광고나 알림 같은 동적 콘텐츠는 기존 콘텐츠를 밀어내지 않도록 아래에 추가하거나, 미리 공간을 확보해야 합니다.

```javascript
// 나쁜 예: 콘텐츠 위에 광고 추가
container.prepend(ad);

// 좋은 예: 콘텐츠 아래에 광고 추가
container.append(ad);

// 또는 미리 공간 확보
adContainer.style.minHeight = '250px';
```

**애니메이션은 transform 사용하기**

```css
/* 나쁜 예: 레이아웃 변경 */
.box {
  transition: width 0.3s;
}
.box:hover {
  width: 200px;
}

/* 좋은 예: transform 사용 */
.box {
  transition: transform 0.3s;
}
.box:hover {
  transform: scaleX(1.2);
}
```

`width`, `height`, `top`, `left` 같은 속성을 애니메이션하면 레이아웃 재계산이 발생합니다. `transform`과 `opacity`만 사용하면 GPU 가속을 받아 부드럽게 동작합니다.

저는 프로젝트에서 이미지에 aspect-ratio를 설정하고, 광고는 항상 콘텐츠 아래에 배치하도록 규칙을 정했습니다. 이것만으로도 CLS 점수가 크게 개선되었습니다.

## 6. 실전에서 적용하기

### 6.1 우선순위를 정하기

모든 최적화를 한 번에 적용할 수는 없습니다. 저는 다음 순서로 진행합니다.

1. **성능 측정하기**

   - Lighthouse로 현재 상태 확인
   - Core Web Vitals 점수 확인 (LCP, FID, CLS)

2. **가장 큰 문제부터 해결하기**

   - LCP가 느리면 → 이미지 최적화
   - CLS가 높으면 → 이미지 크기 지정, 레이아웃 안정화
   - JavaScript가 무거우면 → 코드 스플리팅

3. **측정하고 개선하고 반복하기**
   - 변경 후 다시 측정
   - 효과가 있는지 확인
   - 다음 문제로 이동

### 6.2 간단한 체크리스트

실무에서 자주 사용하는 체크리스트입니다.

**이미지 최적화**

- [ ] WebP 포맷 사용
- [ ] 적절한 크기로 리사이징
- [ ] 중요 이미지에 fetchpriority="high" 설정
- [ ] 하단 이미지에 loading="lazy" 적용
- [ ] width와 height 또는 aspect-ratio 지정

**스크립트 최적화**

- [ ] 스크립트에 defer 또는 async 적용
- [ ] 불필요한 라이브러리 제거
- [ ] 코드 스플리팅 적용

**메모리 관리**

- [ ] 이벤트 리스너 정리
- [ ] 타이머 정리
- [ ] 전역 변수 최소화

### 6.3 완벽함보다 개선을 목표로

처음부터 완벽한 성능을 만들 수는 없습니다. 저도 처음에는 Lighthouse 점수가 50점대였던 프로젝트를 점진적으로 개선해서 90점대로 올렸습니다.

중요한 것은 작은 개선이라도 꾸준히 하는 것입니다. 이미지 하나를 최적화하고, 스크립트 하나에 defer를 추가하고, 불필요한 이벤트 리스너 하나를 정리하는 것. 이런 작은 개선들이 모여서 큰 차이를 만듭니다.

## 마치며

웹 성능 최적화는 한 번에 끝나는 작업이 아닙니다. 새로운 기능이 추가되고, 라이브러리가 업데이트되고, 사용자 환경이 변하면서 지속적으로 관리해야 합니다.

하지만 브라우저와 JavaScript 엔진이 어떻게 동작하는지 이해하면, 문제가 생겼을 때 어디를 봐야 할지 알 수 있습니다. "왜 느린가?"보다 "어디가 병목인가?"를 찾을 수 있게 됩니다.

이 문서에서 다룬 내용은 제가 실무에서 경험하고 학습한 것을 바탕으로 정리한 것입니다. 모든 상황에 정답이 있는 것은 아니지만, 여러분의 프로젝트에서 적용해볼 수 있는 출발점이 되기를 바랍니다.

작은 개선부터 시작하세요. 이미지 하나를 최적화하고, 스크립트 하나에 defer를 추가하는 것만으로도 사용자 경험은 개선됩니다. 그리고 그 작은 개선들이 모여서 큰 차이를 만듭니다.
