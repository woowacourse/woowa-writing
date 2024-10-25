## 의문의 발단

팀 프로젝트 땅콩을 개발하기 위해 컨벤션을 설정하는 과정에서 팀원과 의견이 달라 토론의 시간을 가졌었다.

논의의 주제는 **@RequestMapping** 설정과 관련된 내용이었는데,
나는 최상위 Resource를 각각의 API Endpoint에 설정해 주는 방안을 지향하는 편이었다.

```java
@RequestMapping("/api")
class MemberControlelr {

    @GetMapping("/members/{memberId}")
    public List findAllMembers() {
    }
}
```



프로젝트가 점점 커지면 API 구현을 확인하기 빠르게 확인하기 위해서는, 최상위 리소스도 API 매핑 정보에 함께 두어야 한다고 생각한다.

실제로 과거 스프링으로 프로젝트를 진행할 때, 종종 Endpoint 를 통해서 컨트롤러의 메서드를 검색한 경험이 있었는데
그때는 지금처럼 최상위 Path를 분리하지 않았었기에 편리하게 탐색이 가능했다.

> 의식하고 그렇게 개발한 것은 아니었다.
> 



또 팀 단위로 서버 개발을 진행한다고 했을 때, 프로젝트에 익숙하지 않은 새로운 팀원이 들어왔다고 가정해 보자.

애플리케이션의 코드에서 찾고자 하는 API 메서드의 Endpoint를 검색했는데,
여러 개의 메서드가 탐색 된다면 새로운 팀원은 여러 컨트롤러 또는 메서드를 돌아다니며 원하는 API 메서드가 맞는지 하나하나 확인해야 할 것이다.

이렇듯 최상위 Path를 분리하는 것은 중복코드를 제거한다는 이점보다, 
개발의 유지보수성을 떨어뜨린다는 문제가 더 크게 다가왔다.

물론 프로젝트 크기가 작을 때는 큰 문제는 안 되겠지만 일회성 프로젝트가 아닌 앞으로 운영 및 유지보수 해나갈 프로젝트를 설계하는데 이런 것을 고려하는 것은 오버 엔지니어링의 영역은 아니라고 생각했다.



### 팀원의 의견

하지만 팀원의 생각은 조금 달랐고 충분히 고민해 볼 만한 포인트를 제시해 주었다.

팀원은 의견에 충분히 공감하지만
클라이언트 측의 요청 발생 시,
서블릿의 핸들러 매핑 과정에서 핸들러를 찾을 때 클래스 레벨에 존재하는 **@RequestMapping** 을 먼저 확인 후
메서드 레벨의 **@RequestMapping** 을 확인하며, 이 때문에 클래스 레벨의 **@RequestMapping** 이 **"/api"** 일 경우 핸들러 매핑 과정에서 성능 이슈가 발생할 것이라고 하였다.

때문에 아래와 같이 클래스 레벨에 최상위 리소스를 명시해서
스프링에서 클래스 레벨의 매핑 정보 확인 시에, 컨트롤러를 특정할 수 있도록 해주어야 한다고 하였다.



```java
@RequestMapping("/api/members")
class MemberController {

    @GetMapping("/{memberId}")
    public List findAllMembers() {
    }
}
```

이렇게 설정해야 `/api/members` 를 통해 해당 컨트롤러 객체를 탐색하고,
이 좁은 범위 내에서 또다시 API 메서드를 찾기 때문에 성능 저하가 적을 것이라고 의견을 제시한 것이다.

나는 팀원의 내용이 사실인지에 대해 호기심이 생겨, HandlerMapping 의 과정을 인텔리제이 디버거를 통해 살펴보게 되었다.



---

# 동작 확인해 보기

## API Request 시점

먼저 요청 시 어떤 핸들러 매핑이 동작하는지 확인하기 위해 Request 시점부터 확인해 보기로 하였다.



### 갑자기 RouterFunctionMapping?




