Retrofit이란 무엇인가?

Retrofit은 Square에서 개발한 type-safe HTTP 클라이언트로, 자바 및 안드로이드에서 REST API 통신을 더 쉽고 안전하게 해주는 라이브러리입니다.

개발자가 직접 HTTP 요청을 구성하지 않아도, 마치 인터페이스 메서드를 호출하듯 안전하게 네트워크 요청을 보낼 수 있게 해주는 도구로 사용할 수 있습니다.

네트워크 개념에 익숙하지 않은 독자도 이해할 수 있도록, 먼저 HTTP 통신의 기초부터 설명한 뒤, Retrofit이 왜 유용한지, 그리고 내부적으로 어떻게 동작하는지를 순차적으로 풀어가겠습니다.

1. HTTP와 클라이언트-서버 통신의 기초
HTTP란?

HTTP는 Hypertext Transfer Protocol의 줄임말로, 웹 클라이언트(브라우저, 앱 등)와 웹 서버 간에 데이터를 주고받는 규약입니다.

클라이언트가 “요청(request)”을 보내면, 서버가 “응답(response)”을 돌려주는 구조죠.

여기서 말하는 클라이언트란, 요청을 보내는 쪽을 의미한다.
예를 들어 브라우저나 모바일 앱이 클라이언트가 되고, 요청을 받는 쪽인 서버는 데이터를 보내주는 역할을 한다.

클라이언트와 서버는 어떻게 소통하는가?

엔드포인트(endpoint)
　클라이언트와 서버가 통신하기 위한 주소(URL)를 엔드포인트라 부릅니다.
　예: https://api.base.url/blogpost/1

클라이언트(앱)는 HTTP 요청을 보냅니다.

서버는 요청을 처리하고, JSON, XML 등 형식으로 응답 본문(body)을 돌려줍니다.

2. 안드로이드의 네트워크 통신: HttpURLConnection

안드로이드에서 기본적으로 제공하는 네트워크 통신 수단 중 하나가 HttpURLConnection입니다.
다음은 단순한 요청 예시:

```
fun request() {
    thread {
        val url = URL("https://api.base.url/blogpost/1")
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
이 코드는 동작은 하지만, 실제로 사용할 때 다음과 같은 문제점들이 있습니다:

요청 메서드(GET, POST 등)가 문자열로 지정되어 컴파일 시점에 오류를 잡을 수 없고,
네트워크 요청은 메인(UI) 스레드에서 네트워크 요청을 수행할 수 없어 반드시 별도의 스레드로 감싸야 한다.

네트워크 중단, 응답 실패, 파싱 오류 등의 예외를 모두 직접 처리해야 하며,
응답을 다 읽은 뒤에는 스트림과 연결을 닫아줘야 메모리 누수가 발생하지 않는다.

응답으로 받은 JSON을 직접 파싱해야 하고, 여러 API를 관리할 경우 URL, 파라미터, 헤더 설정 등이 중복되어 코드가 복잡해진다.

즉, 코드가 길고 실수하기 쉽고 중복도 많아집니다.

3. 안드로이드 네트워크 통신: OkHttp
이러한 문제를 해결하기 위해 Square에서 만든 것이 OkHttp입니다.

Retrofit 전 단계로 자주 사용되는 라이브러리가 OkHttp입니다. 이건 HTTP 클라이언트를 좀 더 사용하기 편하도록 추상화해 준 라이브러리예요.

```
val okHttpClient = OkHttpClient()
val request = Request.Builder()
    .url("https://api.base.url/blogpost/1")
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
                val json = JSONObject(body)
                json.getString("title")
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

OkHttp를 쓰면 다음과 같은 이점이 있습니다:

이전과 달리 enqueue() 메서드가 비동기로 동작하므로 스레드를 직접 만들 필요가 없다.
응답 본문도 response.body?.string()으로 한 줄이면 충분하며, Request.Builder로 URL, 헤더, 메서드를 명시적으로 구성할 수 있다.

Request.Builder를 통해 URL, 헤더, 메서드 등을 직관적으로 설정 가능하다.

커넥션 풀(connection pool)을 통한 재사용성, 타임아웃 기본값 관리 등이 내장되어 있어, 필요한 경우에 다시 설정해주면 된다.

또한 OkHttp는 커넥션 풀을 자동으로 관리해 불필요한 연결을 재활용하고,
HTTP 상태 코드(200~299)를 isSuccessful로 간단히 처리할 수 있다.

