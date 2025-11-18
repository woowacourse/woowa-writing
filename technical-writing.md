# 테크니컬 라이팅

## TL;DR

야구보구 앱 프로필 사진 업로드 기능 개발 과정에서 발생한 **AWS S3 Pre-signed URL과 Retrofit Bearer 토큰 인터셉터 충돌 문제를 해결**하고, uCrop을 통한 이미지 최적화와 UX를 개선한 기록입니다.

## 대상 독자

- **Kotlin, Retrofit, OkHttp, Coil** 등 안드로이드 네트워킹에 대한 기본적인 이해가 있는 **주니어 안드로이드 개발자**
- **관심사**: **Pre-signed URL을 받아** AWS S3 **파일 업로드** 기능 구현에 도전하고 싶은 개발자
- **HTTP 인터셉터와 외부 서비스 연동** 시 발생하는 인증 헤더 충돌 문제의 실무 해결 사례가 궁금한 개발자

***

# 프로필 사진 S3 Pre-signed URL 업로드 삽질기

<img width="409" height="123" alt="Image" src="https://github.com/user-attachments/assets/b7d44291-526e-4337-b08d-25b8bae00ade" />

**야구보구**는 우아한테크코스 내에서 개발 중인 프로야구 직관 기반 통계 기록 및 팬 커뮤니티 앱입니다. 경기장에서 버튼 한 번으로 직관 인증과 기록이 가능하고, 팬들끼리 소통할 수 있는 플랫폼을 제공합니다.

개발 당시 구글 SSO 로그인으로 간편 가입을 도입함과 더불어 구글 프로필 이미지를 그대로 사용하여 프로필을 표시했으나, "**프로필 사진을 바꾸고 싶어요!**"라는 사용자 피드백이 들어오면서 프로필 사진 업로드 기능을 추가하게 되었습니다.

사진을 업로드하는 단순해 보이는 이 기능이 어떤 기술적 도전을 가져다주었고 어떤 방식으로 해결했는지 궁금하다면 계속 읽어주세요.

## 그래서 AWS S3 Pre-signed URL이 뭔데?

