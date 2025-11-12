# Retrofit

이 글은 Android에서 네트워크 작업을 간소화해주는 Retrofit 라이브러리에 대해 알아보고, 이를 이해하는데 필요한 개념을 기록한다. 

Retrofit은 내부적으로 OkHttp를 사용한다. OkHttp의 공식 설명(번역)은 다음과 같다.

---

OkHttp는 기본적으로 효율적인 HTTP 클라이언트이다.

- **HTTP/2 지원 -** 같은 호스트에 대한 모든 요청이 하나의 소켓을 공유할 수 있다.
- **커넥션 풀링 -** HTTP/2를 사용할 수 없는 경우에도 요청 지연 시간을 줄인다.
- **투명한 GZIP 압축 -** 다운로드 크기를 줄인다
- **응답 캐싱 -** 반복 요청 시 네트워크를 전혀 사용하지 않을 수 있다.

OkHttp는 네트워크가 불안정할 때도 안정적으로 동작한다. 일반적인 연결 문제를 자동으로 복구하며, 서비스가 여러 IP 주소를 가진 경우 첫 연결이 실패하면 다른 주소로 시도한다. 이는 IPv4와 IPv6 환경이나 여러 데이터 센터에 중복 호스팅된 서비스에서 필요하다. 또한 OkHttp는 최신 TLS 기능(TLS 1.3, ALPN, 인증서 핀닝)을 지원하며, 연결성을 위해 폴백(fallback) 설정도 할 수 있다.

OkHttp 사용법은 간단하다. 요청과 응답 API는 플루언트 빌더와 불변성을 기반으로 설계되어 있으며, 동기 호출과 비동기 호출(콜백 사용) 모두 지원한다.

---

OkHttp를 통해 GET 요청을 보내는 예제 코드를 통해 사용법을 알아보자.

```kotlin
fun main() {
    // OkHttpClient 인스턴스 생성
    val okHttpClient = OkHttpClient()

    // Request 객체 생성
    val request =
        Request
            .Builder()
            .url("https://api.example.com")
            .get() // GET 요청
            .build()

    try {
        val response: Response = okHttpClient.newCall(request).execute()
        if (response.isSuccessful) {
            println("Response Body: ${response.body.string()}")
        } else {
            println("Request failed with code: ${response.code}")
        }
    } catch (e: IOException) {
        e.printStackTrace()
    }
}
```

해당 코드를 세부적으로 파악해보자.

### OkHttpClient

```kotlin
// OkHttpClient 인스턴스 생성
val okHttpClient = OkHttpClient()
```

우선 `OkHttpClient` 를 선언한다. OkHttpClient는 HTTP 요청을 보내고 응답을 읽는 `Call` 객체를 위한 Factory다. 

OkHttp는 **OkHttpClient 인스턴스 하나를 생성하고, 이를 재사용하길 권장한다.** 

모든 HTTP 호출에서 이를 공유할 때 가장 좋은 성능을 낸다. 각 클라이언트는 자체적인 커넥션 풀(connection pool)과 스레드 풀(thread pool)을 가지기 때문이다. 커넥션과 스레드를 재사용하면 지연 시간이 줄고 메모리를 절약할 수 있다. 반대로, 매 요청마다 새로운 클라이언트를 생성하면 유휴 풀(idel pool)이 계속 만들어져 리소스가 낭비된다.

```kotlin
val okHttpClient =
    OkHttpClient
        .Builder()
        .addInterceptor(HttpLoggingInterceptor())
        .cache(
            Cache(cacheDir, cacheSize),
        ).build()
```

 또는 위처럼 Builder 패턴을 활용해 커스텀 설정을 가진 공유 인스턴스를 생성할 수 있다.

`newBuilder`를 통해 공유되는 OkHttpClient 인스턴스를 커스터마이징할 수 있다.

이것은 같은 커넥션 풀, 스레드 풀, 설정(Configuration)을 공유하는 클라이언트를 생성한다. 필요한 설정만 덧붙여 특정 목적에 맞는 클라이언트를 만들 수 있다. 

