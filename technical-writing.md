# [React] 행동대장 서비스로 알아보는 프론트엔드 에러 핸들링 전략과 개선 과정

<img src="https://velog.velcdn.com/images/pakxe/post/891d28af-a07a-4d60-bed9-100ef1c109a4/image.jpg" />

# ■ 시작하며
프론트엔드 개발을 하다 보면, 에러 핸들링 코드를 작성해야만 하는 순간을 마주하게 됩니다. 혹시 그럴 때마다 try-catch 또는 반복적인 코드를 작성하고 계신가요? 만약 그렇다면 이 글을 가볍게 읽어보시길 추천합니다.

이번 글에서는 제가 참여하고 있는 간편한 정산을 도와주기 위한 [행동 대장](https://haengdong.pro/) 프로젝트에 적용된 에러 핸들링 전략을 소개하고자 합니다. 어떤 방식으로 에러를 처리했는지, 왜 그 방식을 선택했는지, 결과는 어땠는지, 그리고 개선 과정까지 함께 살펴볼 예정입니다.

참고로, 에러 바운더리와 [TanStack-Query](https://tanstack.com/query/latest)에 대한 기본적인 이해가 있다면 본 글을 더 수월하게 읽으실 수 있습니다.

이 글에 적힌 것이 완벽한 답은 아니니 '이런 에러 핸들링 방법도 있구나'라고 생각해 주시면 좋을 것 같습니다.


# ■ 에러 핸들링이 뭐에요?
시작은 에러 핸들링이 무엇인지에 대해서 먼저 알아보겠습니다.

에러 핸들링이란 코드 실행 중 발생할 수 있는 예기치 않은 오류나 문제를 탐지하고, 이를 적절히 처리하여 프로그램이 비정상적으로 종료되지 않도록 하는 기술을 말합니다.

이 글에서는 Toast UI 또는 에러 바운더리를 사용한 Fallback UI에 한정하여 설명드릴 예정입니다.

![](https://velog.velcdn.com/images/pakxe/post/34137a87-5fbd-41f9-87e7-9165c01fbde9/image.png)

위 이미지처럼 발생하는 에러를 처리하는 방법이라고 생각하시면 됩니다.

# ■ 에러 핸들링 왜 하나요?
에러를 적절히 핸들링해주면 애플리케이션이 비정상적으로 종료되는 것을 막을 수 있습니다. 사용자 입장에서는 에러가 발생했다고 알려주지 않고 흰 화면만 보일 경우, 매우 당황스러울 수 있습니다.

또한, 개발자 입장에서 에러 로그를 심어 모니터링하면 자주 발생하는 에러를 파악하기 쉬워지고, 이를 통해 버그의 원인을 빠르게 찾고 수정할 수 있습니다.

# ■ 에러 핸들링 v1
## ▌ 요구사항
이런 장점들을 누리기 위해 제가 참여하고 있는 행동대장 서비스에도 에러 핸들링 버전 1을 도입하게 되었습니다.

요구 사항은 아래와 같았습니다.
> **"에러가 발생하면 Toast UI로 안내해주세요."**

![](https://velog.velcdn.com/images/pakxe/post/54a329af-fcb4-473d-87b4-1bc2cc0ffb83/image.gif)

위 GIF처럼 짧은 메시지를 표시하기 위해 화면에 작은 영역을 차지하는 요소를 Toast UI라고 합니다.

## ▌ 주로 사용되고 있는 에러 핸들링 방법은?
그래서 프론트엔드에서 주로 사용되는 에러 핸들링 방법에는 어떤 것들이 있는지 살펴보았습니다.
대부분 두 가지 방법이 많이 사용되고 있었습니다.

![](https://velog.velcdn.com/images/pakxe/post/4de31112-c85d-4d5a-a099-de113346ed05/image.png)

### ▎ 1. try-catch
그러나 try-catch로는 저희 서비스의 에러 핸들링을 하기엔 무리가 있으리라 판단했습니다.

서비스 개발이 어느 정도 진행된 상태에서 에러 핸들링 기능을 도입하는 것이었기 때문에, API 에러가 발생할 수 있는 위치가 굉장히 많았습니다. 모든 위치에 try-catch를 추가하는 것은 시간이 오래 걸리고, 요구 사항이 바뀔 때마다 해당 위치를 찾아서 수정하는 유지보수도 어려웠기 때문입니다.

### ▎ 2. 에러 바운더리
그래서 2번째 방법인 에러 바운더리는 어떨까 고민해보았습니다.

에러 바운더리는 오류가 발생했을 때 Fallback UI를 띄우는 데 특화된 방법입니다. 하지만 에러 바운더리는 에러 발생 시 Fallback UI로 화면을 교체하기 때문에, 기존 화면을 유지하면서 Toast UI를 띄우는 방식은 사용할 수 없었습니다.

그 이유는 에러 바운더리 내부 구현에 있습니다.

![](https://velog.velcdn.com/images/pakxe/post/1a3a2217-0a49-4f25-bd50-328ffa7bcc9b/image.png)

에러 바운더리는 hasError라는 상태를 갖고 있습니다. 에러가 발생해 이 hasError 값이 변경될 때마다 render함수를 실행합니다. 기본적으로는 render함수에서 children을 반환하고, 에러가 발생했을 때 Fallback UI를 반환합니다..

에러가 발생했을 때도 children을 반환하면 기존 화면을 유지할 수 있지 않을까 생각할 수 있습니다.
![](https://velog.velcdn.com/images/pakxe/post/c5625180-8e89-431b-b540-3813cfa074af/image.png)

그러나, 위 이미지를 통해 이해할 수 있듯이, 이 경우 무한 루프에 빠지게 됩니다.

처음에 render 함수가 실행되어 children을 반환합니다. 만약 children에 있는 API 요청에서 계속 500 에러가 발생하는 상황이라면, 이 에러 바운더리의 hasError가 true가 됩니다. 상태가 변경되면서 다시 render 함수가 실행되지만, children을 반환하고, children에서 다시 500 에러가 발생하여 결국 무한 루프에 빠지게 됩니다.

![](https://velog.velcdn.com/images/pakxe/post/6dceeec5-b33f-499a-8217-8543fa014572/image.png)

따라서 에러 바운더리는 이 서비스의 에러 핸들링 방법으로 사용하기 어려웠습니다. 하지만 에러 바운더리의 특징인 최상단에서 한 번에 에러를 핸들링하는 특성을 활용하고 싶었습니다. 이를 통해 반복되는 코드를 줄이고, 에러 핸들링의 책임을 한 곳에 집중할 수 있기 때문입니다.

그래서 이 특징을 갖는 에러 핸들링 방법을 생각해 구현하게 되었습니다.

## ▌ 핵심 구조
일단 이 에러 핸들링 v1의 핵심 전략은 아래와 같습니다.
> 핵심 전략: **최상단의 업데이터와 구독자**

![](https://velog.velcdn.com/images/pakxe/post/314b7b24-ebf5-480a-8ed7-dd37b8a73019/image.png)

저희 서비스의 모든 api요청은 `request`라는 함수를 거치게 됩니다. 이 `requst`함수에서는 api요청 중 에러가 발생했을 경우 RequestError라는 에러 코드가 담긴 에러 객체를 던집니다. 

그리고 `업데이터`는 request함수에서 던져진 RequestError를 잡아 에러 상태를 던져졌던 RequestError로 업데이트합니다.

이후 이 에러 상태를 구독하고 있는 `구독자`가 RequestError안의 에러 코드를 확인해 적절하게 Toast UI또는 전역 에러 바운더리의 Fallback UI를 띄우게 됩니다.

위 과정을 한 스텝씩 실제 구현 코드와 함께 살펴봅시다.

## ▌ 실제 구현
방금 말씀드렸던 구조는 실제 코드에서 아래와 같은 계층 구조로 사용할 수 있습니다. 
<img width="1103" alt="image" src="https://github.com/user-attachments/assets/240284d1-e19c-48be-ad1d-ef6966959d2e">


`전역 에러 바운더리` 하위에 `업데이터` 역할인 queryClient를 둡니다. 그리고 더 하위에 `구독자` 역할인 ErrorCatcher를 둡니다.
그리고 에러가 발생할 수도 있는 페이지 또는 컴포넌트를 안에 위치시킵니다.

앞서 말씀드렸다시피 저희 서비스의 모든 api요청은 request라는 함수를 거칩니다. 이 안에서 api요청을 보낼 때 에러가 발생했다면 `RequestError` 객체를 생성해 throw합니다.

![](https://velog.velcdn.com/images/pakxe/post/56aa8f75-30dd-4ae2-81ea-70a41d49947c/image.png)

다음은 업데이터 역할입니다. 이 기능은 QueryClientBoundary 컴포넌트에 포함되어 있으며, 이는 Tanstack-Query 라이브러리를 통해 캐싱된 값에 접근할 수 있는 컴포넌트입니다.

Tanstack-Query에서는 요청 수행 중 에러가 발생할 때 실행할 콜백을 지정할 수 있습니다. 이곳에서 에러 객체를 받아 에러 상태를 업데이트하는 updateError를 호출하도록 콜백을 설정했습니다.

![](https://velog.velcdn.com/images/pakxe/post/f361807e-290d-42ea-ab70-14ffec0d522e/image.png)

updateError는 useErrorStore에서 반환하는 함수입니다. useErrorStore는 에러 상태와 에러 상태를 업데이트하는 코드를 반환하는 훅으로, useState와 유사한 역할을 합니다.

마지막으로 구독자 역할의 ErrorCatcher입니다. ErrorCatcher는 업데이트되는 에러 상태를 useEffect로 구독하고 있습니다.

![](https://velog.velcdn.com/images/pakxe/post/1353c378-b797-41a7-8ab0-7134ad68efa4/image.png)

useEffect의 내부 코드를 보면 isPredictableError라는 함수에 errorCode를 넘기고 있는 것을 볼 수 있습니다. 이는 이름 그대로 예측 가능한 에러인지를 확인하는 함수입니다.

예측 가능한 에러인 경우 Toast UI를 띄우고, 예측 불가능한 에러는 그대로 throw하여 ErrorCatcher를 감싸고 있는 전역 에러 바운더리에서 잡아 Fallback UI를 띄우게 됩니다.

![](https://velog.velcdn.com/images/pakxe/post/d4af7c6f-4d1e-45c2-9d30-8e93c939c1c9/image.png)

예측이 가능하다라는 것은 백엔드에서 명확하게 전달해준 에러 코드들을 의미합니다. 예를 들어 이름 길이가 제한보다 긴 경우 INVALID_NAME_LENGTH와 같은 에러 코드가 전달됩니다. 이러한 에러 코드는 백엔드에서 '예측 가능한' 상황에 대해 정의된 것이기 때문에, 예측 가능한 에러라는 표현을 사용합니다.

반면, 예측 불가능한 에러는 INTERNAL_SERVER_ERROR와 같은 코드 또는 서버에서 정의한 에러 코드 목록에 없는 에러를 의미합니다. 서버에서 발생한 에러의 경우 Toast UI로 안내해도 빈 화면만 남을 수 있어 사용자에게 혼란을 줄 수 있습니다. 따라서 예측 불가능한 에러는 전역 에러 바운더리를 통해 Fallback UI로 안내하도록 했습니다.

## ▌ 결과
결과적으로는 위 구현물로 서비스의 에러 핸들링 요구사항인 `에러 발생 시 Toast UI로 안내`를 만족할 수 있었습니다.

이 방법의 장점은 여러 가지가 있었습니다.

성공 케이스와 실패 케이스를 한 곳에 작성하지 않고 분리할 수 있어, 핵심 로직에만 집중할 수 있었습니다. 새로운 기능이 추가되더라도 에러 처리를 별도로 고민할 필요가 없었고, 이미 자동으로 에러 처리가 적용되기 때문입니다.

또한, 일부 컴포넌트가 에러 핸들링을 전담하여 책임 분리가 명확해졌고, 에러 핸들링 전략이 변경되더라도 유지보수에 용이했습니다. 에러 로깅이 필요한 경우에도 한 곳에만 작성하면 되기 때문에 관리가 편리했습니다.

# ■ 에러 핸들링 v2
## ▌ 요구사항
변함없이 사용될 것 같았던 v1 에러 핸들링에 새로운 요구사항이 추가되었습니다.

> **"페이지 초기 렌더링 중 데이터를 받아오는 데 오류가 발생했을 경우 Fallback UI를 사용해 주세요"**

즉, GET 메서드에서 오류가 발생했을 때 에러 바운더리를 사용해 Fallback UI를 띄우라는 의미입니다. 그러나 모든 GET 메서드에서 동일하게 Fallback UI를 띄우기보다는, 오류 발생 상황에 맞는 UI를 선택할 수 있도록 하는 것이 좋다고 판단했습니다.

그래서 주어진 요구사항을 좀 더 확장해 재정의했습니다.

> **"GET 메서드에서 오류가 발생했을 경우, Toast UI 또는 Fallback UI 중 하나를 선택할 수 있도록 합니다. 그 외는 v1 그대로 유지합니다."**

이제 이 요구사항을 충족하는 에러 핸들링 v2를 개발해보겠습니다.

## ▌ 핵심 구조
v2의 핵심 전략은 아래와 같습니다.
> 핵심 전략: **커스텀 에러 객체를 사용한 분기**

![](https://velog.velcdn.com/images/pakxe/post/50d528b7-19d4-4689-8103-25590980941c/image.png)

처음으로는 에러 발생 시 Toast UI, Fallback UI중 어떤 UI를 사용할 것인지에 대해 인자를 받습니다. 

그리고 이 인자를 커스텀 에러 객체에 담습니다. 이 에러 객체는 Tanstack-Query 쿼리에서 제공하는 에러가 발생했을 때 실행하는 콜백의 인자로 넘어갑니다. 

만약 fallback일 경우 조건문으로 early return해 v1에서의 에러 상태를 업데이트하는 updateError함수의 호출을 막습니다. Toast UI가 뜨는 것을 막아야 하기 때문입니다. 

만약 toast일 경우 v1에서 구현한 것들이 그대로 실행되도록 합니다.

위 과정을 한 스텝씩 실제 구현 코드와 함께 살펴보도록 하겠습니다.

## ▌ 실제 구현
api 요청에서 에러가 발생했을 때 어떤 UI를 띄울지는 api 요청 훅의 errorDisplayMode 인자로 넘겨 조작할 수 있도록 했습니다.

![](https://velog.velcdn.com/images/pakxe/post/750dac44-4558-4880-ba24-f6c5a30a8340/image.png)

'toast'를 넘길 경우 v1을 그대로 실행합니다. 
'fallback'을 넘길 경우 에러 바운더리를 사용해 Fallback UI를 보여주도록 합니다.

이때 "GET 메서드에서 오류가 발생했을 때 Fallback UI를 사용해라" 라는 의미에 대해서 생각해볼 필요가 있는데요. 이 의미는 `지역적인 에러 바운더리`를 중첩해 해당 에러 바운더리의 Fallback UI를 사용하겠다는 뜻입니다.
![](https://velog.velcdn.com/images/pakxe/post/1b7f3f6e-2473-4375-8df4-027560c11573/image.png)

다만 v1의 코드 그 대로로는 지역적인 에러 바운더리를 사용할 수 없습니다. 

`Page1` 컴포넌트와 이 Page1 컴포넌트를 감싸는 `LocalErrorBoundary1`이 있다고 해봅시다. 그리고 이 외부의 최상단에는 v1에서 구현한 `전역 에러 바운더리`, `업데이터인 queryClient`, `구독자인 ErrorCatcher`가 위치하고 있습니다. 이런 상황에서는 에러가 발생했을 경우 LocalErrorBoundary1로 에러가 던져지는 게 아니라 바로 업데이터, 구독자로 진입하게 됩니다. Tanstack-Query 쿼리에서 throwOnError를 켜도 그렇습니다.

따라서 v1 그대로 둔다면 지역적인 에러 바운더리로 감싸도 에러 바운더리를 사용할 수 없습니다.

v1도 사용할 수 있도록 하면서 지역적인 에러 바운더리도 사용하기 위해선 실제 업데이터-구독자의 진입을 막으면 됩니다. 진입을 막고 throwOnError 옵션을 킨다면 에러는 에러 발생 컴포넌트로부터 상위 컨텍스트로 
자연스레 흐를 수 있게 됩니다. 

진입을 막는 방법은 업데이터 코드가 있는 queryClient에 분기 문을 추가해주는 것입니다.

![](https://velog.velcdn.com/images/pakxe/post/265dee73-fb74-4c94-afb6-ff5511685562/image.png)

v1 구현 코드에서 보았던 업데이터 코드입니다. query 실행 중 에러가 발생했다면 updateError가 호출되고 있습니다. 

updateError가 호출되기 전에 조건문을 추가해 `GET메서드에서 발생한 에러면서 fallback UI를 사용하겠다고 선언된 에러`라면 early return하도록 합니다.
![](https://velog.velcdn.com/images/pakxe/post/f96bb3e8-3fb4-4fa2-ae03-4424be8e19ef/image.png)

이 콜백함수는 첫 번째 인자로 에러 객체가 주입되고 있기 때문에 실제 코드로는 아래처럼 조건문을 구현할 수 있습니다.
![](https://velog.velcdn.com/images/pakxe/post/62b82ca7-7d15-4f68-b2d9-c3b232a597e3/image.png)

이 코드는 에러 객체로 조건문을 걸고 있습니다. 따라서 에러 객체가 Toast UI를 사용하는 에러인지, Fallback UI를 사용하는 에러인지 정보를 담고 있어야 합니다.

그래서 아래처럼 RequestGetError라는 커스텀 에러 객체를 제작했습니다.

![](https://velog.velcdn.com/images/pakxe/post/33f9b923-532b-4a16-9a64-a6023cea7467/image.png)
 
이 RequestGetError는 생성자의 인자로 errorDisplayMode를 넘겨 생성할 수 있습니다. errorDisplayMode 인자의 값으로 가능한 건 계속 말했듯 'toast'와 'fallback'입니다.

이렇게 구현된 RequestGetError 는 이 서비스의 모든 api가 거쳐 가는 곳인 request함수에서 생성되어 throw됩니다.

![](https://velog.velcdn.com/images/pakxe/post/472f0760-fc3b-4586-8049-4ee1e475af55/image.png)

## ▌ 결과 
### ▎ 1. GET 메서드 에러 시 Fallback UI
이제 GET 메서드에서 에러가 발생했을 경우 Fallback UI와 Toast UI 중 선택해서 띄울 수 있게 되었습니다. 

Fallback UI를 사용하는 경우 아래 코드처럼 사용합니다. 
![](https://velog.velcdn.com/images/pakxe/post/e7f60f2b-a164-435c-9c2b-e67de0aed03b/image.png)

에러가 발생할 수 있는 컴포넌트인 TestComponent를 지역적인 에러 바운더리로 감쌉니다. 그리고 api요청 훅에 errorDisplayMode 인자를 'fallback'값을 넘깁니다. 
![](https://velog.velcdn.com/images/pakxe/post/b382cfd9-6121-488f-8f59-af44cee8d967/image.gif)

Fallback UI가 잘 보입니다.

### ▎ 2. GET 메서드 에러 시 Toast UI
Toast UI를 사용하는 경우 아래 코드처럼 사용합니다.
![](https://velog.velcdn.com/images/pakxe/post/ec370bdf-2c62-4f08-af24-ca4b0f98bf93/image.png)

에러 바운더리로 감싸줄 필요 없고, errorDisplayMode인자의 값만 'toast'로 잘 넘겨주면 됩니다.

![](https://velog.velcdn.com/images/pakxe/post/afc7dfd3-a882-4b03-89f3-8737be523745/image.gif)

Toast UI가 잘 보입니다.

---
v2를 구현하는 건 v1보단 어렵지 않았는데요. 아마 책임을 잘 분리해두었기 때문에 빠르게 구현할 수 있던 것 같습니다.

v2로 오면서 지역적인 에러 바운더리를 사용할 수 있게 되었고, 같은 api여도 상황에 맞게 에러 UI를 선택할 수 있는 기능이 추가되었습니다. 

# ■ 마무리
긴 글 읽어주시느라 고생 많으셨습니다. 🙇

이렇게 행동대장 서비스에서 사용하고 있는 에러 핸들링 전략에 대해서 알아보았습니다. 에러는 개발 과정에서 피할 수 없는 존재이지만, 어떻게 대응하고 처리하느냐에 따라 사용자 경험과 서비스의 안정성에 큰 영향을 미칩니다. 이 글에서 다룬 사례와 전략들이 여러분의 프로젝트에 작은 도움이 되었기를 바랍니다.

글을 읽으며 이해가 어려웠던 부분이나 질문하고 싶은 내용이 있으시다면 <a href= "mailto:pigkill40@naver.com" >이메일</a>또는 댓글 남겨주세요.

감사합니다.

> 관련 [PR 링크](https://github.com/woowacourse-teams/2024-haeng-dong/pull/567). 글에서 사용되고 있는 용어와 실제 코드에서의 용어가 다르니 이 점 참고하시길 바랍니다.
