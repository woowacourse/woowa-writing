# 테크니컬 라이팅

## TL;DR

야구보구 앱 프로필 사진 업로드 기능 개발 과정에서 발생한 **AWS S3 Presigned URL과 Retrofit Bearer 토큰 인터셉터 충돌 문제를 해결**하고, uCrop, Coil을 사용해 업로드할 이미지 형식을 최적화하고 UX를 개선한 기록입니다. 

## 대상 독자

- **Kotlin, Retrofit, OkHttp, Coil**등 안드로이드 네트워킹에 대한 기본적인 이해가 있는 **주니어 안드로이드 개발자**
- **관심사**: **Pre-signed URL을 받아** AWS S3 **파일 업로드** 기능 구현에 도전하고 싶은 개발자
- **HTTP 인터셉터와 외부 서비스 연동** 시 발생하는 인증 헤더 충돌 문제의 실무 해결 사례가 궁금한 개발자

## 문서 활용 계획

- **야구보구** 프로필 사진 업로드 기능 구현 과정에서 겪은 **403 Forbidden 에러와 해결 과정**을 상세히 기록하고 공유한다.
- 단순히 기능 구현을 넘어, **안정적인 서비스를 만들기 위한 팀의 고민과 성장**을 보여준다.
- **우아한테크코스 기술 블로그, 팀 기술 블로그, 개인 블로그** 및 외부 개발자 커뮤니티에 공유하며 유사한 고민을 하는 개발자들에게 **실질적인 도움**을 제공한다.

---

# 프로필 사진 S3 Pre-signed URL 업로드 삽질기

<img width="409" height="123" alt="Image" src="https://github.com/user-attachments/assets/b7d44291-526e-4337-b08d-25b8bae00ade" />

