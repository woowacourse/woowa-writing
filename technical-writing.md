안녕하세요. 이번 글에서는 아맞다 서비스에서 PWA를 활용해 웹 알림 시스템을 구축한 과정을 공유합니다. 웹환경에서도 모바일 앱처럼 알림을 보낼 수 있도록 만들기 위해 어떤 고민과 선택을 거쳤는지 다루었습니다.

이 글은 웹 환경에서 푸시 알림을 구현하려는 개발자, 특히 PWA와 FCM을 처음 도입해보는 분들을 위한 경험 공유 글입니다.

# 문제 상황

아맞다는 속한 집단의 놓치기 쉬운 이벤트들을 쉽게 전달받을 수 있도록 돕는 **이벤트 리마인더 서비스**입니다. 서비스의 핵심은 사용자가 이벤트를 ‘놓치지 않도록 알림을 제공하는 것’이었고, 따라서 **안정적이고 즉각적인 알림 시스템 구축**이 중요한 과제였습니다.

하지만 팀은 모두 웹 프론트엔드 개발자로 구성되어 있었기 때문에, 모바일 앱처럼 동작하는 알림 시스템을 웹 환경에서 구현하는데 어려움이 있었습니다.

> “웹에서 알람을 보낼 수 있는 기술이 무엇이 있는가?”
> “PWA, Electron, FCM은 어떤 차이가 있을까?”

이러한 질문에서 출발해, 웹에서도 푸시 알림이 동작하는 원리와 기술적 배경, 그리고 PWA와 FCM을 활용해 실제로 알림 시스템을 구축한 과정을 공유하려 합니다.

특히 다음과 같은 내용을 중심으로 다룰 예정입니다.

- **웹에서 알림이 가능한 구조적 원리**
  (Service Worker는 어디서 동작하며, 왜 브라우저를 닫아도 살아있을까?)
- **PWA가 웹을 앱처럼 바꾸는 메커니즘**
  (홈 화면에 추가가 가능한 이유와 standalone 실행 원리)
- **FCM을 이용한 푸시 알림 전달 흐름**
  (토큰 발급 → 서버 연동 → 백그라운드 알림 수신 과정)
- **브라우저별 제약과 대응 방법**
  (iOS Safari의 한계, 권한 요청 UX 개선 등)

# PWA vs Electron 비교

웹에서 푸시 알림을 구현하기 위해 **PWA**와 **Electron** 두 가지 방식을 검토했습니다. 두 기술 모두 웹 기반이지만, 실행 환경과 배포 방식이 크게 다릅니다.

| **기술**     | **장점**                             | **단점**                          |
| ------------ | ------------------------------------ | --------------------------------- |
| **Electron** | OS API 접근 가능, 브라우저 제약 없음 | 앱 용량 큼, 빌드·보안 설정 복잡   |
| **PWA**      | 설치 간편, 서버 업데이트 즉시 반영   | iOS 일부 제약, 앱스토어 배포 불가 |

우리 서비스의 핵심은 `알림을 빠르고 쉽게 전달하는 것` 이었습니다. 따라서 파일 접근 같은 권한 기능은 불필요했고 가볍고 즉각적인 사용자 경험이 중요했습니다.

Electron은 확장성은 뛰어나지만, OS별 빌드/배포/보안 설정이 복잡해 작은 수정에도 재빌드와 재설치가 필요했습니다. 반면 PWA는 브라우저 기반이라 한번의 배포로 모든 사용자가 최신 버전을 사용할 수 있다는 점이 강점이었습니다.

따라서 단일 알림 기능을 위해 Electron은 과도한 선택이었고, 아맞다의 알림 시스템은 “가벼움과 즉시성”이 핵심 가치였기에
별도 빌드 없이 웹만으로 푸시를 처리할 수 있는 PWA가 가장 합리적인 선택이었습니다. 따라서 **최종적으로 PWA를 선택하게 되었습니다**.

# **웹에서 알림이 가능한 이유: PWA의 구조 이해하기**

## PWA의 구조의 세가지 축

PWA는 아래 **세 가지**가 충족되면, 브라우저가 웹을 **설치 가능한 애플리케이션**으로 취급하고 홈화면에 추가와 백그라운드 실행을 허용합니다.

| **구성 요소**        | **역할**                                                    |
| -------------------- | ----------------------------------------------------------- |
| **Service Worker**   | 브라우저 밖에서 백그라운드 동작 (푸시, 캐시, 오프라인 대응) |
| **Web App Manifest** | 앱의 이름, 아이콘, 시작 화면, 색상 등 앱 정체성 정의        |
| **HTTPS 환경**       | 보안 통신, 푸시 및 서비스워커 동작 필수 요건                |

