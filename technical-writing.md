# 들어가며

이 글은 아래의 독자들을 대상으로 멱등성과 멱등키에 대해 설명하는 글이다.

- **멱등성** / **멱등키** 키워드를 접해봤지만 그 **의미를 잘 모르는** 개발자
- **언제 멱등성을 보장해야 하는지** 모호하게 느껴지는 개발자
- **API에서 어떻게 멱등성을 보장할 수 있는지** 궁금한 개발자

독자는 이 글을 통해 다음의 결과를 얻을 수 있을 것이다.

- 멱등성 / 멱등키의 **등장 배경 및 개념**을 명확히 이해한다.
- **API 멱등성 적용에 대한 기준**을 마련한다.
- **멱등한 API의 설계 방법을 이해하고 직접 적용**할 수 있다.

# 목차

1. 멱등성이란?
   1. 사전적 의미
   2. HTTP에서 말하는 멱등성
2. HTTP 메서드별 멱등성
3. 멱등한 API 설계는 왜 필요한가?
   1. 예시: 외부 API가 포함된 회원가입 로직 중복 실행
   2. 시스템 외부 환경은 완전하지 않다
4. 멱등한 API는 어떻게 만들 수 있을까?
   1. 중복 요청을 누가, 어떻게 구별할까?
   2. 중복 요청을 서버에서 어떻게 처리할까?
5. 서버 코드 예시 살펴보기
6. 모든 API가 멱등해야 할까?
7. 마무리

# 1. 멱등성이란?

혹시 외부 API _(예: 결제 API)_ 를 적용하는 과정에서 **멱등키**를 마주친 적이 있는가?
이처럼 중요한 데이터를 다루는 API에서 멱등키, 멱등성이 적용되는 사례를 종종 발견할 수 있다.

하지만 여기서 **'멱등'** 이라는 단어 자체도 생소하게 느껴질 수도 있다.  
따라서 사전적 의미를 먼저 살펴본 다음, HTTP에서 멱등성이 갖는 의미가 무엇인지 이해해 보자.

## 1.1 사전적 의미

![네이버 국어사전 - 멱등성](/assets/dict-naver.png)

