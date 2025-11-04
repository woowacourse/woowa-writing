# 왜 Retrofit을 사용할까? - HttpURLConnection부터 Retrofit까지의 여정

## 글을 시작하며

안드로이드 개발을 하다 보면 네트워크 통신은 피할 수 없는 필수 요소입니다. 하지만 HttpURLConnection, OkHttp, Retrofit 등 다양한 선택지 앞에서 "왜 이 라이브러리를 써야 하는지" 명확히 답하기 어려울 때가 있습니다.

이 글은 **안드로이드에서 네트워크 통신을 처음 접하거나, 각 라이브러리의 차이점이 궁금한 개발자**를 위해 작성되었습니다. HTTP 통신의 기초부터 시작해 각 방식의 장단점을 실제 코드로 비교하고, Retrofit이 내부적으로 어떻게 동작하는지까지 살펴보겠습니다.

## 이 주제를 선택한 계기

우아한테크코스 레벨2에서 HttpClient 미션을 진행할 때였습니다. 리뷰어님께 이런 질문을 받았습니다.

> "왜 Retrofit을 쓰나요? OkHttp로도 충분하지 않나요?"  
> "왜 OkHttp를 쓰나요? HttpURLConnection으로도 할 수 있는데요?"

순간 당황했습니다. 단순히 "많이 쓰니까", "미션 조건이 써야 해서"라는 막연한 답변밖에 떠오르지 않았기 때문입니다. 그날 이후 각 라이브러리가 해결하려는 문제가 무엇인지, 어떤 점을 개선했는지 직접 확인해보고 싶었습니다.

그래서 같은 API 요청을 HttpURLConnection, OkHttp, Retrofit 세 가지 방식으로 구현해보며 차이점을 비교해보기로 했습니다.

---

## HTTP 통신의 기초

본격적인 비교에 앞서, HTTP 통신의 기본 개념을 짚고 넘어가겠습니다.

### HTTP란?

**HTTP(Hypertext Transfer Protocol)**는 클라이언트와 서버가 데이터를 주고받기 위한 규약입니다. 

- **클라이언트**: 요청을 보내는 쪽 (브라우저, 모바일 앱 등)
- **서버**: 요청을 받아 처리하고 응답을 돌려주는 쪽
- **엔드포인트**: 통신을 위한 주소(URL), 예: `https://api.example.com/posts/1`

클라이언트가 HTTP 요청을 보내면, 서버는 JSON, XML 등의 형식으로 응답 본문을 반환합니다.

---

## 1단계: HttpURLConnection - 가장 기본적인 방식

안드로이드에서 기본 제공하는 네트워크 통신 수단입니다. 다음은 간단한 GET 요청 예시입니다.
```kotlin
fun request() {
    thread {
        val url = URL("https://api.example.com/posts/1")
        val connection = url.openConnection() as HttpURLConnection
        
        val title = if (connection.responseCode == HttpURLConnection.HTTP_OK) {
            val content = connection.inputStream.bufferedReader().use { it.readText() }
            JSONObject(content).getString("title")
        } else {
            "HTTP 오류: ${connection.responseCode}"
        }
        
        runOnUiThread {
            textView.text = title
        }
    }
}
```

### 동작은 하지만, 문제가 많습니다

#### 타입 안전성 부족
- HTTP 메서드가 문자열로 지정되어 컴파일 시점에 오류를 잡을 수 없습니다
- URL이나 파라미터 오타도 런타임에야 발견됩니다

#### 스레드 관리의 어려움
- 메인 스레드에서 네트워크 요청을 할 수 없어 `thread {}`로 감싸야 합니다
- UI 업데이트를 위해 다시 `runOnUiThread {}`를 써야 합니다

#### 복잡한 예외 처리
- 네트워크 중단, 응답 실패, 파싱 오류를 모두 직접 처리해야 합니다
- 스트림과 연결을 수동으로 닫지 않으면 메모리 누수가 발생합니다

#### 중복 코드의 증가
- JSON 파싱을 매번 직접 작성해야 합니다
- 여러 API를 관리할 경우 URL, 헤더, 파라미터 설정이 반복됩니다

---

## 2단계: OkHttp - HTTP 클라이언트의 추상화

