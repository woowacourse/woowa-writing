# Lighthouse 자동화 스크립트 구현하기(LightHouse Node CLI/module, Puppeteer)

>  **Note**

> 이 글은 Lighthouse v12.2.1 기준으로 하고 있습니다.

> node기반 Lighthouse는 크롬에서의 Lighthouse와 측정 값이 다를 수 있습니다. 대부분의 점수 차이는 크롬 Lighthouse에서 제공하는 쓰로틀링 기능의 오해에서 발생합니다.  **크롬의 네트워크와 성능 탭에서 성능 제한을 두더라도 lighthouse 측정 결과에 반영되지 않습니다.** 자세한 사항은 본문을 참고하세요.

## LightHouse란

Lighthouse는 웹페이지 품질을 개선하는 데 사용할 수 있는 오픈소스 자동화 도구입니다. 공개 웹페이지 또는 인증이 필요한 웹페이지에서 실행할 수 있습니다. 

크롬의 DevTools에서 Lighthouse 탭을 통해 해당 도구를 사용할 수 있습니다. chrome의 버전에 따라 점검 가능한 항목이 다르지만, Lighthouse를 통해 사이트의 성능, 접근성, 프로그레시브 웹 앱(PWA), 검색엔진 최적화(SEO) 등을 점검할 수 있습니다.

![image.png](image.png)

Chrome Devtools에 있는 Lighthouse는 크롬과 url 주소만 있으면 많은 사이트들의 지표나 특이사항을 확인할 수 있습니다.

하지만 Chrome Devtools의 Lighthouse는 한 가지 단점이 존재합니다. 바로 측정하고 싶은 페이지가 여러 개일 경우 일일이 모든 사이트에 들어가 하나하나 측정해야 한다는 점입니다.

이런 단점은 Lighthouse CLI와 Node module을 이용한 자동화를 통해 해결할 수 있습니다.

## Lighthouse Node CLI/module

### Lighthouse Node CLI란

Lighthouse Node CLI는 Lighthouse를 쉘 스크립트를 통해 사용할 수 있도록 만들어진 CLI(명령줄 인터페이스)입니다.

해당 인터페이스를 통해 특정 사이트의 성능 점수를 확인할 수 있습니다.

Lighthouse CLI를 사용하기 위해 설치를 먼저 해줍니다.

>  **Note**

> Lighthouse v12.2.1 기준 Node 18 LTS 이상 버전을 요구합니다.

> 아래 명령어는 개발 의존성을 가진 플래그(-D)를 사용합니다. 필요에 따라 다른 플래그를 사용하셔도 무방합니다.

```jsx
// npm
npm install -D lighthouse
// yarn
yarn add -D lighthouse
```

CLI의 기본 사용법은 다음과 같습니다.

```jsx
lighthouse <url> <options>
```

`<url>` 에는 확인하고 싶은 url을, `<options>` 에는 사용하고 싶은 옵션을 넣으면 됩니다. 이때 `<url>` 은 필수입니다.

### CLI 설정

CLI가 동작하는데에 있어 필요한 설정을 추가할 수 있습니다.

우선 빈 설정 파일을 만들어줍니다.

```jsx
// .lighthouserc.js
export default {
  extends: 'lighthouse:default',
  settings: {},
};

```

해당 파일을 참고해 CLI를 호출합니다.

```jsx
lighthouse https://www.google.com/ --config-path=./lighthouse-config.js
```

