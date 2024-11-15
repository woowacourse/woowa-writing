# Github OAuth2.0을 활용한 소셜 로그인 개발기

대부분의 현대 웹 애플리케이션에서는 사용자 인증이 필수입니다. 특히 소셜 로그인 기능은 사용자에게 간편한 인증 수단을 제공하여 이탈률을 낮추고 사용자 경험을 향상시키는 중요한 요소로 자리잡고 있습니다. 

그리고 OAuth2.0은 바로 이 소셜 로그인에서 인증 및 권한 부여를 위해 사용하는 표준 프로토콜입니다. GitHub OAuth2.0을 통해 개발자는 사용자 인증을 쉽게 구현할 수 있으며 사용자는 자신의 GitHub 계정을 사용하여 간편하게 서비스를 이용 가능합니다. 하지만 OAuth2.0을 처음 접하는 개발자들에게는 개념적인 이해보다도 실제 구현 시에 발생할 수 있는 다양한 이슈들이 더 큰 고민거리겠지요. 저희 팀 CoReA에서도 GitHub의 OAuth2.0을 활용하여 소셜 로그인 기능을 구현하였는데요.

따라서 다음 글은 OAuth의 개념부터 시작하여, OAuth2.0의 구조와 동작 원리, GitHub OAuth2.0 API의 사용법, 더 나아가 그 과정에서 겪었던 다양한 트러블슈팅 경험을 공유하는 단초가 되겠습니다.

CORS 문제, JWT 토큰 관리, refreshToken의 부재 등 다양한 이슈 해결하는 과정을 소개하고 있으니 GitHub OAuth2.0 도입을 고려 중인 서버/클라이언트 개발자들에게 많은 도움이 되었으면 좋겠습니다. 

소셜 로그인의 도입은 더 이상 선택이 아닌 필수인 시대, GitHub OAuth2.0을 활용하여 더 안전하고 간편한 사용자 인증을 구현해보세요!


---

## 1. OAuth란?


> 제3의 서비스에 계정 관리를 맡기는 방식

---
### 개념

**제3자의 클라이언트에게 보호된 리소스를 제한적으로 접근하게 해주는 프레임워크**


### 적용 이유

- *실제 아이디와 비밀번호를 우리가 관리하지 않고, 액세스 토큰을 통해 인가받는 서비스*
    - 비밀번호 유출 등의 보안 문제는 리소스 서버가 관리하니 우리는 신경 쓰지 않아도 됨
    - 사용자 입장에서도 안심

- *우리 서비스를 사용하기 위해서는 기본적으로 모두 github에 계정이 있을 것*
    - 구현 시 모든 유저가 사용할 수 있는 아이디 방식일 거라고 생각

- *OAuth 1.0의 경우 인증 Access Token이 만료되지 않음*
    - 보안 취약점으로 이어질 수 있음




### 1. 용어 정리

---

- **리소스 오너(resource owner)**
  : 자신의 정보를 사용하도록 인증 서버에 허가하는 주체

- **리소스 서버(resource server)**
  : 리소스 오너의 정보를 관리하고 보호하는 주체
  *e.g. 네이버, 구글...*

- **인증 서버(authorizaition server)**
  : 클라이언트에게 리소스 오너의 정보에 접근할 수 있는 토큰을 발급하는 애플리케이션

- **클라이언트 애플리케이션**
  : 인증 서버에게 인증을 받고 리소스 오너의 리소스를 사용하는 주체


### 2. 리소스 오너 정보 취득 방법

---

- **권한 부여 코드 승인 타입(authorization code grant type)**
  : OAuth 2.0에서 가장 잘 알려진 인증 방법
  - 클라이언트가 리소스에 접근하기 위해 사용
  - *권한 접근 코드 & 리소스 오너 액세스 토큰을 발급받음*

- **암시적 승인 타입(implicit grant type)**
  : 서버가 없는 자바스크립트 웹 애플리케이션 클라이언트에서 주로 사용
  - *클라이언트가 요청을 보내면 리소스 오너의 인증 과정만 진행*
  - *이외에는 권한 코드 교환 등의 별다른 인증 과정을 거치지 않고 액세스 토큰을 제공 받음*

- **리소스 소유자 암호 자격증명 승인 타입(resource owner password credentials)**
  : *클라이언트의 패스워드를 이용하여 액세스 토큰에 대한 사용자의 자격 증명을 교환*