[**야구보구**](https://play.google.com/store/apps/details?id=com.yagubogu&hl=ko)는 우아한테크코스에서 개발 중인 야구 팬 커뮤니티 앱입니다. 프로야구 경기장에서 버튼 한 번으로 직관 인증과 기록이 가능하고, 팬들끼리 소통할 수 있는 플랫폼을 제공합니다.

개발 당시 구글 SSO 로그인으로 간편 가입을 도입함과 더불어 구글 프로필 이미지를 그대로 사용하여 프로필을 표시했으나, "**프로필 사진을 바꾸고 싶어요!**"라는 사용자 피드백이 들어오면서 프로필 사진 업로드 기능을 추가하게 되었습니다.

사진을 업로드하는 단순해 보이는 이 기능이 어떤 기술적 도전을 가져다주었고 어떤 방식으로 해결했을지 궁금하다면 계속 읽어주세요.


## 그래서 AWS S3 Pre-signed URL이 뭔데?

![일회용 마그네틱 승차권](https://github.com/user-attachments/assets/54c1ae87-6095-4807-a362-00c036b3118c)

티켓 가격, 목적지가 정해져 있는 옛날 지하철 마그네틱 승차권과 비슷하다. 
발급받은 URL로 특정 용량의 파일을 단 한번만 전송이 가능하다.

**Pre-signed URL의 특성**:

- URL 파라미터에 인증 정보(`X-Amz-Signature`, `X-Amz-Credential` 등) 포함
- Content-Type, Content-Length도 서명에 영향을 미칠 수 있음

```
<https://s3.amazonaws.com/bucket/key>?
X-Amz-Algorithm=AWS4-HMAC-SHA256&
X-Amz-Credential=AKIAIOSFODNN7EXAMPLE%2F20231013%2Fus-east-1%2Fs3%2Faws4_request&
X-Amz-Date=20231013T000000Z&
X-Amz-Expires=3600&
X-Amz-Signature=abc123...
```


## 그래서 왜 Pre-signed URL을 쓰는건데?

S3 버킷은 Amazon S3 스토리지 서비스에서 파일을 저장하는 하나의컨테이너입니다. 
야구보구에서는 운영 서버 부하 를 줄이기 위해 . 클라이언트에서 S3에 직접 업로드 할 필요성이 생겼습니다.

일반적인 파일 업로드 방식이라면:
1. 클라이언트 → 서버로 이미지 전송
2. 서버에서 이미지 검증 및 처리  
3. 서버 → S3로 최종 업로드

하지만 Pre-signed URL을 사용하면:
1. 서버에서 임시 업로드 URL만 생성
2. **클라이언트가 S3에 직접 업로드** (서버 경유 없음)
3. 완료 후 서버에 결과만 통지

즉 퍼블릭 액세스가 불가능한 S3 버킷에 직접 접근하기 위한 일회성 URL인 Pre-signed URL인 것입니다.
따라서 야구보구의 운영 서버는 단순히 URL 생성과 완료 확인 역할만 하게 되어 리소스 절약과 업로드 속도 향상의 효과를 얻을 수 있습니다.


## 잘 모르겠고 업로드부터 해 보자

우선 아구보구의 ~~전설은 아니고 레전드인~~ 백엔드 개발자 포라가 `Pre-signed url 조회`와 `Pre-signed url 업로드 확인` 이라는 두개의 API를 만들어 주셨고 아래와 같은 과정을 통해 업로드 기능을 구현하기 시작했습니다.

### API를 통해 사진을 업로드하는 과정

1. 야구보구 서버에서 AWS의 Pre-signed url을 받아오기
    
    업로드할 파일의 MIME 타입, 업로드할 파일 크기(바이트)를  `Request Body`로 알려주면 다음과 같은 응답이 도착한다 key는 S3의 객체의 고유한 식별자 키이며, url은 Pre-Signed URL 입니다.
    
    ```kotlin
    {
      "key": "yagubogu/images/profile/sample.jpg",
      "url": "https://yagubogu-2025.s3.ap-northeast-2.amazonaws.com/yagubogu/images/profile/sample.jpg?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=...&X-Amz-Expires=600&X-Amz-SignedHeaders=host&X-Amz-Signature=..."
    }
    ```
    
2. AWS S3 서버로 이미지 전송
    
    1번에서 받은 Pre-Signed URL로 HTTP PUT 메서드룰 사용해 백엔드 서버를 거치지 않고 AWS로 이미지를 직접 전송해주어야 합니다.
    
    URL로 보낼 이미지 데이터는 `RequestBody`에 담아 보내는데, `contentType(MIME 타입 문자열)`, `contentLength(파일 크기)`, `writeTo(파일 데이터)`를 같이 담아 보냅니다.
    
    - MIME 타입이란?
        
        문서나 파일의 종류와 형식을 나타내는 식별자로서 `image/jpeg` , `text/html` , `video/mp4` 등 서버가 데이터를 어떻게 처리해야 할지 알려주기 위한 꼬리표 같은 녀석입니다.
        
3. 야구보구 서버에 성공했다고 알려주기
    
    AWS S3 버킷에 파일 전송이 성공(200) 했다면 성공했다는 사실을 우리 앱의 백엔드 서버에 알려줘야 업로드가 최종적으로 완료됩니다.
    
   1번에서 얻어온 S3 객체의 키가 식별자이므로 이 키를 야구보구 백엔드로 넘겨주면 객체 키를 통해 백엔드에서 실제 프로필 사진 업데이트 등을 처리합니다. 
    

## 그리고 문제 발생

야구보구의 백엔드로부터 Pre-signed URL을 받아, 아래의 Retrofit 메서드를 사용해 전송을 시도했지만, 403 에러가 돌아왔습니다.

```kotlin
    @PUT
    suspend fun putProfileImageToS3(
        @Url url: String, // 야구보구 백엔드로부터 받은 Pre-signed URL
        @Body imageRequestBody: RequestBody // contentType(MIME 타입 문자열), contentLength(파일 크기), writeTo(파일 데이터) 가 담김
    ): Response<Unit>
```

<img width="408" height="274" alt="Image" src="https://github.com/user-attachments/assets/7bfe2963-3bae-4ee6-a1e7-ffdc81c6ef29" />

## 에러 분석 과정

백엔드에서 보내주는 URL이 잘못됐는지, API 스펙이 틀렸는지 처음엔 의하했지만, 진짜 범인은 따로 존재했습니다.

> 이곳엔 진짜 403에러 로그나 캡쳐 이미지를 넣을 예정입니다

> ```
> S3 Upload failed: 403
> ```

403, 즉 Forbidden에러는 서버의 권한이 존재하지 않는다는 소리이며 인증이 잘못 되었다는 것을 알 수 있는데
Presigned Url 특성상 URL자체로 1회 인증이 가증했어야 합니다. 그렇다면 무엇이 인증을 방해했던 걸까요?

야구보구에서는 백엔드의 API 호출시 편의를 위해 모든 Retrofit 인스턴스에 아래와 같은 OkHttpClient를 낚아채서 헤더를 추가해주는 tokenInterceptor를 추가하였고 따라서 모든 API 요청에는 야구보구 백엔드 서버 인증용 Bearer 토큰이 부착되어 있습니다.

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

AWS에게 아래와 같이 토큰을 함께 헤더에 넣어 요청하게 되었고, 잘못된 인증 헤더를 읽은 AWS에서 업로드를 거부한 것입니다.

```kotlin
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9... // 문제의 헤더!
Content-Type: image/jpeg
Content-Length: 245678
```


## 해결 과정

이 문제를 해결하기 위해 아래의 3가지 방법을 고민했고 여러 장단점을 고려해 순수 OkHttpClient를 사용하기로 하였습니다.

| 방법 | 장점 | 단점 | 채택 여부 |
| --- | --- | --- | --- |
| **URL별 인터셉터 제외** | 기존 구조 유지 | 복잡한 조건문, 유지보수성 저하 | ❌ |
| **별도 Retrofit 인스턴스** | 명확한 분리 | 리소스 중복, 과도한 오버헤드 | ❌ |
| **순수 OkHttpClient 분리** | 간단명확, 리소스 효율적 | 일부 코드 중복 | ✅ |

- **명확한 책임 분리**: 내부/외부 서버 API 용도별 클라이언트 구분
- **유지보수성**: 단순하고 이해하기 쉬운 구조
- **확장성**: 향후 다른 외부 서비스 연동 시에도 순수한 Client 적용 가능

순수 OkHttpClient를 사용한 자세한 해결 방법을 알아보도록 하겠습니다.

## 1단계  클라이언트 인스턴스 분리
이미 만들어둔 RetrofitInstance에 로깅만 하고 있던 순수한 loggingClient가 이미 존재하고 있었고 이를 재활용했습니다.

처음에는 재활요하지 않고 요청시마다 매번 OkHttpClient 클라이언트를 생성했었는데 클라이언트는 자체적인 연결 풀과 스레드 풀을 가지고 있기 떄문에 반드시 재사용 해야 한다는 리뷰을 받았고, 직접 인스턴스를 만들지 않고 의존성을 주입받아 사용하도록 했습니다.

```kotlin
// RetrofitInstance.kt
class RetrofitInstance(
    baseUrl: String,
    tokenManager: TokenManager,
) {
    // Bearer 토큰이 없지만 디버그시 로깅은 작동하는 순수 클라이언트
    val loggingClient: OkHttpClient by lazy {
        OkHttpClient()
            .newBuilder()
            .addInterceptor(httpLoggingInterceptor) // 로깅만 유지
            .build()
    }

    // 야구보구 서버 API 요청용 Bearer 토큰 포함 클라이언트
    private val tokenClient: OkHttpClient by lazy {
        OkHttpClient()
            .newBuilder()
            .addInterceptor(tokenInterceptor) // 인증 토큰 자동 추가
            .addInterceptor(httpLoggingInterceptor)
            .build()
    }
}

```

- `loggingClient`: 로깅 인터셉터만 포함하는 순수 OkHttpClient로 S3 업로드시 사용
- `tokenClient`: 기존대로 Bearer 토큰 포함, 백엔드와 통신하는 일반 API 요청용으로서 Retrofit 적용
- 단일 `OkHttpClient` 인스턴스 재사용으로 연결 풀 최적화

---

## 2단계 - S3 업로드 로직 구현

### MemberRemoteDataSource 수정

```kotlin
// MemberRemoteDataSource.kt
class MemberRemoteDataSource(
    private val context: Context,
    private val memberApiService: MemberApiService,
    private val pureClient: OkHttpClient, // S3 전용 클라이언트
) : MemberDataSource {

    override suspend fun uploadProfileImage(
        url: String, // S3 Pre-signed URL
        imageFileUri: Uri,
        contentType: String,
        contentLength: Long,
    ): Result<Unit> = withContext(Dispatchers.IO) {
        runCatching {
            val requestBody = createRequestBody(imageFileUri, contentType, contentLength)

            val request = Request.Builder()
                .url(url) // Pre-signed URL 사용
                .put(requestBody) // http PUT 메서드로 이미지 데이터 전송
                .build()

            // 핵심: Bearer 토큰 없는 순수 클라이언트 사용
            pureClient.newCall(request).execute().use { response ->
                if (!response.isSuccessful) {
                    val errorBody = response.body?.string()
                    Timber.e("S3 Upload failed: ${response.code}")
                    Timber.e("S3 Error body: $errorBody")
                    throw Exception("Upload failed: ${response.code}")
                }
            }
        }.onFailure { e ->
            Timber.e(e, "S3 Upload exception")
        }
    }
}

```

- `pureClient.newCall()`: Bearer 토큰 인터셉터를 거치지 않음
- `runCatching`: 간결한 예외 처리 (코드 리뷰 반영)
- `use` 블록: 리소스 자동 해제로 메모리 누수 방지

## RequestBody 생성 로직

URI란 업로드할 파일에 접근 가능한 내부 주소로서, Context를 사용해 실제 파일의 정보를 가져옵니다.

```kotlin
// URI를 RequestBody로 변환
private fun createRequestBody(
    uri: Uri,
    contentType: String,
    contentLength: Long,
): RequestBody = object : RequestBody() {

    override fun contentType(): MediaType? =
        contentType.toMediaTypeOrNull()

    override fun contentLength(): Long = contentLength

    override fun writeTo(sink: BufferedSink) {
        context.contentResolver.openInputStream(uri)?.use { inputStream ->
            sink.writeAll(inputStream.source())
        }
    }
}
```


### contentLength(파일 크기) 측정 로직

```kotlin
// URI에서 정확한 파일 크기 추출
fun Uri.fileSize(context: Context): Result<Long?> = runCatching {
    context.contentResolver // ContentResolver의 query()로 SIZE 컬럼 조회
        .query(this, arrayOf(OpenableColumns.SIZE), null, null, null)
        ?.use { cursor ->
            if (cursor.moveToFirst()) {
                val idx = cursor.getColumnIndexOrThrow(OpenableColumns.SIZE)
                cursor.getLongOrNull(idx)  // SIZE 컬럼 값 반환
            } else null
        } ?: context.contentResolver // 실패시 FileDescriptor의 statSize 사용 (fallback)
            .openFileDescriptor(this, "r")
            ?.use { pfd -> pfd.statSize.takeIf { it >= 0 } } // 유효한 크기만 반환
}
```

---

이로서 이미지 업로드 자체의 문제는 해결하였습니다!!


## 서버비는 조상님이 내주시냐?

단순히 프로필 사진을 표시할 용도의 이미지가 용량이 큰 원본 이미지를 사용할 필요가 없기 때문에 압축할 필요성이 있었고 최대 5MB까지의 이미지만 업로드 가능하다는 S3에 제약이 걸려 있었기 때문에 용량을 줄일 필요성이 생겼습니다.
더불어 사용자에게 이미지에서 실제 프로필 사진의 영역을 미리 보여주게끔 하여, UX적으로도 완성도를 높이고 싶었습니다!

### uCrop을 활용한 이미지 크롭

[uCrop](https://github.com/Yalantis/uCrop)은 안드로이드 앱 개발을 위한 이미지 자르기 라이브러리로서 프로필 사진에 사용할 이미지의 미리보기를 표시하면서 크롭하기 위해 선택한 라이브러리 입니다.

아래와 같은 uCrop액티비티 생성 코드에 파일 URI를 넘겨 액티비티를 실행하도록 하고 이미지를 크롭하는 역할을 위임합니다.

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
코드를 통해 아래와 같은 크롭 UI 가이드를 만들 수 있습니다.

<img width="972" height="727" alt="Image" src="https://github.com/user-attachments/assets/b2d0ba3b-b0ee-47c0-8b69-46a9df1d3a74" />

### Coil을 활용한 이미지 압축

uCrop 단독으로는 크롭한 이미지를 바로 사용하지 않고, [Coil](https://coil-kt.github.io/coil/README-ko/) 이미지 라이브러리를 사용해 압축을 진행하였습니다. 
프로젝트 내에서 이미지 표시 이미 Coil을 도입해서 사용되고 있었고, uCrop에 비해 효율적인 압축 알고리즘으로 용량을 절약할 수 있었기에 uCrop 단독으르 크롭한 이미지의 후처리를 진행 하였습니다.

```kotlin
// ImageUtils.kt - Coil 압축 유틸리티
object ImageUtils {
    suspend fun compressImageWithCoil(
        context: Context,
        uri: Uri,
        maxSize: Int = 500, // 500x500 픽셀로 리사이즈
        quality: Int = 85   // JPEG 품질 85%
    ): Uri? = withContext(Dispatchers.IO) {
        try {
            val imageLoader = ImageLoader(context)
            val request = ImageRequest.Builder(context)
                .data(uri)
                .size(maxSize, maxSize) // 균등 리사이즈
                .allowHardware(false)   // 소프트웨어 비트맵 강제
                .build()

            val bitmap = imageLoader.execute(request)
                .image?.toBitmap() ?: return@withContext null

            val outputFile = File(
                context.cacheDir,
                "compressed_${System.currentTimeMillis()}.jpeg"
            )

            outputFile.outputStream().use { out ->
                bitmap.compress(Bitmap.CompressFormat.JPEG, quality, out)
            }

            Uri.fromFile(outputFile)
        } catch (e: Exception) {
            Timber.e(e, "이미지 압축 실패")
            null
        }
    }
}

```

# 구현을 통한 개선점 및 마무리

**이미지 처리 성능**:

- **원본 이미지**: 평균 2-5MB (유저가 선택하는 용량에 따라 천차 만별)
- **압축 후 이미지**: 평균 200-500KB (약 80% 크기 감소, 정사각형 보장)
- **처리 시간**: 1~2초 (크롭 + 압축 + 업로드 전체 과정)

**기술적 성과**:

- Pre-signed URL을 사용하여 백엔드를 거치치 않고 Amazon S3로 직접 업로드
- **80% 이미지 크기 감소** 효과 (평균 2-5MB → 200-500KB)
- **일관된 UX** 제공 (1:1 원형 크롭 가이드)


사용자 피드백을 바탕으로 프로필 이미지 변경 기능을 추가하면서, 단순히 백엔드로 원본 이미지를 전송하는 방식도 고려했습니다. 하지만 클라이언트에서 원형 크롭 가이드를 제공함으로써 사용자에게 최종 결과를 미리 보여줄 수 있었고, 동시에 이미지 처리 부담을 클라이언트로 분산하여 백엔드 리소스를 절약하는 효과를 얻었습니다.

이번 구현에서 가장 큰 교훈은 **"모든 HTTP 요청이 동일한 인터셉터를 필요로 하지 않는다"**는 점이었습니다. 처음에는 Bearer 토큰이 모든 상황에서 자동으로 포함되어야 한다고 생각했지만, 외부 서비스(AWS S3)와 통신할 때는 오히려 방해 요소로 작용하였습니다. 외부 API 연동 시에는 해당 서비스의 인증 방식을 정확히 파악하고, 상황에 맞는 HTTP 클라이언트를 선택하는 것이 핵심임을 배웠습니다.

---

## 부록

### 용어 정리

- **Pre-signed URL**: AWS S3에서 제공하는 임시 업로드/다운로드 URL로, URL 자체에 인증 정보가 포함됨
- **Bearer Token**: API 인증에 사용되는 토큰으로, HTTP Authorization 헤더에 `Bearer {token}` 형태로 포함
- **Interceptor**: HTTP 요청/응답을 가로채어 공통 작업(로깅, 인증 등)을 수행하는 OkHttp 컴포넌트
- **uCrop**: Android용 이미지 크롭 라이브러리, 다양한 크롭 옵션과 UI 커스터마이징 제공
- **Coil**: Kotlin 기반 Android 이미지 로딩 라이브러리, 비동기 처리와 메모리 효율성에 특화

### 참고 문헌 및 출처

- [AWS S3 Pre-signed URLs 공식 문서](https://docs.aws.amazon.com/AmazonS3/latest/userguide/PresignedUrlUploadObject.html)
- [OkHttp Interceptors 가이드](https://square.github.io/okhttp/features/interceptors/)
- [uCrop GitHub Repository](https://github.com/Yalantis/uCrop)
- [Coil Image Loading Library](https://coil-kt.github.io/coil/)
- [야구보구 GitHub Repository](https://github.com/woowacourse-teams/2025-yagu-bogu)

### 관련 커밋 히스토리

- [`33d3904`](https://github.com/woowacourse-teams/2025-yagu-bogu/commit/33d3904): uCrop 라이브러리 의존성 추가
- [`26e99a9`](https://github.com/woowacourse-teams/2025-yagu-bogu/commit/26e99a99024ca84a02de39ee5fb031458ce16cf9): 프로필 사진 변경 API 추가
- [`b1531ce`](https://github.com/woowacourse-teams/2025-yagu-bogu/commit/b1531ce3942c88261700905381973bff32891b5f): 설정 화면에 프로필 사진 crop 및 압축 기능 추가
- [`34bbd57`](https://github.com/woowacourse-teams/2025-yagu-bogu/commit/34bbd579ce4164d5a346feed851f6d52c76a2995): 프로필 사진 업로드 로직 구현 (초기 버전)
- [`b8fec65`](https://github.com/woowacourse-teams/2025-yagu-bogu/commit/b8fec65676c25d0bdfb19123cc128de90a782787): **프로필 사진 변경을 위한 OkHttpClient 의존성 분리** (핵심 해결)
- [`24966d3`](https://github.com/woowacourse-teams/2025-yagu-bogu/commit/24966d333fc1cb7672b0350ffeab58430a51b5cf): S3 이미지 업로드 로직 개선 (runCatching 적용)
- [`482f7f6`](https://github.com/woowacourse-teams/2025-yagu-bogu/commit/482f7f65ec5a2b83e13d157a89c65f7641f3a427): PureInstance 클래스 삭제 및 RetrofitInstance로 통합 (최종 최적화)

---