Square에서 만든 OkHttp는 HttpURLConnection의 불편함을 개선한 라이브러리입니다.
```kotlin
val okHttpClient = OkHttpClient()
val request = Request.Builder()
    .url("https://api.example.com/posts/1")
    .build()
    
okHttpClient.newCall(request).enqueue(object : Callback {
    override fun onFailure(call: Call, e: IOException) {
        runOnUiThread {
            textView.text = "요청 실패: ${e.message}"
        }
    }
    
    override fun onResponse(call: Call, response: Response) {
        if (response.isSuccessful) {
            val body = response.body?.string() ?: "응답이 null"
            val title = try {
                JSONObject(body).getString("title")
            } catch (e: Exception) {
                "파싱 오류: ${e.message}"
            }
            runOnUiThread {
                textView.text = title
            }
        }
    }
})
```

### OkHttp가 개선한 점들

#### 1. 비동기 처리 간소화
- `enqueue()` 메서드가 자동으로 백그라운드에서 실행됩니다
- 더 이상 `thread {}`를 직접 만들 필요가 없습니다

#### 2. 명시적인 요청 구성
- `Request.Builder`를 통해 URL, 헤더, 메서드를 직관적으로 설정할 수 있습니다
- 응답 본문도 `response.body?.string()`으로 간단히 읽을 수 있습니다

#### 3. 내장된 최적화 기능
- **커넥션 풀(Connection Pool)**: 연결을 재사용해 성능을 향상시킵니다
- **타임아웃 기본값**: 별도 설정 없이도 안정적인 타임아웃이 적용됩니다
- **자동 리소스 관리**: 스트림을 열고 닫는 과정을 자동으로 처리합니다

#### 4. 간편한 상태 코드 처리
- `isSuccessful`로 HTTP 200~299 응답을 쉽게 확인할 수 있습니다

### 하지만 여전히 남은 과제

OkHttp도 다음과 같은 작업은 개발자가 직접 해야 합니다:

- JSON 파싱을 매번 수동으로 처리
- 여러 API 엔드포인트 관리
- 콜백 기반 비동기 처리로 인한 가독성 저하
- 공통 헤더나 인증 로직의 중복

---

## 3단계: Retrofit - 인터페이스 기반의 선언적 HTTP 클라이언트

드디어 Retrofit입니다. Retrofit은 "어노테이션 기반 인터페이스"만으로 API를 정의할 수 있게 해줍니다.

### 인터페이스 정의
```kotlin
interface RetrofitService {
    @GET("posts/{id}")
    suspend fun getPost(@Path("id") id: Int): Post
}
```

### Retrofit 객체 생성
```kotlin
val retrofitService = Retrofit.Builder()
    .baseUrl("https://api.example.com/")
    .addConverterFactory(GsonConverterFactory.create())
    .build()
    .create(RetrofitService::class.java)
```

### 실제 사용
```kotlin
lifecycleScope.launch {
    runCatching { retrofitService.getPost(1) }
        .onSuccess { post: Post ->
            textView.text = post.title
        }
        .onFailure { e ->
            when (e) {
                is HttpException -> {
                    textView.text = "HTTP 오류: ${e.code()}"
                }
                is IOException -> {
                    textView.text = "네트워크 오류: ${e.message}"
                }
                else -> {
                    textView.text = "예외: ${e.message}"
                }
            }
        }
}
```

### Retrofit이 해결한 모든 문제들

#### 1. 극적인 코드 간소화
- 복잡한 네트워크 로직이 인터페이스 선언 몇 줄로 대체됩니다
- 마치 로컬 함수를 호출하듯 API를 사용할 수 있습니다

#### 2. 타입 안전성 보장
- JSON 응답이 자동으로 데이터 클래스로 변환됩니다
- 컴파일 시점에 타입 오류를 발견할 수 있습니다
- 키 누락이나 형 변환 오류가 크게 줄어듭니다

#### 3. 자동 JSON 파싱
- Gson, Moshi, Kotlinx Serialization 등 다양한 Converter를 지원합니다
- 필요하다면 커스텀 `ConverterFactory`를 만들어 직렬화를 제어할 수도 있습니다

