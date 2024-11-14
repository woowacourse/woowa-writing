# 크롬 익스텐션 구성 요소 완벽 이해해보기

> 우아한테크코스 6기 FE 마스터위

## 목차

- 서언
- 본론 1 : 익스텐션 구성요소 살펴보기 - popup, background
- 본론 2 : 익스텐션 구성요소 살펴보기 - manifest.json, content 스크립트
- 본론 3 : 복잡한 요구사항 구현해보기 - 데이터 저장과 context 간 데이터 전송
- 마치며

## 서언

혹시 웹 서비스를 이용할 때, 추가로 어떤 기능이 있으면 좋겠다고 생각해보신 적 있으신가요? 예를 들어, 여러 나라의 언어가 섞인 유튜브 댓글에서 한국어만 보고 싶다는 생각처럼 말입니다. 이렇게 기존의 웹 서비스에서 추가적인 기능을 더하는 것은 모두 익스텐션으로 가능합니다.

Chrome, Firefox, Safari 등 어떤 브라우저를 쓰시더라도 크롬 익스텐션(Chrome extension)은 한번 쯤 들어보셨을 거에요. 한국어로는 "확장 프로그램"이라고 하며, 한 두가지 익스텐션은 직접 설치해서 사용해보셨을 것이라고 생각합니다. 만약 아니시라면 다음 이미지와 같이 ‘Chrome Web Store’에서 마치 앱을 설치하는 것처럼 익스텐션을 설치하실 수 있습니다.

<img src='./images/1.png' width=500>

익스텐션은 브라우저의 기능을 확장할 수 있는 작은 웹 애플리케이션입니다. 다음 이미지처럼 해당 위치에서 찾을 수 있고 브라우저 툴 바에 자주 쓰는 익스텐션을 고정시켜놓고 사용할 수도 있습니다. 이제부터 크롬 익스텐션이 어떻게 구성되어있고, 브라우저와 함께 어떻게 동작하는지 알아보겠습니다.

<img src='./images/2.png' width=500>

## 들어가기에 앞서