## 설치는 어떻게 이뤄지나?(홈 화면에 추가)

브라우저는 manifest.json의 메타데이터를 인식하면 사이트를 **설치 가능한 앱**으로 판단하고 beforeinstallprompt 이벤트를 발생시킵니다. 사용자가 설치를 수락하면 OS에 앱 정보가 등록되고, 홈 화면에서 **주소창이 없는 독립 실행(standalone)** 모드로 실행됩니다.

## **과거의 웹은 왜 알림이 불가능했나?**

일반적인 웹 페이지의 JavaScript는 탭이 닫히면 함께 종료됩니다. 즉, 브라우저가 닫힌 상태에서는 **백그라운드에서 코드가 실행될 수 없기 때문에** 과거의 웹에서는 푸시 알림이 원천적으로 불가능했습니다.

## **Service Worker: 웹을 스스로 동작하게 만드는 핵심 엔진**

**Service Worker**는 웹앱이 **브라우저 밖에서도 동작할 수 있도록 해주는 백그라운드 스크립트**입니다.

페이지 스레드와 분리된 **브라우저 내부의 독립적인 스레드**에서 실행되어 페이지가 닫혀 있어도 브라우저(또는 OS의 푸시 서비스)가 워커를 깨워 네트워크 요청 가로채기(fetch), 푸시 수신(push), 캐시 관리(offline) 같은 작업을 수행합니다.

이 덕분에 웹은 브라우저가 닫힌 이후에도 알림 수신/오프라인 실행/자동 업데이트가 가능한 앱처럼 동작할 수 있습니다.

단순히 UI를 앱처럼 보이게 만드는 것만으로는 웹을 앱답게 만들 수 없습니다. **페이지 외부에서도 스스로 실행될 수 있는 환경**을 구축해야 하는데요. 그 역할을 하는 것이 바로 **Service Worker**입니다.

- 서버 요청 없이 캐시를 통해 앱을 실행할 수 있고 (오프라인 실행)
- 푸시 신호를 받아 브라우저를 깨워 알림을 표시하며 (백그라운드 동작)
- 새 버전이 배포되면 자동으로 업데이트를 수행합니다.

결국 **Service Worker는 웹앱이 브라우저의 제어권을 일부 가져오는 통로**입니다.

이 시점부터 웹은 더 이상 단순한 문서가 아니라, **스스로 동작하고 업데이트할 수 있는 하나의 애플리케이션**이 됩니다.

# **FCM을 이용한 알림 시스템 구현**

웹에서 푸시 알림을 구현하기 위해 FCM(Firebase Cloud Messaging)을 사용했습니다.

FCM은 서버와 브라우저(클라이언트) 사이에서 메시지를 중계해주는 서비스로, 클라이언트가 발급받은 **고유 토큰**을 기반으로 특정 사용자에게 푸시를 전달할 수 있습니다.

### 전체 흐름

1. 사용자가 브라우저에서 **알림 권한 허용**
2. 클라이언트가 FCM SDK를 통해 **고유 토큰 발급**
3. 발급받은 토큰을 서버에 전송 및 저장
4. 서버가 FCM API로 해당 토큰에 **푸시 메시지 전송**
5. **서비스워커**가 백그라운드에서 메시지를 수신하고 알림 표시

## 초기화 및 토큰 발급

### FCM 초기화

FCM을 사용하려면 서비스워커 내부에서 Firebase를 초기화해야 합니다. 이 파일은 브라우저가 직접 접근하므로 **/public 디렉토리**에 위치해야 합니다.

```jsx
importScripts(
  "https://www.gstatic.com/firebasejs/9.23.0/firebase-app-compat.js"
);
importScripts(
  "https://www.gstatic.com/firebasejs/9.23.0/firebase-messaging-compat.js"
);

firebase.initializeApp(firebaseConfig);
const messaging = firebase.messaging();
```

_importScripts를 사용하는 이유는 서비스워커가 일반 ES6 모듈 환경이 아니기 때문입니다._

### 토큰 발급 및 서버 등록

사용자가 알림을 허용하면, 브라우저는 고유한 FCM 토큰을 발급받습니다. 이 토큰은 특정 **기기+브라우저 조합을 식별하는 값**으로, 서버가 이 토큰을 저장해 두면 이후 해당 사용자에게 푸시를 보낼 수 있습니다.

