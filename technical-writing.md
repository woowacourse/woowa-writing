# 프론트에서 FCM을 이용한 알림 구현하기

## 도입 배경

모우다 서비스는 모임, 채팅, 게시판 등 커뮤니티 성격이 강한 서비스를 제공하고 있다. 이런 상황에서 내부 QA 진행 중, 모임 수정, 취소, 채팅 상황에서 사용자가 즉각적으로 알아야 하는 정보를 파악하기 쉽지 않다는 문제가 있었다. 또한 모임 생성 정보를 서비스에 접속해야 알 수 있다는 것이 모임을 독려하는 데 어려움이 있다고 판단하였다.

이러한 문제를 해결하기 위해 알림 기능을 추가하려고 한다. 이 문서는 웹 애플리케이션에 실시간 푸시 알림을 구현하려는 개발자와, 해당 기능에 대한 이해가 필요한 모우다의 프론트엔드 개발자 간의 지식 공유를 목적으로 작성되었으며, 웹 알림의 개념과 FCM 기반 웹 푸시 알림 구현 방법을 설명한다.


# 웹  알림의 기본 개요

## 1.1 웹 알림이란

웹 알림은 사용자가 브라우저를 통해 특정 웹 사이트나 웹 애플리케이션으로부터 받는 알림 메시지를 말한다. 이러한 알림은 사용자가 웹 페이지에 머물러 있지 않더라도 중요하거나 관심있는 정보를 실시간으로 전달하는 것을 가능하게 한다. 이러한 기능을 통해 서비스 제공자는 이용자에게 지속적으로 서비스 가치를 전달할 수 있으며, 실시간으로 정보를 전달함으로써 사용자 경험을 개선할 수 있다.

브라우저에서의 알림 기능은 특히 크로스 플랫폼 환경에서 중요한 역할을 한다. 즉, 모바일 앱에서만 가능했던 실시간 푸시 알림 기능을 웹에서도 제공함으로써 모바일과 데스크탑 모두에서 일관된 사용자 경험을 제공할 수 있다. 이러한 이유로 웹 알림은 사용자 참여도를 높이고 웹 애플리케이션의 가치를 증대하는 핵심 요소로 자리잡고 있다.

![image.png](asset/image.png)

이러한 알림 기능은 다양한 웹 사이트에서 활용되고 있다. 예를 들어 커뮤니티 기능을 제공하는 서비스 ‘슬랙’과 ‘인스타그램’이 기능을 활용한다. 대화에 읽지 않는 활동이 있을 경우 이를 표시하여 사용자가 서비스를 지속적으로 접근하고 이용을 촉진하도록 만들 수 있다.

## 1.2 웹 알림 기본 요구사항

### 푸시 알림을 구현하기 위해