스트림을 열고 닫는 과정을 직접 작성할 필요도 없다.

하지만 OkHttp도 JSON 파싱, 여러 API 관리, 결합된 예외 처리, 콜백 기반 비동기 처리 등은 여전히 개발자가 직접 관리해야 합니다.

4. 안드로이드 네트워크 통신: Retrofit 인터페이스 + 어노테이션 기반으로 추상화

이제 본론인 Retrofit을 소개하겠습니다.

다음과 같은 코드만으로 API 요청을 정의할 수 있다.

```
interface RetrofitService {
    @GET("posts/{id}")
    suspend fun getPost(@Path("id") id: Int): Post
}
```
Retrofit 객체를 생성하면 된다.
```
val retrofitService = Retrofit.Builder()
    .baseUrl("https://api.base.url/")
    .addConverterFactory(GsonConverterFactory.create())
    .build()
    .create(RetrofitService::class.java)
```
네트워크 요청을 보내려면 단순히 retrofitService.getPost(1)을 호출하면 된다.
코루틴을 이용해 다음과 같이 처리할 수 있다.
```
lifecycleScope.launch {
    runCatching { retrofitService.getPost(1) }
        .onSuccess { post: Post ->
            textView.text = post.title
        }
        .onFailure { e ->
            when (e) {
                is HttpException -> {
                    // 서버 응답은 받았지만 2xx가 아님
                    textView.text = "HTTP 오류: ${e.code()}"
                }
                is IOException -> {
                    // 네트워크/타임아웃 오류
                    textView.text = "네트워크 오류: ${e.message}"
                }
                else -> {
                    textView.text = "예외: ${e.message}"
                }
            }
        }
}

```

Retrofit을 사용하면 비동기 네트워크 요청, 예외 처리, JSON 파싱까지 모두 자동으로 처리된다.

5. Retrofit 내부 동작 원리 (어노테이션 → 실행 흐름)

Retrofit의 핵심은 **리플렉션(Reflection)**과 **프록시(Proxy)**이다.
Retrofit은 우리가 작성한 인터페이스를 런타임에 분석해, 실제 네트워크 요청을 수행할 수 있는 “대리자” 객체를 만든다.


이 프록시는 인터페이스의 메서드 호출을 가로채고, 어노테이션 정보를 읽어 실제 HTTP 요청을 구성합니다.

이 과정에서 InvocationHandler가 사용됩니다. 이 대리자는 InvocationHandler라는 인터페이스를 구현하며,
인터페이스의 메서드가 호출될 때 대신 실행되는 동적 프록시 객체다.

Retrofit은 Java의 리플렉션(reflection)과 동적 프록시 기술을 사용하여, 개발자가 별도의 구현체를 작성하지 않아도 인터페이스를 실행 가능한 형태로 만듭니다.

어노테이션(@GET, @POST, @Path, @Query, @Body 등)을 읽어 RequestFactory 또는 ServiceMethod 객체를 생성합니다.


이 ServiceMethod는 파라미터 핸들러(ParameterHandler)를 통해 각 파라미터에 맞는 역할 (예: @Path 삽입, @Query 추가, @Body 직렬화 등)을 수행하는 로직을 내부에 가집니다.

이렇게 구성된 ServiceMethod 객체는 캐시되고, 같은 메서드가 여러 번 호출되면 재사용됩니다.

6. 실제 네트워크 요청 실행

인터페이스의 메서드를 호출하면, 프록시가 가로채서 ServiceMethod를 통해 OkHttp Request 객체를 만듭니다.

그 Request는 내부적으로 OkHttp 클라이언트를 통해 실행됩니다.

응답이 돌아오면, Retrofit의 Converter(예: GsonConverterFactory)가 응답 본문(JSON 등)을 Kotlin/Java 객체로 변환해 줍니다.

또한, 메서드의 반환 타입이 코루틴, RxJava, Call<T> 등 다양한 형태를 허용하기 위해 CallAdapter 체계를 사용합니다.

7. Retrofit의 장점

Retrofit의 가장 큰 장점은 간결함과 안전성, 그리고 확장성이다.

개발자는 복잡한 네트워크 로직을 작성하지 않아도, 마치 로컬 함수를 호출하듯 API를 호출할 수 있다.
타입 안전성이 보장되어 JSON 응답을 정의한 데이터 클래스 타입으로 바로 받을 수 있고,
형 변환이나 키 누락 오류를 줄일 수 있다.