```tsx
const token = await getToken(messaging, {
  vapidKey: process.env.FCM_VAPID_KEY,
});

if (token) {
  await registerFCMToken(token);
}
```

_VAPID Key는 웹 푸시 인증을 위한 공개 키이며, Firebase 콘솔의 클라우드 메시징 설정에서 발급받습니다._

## **알림 수신 및 표시**

### 포그라운드 메시지 수신 (사용 중인 탭에서 수신)

```jsx
export const setupForegroundMessage = () => {
  onMessage(messaging, (payload) => {
    if (Notification.permission === "granted") {
      new Notification(payload.notification?.title || "새 알림", {
        body: payload.notification?.body || "",
        icon: "/icon-192x192.png",
        data: payload.data,
      });
    }
  });
};
```

앱이 열려 있는 상태에서 FCM 메시지를 수신하면, 서비스 워커 대신 클라이언트 단에서 직접 시스템 알림을 표시합니다.

### 백그라운드 메시지 수신 **(탭이 닫힌 상태)**

```jsx
messaging.onBackgroundMessage((payload) => {
  const notificationTitle = payload.notification?.title || "새 알림";
  const notificationOptions = {
    body: payload.notification?.body || "내용 없음",
    icon: "/icon-192x192.png",
    data: payload.data,
  };

  self.registration.showNotification(notificationTitle, notificationOptions);
});
```

페이지가 닫혀 있어도 브라우저는 Service Worker를 깨워 푸시 메시지를 수신하고, 시스템 알림을 표시합니다.

### 알림 클릭 처리

```jsx
self.addEventListener("notificationclick", (event) => {
  event.notification.close();
  const urlToOpen = event.notification.data?.url || "/";

  event.waitUntil(
    clients.matchAll({ type: "window" }).then((clientList) => {
      for (const client of clientList) {
        if (client.url === urlToOpen && "focus" in client) {
          return client.focus();
        }
      }
      if (clients.openWindow) {
        return clients.openWindow(urlToOpen);
      }
    })
  );
});
```

사용자가 알림을 클릭하면, 이미 열려 있는 창으로 포커스를 이동하거나, 없다면 새 창을 엽니다.

## 브라우저 권한 및 토큰 관리

모든 주요 브라우저에서 알림 권한 요청은 **사용자의 명시적 동작(클릭 등)** 내에서만 가능합니다. 특히 iOS Safari는 이 제약이 더 엄격합니다. 페이지 로드 시 자동으로 권한을 요청하면 무시되므로, 사용자가 버튼을 클릭했을 때 요청하도록 구현해야 합니다.

```tsx
export const NotificationButton = ({ onClose }: NotificationButtonProps) => {
  const { permission, isLoading, handleNotificationClick } = useNotification();

  const handleNotifyClick = () => {
    handleNotificationClick().then((success) => {
      if (success) {
        onClose();
      }
    });
  };

  return (
    <Button size="full" onClick={handleNotifyClick} disabled={isLoading}>
      {isLoading ? '설정 중...' : permission === 'granted' ? '알림 허용됨' : '알림 받기'}
    </Button>
  );
```

# 트러블슈팅 및 주의할 점

## 개발 환경에서의 테스트 방법

FCM 푸시는 로컬 환경에서 바로 테스트하기 어렵기 때문에, Firebase Console의 **테스트 메시지 전송 기능**을 활용했습니다.

### **방법**