![지하철 일회용 승차권, Perplexity 생성 이미지](https://github.com/user-attachments/assets/149743c0-db74-4b0b-8f20-c06fadf39ab9)


**제한된 접근 권한을 가진 일회용 티켓**과 비슷합니다. 특정 파일, 특정 작업(업로드/다운로드), 특정 시간 동안만 유효한 임시 접근 권한을 제공합니다.

발급받은 URL로 지정된 용량의 파일을 단 한 번만 전송이 가능합니다.

**Pre-signed URL의 특성**:

- URL 파라미터에 인증 정보(`X-Amz-Signature`, `X-Amz-Credential` 등) 포함
- Content-Type, Content-Length도 서명에 영향을 미칠 수 있음

```
https://s3.amazonaws.com/bucket/key?
X-Amz-Algorithm=AWS4-HMAC-SHA256&
X-Amz-Credential=AKIAIOSFODNN7EXAMPLE%2F20231013%2Fus-east-1%2Fs3%2Faws4_request&
X-Amz-Date=20231013T000000Z&
X-Amz-Expires=3600&
X-Amz-Signature=abc123...
```

## 그래서 왜 Pre-signed URL을 쓰는 건데?

S3 버킷은 Amazon S3 스토리지 서비스에서 파일을 저장하는 하나의 컨테이너입니다. 
야구보구에서는 운영 서버 부하를 줄이기 위해 클라이언트에서 S3에 직접 업로드할 필요성이 생겼습니다.

일반적인 파일 업로드 방식이라면:
1. 클라이언트 → 서버로 이미지 전송
2. 서버에서 이미지 검증 및 처리  
3. 서버 → S3로 최종 업로드

하지만 Pre-signed URL을 사용하면:
1. 서버에서 임시 업로드 URL만 생성
2. **클라이언트가 S3에 직접 업로드** (서버 경유 없음)
3. 완료 후 서버에 결과만 통보

즉, 퍼블릭 액세스가 불가능한 S3 버킷에 직접 접근하기 위한 일회성 URL이 Pre-signed URL인 것입니다.
따라서 야구보구의 운영 서버는 단순히 URL 생성과 완료 확인 역할만 하게 되어 리소스 절약과 업로드 속도 향상의 효과를 얻을 수 있습니다.

## 잘 모르겠고 업로드부터 해보자

우선 야구보구의 백엔드를 담당하는 두리와 포라가 `Pre-signed url 조회`와 `Pre-signed url 업로드 확인`이라는 두 개의 API를 만들어 주셨고 아래와 같은 과정을 통해 업로드 기능을 구현하기 시작했습니다.

### API를 통해 사진을 업로드하는 과정

1. **야구보구 서버에서 AWS의 Pre-signed URL 받아오기**
    
    업로드할 파일의 MIME 타입, 업로드할 파일 크기(바이트)를 `Request Body`로 알려주면 다음과 같은 응답이 도착합니다. key는 S3 객체의 고유한 식별자 키이며, url은 Pre-Signed URL입니다.
    
    ```json
    {
      "key": "yagubogu/images/profile/sample.jpg",
      "url": "https://yagubogu-2025.s3.ap-northeast-2.amazonaws.com/yagubogu/images/profile/sample.jpg?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=...&X-Amz-Expires=600&X-Amz-SignedHeaders=host&X-Amz-Signature=..."
    }
    ```
    
2. **AWS S3 서버로 이미지 전송**
    
    1번에서 받은 Pre-Signed URL로 HTTP PUT 메서드를 사용해 백엔드 서버를 거치지 않고 AWS로 이미지를 직접 전송해야 합니다.
    
    URL로 보낼 이미지 데이터는 `RequestBody`에 담아 보내는데, `contentType(MIME 타입 문자열)`, `contentLength(파일 크기)`, `writeTo(파일 데이터)`를 같이 담아 보냅니다.
    
    > **MIME 타입이란?**
    > 
    > 문서나 파일의 종류와 형식을 나타내는 식별자로서 `image/jpeg`, `text/html`, `video/mp4` 등 서버가 데이터를 어떻게 처리해야 할지 알려주기 위한 꼬리표 같은 역할입니다.
        
3. **야구보구 서버에 성공했다고 알려주기**
    
    AWS S3 버킷에 파일 전송이 성공(200)했다면 성공했다는 사실을 우리 앱의 백엔드 서버에 알려줘야 업로드가 최종적으로 완료됩니다.
    
    1번에서 얻어온 S3 객체의 키가 식별자이므로 이 키를 야구보구 백엔드로 넘겨주면 객체 키를 통해 백엔드 내부적으로 프로필 사진 업데이트 등의 비즈니스 로직을 처리합니다.

## 그리고 문제 발생

야구보구의 백엔드로부터 Pre-signed URL을 받아, 아래의 Retrofit 메서드를 사용해 전송을 시도했지만, 400 에러가 돌아왔습니다.

```kotlin
@PUT
suspend fun putProfileImageToS3(
    @Url url: String, // 야구보구 백엔드로부터 받은 Pre-signed URL
    @Body imageRequestBody: RequestBody // contentType(MIME 타입 문자열), contentLength(파일 크기), writeTo(파일 데이터) 가 담김
): Response<Unit>
```

## 에러 분석 과정

백엔드에서 보내주는 URL이 잘못됐는지, API 스펙이 틀렸는지 처음엔 의심했지만, 진짜 범인은 따로 존재했습니다.

### 실제 발생한 에러 로그

```bash
2025-11-04 17:35:24.119  okhttp.OkHttpClient  <-- 400 Bad Request 
[https://techcourse-project-2025.s3.ap-northeast-2.amazonaws.com/...] (376ms)

2025-11-04 17:35:24.121  okhttp.OkHttpClient  
<?xml version="1.0" encoding="UTF-8"?>
<Error>
  <Code>InvalidArgument</Code>
  <Message>Only one auth mechanism allowed; only the X-Amz-Algorithm query parameter, 
           Signature query string parameter or the Authorization header should be specified</Message>
  <ArgumentName>Authorization</ArgumentName>
  <ArgumentValue>Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...</ArgumentValue>
  <RequestId>1DZTTGMM54A549K3</RequestId>
</Error>
```

야구보구에서는 백엔드 API 호출 시 편의를 위해 모든 Retrofit 인스턴스에 아래와 같은 OkHttpClient를 **가로채서** 헤더를 추가해주는 tokenInterceptor를 추가하였고, 따라서 모든 API 요청에는 야구보구 백엔드 서버 인증용 Bearer 토큰이 부착되어 있습니다.

```kotlin
// RetrofitInstance.kt
private val tokenClient: OkHttpClient by lazy {
    OkHttpClient()
        .newBuilder()
        .addInterceptor(tokenInterceptor) // 모든 요청에 Bearer 토큰 자동 추가
        .addInterceptor(httpLoggingInterceptor)
        .build()
}
```

AWS로 요청을 보낼 때 Pre-signed URL 자체의 AWS 서명 정보와 더불어 불필요한 Authorization 헤더에 Bearer 토큰이 붙었던 것이 문제였습니다.

## 해결 과정

이 문제를 해결하기 위해 코드 리뷰를 거치며 여러 방법을 고민했고, 최종적으로 **별도 Retrofit 인스턴스** 방식을 채택했습니다.

| 방법 | 장점 | 단점 | 채택 |
| --- | --- | --- | --- |
| **URL별 인터셉터 제외** | 기존 구조 유지 | 복잡한 조건문, 유지보수성 저하 | ❌ |
| **순수 OkHttpClient 사용** | 직접 제어, 직관적 | 타입 안전성 손실, 코드 일관성 저하 | ❌ |
| **별도 Retrofit 인스턴스** | Retrofit 장점 유지, 명확한 분리 | 약간의 리소스 증가 | ✅ |

### 1단계 - 클라이언트 인스턴스 분리

코드 리뷰 과정에서 **OkHttpClient는 반드시 재사용해야 한다**는 피드백과 **Retrofit의 일관성을 유지하는 것이 좋겠다**는 의견을 받아 최종적으로 별도 Retrofit 인스턴스를 구성했습니다.

```kotlin
// RetrofitInstance.kt
class RetrofitInstance(
    baseUrl: String,
    tokenManager: TokenManager,
) {
    // Bearer 토큰이 없는 순수 클라이언트 (로깅만)
    val baseClient: OkHttpClient by lazy {
        OkHttpClient()
            .newBuilder()
            .addInterceptor(httpLoggingInterceptor) // 로깅만 유지
            .build()
    }

    // 야구보구 서버 API 요청용 Bearer 토큰 포함 클라이언트
    private val baseTokenClient: OkHttpClient by lazy {
        baseClient
            .newBuilder()
            .addInterceptor(tokenInterceptor) // 인증 토큰 자동 추가
            .authenticator(tokenAuthenticator)
            .readTimeout(30, TimeUnit.SECONDS)
            .build()
    }

    // ThirdParty 전용 Retrofit (토큰 없음)
    private val baseRetrofit: Retrofit by lazy {
        Retrofit.Builder()
            .baseUrl(baseUrl) // 더미 URL (실제로는 @Url이 오버라이드)
            .client(baseClient) // Bearer 토큰 없는 순수 클라이언트
            .addConverterFactory(json.asConverterFactory(MEDIA_TYPE.toMediaType()))
            .build()
    }

    // 야구보구 서버 API용 Retrofit (토큰 포함)
    private val baseTokenRetrofit: Retrofit by lazy {
        Retrofit.Builder()
            .baseUrl(baseUrl)
            .client(baseTokenClient) // Bearer 토큰 포함 클라이언트
            .addConverterFactory(json.asConverterFactory(MEDIA_TYPE.toMediaType()))
            .build()
    }

    // 서드파티 API 서비스 (S3, Google Drive 등)
    val thirdPartyApiService: ThirdPartyApiService by lazy {
        baseRetrofit.create(ThirdPartyApiService::class.java)
    }

    // 야구보구 내부 API 서비스들
    val memberApiService: MemberApiService by lazy {
        baseTokenRetrofit.create(MemberApiService::class.java)
    }
}
```

**핵심 설계 원칙**:
- `baseClient`: 로깅만 포함하는 순수 클라이언트 → **ThirdParty API용**
- `baseTokenClient`: Bearer 토큰 포함 클라이언트 → **야구보구 내부 API용**

### 2단계 - ThirdPartyApiService 정의

Bearer 토큰을 추가하지 않은 Retrofit인스턴스를 사용해 통신하는 API들을 정의해둔 ThirdPartyApiService를 작성합니다.
`@Url` 어노테이션을 사용해 완전한 URL이 제공되면 Retrofit 인스턴스의 baseUrl을 오버라이드합니다.

```kotlin
// ThirdPartyApiService.kt
interface ThirdPartyApiService {
    @PUT
    suspend fun putImageToS3(
        @Url url: String, // 전체 URL이 제공되면 baseUrl 무시됨
        @Body requestBody: RequestBody,
    ): Response<Unit>
}
```


### 3단계 - ThirdPartyDataSource 구현

2단계에서 정의한 s3로 이미지를 전송하는 ApiService를 호출해 실제 업로드 동작을 구현합니다.

```kotlin
// ThirdPartyDataSource.kt
class ThirdPartyDataSource(
    private val thirdPartyApiService: ThirdPartyApiService,
    private val contentResolver: ContentResolver,
) {
    suspend fun uploadImageToS3(
        url: String, // S3 Pre-signed URL
        imageFileUri: Uri,
        contentType: String,
        contentLength: Long,
    ): Result<Unit> = safeApiCall {
        val requestBody = createRequestBody(imageFileUri, contentType, contentLength)
        
        // 핵심: Bearer 토큰이 없는 Retrofit 인스턴스 사용
        thirdPartyApiService.putImageToS3(url, requestBody)
    }

    private fun createRequestBody(
        uri: Uri,
        contentType: String,
        contentLength: Long,
    ): RequestBody = object : RequestBody() {
        
        override fun contentType(): MediaType? = contentType.toMediaTypeOrNull()
        override fun contentLength(): Long = contentLength

        override fun writeTo(sink: BufferedSink) {
            contentResolver.openInputStream(uri)?.use { inputStream ->
                sink.writeAll(inputStream.source())
            }
        }
    }
}
```


## 그래서 왜 이 방식을 최종 선택했을까?

처음엔 Retrofit 인스턴스를 만들 경우 BaseUrl 지정을 강제하므로 이것을 우회하기 위해 Retrofit 대신 순수 OkHttpClient를 사용하는 방법을 선택했습니다.

```kotlin
// 순수 OkHttpClient 사용 시의 문제점
override suspend fun uploadProfileImage(...): Result<Unit> = 
    withContext(Dispatchers.IO) {
        runCatching {
            val request = Request.Builder()
                .url(url)
                .put(requestBody)
                .build()
                
            // 매번 새 클라이언트 생성(성능 문제 발생)
            OkHttpClient().newCall(request).execute().use { response ->
                // 수동 에러 처리
                if (!response.isSuccessful) {
                    throw Exception("Upload failed: ${response.code}")
                }
            }
        }
    }
```

하지만 코드 리뷰 과정에서 순수 OkHttpClient 사용의 아래와 같은 단점으로 인해 사용하지 않게 되었습니다.

**문제점들**

1. **타입 안전성 손실**: Retrofit의 어노테이션 기반 타입 체크 장점을 잃게 됨
2. **코드 일관성 저하**: 90% API는 Retrofit, 10%는 순수 OkHttp를 사용해 API 호출시 일관성 없음
3. **예외 처리 복잡화**: Retrofit의 통합된 에러 핸들링 패턴 활용 불가
4. **유지보수성 저하**: 개발자가 HTTP 요청/응답을 수동으로 처리해야 함
5. **Coroutine 통합 어려움**: `suspend` 함수의 자연스러운 흐름 방해

### 별도 Retrofit 인스턴스의 장점

이후 `@Url` 어노테이션의 존재를 알게 되어 최종적으로 Retrofit 인스턴스를 별도로 만들어두고 사용하는 방식을 선택했습니다.

```kotlin
// ThirdPartyApiService - Retrofit의 모든 장점 유지
interface ThirdPartyApiService {
    @PUT
    suspend fun putImageToS3(
        @Url url: String, // baseUrl 자동 오버라이드
        @Body requestBody: RequestBody,
    ): Response<Unit> // 타입 안전성 유지
}
```

**장점들**
- **타입 안전성**: `Response<Unit>` 반환으로 명확한 타입 체크
- **Coroutine 지원**: `suspend` 키워드로 자연스러운 비동기 처리
- **일관된 아키텍처**: 모든 네트워크 호출을 Retrofit으로 통일
- **공통 유틸 활용**: `safeApiCall` 등 기존 에러 처리 패턴 재사용
- **확장성**: 향후 다른 외부 서비스 연동 시에도 동일한 패턴 적용

따라서 **Pre-signed URL 등으로 서드파티 Base URL이 필요한 경우**, 현재 구성처럼 URL 자체를 `@Url` 어노테이션을 사용한 API 서비스를 활용하면서, baseUrl은 우리의 도메인을 그대로 Retrofit에 할당해주는 방식을 채택하였습니다.

***

## 프로필에 적합한 이미지 규격 만들기

Pre-signed URL을 통해 이미지를 업로드를 이용해 사용자가 보유한 원본 이미지를 바로 업로드 할 경우 정사각형 이미지가 아닌 경우가 있어 1:1 비율로 크롭하고 압축할 필요가 생겼습니다.

일부 초고화질 이미지의 경우 프로필 사진에 적합하지 않은 너무 큰 파일 크기를 가지기 때문에 백엔드에서 요구한 최대 5MB까지의 업로드 제약을 만족시키기 위해서 리사이징 및 압축이 필요했습니다.

더불어 사용자에게 실제 프로필 사진으로 사용될 영역을 미리 보여주게끔 하여, UX적으로도 완성도를 높이고 싶었습니다.

### uCrop을 활용한 이미지 크롭 및 압축

[uCrop](https://github.com/Yalantis/uCrop)은 안드로이드 앱 개발을 위한 이미지 자르기 라이브러리로서 프로필 사진에 사용할 이미지의 미리보기를 표시하면서 크롭하기 위해 선택한 라이브러리입니다.

아래와 같은 uCrop 액티비티 생성 코드에 파일 URI를 넘겨 액티비티를 실행하도록 하고 이미지를 크롭하는 역할을 위임합니다.

```kotlin
// SettingMainFragment.kt - uCrop 설정
private fun launchUCropActivity(sourceUri: Uri) {
    val destinationUri = Uri.fromFile(
        File(requireContext().cacheDir, "cropped_image_${System.currentTimeMillis()}.jpg")
    )

    val options = UCrop.Options().apply {
        setCircleDimmedLayer(true) // 원형 크롭 가이드
        setFreeStyleCropEnabled(false) // 정사각형 고정
        setToolbarColor(requireContext().getColor(R.color.primary500))
        // uCrop 내장 압축 설정
        setCompressionFormat(Bitmap.CompressFormat.JPEG)
        setCompressionQuality(85) // 85% 품질
    }

    val uCropIntent = UCrop
        .of(sourceUri, destinationUri)
        .withAspectRatio(1f, 1f) // 1:1 비율 강제
        .withMaxResultSize(1000, 1000) // 최대 해상도 제한
        .withOptions(options)
        .getIntent(requireContext())

    uCropLauncher.launch(uCropIntent)
}
```

코드를 통해 다음과 같은 크롭 UI 가이드를 만들 수 있습니다.

![야구보구 어플리케이션 화면 캡쳐, 적용한 프로필 이미지는 Perplexity 생성](https://github.com/user-attachments/assets/e474ceef-8e4c-44e5-b336-8c055df367c5)


### 이미지 처리 라이브러리 선택 과정

처음에는 [uCrop](https://github.com/Yalantis/uCrop)과 더불어 [Coil](https://coil-kt.github.io/coil/README-ko/) 이미지 라이브러리를 사용해 추가 압축을 진행하려 했습니다. 프로젝트 내에서 이미지 표시를 위해 이미 Coil을 도입해서 사용하고 있었고, uCrop에 비해 다양한 압축 옵션을 제공하기 때문이었습니다.

또한 uCrop은 2가지 버전을 제공하고 있어 실험을 통해 적합한 방법을 알아보기로 했습니다.

```gradle
implementation 'com.github.yalantis:ucrop:2.2.11' // 경량 일반 솔루션
implementation 'com.github.yalantis:ucrop:2.2.11-native' // 이미지 품질을 유지하기 위해 네이티브 코드 사용 (+ APK 크기에 약 1.5MB)
```

### 실험

실험을 위해 19.2MB 용량, 6200×6200픽셀 고화질의 허블 울트라 딥필드 이미지를 사용했습니다. [원본 이미지 열람](https://upload.wikimedia.org/wikipedia/commons/archive/2/2f/20081125003002%21Hubble_ultra_deep_field.jpg)

실험 방법은 아래의 3개 방식으로 이미지를 직접 크롭해서 비교해 보는 방법으로 진행했습니다:

- **uCrop + Coil**: uCrop을 사용해서 1000x1000 100% 품질로 크롭 뒤 Coil에서 500x500 85% 품질로 리샘플링
- **uCrop-native**: uCrop-native를 사용해서 500x500 85% 품질로 크롭
- **uCrop 단독**: uCrop을 사용해서 500x500 85% 품질로 크롭

실험을 진행한 결과를 요약하면 다음 이미지와 같습니다:

<img width="1520" height="500" alt="ucrop 비교" src="https://github.com/user-attachments/assets/aa55bc92-0cfc-4c40-92f5-a59dce53ed9b" />

**실험 결과**:

- **Coil3을 함께 사용**: 가장 이미지 크기가 작았지만 세부 디테일이 뭉개지는 이미지
- **uCrop 단독 사용**: 표준적인 이미지와 적당한 크기
- **uCrop-native 사용**: 가장 큰 크기의 이미지를 얻었고 디테일을 살리기 위해서라고 추측되는 약간의 **흐려짐** 현상을 확인

실험 결과, **uCrop 단독**을 선택했으며 이유는 다음과 같습니다:

**1. 성능 최적화**
- uCrop 단독 사용 시 처리 파이프라인 1회 (vs Coil 조합 2회)
- 업로드 속도 향상

**2. 코드 복잡도 감소**
- 단일 라이브러리로 의존성 최소화
- 유지보수 용이성 확보

**3. APK 크기 최적화**
- uCrop-native 대비 1.5MB 절감
- 이미지 품질 차이 미미

최종적으로 사용자 피드백을 바탕으로 프로필 이미지 변경 기능을 추가하면서, 원형 크롭 가이드를 제공하여 사용자에게 최종 결과를 미리 보여줄 수 있었고, 동시에 이미지 처리 부담을 클라이언트로 분산하여 백엔드 리소스를 절약하는 효과를 얻었습니다.

## 마치며

이번 구현에서 가장 큰 교훈은 세 가지였습니다:

1. **"모든 HTTP 요청이 동일한 인터셉터를 필요로 하지 않는다"**: Bearer 토큰이 모든 상황에서 자동으로 포함되어야 한다고 생각했지만, 외부 서비스(AWS S3)와 통신할 때는 오히려 방해 요소로 작용했습니다.

2. **"아키텍처 일관성과 성능 최적화의 균형점 찾기"**: 처음에는 순수 OkHttpClient로 해결하려 했지만, 코드 리뷰를 통해 **Retrofit의 장점을 포기하지 않으면서도 Bearer 토큰 충돌을 해결**하는 더 나은 방법을 찾았습니다.

4. **실험을 통한 트러블 슈팅의 중요성**: uCrop으로 이미지를 자른 뒤 coil로 후처리를 하며 이미지 품질이 단순히 좋을 것이라 생각했지만 실제 실험을 통해 적합한 방식을 선택해 나가는 접근법을 배웠습니다.

지금까지 프로필 이미지 업로드 과정의 트러블 슈팅과 성능 및 UX적 고민을 어떻게 풀어나갔는지의 과정에 그리고 여기서 얻은 교훈들에 대해 소개해 드렸습니다. 이 글이 동일한 문제와 고민을 겪고 있는 분들께 도움이 되었길 바랍니다.

***

### 참고 문헌 및 출처

- [AWS S3 Pre-signed URLs 공식 문서](https://docs.aws.amazon.com/AmazonS3/latest/userguide/PresignedUrlUploadObject.html)
- [OkHttp Interceptors 가이드](https://square.github.io/okhttp/features/interceptors/)
- [uCrop GitHub Repository](https://github.com/Yalantis/uCrop)
- [Coil Image Loading Library](https://coil-kt.github.io/coil/)
- [야구보구 GitHub Repository](https://github.com/woowacourse-teams/2025-yagu-bogu)

### 관련 커밋 히스토리

- [`33d3904`](https://github.com/woowacourse-teams/2025-yagu-bogu/commit/33d3904): uCrop 라이브러리 의존성 추가
- [`26e99a9`](https://github.com/woowacourse-teams/2025-yagu-bogu/commit/26e99a99024ca84a02de39ee5fb031458ce16cf9): 프로필 사진 변경 API 추가
- [`34bbd57`](https://github.com/woowacourse-teams/2025-yagu-bogu/commit/34bbd579ce4164d5a346feed851f6d52c76a2995): 프로필 사진 업로드 로직 구현 (초기 버전 - Bearer 토큰 충돌 발생)
- [`cb9a613`](https://github.com/woowacourse-teams/2025-yagu-bogu/commit/cb9a61350f77b686586e25018d81928550223fcd): uCrop 이미지 압축 로직 적용
- [`b28759d`](https://github.com/woowacourse-teams/2025-yagu-bogu/commit/b28759d2280fde765dc8956983a9762046ae0c08): **S3 이미지 업로드 로직 ThirdPartyRepository로 분리** (핵심 아키텍처 개선)
- [`fbd7cea`](https://github.com/woowacourse-teams/2025-yagu-bogu/commit/fbd7cea2416fb782350d3a628beebae041002945): **ThirdParty 요청에 토큰 헤더 제거** (Bearer 토큰 충돌 해결)
- [`09f2252`](https://github.com/woowacourse-teams/2025-yagu-bogu/commit/09f22528e8303f4c23b7719894372785e9d0d419): Pre-signed URL 관련 메서드명 변경 (최종 정리)
