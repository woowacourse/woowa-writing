# Context 쉽고 재미있게 이해하기

---

## 목차

### Chapter 0: 들어가며

### Chapter 1: Context란?
- **Context는 맥락이다**
- **안드로이드의 Context도 비슷해요!**
- **안드로이드 Context의 정의**

### Chapter 2: Context 더 이해하기
- **Context의 두 종류**
  - Application Context
  - Activity Context
- **두 Context의 차이점: 비유를 들어 이해하기**
  - Application Context 대신 Activity Context를 사용한다면?
  - Activity Context 대신 Application Context를 사용한다면?

### Chapter 3: Context 올바르게 사용하기
- **Context 사용의 기준: Lifecycle**
- **Context 사용 예시: 코드로 이해하기**
  - Activity Context가 필요한 상황
  - Application Context가 필요한 상황
- **결론**

---

<br>

# Chapter 0: 들어가며
안드로이드를 개발해 본 경험이 있으신가요? 그렇다면, Context에 대해서 잘 알고 계시나요?   
아마 잘 알지는 못하더라도 최소한 한 번쯤 들어보았거나 직접 사용해 봤을 것입니다.

Context는 안드로이드 앱에서 매우 중요한 역할을 합니다.  
하지만 개발을 진행하면서, 그 역할이 무엇이고 어떤 때에 사용되는지 잘 모르는 경우가 많습니다.  
때로는 개발자가 어떤 특성이 있는지 모르는 채 사용하기도 하여, 애플리케이션의 비정상적인 동작을 일으키기도 합니다.

저도 Context에 대해 잘 모르고서 개발을 이어가다가 테스트 도중 애플리케이션이 비정상적으로 종료된 경험이 있습니다.  

![dialog_applicaion_context](./technical_writing_images/dialog_applicaion_context.png)  
Fragment에서 다이얼로그를 띄우는 코드입니다.  
Dialog 생성자에 Context를 넘겨줄 때 큰 고민 없이 `applicationContext` 를 사용했습니다.  

![error_메시지](./technical_writing_images/error_메시지.png)  
그러나 다이얼로그를 띄우는 시점에서, 위와 같은 에러가 발생하며 앱이 종료됐습니다. 🤔  

원인은 올바르지 않은 Context를 넘겨준 데에 있었습니다. `applicationContext`가 아니라 Activity의 Context를 넘겨주어야 하기 때문에, 아래와 같이 코드를 수정해야 합니다.  

```kotlin
private fun showConfirmationDialog() {
    val dialog =
        Dialog(requireActivity()).apply {  // applicationContext가 아닌 Activity의 Context를 넘겨줍니다.
            // ...
        }
}
```

에러를 수정하는 과정에서 왜 이런 비정상 종료라는 현상이 발생했고, 또 Context가 어떤 역할을 하는 것인지 궁금해졌습니다.  
그래서 제가 Context를 공부하며 이해한 내용을 바탕으로, 여러분께 Context가 무엇인지 비유를 들어 쉽게 설명해 드리려 합니다.

<br>

# Chapter 1: Context란?

## Context는 맥락이다

Context란 무엇일까요?

안드로이드의 Context를 이해하기 전에, 'Context'라는 단어의 뜻을 먼저 살펴보겠습니다.

> Context : 문맥, 맥락

Context란, **문맥**, **맥락**이라는 뜻을 가진 단어입니다.  
**맥락**은 어떤 이야기나 대화의 **주된 주제 및 흐름, 배경**을 의미합니다.

> Q. 나이가 어떻게 되세요?  
> A. 저는 개발이 정말 좋아요!

위 예시처럼 나이가 어떻게 되냐고 물었는데 개발을 정말 좋아한다고 대답한다면, 좀 뜬금없고 대화의 흐름이 어색하겠죠? 맥락에서 벗어난 대답을 했다고 볼 수 있어요.  
이처럼 맥락은 대화나 이야기 속 주된 주제를 나타내며, 대화, 이야기가 **자연스럽게 이어지도록 중심 역할을 해주는 아주 중요한 요소**입니다.

