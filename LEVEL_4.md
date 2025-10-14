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


8. 추가 retrofit 활용 사례
팀 코스픽에서 코스 조회 API 응답 압축을 진행하기로 결정 되어, 그에 맞는 응답 압축 대응을 했어야 했다.

백엔드가 응답 압축하는 방법으로 택한 것은 gzip이라는 친구였다. 

gzip이란 무엇일까? 파일 압축에 쓰이는 응용 소프트웨어로, 유닉스 시스템에 쓰이던 압축 프로그램을 대체하기 위한 자유 소프트웨어이다.

우리에게 익숙한 zip과 같이 DEFLATE 라는 알고리즘을 따르지만, 여러 파일을 하나의 파일로 압축하는 옵션이 없다는 점에서 차이가 난다.

okhttp에는 transparent GZIP이라는 응답 데이터를 압축하여 네트워크 트래픽을 줄여주는 기능이 있다.

해당 기능을 활성화해준다면 gzip으로 응답을 받아 더욱 개선된 사용자 경험을 제공해줄 수 있다는 기대를 가지고 적용했으나, 설정을 적용하니 오히려 앱이 크러시가 났다.

그 이유는 Retrofit에는 서버로부터 헤더에 gzip이라는 정보가 있으면 내부적으로 압축을 해제하는 처리를 하고 있었기 때문이었다.

아쉬운대로 적용 전과 적용 후의 성능을 안드로이드 스튜디오에서 app inspector를 사용하여 애뮬레이터 환경에서 로컬 서버를 사용하여 API 응답 압축 대응 커밋 전 후를 기준으로 실행시키며 네트워크를 요청하며 관찰하였다.

결과는 적용 전에는 209KB 사이즈였던 응답은 gzip 적용 후 9.3KB로 줄어드는 약 95% 감소하였으며, 476ms이 걸리던 시간 요청에 대해서는 261ms으로 감소하여 약 45%의 속도 개선을 확인할 수 있었다.