[네이버 국어사전](https://ko.dict.naver.com/#/entry/koko/3b4f7ca2ddb64633a9607112fb4af0e0)에 따르면 멱등성을 **"연산을 여러 번 적용하더라도 결괏값이 달라지지 않는 성질"** 이라고 설명한다.

그렇다면 HTTP에서 말하는 멱등성은 무엇을 의미할까?

## 1.2 HTTP에서 말하는 멱등성

[MDN 문서](https://developer.mozilla.org/en-US/docs/Glossary/Idempotent)와 [IETF 문서](https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/)에 따르면 각각 다음과 같이 설명한다.

> 참조 문서는 모두 **2025.10.14** 일자 기준으로 조회한 것이다.

- MDN 문서
  ![MDN - Idempotency](/assets/mdn-idempotency.png)

- IETF 문서
  ![IETF - Idempotency Introduction](/assets/ietf-idempotency.png)

위 내용을 요약하여 **HTTP에서의 멱등성**을 다음과 같이 설명할 수 있다.

```
한번 요청을 보낸 후 똑같은 요청을 여러 차례 보내더라도, 서버 리소스 상태가 똑같이 유지되는 것
```

> 참조한 두 문서 모두 **서버 리소스 변화**에 초점을 두고 있다.  
> 이에 따라 멱등성 여부는 **"중복 요청에 따른 서버 리소스 변화 유무"** 에 의해 결정된다고 볼 수 있다.

그리고 위의 두 문서에서 한 가지 더 주목할 부분이 있는데, 바로 **HTTP 메서드에 따라 멱등성 보장 여부**를 설명한다는 점이다.

# 2. HTTP 메서드별 멱등성

**HTTP 메서드**는 '서버가 관리하는 리소스에 어떤 행동을 할 것인지'를 나타낸다.  
즉, HTTP 메서드에 따라 서버 리소스 변화를 예측할 수 있으므로 **HTTP 메서드별로도 멱등성 보장 여부를 구별**할 수 있다.

| HTTP 메서드 | 멱등성 |
| ----------- | ------ |
| GET         | ✅     |
| DELETE      | ✅     |
| PUT         | ✅     |
| PATCH       | ⚠️     |
| POST        | ❌     |

앞서 멱등성의 정의를 떠올리며, 각 메서드의 멱등성 여부를 판단할 때는 **"재요청을 할 때의 서버 리소스 변화"** 를 기준으로 판단한다.

## 2.1 GET - 멱등

GET 요청은 서버의 리소스를 조회하는 안전한(safe) 메서드로, **서버의 상태를 변경하지 않는다.**  
[RFC 9110: Method Definitions](https://www.rfc-editor.org/rfc/rfc9110.html#name-methods)에 따르면 GET은 본질적으로 읽기 전용 작업이므로, 동일한 GET 요청을 여러 번 수행해도 서버의 리소스 상태는 변하지 않는다. 따라서 GET은 멱등성을 만족한다.

예를 들어, `GET /users/123`을 1번 호출하든 100번 호출하든 사용자 정보를 조회할 뿐, **서버의 상태에는 어떠한 변화도 일으키지 않는다.**

## 2.2 DELETE - 멱등

DELETE 요청은 리소스 삭제를 수행하므로, 얼핏 보면 비멱등처럼 보일 수 있다.  
하지만 앞선 멱등성의 정의를 다시 떠올려보자. **멱등성 여부를 판단할 때는 "재요청" 상황에 기반**한다.

특정 리소스에 대한 DELETE 요청을 서버가 성공적으로 처리했다면, 이후 똑같은 요청이 반복해서 들어오더라도 해당 리소스는 삭제된 상태로 남아있다.

- _예: `DELETE /users/123`을 여러 번 호출해도 사용자 123은 삭제된 상태를 유지한다._

> Q. 요청마다 응답이 달라져도 멱등하다고 할 수 있는가?
>
> A. [RFC 9110: Idempotent Methods](https://www.rfc-editor.org/rfc/rfc9110.html#name-idempotent-methods)에서는 DELETE의 멱등성을 명확히 정의하고 있다. 특히 **"응답이 다를 수 있다"** _(... though the response might differ.)_ 라고 언급한 부분을 주목하자.
>
> 예를 들어 첫 번째 DELETE 요청은 `200` / `202` / `204` 응답을 반환한 후, 이후의 DELETE 요청들은 `404`를 반환할 수도 있다.  
> 즉, 응답 코드는 다를 수 있으나 **서버의 상태**는 "리소스가 삭제됨"으로 **동일하게 유지되므로** 멱등하다고 할 수 있다.

## 2.3 PUT - 멱등

PUT 요청은 리소스 전체를 교체(replace)하는 메서드로, [RFC 9110: Idempotent Methods](https://www.rfc-editor.org/rfc/rfc9110.html#name-idempotent-methods)에서도 멱등한 메서드로 정의되어 있다. 핵심은 **요청 본문에 담긴 데이터로 대상 리소스의 상태를 완전히 대체한다**는 점이다.

동일한 PUT 요청을 여러 번 수행하면, 첫 번째 요청 이후의 모든 요청은 서버의 리소스를 동일한 상태로 유지한다. 예를 들어, 다음과 같은 요청을 생각해보자.

```http
PUT /users/123
Content-Type: application/json

{
  "name": "김철수",
  "email": "kim@example.com",
  "age": 30
}
```

이 요청을 1번 보내든 10번 보내든, 서버의 `/users/123` 리소스는 최종적으로 동일한 상태가 된다. 따라서 PUT은 멱등성을 만족한다.

## 2.4 PATCH - 비멱등

PATCH 요청은 [RFC 5789](https://www.rfc-editor.org/rfc/rfc5789.html)에 정의된 메서드로, **리소스의 부분적인 수정**을 수행한다.  
[RFC 9110](https://www.rfc-editor.org/rfc/rfc9110.html#name-idempotent-methods)에서 PATCH를 명시적으로 비멱등 메서드로 분류하지는 않지만, 구현 방식에 따라 멱등성이 달라질 수 있다.

PATCH가 일반적으로 비멱등인 이유는 **요청 본문에 "변경 방법"을 담기 때문**이다.

예를 들어:

```http
PATCH /users/123
Content-Type: application/json

{
  "age": 31
}
```

위 요청이 **"age를 31로 설정"** 을 의미한다면 멱등이다.

하지만 다음과 같은 경우는 비멱등이다:

```http
PATCH /users/123
Content-Type: application/json

{
  "increment": "age"
}
```

위 요청이 **"age 값 1 증가"** 를 의미한다면, 요청할 때마다 age 값이 계속 증가하므로 비멱등이다.

PATCH는 주로 부분 업데이트에 사용되며, 구현에 따라 멱등/비멱등이 결정된다. 그러나 일반적으로는 비멱등 메서드로 간주된다. ([MDN 문서](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Methods#safe_idempotent_and_cacheable_request_methods))

## 2.5 POST - 비멱등

POST 요청은 [RFC 9110](https://www.rfc-editor.org/rfc/rfc9110.html#name-idempotent-methods)에서 명시적으로 비멱등 메서드로 정의한다.

> POST는 주로 새로운 리소스 생성 또는 서버의 상태 변경 작업에 사용된다.

POST가 비멱등인 이유는 **동일한 요청을 여러 번 보낼 때마다 서버에 새로운 변화가 발생**하기 때문이다.

예를 들어:

```http
POST /users
Content-Type: application/json

{
"name": "이영희",
"email": "lee@example.com"
}
```

이 요청을 3번 보내면, 동일한 내용의 사용자가 3개 생성될 수 있다(각각 다른 ID를 가진).  
즉, 서버의 상태가 매 요청마다 변경되므로 POST는 비멱등이다.

다른 예시:

- **결제 처리** - 동일한 결제 요청을 여러 번 보내면 중복 결제가 발생
- **게시글 작성** - 같은 내용을 여러 번 POST하면 중복 게시글 생성
- **좋아요 카운트 증가** - 요청할 때마다 카운트가 증가

따라서 기본적으로 POST는 멱등성을 보장하지 않으며, **중복 요청 방지가 필요한 경우 시스템 내부적으로 별도의 메커니즘(예: 멱등성 키)을 구현해야 한다.**

# 3. 멱등한 API 설계는 왜 필요한가?

> = 재요청을 보내도 안전한 API 설계는 왜 필요한가?

**똑같은 요청을 여러 번 보내도 서버 리소스 변화가 없는 API**가 왜 필요한지 아직 이해가 잘 안 될 수 있다.

지금부터는 필자가 실제 겪은 사례를 기반으로, 멱등한 API 설계의 필요성을 설명하려 한다.

## 3.1 예시: 외부 API가 포함된 회원가입 로직 중복 실행

소셜 로그인을 활용한 회원가입 처리 과정에서 멱등성 부재로 인한 문제가 발생했다. 다음 시나리오를 살펴보자.

### 배경

서버는 다음과 같은 흐름으로 요청을 처리한다:

1. 클라이언트로부터 인증 코드를 받는다
2. 소셜 API를 호출하여 사용자 정보를 조회한다
3. 조회된 정보로 데이터베이스에 사용자를 생성한다
4. 클라이언트에 응답을 반환한다

### 문제 상황 시나리오

![signup1](/assets/signup1.png)

사용자가 소셜 로그인을 통해 회원가입을 시도한다.

![signup2](/assets/signup2.png)

서버 내부에서는 첫 번째 요청이 성공적으로 처리되어 데이터베이스에 사용자의 정보가 생성되었다.

![signup3](/assets/signup3.png)

그런데 첫 요청의 응답이 지연되어, 클라이언트 측에서 타임아웃이 발생했다.

![signup4](/assets/signup4.png)

이때, 클라이언트는 시스템 내부 정책에 따라 동일한 회원가입 요청을 재전송한다.

![signup5](/assets/signup5.png)

여기서 문제가 발생한다.  
앞선 요청에서 생성된 리소스와, 재요청에서 생성되어야 할 리소스가 제약 조건에 의해 서로 충돌하는 것이다.

![signup6](/assets/signup6.png)

결국 서버 내부에서는 회원가입이 성공했음에도 불구하고, 요청이 실패한 것처럼 보인다.

## 3.2 시스템 외부 환경은 완전하지 않다

앞선 시나리오를 통해 알 수 있는 사실은, **제어할 수 없는 외부 요인으로 인해 예기치 못한 문제가 발생할 수도 있다**는 점이다.

이처럼 중복 요청 문제가 발생할 수 있는 시나리오 예시는 다음과 같다:

1. 더블 클릭
2. 불안정한 네트워크
3. 요청 처리 시간 지연

그리고 중복 요청으로 인한 리소스 변화가 어떨 때는 사용자에게 치명적인 영향을 줄 수 있다.

> _**중복 요청으로 인해 치명적인 영향을 줄 수 있는 시나리오**는 `6. 모든 API가 멱등해야 할까?` 에서 언급한다._

그러므로 멱등성을 보장하지 않는 HTTP 메서드 (예: `POST`, `PATCH`) 에 대해서는 시스템 내부에서 직접 멱등성을 보장해주어야 한다.

# 4. 멱등한 API는 어떻게 만들 수 있을까?

지금까지의 내용을 정리하면 다음과 같다:

- 외부 요인으로 인해 예기치 못한 중복 요청이 발생할 수 있다.
- 멱등을 보장하지 않는 HTTP 메서드 (예: `POST`, `PATCH`) 에 대해서는 시스템 내부에서 직접 멱등성을 보장해야 한다.

이렇게 _Why?_ 에 대한 내용을 다뤘으니, 이러한 의문이 들 수 있다.  
**"그렇다면 멱등한 API는 어떻게 만들 수 있을까?"** _(How?)_

## 4.1 중복 요청을 누가, 어떻게 구별할까?

멱등한 API를 만들기에 앞서, 가장 먼저 해결해야 할 문제는 **중복 요청을 누가 / 어떻게 구별할 것인지**에 대한 문제다.

### 4.1.1 클라이언트 vs 서버

HTTP 요청과 응답을 다루는 주체로는 **클라이언트**와 **서버**가 있다.

![client vs server](/assets/client-server.png)

그렇다면 둘 중 누가 중복 요청 여부를 판단할 수 있을까?

1. **서버**  
   먼저 서버가 구별하는 경우를 생각해보자.  
   [HTTP는 무상태(MDN)](https://developer.mozilla.org/en-US/docs/Web/HTTP)이므로, 현재로서는 **서버가 서로 다른 두 요청이 중복 요청인지 구별할 방법이 없다.** 👉🏻 ❌

2. **클라이언트**  
   다시 생각해보면, **현재 요청을 어떤 상황에서 보내는지**, 그 상황을 정확히 판단할 수 있는 주체는 클라이언트 뿐임을 알 수 있다. 👉🏻 ✅

### 4.1.2 멱등키: 클라이언트가 중복 요청을 구별할 책임

![idempotency-key1](/assets/idempotency-key1.png)

이처럼 **중복 요청을 구별할 수 있는 정보**는 클라이언트가 표시하게 된다.

![idempotency-key2](/assets/idempotency-key2.png)

그리고 이렇게 **중복 요청을 구별할 수 있는 정보**를 `멱등키` 라고 부른다.

> [IETF 문서](https://datatracker.ietf.org/doc/html/draft-ietf-httpapi-idempotency-key-header-06#section-2.2)에 따르면, 멱등키는 각 요청마다 고유해야 한다고(MUST) 설명한다.

![duplicate request](/assets/duplicate-request.png)
![different request](/assets/different-request.png)

이제 클라이언트는 각 회원가입 요청마다 `멱등키`를 첨부한다.

## 4.2 중복 요청을 서버에서 어떻게 처리할까?

**`멱등키`가 첨부된 요청**을 서버가 어떻게 멱등하게 처리할 수 있을까?

### 4.2.1 멱등키 저장소

요청마다 멱등키를 확인하고, 이전 요청의 멱등키와 비교하기 위해 서버는 멱등키 저장소를 추가해야 한다.

다음은 멱등키 저장소 스키마를 간단하게 정의 해본 것이다:

```java
/* 요청 처리 상태 */
public enum ProgressStatus {
    COMPLETED,
    PROCESSING,
    ;
}

/* 멱등키 저장 스키마 */
@Getter
@Entity
public class IdempotencyRecord {

    @Id
    private String key;
    private ProgressStatus progressStatus;
    private HttpStatus responseStatus;
    private String responseBody;
}
```

이해하기 쉽게 표로 표현하면 다음과 같다:

| 멱등키  | 요청 처리 상태 | 응답 상태 | 응답 본문                                      |
| ------- | -------------- | --------- | ---------------------------------------------- |
| example | COMPLETED      | 200       | { nickname: "밍곰", email: "hello@gmail.com" } |

### 4.2.2 회원가입 중복 요청 시나리오

이제 회원가입 API가 멱등함을 가정하고, 앞선 시나리오를 다시 살펴본다.

![idempotent-signup1](/assets/idempotent-signup1.png)

클라이언트는 멱등키를 첨부하여 서버에 요청을 보낸다.

그리고 서버는 **처음 들어온 멱등키에 대한 요청**을 `PROCESSING`(진행중) 상태로 표시하여 멱등키 저장소에 저장한다.

1. **서버가 요청을 처리하는 도중에 중복 요청이 발생한 경우**

   ![idempotent-signup2](/assets/idempotent-signup2.png)

   서버는 멱등키를 기반으로 요청 처리 정보를 조회하고, 현재 요청이 이전에 이미 들어온 요청임을 인지할 수 있다. 그리고 **서버는 재요청에 대한 로직을 처리하지 않고**, `요청이 아직 처리중임`을 알리는 응답을 보낸다.

   ![idempotent-signup3](/assets/idempotent-signup3.png)

   앞선 요청에 대한 처리가 모두 완료되면, 해당 요청에 대한 진행 상태를 `COMPLETED`(완료) 상태로 표시한다. 그리고 해당 요청에 대한 응답 정보를 기록할 수 있다.

2. **서버가 요청을 처리한 이후 중복 요청이 발생한 경우**

   ![idempotent-signup4](/assets/idempotent-signup4.png)

   마찬가지로 서버는 현재 요청이 이미 처리 완료된 요청임을 인지할 수 있다. 따라서 **재요청에 대한 로직을 처리하지 않고**, 저장된 응답을 보낸다.

위 시나리오를 통해 알 수 있는 멱등한 API의 핵심 특징은 다음과 같다:

```
요청별 응답을 서버에 기록하여 중복 처리를 막을 수 있다.
```

이로써 리소스의 중복 생성을 막을 수 있는 것이다.

# 5. 서버 코드 예시 살펴보기

서버 구현 코드도 간단하게 살펴본다.

## HandlerInterceptor

클라이언트의 요청에 첨부된 멱등키를 조회해야 하므로, Spring 프레임워크의 `HandlerInterceptor` 구현체를 추가했다.

```java
@RequiredArgsConstructor
@Component
public class IdempotencyInterceptor implements HandlerInterceptor {

    private final IdempotencyRepository repository;

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws IOException {
        String key = request.getHeader("Idempotency-Key");

        if (key == null) {
            return true;
        }

        Optional<IdempotencyRecord> existing = repository.findByKey(key);

        if (existing.isEmpty()) {
            /* 첫 요청 - PROCESSING 상태로 저장 */
            IdempotencyRecord newRecord = new IdempotencyRecord(key, ProgressStatus.PROCESSING);
            repository.save(newRecord);

            return true;
        }

        IdempotencyRecord record = existing.get();

        switch (record.getProgressStatus()) {
            /* 이미 처리 완료된 요청인 경우 */
            case COMPLETED -> {
                response.setStatus(record.getStatusCode().value());
                response.getWriter().write(record.getResponseBody() != null ? record.getResponseBody() : "");
                return false;
            }
            /* 처리중인 요청인 경우 */
            case PROCESSING -> {
                response.setStatus(HttpServletResponse.SC_CONFLICT);
                return false;
            }
        }

        return false;
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        String key = request.getHeader("Idempotency-Key");

        if (key == null) {
            return;
        }

        IdempotencyRecord record = repository.findByKey(key).orElse(null);
        if (record == null) {
            return;
        }

        if (record.getProgressStatus() == ProgressStatus.PROCESSING) {
            byte[] body = new byte[0];
            if (response instanceof ContentCachingResponseWrapper wrapper) {
                body = wrapper.getContentAsByteArray();
            }
            String bodyToSave = new String(body, StandardCharsets.UTF_8);

            /* HttpServletResponse body 추출 불가능 - 응답 본문 캡처 */
            record.setResponseBody(bodyToSave);
            record.setProgressStatus(ProgressStatus.COMPLETED);
            record.setStatusCode(HttpStatus.valueOf(response.getStatus()));
            repository.save(record);
        }
    }
}
```

### 1. preHandle: 요청 멱등키 확인

핸들러에서 요청을 처리하기 전, 요청 헤더에 있는 멱등키를 확인하기 위해 `preHandle` 메서드를 구현했다.

### 2. afterCompletion: 요청 처리 결과 저장

핸들러에서 요청을 처리한 후, 요청 처리 결과를 기록하기 위해 `afterCompletion` 메서드를 구현했다.

### 2-1. 응답 본문 추출을 위한 추가 설정

`afterCompletion` 메서드 내부에서 `HttpServletResponse` 인스턴스의 본문을 추출하기 위해 다음과 같이 Filter 구현체도 추가한다.

```java
@Component
public class ContentCachingResponseFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
            throws IOException, ServletException
    {
        ContentCachingResponseWrapper wrapper = new ContentCachingResponseWrapper(res);
        try {
            chain.doFilter(req, wrapper);
        } finally {
            /* 캐싱한 응답을 클라이언트에게 flush */
            wrapper.copyBodyToResponse();
        }
    }
}
```

이처럼 시스템 내부에서 멱등성을 보장하기 위해서는 추가적으로 고려할 사항들이 늘어난다.

> 특히 멱등키를 위한 저장소가 추가됨에 따라 **In-memory 기반 저장소** 도입을 고려할 수도 있다.  
> 이 과정에서 동시성 문제, SPOF 문제 등과 같이 고려해야할 사항이 추가된다.

그리고 멱등한 API가 늘어나면 그만큼의 서버 리소스를 더 차지하게 된다.

# 6. 모든 API가 멱등해야 할까?

그렇다면 모든 API에 멱등성을 보장해야 할까? 우선 몇 가지 예시를 떠올려보자.

### 예시1: 블로그 / 커뮤니티 게시글 업로드

첫 번째 예시는 **블로그 / 커뮤니티 게시글을 업로드하는 경우**다.

여러분도 블로그나 커뮤니티에 게시글을 업로드할 때, **동일한 포스팅이 중복되어 생성되었던 경험**이 있는가? 필자도 종종 이런 경험을 겪은 적이 있다. 이때 중복 생성된 게시글을 방치하기도 하고, 어떨 때는 직접 삭제하기도 한다.

사실 사용자로서 이 시나리오는 중복 요청에 대한 영향이 그닥 치명적이지 않게 느껴진다.

### 예시2: 결제 / 재고 정보 생성 또는 수정

그렇다면 **결제 / 재고 정보를 생성 혹은 수정하는 경우**는 어떨까?  
이 경우, 중복 요청을 모두 처리했을 때 사용자에게 미치는 영향이 크게 느껴진다.

- 결제 요청을 중복 처리하면, 사용자는 하나의 물건을 구입했음에도 불구하고 결제를 두 차례 진행할 수도 있다.
- 재고 정보 업데이트를 중복 처리하면, 재고 정보가 실제와 다르게 기록될 수도 있다.

따라서 모든 API에서 멱등을 보장한다기 보다는, **"재요청 중복 처리에 의한 결과가 치명적인 곳에서만 적용하는 것"** 이 합리적이다.

## 6.1 결과: 회원가입 API에서 멱등을 보장해야 하는가?

필자는 이러한 고민을 거쳐, 결론적으로 **회원가입 API에서는 멱등성을 보장하지 않았다.**  
회원가입 과정에서는 앞선 시나리오와 같이 `4XX` 오류가 발생하더라도, 동일한 정보로 로그인을 시도하는 것처럼 사용자의 행동을 우회할 수 있다고 생각했기 때문이다. _(마치 중복 업로드된 게시글을 사용자가 삭제하는 것처럼 말이다.)_  
즉, 회원가입 API에서는 중복 요청에 의한 리소스 변화가 치명적이지 않다는 판단 하에 이와 같은 결정을 내렸다.

이처럼 특정 API에 멱등성을 적용하기에 앞서, **중복 요청에 의한 리소스 변화**가 시스템 및 사용자에게 어떤 영향을 주는지 떠올리고 적용 여부를 판단하는 것이 필요하다.

# 7. 마무리

중복 요청 처리에 의한 결과가 치명적인 경우, 시스템을 보호할 수 있는 API를 설계하자. 그리고 이 과정에서 멱등키 도입을 고려할 수 있다.