웹에서 앱과 동일한 push알림을 구현하기 위해 [Notification API](https://developer.mozilla.org/ko/docs/Web/API/Notifications_API), [Push API](https://developer.mozilla.org/ko/docs/Web/API/Push_API)을 사용한다.

### [Notification API](https://developer.mozilla.org/ko/docs/Web/API/Notifications_API)

브라우저가 제공하는 시스템 알림을 표시할 수 있도록 제어할 수 있게하는 API이다. 이러한 알림은 최상단 브라우징 컨텍스트 뷰포트의 바깥에 위치하고 있어 사용자가 탭을 변경하거나 다른 앱으로 이동했을 때에도 표시할 수 있다. 

알림은 기본적으로 두 단계를 거쳐 완성된다. 

첫째, 사용자가 시스템 알림 표시에 대한 권한을 허용해야 한다. [Notification.requestPermission()](https://developer.mozilla.org/en-US/docs/Web/API/Notification/requestPermission_static) 메서드를 호출하여 사용자가 서비스로부터의 알림을 허용하지, 차단할지, 현재 시점에 선택하지 않을 지 선택할 수 있다. 선택된 이후에는, 사용자가 브라우저의 설정을 변경하지 않는 한 앱이나 브라우저가 초기화되기 전까지 유지된다. 

둘째, Notification 생성자를 사용해 알림을 생성한다. 필수값 title인자와 텍스트 방향, 바디 내용, 표시할 아이콘, 재생할 알림 사운드등 옵션을 지정하는 옵션 객체를 선택적으로 사용할 수 있다. 


### [Push API](https://developer.mozilla.org/ko/docs/Web/API/Push_API)

웹이 활성화 되어 있는지 여부와 상관없이 푸시 메시지를 수신할 수 있도록해주는 기능을 제공하는 API이다.  Push API를 사용하기 위해서는 후술할 **Sevice Worker**가 활성화 되어 있어야 한다.



## 1.3 서비스 워커를 이용한 푸시 알림

[Service Worker](https://developer.mozilla.org/ko/docs/Web/API/Service_Worker_API)는 페이지의 메인 javascript와 독립된 스레드에서 실행되며, 브라우저와 네트워크 사이에 존재하므로 브라우저 탭을 닫더라도 네트워크와 통신이 가능하다. 이를 통해 백그라운드에서 푸시 메시지를 수신할 수 있는 환경을 제공한다.

![image.png](asset/image3.png)

서비스 워커는 웹 애플리케이션과 관련 없이 독립적인 라이프사이클을 가진다.

![image.png](asset/image4.png)

### 설치 중 (Installing)

서비스 워커를 등록하면, 자바스크립트가 다운로드 된 후, 파싱되고 나면, **서비스 워커**는 설치 중 상태에 들어가게 된다.

설치가 성공적으로 이루어지면, 설치됨 상태가 되고, 설치 중 오류가 발생하면 서비스 워커는 중복 상태가 된다. 이 경우 페이지를 새로 고침하여 서비스 워커를 다시 등록해야 한다

### 설치됨/대기중 (Installed/waiting)

서비스 워커가 성공적으로 설치되면, 설치됨 상태로 넘어가게 되고, 현재 활성화 되어있는 다른 서비스 워커가 앱을 제어하고 있지 않으면, 바로 활성화 중 상태로 전환된다.

앱을 제어하고 있는 경우에는 대기 중 상태가 유지 된다.

### 활성화 중 (Activating)

서비스 워커가 활성화되어 **앱을 제어하기 전**, activate 이벤트가 발생한다.

### 활성화 됨 (Activated)

서비스 워커가 활성화 되면 페이지를 제어하고, fetch 이벤트와 같은 동작 이벤트를 받을 준비가 된다.

서비스 워커는 페이지 로딩이 시작하기 전에만 페이지 제어 권한을 가져올 수 있다. 즉, 서비스 워커가 활성화 되기 전에 로딩이 시작된 페이지는 서비스 워커가 제어할 수 없다.

### 중복 (Redunant)

서비스 워커가 등록중, 설치 중 실패하거나 새로운 버전으로 교체되면 중복 상태가 된다.

이 상태의 서비스 워커는 앱에 아무런 영향을 미치지 못한다.

서비스 워커의 이런 특성으로 인해 사용에 몇가지 주의사항이 존재한다.

- 서비스 워커는 웹 애플리케이션과 다른 독립적인 라이프 사이클을 가지고 있기 때문에 페이지의 DOM에 접근할 수 없다.
- 보안 상의 이유로 HTTPS에서만 동작한다. 네트워크 요청을 수정할 수 있기 때문에 중간자 공격에 취약하기 때문이다. 단 localhost는 예외이다.

이러한 서비스 워커의 특징 덕분에 웹 애플리케이션이 종료돼도 서비스 워커가 동작하여 알림 메시지를 수신할 수 있다.

## FCM을 이용한 웹 푸시 알림 구현

## 2.1 웹 알림의 흐름(Web Push Protocol)

push 알림을 수신하는 브라우저, 발송하는 서버 사이에 다음같은 상호작용으로 동작한다.

클라이언트는 푸시 서비스로 구독 요청을 보내고 구독에 성공한 경우, 브라우저를 식별할 수 있는 정보를 포함한 구독 정보를 브라우저에게 제공한다. 이 구독 정보를 서버에 저장해 두었다가 푸시 메시지를 보내야할 때, 구독 정보와 메시지를 푸시 서비스로 보내고 푸시 서비스 구독 정보를 바탕으로 클라이언트에서 푸시 메시지를 제공한다.

푸시 메시지를 보낼 때 보안을 위해 VAPID(Voluntary Application Server Identification) 인증 방식을 사용하여 메시지를 안전하게 전송한다. 서버에서 푸시 서비스에게 푸시 알림 요청을 보낼 때, 일련의 정보가 담긴 JWT를 VAPID비공개 키로 암호화한다. 푸시 서비스는 VAPID 공개키를 사용하여 서버의 푸시 알림 요청에 대한 유효성을 검증한다.
![image.png](asset/image 6.png)

## 2.2 push 서비스로 FCM을 이용하자

Web Push Protocol을 쉽게 구현하기 위해 FCM을 이용할 수 있다. FCM을 이용하면 무료로 Push 서비스를 구현할 수 있고 공식 문서와 인테넷에 관련 자료를 얻기 쉽다고 판단하여 결정하게 되었다.

FCM을 사용하가 위해서는 몇가지 설정이 필요로 하다. 

1. **SDK 추가**

[firebase](https://firebase.google.com/?hl=ko)로 이동하여 console로 이동 후, 프로젝트를 생성해야 한다. firebase에서 제공하는 가이드 라인에 따라 프로젝트를 생성 후, 프로젝트 설정을 확인하면 SDK를 설정하는 방법을 확인할 수 있다.

![image.png](asset/image7.png)

위의 코드를 프로젝트의 src폴더에 넣어 설정할 수 있다.

2. **Notification 권한을 받은 후, VAPID키 발급**

사용자에게 알림 권한을 받고, 브라우저에 해당하는 고유의 토큰을 발급 받아야한다. 토큰을 받기 위해서는 먼저 VAPID키를 발급 받아야한다. 처음에 생성한 프로젝트에서 ‘클라우드 메시징’에서 웹 푸시 인증서에서  키쌍을 생성할 수 있다.

다음으로 알림 권한을 요청하는 함수를 작성한 코드다.

```jsx
 import { getMessaging, getToken } from 'firebase/messaging';

import { app } from './initFirebase';
import checkCanUseFirebase from '@_utils/checkCanUseFirebase';

const messaging = checkCanUseFirebase() ? getMessaging(app) : null;

export function requestPermission(mutationFn: (currentToken: string) => void) {
  if (!checkCanUseFirebase()) return;
  console.log('권한 요청 중...');
  Notification.requestPermission().then((permission) => {
    if (permission === 'granted') {
      console.log('알림 권한이 허용됨');
      //@ts-expect-error 파이어베이스가 사용되면 messaging이 존재
      getToken(messaging, {
        vapidKey: process.env.VAPID_KEY,
      })
        .then((currentToken) => {
          if (currentToken) {
            console.log(currentToken);
            mutationFn(currentToken);
          } else {
            // Show permission request UI
            console.log(
              'No registration token available. Request permission to generate one.',
            );
            // ...
          }
        })
        .catch((err) => {
          console.log('An error occurred while retrieving token. ', err);
          // ...
        });
      // FCM 메시지 처리
    } else {
      console.log('알림 권한 허용 안됨');
    }
  });
}
```

다음 코드는 알림을 요청하고 토큰을 발급받는 코드예시이다.

Notification.requestPermission()함수를 사용해 사용자가 알림을 허용한 경우,앞에서 발급받은 VAPID와 초기화된 Firebase 앱 인스턴스로 생성된 messaging 객체를  getToken함수를 인자로 넘겨주어 고유한 토큰을 발급받을 수 있다. 이 발급받은 토큰을 서버에 넘겨주어 저장한다.

3. **메시지 수신 설정**

알림 메시지에는 두가지 형식이 존재한다. 웹 어플리케이션이 동작하는 상황에서 수신할 수 있는 foreground 메시지와 service worker가 백그라운드에서 동작할 때 수신할 수 있는 background 메시지가 있다.

먼저 forground 메시지를 설정하는 코드를 설명한다.

```jsx
const messaging = getMessaging(app);

  onMessage(messaging, (payload) => {
    console.log('포그라운드 알림 도착: ', payload);

    const notificationTitle = payload.notification?.title || '알림';
    const notificationOptions = {
      body: payload.notification?.body || '',
      icon: payload.notification?.icon,
      data: { link: payload.fcmOptions?.link || '/' },
    };

    if (Notification.permission === 'granted') {
      try {
        const notification = new Notification(
          notificationTitle,
          notificationOptions,
        );

        notification.onclick = function (event) {
          event.preventDefault();
          window.open(notificationOptions.data.link, '_blank');
        };
      } catch (error) {
        console.error('알림 생성 중 오류 발생:', error);
      }
    } else {
      console.warn('알림 권한이 허용되지 않았습니다.');
    }
  });
```

messaging 객체를 생성하고 onMessage 함수를 이용하여 FCM에서 웹 페이지가 열려 있는 동안 push 메시지를 수신할 때 호출된다. 클라이언트는 FCM SDK를 통해 메시지를 수신하고, 메시지가 도착하면 미리 등록된 onMessage() 핸들러가 자동으로 호출된다. 

알림이 허용되어 있으면, payload객체를 통해 new Notification으로 알림 객체를 생성한다. 추가적으로 알림을 클릭 시, 해당 페이지로 이동하는 이벤트를 추가했다.

다음으로 background 메시지를 설정하는 방식이다.

background에서 메시지를 사용하기 위해서는 서비스 워커 설정이 필요로 하다. 이때 서비스 워커는 public 폴더에 위치해야 한다. 서비스 워커는 해당 사이트 루트 경로에 있어야 정상적으로 동작할 수 있기 때문이다. 

그렇기 때문에 정적인 파일 위치하는 public폴더에 위치시켜야 한다. 웹 애플리케이션은 서비스워커를 firebase SDK를 서비스 워커에 별도로 설치를 해야한다. 이를 위해 브라우저 클라이언트에서 사용한 것과 동일한 firebase 설정 객체를 이용해서 firebase 앱을 초기화 해야한다.
다음은 sevice worker에서 firebase 앱을 설정하는 예시이다.

```jsx
// firebaseConfig.js
const firebaseConfig = {
  apiKey: "YOUR_API_KEY",
  authDomain: "YOUR_PROJECT_ID.firebaseapp.com",
  projectId: "YOUR_PROJECT_ID",
  storageBucket: "YOUR_PROJECT_ID.appspot.com",
  messagingSenderId: "YOUR_MESSAGING_SENDER_ID",
  appId: "YOUR_APP_ID",
  measurementId: "YOUR_MEASUREMENT_ID"
};

self.firebaseConfig = firebaseConfig;
```

```jsx
//firebase-messaging-sw.js
importScripts(
  "https://www.gstatic.com/firebasejs/10.8.0/firebase-app-compat.js"
);
importScripts(
  "https://www.gstatic.com/firebasejs/10.8.0/firebase-messaging-compat.js"
);

importScripts('/firebaseConfig.js');

self.addEventListener("install", function () {
  self.skipWaiting();
});

self.addEventListener("activate", function () {
  console.log("fcm service worker가 실행되었습니다.");
});

firebase.initializeApp(firebaseConfig);
```

public폴더에서는 모듈이 동작하지 않기 때문에 importScripts를 사용한다.

여기서 firebase-messaging-sw.js에 다음과 같은 코드를 추가하여 커스텀된 알림을 사용할 수 있다.

```jsx
messaging.onBackgroundMessage((payload) => {
    const notificationTitle = payload.title;
    const notificationOptions = {
        body: payload.body
        // icon: payload.icon
    };
    self.registration.showNotification(notificationTitle, notificationOptions);
});
```

하지만 background 메시지는 기본적으로 firebase SDK에서 알림 메시지를 처리하고 있다. 그래서 다음과 같이 커스텀 알림 메시지를 사용할 경우 알림 메시지가 두번 오는 결과가 발생한다. 그래서 커스텀 알림을 사용할 필요가 없는 경우, 굳이 위의 코드는 필요하지 않다.

또한 알림을 클릭 했을 때, 이벤트도 설정할 수 있다.
예시로 알림을 클릭 했을 시, 해당 페이지로 이동하는 로직을 추가했다.

```jsx
self.addEventListener('notificationclick', function(event) {
  console.log('[firebase-messaging-sw.js] 알림이 클릭되었습니다.');

  // 알림 데이터를 가져오기
  const link = event.notification.data.FCM_MSG.notification.click_action;

  event.notification.close(); // 알림 닫기

  // 사용자가 알림을 클릭했을 때 해당 링크로 이동
  if (link) {
    event.waitUntil(
      clients.matchAll({ type: 'window', includeUncontrolled: true }).then(windowClients => {
        // 이미 열린 창이 있는지 확인
        for (let i = 0; i < windowClients.length; i++) {
          const client = windowClients[i];
          if (client.url === link && 'focus' in client) {
            return client.focus();
          }
        }
        // 새 창을 열거나 이미 있는 창으로 이동
        if (clients.openWindow) {
          return clients.openWindow(link);
        }
      })
    );
  }
});
```

background 알림 메시지를 받기 위한 설정을 완료하였다.

마지막으로 작성한 서비스 워커 코드를 브라우저에 등록할 필요가 있다.

```jsx
if ('serviceWorker' in navigator) {
  navigator.serviceWorker
    .register(`/firebase-messaging-sw.js`)
    .then((registration) => {
      console.log('Service Worker registered with scope:', registration.scope);
      initializeForegroundMessageHandling();
    })
    .catch((error) => {
      console.log('Service Worker registration failed:', error);
    });
} else {
  // 서비스 워커가 지원되지 않는 경우에도 포그라운드 메시지 처리를 초기화
  initializeForegroundMessageHandling();
}
```

처음에 if ('serviceWorker' in navigator)는 브라우저가 Service Worker 기능을 지원하는지 화인하고 /firebase-messaging-sw.js라는 파일을 Service Worker로 등록한다. 

4. **지원하지 않는 브라우저 예외처리하기**

모바일 환경에서 safri나 카카오톡 공유를 통한 링크 통해 서비스에 접속하여 모임 목록 페이지에 진입한 경우, 흰색화면이 출력되는 경우가 있다. 이때 개발자 도구를 확인하면 다음과 같은 에러가 발생한 것을 확인할 수 있다.

![8.png](asset/8.png)

이는 해당 환경의 브라우저에서는 Notification을 지원하지 않아 에러가 발생하여 javascript 코드가 중단되어 서비스를 이용하지 못하는 문제이다. 특정 브라우저나 환경에서 Notification을 지원하지 않기 때문에 알림 기능을 지원하지 않을 수 있다. 하지만 이 때문에 어떤한 설명도 없이 전체 서비스를 이용하지 못하는 것은 사용자 관점에서 큰 문제다. 

Notification을 지원하는 브라우저인지 확인해서 지원하지 않는 브라우저라면 해당 코드를 호출하지 않는 방식으로 문제를 해결할 수 있다.

```jsx
import { isSupported } from 'firebase/messaging';
export default async function checkCanUseFirebase() {
  if (location.hostname === 'localhost') return true;
  if (location.protocol !== 'https:') return false;
  const messagingSupported = await isSupported();
  if (!messagingSupported) {
    console.error("This browser doesn't support Firebase Messaging.");
    return false;
  }
  return true;
}
```

해당 함수는 알림 서비스를 이용할 수 있는 브라우저인지 확인하는 함수로 Firebase Messaging 지원 여부를 확인하는 isSupported를 사용하여 검사할 수 있다. 

해당 함수를 initializeFirebaseApp함수 내부에 호출하여 false인 경우는 initializeFirebaseApp를 바로 return하는 방식으로 구현하면  모바일에서 safri나 카카오 브라우저에서 알림을 제외한 서비스를 정상적으로 이용할 수 있다.

```jsx
export const initializeFirebaseApp = async () => {
  const canUseFirebase = await checkCanUseFirebase();
  if (canUseFirebase) {
    return initializeApp(firebaseConfig);
  } else {
    console.warn('Firebase는 이 환경에서 지원되지 않습니다.');
    return undefined;
  }
};
```


## FCM을 이용하면서 발생한 유의사항

- public 폴더에서 .env를 사용할 수 없다. 그렇기 때문에 해당 SDK 키가 github에 노출될 위험이 존재한다. firebaseConfig.js로 별도로 분리하고 배포 과정에서 동적으로 파일을 생성하고 빌드하도록 구성하고 있다.
- Safari나 Firefox같은 경우 푸시 알림 악용을 방지하기 위해 알림 허용 요청을 발생시키기 위해서는 사용자 제스처가 촉발되어야 알림 권한 허용창을 제시할 수 있다.
<!-- - 크롬과 Firefox에서는 사이트가 보안 콘텍스트(즉, HTTPS)가 아니면 알림을 아예 요청할 수 없으며 크로스 오리진 [<iframe>](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/iframe)으로부터의 알림 권한은 요청할 수 없다.
- 23년 3월, 애플이 이번 iOS 16.4 버전부터 웹을 통한 푸시 알림을 허용하면서 아이폰 사용자들도 사파리나 크롬 등을 통해 웹푸시를 수신할 수 있게 되었다. 하지만 애플에서 웹 푸시의 기능을 제한하기 위해 ios와 ipad os에서 웹 알림을 사용하기 위해서는 PWA로 웹앱을 설치해야 한다. -->

## 참고자료

https://developer.mozilla.org/en-US/docs/Web/API/Notification

https://developer.mozilla.org/ko/docs/Web/API/Push_API

https://firebase.google.com/docs/cloud-messaging?hl=ko

https://velog.io/@chchaeun/Service-Worker%EB%A1%9C-%EC%9B%B9%EC%97%90%EC%84%9C-%EC%82%AC%EC%9A%A9%EC%9E%90%EC%97%90%EA%B2%8C-%EC%95%8C%EB%A6%BC%EC%9D%84-%EB%B3%B4%EB%82%B4%EC%9E%90

https://geundung.dev/114