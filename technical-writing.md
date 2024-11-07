# CORS의 개념과 크로스 오리진 간의 쿠키 전송

## 문제 상황
---

> [!CAUTION]  
> Access to fetch at ‘https://api.example.org/path’ from origin ‘http://localhost:8080’ has been blocked by `CORS policy`: No ‘Access-Control-Allow-Origin’ header is present on the requested resource. If an opaque response serves your needs, set the request’s mode to ‘no-cors’ to fetch the resource with CORS disabled.

CORS Error는 웹 개발 경험이 있는 사람이라면 누구나 한 번쯤은 겪어 봤을 오류 상황이다.  
이번 글에서는 해당 오류가 어떤 상황에서 발생하는지 살펴보고, 이를 해결하기 위한 여러 선택지를 알아보자.

## SOP와 CORS
---

### Origin
CORS는 Cross-Origin Resource Sharing의 줄임말로, 서로 다른 출처(Origin)간의 리소스 공유를 위한 규약이다.  
이를 정확하게 이해하기 위해서는 `Origin`의 의미부터 이해해야 할 필요가 있다.

### URL의 구성 요소
우리는 URL로 특정 사이트에 접속한다. URL은 하나의 문자열이지만, 여러 부분으로 구성되어 있다.

![image](https://github.com/user-attachments/assets/40c6141c-2981-43da-bda8-144bfb1cac6c)


다음 예시를 보자.
```
https://www.example.org:8080/user?name=pedro&team=momo#Detail
```
- `https://` (Protocol)
  - 리소스에 접근하는 데 사용되는 프로토콜을 명시하는 부분으로, Scheme이라고도 부른다.
- `www.example.org`: (Host)
  - Host 부분으로, 해당 사이트의 도메인 이름이나 IP 주소를 나타낸다.
- `:8080` (Port)
  - 호스트 서버의 특정 서비스에 접근하기 위한 포트 번호를 나타낸다.
- `/user` (Path)
  - 서버 내 특정 리소스의 경로를 나타낸다.
- `?name=pedro&team=momo` (Query String)
  - 웹 서버로 전달하는 매개변수로, 키-값 쌍으로 구성된다.
- `#Detail` (Fragment)
  - 웹 페이지 내의 특정 섹션을 가리키는 부분이다.

CORS 이해를 위해서는 `Protocol`, `Host`, `Port` 부분이 중요한데, 이는 Origin이 이 3가지의 조합으로 정의되어 있기 때문이다.

### Same-Origin
> cross
> 4. 동사 (서로) 교차하다[엇갈리다]  
> *- 네이버 어학사전*

앞선 예시에서 Origin을 정의하는 URL 요소들에 대해 알아 보았다.  
CORS의 `C` 가 cross인 만큼, 여러 URL 중 어떤 URL이 같은 Origin이고, 어떤 URL이 다른 Origin인지 판단할 수 있어야 한다.

다시 설명하지만, Origin은 `Protocol + Host + Port`로 구성된다. 따라서 아래의 URL들은 모두 같은 Origin이다.
```
1. https://www.example.org:8080/path?search=value
2. https://www.example.org:8080?name=pedro
3. https://www.example.org:8080
```

다음 예시는 Protocol, Host, Port 중 하나라도 일치하지 않는 요소가 있으므로 모두 다른 Origin이다.
```
1. http://www.example.org:8080
2. https://api.example.org:8080
3. https://www.example.org:3306
```

> [!TIP]  
> [RFC 6454](https://datatracker.ietf.org/doc/html/rfc6454#section-3.2.1)에서 좀 더 자세한 예시를 확인할 수 있다.

## SOP: Same Origin Policy
---
Origin의 정의에 대해 알아 봤으니 이제 Same Origin과 Cross Origin에 대해 이해할 수 있다.

SOP는 Same Origin Policy의 약어로, 같은 Origin에서만 리소스 공유를 허용하는 정책이다.  
다시 말해, 같은 Origin 서버의 리소스는 자유롭게 접근할 수 있지만, 다른 Origin서버의 리소스에 접근하는 것은 금지된다.

### SOP가 필요했던 이유
인터넷 환경에는 어디에나 위험이 도사리고 있다. SOP로 같은 Origin에서만 리소스를 공유할 수 있도록 제한하지 않는다면 악의적인 사용자가 XSS(Cross-Site Scripting)나 CSRF(Cross-Site Request Forgery) 공격으로 정상적인 사용자의 정보를 탈취할 수 있다.  
아래는 SOP가 존재하지 않을 때의 공격 시나리오를 간단하게 나타낸 것이다.

1. 해커는 자신의 페이지(H) 접속 시 특정 사이트(A)에 요청을 보내도록 피싱 사이트를 개설한다.
2. 사용자가 해당 페이지로 접속하면 자신이 모르는 사이에 사이트 A로 요청을 보내게 된다.
3. 요청에는 브라우저에 저장된 사용자의 쿠키가 포함될 수 있다. A의 서버는 해당 쿠키 정보를 이용하여 사용자를 인증/인가하고, 정상적인 응답을 브라우저로 전달한다.
4. 피싱 사이트(H)에서는 사용자의 개인정보가 포함된 서버 A의 응답을 다시 해커의 서버(H)로 전송한다.
5. 해커가 사용자 정보 탈취에 성공한다.

이와 같은 비정상적인 경우를 미연에 방지하기 위해, 웹 보안의 기본적인 방어선으로서 SOP가 등장했다.

## CORS
---
SOP는 2011년에 [RFC 6454](https://datatracker.ietf.org/doc/html/rfc6454#section-3)에서 처음 등장한 개념이다.  
당시까지만 해도 웹은 프론트엔드와 백엔드를 별도로 구성하지 않는 경우가 대다수였다. 따라서 사용자 요청에 대한 모든 처리과정이 같은 도메인 내에서 일어날 수 있었고, 브라우저에서 다른 Origin으로 요청을 보내는 경우는 드물었다.  
오히려 다른 Origin으로 요청을 보내는 행위는 위 문단에서 언급했던 CSRF나 XSS등의 악의적 요청으로 간주하는 것이 안전했기에, 클라이언트(브라우저) 차원에서 다른 Origin으로의 요청을 원칙적으로 차단했던 것이다.

하지만 시간이 흐르면서 웹 애플리케이션의 복잡성이 증가했고, 여러 도메인의 리소스들을 활용하는 경우가 많아졌다. 하나의 웹 서비스에서 다양한 외부 API를 활용하는 사례도 증가하였고, 단일 페이지 애플리케이션(SPA)과 같은 기술의 발전으로 인해 다양한 도메인의 리소스를 원활하게 통합해야 할 필요성이 생겼다.

이러한 시대 흐름을 반영하기 위해 SOP의 예외 조항으로 생겨난 것이 CORS이다.  
서버는 자신을 호출할 수 있는 Origin을 화이트리스트로 관리하고, 해당 리스트에 있는 Origin들은 '서버에서 신뢰하는 출처'로써 Origin이 다르더라도 요청이 정상적으로 수행될 수 있도록 한다.

요약하자면 'CORS 정책을 만족한다면, SOP를 위배하더라도 해당 요청은 허용하겠다.' 라는 의미가 된다. 이렇게 CORS는 SOP의 제한을 완화하면서도, '기본 방어선'이 무너지지 않도록 보안을 유지할 수 있는 구조를 제공한다.

### CORS의 유효성
CORS 허용 Origin의 리스트는 서버에서 관리하지만, 이는 서버를 보호하기 위함이 아니다. 악의적인 비 정상 클라이언트가 웹 상에 배포되어 정상적인 사용자가 해당 클라이언트로 서버에 요청을 보낼 때, 사용자의 브라우저가 해당 요청을 미리 거부하도록 하여 보안 수준을 유지한다.

> [!CAUTION]  
> 다시 강조하지만 이 '보안 수준'은 기본적인 방어선으로, 모든 상황에 대한 보안을 보장하지는 못한다.  
> 예를 들어, 아래와 같은 시나리오에서는 CORS는 물론 SOP도 악의적인 사용자를 막을 수 없다.
1. 게시판 형태의 서비스인 `example.com`에 공격자가 XSS 공격을 위해 악성 스크립트가 포함된 글을 등록한다.
2. 정상 사용자가 공격자의 글을 조회하면, 악성 스크립트는 `api.example.com`으로 사용자 정보 조회를 요청한다.
3. 해당 요청에서 `api.example.com`으로 전송된 쿠키는 정상적인 사용자를 식별할 수 있는 값이 포함되어 있으므로 사용자의 정보를 정상적으로 응답한다.
4. 악성 스크립트는 해당 응답을 받아 공격자 서버인 `attack.com`으로 전송한다.
   - 4-1) 이때, `example.com`과 `attack.com`은 다른 Origin이지만, 공격자가 `attack.com`의 서버 CORS 설정에 `example.com`을 직접 추가할 수 있으므로 요청이 허용된다.
5. 공격자가 사용자 정보 탈취에 성공한다.

## CORS의 3가지 시나리오
---

### 단순 요청 (Simple Request)
![image](https://www.baeldung.com/wp-content/uploads/sites/4/2021/01/Simple-Request.png)
- 이미지 출처: [Baeldung CS](https://www.baeldung.com/cs/cors-preflight-requests)

브라우저는 다른 Origin으로 요청을 전송할 때 `Origin` 헤더를 자동으로 추가하여 전송한다.
```
Origin: http://www.example.org
```

서버는 요청을 처리하고, 응답에 `Access-Control-Allow-Origin` 헤더를 설정하여 전송한다. 이 헤더에는 서버에서 허용된 Origin 정보가 담겨 있다.
```
Access-Control-Allow-Origin: http://www.example.org
Access-Control-Allow-Origin: *  # 와일드카드로 모든 출처를 허용할 수도 있다.
```

브라우저는 요청 헤더의 `Origin`으로 설정한 값이 응답 헤더의 `Access-Control-Allow-Origin` 에 포함되어 있다면 해당 요청을 안전하다고 평가하고, 응답을 정상 처리한다.  
만약 두 값이 다르거나 `Access-Control-Allow-Origin`이 설정되어 있지 않다면 해당 응답은 파기된다.

#### 단순 요청 만족 조건
임의의 요청이 단순 요청으로 처리되기 위해서는 다음과 같은 조건을 모두 만족해야 한다.
- `GET`, `POST`, `HEAD` 메서드만 사용할 수 있다.
- `Accept`, `Accept-Language`, `Content-Type`, `Content-Language` 헤더만 전송한다.
- `Content-Type`의 값은 아래 3가지만 허용한다.
  - `text/plain`
  - `multipart/form-data`
  - `application/x-www-form-urlencoded`

하지만 프론트엔드와 백엔드 서버가 분리되어 CORS 설정이 필요한 대부분의 경우 데이터 교환은 JSON으로 이루어진다. 이 경우 `Content-Type`은 `application/json` 이 주로 사용되며, 사용자 식별을 위해 쿠키를 전송해야 하는 경우 `Cookie` 헤더를 사용하기 때문에 단순 요청을 만족하는 요청은 그렇게 많지 않다.

### 사전 요청 (Preflight Request)
이후 다시 언급하겠지만, CORS 위반 여부를 검사하고 응답 처리 여부의 판단은 브라우저가 담당한다. 그렇기에 서버에서는 요청의 CORS 위반 여부와 무관하게 요청이 수신되면 처리 후 응답을 전송한다.  
단순 요청에서는 응답을 받은 브라우저가 `Access-Control-*` 응답 헤더와 비교 후 파기 여부를 결정한다.  
이때, `GET`이나 `HEAD`와 같은 조회성 메서드들은 큰 문제가 되지 않지만, `POST`나 `DELETE`같이 상태를 변경시키는 메서드들은 서버에 부수 효과(Side Effect)를 야기할 수 있다. 예를 들어, 사용자의 특정 정보를 삭제하는 `DELETE` 요청을 다른 Origin에서 전송하면 브라우저는 CORS 정책에 따라 서버의 응답을 파기하지만, 서버에서는 정보가 이미 삭제된 상태로 남아 있을 것이다.

![image](https://www.baeldung.com/wp-content/uploads/sites/4/2021/01/Screenshot-2021-01-13-at-23.13.47.png)
- 이미지 출처: [Baeldung CS](https://www.baeldung.com/cs/cors-preflight-requests)

이러한 부수 효과를 방지하기 위해 사전 요청(Preflight Request)가 등장하였다.  
사전 요청은 말 그대로 실제 요청을 보내기 전에 서버 측이 해당 요청의 메서드와 헤더에 대해 인식하고 있는지를 확인하기 위해 보내는 요청이다.  
`Access-Control-Request-Method`와 `Origin` 헤더를 사용하여 `HTTP OPTION` 메서드를 사용하여 서버로 요청을 전송한다. 선택적으로 `Access-Control-Request-Headers` 헤더를 추가로 사용할 수도 있다.

간단히 말해 실제 요청하고자 하는 메서드와 헤더 정보를 서버에게 미리 제공하는 것이다. 서버의 CORS 정책 위반 여부를 미리 검증하고, CORS 위반 요청의 경우 본 요청 자체를 보내지 않음으로써 서버를 보호하는 역할을 한다.

#### 사전 요청의 오버헤드
사전 요청은 본 요청을 전송하기 전 미리 요청을 보내 CORS 위배 여부를 확인하는 요청이라고 했다. 이는 결국 하나의 요청을 수행하기 위해 매 번 두 개의 요청을 전송해야 한다는 뜻이고, 애플리케이션 성능에 나쁜 영향을 미친다.

이러한 문제를 해결하기 위해 서버는 `Access-Control-Max-Age` 응답 헤더로 브라우저에서 사전 요청의 결과를 캐싱하는 기간을 초 단위로 설정할 수 있다.  
브라우저는 사전 요청을 진행할 때 마다 캐시를 확인하여 캐시에 유효한 값이 존재하는지 확인한다. 캐싱되어 있지 않은 경우에만 실제 사전 요청을 전송하면 오버헤드를 최소화할 수 있다.

### 자격 증명 포함 요청 (Credential Request)
브라우저 쿠키와 같이 사용자를 식별할 수 있는 정보가 포함된 요청에 대해서는 좀 더 엄격한 정책이 적용된다. 클라이언트에서 이러한 요청을 전송하기 위해서는 `credentials` 옵션을 명시해 주어야 한다.

자바스크립트의 fetch API는 다음 3가지의 옵션을 제공한다.
- `omit`: 모든 요청에 자격 증명을 포함하지 않음
- `same-origin`: 동일 출처간의 요청에만 자격 증명을 포함하여 전송
- `include`: 모든 요청에 자격 증명을 포함하여 전송

```javascript
fetch("http://example.org"), {
	method: 'POST',
	credentials: 'include',  // 이 부분
});
```

Axios나 XMLHttpRequest는 `withCredentials` 옵션을 `true`로 설정하여 요청 전송 시 자격 증명을 포함할 수 있다.
```javascript
axios.post("https://example.org", {
	id: 'pedro', password: 'pass'
}, { 
	withCredentials: true  // 이 부분
});
```

```javascript
const xhr = new XMLHttpRequest();
xhr.open("POST", "http://example.com/", true);
xhr.withCredentials = true;  // 이 부분
xhr.send(null);
```


> [!IMPORTANT]  
> 서버에서는 클라이언트의 자격 증명 포함 요청에 대응하기 위해 `Access-Control-Allow-Credentials` 응답 헤더를 `true`로 설정하여 응답해야 한다.  
> 이 경우 다음과 같은 응답 헤더의 값은 와일드카드(`*`)로 설정할 수 없다.
> - `Access-Control-Allow-Origin`
> - `Access-Control-Allow-Methods`
> - `Access-Control-Allow-Headers`

> [!WARNING]  
> 로그인 API 등 요청에는 사용자 정보가 필요하지 않지만, `Set-Cookie`와 같이 응답헤더로 쿠키를 설정해야 하는 경우에도 `credentials` 옵션을 명시해 주어야 한다.  
> `withCrednetials`가 `true`로 설정되지 않은 `XMLHttpRequest` 는 쿠키를 설정할 수 없으며, 서버의 응답에 `Set-Cookie` 헤더가 존재하더라도 브라우저는 이를 무시한다.

### CORS는 브라우저 스펙이다
![image](https://github.com/user-attachments/assets/9969111d-d920-4b5c-bbd3-17a9240ab574)

위 시나리오에서 이미 눈치 챈 독자도 있겠지만, CORS 위배 여부의 판단은 **서버에서 제공한 정보를 기반으로 브라우저가 수행**한다. 서버는 단순히 허용 정책을 설정하는 역할을 할 뿐이며, 서버로부터 정상적인 응답이 수신되더라도 브라우저가 Origin이 다르다고 판단하면 해당 응답을 파기한다.

> [!NOTE]  
> 실제로 크로미움 기반 브라우저는 실행 시 `flag`를 추가하여 CORS 검사를 비활성화 할 수 있는 기능을 제공하고 있으며, 상황에 따라 임시로 다른 Origin간의 통신이 필요할 때 유용하게 사용할 수 있다.
>
> CORS는 브라우저 스펙이므로 서버-서버 간 통신에는 적용되지 않는다. 따라서 프록시 서버를 거쳐 통신하는 방법으로 CORS에러를 해결할 수도 있다. 이에 대한 자세한 설명은 잠시 후 알아보도록 하자.

## 개발환경의 CORS Error 해결
---

### 서버 측에 Allowed Origin 설정하기
CORS Error의 가장 정석적으로 해결하는 방법은 CORS 정책의 도입 의도에 맞게, 요청을 보낼 Origin을 서버의 화이트리스트로 등록해 두고 서버 응답 시 `Access-Control-*` 헤더를 포함하여 전송하는 것이다.

SpringBoot의 경우 다음과 같이 `WebMvcConfigurer`의 `addCorsMappings()`메소드를 오버라이드하여 전역적으로 설정할 수 있다.
```java
@Configuration  
public class CorsConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")  // Path 패턴
                .allowedOriginPatterns("http://example.org:8080", ...)  // 허용할 Origin
                .allowedMethods("GET", "POST", ...)  // 허용할 Method
                .allowCredentials(true)  // 자격 증명 포함 요청 허용
                .maxAge(6000);  // Preflight 요청 캐싱
    }  
}
```

또는 다음과 같이 특정 컨트롤러, 특정 메서드에만 CORS 설정을 적용할수도 있다.
```java
@Controller
@CrossOrigin(origins = "http://example.org:8080", methods = RequestMethod.GET) 
public class UserController {
    @CrossOrigin(origins = "http://example.org:8080", methods = RequestMethod.GET) 
    @GetMapping("/user/{userId}")  
    public ResponseEntity<User> find(@PathVariable long userId){
        return null;
    }
}
```

> [!WARNING]  
> 와일드카드 `*` 으로 헤더 값을 설정하는 것은 SOP를 완전히 비활성화하는 설정과 동일하다. 앞서 언급했던 것과 같이 최소한의 보안을 위해 존재하는 정책이므로 직접 허용할 Origin을 명시하여 사용하도록 하자.

### 로컬 프록시 서버 활용
CORS Error를 가장 많이 접할 수 있는 상황은 프론트엔드 로컬 개발환경에서 원격에 배포된 백엔드 서버에 요청을 전송하는 경우이다.

CORS는 브라우저에서 관리하는 정책이므로, 서버와 서버간의 통신에는 적용되지 않는다.  
이 점을 활용하여 로컬 환경에 프록시 서버를 구축하고, 클라이언트는 로컬 프록시로 요청을 보낸 다음 프록시에서 해당 요청을 실제 서버로 포워딩해주는 작업으로 CORS 정책을 **우회**할 수 있다.

하나의 예시로, WebPack의 devServer는 [CORS 정책 우회를 위한 프록시 기능](https://webpack.kr/configuration/dev-server/#devserverproxy)을 제공하고 있다.
```javascript
module.exports = {
  devServer: {
    proxy: [
      {
        context: ['/api/**'],
        target: 'http://api.example.org',  // 프록시에서 포워딩할 API 서버
      },
    ],
  },
}
```

위와 같이 설정할 경우 브라우저가 `http://localhost:3000/api/**` 으로 요청을 전송하면 로컬 프록시가 이를 수신하여 `http://api.example.org/api/**` 으로 대신 요청을 전달한다.  
이때 프록시와 API 서버는 다른 Origin 이지만 서버-서버 통신이기 때문에 CORS의 영향 없이 정상적으로 통신할 수 있다. API서버의 응답을 수신한 프록시 서버가 브라우저에게 응답을 전달하면 브라우저는 동일한 Origin(`http://localhost:3000`)으로 인식하므로 API 서버와 자유로운 통신이 가능하다.

### Chromium 기반 브라우저에서 CORS 검사 비활성화
Chromium 기반 브라우저에서는 크롬 `--disable-web-security` 명령행 인수를 전달하여 브라우저를 실행하면 CORS 검사를 비활성화할 수 있다.

macOS 기준으로 다음과 같은 커맨드로 사용한다.
```zsh
open -n -a /Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome --args --disable-web-security
```

![image](https://github.com/user-attachments/assets/c656e511-bc8f-4c0b-b39b-377fc4c51128)

> [!WARNING]  
> 실행 시 표시되는 경고 문구에서도 알 수 있듯, 보안 옵션을 강제로 비활성화하는 설정이므로 임시 테스트 용도로만 사용해야 한다.

## 새로운 문제 상황 - 쿠키 전송 실패
---
위의 CORS 관련 설정을 모두 해 주었음에도 웹 클라이언트(로컬)와 서버 간에 쿠키가 전송되는 경우 개발자의 의도와 다르게 동작하는 경우가 있다.  
이는 쿠키의 속성 중 `SameSite` 설정과 관련이 있는데, 조건을 만족하지 않는 경우 별다른 메시지 없이 클라이언트의 쿠키가 전송되지 않지 때문에 문제 원인 파악에 어려움을 겪게 된다.

## 쿠키의 SameSite 속성
---

### SameSite 속성 ([RFC6265bis](https://datatracker.ietf.org/doc/html/draft-ietf-httpbis-rfc6265bis#name-the-samesite-attribute-2))
> Controls whether or not a cookie is sent with cross-site requests, providing some protection against cross-site request forgery attacks ([CSRF](https://developer.mozilla.org/en-US/docs/Glossary/CSRF)).

[MDN 문서](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie#samesitesamesite-value)에 따르면 쿠키의 `SameSite` 속성은 cross-site 요청 간에 쿠키의 전송 여부를 제어한다.  
지금까지는 same-origin과 cross-origin만 구분했는데, `site`라는 용어가 등장했다. 어떠한 차이가 있는지 알아보자.

### SameSite의 판단 기준
앞서 언급했던 URL 예시를 다시 살펴보자.
```
https://api.example.org:8080/search?query=value&page=3#Title
```
- `https://` (Protocol)
  - 리소스에 접근하는 데 사용되는 프로토콜을 명시하는 부분으로, Scheme이라고도 부른다.
- `api.example.org`: (Host)
  - Host 부분으로, 해당 사이트의 도메인 이름이나 IP 주소를 나타낸다.
- `:8080` (Port)
  - 호스트 서버의 특정 서비스에 접근하기 위한 포트 번호를 나타낸다.

Origin이 `Protocol + Host + Port`로 구성되는 반면, Site는 `Host` 부분과만 관계가 있다.  
정확히는 `eTLD+1` 값으로 정의된다.

#### 도메인 구성 요소
TLD(최상위 도메인)란 도메인 이름에서 마지막 점 다음에 오는 부분이다.  
`eTLD`는 '유효 최상위 도메인'을 의미하며, 단일 조직이 이름을 등록할 수 있는 모든 도메인 접미사를 의미한다.

> [!TIP]  
> 도메인은 도메인 이름 등록 기관에 의해 관리된다. 많은 등록 기관이 최상위 수준(TLD)보다 낮은 수준에서도 도메인을 등록하도록 허용하며, 이를 차상위 도메인(SLD)라고도 한다.  
> 차상위 도메인은 항상 존재하는 것이 아니기 때문에, 어느 부분까지가 도메인 이름 접미사(`ac.kr` 등)이고, 어디까지가 도메인 이름인지 일반적인 규칙으로 확인할 수 없다.  
> [이곳](https://publicsuffix.org/list/)에서 공용 도메인 이름 접미사 목록을 확인할 수 있다.

`eTLD+N`은 eTLD로부터 N단계 앞의 부분까지 포함한 값이다.  
다음과 같은 도메인이 있다고 하자.
```
api.example.org
```
위 예시에서 TLD는 `org` 이며, `eTLD+1`은 `example.org` 이다.

따라서 아래 목록은 모두 same-site이다.
```
http://api.example.ac.kr:8080
http://www.example.ac.kr:3000
https://example.ac.kr
```

반대로, 아래 목록은 모두 cross-site이다.
```
http://www.example.org
http://www.example.com
http://www.example.ac.kr
http://wwww.google.com
```

### SameSite 속성의 값
쿠키의 SameSite 옵션에는 다음과 같은 세 가지의 값을 지정할 수 있다.
- `Strict`: 가장 엄격한 정책으로, 쿠키가 설정된 사이트와 같은 사이트(same-site)의 요청에만 쿠키를 전송한다.
- `Lax`: Strict과 같이 같은 사이트가 아니면 쿠키를 전송하지 않지만, 안전한 HTTP Method를 이용한 Top-Level Navigation에 한해 쿠키를 예외적으로 전송한다.
- `None`: 모든 cross-site 요청에 쿠키 전송을 허용한다. HTTPS로만 쿠키를 전송하는 `secure` 설정과 함께 사용해야 한다.

> [!NOTE]  
> 안전한 HTTP Method는 `GET`을 의미하며, [Top-Level Navigation](https://datatracker.ietf.org/doc/html/draft-ietf-httpbis-rfc6265bis#section-8.8.2)은 링크 클릭 등 주소창의 URL이 변경되는 페이지 이동을 말한다.  
> A.com에서 B.com으로 이동하는 링크를 클릭했을 때, B.com의 쿠키가 `SameSite=Strict`으로 설정된 경우에는 전송되지 않지만 `Lax` 혹은 `None`으로 설정된 경우 함께 전송된다.

> [!TIP]  
> `SameSite`의 기본 값은 `None`이였지만, 2020년 Chrome v80 이후 버전부터 `Lax`를 기본값으로 사용하고 있다.  
> 따라서 아무 설정도 명시하지 않으면 `localhost:3000` 의 프론트엔드에서 `api.server.org`의 API를 호출하면 cross-site간의 호출이므로 `credentials` 설정과 관계없이 쿠키가 전송되지 않는다.

### SameSite=None의 위험성
보안 설정을 강제로 비활성화하는 것은 항상 위험한 행동이다. SameSite 설정 때문에 쿠키가 설정되지 않는다고 해서 개발 시 `None` 값을 설정하는 경우가 있다. 이럴 때 어떤 위험이 발생할 수 있을까?

쿠기는 크게 두 종류로 구분할 수 있는데, 사용자가 직접 방문한 웹사이트에서 생성한 **퍼스트파티 쿠키**와, 사용자가 방문한 웹사이트가 아닌 다른 도메인에서 생성되는 **서드파티 쿠키**가 바로 그것이다.

서드파티 쿠키는 주로 광고 추적 등 사용자의 추적을 목적으로 활용되며, **사용자의 모든 요청에 함께 전송된다.**  
`SameSite=None` 설정은 해당 쿠키를 마치 서드파티 쿠키처럼 관리하라는 설정이다. 모든 크로스 사이트 요청에 쿠키를 전송할 수 있게 되어 CSRF 공격에 취약해지는 것은 물론, 사용자의 브라우징 활동을 추적하는 데 악용될 가능성도 있다.

## 개발환경에서 안전하게 쿠키 주고받기
---
일반적으로 프론트엔드 서버와 백엔드 서버가 분리되어 있더라도, 동일한 `eTLD+1`을 가지므로 배포된 상황에서는 `SameSite` 관련 오류를 경험하기 힘들다. 문제는 프론트엔트 로컬 개발환경에서, 이미 배포된 API 서버로 요청을 전송할 때 발생한다.

다음과 같은 상황을 가정해 보자.
```
프론트: http://localhost:3000
백엔드: http://api.test.com
```
이 둘은 명백한 cross-site기 때문에, `SameSite=None`으로 설정해 주지 않는 이상 두 사이트는 쿠키를 교환할 수 없다.

이러한 경우, 로컬 환경의 host 파일을 활용하여 임시로 프론트엔드와 백엔드를 SameSite로 만들어 주면 `SameSite=None` 설정 없이 쿠키 차단을 우회할 수 있다.

### host 파일
host 파일은 운영체제에서 관리하는 시스템 파일로, 도메인과 IP를 매핑하는 역할을 한다.  
로컬 기기에 파일 형태로 존재하며, 이곳의 설정값은 DNS 쿼리보다 우선된다.

host파일은 다음과 같은 경로에 위치한다.
```
macOS, Linux: /etc/hosts
Windows: C:\Windows\System32\drivers\etc\hosts
```

우리의 목적은 로컬 환경을 백엔드와 SameSite로 만드는 것이므로, 다음과 같이 `eTLD+1`이 같은 주소를 루프백 IP로 매핑해 준다.

```zsh
sudo nano /etc/hosts
127.0.0.1 localfront.test.com
```

이후 매핑된 주소(`localfront.test.com`) 으로 접속하여 API를 호출하면 브라우저는 현재 사이트가 API 서버와 SameSite라고 인식하므로, 쿠키를 정상적으로 교환할 수 있다.

## HTTP 요청의 Same Origin, SameSite 여부 확인
---
최근의 브라우저들은 [`Sec-Fetch-Site` 요청 헤더](https://web.dev/articles/same-site-same-origin?hl=ko)를 설정하여 요청을 전송한다. 이 헤더의 값은 다음 4가지 중 하나로 설정된다.
- `cross-site`: 다른 Origin 간의 요청
- `same-site`: 동일한 Site 간의 요청 (스키마 일치 여부 포함)
- `same-origin`: 동일한 Origin 간의 요청
- `none`: 사용자가 직접 시작한 작업(URL 직접 입력 등)

브라우저의 개발자 도구로 요청 헤더를 조회하면 현재 요청이 어떤 Origin과 Site로 평가되어 전송되는지를 확인할 수 있다.

> [!IMPORTANT]  
> 2019년 이전까지는 [SameSite](https://html.spec.whatwg.org/#same-site)에 기본적으로 스키마를 포함하지 않고 판단하였고, [schemeful-samesite](https://web.dev/articles/schemeful-samesite)라는 개념을 별도로 두어 조금 더 엄격한 보안 수준이 필요할 때 사용했다.  
> 2019년 말에 SameSite 정의가 스키마를 포함하는 것으로 변경되었으며, 반대로 [schemeless-same-site](https://web.dev/articles/same-site-same-origin?hl=ko#schemeless-same-site)를 두어 레거시 개념을 지원하도록 했다.  
> 2024년 9월 현재 `Sec-Fetch-Site` 헤더의 `same-site`는 스키마를 포함하여 계산된다.
>

[`Sec-*` 접두사를 가진 헤더들은 JavaScript로 수정할 수 없기 때문에](https://www.w3.org/TR/fetch-metadata/#sec-prefix), 서버에서는 이 헤더를 신뢰하고 사용할 수 있지만, 2020년 중순을 기준으로는 Chrome을 제외한 모든 브라우저에서 지원하지 않는 헤더였기 때문에 구형 브라우저를 지원해야 하는 서비스에서는 주의가 필요하다.

## 마무리
---
프로젝트에서 예상치 못한 시점에 마주하는 CORS 오류는 늘 귀찮은 존재였다. 독자 모두가 이 글로 CORS의 상세한 동작 과정에 대해 이해하고, 다시는 오류를 마주하지 않기를 바란다.
