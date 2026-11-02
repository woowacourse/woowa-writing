---
title: "인스타그램 릴스를 공유하면 앱을 열지 않고 저장된다"
author: 엠버
---

# 인스타그램 릴스를 공유하면 앱을 열지 않고 저장된다

### React Native 앱에 iOS Share Extension을 붙이고, 저장 요청까지 연결한 과정

> **이 글에서 다루는 것**: Share Extension Target 추가 · 공유된 URL 꺼내기 · Extension 안에서 인증과 저장 요청 처리 · 앱과 로그인 정보 나누기 · 공유 시트 위 화면 줄이기
> **읽기 전에**: React Native 앱을 빌드·실행해 본 경험이면 충분합니다. Swift와 Xcode Target은 필요한 만큼 설명합니다.
> 코드는 여기담(Yeogidam) 앱에 실제로 들어간 코드를 지면에 맞게 줄인 것입니다.

---

## 0. 결론부터

여기담은 인스타그램 릴스에 나온 장소를 모아 두는 앱입니다. 지금은 릴스에서 공유 버튼을 누르고 **여기담**을 고르면, 앱을 열지 않아도 링크가 서버에 접수되고 장소 추출이 시작됩니다. 만들면서 얻은 결론은 다섯 가지입니다.

1. Share Extension은 앱 안의 화면이 아니라 **앱과 별개의 프로세스**다. 우리는 RN 없이 **순수 Swift(UIKit)** 로 만들었다.
2. URL은 `extensionContext` → `NSItemProvider`로 꺼낸다. 공유하는 앱마다 담는 형태가 달라서 **여러 경로를 순서대로 시도**해야 한다.
3. Extension은 오래 살아 있지 못한다. API 호출은 직접 기다리지 않고 **백그라운드 `URLSession`에 맡기고 먼저 닫는다.**
4. 앱과 Extension은 메모리를 나누지 않는다. 로그인 정보는 **공유 Keychain**, 처리 결과는 **App Group `UserDefaults`** 로 주고받는다.
5. 기본 화면은 **없앨 수 없다.** 다만 시스템 배경을 투명하게 만들고 하단 카드 하나만 보이게 줄일 수 있다.

---

## 1. 문제 — 공유했는데 왜 앱을 열어야 하지?

처음 공유 흐름은 이랬습니다.

```text
인스타그램 → 공유 → (앱 열림) → URL 전달 → 앱에서 저장 요청 → 처리 상태 확인
```

사용자는 릴스를 넘겨 보다가 마음에 드는 장소를 발견한 **그 순간**에 공유를 누릅니다. 그때마다 앱으로 끌려가면 보던 흐름이 끊깁니다. 장소를 세 곳 저장하려면 인스타그램과 앱을 세 번 오가야 하니, 공유 기능을 쓰는 의미가 줄어듭니다. 우리가 원한 모습은 다음과 같습니다.

```text
인스타그램 → 공유 → 공유 시트에서 '여기담' 선택 → 카드 안내 → 인스타그램으로 복귀
```

구현 전에 막막했던 질문은 여섯 가지였습니다. ① Share Extension Target은 어떻게 추가하나 ② 공유된 URL은 어떻게 꺼내나 ③ 앱과 Extension의 실행 환경은 어떻게 다른가 ④ 꺼낸 URL을 어떻게 저장 요청까지 잇나 ⑤ 끝난 뒤 어떻게 닫나 ⑥ 뜨는 화면을 줄일 수 있나. 아래에서 순서대로 답합니다.


---

## 2. 비교한 방법 — URL만 받아 둘까, 접수까지 할까

| 기준 | A. 앱을 열어 처리 (Extension은 전달만) | B. Extension에서 접수까지 |
|---|---|---|
| 공유 후 앱으로 이동 | 필요 | **불필요** |
| 사용 흐름 방해 | 큼 | **거의 없음** |
| Extension에 필요한 데이터 | URL 정도 | URL + 로그인 세션 + API 주소 |
| 앱·Extension이 나눠야 할 정보 | 적음 | URL, 세션, 처리 결과 |
| 구현 난이도 | 낮음 | **높음** |

