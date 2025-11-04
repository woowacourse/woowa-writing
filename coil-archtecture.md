# Coil: 단순함 뒤에 숨겨진 아키텍처   

![](https://coil-kt.github.io/coil/logo.svg)

### 들어가며
많은 개발자가 이미지 로딩 라이브러리를 "단순히 URL을 요청하고,   
비트맵으로 변환하여 View에 전달하는" 간단한 작업으로 치부하곤 합니다. 저 역시 그랬습니다.   

하지만 막상 Coil의 내부를 깊이 들여다보니, 그 생각이 얼마나 안일했는지 깨닫게 되었습니다.   
Coil은 겉으로 보이는 단순함 뒤에 캐싱 전략, 메모리 관리, 생명주기 제어,   
비동기 처리 최적화 등 수많은 복잡성과 섬세한 고려 사항을 담고 있었습니다.  

Coil의 내부 동작을 분석한 경험을 공유하며,  
Android 환경에서 Coil이 이미지 로딩을 위해 무엇을 고려하는지 실피며 자세한 내부 동작 원리를 알아보자 합니다. 

### ImageRequest

```kotlin
inline fun ImageView.load(
   data: Any?,
   imageLoader: ImageLoader = context.imageLoader,
   builder: ImageRequest.Builder.() -> Unit = {},
): Disposable {
   val request = ImageRequest.Builder(context)
       .data(data)
       .target(this)
       .apply(builder)
       .build()
   return imageLoader.enqueue(request)
}
```
흔히 사용하는 ```ImageView.load()```의 최상단 내부 구조입니다.   
이 중 눈여겨 봐야 할 부분은 ImageRequest와 ImageLoader 입니다.    

ImageRequest는 이미지를 로딩할 때 다양한 설정 정보들을 담는 클래스입니다.    
data() 함수의 인자는 Any? 입니다.    
여기에는 url 링크, Android Drawable 등이 들어갈 수 있습니다.    
target() 함수는 이미지가 로딩된 후 어떤 객체에 반영할 것인지 지정하는 확장 함수입니다.    
이외에도  diskCachePolicy, networkCachePolicy, scale 등 캐시와 이미지 사이즈,   
등을 설정할 수 있는 다양한 옵션을 제공합니다. cache에 대해서는 뒤에서 자세히 다루겠습니다.    
 
### ImageLoader
>A service class that loads images by executing ImageRequests. Image loaders handle caching, data fetching, image decoding, request management, memory management, and more. Image loaders are designed to be shareable and work best when you create a single instance and share it throughout your app.

ImageLoader는 Android 환경에서 싱글톤으로 제공되며. ```context.imageLoader``로 접근할 수 있습니다. 

ImageLoader의 설명입니다 ImageRequest에서 설정했던 요청 옵션들을 ImageLoader에서 실행합니다.   
요청은 enqueue, execute 두 가지로 나뉘며 enqueue는 메시지 큐 방식의 비동기를 실행하고 execute는 코루틴 기반으로 실행합니다.   
하지만 사실 enqueue도 execute와 동일한 내부 메서드를 사용합니다

ImageLoader는 인터페이스며, RealImageLoader라는 실제 구현체에서 이미지 요청을 처리합니다.  

```kotlin
// RealImageLoader (ImageLoader 인터페이스의 구현체)
override fun enqueue(request: ImageRequest): Disposable {
   // Start executing the request on the main thread.
   val job = scope.async(options.mainCoroutineContextLazy.value) {
       //execute()에서도 플래그만만 다를 뿐, 동일한 메서드를 실행합니다.
       execute(request, REQUEST_TYPE_ENQUEUE)
   }

   // Update the current request attached to the view and return a new disposable.
   return getDisposable(request, job)
}
```

Disposable은 ImageLoader로 로드한 후 반환하는 객체입니다.   
Disposable 객체는 Kotlin의 job을 래핑하여 View에 이미지가 붙었을 때,  
coil에서 이미지 리소스를 해제하는 클래스라고 보시면 되겠습니다. 

### Execute

우리는 enqueue와 execute 모두 execute를 사용한다는 점을 알았습니다.
이제부터는 공통 요청 부분을 보도록 합시다

```kotlin
val requestDelegate = requestService.requestDelegate(
   request = initialRequest,
   job = coroutineContext.job,
   findLifecycle = type == REQUEST_TYPE_ENQUEUE,
).apply { assertActive() }
```

이미지 요청을 할 객체인 Service를 만드는 부분입니다.   
RequestDelegate라는 객체로 Service를 감싸고 있습니다.   

**Wrap request to automatically dispose and/or restart the ImageRequest based on its lifecycle.**

requestDelegate 메서드에 작성된 설명입니다.   
requestService를 바로 사용할 수도 있지만 Delegate를 사용하게 된다면 자동으로 job을 닫고,   
액티비티의 생명주기에 따라 재요청을 보낼 수 있습니다. (하위 구현체인 ViewTargetRequestDelegate에서 수행합니다)

```kotlin
internal interface RequestDelegate {
   /** Throw a [CancellationException] if this request should be cancelled before starting. */
   fun assertActive() {}

   /** Register all lifecycle observers. */
   fun start() {}

   /** Wait until the request's associated lifecycle is started. */
   suspend fun awaitStarted() {}

   /** Called when this request's job is cancelled or completes successfully/unsuccessfully. */
   fun complete() {}

   /** Cancel this request's job and clear all lifecycle observers. */
   fun dispose() {}
}
```

RequestDelegate의 인터페이스는 다음과 같으며,   
ImageLoader의 구현체인 RealImageLoader의 execute() 메서드에서 start() 메서드를 호출합니다.   
하단 코드는 ImageView.load()를 실행했을 때 제공되는 ViewTargetRequestDelegate의 구현체입니다.   

```kotlin
override fun start() {
   lifecycle?.addObserver(this)
   if (target is LifecycleObserver) {
       lifecycle?.removeAndAddObserver(target)
   }
   target.view.requestManager.setRequest(this)
}
```

이후, 실제 이미지 요청 전(네트워크, 로컬 둘 다) PlaceHolder를 설정하고,  
sizeResolver에서 View에 들어갈 이미지 사이즈를 계산을 마칩니다.   
 
```kotlin
// Set the placeholder on the target.
val cachedPlaceholder = request.placeholderMemoryCacheKey?.let { memoryCache?.get(it)?.image }
request.target?.onStart(placeholder = cachedPlaceholder ?: request.placeholder())
eventListener.onStart(request)
request.listener?.onStart(request)

// Resolve the size.
val sizeResolver = request.sizeResolver
eventListener.resolveSizeStart(request, sizeResolver)
val size = sizeResolver.size()
eventListener.resolveSizeEnd(request, size)
```

### EngineInterceptor  
그 다음으로는 Interceptor를 통해 이미지를 요청합니다.  

```kotlin
// Execute the interceptor chain.
val result = withContext(request.interceptorCoroutineContext) {
   RealInterceptorChain(
       initialRequest = request,
       interceptors = components.interceptors,
       index = 0,
       request = request,
       size = size,
       eventListener = eventListener,
       isPlaceholderCached = cachedPlaceholder != null,
   ).proceed()
}
```

눈여겨볼 점은 interceptor는 여러개가 등록될 수 있다는 점입니다.   
커스텀 Interceptor를 등록하여 로깅, 재시도 요청, timeout 기능 등을 구현할 수 있습니다.   

```kotlin
override suspend fun proceed(): ImageResult {
   val interceptor = interceptors[index]
   val next = copy(index = index + 1)
   val result = interceptor.intercept(next)
   checkRequest(result.request, interceptor)
   return result
}
```

ImageLoader를 커스텀하지 않으면,   
Coil에서 기본적으로 제공하는 EngineInteceptor 하나만 등록하게 됩니다.   
이 Interceptor는 Coil의 핵심 클래스이며, 실제 ImageRequest를 받아 이미지를 요청하는 클래스입니다. 

### Engineinterceptor.intercept()  
다음으로  Engineinterceptor.intercept()의 메서드를 살펴봅시다.   
함수 시그니처는 다음과 같습니다.   

```kotlin
override suspend fun intercept(chain: Interceptor.Chain): ImageResult 
```

Engineinterceptor.intercept()의 하는 일을 다음과 같습니다.   
**1. data의 타입을 분석하여 실제 Coil이 처리 가능한 타입으로 변환**

intercept 메서드가 반환하는 ImageResult 타입은 이미지 요청 결과로 받은 실제 이미지 비트맵 정보가 담겨 있습니다.   
첫 번째로 EngineInterceptor가 하는 일은 data의 타입을 분석하여   
실제 Coil이 처리 가능한 타입으로 변환하는 것입니다.   

```kotlin
val mappedData = imageLoader.components.map(data, options)
```

Coil에서 처리가 가능한 타입은 다음과 같습니다. 
- android.net.Uri  
- Integer  
- File  
- String  
- okio.Path
일반적인 이미지 url은 android.net.Uri로 변환됩니다.   

**2. cacheKey를 생성하여 메모리에 캐시에 이미지가 있는지 확인한 후, 이미지가 있다면 바로 결과를 반환**  

```kotlin
val cacheKey = memoryCacheService.newCacheKey(request, mappedData, options, eventListener)
val cacheValue = cacheKey?.let { memoryCacheService.getCacheValue(request, it, size, scale) }

// Fast path: return the value from the memory cache.
if (cacheValue != null) {
   return memoryCacheService.newResult(chain, request, cacheKey, cacheValue)
}
```

Coil은 메모리 캐시를 MemoryCacheService에서 관리하며,    
StrongMemoryCache와 WeakMemoryCache 두 가지의 캐시로 관리됩니다.    

StrongMemoryCache는 내부적으로 LinkedHashMap을 사용하여 Lru 알고리즘을 구현하여,   
StrongMemoryCache는 MaxSize를 넘지 않도록 가장 오랫동안 사용되지 않은 캐시부터 제거합니다.  

WeakMemoryCache는 StrongMemoryCache에서 요소가 제거되었을 때, 값이 이동하며,   
WeakMemoryCache는 Map의 값으로 WeakReference를 사용하여 GC에 의해 언제든지 제거되도록 유도하여 캐시를 관리합니다.    


**3. mapper로 치환한 타입에 따라 Fetcher를 선택하고 이미지를 요청** 
요청은 EngineInterceptor의 private 메서드인 fetch()에서 진행합니다.  
Fetcher의 종류에는 여러가지가 있지만 이미지 url을 로드할 때는 NetworkFetcher를 사용합니다.  

NetworkFetcher는 내부적으로 네트워크 요청 전, 추가적으로 디스크 캐시에서 이미지가 캐싱되었는지 확인합니다. 

```kotlin
//NetworkFetcher의 메서드
private fun readFromDiskCache(): DiskCache.Snapshot? {
        if (options.diskCachePolicy.readEnabled) {
            return diskCache.value?.openSnapshot(diskCacheKey)
        } else {
            return null
        }
    }
``` 
디스크 캐시를 관리하는 DiskCache 또한 LinkedHashMap으로 구현되어 Lru 알고리즘을 사용하며,    
내부적으로 Okio 라이브러리를 사용하여 디스크 캐시를 관리하고 있습니다.     
디스크에 캐시가 없다면 그제서 네트워크 요청을 보냅니다.   

```kotlin
@JvmInline
internal value class CallFactoryNetworkClient(
   private val callFactory: Call.Factory,
) : NetworkClient {
   override suspend fun <T> executeRequest(
       request: NetworkRequest,
       block: suspend (response: NetworkResponse) -> T,
   ) = callFactory.newCall(request.toRequest()).await().use { response ->
       block(response.toNetworkResponse())
   }
}
```

그 후 OkHttp를 사용하여 네트워크 요청을 처리한 후,   
내부적으로 요청 결과값을 Android에서 제공하는 BitmapFactory를 사용하여 결과값을 디코딩합니다.  

```kotlin
fun interface Decoder {
   /**
    * Decode the [SourceFetchResult] provided by [Factory.create] or return 'null' to delegate
    * to the next [Factory] in the component registry.
    */
   suspend fun decode(): DecodeResult?

   fun interface Factory {
       /**
        * Return a [Decoder] that can decode [result] or 'null' if this factory cannot
        * create a decoder for the source.
        *
        * Implementations **must not** consume [result]'s [ImageSource], as this can cause calls
        * to subsequent decoders to fail. [ImageSource]s should only be consumed in [decode].
        *
        * Prefer using [BufferedSource.peek], [BufferedSource.rangeEquals], or other
        * non-consuming methods to check for the presence of header bytes or other markers.
        * Implementations can also rely on [SourceFetchResult.mimeType], however it is not
        * guaranteed to be accurate (e.g. a file that ends with .png, but is encoded as a .jpg).
        *
        * @param result The result from the [Fetcher].

        * @param options A set of configuration options for this request.

        * @param imageLoader The [ImageLoader] that's executing this request.
        */

       fun create(
           result: SourceFetchResult,
           options: Options,
           imageLoader: ImageLoader,
       ): Decoder?
   }
}
```

### 만약 여러 스레드에서 동시에 요청을 보낸다면?

Coil은 네트워크에 요청 전 메모리 캐시와 디스크 캐시를 조회합니다.  
모든 캐시는 전부 synchronized를 사용하며, 
이미지 로딩 라이브러리 특성 상 메인 스레드에서 주로 이미지 요청을 시도합니다.  
때문에 대부분의 경우에는 한 번 요청을 보냈을 때 캐시에 저장된 값을 사용하지만,  
만약 여러 스레드에서 같은 URL을 동시에 요청했을 때, 요청을 여러번 보내게 됩니다.  

관련 이슈 : https://github.com/coil-kt/coil/issues/1461

이 부분은 추후 개선이 될 여지가 높아 기다려봐도 좋을 것 같습니다.   

### LruCache 
이미지 비트맵은 상당히 큰 용량을 차지합니다.   
적절한 캐싱 전략을 세우지 않으면 OOM이 발생하여 크래시가 날 수 있습니다.  

Coil3는 이미지를 처리하기 위해 ImageRequest에 명시한 Size대로 다운샘플링합니다.    
하지만 이것으론 충분하지 않습니다. 앱을 오래 사용하게 될 떄 이미지를 계속 메모리에 적재하게 된다면 
언젠가는 OOM이 필연적으로 발생하게 됩니다. 이를 방지하기 위해 Coil은 LruCache를 사용합니다.  

```kotlin
internal class RealStrongMemoryCache(
   override val initialMaxSize: Long,
   private val weakMemoryCache: WeakMemoryCache,
) : StrongMemoryCache {
   private val cache = object : LruCache<Key, InternalValue>(initialMaxSize) {

       override fun sizeOf(
           key: Key,
           value: InternalValue,
       ) = value.size

       override fun entryRemoved(
           key: Key,
           oldValue: InternalValue,
           newValue: InternalValue?,
       ) = weakMemoryCache.set(key, oldValue.image, oldValue.extras, oldValue.size)
   }
```

LruCache는 **"Least Recently Used Cache"**의 약자로, 메모리 캐시를 구현하는 데 사용되는 클래스입니다.  
그 주된 목적은 메모리 내에 제한된 크기의 공간을 설정하고, 자주 사용되는 데이터를 빠르게 접근하기 위해 저장하는 것입니다.   
Android 환경에서는 이미지 로딩 라이브러리(Coil, Glide 등)나 기타 데이터 캐싱 목적으로 널리 사용되며, 
디스크 I/O나 네트워크 통신과 같은 비용이 큰 작업을 줄여 애플리케이션의 성능과 반응성을 높이는 데 필수적입니다.  

LruCache는 이름처럼 LRU(Least Recently Used) 알고리즘을 기반으로 작동합니다.   
이 알고리즘의 핵심은 캐시가 지정된 최대 크기에 도달했을 때, 어떤 데이터를 버릴지 결정하는 것입니다.    
LruCache는 데이터가 접근된 시간을 기준으로 순서를 관리하며, 캐시 공간이 필요해지면 가장 오랫동안 사용되지 않은(가장 오래된) 데이터를 먼저 제거합니다.   
이는 최근에 사용된 데이터는 곧 다시 사용될 가능성이 높다는 시간적 지역성(Temporal Locality) 원칙에 근거합니다.    