> Q. 나이가 어떻게 되세요?  
> A1. 올해로 26살입니다.  
> A2. 몇 살처럼 보이나요?  
> A3. 6살이에요. 🤥

다르게 말해서, **맥락이 자연스럽게 이어질 수 있다면**, 대화나 이야기의 **구성 요소로서 어떤 것이든 올 수 있습니다.**  
위의 대화 예시처럼 질문에 대한 대답이 여러 가지가 될 수 있어요. 나이를 묻는 말에 대한 대답은 각각 다른 내용이지만, 그 맥락은 모두 같고 자연스러운 흐름이 됩니다.

## 안드로이드의 Context도 비슷해요!

안드로이드에서의 **Context**도 '맥락'이라는 의미와 크게 다르지 않습니다. 소설 또는 영화의 이야기 흐름에 빗대어서 설명하겠습니다.  

![소설과 영화의 흐름](./technical_writing_images/소설&영화의_흐름.png)  
소설과 영화에는 이야기의 근간이 되는 배경이 설정되어 있습니다.  
언제를 기준으로 진행되는 이야기인지, 어디서 사건이 일어나는지 등을 나타내는 시대적, 공간적 배경이 있습니다.  
그리고 이런 배경 속에서 이야기의 전체적인 줄거리가 흘러갑니다. 이야기가 처음 시작하여, 크고 작은 사건들이 발생하고, 마지막에 마무리된다는 이야기 흐름 말이죠. 이렇듯 소설과 영화에는 이야기를 나타내는 전반적인 ‘맥락’이 존재합니다.


애플리케이션은 **소설, 영화와 같은 한 편의 이야기**로 볼 수 있습니다.  
![애플리케이션의 흐름_1](./technical_writing_images/애플리케이션의_흐름_1.png)

애플리케이션은 안드로이드 시스템 위에서 실행됩니다.  
이야기의 밑바탕이 되는 시대, 공간적 배경과 등장인물, 무대 장치 등의 정보들로부터 이야기가 진행되는 것처럼,
애플리케이션도 **앱이 실행되는 환경과 시스템에서 제공해주는 다양한 서비스, 리소스 자원 등의 정보**를 기반으로 실행됩니다.

즉, 애플리케이션이 시작하고 종료되기까지 흐름이 자연스럽게 이루어질 수 있도록,
**애플리케이션의 실행에 필요한 자원과 정보를 애플리케이션에게 제공해주는 역할**을 하는 것이 안드로이드 시스템의 맥락, Context입니다.

이야기는 챕터로 구분된 여러 사건들이 발생하며, 사건이 진행될 때에도 맥락이라는 부가 정보가 필요합니다.  
사건이 일어난 배경과 상황이 있을 것이고, 지금의 사건으로 새로운 사건이 파생될 수 있습니다.  
또 새로운 등장 인물이 나타나거나, 사건 해결의 실마리를 던져주는 장치가 나타나기도 하죠.  

마찬가지로, 애플리케이션의 화면을 구성하는 Activity에도 Context라는 맥락이 존재합니다.  
Activity는 애플리케이션이라는 **이야기 속에서 진행되는 크고 작은 사건 또는 챕터**에 비유할 수 있습니다.  
Activity가 실행되어 기능을 제공하거나 사용자와 상호작용을 하기 위해서, **시스템의 서비스, 장치나 리소스 자원과 같은 정보가 필요**할 수 있습니다.  
이 정보들 역시 Context에 접근하여 제공받을 수 있습니다.

![애플리케이션의 흐름_2](./technical_writing_images/애플리케이션의_흐름_2.png)  
정리하자면, 안드로이드에서의 **Context**는 애플리케이션의 **흐름 및 구성요소**, 현재 화면에서 보여주고 있는 것들에 대한 **흐름 및 구성요소**를 나타냅니다.  