- **클라이언트 자격증명 승인 타입(client credentials grant)**
  : *클라이언트가 컨텍스트 외부에서 액세스 토큰을 얻어 특정 리소스에 접근 요청*


### 3. 권한 부여 코드 승인 타입

---


1. **권한 요청**
    - 클라이언트(*스프링 부트 서버*)가 특정 사용자 데이터에 접근하기 위해 권한 서버에 보내는 요청
    - 권한 서버(*카카오, 구글...*)에 보내는 요청에는 아래와 같은 것들이 포함
        - `parameter: 클라이언트 ID, 리다이렉트 URI, 응답 타입`

- *권한 요청 예시*

```java
GET spring-authorization-server.example/authorize?
	client_id=66a36b4c2& // 인증 서버가 클라이언트에 할당한 고유 식별자
	redirect_uri=http://localhost:8080/myapp& 
	// 로그인 성공 시 이동 URI
	// 이 경로로 authorization code 전달
	reponse_type=code& // 클라이언트가 제공받길 원하는 응답 타입
	scope=profile // 제공받고자 하는 리소스 오너의 정보 목록
```

2. **데이터 접근 권한 부여**
    - 인증 서버에 요청을 처음 보내는 경우 사용자에게 보이는 페이지를 로그인 페이지로 변경
    - 사용자의 데이터에 접근 동의를 얻는 과정 (최초 1회)
        - `scope` 기준
    - 이후 인증 서버에서 동의 내용을 저장, 로그인 진행
        - `userID`와 인가된 `scope`를 저장
        - 이후 해당 `clientID`와 `userID`로 접근 시 `scope`에 해당하는 데이터는 접근 가능
    - 로그인 성공 시 권한 부여 서버는 데이터 접근 인증 및 권한 부여 수신

3. **인증 코드 제공**
    - 로그인 성공 시 `redirect_uri`(권한 요청에서 보낸 parameter)로 리다이렉션
    - parameter: 인증 코드(authorization code)

- *인증 코드 예시*

```java
GET http://localhost:8080/myapp?code=a1s2d3f4mcj2
```

4. **액세스 토큰 응답**
    - 액세스 토큰: 로그인 세션에 대한 보안 자격을 증명하는 식별 코드
    - 인증 코드를 액세스 토큰으로 교환하는 과정 (refresh token도 이때 배부)
        - 교환 이후 인증 코드는 삭제함
    - 보통 `/token POST` 요청

- */token POST 요청 예시*

```java
POST spring-authorization-server.example.com/token
{
	"client_id": "66a36b4c2",
	"client_secret": "aabb11dd44", // OAuth 서비스에 등록할 때 제공 받은 비밀키
	"redirect_uri": "https//localhost:8080/myapp",
	"grant_type": "authorization_code", // 권한 유형 확인
	"code": "a1b2c3d4e5f6g7h8"
}

// 권한 서버는 요청 값을 기반으로 유효 정보 여부 확인, 유효하다면 액세스 토큰 응답
```

- *권한 서버의 액세스 토큰 응답 예시*

```java
{
	"access_token": "aasdffb",
	"token_type": "Bearer",
	"expires_in": 3600,
	"scope": "openid profile",
	
	...etc...

}
```

5. **데이터 접근(API 응답 & 반환)**
    - 액세스 토큰을 기반으로 리소스 오너의 정보를 받아옴
    - 정보가 필요할 때마다 API 호출로 정보를 요청
        - 리소스 서버는 토큰 유효성 검사 후 응답

- *리소스 오너의 정보 요청 예시*

```java
GET spring-authorization-resource-server.example.com/userinfo
Header: Authorization: Bearer aasdffb
```

6. **상황에 따라 refresh token 사용 (e.g. 자동 로그인)**

---

## 2. GitHub OAuth 2.0 API 설명
   
GitHub OAuth2.0 API는 다양한 엔드포인트를 제공하여 개발자가 인증 및 권한 부여 프로세스를 관리할 수 있도록 합니다. 