이전에 Spring MVC를 학습할 때는 RequestMappingHandlerMapping이 가장 먼저 동작한다고 배웠기 때문에
RequestMappingHandlerMapping 클래스에 브레이크 포인트를 걸어보았다.





그런데 결과가 내 예상과는 달랐다.

RequestMappingHandlerMapping에 설정한 브레이크 포인트가 RouterFunctionMapping 클래스의 getHandlerInternal 메서드로 이동했다.

다른 핸들러 매핑 클래스로 스텝 오버(Step Over)가 발생한 것이다.





왜인지 살펴보기 위해 조금 더 전 시점으로 올라가서 DispatcherServlet이 handlerMapping을 반환하는 시점으로 이동해서 원인을 파악할 수 있었다.





내가 알고 있던 RequestMappingHandlerMapping이 0순위가 아니라, RouterFunctionMapping이 0순위로 수행되고 있던 것이었다.





getHandler 메서드의 docs를 보면 순서에 따라 핸들러 매핑이 수행된다고 하는데,
0순위가 RouterFunctionMapping이었다.

> 이 시점에서 RequestMappingHandlerMapping에 설정해 둔 브레이크 포인트가 RouterFunctionMapping으로 스탭오버가 발생한 이유는 아직 잘 모르겠다.
> 





다시 본론으로 돌아와서 그럼 이곳부터 핸들러 매핑 과정을 살펴보겠다.

this.routerFunction이 null이기 때문에 바로 getHandlerInternal()은 return null을 수행한다.