해당 설정 파일에서 export하고 있는 객체를 바꿔주면 추가적인 설정을 할 수 있습니다. 이 글에서 다루지 않는 속성들은 [Lighthouse Configuration](https://github.com/GoogleChrome/lighthouse/blob/main/docs/configuration.md)에서 확인하실 수 있습니다.

#### extends

확장할 설정을 정합니다. 유효한 값은 'lighthouse:default'뿐입니다.

`lighthouse:default` 의 설정을 확인하기 위해서는  [default-config.js](https://github.com/GoogleChrome/lighthouse/blob/main/core/config/default-config.js)에서 확인하실 수 있습니다.

이 설정이 없으면 일일이 모든 `artifacts` , `audits` 등 원하는 옵션 설정을 다 해주어야 하기 때문에, 무조건 이 설정을 넣는 것을 추천드립니다.

#### settings

`locale` : 결과물의 언어를 설정합니다. (영어의 경우 ‘en-US’, 한국어의 경우 ‘ko’, 기본값은 영어)

`throttlingMethod`:네트워크 쓰로틀링 방법을 설정할 수 있습니다.(기본값은 `simulate`)

- `simulate` ,`devtools` , `provided`
- `provided` 는 네트워크 쓰로틀링을 없앲니다.
- `simulate` ,`devtools` 의 차이는 [Network Throttling](https://github.com/GoogleChrome/lighthouse/blob/main/docs/throttling.md)에서 확인하세요.

`throttling` : 쓰로틀링 정도를 설정할 수 있습니다.(기본값은 아래에서 설명하는 느린 4G)

설정은 다음 키값과 밸류값을 객체로 두어 사용 할 수 있습니다.

- `rttMs`:네트워크 왕복 시간을 밀리세컨드 단위로 제한합니다.(TCP계층에서 영향)(0은 제한 없음을 의미)
- `requestLatencyMs` :네트워크 요청 지연을 밀리세컨드 단위로 둡니다(HTTP 계층에서 영향)(0은 제한 없음을 의미)
- `throughputKbps`: 네트워크 요청시 초당 처리할 수 있는 키로바이트양 (0은 제한 없음을 의미)
- `downloadThroughputKbps`: 네트워크 요청시 초당 다운로드 받을 수 있는 키로바이트양 (0은 제한 없음을 의미)
- `uploadThroughputKbps`: 네트워크 요청시 초당 업로드 할 수 있는 키로바이트양 (0은 제한 없음을 의미)
- `cpuSlowdownMultiplier`: cpu를 n배 더 느리게 하는 정도(3을 입력하면 cpu는 1/3 정도로 느려짐)
- `simulate` 에서는 `rttMs` 와 `throughtputKbps`,   `devtools` 에서는 `requestLatencyMs` 와 `downloadThroughputKbps` 옵션을 사용합니다.
- 크롬 devtools의 네트워크 탭에서 네트워크 설정을 할 수 있습니다.
해당 네트워크 설정과 같은 설정을 하기 위해서는 다음 값을 `throttling` 에 넣으면 됩니다.

```jsx
// 빠른 4G
{
  rttMs: 40,
  throughputKbps: 10240,
  requestLatencyMs: 0,
  downloadThroughputKbps: 0,
  uploadThroughputKbps: 0,
}

// 느린 4G
{
  rttMs: 150,
  throughputKbps: 1638.4,
  requestLatencyMs: 562.5,
  downloadThroughputKbps: 1474.56,
  uploadThroughputKbps: 675,
}
// 3G
{
  rttMs: 300,
  throughputKbps: 700,
  requestLatencyMs: 1125,
  downloadThroughputKbps: 630,
  uploadThroughputKbps: 630,
}

```

> **Warning**

> 크롬의 네트워크 탭에서 다음과 같은 설정을 두고 크롬에서 lighthouse를 사용하는 것과는 다른 방식입니다. 크롬의 네트워크 탭에서 성능 제한을 두더라도 lighthouse 측정에 반영되지 않습니다.

> 이는 리포트 하단을 보면 알 수 있습니다. 아래의 두 스크린샷에서 확인할 수 있듯이, 네트워크 탭과 쓰로틀링은 관련이 없습니다.(네트워크 탭의 주의 문구를 보면 쓰로틀링여부를 확인할 수 있습니다)

![image.png](image%201.png)

![image.png](image%202.png)

> 만약 크롬 lighthouse와 동일한 성능으로 측정을 하고 싶으면 `throttling` 에 다음과 같은 값을 넣으십시오.

```jsx
// 모바일 (느린 4G + cpu 4배 감소)
{
  rttMs: 150,
  throughputKbps: 1638.4,
  requestLatencyMs: 562.5,
  downloadThroughputKbps: 1474.56,
  uploadThroughputKbps: 675,
  cpuSlowdownMultiplier: 4
 }
 
 // 데스크탑 (빠른 4G)
 {
  rttMs: 40,
  throughputKbps: 10240,
  requestLatencyMs: 0,
  downloadThroughputKbps: 0,
  uploadThroughputKbps: 0,
}
```

`onlyCategory`: 점검할 카테고리를 설정합니다. 빈 배열을 넣게 되면 아무런 테스트도 하지 않으며, 아무런 값을 넣지 않는다면 모든 카테고리에 대한 검사(`'accessibility', 'best-practices', 'performance', 'seo'` )를 진행합니다. 값은 문자열 배열입니다.

`skipAudits` : 검사하지 않을 audits를 설정합니다. 검사하는 audit의 리스트는 [default-config.js의 categories](https://github.com/GoogleChrome/lighthouse/blob/0c7c183ad25d41192aad23a37a37281d5aa364f4/core/config/default-config.js#L376) 배열 내의 id 값들 입니다.

`extraHeaders` : 네트워크 요청시에 특정 헤더를 추가해서 보냅니다. 이 설정을 사용하여 로그인 등 인증 상태를 구현할 수 있습니다. 값은 객체 형태로 설정합니다.

위 설정을 활용한 설정파일의 예시는 다음과 같습니다.

```jsx
export default {
  extends: 'lighthouse:default',
  settings: {
    locale: 'ko',
    onlyCategories: ['performance'],
    throttlingMethod: 'simulate',
    throttling: {
      // 네트워크 설정: 3G
      // throttlingMethod가 devtools로 변경되었을 때를 위해
      // requestLatencyMs와 downloadThroughputKbps 설정을 해주었습니다.
      rttMs: 300,
      throughputKbps: 700,
      requestLatencyMs: 1125,
      downloadThroughputKbps: 630,
      uploadThroughputKbps: 630,
      // 현재 cpu에서 1/2만큼 쓰로틀링
      cpuSlowdownMultiplier: 2,
    },
    extraHeaders: {
      token: 'mocked',
    },
  },
};

```

### 주요 플래그

CLI의 주요 플래그에 대해 알아보겠습니다. 이 글에서 다루지 않는 옵션들은 [lighthouse github readme](https://github.com/GoogleChrome/lighthouse?tab=readme-ov-file#using-the-node-cli)에서 확인하실 수 있습니다.

#### 로깅

`--quiet` 

기본적으로 lighthouse node CLI는 인터페이스 실행 시 터미널에 어떤 작업이 일어나고 있는지 알려줍니다. 

![image.png](image%203.png)

이러한 로깅을 끕니다.

#### output

`--output` 

결과값을 저장할 확장자입니다. 기본값은 html입니다

`json` 과 `csv` , `html` 이 가능하며, 복수의 값도 가능합니다. 구분은 `,` 로 합니다.

ex) `--output=html,json` 

`--output-path` 

결과값을 생성할 위치입니다. `--output` 의 값이 여러 개일 경우 선택한 생성 위치 뒤에 `.report.확장자` 가 붙어서 저장됩니다

`--view`

lighthouse 실행이 종료되고 생성된 파일들을 엽니다.

해당 플래그들 역시 파일로 만들어 관리할 수 있습니다.

```jsx
// lighthouse-flags.json
{
  "quiet": true,

  "output": "html",
  "outputPath": "./report.html",
  "view": true
}

```

위와 같이 flag 파일을 만든 후

```jsx
lighthouse https://www.google.com/ --config-path=./lighthouse-config.js --cli-flags-path=lighthouse-flags.json
```

와 같이 사용할 수 있습니다.

### Lighthouse Node module 사용

앞서 설명한 Lighthouse CLI를 활용하면 스크립트를 활용해 Lighthouse를 사용할 수 있습니다. 그런데 돌이켜보면 CLI만 사용했을 때에는 chrome Lighthouse와 차이가 크게 나지 않아보입니다(특히 자동화 관점에서).

Lighthouse를 더 프로그래머스럽게 사용하기 위해, Lighthouse Node module을 사용할 수 있습니다.

Lighthouse Node module을 사용하기 위해서는 별도의 브라우저 환경이 필요합니다. 테스트하려는 환경이 localstorage 설정이나 브라우저 권한 설정 등이 필요없다면 `chrome-launcher`를 통해 브라우저 환경을 만들어줄 수 있습니다.

아래 명령어는 개발 의존성을 가진 플래그(-D)를 사용합니다. 필요에 따라 다른 플래그를 사용하셔도 무방합니다.

```jsx
// npm
npm install -D chrome-launcher
// yarn
yarn add -D chrome-launcher
```

기존에 사용했던 flag와 config를 활용해서 lighthouse node module을 사용해보겠습니다.

우선 파일을 작성합니다.

```jsx
import * as chromeLauncher from 'chrome-launcher';

import configs from './lighthouse-config.js';
import flags from './lighthouse-flags.json' assert { type: 'json' };
import fs from 'fs';
import lighthouse from 'lighthouse';
const chrome = await chromeLauncher.launch({});

const runnerResult = await lighthouse(
  'https://www.google.com',
  { ...flags, port: chrome.port },
  configs,
);

const reportHtml = runnerResult.report;
fs.writeFileSync('lhreport.html', reportHtml);

chrome.kill();

```

그리고 다음과 같이 실행합니다.

```jsx
node ./lighthouse.js
```

>  **Note**

> node.js 환경에서는 기본적으로 cjs 모듈 시스템을 사용합니다. `import` 키워드를 사용하기 위해 package.json에 다음과 같은 설정을 추가해주셔야 합니다.

```jsx
...
 "type": "module"
}
```

> 아직까지 JSON파일을 모듈처럼 사용하는 것은 실험적인 기능입니다. 다음과 같은 경고 문구가 나올 수 있습니다.

![image.png](image%204.png)

> 이를 없애기 위해선 만들어둔 `lighthouse-flags.json` 파일을 js 형태로 바꾸어 `export default` 해주면 됩니다.

## puppeteer로 브라우저 기능 적극 활용하기

앞서 작성된 코드를 활용해 lighthouse를 사용할 수 있었습니다. 그런데 해당 코드에서는 chrome-launcher를 사용해 브라우저를 추가적으로 조작하는 것이 어려웠습니다.

chrome-launcher를 이용하면 브라우저의 권한이 필요한 페이지를 테스트하는 경우 등 일부 상황에서는 제한적인 테스트를 진행해야 합니다.

이런 지점을 해소하기 위해, puppeteer를 이용할 수 있습니다.

puppeteer는 node 환경에서 Chrome 또는 Firefox를 제어할 수 있는 높은 수준의 API를 제공합니다. puppeteer는 그 자체로 훌륭한 테스트 도구입니다. 이번 포스트에서는 lighthouse를 사용하기 위한 보조 도구로써 사용해보겠습니다.

우선 puppeteer를 설치해줍니다.

```jsx
// npm
npm install -D puppeteer
// yarn
yarn add -D puppeteer
```

이후 위에서 사용했던 코드를 수정해줍니다.

```jsx
import configs from './lighthouse-config.js';
import flags from './lighthouse-flags.json' assert { type: 'json' };
import fs from 'fs';
import lighthouse from 'lighthouse';
import puppeteer from 'puppeteer';

const browser = await puppeteer.launch({ headless: false });
const page = await browser.newPage();

const runnerResult = await lighthouse(
  'https://www.google.com',
  flags,
  configs,
  page,
);

const reportHtml = runnerResult.report;
fs.writeFileSync('lhreport.html', reportHtml);

await browser.close();

```

#### 로컬스토리지 설정

puppeteer의 `goto` 와 `evaluate`를 사용하면 로컬스토리지를 활용할 수 있습니다.

코드를 다음과 같이 변경하면 특정 사이트의 url 로컬스토리지를 활용할 수 있습니다.

```jsx
import configs from './lighthouse-config.js';
import flags from './lighthouse-flags.json' assert { type: 'json' };
import fs from 'fs';
import lighthouse from 'lighthouse';
import puppeteer from 'puppeteer';

const browser = await puppeteer.launch({ headless: false });
const page = await browser.newPage();

const url = 'https://www.google.com';

const localStorageKey = 'key';
const localStorageValue = 'value';

// 로컬스토리지 설정
await page.goto(url)
await page.evaluate(()=>{
	localStorage.setItem(localStorageKey, localStorageValue);
})

const runnerResult = await lighthouse(
  url,
  flags,
  configs,
  page,
);

const reportHtml = runnerResult.report;
fs.writeFileSync('lhreport.html', reportHtml);

await browser.close();
```

#### 브라우저 권한

puppeteer를 활용하면 브라우저의 특정 권한을 활용해줄 수 있습니다.

코드는 다음과 같습니다.

```jsx
import config from './lighthouse-config.js';
import flags from './lighthouse-flags.json' assert { type: 'json' };
import fs from 'fs';
import lighthouse from 'lighthouse';
import puppeteer from 'puppeteer';

const browser = await puppeteer.launch({ headless: false });
const context = browser.defaultBrowserContext();

const url = 'https://www.google.com';

const permissions = ["geolocation", "notifications"];
context.overridePermissions(url, permissions);

const page = await browser.newPage();

const runnerResult = await lighthouse(
  url,
  flags,
  configs,
  page,
);

const reportHtml = runnerResult.report;
fs.writeFileSync('lhreport.html', reportHtml);

await browser.close();
```

이로써 lighthouse를 조금 더 프로그래머스럽게 사용할 수 있게 되었습니다.

## 참고자료

[https://developer.chrome.com/docs/lighthouse](https://developer.chrome.com/docs/lighthouse)

[https://github.com/GoogleChrome/lighthouse](https://github.com/GoogleChrome/lighthouse)

[https://github.com/puppeteer/puppeteer](https://github.com/puppeteer/puppeteer)

[https://stackoverflow.com/questions/46643345/answer-to-chromes-notifications-using-puppeteer](https://stackoverflow.com/questions/46643345/answer-to-chromes-notifications-using-puppeteer)

[https://stackoverflow.com/questions/51789038/set-localstorage-items-before-page-loads-in-puppeteer](https://stackoverflow.com/questions/51789038/set-localstorage-items-before-page-loads-in-puppeteer)

