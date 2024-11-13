
# 패스키(Passkey) 구현하기

## 패스키란 무엇인가

패스키(Passkey)는 사이트에 로그인 할 때 패스워드 없이 가지고 있는 기기(핸드폰, 노트북 등)를 이용하여 인증하는 방식이다.

구글에 로그인 할 때 아래처럼 기기에 등록된 계정에 로그인하기 위해 지문을 요청하는 작업이 나타나는데 이를 구현하는 것이 패스키의 역할이다.

<div align="center">
  <img src="images/passkey-1.png" alt="votmzl" width="25%"> 
  <p style="text-align: center;">그림 1 - 구글 로그인 시 나타나는 패스키</p> 
</div>

패스키는 웹사이트의 패스워드를 외우지 않아도 되어 편하고 암호가 기기 안에 안전하게 관리되고 있는 장점이 있다.

패스키가 나오기 전까진 회원 인증을 위해 아이디와 패스워드를 기억하며 로그인해왔다.

이로 인해 사이트가 달라도 같은 아이디와 비밀번호를 사용하는 경우가 많았었는데, 어느 한 사이트에서 회원 아이디와 비밀번호가 유출되는 상황이 발생하면 다른 사이트도 똑같이 피해를 볼 수 있었다.

FIDO alliance 따르면 패스워드로 인한 피해는 아래와 같이 나타났다고 한다. 이처럼 패스워드로 인한 피해가 증가하는 상황에서 패스워드 사용을 감소시키기 위해 개발된 것이 패스키이다.

<div align="center">
  <img src="images/passkey-2.png" alt="votmzl" width="100%"> 
  <p style="text-align: center;">그림 2 - 패스워드 문제로 인한 피해 증가량 <a href="https://fidoalliance.org">(출처 : FIDO)</a></p>
</div>

## 패스키의 사용 단계

패스키는 아래의 두 단계를 거쳐 사용할 수 있다.

```
1. 패스키 등록 과정

2. 패스키 검증 과정
```

패스키를 사용하기 위해선 해당 사이트에 패스키 기기를 등록해야 한다. 이후 회원이 로그인 할 때 기기에 등록된 패스키를 이용하여 인증할 수 있다.

### 패스키 등록

패스키를 등록하려먼 아래의 순서대로 진행된다.

<div align="center">
  <img src="images/passkey-3.png" alt="votmzl" width="100%"> 
  <p style="text-align: center;">그림 3 - 패스키 등록 과정</p>
</div>

`userHandler`: 등록된 패스키를 불러오기 위해 필요한 키 **패스키는 여러 기기를 등록 가능하다**

`challenge`: 패스키 등록 정보가 중간에 변조되었는지 검증하기 위한 난수. 클라이언트는 challenge를 기기안에 있는 개인키로 암호화 한 후 서버에 전달한다. 이후 서버는 클라이언트가 전달한 공개키로 복호화 하여 패스키 등록 정보가 변조되었는지 확인할 수 있다.

`비대칭키`: **암호화하는 키와 복호화 하는키가 다른 키 쌍**을 말한다. 개인키로 암호화 한 정보는 공개키로만 복호화가 가능하고 공개키로 암호화 한 정보는 개인키로만 복호화가 가능한 특징이 존재한다. SSL의 암호화 키 교환이나 전자서명에 사용된다.

### 패스키 검증

패스키를 검증하려면 아래의 순서대로 진행된다.

<div align="center">
  <img src="images/passkey-4.png" alt="votmzl" width="100%"> 
  <p style="text-align: center;">그림 4 - 패스키 검증 과정</p>
</div>

`인증 횟수 정보`: 패스키는 같은 기기 정보를 복사하여 기기 인증을 악용하는 것을 방지하기 위해 인증 횟수를 갱신한다. 인증 횟수 정보는 클라이언트 측에서 계산한 후 서버에 전달 되는데 만약 기기를 복재한 후 인증하고자 한다면 인증 횟수가 현 기기와 차이가 날경우 오류가 발생하여 인증을 거부할 수 있다.

## 패스키 구현