1. 대상 기기에 앱을 설치하고 실행합니다. Apple 기기에서는 원격 알림을 수신할 수 있는 권한 요청을 수락해야 합니다.
2. 앱을 기기에서 백그라운드 상태로 만듭니다.
3. Firebase Console에서 [메시지 페이지](https://console.firebase.google.com/project/_/messaging/?hl=ko)를 엽니다.

 <img width="439" height="208" alt="Image" src="./images//메시지 페이지.png" />

4. 첫 번째 메시지인 경우 **첫 번째 캠페인 만들기**를 선택합니다.
   1. **Firebase 알림 메시지**를 선택하고 **만들기**를 선택합니다.
5. 그렇지 않으면 **캠페인** 탭에서 **새 캠페인**을 선택한 후 **알림**을 선택합니다.

<div style="display: flex; gap: 8px;">
  <img src="./images/캠페인 생성 버튼.png" alt="캠페인 생성 버튼" width="48%" />
  <img src="./images/알림 버튼.png" alt="알림 버튼" width="48%" />
</div>

6. 메시지 본문을 입력후, 오른쪽 창에서 **테스트 메시지 전송**을 선택합니다.

 <img width="539" height="178" alt="Image" src="./images/제목 입력 후 테스트 메시시 전송.png" />

7. **FCM 등록 토큰 추가** 필드에 이 가이드의 앞선 섹션에서 가져온 등록 토큰을 입력합니다.

  <img width="339" height="208" alt="Image" src="./images/fcm 토큰.png" />

8. 알림 제목과 토큰을 모두 입력했다면, **테스트 메시지 전송을** 선택합니다.

  <img width="539" height="178" alt="Image" src="./images/이후 테스트 메시지 전송.png" />

### **결과**

테스트 결과, 브라우저에서 정상적으로 알림이 오는 것을 확인할 수 있습니다.

<img width="539" height="138" alt="Image" src="./images/테스트 결과.png" />

### **알림이 안온다면, 시스템 설정에서 크롬 알림 설정 확인 부탁드립니다.**

<img width="439" height="478" alt="Image" src="./images/알림 허용.png" />

# 결과 및 성과

## 기술적 개선 (사용자 경험)

PWA 기반 알림 시스템 도입으로 사용자 경험이 크게 개선되었습니다. 앱 설치 없이도 실시간 이벤트 알림을 받을 수 있게 되었고, 주최자가 게시글을 작성하면 즉시 참가자에게 알림이 전달됩니다. 또한 이벤트 하루 전 자동 리마인더 기능을 통해 참석률 향상에 기여했습니다.

## 서비스 운영 측면의 효과

FCM을 활용하여 별도의 알림 인프라 구축하지 않고도 안정적인 푸시 전송 환경을 마련했습니다. 이를 통해 다음과 같은 이점을 얻었습니다.

- **비용 절감**: 자체 알림 서버 구축 및 유지보수 비용 제로
- **빠른 배포**: PWA 특성상 한 번의 배포로 모든 사용자에게 즉시 반영
- **운영 효율**: 앱스토어 등록 없이 실시간 알림 가능한 웹앱 구현

# 회고 및 배운점

## 기술적 인사이트

이번 프로젝트를 통해 웹 환경에서도 네이티브 수준의 푸시 알림 구현이 가능하다는 것을 확인했습니다. 특히 Service Worker를 활용한 백그라운드 메시지 처리 구조를 구현하면서, 브라우저에서도 Event-Driven Architecture가 효과적으로 동작할 수 있음을 체득했습니다.

또한 이번 경험을 통해 서비스 워커/PWA에 대해 이해할 수 있었습니다.

## 한계점 및 개선 과제

### **플랫폼별 제약사항**

- iOS Safari의 푸시 알림 제약 (iOS 16.4 이상에서만 지원)
- 사용자 권한 요청 타이밍 제어의 어려움
- Chrome에서 한 번 거절한 권한은 사용자가 직접 설정에서 변경해야 함
- ‘홈 화면에 추가’ 인지 부족 — PWA는 설치형 앱과 달리 사용자가 직접 ‘홈 화면에 추가’를 수행해야 하지만, 이 과정을 명시적으로 안내하지 않으면 설치율이 낮게 유지되는 한계가 있습니다.

### 향후 개선 방향

1. **권한 요청 UX 개선**

   알림의 필요성을 먼저 안내하고, 첫 이벤트 참여 등 **자연스러운 시점**에 권한을 요청할 예정입니다.

2. **알림 개인화**

   사용자별로 알림 빈도나 타입을 조정할 수 있는 **맞춤형 알림 설정 기능**을 제공할 예정입니다.

3. **모니터링 및 분석**

   **전송 성공률·클릭률 등 주요 지표를 시각화**해 플랫폼별 성능을 추적하고 개선에 반영할 계획입니다.

4. **PWA 설치 UX 개선**

   사용자가 ‘홈 화면에 추가’ 기능을 자연스럽게 인식할 수 있도록 브라우저의 `beforeinstallprompt` 이벤트를 감지해 맞춤형 배너나 가이드 모달을 노출하는 방식으로 개선할 계획입니다.

---

### 참고 자료

https://firebase.google.com/docs/cloud-messaging/js/client?hl=ko
https://firebase.google.com/docs/cloud-messaging/js/first-message?hl=ko
https://gist.github.com/ninanung/3c3520359abed543a2bb8e09e49212e4
https://quickchabun.tistory.com/161
https://developer.mozilla.org/en-US/docs/Web/Progressive_web_apps/Manifest/Reference