1. 사용자가 애플리케이션에 로그인하려면, 사용자는 GitHub의 인증 페이지로 리디렉션됩니다.
2. 이후 사용자 인증이 완료되면, GitHub은 애플리케이션으로 인증 코드를 반환합니다. 
3. 이 인증 코드를 사용하여 클라이언트 애플리케이션은 GitHub의 엔드포인트에 POST 요청을 보내 액세스 토큰을 요청합니다.
```java
POST /login/oauth/access_token 
```
- 요청 시, 클라이언트 ID, 클라이언트 비밀번호, 인증 코드, 리디렉션 URI를 포함해야 합니다.
4. 성공적으로 요청이 처리되면, 응답으로 액세스 토큰과 해당 토큰의 유효기간이 포함된 JSON 객체를 반환받습니다.
5. 이 액세스 토큰을 사용하여 리소스 서버에 요청을 보내고, 사용자의 GitHub 정보를 안전하게 접근할 수 있습니다. 
- 각 엔드포인트의 사용 예제 및 응답 형식은 [GitHub API 문서](https://docs.github.com/ko/apps/oauth-apps/building-oauth-apps/authorizing-oauth-apps)를 통해 자세히 확인할 수 있습니다.

***주의***

깃허브에서는 다른 OAuth2.0과는 다르게 refreshToken을 제공하지 않습니다! 

따라서 사용자가 refreshToken을 활용하기 위해서는 해당 기능을 구현한 서비스가 자체적으로 refreshToken을 생성해야 합니다.

## 3. 개발 트러블슈팅
소셜 로그인 기능을 개발하는 과정에서 다양한 문제를 경험하게 됩니다. 
   
특히 팀 프로젝트에서 GitHub OAuth2.0을 적용하면서 CORS 문제, 토큰 관리, API 호출과 같은 여러 가지 이슈가 발생했습니다. 

예를 들자면 클라이언트에서 API 서버로 요청을 보낼 때 CORS 정책으로 인해 요청이 차단되는 경우가 있었네요. 이 문제를 해결하기 위해 저희 팀은 Interceptor를 통해 Preflight 요청을 처리하는 방법을 적용했습니다. 

또한, JWT 토큰을 사용하여 서버와 클라이언트 간의 인증 정보를 안전하게 전달하는 방법을 고민하게 되었습니다. 액세스 토큰과 refreshToken의 저장 위치에 대한 논의도 많았습니다. 헤더, 쿠키, 세션 등 다양한 옵션을 고려했으며, 최종적으로는 헤더를 사용하게 되었습니다.

### 3.1 CORS

#### 발단

소셜 로그인을 신나게 구현하고 *이제 프론트와 연결만 하면 되겠지~* 라는 안일한 생각과 함께 신나게 귀가한 어떤 날.

![img_1.png](img_1.png)

그렇게 됐습니다.

---
### Spring CORS 기본 코드 - [자세한 설정](https://inpa.tistory.com/entry/WEB-%F0%9F%93%9A-CORS-%F0%9F%92%AF-%EC%A0%95%EB%A6%AC-%ED%95%B4%EA%B2%B0-%EB%B0%A9%EB%B2%95-%F0%9F%91%8F#spring_%EC%84%B8%ED%8C%85)

```java
// 스프링 서버 전역적으로 CORS 설정

@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
        	.allowedOrigins("http://localhost:3000",
        	"https://d2n5lw9a9hwjzs.cloudfront.net/") 
        	// 허용할 출처
            .allowedMethods("GET", "POST", "DELETE", "PUT") // 허용할 HTTP method
            .allowCredentials(true) // 쿠키 인증 요청 허용
            .maxAge(3000) // 원하는 시간만큼 pre-flight 리퀘스트를 캐싱
    }
}
```

기본적으로 CORS가 가능하도록 하기 위해 설정했던 configuration은 위와 같습니다.

하지만 별 문제 없어 보이는 위의 코드 설정에도 소셜 로그인 기능 추가 이후 우리의 Spring은 차갑고 냉정한 에러 메시지를 계속 뱉었습니다.


```java
preflighthandler cannot be cast to org.springframework.web.method.handlermethod
```

이대로 잘 돌아가던 서버가 갑자기 어떤 요청을 해도 난생 처음 보는 CORS ERROR를 뱉어내는 상황이라 사뭇 당황스러웠던 동료 크루와 저는 오전 동안 침울한 얼굴로 구글에

<br>CORS란?
<br>CORS 에러 처리하는 법
<br>preflight는 또 뭔가요, 와 같은 키워드들을 검색하며 각자 유익한 시간을 보냈습니다.

그리하여 알게 된 사실은 다음과 같습니다.

문제는 의외로 CORS 설정이 아니었다는 거지요.

```java

 @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(authorizationInterceptor)
                .addPathPatterns("/**");
    }
```

놀랍게도 범인은 이 녀석이었습니다.

### AuthorizationInterceptor
독자들이 Interceptor의 정의 정도는 대강 알고 있는 멋진 개발자일 것으로 믿으며 바로 전문을 소개하겠습니다.

```java
package corea.auth.interceptor;

import corea.auth.RequestHandler;
import corea.auth.annotation.LoginMember;
import corea.auth.infrastructure.TokenProvider;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Component;
import org.springframework.web.cors.CorsUtils;
import org.springframework.web.method.HandlerMethod;
import org.springframework.web.servlet.HandlerInterceptor;
import org.springframework.web.servlet.resource.ResourceHttpRequestHandler;

@Component
@RequiredArgsConstructor
public class AuthorizationInterceptor implements HandlerInterceptor {

    private static final Class<LoginMember> AUTH_ANNOTATION = LoginMember.class;

    private final TokenProvider tokenProvider;
    private final RequestHandler requestHandler;

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        boolean hasAnnotation = checkAnnotation(handler);
        if (hasAnnotation) {
            String accessToken = requestHandler.extract(request);
            tokenProvider.validateToken(accessToken);
        }
        return true;
    }

    private boolean checkAnnotation(Object handler) {
        if (handler instanceof ResourceHttpRequestHandler) {
            return false;
        }

        HandlerMethod handlerMethod = (HandlerMethod) handler;

		// 메소드 레벨에 붙은 어노테이션 탐색
        if (null != handlerMethod.getMethodAnnotation(AUTH_ANNOTATION) ||
                null != handlerMethod.getBeanType().getAnnotation(AUTH_ANNOTATION)) {
            return true;
        }
		// 파라미터 레벨에 붙은 어노테이션 탐색
        for (Parameter parameter : handlerMethod.getMethod().getParameters()) {
            if (parameter.isAnnotationPresent(AUTH_ANNOTATION)) {
                return true;
            }
        }
        return false;
    }
}
```

이 Interceptor의 존재의의는 다음과 같았습니다.

우리 프로젝트는 로그인하지 않은 상태에서도 디스플레이 가능한 정보들이 꽤 있습니다.

- *현재 모집 중인 방*
- *현재 모집 완료된 방*
- *랭킹* 등등...
- 
하지만 로그인한 상태에서만 디스플레이 되어야 하는 정보들도 당연히 존재합니다.

- *현재 참여 중인 방*
- *마이페이지*
- *피드백 화면* 등등...

- 그러한 정보들의 컨트롤러를 각각 분리해보자는 의견도 있었지만,

지금 당장으로는 @AccessedMember, @LoginMember 두 어노테이션을 만들어
이 두 어노테이션을 각각 파라미터로 받는 컨트롤러 메서드들을 분리하는 것으로 처리되었습니다.

```java
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
@Hidden()
public @interface LoginMember {
}

@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
@Hidden()
public @interface AccessedMember {
}

@GetMapping("/{id}")
public ResponseEntity<RoomResponse> room(@PathVariable long id, @AccessedMember AuthInfo authInfo) {
        RoomResponse response = roomService.findOne(id, authInfo.getId());
        return ResponseEntity.ok(response);
    }

@GetMapping("/{id}/reviewers")
public ResponseEntity<MatchResultResponses> reviewers(@PathVariable long id, @LoginMember AuthInfo authInfo) {
        MatchResultResponses response = matchResultService.findReviewers(authInfo.getId(), id);
        return ResponseEntity.ok(response);
    }
```

대충 이런 식입니다.

따라서 우리는 각 API 요청들이 들어올 때 파라미터에서 어떤 어노테이션을 받고 있는지 확인하고
만약 `@LoginMember`를 파라미터로 받고 있다면 인증 정보를 확인하는 작업을 추가했습니다.
그리고 바로 이 과정에서 문제가 발생합니다.

`AuthorizationInterceptor`가 발동하는 시점에서 `PreFlightHandler`는 아직 `HandlerAdapter`에서 실행되지 않은 상태입니다.

다시 말해 `PreFlightHandler`가 실행되기도 전에 우리의 `AuthorizationInterceptor`가 실행됩니다.

정확히 말하자면 `checkAnnotation`에 작성된

```java
HandlerMethod handlerMethod = (HandlerMethod) handler;
```

이 부분이 `PreFlightHandler`를 호출하고, `HandlerMethod`로 캐스팅하려고 시도하면서 생긴 에러라는 겁니다.

이때 해당 `handler`가 `Controller`인지 `HandlerMethod`인지 알 수 없는 상황에서 무조건 캐스팅을 하려고 하니 문제가 날 수밖에 없습니다.


```java
@Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        if (CorsUtils.isPreFlightRequest(request)) {
            return true;
        } // 추가

        boolean hasAnnotation = checkAnnotation(handler);
        if (hasAnnotation) {
            String accessToken = requestHandler.extract(request);
            tokenProvider.validateToken(accessToken);
        }
        return true;
    }
```

따라서 만약 해당 요청이 본 요청이 아닌 예비 요청이라면
`Preflight`가 아예 `checkAnnotation`로직을 타지 않도록 사전 분기를 해주었습니다.

고작 세 줄만으로 에러를 수정한 모습입니다. 프로그래밍의 묘미라고 생각합니다.

```java
public static boolean isPreFlightRequest(HttpServletRequest request) {  
    return HttpMethod.OPTIONS.matches(request.getMethod()) &&
    request.getHeader("Origin") != null && 
    request.getHeader("Access-Control-Request-Method") != null;  
}
```

## 결론

이처럼 OAuth는 인터넷에서 자원에 대한 권한을 안전하게 부여하는 인증 프로토콜입니다. 

사용자가 비밀번호와 같은 민감한 정보를 제3자 서비스와 공유하지 않고도 특정 리소스(예: 사용자 데이터)에 대한 접근 권한을 부여할 수 있도록 도와줍니다. OAuth는 주로 소셜 로그인 기능을 구현할 때 많이 사용되며 사용자는 Facebook, Google, GitHub과 같은 플랫폼의 계정을 통해 다른 웹사이트나 애플리케이션에 로그인할 수 있습니다.

### OAuth의 주요 용도
- **안전한 인증**: 사용자 정보를 안전하게 보호하면서 다른 애플리케이션과의 연동을 가능하게 합니다.
- **편리한 로그인**: 사용자는 여러 서비스에 대해 별도의 계정을 만들 필요 없이, 이미 존재하는 계정을 이용해 쉽게 로그인할 수 있습니다.
- **접근 제어**: 사용자가 특정 데이터에 대한 접근을 제어할 수 있어, 필요에 따라 권한을 설정할 수 있습니다.

- ### 개발 시 주의해야 할 사항
- **보안**: OAuth를 사용할 때는 항상 보안에 주의해야 합니다. 인증 토큰이나 클라이언트 비밀 키를 안전하게 저장하고 관리해야 합니다. 공개되지 않도록 주의하고 HTTPS를 사용하여 통신을 암호화해야 합니다.

- **Token 관리**: 발급한 액세스 토큰의 유효 기간을 설정하고 만료된 토큰의 갱신 방법을 마련해야 합니다. 또한 사용자에게 적절한 권한을 부여하고 필요한 경우만 최소한의 권한을 주도록 설정해야 합니다.

- **에러 처리**: 인증 과정에서 발생할 수 있는 다양한 오류를 적절히 처리할 수 있는 방법을 마련해야 합니다. 예를 들어 잘못된 토큰이나 만료된 세션에 대한 처리와 같이 사용자에게 명확한 오류 메시지를 제공하는 것이 중요합니다.

- **사용자 경험**: 사용자에게 명확하고 간편한 인증 과정을 제공해야 합니다. 복잡한 절차나 불필요한 단계는 사용자 경험을 저하시킬 수 있으므로 최대한 간단하게 구성하는 것이 좋습니다.

이처럼 OAuth를 활용하여 소셜 로그인을 구현할 때는 보안과 사용자 경험 모두를 고려한 신중한 접근이 필요합니다. 
하지만 그보다도 OAuth를 사용했을 때 도드라지는 장점들이 있으니 이토록 많은 사람들이 사용하고 있는 거겠지요.

여러분도 OAuth를 이용해 여러분들만의 서비스를 개발해보시는 건 어떨까요?