#### 4. 코루틴 완벽 지원
- `suspend` 키워드만으로 비동기 처리가 가능합니다
- 콜백 지옥에서 벗어나 순차적인 코드 작성이 가능합니다

#### 5. OkHttp 기반의 확장성
- `Interceptor`를 통해 공통 헤더, 로깅, 인증 토큰 삽입이 쉽습니다
- OkHttp의 모든 최적화 기능을 그대로 활용할 수 있습니다

#### 6. 뛰어난 유지보수성
- API가 늘어나도 코드가 복잡해지지 않습니다
- 각 API가 독립적인 메서드로 표현되어 관리가 쉽습니다
- 테스트 코드 작성이 훨씬 간편합니다

---

## Retrofit 내부 동작 원리 - 어떻게 인터페이스만으로 동작할까?

Retrofit의 마법 같은 동작은 **리플렉션(Reflection)**과 **동적 프록시(Dynamic Proxy)**를 활용합니다.

### 1. 프록시 객체 생성

`create()` 메서드가 호출되면 Retrofit은 Java의 `Proxy.newProxyInstance()`를 사용해 인터페이스의 프록시 객체를 생성합니다. 이 프록시는 `InvocationHandler`를 구현하며, 모든 메서드 호출을 가로챕니다.

### 2. 어노테이션 파싱

메서드가 호출되면 프록시는:
- `@GET`, `@POST` 등의 HTTP 메서드 정보
- `@Path`, `@Query`, `@Body` 등의 파라미터 정보
- 반환 타입 정보

를 리플렉션으로 읽어 `ServiceMethod` 객체를 생성합니다.

### 3. 요청 객체 구성

`ServiceMethod`는 내부의 `ParameterHandler`를 통해:
- `@Path`로 URL 경로 삽입
- `@Query`로 쿼리 파라미터 추가
- `@Body`로 요청 본문 직렬화

등의 작업을 수행해 OkHttp의 `Request` 객체를 만듭니다.

### 4. 네트워크 요청 실행

구성된 `Request`는 OkHttp 클라이언트를 통해 실제 네트워크 요청으로 실행됩니다.

### 5. 응답 변환

응답이 돌아오면:
- `Converter`(예: Gson)가 JSON을 객체로 변환
- `CallAdapter`가 반환 타입에 맞게 결과를 래핑 (코루틴, RxJava, Call 등)

### 6. 캐싱 최적화

생성된 `ServiceMethod`는 캐시되어, 같은 메서드가 재호출될 때 재사용됩니다.

---

## Retrofit의 비동기 처리 - suspend 키워드의 비밀

Retrofit 2.6.0 이후부터는 `suspend` 함수를 직접 사용할 수 있습니다. 하지만 어떻게 키워드 하나만으로 비동기가 동작할까요?

### CallAdapter의 역할

Retrofit은 `CallAdapter.Factory`를 통해 함수의 반환 타입을 감지합니다:

- `Call<T>` → 일반 Call 어댑터
- `suspend fun` → 코루틴 어댑터
- `Observable<T>` → RxJava 어댑터

`suspend` 함수가 감지되면 Retrofit은:

1. 네트워크 요청을 `Dispatchers.IO`에서 실행
2. 응답을 호출한 코루틴 컨텍스트로 반환
3. 스레드 전환(IO → Main)을 자동으로 처리

따라서 개발자는 단순히 이렇게만 작성하면:
```kotlin
@GET("posts/{id}")
suspend fun getPost(@Path("id") id: Int): Post
```

Retrofit이 알아서 백그라운드에서 실행하고 결과를 안전하게 반환합니다.

---

## 실전 활용: 에러 처리 패턴

Retrofit은 명확한 에러 분류를 제공합니다:

### HttpException vs IOException

- **HttpException**: 서버 응답은 받았지만 상태 코드가 200~299가 아닌 경우 (404, 500 등)
- **IOException**: 네트워크 단절, 타임아웃 등 통신 자체가 실패한 경우

### Result 래핑 패턴

실무에서는 sealed class로 결과를 래핑하는 패턴이 흔합니다:
```kotlin
sealed class NetworkResult<out T> {
    data class Success<T>(val data: T) : NetworkResult<T>()
    data class Error(val exception: Throwable) : NetworkResult<Nothing>()
}
```