패스키는 Web Authentication API(WebAuthn)을 사용하여 구현이 가능하다. WebAuthn은 FIDO2 기반 인증을 구현하기 위한 웹 표준 기술이다. 즉 패스키는 FIDO2 기반 인증 기술을 구현한 구현체라고 생각하면 된다.

언어별로 WebAuthn 라이브러리를 지원하는 목록은 [passkeys.dev에서 확인 가능하다.](https://passkeys.dev/docs/tools-libraries/libraries/) 

### Java 기반 WebAuthn

java 어플리케이션에서 지원하는 라이브러리는 [yubico의 java-webauthn-server](https://developers.yubico.com/java-webauthn-server/)이다. yubico는 W3C에서 [WebAuthn 웹 표준을 개발하기 위해](https://www.w3.org/TR/webauthn-2/)) 참여한 단체이다.

패스키를 구현하기 위해선 아래의 의존성을 추가해야 한다.

``` groovy
implementation("com.yubico:webauthn-server-core:2.5.3")
```

#### CredentialRepository 구현

추가한 후 패스키 등록 정보를 저장하기 위한 저장소인 `CredentialRepository`를 구현해야 한다.  이 레포지토리는 `RelyingParty`객체가 패스키를 검증하기 위해 사용한다. CredentialRepository에 대한 자세한 설명은 [yubico WebAuthn Github 소스코드에 있다](https://github.com/Yubico/java-webauthn-server/blob/main/webauthn-server-core/src/main/java/com/yubico/webauthn/CredentialRepository.java). 이 글에선 구현의 용이함을 위해 In-Memory방식으로 구현하는 방법을 설명한다.


``` java
public interface CredentialRepository {

  Set<PublicKeyCredentialDescriptor> getCredentialIdsForUsername(String username);
  Optional<ByteArray> getUserHandleForUsername(String username);
  Optional<String> getUsernameForUserHandle(ByteArray userHandle);
  Optional<RegisteredCredential> lookup(ByteArray credentialId, ByteArray userHandle);
  Set<RegisteredCredential> lookupAll(ByteArray credentialId);
}
```

##### 1. getUserHandleForUsername

패스키를 등록한 `username`에 대응하는 `userHandle`값을 반환해야 해야 한다. 필자는 `Map`을 이용하여 `username`과 대응되는 `userHandle`을 저장하도록 구현했다.

``` java
public class InMemoryCredentialRepository implements CredentialRepository {

    private final Map<String, ByteArray> userIdMapping = new ConcurrentHashMap<>();

    public Optional<ByteArray> getUserHandleForUsername(String username) {
        return Optional.ofNullable(userIdMapping.get(username));
    }
}
```

#####  2. getUsernameForUserHandle

`username`를 등록하지 않은 계정을 위해 `userHandle`에 대응되는 `username`을 반환한다.

이를 통해 패스키 등록하거나 인증하기 위해선 `username`혹은 `userHandle`값을 서버에 전달해야 함을 알 수 있다. 

일관된 저장공간을 갖기 위해 `userIdMapping`을 재활용했다.

``` java
public class InMemoryCredentialRepository implements CredentialRepository {

    private final Map<String, ByteArray> userIdMapping = new ConcurrentHashMap<>();

    public Optional<String> getUsernameForUserHandle(ByteArray userHandle) {
        return userIdMapping.entrySet().stream()
                .filter(entry -> entry.getValue().equals(userHandle))
                .map(Map.Entry::getKey)
                .findFirst();
    }
}
```

##### 3.  getCredentialIdsForUsername

`username`로 등록한 패스키들을 반환한다. `PublicKeyCredentialDescriptor`는 패스키 등록 시 생성한 `RegistrationResult` 객체에서 `getKeyId`메서드의 반환 값을 바탕으로 만들 수 있다.  

필자는 아래와 같이 라이브러리에서 제공하는 빌더를 이용해 인증에 필요한 값을 생성한다. credentials의 키 값으로 userHandle을 사용한 이유는 뒤의 나오는 `lookup`메서드 타입 맞추기 위해 사용하게 되었다.

``` java
public class InMemoryCredentialRepository implements CredentialRepository {

    private final Map<ByteArray, List<RegisteredCredential>> credentials = new ConcurrentHashMap<>();
    private final Map<String, ByteArray> userIdMapping = new ConcurrentHashMap<>();

    public Set<PublicKeyCredentialDescriptor> getCredentialIdsForUsername(String username) {
        ByteArray userId = userIdMapping.get(username);
        if (userId == null) {
            return Collections.emptySet();
        }
        return credentials.getOrDefault(userId, Collections.emptyList()).stream()
                .map(registeredCredential ->
                        PublicKeyCredentialDescriptor.builder()
                                .id(registeredCredential.getCredentialId())
                                .build())
                .collect(Collectors.toSet());
    }
}
```

##### 4. lookup

패스키로 인증할 때 값을 검증하는 용도로 사용한다. 등록된 패스키 값 뿐만 아니라, 패스키로 인증한 횟수를검사하여 기기 복제와 재전송 공격을 방지하는 역할을 한다. 

```java
public class InMemoryCredentialRepository implements CredentialRepository {

    public Optional<RegisteredCredential> lookup(ByteArray credentialId, ByteArray userHandle) {
        return credentials.getOrDefault(userHandle, Collections.emptyList()).stream()
                .filter(cred -> cred.getCredentialId().equals(credentialId))
                .findFirst();
    }
}   
```

##### 5. lookupAll

패스키 등록 시 중복을 방지하기 위해 사용되는 메서드이다. 

``` java
public class InMemoryCredentialRepository implements CredentialRepository {

    public Set<RegisteredCredential> lookupAll(ByteArray credentialId) {
        return credentials.values().stream()
                .flatMap(Collection::stream)
                .filter(cred -> cred.getCredentialId().equals(credentialId))
                .collect(Collectors.toSet());
    }

}
```

#### RelyParty 설정

패스키 등록 및 검증을 위해선 `RelyingParty`를 설정해야 한다. `RelyingParty`의 역할은 아래와 같다.

``` 
1. 요청값 검증을 위한 challenge 및 암호화 알고리즘 스팩 정보 생성
2. 패스키를 등록 및 검증할 사이트 정보 전달
3. 클라이언트가 전달한 요청값들을 검증
```

클라이언트에게 보낼 임의의 challenge값을 생성하거나 암호화 알고리즘 스팩을 전달하고 클라이언트가 패스키 등록 및 인증을 하기 위한 요청값들을 검증하는 역할을 한다. `RelyingParty`를 설정하기 위해선 위의 `CredentialRepository`를 반드시 등록해야 한다.

``` java
    @Bean
    public RelyingParty relyingParty() {
        RelyingPartyIdentity rpIdentity = RelyingPartyIdentity.builder()
                .id("www.fromitive.site")
                .name("passkey 예제")
                .build();

        // CredentialRepository 인스턴스 생성
        return RelyingParty.builder()
                .identity(rpIdentity)
                .credentialRepository(inMemoryCredentialRepository) // CredentialRepository 등록
                .origins(Set.of("https://www.fromitive.site"))
                .build();
    }
```

#### 패스키 등록

패스키를 등록하기 위해선 아래의 순서대로 요청해야 한다.

```
1. 등록할 계정 정보를 서버에 전달(/register/request)
  - 서버는 계정 정보를 바탕으로 RelyingParty를 이용해 challenge 요청 값 생성
  - 요청 값은 재사용을 위해 세션에 저장
2. 클라이언트는 서버가 전달한 값을 이용해 패스키 정보 생성
3. 패스키 정보를 서버에 전달(/register/finish)
  - 세션에 저장한 요청 값을 불러와 서 변조 여부 검증
  - 성공적으로 검증하면 패스키 정보를 CredentialRepository에 저장
```

처음 클라이언트가 패스키 등록 요청을 위해 `GET /register/request?userName=`을 요청한다. 이에 서버는 클라이언트의 인증 정보를 검증하기 위한 설정 정보인 `PublicKeyCredentialCreationOptions`를 생성한 후 세션에 저장하고 클라이언트에게 응답한다.

``` java
@GetMapping("/register/request")
    public ResponseEntity<PublicKeyCredentialCreationOptions> startRegistration(@RequestParam String userName,
                                                                                HttpSession httpSession) {
        PublicKeyCredentialCreationOptions options = registerationService.start(userName);
        httpSession.setAttribute("options", options);
        httpSession.setAttribute("name", userName);
        return ResponseEntity.ok(options);
    }
```

이를 받은 클라이언트는 서버가 전달한 정보를 바탕으로 Base64 및 json을 디코딩 하고 WebAuthn API를 이용해 패스키를 생성한다.

> [!WARNING]  
> javascript에서 사용하는 WebAuthn API인 `navigator`는 반드시 도메인 인증이 완료된 SSL를 적용한 HTTPS 환경에서 실행해야한다.
> HTTP로 요청을 전송할 경우 브라우저 측에서 `navigator` 객체를 불러올 수 없게되니 테스트 시 주의해야 한다.

``` javascript
    // 서버에서 받은 challenge와 user ID 값을 URL-safe base64에서 표준 base64로 변환 후 디코딩
    options.challenge = Uint8Array.from(atob(base64UrlToBase64(options.challenge)), c => c.charCodeAt(0));
    options.user.id = Uint8Array.from(atob(base64UrlToBase64(options.user.id)), c => c.charCodeAt(0));

    // Step 2: WebAuthn API를 통해 passkey 생성
    const credential = await navigator.credentials.create({
        publicKey: options
    });

    // Step 3: 생성된 자격 증명 데이터를 서버로 전송합니다.
    const attestationResponse = {
        id: credential.id,
        rawId: toBase64Url(btoa(String.fromCharCode(...new Uint8Array(credential.rawId)))),
        type: credential.type,
          response: {
            clientDataJSON: toBase64Url(btoa(String.fromCharCode(...new Uint8Array(credential.response.clientDataJSON)))),
            attestationObject: toBase64Url(btoa(String.fromCharCode(...new Uint8Array(credential.response.attestationObject))))
          },
          clientExtensionResults: {}
    };
```

이후 클라이언트 측에서 생성한 패스키 등록 정보를 정보를 검증한다. 검증이 성공하면 패스키를 저장하고 클라이언트에 성공 메시지를 전달한다.

``` java
    @PostMapping("/register/finish")
    public ResponseEntity<Void> finishRegistration(
            @RequestBody PublicKeyCredential<AuthenticatorAttestationResponse, ClientRegistrationExtensionOutputs> credential
            , HttpSession session) {
        PublicKeyCredentialCreationOptions options = (PublicKeyCredentialCreationOptions) session.getAttribute(
                "options");
        String userName = (String) session.getAttribute("name");
        registerationService.finish(options, credential, userName);
        return ResponseEntity.ok().build();
    }
```

#### 패스키 검증

패스키를 검증하기 위해선 아래의 순서대로 요청해야 한다.

```
1. 인증 요청할 계정 정보를 서버에 전달(/assert/request)
  - 서버는 인증할 계정 정보를 바탕으로 challenge를 생성
  - challenge 정보는 검증을 위해 세션에 저장 후 클라이언트에게 전달
2. 클라이언트는 전달받은 challenge 정보를 바탕으로 패스키를 인증
3. 서버는 전달 받은 패스키 인증 정보 검증
  - 세션에 저장한 challenge 정보를 불러온 후 인증 정보가 변조가 되었는지 확인
4. 인증이 성공하면 인증 횟수를 CredentialRepository에 업데이트
```

클라이언트가 특정 식별 정보에 대한 패스키 인증을 요청하면 서버는 아래와 같이 `AssertionRequest` 객체를 생성하고 세션에 저장 한 후 클라이언트에 전달한다.

``` java
 @GetMapping("/assertion/request")
    public ResponseEntity<String> startAssertion(@RequestParam String userName, HttpSession session)
            throws JsonProcessingException {
        AssertionRequest request = assertionService.start(userName);
        session.setAttribute("assertionRequestOptions", request);
        String credentialGetJson = request.toCredentialsGetJson();
        return ResponseEntity.ok(credentialGetJson);
    }
```

클라이언트는 서버가 전달한 정보를 디코딩 한 후 `navigator`를 통해 패스키 인증 정보를 생성 

``` javascript
    const loginResponse = await fetch(`/assertion/request?username=${encodeURIComponent(username)}`);
    const assertionOptions = await loginResponse.json();
    const publicKey = assertionOptions.publicKey

    publicKey.challenge = Uint8Array.from(atob(base64UrlToBase64(publicKey.challenge)), c => c.charCodeAt(0));
    publicKey.allowCredentials = publicKey.allowCredentials.map(cred => {
        return {
            ...cred,
            id: Uint8Array.from(atob(base64UrlToBase64(cred.id)), c => c.charCodeAt(0))
        };
    });

    const assertion = await navigator.credentials.get({ publicKey: publicKey });

```

이후 서버는 인증 정보를 검증한 후 성공 여부를 전달한다.

``` java
    @PostMapping("/assertion/finish")
    public ResponseEntity<String> finishAssertion(
            @RequestBody PublicKeyCredential<AuthenticatorAssertionResponse, ClientAssertionExtensionOutputs> credential,
            HttpSession session) throws AssertionFailedException {

        AssertionRequest request = (AssertionRequest) session.getAttribute("assertionRequestOptions");
        String userName = assertionService.finish(request, credential);

        return ResponseEntity.ok(userName);
    }
```

구체적인 패스키 소스코드는 [fromitive 깃허브](https://github.com/fromitive/passkey-example)에서 확인이 가능하다.

> [!WARNING]  
> 소스코드 예제를 그대로 프로젝트에 사용하면 안됩니다. 패스키를 구현하기만 했을 뿐 등록을 위한 선행 인증과정이 없기 때문에 회원 정보만 알면 본인이 가지고 있는 기기가 아니더라도 인증이 가능하기 때문에 보안에 취약합니다. 


## 구현한 패스키 실행

SSL이 적용된 서버에 접속하면 아래와 같이 등록할 수 있는 페이지가 나타난다. 임의의 값을 입력한후 `Register passkey`를 누르면 아래처럼 기기에서 제공하는 인증 방식이 나타난다. 

<div align="center">
  <img src="images/passkey-5.png" alt="votmzl" width="50%"> 
  <p style="text-align: center;">그림 5 - 패스키 등록 화면</p>
</div>

인증을 성공하면 서버에서 인증이 성공적으로 등록되었다는 응답 값을 보낸다.

<div align="center">
  <img src="images/passkey-6.png" alt="votmzl" width="50%"> 
  <p style="text-align: center;">그림 6 - 패스키 등록 성공 화면</p>
</div>

등록한 패스키는 login.html에 접속하면 아래와 같이 사용할 수 있고 사용자 정보를 입력하고 기기 인증을 진행한다.

<div align="center">
  <img src="images/passkey-7.png" alt="votmzl" width="50%"> 
  <p style="text-align: center;">그림 7 - 패스키 인증 화면</p>
</div>

인증에 성공하면 로그인이 성공했다는 메시지가 나타난다.

<div align="center">
  <img src="images/passkey-8.png" alt="votmzl" width="50%"> 
  <p style="text-align: center;">그림 8 - 패스키 인증 성공 화면</p>
</div>


## 패스키 구현 시 주의할 점

### 1. 패스키 등록 전 회원 인증 필요

위의 예제에서 봤듯이 패스키는 여러개의 기기를 등록할 수 있으므로 패스키 등록은 인가된 사용자만 사용할 수 있게 해야 한다. 즉 패스키를 사용하기 위해선 회원 본인 인증이 선행되어야 등록할 수 있다.

### 2. 패스키 인증 실패 시 다른 인증 방식과 함께 사용

갖고 있는 기기를 이용해 인증하는 것은 간편하지만 등록한 기기를 더이상 사용하지 못하거나 기기 변경으로 인해 인증에 실패할 수 있다. 패스키를 인증하기 위해선 기기가 가지고 있는 고유한 개인키가 필요하기 때문에 개인키를 획득할 수 없으면 인증하지 못한다. 따라서 패스키로 인증할 수 없을경우 다른 인증방식을 활용하여 회읜이 인증할 수 있도록 fallback기능을 제공하는 것이 필요하다.

## 참고 자료

- https://developer.okta.com/blog/2022/04/26/webauthn-java
- https://developers.yubico.com/java-webauthn-server/
- https://www.w3.org/TR/webauthn-2/
- https://passkeys.dev/docs/tools-libraries/libraries/
- https://fidoalliance.org/passkeys/