Gson, Moshi, Kotlinx Serialization 등 다양한 Converter를 사용할 수 있으며,
필요하다면 직접 ConverterFactory를 만들어 커스텀 직렬화를 구현할 수도 있다.
OkHttp를 기반으로 하기 때문에 Interceptor를 통한 공통 헤더 추가, 로깅, 인증 토큰 삽입도 쉽다.

무엇보다 Retrofit은 API 개수가 늘어나도 코드가 복잡해지지 않는다.
각 API는 독립적인 인터페이스 메서드로 표현되며,
이를 한 곳에서 관리할 수 있어 유지보수성이 높고,
테스트 코드 작성도 훨씬 수월해진다.

Retrofit은 네트워크 세부 구현을 OkHttp로 숨기고,
개발자에게는 “어노테이션 기반 인터페이스”라는 추상적인 계약만 노출한다.
덕분에 우리는 네트워크 통신이라는 복잡한 문제를
안전하고 직관적인 방법으로 다룰 수 있게 되었다.


8. Retrofit 활용 사례 — Gzip 응답 압축 대응
팀 코스픽에서 코스 조회 API 응답을 압축하기로 결정되었다.
 이에 따라 클라이언트에서도 압축된 응답을 처리할 수 있도록 대응해야 했다.
백엔드가 선택한 응답 압축 방식은 gzip이었다.
gzip은 파일 압축에 사용되는 응용 소프트웨어로, 유닉스 시스템에서 쓰이던 기존 압축 프로그램을 대체하기 위해 만들어진 자유 소프트웨어다.
 우리에게 익숙한 zip처럼 DEFLATE 알고리즘을 사용하지만, 여러 파일을 하나의 압축 파일로 묶는 기능은 없다.
OkHttp는 이러한 gzip 응답을 자동으로 처리할 수 있도록 Transparent GZIP 기능을 지원한다.
 이 기능은 서버가 Content-Encoding: gzip 헤더를 보낼 경우, 내부적으로 압축을 해제해주는 역할을 한다.
 즉, 개발자가 별도로 설정하지 않아도 네트워크 트래픽을 줄이고 빠른 응답을 받을 수 있다.
당시 우리는 이 기능을 직접 활성화해서 사용자 경험을 개선하려고 했다.
 그러나 적용 후 앱이 크래시가 발생했다.
 이유는 Retrofit이 이미 내부적으로 gzip 헤더가 포함된 응답을 자동으로 해제하고 있었기 때문이다.
 즉, OkHttp와 Retrofit이 동시에 압축 해제를 시도하면서 충돌이 일어났다.
그래서 우리는 기능 자체를 비활성화하고, 압축이 적용되었을 때의 성능 차이를 확인하는 방향으로 검증을 진행했다.
 측정은 Android Studio의 App Inspector를 이용해, 애뮬레이터 환경에서 로컬 서버를 대상으로 수행했다.
 비교 기준은 “API 응답 압축 대응 커밋 전후”였다.
결과는 다음과 같았다.
응답 크기: 209KB → 9.3KB (약 95% 감소)


응답 시간: 476ms → 261ms (약 45% 개선)


압축 덕분에 데이터 전송량과 응답 속도 모두 크게 줄어드는 효과를 확인할 수 있었다.
 비록 Retrofit 내부 gzip 처리로 인해 직접적인 설정은 제거했지만,
 응답 압축이 네트워크 성능 개선에 얼마나 큰 영향을 주는지 실감할 수 있는 경험이었다.

9. Retrofit의 비동기 처리와 코루틴 (suspend 함수 내부 동작)

Retrofit 2.6.0 이후부터는 Call<T> 대신 suspend fun을 바로 사용할 수 있다.
이 말은 곧, enqueue() 같은 콜백 코드를 직접 작성하지 않아도 비동기 처리가 자동으로 이루어진다는 뜻이다.

그렇다면 왜 suspend 키워드를 붙이기만 해도 자동으로 비동기가 될까?

Retrofit은 내부적으로 CallAdapter라는 구조를 사용한다.
CallAdapter.Factory는 함수의 반환 타입을 감지하고, 그에 맞는 어댑터를 선택한다.
예를 들어, 함수가 Call<T>를 반환하면 일반 Call 어댑터를, suspend fun이면 코루틴 어댑터를 선택한다.
따라서 Retrofit이 직접 코루틴을 인식하고, 비동기로 동작하는 코드를 자동으로 만들어준다.