구현은 어려워도 사용자가 얻는 것이 분명한 **B**를 골랐습니다. 다만 B를 고르려면 먼저 알아야 할 사실이 있습니다. **Share Extension은 앱에 화면을 하나 추가하는 것이 아니라, 별도의 번들·프로세스·메모리를 가진 작은 앱을 하나 더 만드는 일**입니다.

| | 본 앱 (React Native) | Share Extension |
|---|---|---|
| Xcode Target | `Frontend` (application) | `FrontendShareExtension` (app-extension) |
| 프로세스 | 앱 프로세스 | 공유 시트가 띄우는 **별도 프로세스** |
| JS 런타임 / 앱 상태 | 있음 | **없음** (접근 불가) |
| 수명 | 사용자가 닫을 때까지 | 시트가 닫히면 곧 종료, 메모리 제한이 엄격 |
| 데이터 공유 | — | App Group · 공유 Keychain 으로만 |

앱이 가진 로그인 상태·API 클라이언트·토큰 갱신 로직은 Extension에서 쓸 수 없습니다. 그래서 필요한 것을 **Extension 쪽에 네이티브로 다시 만들거나, 둘이 나눠 쓸 저장소에 내려놓아야** 합니다. 우리는 Extension에 React Native를 올리지 않고 Swift만 썼고, 공통 코드는 `ios/Shared/`에 두어 두 Target에 함께 등록했습니다.

```text
ios/
├── Frontend/          # RN 앱 (AppDelegate, ShareIntentModule 브리지)
├── ShareExtension/    # Share Extension  (ShareViewController 한 줄)
├── ActionExtension/   # Action Extension (같은 화면을 재사용)
└── Shared/            # ShareCardViewController(화면) · ShareIntake(접수) · ShareIntentStorage(저장소·인증·전송)
```


---

## 3. 해 본 일 — 만든 순서대로

```text
인스타그램 → 공유 시트 → '여기담' 선택 → ShareViewController 생성 (Extension 프로세스 시작)
 → NSExtensionItem / NSItemProvider 에서 URL 추출 · Instagram 검증
 → 공유 Keychain 의 로그인 세션 확인 (필요하면 토큰 갱신)
     ├─ 로그인 필요   → "로그인 해주세요" 카드 + 버튼
     ├─ 네트워크 없음 → 기기에 보관, 앱 진입 때 전송
     └─ 정상         → 백그라운드 URLSession 에 업로드 등록
 → "공유 완료!" 카드 → 4초 뒤 completeRequest 로 종료 (업로드는 OS 가 이어서 수행)
```

### 3-1. Target 만들기

**File → New → Target → Share Extension**으로 추가합니다. 우리는 공유 시트의 앱 아이콘 줄(Share Extension)과 아래 동작 목록(Action Extension) 두 곳에 모두 나타나도록 Target을 둘 만들고, 화면은 같은 클래스를 상속해 재사용합니다.

```swift
final class ShareViewController: ShareCardViewController {}   // ios/ShareExtension
final class ActionViewController: ShareCardViewController {}  // ios/ActionExtension
```

처음 추가할 때 확인할 것은 세 가지입니다. **(1)** Bundle ID가 앱의 하위 ID인가 (`com.yeogidamm.app` → `.Share`). **(2)** App Groups capability를 앱과 Extension **양쪽에 같은 ID**로 켰는가. **(3)** 공통 소스(`Shared/*.swift`)를 **각 Target의 Compile Sources에 모두** 등록했는가.


### 3-2. 받을 데이터 선언 — Info.plist

```xml
<key>NSExtensionActivationRule</key>
<dict>
  <key>NSExtensionActivationSupportsText</key><true/>
  <key>NSExtensionActivationSupportsWebURLWithMaxCount</key><integer>1</integer>
</dict>
<key>NSExtensionPointIdentifier</key><string>com.apple.share-services</string>
<key>NSExtensionPrincipalClass</key><string>$(PRODUCT_MODULE_NAME).ShareViewController</string>
```