![7](https://github.com/user-attachments/assets/cdec36ff-2341-47ef-b80f-a09677ab1a58)

- 그럼 AbstractHandlerMapping이라는 클래스의 getHandler로 값(null)이 넘어오게 되는데
그 값은 handler에 저장된다.
- 그리고 handler가 null이기 때문에 getDefaultHandler() 메서드가 실행된다.
    - 근데 그 값 또한 null이라서 null을 return 한다.
 


![8](https://github.com/user-attachments/assets/2dc765d4-470b-47fd-8d55-ab39cfa69fb9)

해당 값을 전달받은 DispatcherServlet은 handler를 반환하지 못해서 다음 순번 HandlerMapping에게 핸들러 탐색권이 넘어가게 되는데..!



### 왜 RouterFunctionMapping은 실패한걸까?

RouterFunctionMapping은
Spring 구성에서 하나 이상의 RouterFunction 빈을 감지하고, 순서를 지정하고, RouterFunction.andOther를 통해 결합한 다음, 결과적으로 구성된 RouterFunction으로 요청을 라우팅한다.

<br>

<img width="560" alt="9" src="https://github.com/user-attachments/assets/41983e41-521b-4d4d-af7f-92b77bfb2611">


대략 이런 형태의 Bean으로 Mapping 되는 것인데, 일반적인 경우로는 잘 사용되지 않을 것 같다.

우리가 개발하는, 그리고 대부분의 개발자는 RequestMapping을 사용한다. RouterFunction 빈으로 핸들러 메서드를 정의하지 않았으니 잡히지 않는 것이다.

좀 더 자세히 알아보면 좋겠지만 이번에 알고자 하는 부분은 RequestMapping에 성능 이슈가 발생하는지에 대한 것이니 우선 넘어가자.

WebFlux의 기능으로 보이는데 자세히 알고 싶다면 [**래퍼런스**](https://docs.spring.io/spring-framework/reference/web/webflux-functional.html#webflux-fn-running) 를 참조해 보자.
> 



### RequestMappingHandlerMapping 호출

![10](https://github.com/user-attachments/assets/d03b7ef2-2b39-4173-95d8-8d8fc568c0d9)

방금 실패한 0순위 다음으로 호출되는 것이 바로 1순위인 RequestMappingHandlerMapping이다.

RequestMappingHandlerMapping은 RequestMapping 애노테이션을 통해 정의한 핸들러를 조회할 수 있기 때문에 내 예상대로라면 이번 순번인 RequestMappingHandlerMapping에서 핸들러가 조회될 것이다.

흐름을 따라가 보자.



![11](https://github.com/user-attachments/assets/587b448c-2cc4-4bcc-81af-797e3986f28c)

먼저 getHandler 메서드를 요청하면 AbstractHandlerMapping의 getHandler가 호출된다.

추상화가 되어있는 모양이다.



![12](https://github.com/user-attachments/assets/ba02a8d6-ff35-4612-bef9-89b71c290a0f)

그리고 내부적으로 호출하는 getHandlerInternal은 Super Class인 AbstractHandlerMethodMapping에 request 정보를 담아 getHandlerInternal 메서드를 호출하게 된다.



![13](https://github.com/user-attachments/assets/22858862-0d1c-4945-a292-e5b06f459dfa)

AbstractHandlerMethodMapping의 getHandlerInternal 메서드는
우선 핸들러 매핑 정보를 담고 있는 mappingRegistry에 ReadLock을 걸어 변경을 막는다.



![14](https://github.com/user-attachments/assets/aef0df0d-8fe9-4ef8-bcf6-a57c2027a194)

그리고 lookupHandlerMethod를 수행해서 mappingRegistry의 매핑 정보를 드디어 조회해 오는데..!

드디어 핸들러 메서드를 발견할 수 있었다.

이 시점에 mappingRegistry의 readLock을 해제하고, 발견된 핸들러 메서드는 쭉 리턴되어 DispatcherServlet로 전달한다.



![15](https://github.com/user-attachments/assets/bc0cac8a-9e99-4dc1-9e2e-7e28371b1991)

mappingReigstry는 RequestMapping으로 정의된 핸들러의 매핑정보를 보관하는 레지스트리인데,
위 사진을 통해 알 수 있듯이 클래스 레벨에 설정된 `/api` 엔드포인트와 메서드 레벨에 설정된 `/~` 엔드포인트가 결합되어 관리되는 것을 확인할 수 있다.

나의 궁금증은 클래스 레벨 메서드 레벨의 RequestMapping이 요청 과정에서 하나하나 이루어지냐였기 때문에
여기서 나의 호기심은 1차적으로 해소될 수 있었다.



결과는 **‘그렇지 않다’** 였다.

그리고 어떻게 이렇게 등록되는지도 너무 궁금해서 아래서 살펴볼 예정이다.

![16](https://github.com/user-attachments/assets/84e3e28f-48e5-44a0-940a-56649f74511b)



아무튼 이 과정을 통해서 알게된 점은

- 0순위 핸들러매핑은 RequestMappingHandlerMapping이 아니라 RouterFunctionMapping이라는 점
- 미리 클래스 레벨과 메서드 레벨의 RequestMapping이 조합되어서 관리되기 때문에 팀원이 예상한 성능문제는 없다는 점 (Map에서 관리되니 O(1)로 조회가 될 것으로 예상)

정도가 되겠다.



## Application 초기화 시점

그럼 언제 어떻게 HandlerMapping 정보가 저렇게 만들어지는지도 궁금하지 않은가??

이번엔 Application 초기화 시점을 타고 올라가면서 어떻게 매핑이 되는지 확인해보자.



![17](https://github.com/user-attachments/assets/a2b4eb06-73f9-45f7-9e8f-6809652137eb)


나는 브레이크 포인트를 선정하기 위해 RequestMappingHandlerMapping을 먼저 둘러보던 중 굉장히 수상쩍은 메서드를 발견했다.

해당 메서드 내부에서 호출되는 createRequestMappingInfo 메서드를 보자마자 직감이 와서 getMappingForMethod 에 브레이크 포인트를 걸고 디버깅 모드로 애플리케이션을 실행시켰다.

그리고 애플리케이션 초기화 시점에 해당 브레이크 포인트에 브레이크가 걸렸다.

> 찾아보니 getMappingForMethod 메서드는 애플리케이션 초기화 시점에 호출되어 핸들러 메서드와 요청 매핑 정보를 설정함으로써, 요청 처리를 효율적으로 준비하고 성능을 최적화하는 역할을 한다고 한다.
> 






표시되는 값을 통해 현재 서버에 존재하는 API Endpoint 정보를 매핑하는 과정을 확인할 수 있었는데,
1차적으로 createRequestMappingInfo(method)를 통해 메서드 레벨에 걸려있는 RequestMapping 값을 가져온다.





그 다음 createRequestMappingInfo(handlerType)를 통해 클래스 레벨에 설정된 Endpoint인 `/api` 를 가져와
info에 결합하는 것을 확인할 수 있다.





최종적으로 결합이 완료되면 해당 값을 return 한다.



![21](https://github.com/user-attachments/assets/12063b70-d03e-4968-92fd-ef5bdaff01a5)

만들어진 RequestMappingInfo 객체는 AbstractHandlerMethodMapping 클래스로 전달되고,
여기서 또다시 return 되어서



![22](https://github.com/user-attachments/assets/e0702c73-0977-498e-935d-34c076416883)

MethodIntrospector로 전달되어 result 변수에 담기게 된다.





그리고 null 검증을 거쳐  `methodMap` 에 담긴다.





`methodMap`은 다음과 같은 Key, Value 형태를 가지는데,

- Key는 매핑할 핸들러 메서드를 담는 Reflection API의 Method 타입 객체
- Value는 요청 정보를 담을 RequestMappingInfo 타입 객체

를 가진다.



![25](https://github.com/user-attachments/assets/c35ba1c6-fd9c-4447-9496-ce3daae29ce3)

위 methodMap은 return 되어 AbstractHandlerMethodMapping의 methods 변수에 담기는데,
이는 로깅 등의 처리를 거쳐서 registerHandlerMethod 메서드로 넘어간다.



![26](https://github.com/user-attachments/assets/0e9104d9-cceb-4248-8979-45d9a1b0e605)

그렇게 해서 결론적으로 mappingRegistry에 매핑 정보가 담기게 된다.



---

# 결론

- 팀원이 염려하던 문제는 발생하지 않는다.
    - 애플리케이션 init 시점에, `RequestMappingHandlerMapping`은 모든 `@RequestMapping` 및 구체적인 매핑 애노테이션인 `@GetMapping`, `@PostMapping` 등이 붙은 메서드를 찾고 결합하여 저장한다.
    - 요청이 들어오면 ‘URL 패턴’ 을 기반으로 핸들러 메서드를 검색한다. (내부적으로 Map 타입으로 관리하기 때문에 아마 시간복잡도가 O(1)로 굉장히 빨리 찾을 것이다.)
- `RequestMappingHandlerMapping`보다 `RouterFunctionMapping`이 먼저 동작한다.



---

# 마치며

결론적으로, `@RequestMapping`과 `@GetMapping` 등의 세부 Mapping이 분리되어 있어도
Spring은 이를 결합하여 관리하며, 핸들러 매핑 시에 최적화된 검색을 통해 한 번에 핸들러를 찾을 수 있다,

따라서 팀원의 우려는 실제로는 성능에 큰 영향을 미치지 않는다는 것을 알 수 있었고,
팀원의 의견 덕분에 깊이 생각해 보지 않았던 **HandlerMapping** 과정에 대해 이해할 수 있었던 유익한 학습 과정이었다.



> **[레벨3가 끝난 시점에 적는 글] (2024.08.23)**
실제로 서비스가 커지고 API 메서드가 굉장히 많이 생겨나면서 Endpoint를 통해 API를 검색하는 일이 잦았고, 이 과정에서 해당 내가 제안했던 매핑 설정이 유효하게 작용하였음. 팀원들도 이에 공감하여 이 기록에 글을 남긴다.
결론적으로 괜찮은 방식이라고 확신하게 되었다.
>