```kotlin
val okHttpClient =
    OkHttpClient
        .Builder()
        .readTimeout(1000, TimeUnit.MILLISECONDS)
        .writeTimeout(1000, TimeUnit.MILLISECONDS)
        .build()
```

위는 읽기, 쓰기 시간을 1000ms로 제한한 클라이언트를 생성하는 예시 코드다.

OkHttpClient의 종료(shutdown)는 필수적이지 않다. 

유휴 상태가 된 스레드와 커넥션은 자동으로 해제된다. 하지만 사용하지 않는 리소스를 적극적으로 정리해야 하는 애플리케이션이라면, 명시적으로 해제할 수도 있다. `dispatcher`의 ExecutorService를 종료(shutdown)하면 된다. 이 경우 이후의 요청은 더 이상 실행되지 않는다.

```kotlin
okHttpClient.dispatcher.executorService.shutdown()
```

OkHttpClient를 선언한 이유는 `Call` 을 통해 HTTP 요청하거나 응답을 받기 위해서다. OkHttpClient의 `newCall` 메서드를 통해 `Request` 를 전달하면 Call을 얻을 수 있다.

```kotlin
override fun newCall(request: Request): Call = RealCall(this, request, forWebSocket = false)
```

### Request

```kotlin
// Request 객체 생성
val request =
    Request
        .Builder()
        .url("https://api.example.com")
        .get() // GET 요청
        .build()
```

`Request` 객체는 HTTP 요청이다. Request를 만들기 위해서 Builder를 사용한다. Builder를 통해 `url`, `method`(GET, HEAD, POST, DELETE, PUT, PATCH), `body`, `tag` 등을 넣을 수 있다. 기본적으로 method는  body가 null일 경우 GET이며, 그렇지 않을 경우 POST이다. Request 객체는 OkHttpClient의 `newCall` , `newWebSocket` 같은 메서드로 전달할 수 있다.

### Call

```kotlin
try {
    val response: Response = okHttpClient.newCall(request).execute()
    if (response.isSuccessful) {
        println("Response Body: ${response.body.string()}")
    } else {
        println("Request failed with code: ${response.code}")
    }
} catch (e: IOException) {
    e.printStackTrace()
}
```

`okHttpClient.newCall(request)` 는 새로운 `Call` 객체를 만든다.

Call은 실행(execution)하기 위해 준비된 요청(request)이다. Call은 취소될 수 있다. 이 객체는 하나의 요청과 응답 쌍을 표현하며, 이것은 두 번 실행될 수 없다.

```kotlin
val response: Response = okHttpClient.newCall(request).execute()
```

`execute()` 메서드는 요청을 즉시 실행하고, 응답이 처리 가능해지거나 오류가 발생할 때까지 블로킹(block)된다.

요청이 취소되었거나, 연결 문제 혹은 타임아웃 등으로 실행할 수 없는 경우 `IOException`이 발생한다. 네트워크 교환 도중 실패할 수 있으므로, 원격 서버가 요청을 수락한 후에 실패가 발생할 수도 있다.

동일한 Call이 이미 실행된 경우 `IllegalStateException`이 발생한다.

```kotlin
fun enqueue(responseCallback: Callback)
```

`enque()` 메서드는 요청을 비동기적으로 미래의 어느 시점에 실행되도록 예약(schedule)한다. OkHttpClient의 `dispatcher` 가 요청이 언제 실행될지 결정한다. 일반적으로 다른 요청이 실행 중이지 않으면 즉시 실행한다. 

비동기적으로 실행되기에, 클라이언트가 HTTP 응답 또는 실패 예외를 담은 `responseCallback` 을 호출한다. 따라서 메서드 인자로 `responseCallback` 을 넘겨야 한다.

`Callback` 인터페이스는 다음처럼 생겼다.

```kotlin
interface Callback {
  fun onFailure(
    call: Call,
    e: IOException,
  )

  @Throws(IOException::class)
  fun onResponse(
    call: Call,
    response: Response,
  )
}
```

다음 코드는 Callback 인터페이스를을 구현하는 익명 객체를 만들고 `enqueue` 메서드로 전달한다.