이렇게 하면 ViewModel에서 명확한 상태 관리가 가능합니다:
```kotlin
when (val result = repository.getPost()) {
    is NetworkResult.Success -> showPost(result.data)
    is NetworkResult.Error -> showError(result.exception)
}
```

---

## 실전 사례: Gzip 압축 대응기

팀 프로젝트에서 API 응답 압축을 도입하면서 겪은 경험을 공유하겠습니다.

### 상황

백엔드에서 응답을 gzip으로 압축하기로 결정했고, 클라이언트에서도 대응이 필요했습니다.

### 시행착오

OkHttp의 Transparent GZIP 기능을 명시적으로 활성화했더니 앱이 크래시했습니다. 원인은 **Retrofit이 이미 내부적으로 gzip 헤더(`Content-Encoding: gzip`)를 자동으로 처리**하고 있었기 때문입니다. OkHttp와 Retrofit이 동시에 압축 해제를 시도하면서 충돌이 발생한 것이죠.

### 해결과 검증

명시적 설정을 제거하고, Android Studio의 App Inspector로 성능을 측정했습니다:

- **응답 크기**: 209KB → 9.3KB (약 95% 감소)
- **응답 시간**: 476ms → 261ms (약 45% 개선)

Retrofit이 이미 최적화를 처리하고 있었고, 개발자가 별도로 신경 쓸 필요가 없었습니다.

---

## 테스트 가능한 설계

Retrofit의 인터페이스 기반 설계는 테스트를 매우 쉽게 만듭니다.

### Fake Repository 패턴
```kotlin
class FakePostRepository : PostRepository {
    override suspend fun getPost(id: Int): Post {
        return Post(id, "테스트 제목", "테스트 내용")
    }
}
```

### MockWebServer 활용

OkHttp 팀에서 제공하는 `MockWebServer`를 사용하면 실제 HTTP 통신처럼 테스트할 수 있습니다:
```kotlin
val mockWebServer = MockWebServer()
mockWebServer.enqueue(MockResponse().setBody("""{"id":1,"title":"테스트"}"""))

val retrofit = Retrofit.Builder()
    .baseUrl(mockWebServer.url("/"))
    .addConverterFactory(GsonConverterFactory.create())
    .build()
```

이를 통해 API 호출과 응답 파싱을 독립적으로 검증할 수 있습니다.

---

## 결론: Retrofit을 선택해야 하는 이유

처음 받았던 질문으로 돌아가 보겠습니다.

> "왜 Retrofit을 쓰나요?"

이제는 명확하게 답할 수 있습니다.

### 변경에 강한 코드

API 명세가 바뀌어도 인터페이스만 수정하면 됩니다. URL이 변경되거나, 파라미터가 추가되어도 어노테이션 몇 줄만 고치면 전체 코드가 즉시 반영됩니다.

### 유지보수의 용이함

10개의 API를 관리하든 100개의 API를 관리하든, 코드 복잡도는 선형적으로 증가하지 않습니다. 각 API가 독립적인 메서드로 표현되어 있어, 특정 API만 수정하거나 디버깅하기가 매우 쉽습니다.

### 팀 협업의 효율

네트워크 로직이 추상화되어 있어 백엔드 개발자와의 소통도 간편합니다. "이 엔드포인트에 이런 파라미터를 추가해주세요"라는 요청이 오면, 인터페이스에 `@Query` 하나만 추가하면 끝입니다.

### 안전성과 생산성의 균형

타입 안전성을 보장하면서도 보일러플레이트 코드를 최소화합니다. 컴파일 타임에 오류를 잡으면서도, 실제 작성하는 코드는 몇 줄에 불과합니다.

---

HttpURLConnection부터 시작해 OkHttp를 거쳐 Retrofit까지, 각 단계는 이전 방식의 문제를 해결하기 위한 진화였습니다. 

Retrofit은 단순히 "편한 라이브러리"가 아니라, **변경에 유연하고 유지보수하기 쉬운 코드를 작성할 수 있게 해주는 아키텍처적 도구**입니다.

다음에 누군가 "왜 Retrofit을 쓰나요?"라고 묻는다면, 이제 자신 있게 답할 수 있을 것입니다.