Retrofit이 이때 네트워크 요청은 Dispatchers.IO에서 실행하고, 응답은 withContext(Dispatchers.Main)으로 반환한다.
즉, 개발자가 스레드 전환(IO → Main)을 직접 신경 쓸 필요가 없다.
Retrofit이 코루틴 컨텍스트와 OkHttp 호출을 연결해주는 중간 계층 역할을 하기 때문이다.

결과적으로 우리는 단순히 아래처럼 선언하기만 해도,

```
@GET("posts/{id}")
suspend fun getPost(@Path("id") id: Int): Post

```

Retrofit이 알아서 백그라운드 스레드에서 네트워크를 처리하고, UI 스레드로 결과를 돌려준다.
 이런 구조 덕분에 코드가 훨씬 깔끔하고, 가독성이 높아진다.
공식 문서에서는 “Retrofit은 suspend 함수를 CallAdapter로 변환하며, 코루틴 컨텍스트 내에서 OkHttp의 Call을 실행한다”고 설명한다.

11. Retrofit과 에러 처리 (HttpException, IOException, Result Wrapping)

Retrofit은 단순히 네트워크 요청만 하는 도구가 아니다.
에러를 어떻게 분류하고, 어떻게 처리할 수 있게 해주는지도 명확하게 정의되어 있다.

먼저, 서버로부터 받은 응답의 상태 코드가 200~299가 아닐 경우, Retrofit은 HttpException을 던진다.
즉, HTTP 요청은 성공적으로 완료됐지만 서버가 오류를 반환한 경우(404, 500 등)가 여기에 해당한다.

반면, 네트워크 자체가 단절되었거나 타임아웃이 발생한 경우는 IOException으로 구분된다.
이 차이 덕분에 우리는 서버 문제와 네트워크 환경 문제를 구분해서 처리할 수 있다.

실무에서는 이 예외를 그대로 쓰지 않고, Result나 sealed class로 감싸는 패턴이 흔하다.
이 방식은 모든 결과를 성공(Success)과 실패(Error)로 명확하게 표현할 수 있어서,
UI 단에서는 단순히 상태를 구독하고 렌더링만 하면 된다.

예를 들어 다음과 같이 표현할 수 있다:

```
sealed class NetworkResult<out T> {
    data class Success<T>(val data: T) : NetworkResult<T>()
    data class Error(val exception: Throwable) : NetworkResult<Nothing>()
}

```

이렇게 하면 ViewModel에서는 try-catch 대신 when으로 분기 처리할 수 있다:

```
when (val result = repository.getPost()) {
    is NetworkResult.Success -> showPost(result.data)
    is NetworkResult.Error -> showError(result.exception)
}

```

공식 Retrofit 문서에서는 “HttpException은 HTTP 오류 상태 코드를 나타내며, IOException은 연결 또는 타임아웃 오류를 나타낸다”고 명시되어 있다.

13. 테스트 및 모킹(Mock) 서버 활용

Retrofit은 테스트하기 쉬운 구조를 가지고 있다.
그 이유는 인터페이스 기반 설계 때문이다.

Retrofit의 핵심은 “인터페이스만 정의하면, 나머지는 런타임에 자동으로 구현체를 만들어준다”는 점이다.
덕분에 실제 API 요청 없이도, 테스트 환경에서 가짜 응답을 돌려주는 코드(모킹, Mock)를 만들기 쉽다.

예를 들어, Retrofit이 아닌 단순한 Fake Repository를 만들어 사용할 수도 있다:

```
class FakePostRepository : PostRepository {
    override suspend fun getPost(id: Int): Post {
        return Post(id, "테스트 제목", "테스트 내용")
    }
}

```

또는, OkHttp의 MockWebServer를 이용해서 실제 HTTP 통신처럼 테스트할 수도 있다.
 이 서버는 로컬 환경에서 요청을 받고, 미리 정의한 응답을 돌려준다.
 이를 통해 “API가 제대로 호출되는지”, “응답이 올바르게 파싱되는지”를 검증할 수 있다.
공식 GitHub 문서에서도 MockWebServer를 Retrofit 테스트용으로 함께 사용하는 예시를 제공한다.
 MockWebServer는 OkHttp 팀에서 만든 도구로, Retrofit과 완벽하게 호환된다.