## 안드로이드 Context의 정의
**맥락**이라는 의미에 집중하면서, 공식 문서의 내용을 참고해 조금 더 자세하게 알아보겠습니다.

> Interface to global information about an application environment.
> This is an abstract class whose implementation is provided by the Android system.  
> It allows access to application-specific resources and classes,
> as well as up-calls for application-level operations
> such as launching activities, broadcasting and receiving intents, etc.

[안드로이드 공식 문서](https://developer.android.com/reference/kotlin/android/content/Context)에 나타나있는 설명입니다. 정리하면 아래와 같습니다.

- 애플리케이션 환경에 관한 글로벌 인터페이스
- 추상 클래스로 구현되어 있으며, 안드로이드 시스템에서 구현체 제공
- 애플리케이션의 특정 리소스와 클래스에 대한 접근 가능
- 상위 애플리케이션 레벨의 API 호출 가능(Activity 실행, Intent 송수신 등)

Context로 애플리케이션 환경에 접근하여 여러 리소스와 클래스, 기능에 접근할 수 있습니다.  
또한 화면 실행 및 전환 등 Activity 레벨에서는 할 수 없는 작업들이 존재하는데요.  
Context를 이용해서 상위 Application 레벨의 API를 호출하여, 상위 레벨의 작업들을 수행할 수 있습니다.

요약하자면, Context는 애플리케이션 실행에 관한 정보들과 관련된 자원 및 기능들을 제공하고 관리하는 역할을 합니다.

<br>

# Chapter 2: Context 더 이해하기

앞서 애플리케이션을 소설과 영화에 빗대어 표현했습니다.  
소설, 영화에서 전체 이야기를 관통하는 맥락과, 각 챕터 또는 하나의 사건에 대한 맥락으로 구분했습니다.  
안드로이드의 Context도 두 종류로 나눌 수 있습니다.  
애플리케이션이 실행, 종료되기까지 전체 이야기에 관한 Context가 있습니다. 그리고 각 화면이 시작되고 종료되는, 즉 Activity에 관련된 Context가 있습니다.

이렇게 두 가지의 Context로 분류되며, 각각 Application Context와 Activity Context라고 합니다.

## Context의 두 종류

### Application Context

Application Context는 애플리케이션 전역에서 사용되는 Context입니다.
[공식 문서](https://developer.android.com/reference/kotlin/android/content/Context#getapplicationcontext)에 나타나는 설명을 참고하여
Application Context의 특징에 대해 살펴보겠습니다.  

> Return the context of the single, global Application object of the current process.  
> This generally should only be used if you need a Context whose lifecycle is separate from the current context, 
> that is tied to the lifetime of the process rather than the current component.

위 공식 문서의 설명을 바탕으로, Application Context의 특징을 하나씩 살펴보겠습니다.

- 단일 전역 Application 객체의 Context를 반환

Application Context가 싱글톤 인스턴스로서 존재한다는 것을 의미합니다.

- 전체 프로세스의 수명에 연결되어 있음

Application 의 생명 주기와 연결되어있다는 것을 의미하며, 앱의 시작부터 종료까지 생존합니다.

- 현재 화면 흐름과 별도의 생명 주기를 가진 Context가 필요한 경우에만 사용

이는 Application Context를 사용할 때의 주의점에 관한 것인데, **무척 중요한 내용입니다.**  
잠시 뒤에 자세히 살펴볼게요!

<br/>

### Activity Context

다음으로는 Activity Context의 특징을 살펴보겠습니다.

- Activity에서 사용되는 Context 입니다.
- 특정 Activitiy의 생명 주기에 종속됩니다.  
Activity 안에서만 사용이 가능하며, 특정 Activitiy의 생명 주기에 종속되어 있습니다.
- Activity 범위 안에서 사용되거나, Activity와 같은 생명 주기를 가진 객체를 생성할 때 사용합니다.

마지막 특징은 Activity Context를 사용할 때의 주의점입니다. 

## 두 Context의 차이점: 비유를 들어 이해하기

Context에는 Application Context와 Activity Context 두 종류가 있다는 것을 알았습니다.  
그렇다면, 둘 중에 어떤 것을 사용해야 할까요? 아무 것이나 사용해도 되는 것일까요?  
아닙니다. 적절한 Context를 사용하지 않은 경우, 애플리케이션의 비정상적인 종료를 유발할 수 있으므로 각별한 주의가 필요합니다.  

그렇다면 어떤 상황에서 어느 것을 사용해야 할까요? 이를 이해하기 위해서는 두 Context의 중요한 차이점에 대해 짚고 넘어가야 합니다.  
두 Context의 가장 큰 차이점은 바로 Lifecycle, 즉 **생명 주기**입니다.  
위에서 언급했던 두 Context의 사용 시 주의점을 살펴보면, 모두 생명 주기에 연관되어있는 것을 알 수 있습니다.

- Application Context : **_현재 화면 흐름과 별도의 생명 주기_** 를 가진 Context 가 필요한 경우에만 사용
- Activity Context : **_Activity 범위 안에서 사용_** 되거나, **_Activity 와 같은 생명 주기_** 를 가진 객체를 생성할 때 사용

즉, Activity 의 생명 주기에 종속되어 있느냐(Activitiy Context), 그렇지 않느냐(Application Context)에 따라 사용하는 Context가 달라집니다.

<br>

위의 두 주의점을 보아서는 Context를 잘못 사용했을 때 어떤 문제점이 나타날지 이해하기 어렵습니다.  
그러니 이번에도 간단한 비유를 들어서 조금 더 쉽게 알아볼게요!

### Application Context 대신 Activity Context를 사용한다면?

![activity_context의_잘못된_사용_비유](./technical_writing_images/activity_context의_잘못된_사용_비유.png)
영화 촬영에 빗대어 보겠습니다.  
그리고 Application Context를 영화 전체 줄거리와 촬영에 필요한 장비들을 잘 알고 있는 **감독 및 본부 팀**이라 가정하고,  
Activity Context는 한 장면에서 **배역에 따라 연기를 하는 배우들**이라고 가정해보겠습니다.

촬영에 사용될 새로운 조명이 들어왔습니다. 이 조명을 관리할 인원이 필요한 상황입니다.   
일반적으로는 감독 또는 본부 팀(Application Context)에게 관리를 맡길 텐데,  
만약 배우들(Activity Context)에게 관리를 맡긴다면 어떻게 될까요?

배우들은 자신들의 장면 촬영이 모두 끝났는데도, 다음 장면에서도 사용될 조명을 관리해야 하기 때문에 퇴근을 할 수 없습니다.  
계속해서 퇴근하지 못하고 일을 하고 있는 배우들로 인해 인력이 낭비되고, 인건비가 증가하게 될 것입니다.  
잘못하다가는 예산이 부족하여 영화 촬영이 망할 수도 있습니다.

**메모리 누수 현상**

![메모리_누수_문제_1](./technical_writing_images/메모리_누수_문제_1.png)
예산 부족에 포인트를 두어 더 자세히 설명드리겠습니다.  
Application Context를 사용하는 경우는 애플리케이션 전역에서 사용되는 객체, 또는 라이브러리가 Context가 필요한 경우입니다.  
DataBase 인스턴스 등을 사용할 때, Application Context가 필요합니다.  

![메모리_누수_문제_2](./technical_writing_images/메모리_누수_문제_2.png)
만약 DB와 같은 객체가 Application Context가 아닌 Activity Context를 사용한다면, 그림과 같이 Activity Context를 참조하게 됩니다.  
그런데 이 Activity가 종료되고 더 이상 사용하지 않게 되어도, DB는 메모리 상에 여전히 남아있고 계속해서 Activity의 Context를 참조합니다.  
이 경우 메모리 공간을 관리해주는 Garbage Collector가 참조가 남아있으며 여전히 사용 중이라고 판단하게 되고, 사용하지 않는 Activity를 메모리에서 지울 수 없게 됩니다. 

![메모리_누수_문제_3](./technical_writing_images/메모리_누수_문제_3.png)
이렇듯 객체에 대한 참조가 남아있어 메모리에서 지울 수 없게 되어 객체가 메모리 공간을 차지하고 있는 것을 메모리 누수 현상이라고 부릅니다.  
Activity에 의해 메모리 누수가 발생한 상황에서, 다른 Activity가 실행되거나 다른 객체가 인스턴스화 되어 메모리에 올라간다면 메모리 공간이 부족해지고,
잘못하면 서비스가 비정상적으로 종료될 수 있습니다.

> Activity Context 라는 배우가 계속 남아서 일을 하게 되면서 인건비가 증가하고, 결국에는 메모리라는 예산이 부족해질 수 있습니다.

### Activity Context 대신 Application Context를 사용한다면?

메모리 누수 현상을 피하기 위해서 Application Context만 사용해도 되지 않을까요? 
그렇지 않습니다. 이 역시도 영화 촬영에 비유해서 설명해보겠습니다.

![application_context의_잘못된_사용_비유](./technical_writing_images/application_context의_잘못된_사용_비유.png)
영화 촬영이 시작되었습니다.  
연기를 하는 배우들을(Activity Context) 촬영해야 하는데, 만약 감독과 본부 팀(Application Context)을 촬영한다면 어떻게 될까요?  
이 사람들은 배우들보다 연기를 못하기 때문에, 촬영에 어려움을 겪게 될 수 있습니다.  

이렇듯 Application Context만 사용하면 촬영, 즉 보여주는 것에 문제가 발생할 수 있습니다.

Activity Context를 사용해야하는 상황은 Activity와 같은 생명 주기를 가진 객체가 Context를 필요로 하는 경우,   
그리고 View와 관련된 UI 작업에 Context가 필요한 경우입니다.  

Context에 접근하여 여러 가지 리소스를 얻어올 수 있는데, 그 중 하나는 Theme입니다.
Context는 xml에서 설정한 Theme, 즉 테마에 대한 정보도 가지고 있습니다.
그런데 Activity Context 대신 Application Context를 사용하면, Activity에서 사용하는 테마가 아닌 Application의 테마가 적용될 수 있습니다.   
Application의 테마란 애플리케이션 전역에 설정된 기본 테마를 의미합니다.
한 화면에 맞추어 디자인된 테마가 아니라 기본 테마가 설정되면 UI 출력에 문제가 발생할 수 있습니다.

![dialog_applicaion_context](./technical_writing_images/dialog_applicaion_context.png)  
![error_메시지](./technical_writing_images/error_메시지.png)  
또한 View를 그릴 때에 필요한 요소들 중 일부를 지원하지 않습니다. 대표적으로 Activity의 윈도우에 대한 접근이 불가능한데요.   
만약 다이얼로그를 띄울 때 Activity Context가 아닌 Application Context를 넘겨준다면,  
다이얼로그 출력 시 Window 접근에 관련된 에러가 발생하며 애플리케이션이 강제 종료됩니다.   

<br>

# Chapter 3: Context 올바르게 사용하기

## Context 사용의 기준: Lifecycle

Context는 다양한 리소스와 기능들을 제공해주지만, 잘못 사용할 경우에는 메모리 누수 등의 문제를 일으킬 수 있습니다.  
그렇다면 이 Context를 어떻게 적절하게 사용해야 할까요?   
안드로이드 구성요소의 생명 주기를 파악하면, 어떤 Context를 사용해야할 지 힌트를 얻을 수 있습니다.  

![생명주기_1](./technical_writing_images/생명주기_1.png)
어떤 객체가 Context를 넘겨받는다는 것은 Context를 넘겨주는 주체, 즉 Activity 또는 Application의 생명 주기에 종속된다는 것을 의미합니다. 

![생명주기_2](./technical_writing_images/생명주기_2.png)
그러므로 Context를 받는 객체들의 생명주기가 얼마나 긴 지를 파악한다면, 어떤 Context를 넘겨주어야 할 지 힌트를 얻을 수 있습니다.


또한, View에 관련된 UI 작업의 경우는 Activity Context를 사용하는 것이 바람직합니다.  

![activity와_views_1](./technical_writing_images/activity와_views_1.png)
여러 View나 Fragment 등의 UI 작업은 결국 Activity 내부에서 이루어집니다.   

![activity와_views_2](./technical_writing_images/activity와_views_2.png)
UI 작업을 행하는 주체는 Activity와 그 생명주기에 종속된 View 또는 Fragment이고,  
이들은 View에 관한 자원에 접근할 수 있어야 하므로 Activity Context를 넘겨주어야 합니다.   


## Context 사용 예시: 코드로 이해하기

이제 Context를 구체적으로 어떤 상황에서 어떻게 사용할 수 있는지 예시와 함께 살펴봅시다.

### Application Context가 필요한 상황

Activity, Fragment 등 안드로이드 구성 요소의 생명 주기를 벗어난 생명 주기를 갖는 객체가 Context가 필요한 경우, Application의 Context를 넘겨주는 것이 적합합니다.  
또는 안드로이드 구성 요소의 생명 주기에 상관 없이, 애플리케이션 내 전역적으로 접근할 수 있는 경우에도 Application의 Context를 넘겨주는 것이 좋습니다.

1. **시스템 서비스 접근**
    - 앱에서 위치 서비스, 알림 서비스, 인터넷 연결 상태 등 시스템에 접근해야할 때 사용합니다.
       ```kotlin
       // 시스템으로부터 SW 키보드를 관리하는 Manager를 가져옵니다.
       val inputMethodManager = getSystemService(INPUT_METHOD_SERVICE) as InputMethodManager
       // 위치 서비스를 가져옵니다.
       val locationManager = context.getSystemService(Context.LOCATION_SERVICE) as LocationManager
       ```


2. **파일 접근 및 저장**
    - 애플리케이션 내부 또는 외부 파일 디렉터리에 파일을 저장하거나 불러올 때 사용합니다.
       ```kotlin
       // getFilesDir, getExternalFilesDir 같은 메서드로 파일 경로를 얻습니다.
       val file = File(context.filesDir, "file.txt")
       ```


3. **SharedPreferences 접근**
    - 간단한 설정 값을 저장하거나 불러올 수 있는 SharedPreference에 접근할 때에도 Context가 필요합니다.
       ```kotlin
       // Context의 getSharedPreferences 메서드를 사용해 SharedPreference에 접근할 수 있습니다.
       val sharedPreferences = context.getSharedPreferences("my_prefs", Context.MODE_PRIVATE)
       val editor = sharedPreferences.edit()
       editor.putString("key", "value").apply()
       ```


### Activity Context가 필요한 상황

Context가 필요한 객체가 Activity의 생명주기에 속하거나, UI 작업과 연관된 리소스에 접근해야 할 때 Activity Context가 필요합니다.

1. **View 동적으로 생성**
    - 코드에서 동적으로 View 객체를 생성해야 할 때 Activity의 Context를 사용합니다.
       ```kotlin
       // 동적으로 View를 생성할 때 사용할 수 있습니다.
       val button = Button(context).apply {
             text = "Click Me"
             layoutParams = LinearLayout.LayoutParams(
                 LinearLayout.LayoutParams.WRAP_CONTENT,
                 LinearLayout.LayoutParams.WRAP_CONTENT
             )
         }
       ```


2. **다이얼로그, 또는 토스트를 표시**
    - 화면에 다이얼로그나 토스트를 띄워서, 사용자에게 원하는 메시지를 보여주고 싶을 때 사용합니다.
       ```kotlin
       // Activity, 또는 Fragment에서 토스트를 띄울 때, Context를 사용합니다.
       Toast.makeText(context, "Hello!", Toast.LENGTH_SHORT).show()
       ```


3. **Application, Activity, Fragment간의 데이터 공유**
   - 특정 Activity, 또는 Fragment에서 다른 Component로 데이터를 전달할 때 사용됩니다. Intent를 함께 사용합니다.
      ```kotlin
      // Intent를 사용하여 다른 Activity를 실행할 때 사용됩니다.
      val intent = Intent(context, AnotherActivity::class.java)
      context.startActivity(intent)
      ```

### Activity Context와 Application Context에 구애받지 않는 경우

반면 두 Context 중 아무 Context를 사용해도 괜찮은 경우도 존재합니다.   
`drawables`, `strings`, `colors` 등 애플리케이션에 정의된 리소스에 접근하는 경우에는 어떤 Context를 사용하는가가 큰 상관이 없습니다.

- **리소스 접근**
    - 문자열 리소스 접근: 앱에서 제공하는 문자열 리소스에 접근하여 화면에 텍스트를 출력할 때 사용할 수 있습니다.
       ```xml
       <resources>
          <string name="app_name">MyApplication</string>
          <string name="format_date">%1$d.%2$d.%3$d</string>
       </resources>
       ```
       ```kotlin
       // 문자열 리소스로부터 application 의 이름을 가져옵니다.
       val appName = context.getString(R.string.app_name)
       // 문자열 리소스에 작성된 포맷 문자열을 가지고 와 동적으로 값을 넣을 수 있습니다.
       val formattedDate = context.getString(R.string.format_date, 2024, 10, 1)
       ```
    - 색상 및 drawable 접근: 애플리케이션에 정의된 색상이나, 이미지, icon 등의 drawable을 가져올 때 사용됩니다.
       ```kotlin
       // 색상을 가져옵니다.
       val color = ContextCompat.getColor(context, R.color.primary_color)
       // drawable 의 icon 또는 이미지를 가지고 올 수 있습니다.
       val closeIcon = ContextCompat.getDrawable(context, R.drawable.icon_close)
       val backgroundImage = ContextCompat.getDrawable(context, R.drawable.image_background)
       ```
      추가로 ContextCompat은 안드로이드 하위 버전과의 호환을 위해서 사용되는 Context입니다.   
      일반적인 Context처럼 사용할 수 있으며, 첫번째 인자로 Context를 넘겨줍니다.


## 결론
이렇듯 Context는 안드로이드 앱에서 중요한 요소들을 관리하고 접근할 수 있는, 매우 필수적인 객체입니다.  
Context의 개념을 잘 이해하고 사용 시 주의점을 잘 지켜내어 올바르게 접근하는 것이 중요합니다.  
아래에 정리된 주의사항만 잘 기억한다면, Context의 잘못된 사용으로 애플리케이션의 비정상적인 종료를 방지할 수 있습니다.

### Application Context가 필요한 경우
- Context를 사용하는 객체가 Activity, Fragment 등 안드로이드 구성 요소의 생명 주기를 벗어난 경우
- Context를 사용하는 객체가 애플리케이션 전역에서 접근할 수 있는 경우

### Activity Context가 필요한 경우
- Context를 필요로 하는 객체가 Activity의 생명주기에 종속된 경우
- Context를 필요호 하는 객체가 View, Fragment, Dialog 등 UI 작업과 관련된 객체인 경우

### 출처

- [안드로이드 공식 문서 - Context](https://developer.android.com/reference/kotlin/android/content/Context)
- [안드로이드 공식 문서 - ApplicationContext](https://developer.android.com/reference/kotlin/android/content/Context#getapplicationcontext)
- [[Android] Context, 너 대체 뭐야?](https://velog.io/@haero_kim/Android-Context-%EB%84%88-%EB%8C%80%EC%B2%B4-%EB%AD%90%EC%95%BC)
- [[Java]가비지 컬렉터(Garbage Collector)란?](https://velog.io/@yarogono/Java%EA%B0%80%EB%B9%84%EC%A7%80-%EC%BB%AC%EB%A0%89%ED%84%B0Garbage-Collector%EB%9E%80)