```kotlin
okHttpClient.newCall(request).enqueue(object : Callback {
    override fun onFailure(call: Call, e: IOException) {
        e.printStackTrace()
    }
 
    override fun onResponse(call: Call, response: Response) {
        if (response.isSuccessful) {
            println(response.body?.string())
        }
    }
})
```

## Retrofit

지금까지 OkHttp 라이브러리를 통해 HTTP 요청을 보내고 받는 방법을 알아봤다.

단순히 HTTP 요청을 보내고 응답을 받는 것은 OkHttp로도 충분하다. 요청마다 URL을 문자열로 작성하고, 응답을 직접 파싱해야 하며, API가 많아질수록 코드가 복잡해진다.

Retrofit은 이러한 반복적인 작업을 인터페이스 기반으로 추상화해, 선언적이고 유지보수하기 쉬운 방식으로 네트워크 코드를 관리할 수 있도록 돕는다. 쉽게 말해, Retrofit은 OkHttp를 더욱 쉽게 사용하기 위한 추상화 계층이라 볼 수 있다.

[2.6.0 버전](https://github.com/square/retrofit/blob/trunk/CHANGELOG.md#260---2019-06-05)부터는 Kotlin의 `suspend` 키워드를 지원한다. 이것이 `suspend` 키워드를 사용해 정의한 함수에서 `Call` 을 쓰지 않고, `Response<T>`  또는 `T` 타입으로 반환값을 정의할 수 있는 이유이다.

```kotlin
@GET("users/{id}")
suspend fun user(@Path("id") id: Long): User
```

위 코드는 `fun user(...): Call<User>` 처럼 정의되어 있고, `Call.enque` 를 호출한 것처럼 처리된다. 또한, 응답 메타데이터(예: HTTP 상태 코드, 헤더 등)에 접근하고 싶다면 `Response<User>`를 반환하도록 선언할 수도 있다.

```kotlin
@GET("users/{id}")
suspend fun user(@Path("id") id: Long): Response<User>
```

---

Retrofit을 통해 API 요청을 방법을 예제 코드를 통해 알아보자.

```kotlin
interface GitHubService {
    @GET("/repos/{owner}/{repo}/contributors")
    fun listRepos(
        @Path("owner") owner: String,
        @Path("repo") repo: String,
    ): Call<List<Contributor>>
}
```

우선적으로 사용할 API를 선언한 인터페이스를 만들어야 한다. `/repos/{owner}/{repo}/contributors`는 GitHub 특정 리포지토리의 컨트리뷰터들을 요청할 수 있는 엔드포인트다. 앞에 BASE_URL이 명시되어 있지 않은데, 이것은 주로 재사용될 것이므로 Retrofit 객체를 만들 때 등록한다. `GET` 요청을 보낼 것이므로 `@GET` 어노테이션 내에 url을 명시한다.

```kotlin
private const val BASE_URL = "https://api.github.com"

fun main() {
		// 1. Retrofit 객체 생성
    val json = Json { ignoreUnknownKeys = true }
    val retrofit: Retrofit =
        Retrofit
            .Builder()
            .baseUrl(BASE_URL)
            .addConverterFactory(
                json.asConverterFactory("application/json; charset=UTF8".toMediaType()),
            ).build()

		// 2. Retrofit을 통해 API Service 인터페이스 구현체 생성
    val gitHubService: GitHubService = retrofit.create(GitHubService::class.java)
    
    // 3. API 요청 및 응답 처리
    val requestCall: Call<List<Contributor>> = gitHubService.listRepos("square", "retrofit")
    val response: Response<List<Contributor>?> = requestCall.execute()
    
    val contributors: List<Contributor>? = response.body()
    contributors?.forEach {
        println(it.login)
    }
}

```

코드는 다음 로직을 순차적으로 실행한다.

1. `Retrofit` 객체 생성한다.
2. Retrofit을 통해 API Service 인터페이스(`GitHubService`)의 구현체를 생성한다.
3. `Call` 객체를 `execute()`하고, `Response` 응답을 처리한다.

3번 과정부터는 OkHttp를 통해 응답을 처리하는 방식과 동일하다.


### 참고 문서 출처
- [OkHttp](https://square.github.io/okhttp/)
- [Retrofit](https://square.github.io/retrofit/)