URL 한 개 또는 텍스트를 받겠다는 선언입니다. 앱에 따라 URL이 `public.url`이 아니라 텍스트로 오기도 해서 넓게 열었습니다. 너무 엄격하면 인스타그램에서 아예 안 나타나고, 너무 느슨하면 아무 공유에나 나타납니다. 그래서 **진짜 Instagram 링크인지는 코드에서 판별**합니다. `NSExtensionPrincipalClass`는 모듈명이 붙은 이름이어야 하며, 오타가 나면 시트에 나타나지 않거나 곧바로 종료됩니다.

### 3-3. 공유된 URL 꺼내기

Extension이 시작되면 `extensionContext.inputItems`에 `NSExtensionItem`이 들어 있고, 각 아이템의 `attachments`(`NSItemProvider`)에서 타입별로 값을 비동기로 꺼냅니다. `public.url`만 읽으면 될 줄 알았지만, URL이 URL 타입·텍스트·본문 문구 중 어디에 담겨 올지 공유하는 쪽이 정합니다. 그래서 **세 단계로 시도하고 처음 통과한 것을 씁니다.**

```swift
// ios/Shared/ShareIntake.swift (발췌)
let item = extensionContext?.inputItems.compactMap { $0 as? NSExtensionItem }.last

// 1단계: URL 타입(public.url) 첨부
for provider in item.attachments ?? [] {
  if let s = try await loadURLString(from: provider), ShareIntentStorage.isInstagramContentURL(s) { return makePayload(s) }
}
// 2단계: 텍스트 타입(public.plain-text / public.text) 첨부
for provider in item.attachments ?? [] {
  if let t = try await loadText(from: provider), ShareIntentStorage.isInstagramContentURL(t) { return makePayload(t) }
}
// 3단계: 첨부가 아니라 아이템의 본문·제목 텍스트에 섞여 온 경우
for t in [item.attributedContentText?.string, item.attributedTitle?.string].compactMap({ $0 }) { ... }

throw ShareIntentStorageError.unsupportedContent   // 못 찾으면 "지원하지 않는 공유"
```

`loadItem(forTypeIdentifier:)`는 콜백 기반이라 `withCheckedThrowingContinuation`으로 감싸야 `async/await`로 쓸 수 있고, 돌려주는 값도 `URL`·`NSURL`·`String`·`Data`로 제각각이라 각각 풀어야 합니다. Instagram 여부는 **호스트와 경로**로 엄격히 판별합니다.

```swift
guard host == "instagram.com" || host == "www.instagram.com" else { return false }   // contains 가 아니라 일치
// 경로에 reel 또는 p 가 있고 바로 뒤에 shortcode 가 있어야 통과: /reel/{id}/, /p/{id}/
```

통과한 URL은 `https://www.instagram.com/{reel|p}/{shortcode}/`로 정규화해 추적 파라미터를 떼어 보냅니다. Instagram이 아닌 링크는 **저장도 API 호출도 하지 않고** "릴스·게시물만 공유할 수 있어요" 카드를 보여준 뒤 닫습니다.

### 3-4. 앱과 Extension이 데이터를 나누는 방법

서버는 로그인한 사용자만 받는데, 로그인 상태는 RN 앱이 가지고 있습니다. 앱이 **Extension도 읽을 수 있는 곳에 세션을 내려놓아야** 합니다. 저장소는 역할별로 나눴습니다.

| 저장소 | 담는 것 | 이유 |
|---|---|---|
| **공유 Keychain** (App Group ID를 access group으로) | 액세스·리프레시 토큰, 만료 시각, userId | 민감한 값을 `UserDefaults`에 두지 않기 위해 |
| **App Group `UserDefaults`** | 요청별 처리 결과, API 주소, 분석 이벤트 대기열 | 민감하지 않고 앱이 읽어 간다 |
| **App Group 컨테이너 파일** | 업로드 본문 JSON (전송 중에만) | 백그라운드 업로드가 파일을 요구한다 |

```swift
[kSecClass: kSecClassGenericPassword, kSecAttrService: "share_auth_session_v1",
 kSecAttrAccessGroup: "group.com.yeogidamm.app.shared"]   // 앱·Extension 이 같은 항목을 읽는다
// 저장 시 kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly: 첫 잠금 해제 후엔 백그라운드에서도 읽히고, 백업·기기 이전은 안 된다
```