사실 웹 익스텐션(Web Extension)이라고 하는 것이 "크롬" 익스텐션보다 더 포괄적이고 정확한 명칭일 것입니다. 크롬의 익스텐션에만 있는 기능이 아니기 때문입니다. 다만 이번 글에서는 크롬 팀이 주도한 Extension manifest v3에 기반하여 작성하기에, "크롬" 익스텐션 명칭을 하였습니다. [MDN의 참고자료](https://developer.mozilla.org/ko/docs/Mozilla/Add-ons/WebExtensions)

## 익스텐션 구성요소 살펴보기 - popup, background

그럼 익스텐션이 어떻게 구성되는지 살펴보겠습니다. 익스텐션은 다음 그림처럼 크게 세 가지 파트와 manifest라는 설명서로 이루어져 있습니다.

<img src='./images/3.png' width=500>

### Popup 페이지

그 중 먼저 Popup 페이지에 대해 알아보겠습니다. Popup 페이지는 HTML + CSS + JS로 구성되는 전형적인 웹 애플리케이션입니다. 브라우저 상단의 툴바에 있는 익스텐션 아이콘을 눌렀을 때, 팝업되어 나오는 화면입니다. 한 가지 유의하여 기억해둘 점은, 현재 띄워져 있는 브라우저의 context와 Popup 페이지는 별개의 context를 가진다는 점 입니다. 여기서 context란 Javascript 코드가 실행되는 하나의 실행 환경이라고 간단히 이해하고 넘어가겠습니다. 예를 들어, 서로 다른 context라면 한 곳에서 선언한 변수를 다른 한 곳에서는 참조하지 못하는 환경입니다.

다음 그림과 같이 UI를 제공해 익스텐션을 직접 제어하거나 사용자와 익스텐션 간 상호작용 할 수 있는 공간입니다. 앞서 Popup 페이지의 context에 대한 이야기를 하였습니다. Popup 페이지의 context는 유저의 클릭 등으로 인해 팝업되어 있는 동안에만 존재하며 페이지가 닫히면 해당 context도 사라진다는 것을 유의해야합니다.

<img src='./images/4.png' width=500>

### background 스크립트

다음으로 background 스크립트를 보겠습니다.
이는 익스텐션의 백그라운드 프로세스로, background 스크립트 역시 Popup 페이지와 마찬가지로 현재 띄워져 있는 브라우저와 독립적인 Javascript context를 갖습니다. background 스크립트에서 한 가지 유의할 점은, 하나의 서비스 워커로써 동작을 한다는 것입니다. 서비스 워커로 동작하기 때문에 브라우저나 익스텐션에서 발생하는 다양한 이벤트를 감지하고 처리합니다. 익스텐션의 background 스크립트는 이벤트 기반임을 꼭 기억하시길 바랍니다. (manifest v3 기준)

여기서 서비스 워커란, 브라우저의 웹 페이지와 별개로 작동하며 사용자와의 상호작용이 필요하지 않은 기능만을 제공합니다. 따라서 서비스 워커의 라이프사이클은 웹 페이지와 완전히 별개이며, 브라우저에서 접근할 수 있는 `document API`나 `window API`에 접근하지 못합니다. 또 비동기적 실행을 하는 특징을 가지고 있습니다.

### Popup 페이지 + background 스크립트

그럼 지금까지 알아본 popup 페이지와 background 스크립트를 가지고 하나의 익스텐션을 구성해보겠습니다.

Popup 페이지에서는 사용자가 한 서비스에 로그인을 한다고 가정하겠습니다. Popup 페이지에서는 로그인에 필요한 아이디와 비밀번호를 입력 받아야할 것입니다. 이렇게 사용자와의 상호작용에 필요한 UI를 Popup 페이지에 구현해두겠습니다. 이후 백엔드 API 서버와 통신 후 유저 정보를 저장하는 방식은 `<복잡한 요구사항 구현해보기>` 챕터에서 알아보겠습니다.

로그인을 한 후, 저희가 구현할 동작은 `contextMenu`를 추가할 것입니다. `contextMenu`란 브라우저를 사용하다보면 우클릭을 했을 때 나오는 메뉴입니다. 해당 메뉴에 저희는 이 익스텐션을 위한 옵션을 추가할 것입니다.

<img src='./images/5.png' width=300>

background 스크립트는 이벤트 기반으로 동작한다고 설명드렸는습니다. 아래 코드 예시를 보시면 먼저 `onInstalled` 되었을 때, 즉 익스텐션이 브라우저에 설치되는 이벤트가 발생했을 때 실행됩니다. background 스크립트에는 `onInstalled` 이벤트 리스너를 추가했기 때문에 이벤트를 감지할 수 있습니다. 이때 id는 `uploadToCodeZap`이고 title은 `코드잽에 소스코드 업로드하기` 메뉴 옵션이 추가됩니다.

이후 해당 메뉴 옵션이 `click`되었는지의 이벤트를 감지하는 리스너도 추가하겠습니다. 만약 이 옵션이 `click`된다면 background 스크립트는 이 이벤트를 감지해 background에서 데이터를 처리 후 백엔드 API 서버로 데이터를 업로드하는 동작을 처리하게 될 것입니다.

만약 이렇게 `click` 이벤트를 감지하는 로직이 background 스크립트가 아닌 Popup 페이지의 스크립트 context에 구현되어 있다면 어땠을까요? Popup 페이지가 닫히면 Popup 페이지의 context가 사라져 더이상 `click` 이벤트를 감지하지 못할 것입니다.

전역적으로 처리해야할 동작은 브라우저가 살아있는 한 항상 동작 중인 background 스크립트에서 일을 하는 것이 올바를 것입니다.

## 익스텐션 구성요소 살펴보기 - manifest.json, content 스크립트

이쯤에서 익스텐션의 구성요소 중 유일하게 필수 요소인 manifest에 대해 알아봅시다.
manifest.json은 익스텐션의 메타데이터를 표시하고 익스텐션이 동작을 하기 위해 필요한 권한 설정 및 파일 정의를 합니다.

<img src='./images/6.png' width=300>

- `manifest_version` : 익스텐션 manifest의 버전입니다. v2에서 v3로 바뀌며 보안 및 성능이 개선되었습니다.

- `permissions` : 익스텐션에서 필요한 특정 Chrome API 권한을 명시합니다. 주의할 점은, 익스텐션에서 사용하는 권한만 정확히 명시해야합니다. 만약 쓰지 않는 권한을 이곳에 명시한다면 Chrome Web Store에 배포할 때, 심사에서 반려당할 것입니다.

- `host_permissions` : 특정 웹사이트에 대해 익스텐션이 동작할 권한을 요청합니다. URL 형식으로 웹사이트에 대한 권한을 요청하며 v3에서 추가된 항목입니다.

- `background` : background 스크립트로 동작할 파일을 정의하고, 서비스 워커로써 동작하는 것은 v3에서 성능상 이점을 위해 추가되었습니다. 이는 v2까진 지속적으로 브라우저의 백그라운드에서 리소스를 차지했다면, 서비스워커로 바뀌며 이벤트 기반, 비동기적 실행을 하게 되었습니다.

- `content_scripts` : 웹페이지의 context에서 실행될 context 스크립트를 정의합니다.

  - `matches` : 해당 content 스크립트가 실행될 URL 패턴을 정의합니다. `host_permissions`의 URL과 다른 점으로 content 스크립트가 동작할 브라우저의 웹 페이지로 직접 DOM을 수정하거나 접근할 권한을 부여할 URL입니다. `host_permissions`의 URL에는 전체적인 익스텐션이 동작할 URL 입니다. 예를 들어 네트워크 요청을 가로채거나 Chrome API를 사용할 수 있습니다.

- `actions` : 익스텐션 아이콘을 클릭했을 때 나타날 Popup 페이지와 아이콘을 정의합니다.

- `icons` : 익스텐션에서 사용하는 아이콘의 모음입니다.

  - `16x16` : 브라우저의 상단 툴 바에서 사용되는 파일입니다.
  - `48x48` : 익스텐션 관리 페이지에서 사용되는 파일입니다.
  - `128x128` : 웹스토어 또는 기타 고해상도에서 사용되는 파일입니다.

이제 content 스크립트에 대해 알아보겠습니다. content 스크립트는 웹 페이지의 context에서 실행되는 파일입니다. 따라서 웹 페이지 DOM에 직접 접근해서 내용을 변경하거나 페이지와 상호작용할 수 있습니다.

이전에 알아본 manifest에서 허용한 URL에만 content 스크립트가 접근할 수 있음을 꼭 기억하시길 바랍니다. 다음 그림처럼, 만약 content 스크립트의 `matches`에 URL을 등록한다면, 해당 URL에 대한 데이터를 읽고 변경해도 괜찮은지에 대한 권한을 사용자에게 요청합니다.

예를 들어, 변경하고자 하는 웹 페이지 서비스의 DOM 요소를 확인합니다. DOM요소의 class가 `.nav_bar` 라면, 해당 DOM 요소를 `querySelector`로 가져온 후 원하는 element를 `appendChild`하는 동작을 수행할 수 있을 것입니다.

## 복잡한 요구사항 구현해보기 - 데이터 저장과 context 간 데이터 전송

자 그럼 앞서 살펴본 Popup, background, content를 조금 더 이해해보기 위해, 더 복잡한 요구사항을 가진 예시를 보겠습니다.

구현할 요구사항은 다음과 같습니다.

1. 사용자가 Popup 페이지에서 로그인을 한 후 유저 정보를 저장한다.
2. 사용자가 웹 브라우저에서 코드(텍스트 데이터)를 선택해 익스텐션에서 백엔드 API로 업로드한다.

### 데이터 저장하기 - Web Storage와 Storage API

먼저 Popup에서 로그인을 하기 위해 백엔드 API 서버와 통신을 하고 유저 정보를 `response`로 받아올 것 입니다. 이 정보를 익스텐션에 저장해두어야 할텐데, 어디에 어떻게 저장해야할까요?

크게 두 가지 선택지가 있을 것 같습니다. 첫번째는 보통의 웹 브라우저처럼 `localStorage`와 `sessionStorage` 같은 web storage를 활용하는 방법입니다. 두번째는 `indexedDB`를 사용하는 방법입니다.

여기서 주의할 점은 Content 스크립트와 Popup 페이지, background는 각각 독립된 context를 갖는다는 것입니다. 또 Content 각각의 `Origin`이 다릅니다. 아래 그림에서 보시면 Content 스크립트가 동작하는 context인 브라우저의 웹 페이지 개발자도구를 통해 web storage를 보시면 `Origin`이 해당 웹 페이지의 URL로 되어있는 것을 확인할 수 있습니다.

하지만 Popup 페이지의 개발자도구를 통해 web storage를 보시면 통상 봐왔던 URL 형식의 `Origin`이 아닐 것입니다. 크롬 익스텐션에서는 특이한 `Origin` 주소를 갖습니다. `chrome-extension://{extension-key}` 형식의 주소입니다. 이는 곧, 두가지 `Origin`, Content 스크립트와 Popup 페이지에서 접근하는 web storage가 다르다는 것을 의미합니다.

<img src='./images/8.png' width=300>

<img src='./images/9.png' width=300>

또 background 스크립트는 서비스 워커로써 동작하기 때문에 web storage에 접근할 수조차 없습니다.

그렇다면 익스텐션의 모든 context에서 접근 가능한 저장소는 없을까요? Chrome API 에서는 이를 위해 `chrome.storage API`를 제공하고 있습니다. 관련 두가지 API를 살펴보겠습니다.

먼저 `chrome.storage.sync API` 입니다. 이는 web storage와 유사하게 `get`, `set` 메서드가 존재합니다. 특징으로는 저장하는 공간이 클라이언트의 리소스가 아닌 chrome 브라우저에 로그인되어있는 유저의 클라우드에 저장을 합니다. Google chrome에서 제공하는 클라우드에 저장하는 만큼 한정적인 용량을 제공합니다.

`chrome.storage.sync` 대신 web storage와 비슷하게 클라이언트 단의 리소스를 사용하는 API로 `chrome.storage.local API`가 있습니다. 이 또한 `get`, `set` 메서드를 제공합니다. `sync`와 `local` API를 통해 익스텐션의 각 context간 동일한 저장소를 접근해 사용할 수 있습니다.

### 데이터 전달하기 - Message Passing

다음으로 `사용자가 웹 브라우저에서 코드(텍스트 데이터)를 선택해 extension에서 백엔드 API로 업로드한다.` 요구사항을 만족시켜 보겠습니다.

브라우저에서 선택한 텍스트 데이터는 content 스크립트의 context에 존재할 것이고 익스텐션의 background 스크립트의 context와 해당 데이터를 공유하여 background 스크립트에서 백엔드 API 서버로 업로드 요청을 보낼 것입니다. 하지만 다른 context로 인해 해당 데이터를 변수 등으로 공유할 수 없을 것입니다.

이때 `Message Passing` 기법을 통해 다른 context간 데이터를 공유할 수 있습니다.

다음 코드를 보시면 먼저 background 스크립트에서 `onMessage` 이벤트 리스너를 등록해, 만약 `message.action`이 `sendSourceCode`일 때만 메시지에서 데이터를 꺼내어 쓸 것입니다. content 스크립트에서는 공유하고자 하는 데이터를 action과 함께 익스텐션의 각 context에 메시지를 보냅니다.

<img src='./images/7.png' width=500>

이렇게 메시지를 전달하는 방식으로 익스텐션 내의 각 context끼리 데이터를 공유할 수 있습니다.

## 마치며

지금까지 익스텐션의 주요 구성요소에 대해 살펴 보았습니다. 이에 더불어 익스텐션을 처음 접할 때, 헷갈릴 수 있는 context와 Origin에 대해 정확히 알고 간다면 앞으로 익스텐션을 개발할 때 각각의 역할을 충실히 수행할 수 있을 것입니다.

예시로 쓰인 `contextsMenus API`와 `storage API`를 제외하고 수많은 `Chrome API`가 존재합니다.
`chrome.webRequest`로 네트워크 요청을 가로채거나 `chrome.bookmarks`로 북마크에 접근하고 `devtools`에 접근해 커스텀할 수 있는 기능도 제공됩니다. `i18n`을 통해 다국어 지원, `commands`를 통해 단축키를 커스텀하고 `tts`도 지원 가능합니다.

이번 글에서의 예시와 공식문서에는 `Vanilla Javascript`로 작성되어있지만 `Webpack/Vite + TypeScript + React`로 웹 프론트엔드에서 익숙한 기술 스택으로 빠르게 개발이 가능합니다.

`chrome web store`에 개발자 아이디를 등록하는 비용은 일회성으로 $5 지불로 저렴한 편에 속하며, 소스코드만 올린다면 심사 후 구글에서 배포 관리를 해주니 이 또한 편리할 것입니다.

이 글을 읽으신 모든 분들도 자신만의 아이디어를 크롬 익스텐션을 통해 가볍게 구현해보면 어떨까요?