RN 쪽에서는 로그인·토큰 갱신·로그아웃 때마다 네이티브 모듈로 세션을 내려놓습니다. **로그아웃 때는 `null`을 넘겨 Keychain 항목을 지워야** 합니다. 로그아웃했는데 공유 세션이 남아 추출 API가 계속 호출되던 문제가 실제로 있었습니다(#387).

```ts
// src/lib/share-intent/index.ts
export async function syncShareSession(session: NativeShareSession | null) {
  await nativeShareIntentModule?.setApiConfiguration?.(getApiBaseUrl());  // Extension 이 호출할 서버 주소
  await nativeShareIntentModule?.setShareSession?.(session);               // → 공유 Keychain
}
```

### 3-5. 접수 전에 인증 확인하기

저장 요청 전에 "지금 요청을 보낼 수 있는 상태인가"를 네 가지 결과로 가릅니다: `ready(토큰)` / `loginRequired` / `waitingForNetwork` / `waitingForAuth`. 세션이 없으면 `loginRequired`, 액세스 토큰이 60초 넘게 남았으면 `ready`, 아니면 리프레시 토큰으로 `POST /api/v1/auth/token-refreshes`(타임아웃 12초)를 호출합니다.

가장 까다로운 부분은 **앱과 Extension이 같은 리프레시 토큰을 동시에 쓰는 경우**입니다. 토큰이 매번 회전하는 서버에서는 한쪽이 먼저 갱신하는 순간 다른 쪽 토큰이 무효가 되고, 그대로 두면 멀쩡한 사용자에게 "로그인 해주세요"가 뜹니다.

```swift
// 갱신이 400/401 로 거절됐을 때
if let latest = try? loadSession(), latest.refreshToken != session.refreshToken {
  return .ready(latest.accessToken)          // 다른 프로세스(앱)가 먼저 회전했다 → 새 토큰을 쓴다
}
if json["errorCode"] as? String == "AUTH401_005" { return .waitingForAuth }  // 직전 토큰 재사용 → 기다린다
try? saveSession(nil); return .loginRequired("refresh_token_invalid")        // 그 외에만 로그인 요구
```

즉 **"리프레시가 거절됐다 = 로그아웃"이 아닙니다.** 저장소를 다시 읽어 다른 프로세스가 새 토큰을 넣어 두었는지 먼저 확인합니다.

### 3-6. 저장 요청은 Extension이 기다리지 않는다

`URLSession.shared`로 요청하고 응답을 기다린 뒤 카드를 띄우는 방식은 위험합니다. 추출 요청은 오래 걸릴 수 있는데, 사용자는 카드가 뜨자마자 인스타그램으로 돌아가고 시트가 닫히면 **Extension 프로세스가 곧 종료**될 수 있기 때문입니다. 요청이 반쯤 나간 채 죽으면 서버에 닿았는지도 알 수 없습니다. 그래서 요청의 실행 주체를 **OS의 네트워킹 데몬(백그라운드 `URLSession`)** 으로 넘겼습니다.

```swift
let configuration = URLSessionConfiguration.background(withIdentifier: "com.yeogidamm.app.instagram-share-upload")
configuration.sharedContainerIdentifier = appGroupIdentifier   // 앱·Extension 이 같은 컨테이너를 쓴다
configuration.sessionSendsLaunchEvents = true                  // 끝나면 앱을 깨워 결과를 전달

// 백그라운드 업로드는 메모리 Data 가 아니라 '파일'만 올릴 수 있다 → 본문을 파일로 쓴다
try JSONSerialization.data(withJSONObject: ["instagramUrl": url]).write(to: bodyFile, options: .atomic)
let task = session.uploadTask(with: request, fromFile: bodyFile)   // POST /api/v1/shares, Bearer 토큰
task.taskDescription = requestId                                    // 응답이 돌아왔을 때 어느 건인지 식별
task.resume()
```

기억할 점은 네 가지입니다. 업로드는 **`uploadTask(with:fromFile:)`만 허용**되고, 이 파일에는 URL이 들어 있으니 **전송이 끝나면 지워야** 합니다. `sharedContainerIdentifier`가 있어야 Extension이 만든 업로드를 앱이 이어받습니다. 앱이 종료된 상태에서 끝나면 OS가 `AppDelegate.application(_:handleEventsForBackgroundURLSession:completionHandler:)`를 불러 주므로 같은 식별자의 세션을 다시 만들어 결과를 받아야 합니다. 식별자는 앱 전체에서 유일해야 합니다.

응답은 Extension이 아니라 **델리게이트가 받아 결과로 기록**하고, 앱이 그 결과를 읽습니다.

| 상황 | 기록되는 상태 | 비고 |
|---|---|---|
| 2xx + `sharedMediaId` | `API_SUCCEEDED` | 추출 완료가 아니라 **접수 성공**(202) |
| 2xx인데 ID 없음 | `API_FAILED` | Sentry 보고 |
| 4xx / 5xx | `API_FAILED` (서버 `errorCode`) | 401은 `AUTH401_001`로 통일 |
| 요청이 **나가기 전** 네트워크 오류 | `WAITING_FOR_NETWORK` | 앱 진입 때 다시 시도 |
| 요청이 **나간 뒤** 네트워크 오류 | `API_FAILED` | **자동 재시도 안 함** |

마지막 두 줄이 설계의 핵심 정책입니다. **"요청을 보내기 시작한 뒤에는 같은 건을 자동으로 다시 보내지 않는다."** 서버에 멱등키가 없어서, 응답이 유실된 요청을 다시 보내면 같은 링크의 공유가 하나 더 생깁니다. 그래서 "서버에 닿았는지 알 수 없는 요청"은 재시도하지 않고, "한 바이트도 나가지 않은 요청"만 안전하게 다시 시도합니다. 이 경계를 구현한 곳이 `didSendBodyData`입니다.

```swift
func urlSession(_ s: URLSession, task: URLSessionTask, didSendBodyData bytesSent: Int64, ...) {
  guard bytesSent > 0, var result = loadResult(task.taskDescription), result.requestSentAt == nil else { return }
  result.requestSentAt = now          // 본문 첫 바이트가 나간 시점부터 재호출 금지
  result.transferStatus = "REQUESTING"
  result.url = ""                     // 이 시점부터 기기에 URL 을 남기지 않는다
  result.rawSharedText = nil
  try? saveResult(result)
}
```

URL을 들고 있는 시간을 "전송에 필요한 동안"으로 최소화하고, 이후 상태 확인은 서버가 돌려준 ID(`sharedMediaId`)만으로 합니다.

### 3-7. 닫기, 로그인, 앱 열기

접수가 끝나면 카드를 보여주고 4초 뒤 `completeRequest`로 닫습니다. 카드의 빈 곳을 누르면 바로 닫힙니다. 모든 종료 경로(성공·실패·로그인·탭)가 `finish()` 하나를 지나가게 해 두었습니다.

```swift
private func finish() {
  autoDismissTask?.cancel()
  extensionContext?.completeRequest(returningItems: [], completionHandler: nil)
}
```

카드의 **"공유 완료!"는 "저장이 끝났다"가 아니라 "링크가 접수되어 전송 작업이 등록됐다"** 는 뜻입니다. 장소 추출은 서버에서 비동기로 진행되고, 결과는 앱 보관함에서 확인합니다. Extension이 그것을 기다릴 수도, 기다려서도 안 됩니다.

**로그인이 필요하면** 카드에 "로그인 해주세요"와 버튼이 뜹니다. 버튼을 누르지 않고 닫으면 **URL을 저장하지 않습니다**(의사를 밝히지 않은 링크를 기기에 남기지 않기 위해). 버튼을 누르면 URL을 `LOGIN_HANDOFF`로 임시 저장하고 앱 열기를 시도하며, 이 상태는 **30분 뒤 만료**됩니다. 로그인에 성공하면 앱이 이어서 전송합니다.

**앱을 여는 것은 보장되지 않습니다.** iOS는 Share Extension이 호스트 앱을 여는 것을 공식 지원하지 않으며, `extensionContext.open(_:)`은 공유·작업 Extension에서 대개 `false`를 돌려줍니다. 세 단계로 대응했습니다.

```swift
extensionContext.open(url) { opened in                          // ① 공식 경로
  if opened { finish(); return }
  openViaResponderChain(url) { opened in                        // ② responder chain 에서 UIApplication 을 찾아 openURL (비공식)
    if opened { finish() } else { showCard("앱을 직접 열어 주세요") }   // ③ 실패 시 안내 카드
  }
}
```

②는 Apple이 지원하지 않는 방법이라 OS 버전에 따라 동작이 달라질 수 있습니다. 그래서 ③을 반드시 두었고, 앱이 열리지 않아도 링크는 기기에 남아 있어 앱을 직접 열면 이어서 처리됩니다.


### 3-8. 공유 시트 위의 화면을 얼마나 줄일 수 있나

가장 확인하고 싶었던 질문입니다. 결론은 **"화면 자체는 없앨 수 없다. 대신 보이는 면적을 카드 하나로 줄일 수는 있다."** 입니다. Extension의 뷰는 시스템이 만든 컨테이너 안에 올라가고, 그 컨테이너의 배경은 시스템이 깝니다. 우리 앱의 초기 화면은 그 위에 가운데 회색 박스를 띄운 형태였고, 이를 다음 세 가지로 바꿨습니다(#424).

**① 시스템 배경을 투명하게 만든다.** `view.superview`를 끝까지 타고 올라가며 배경색을 지웁니다. 시스템이 레이아웃 중에 배경을 다시 칠하는 경우가 있어 `viewWillAppear` · `viewDidAppear` · `viewDidLayoutSubviews`에서 **반복 호출**합니다.

```swift
private func makeHostContainerTransparent() {
  var current = view.superview
  while let container = current { container.backgroundColor = .clear; current = container.superview }
  view.window?.backgroundColor = .clear
}
```

**② 하단 카드 하나만 그린다.** 높이 72pt, 좌우 16pt 여백에 마스코트·한 줄 문구·버튼 하나. 문구는 `adjustsFontSizeToFitWidth`로 한 줄에 맞춰 카드가 커지지 않게 했습니다. **③ 카드 뒤는 25% 검정 딤**으로만 덮고 카드와 함께 페이드인합니다. Action Extension은 Info.plist에 `NSExtensionActionWantsFullScreenPresentation`을 주어 전체 화면 오버레이로 쓰고 그 위에 같은 카드를 얹었습니다.

**그래도 못 없애는 것**이 있습니다. 공유 시트 자체, 여기담을 탭한 순간의 시스템 전환 애니메이션, 그리고 iOS 버전마다 다를 수 있는 컨테이너 구조입니다. 컨테이너 계층을 돌며 색을 지우는 방식이라 구조가 바뀌어도 비교적 견디지만, 새 OS에서는 실기기로 다시 확인해야 합니다.


---

## 4. 달라진 점과 근거

**흐름이 바뀌었다.** 처리의 시작점이 React Native 앱에서 iOS Share Extension으로 옮겨 갔습니다.

```text
[이전]  인스타그램 → URL 공유 → 앱 실행 → URL 전달 → 저장 요청
[이후]  인스타그램 → URL 공유 → Share Extension → URL 추출 → 접수·전송 등록 → Extension 종료
        (앱은 나중에 열어도 된다. 열리면 결과를 읽어 처리 상태를 조회한다)
```

사용자에게는 인스타그램을 떠나지 않는다는 점이, 개발자에게는 "화면 중심 코드"가 **"저장소·상태 중심 코드"** 로 바뀐 점이 변화입니다. Extension이 끝난 뒤의 일은 앱이 이어받습니다. 앱이 켜지거나 다시 활성화되면 `resumeWaitingShares()`가 `LOGIN_HANDOFF` · `WAITING_FOR_NETWORK` · `WAITING_FOR_AUTH` 건을 찾아 토큰을 다시 확인하고 전송을 등록하며, 접수된 건은 `sharedMediaId`로 서버의 추출 상태를 3초 간격으로 조회해 완료되면 보관함을 새로 불러옵니다.

**구현하며 확인한 사실**과 그 근거는 다음과 같습니다.

| 확인한 사실 | 근거 |
|---|---|
| Extension은 별도 Target·별도 프로세스라 앱의 상태·메모리를 쓸 수 없다 | `FrontendShareExtension` Target, Swift만 사용 |
| URL은 `extensionContext` → `NSItemProvider`로 꺼내며 담기는 형태가 다양하다 | `ShareIntake.extractPayload`의 3단계 시도 |
| Extension 안에서도 네트워크 요청은 되지만 오래 기다리면 안 된다 | 토큰 갱신 12초 제한, 업로드는 백그라운드 세션 |
| 인증 정보는 별도 공유 장치가 필요하고, 두 프로세스가 같은 리프레시 토큰을 쓰면 경합한다 | 공유 Keychain, 갱신 거절 시 저장소 재확인 |
| 공유 Extension에서 앱을 여는 것은 공식 경로로는 대개 실패한다 | `open` → responder chain → 안내 카드 |
| 시스템 배경은 없앨 수 없지만 투명하게 만들 수 있다 | `makeHostContainerTransparent` |

코드 구조도 정리됐습니다. 화면과 접수 로직이 `ShareViewController` 한 파일에 섞여 있다가, Action Extension을 추가하면서 **화면(`ShareCardViewController`) · 접수(`ShareIntake`) · 저장소(`ShareIntentStorage`)** 로 나눴습니다(#424). 두 Extension은 각각 한 줄짜리 서브클래스가 되었고, 문구나 카드 디자인은 한 곳만 고치면 됩니다. 한편 한때 Extension 안에서 저장할 폴더를 고르는 기능도 넣었다가(`6bad4af`) 이후 제거했습니다(#481). 폴더 정리는 앱에서 하고, Extension은 접수에 집중합니다.

이 글에서는 "빨라졌다", "이탈이 줄었다" 같은 수치 표현을 쓰지 않았습니다. 공유 흐름은 `share_id` 하나로 이벤트(`reel_share_received` → `share_delivery_status_changed` → `extraction_request_started/finished`)를 이어 볼 수 있게 PostHog로 남기고 있지만, 전후를 비교할 만한 표본은 아직 확인하지 않았습니다. 분석 이벤트와 Sentry에는 URL·공유 텍스트·토큰을 보내지 않는 원칙을 두었습니다.


---

## 5. 주의할 점 — 이렇게 하면 위험하다

**Extension을 "앱의 한 화면"으로 생각하지 말 것.** 앱의 전역 상태·로그인 정보·API 클라이언트를 전혀 쓸 수 없습니다. 필요한 값은 앱이 미리 App Group 저장소에 내려놓아야 하는데, 이 동기화는 **빠져도 앱이 멀쩡히 동작해서 눈에 띄지 않습니다.** 우리도 세션을 로그인과 토큰 갱신 때만 복사했더니 이전에 로그인해 둔 사용자는 공유 시트에 토큰이 없어 로그인 안내가 뜰 수 있었습니다. 지금은 앱을 실행할 때마다 저장된 세션을 복사하고, 공유 시트가 더 새 토큰으로 회전했다면 앱이 그것을 먼저 가져옵니다(#502).

**긴 작업을 Extension 안에서 하지 말 것.** 요청을 `await`로 기다린 뒤 카드를 띄우는 구조는 정상 네트워크에서는 잘 되다가 **느린 네트워크에서만** 요청이 유실되어 재현이 어렵습니다. 오래 걸릴 수 있는 일은 백그라운드 `URLSession`에 맡기고, 꼭 동기적이어야 하는 일(토큰 갱신)은 타임아웃을 짧게 두고, 서버가 오래 걸리는 일은 **접수까지만** 하고 결과는 앱이 조회합니다.

**공유 데이터 형식을 가정하지 말 것.** 한 가지 타입만 읽으면 어떤 사용자에게는 영영 안 됩니다. 여러 경로를 시도하고, 끝내 못 찾으면 "지원하지 않는 공유"로 안내합니다. 지원하지 않는 링크처럼 **예상 가능한 사용자 입력은 오류 보고에서 제외**해야 대시보드가 정상 동작으로 가득 차지 않습니다. 호스트는 `contains`가 아니라 **일치**로 검사합니다(`evilinstagram.com` 방지).

**모든 실패에 종료 경로를 만들 것.**

| 상황 | 보여주는 것 | 기기에 남기는 것 |
|---|---|---|
| Instagram이 아닌 링크 | "릴스·게시물만 공유할 수 있어요" | 없음 |
| 로그인 필요 | "로그인 해주세요" + 버튼 | 버튼 누르기 전엔 없음 |
| 네트워크 없음 | "공유 완료!"(접수 기준) | URL 보관, 앱 진입 때 전송 |
| 저장소 쓰기 실패 | "릴스를 전달하지 못했어요" | 없음 |
| 앱을 열지 못함 | "앱을 직접 열어 주세요" | URL 보관 |
| 전송 중 응답 유실 | (카드는 이미 닫힘) | 실패 결과만, **자동 재시도 없음** |

마지막 줄이 가장 불편합니다. 응답이 유실되면 사용자는 "공유 완료!"를 보았는데 장소가 보관함에 생기지 않습니다. **중복 저장보다 누락을 택한** 정책이며, 서버에 멱등키가 생기면 재시도 정책을 다시 정해야 합니다.

**앱을 여는 것에 의존하지 말 것.** 비공식 대체 경로는 OS 업데이트에 약합니다. 앱이 열리지 않아도 사용자가 다음 행동을 알 수 있게 안내 카드를 두세요. 이 연결 경험은 시뮬레이터만으로 확인할 수 없어 **실기기·여러 iOS 버전 점검이 필요**하며, 우리도 실기기 확인 항목으로 남겨 둔 상태입니다.

**이런 경우에는 쓰지 않는 편이 낫습니다.** 공유 직후 폴더·메모처럼 **추가 입력을 꼭 받아야 한다면** Extension에 복잡한 UI를 넣기보다 앱으로 넘기는 방법 A가 낫습니다. 요청이 오래 걸리고 **반드시 한 번만** 수행되어야 한다면(결제 등) 멱등키 없이 이 구조를 쓰면 안 됩니다.

**개인정보.** 공유된 URL은 사용자의 관심사를 드러낼 수 있습니다. 버튼을 누르지 않은 URL은 저장하지 않고, 임시 저장 URL은 30분 뒤 만료하며, 전송이 시작되면 결과 저장소의 URL을 비우고 업로드 본문 파일은 완료 시 삭제합니다.

---

## 6. 처음 붙이는 분을 위한 순서

1. **Target 추가** → Bundle ID가 앱의 하위 ID인지, 서명 팀이 같은지 확인
2. **App Groups** → 앱과 Extension에 **같은 ID**로 활성화
3. **빈 화면부터** → Extension이 시트에 나타나고 `viewDidLoad`가 불리는지 먼저 확인
4. **URL 추출** → `inputItems → attachments → loadItem`, 공유하는 앱마다 타입을 로그로 확인
5. **공유 저장소** → App Group에 한 줄 써서 앱에서 읽히는지, Keychain은 앱에서 넣고 Extension에서 읽히는지 확인
6. **전송** → 처음엔 `URLSession.shared`로 확인한 뒤 백그라운드 세션으로 교체, 모든 경로에서 `completeRequest`가 불리는지 확인
7. **실기기** → 실제 인스타그램 공유, 비행기 모드, 로그아웃 상태, 앱 강제 종료 상태를 한 번씩

## 7. 마치며

처음에는 "공유 화면을 하나 더 만드는 일"이라고 생각했습니다. 해 보니 **앱 하나를 더 만드는 일**에 가까웠습니다. 별도의 프로세스에서, 앱의 메모리를 쓰지 못한 채, 짧은 시간 안에 끝내야 하는 작은 앱입니다. 그래서 설계의 핵심은 화면이 아니라 세 가지로 모였습니다.

- **무엇을 앱과 나눠 쓸 것인가** — App Group, 공유 Keychain
- **무엇을 지금 하고 무엇을 맡길 것인가** — 접수까지만 하고 나머지는 백그라운드 전송과 앱
- **실패했을 때 어디까지 보장할 것인가** — 한 번만 호출하고, 누락을 감수하는 정책


---
