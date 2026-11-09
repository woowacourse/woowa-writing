# Android와 KMP 프로젝트에서 DI 도구 선택하기

## 이 글의 목적과 대상 독자

이 글은 Kotlin 문법과 Android의 `ViewModel`, `Repository` 역할을 아는 개발자를 위한 DI 도구 선택 안내서다. DI 라이브러리 경험은 없어도 된다. 의존성 주입의 기본 원리부터 설명하고, 수동 DI, Hilt, Koin, Metro를 같은 객체 그래프에 적용하는 예를 통해 Android와 KMP에서의 선택 기준을 정리한다.

이 글의 목적은 한 라이브러리를 정답으로 정하는 데 있지 않다. 프로젝트의 플랫폼 범위, 공통 코드 요구, 빌드 환경, 팀의 경험에 맞는 도구를 고를 때 확인할 기준을 제시한다. 예제는 `Repository`와 `ViewModel`의 연결로 한정한다. 실제 설정은 프로젝트가 사용하는 Kotlin, Android Gradle Plugin, DI 도구 버전에 맞춰 공식 문서에서 확인한다.

## 1. 문제: DI 도구 선택은 Android 화면 밖으로 이어진다

화면 계층과 데이터 계층을 분리한 Android 앱에서는 객체 사이에 의존 관계가 생긴다. 예를 들어 `ProfileViewModel`은 `ProfileRepository`를 사용하고, `DefaultProfileRepository`는 네트워크 API에 의존한다. 각 객체가 필요한 구현을 직접 만들면 생성 코드가 여러 곳에 퍼져 테스트에서 가짜 구현을 넣기 어렵다. 의존성 주입(Dependency Injection, DI)은 객체가 사용할 의존성을 외부에서 받아 객체 생성과 사용을 나누는 방식이다.

Android 전용 프로젝트라면 Hilt가 익숙한 선택지다. 공통 로직을 `commonMain`으로 옮기는 KMP 프로젝트에서는 Android 프레임워크와 연결된 구성을 공통 코드에서 사용할 수 있는지 확인해야 한다. Android 화면에만 DI를 적용할지, 공통 모듈에서도 객체 그래프를 구성할지, 플랫폼별 구현을 어디서 연결할지에 따라 선택이 달라진다.

Hilt는 Android 구성 요소와 통합되지만, 같은 구성을 공통 소스셋에서도 사용할 수 있다고 전제해서는 안 된다. Koin은 Kotlin DSL이나 애노테이션으로 객체를 정의하고, Metro는 컴파일러 플러그인으로 그래프를 생성한다. 각 도구의 기능과 KMP 지원 범위는 버전에 따라 바뀔 수 있으므로, 아래 예제는 선택 기준을 설명하는 출발점으로 삼고 실제 도입 전 공식 문서와 프로젝트 빌드로 확인한다.

### DI를 적용하지 않았을 때

다음 예제는 화면 모델이 `Repository` 구현과 API를 직접 생성한다.

```kotlin
class ProfileViewModel : ViewModel() {
    private val repository = DefaultProfileRepository(
        api = ProfileApi(HttpClient())
    )
}
```

작은 예제에서는 간단하지만, `ProfileViewModel`이 구현 선택과 생성까지 맡는다. 테스트에서 가짜 `Repository`를 전달하려면 생성 경로를 바꿔야 한다. API 설정이 바뀌면 여러 화면의 생성 코드도 함께 수정해야 할 수 있다.

### 의존성을 밖에서 전달하기

생성자를 통해 필요한 계약을 받으면 사용 책임과 생성 책임을 분리할 수 있다.

```kotlin
interface ProfileRepository {
    suspend fun loadProfile(): Profile
}

class ProfileViewModel(
    private val repository: ProfileRepository
) : ViewModel()
```

`ProfileViewModel`은 구체 구현을 알지 못한다. 테스트는 `FakeProfileRepository`를, 앱은 실제 구현을 전달한다. DI 도구는 이 객체들을 생성하고 연결하는 방법을 관리한다. 생성자에 의존성을 전달하는 일과 DI 프레임워크를 도입하는 일은 다르다. 객체 그래프가 작다면 수동 생성만으로 충분할 수 있다.

## 2. 비교한 방법: 수동 DI, Hilt, Koin, Metro

비교 예제는 `ProfileViewModel → ProfileRepository → ProfileApi` 객체 그래프다. 각 방식에서 객체를 어디서 선언하는지, 누락된 의존성을 언제 찾는지, 플랫폼별 경계를 어떻게 나누는지 살펴본다.

| 기준 | 수동 DI | Hilt | Koin | Metro |
|---|---|---|---|---|
| 의존성 선언 | 일반 Kotlin 생성 코드 | 애노테이션과 생성 규칙 | 모듈 DSL 등 Kotlin 선언 | 컴파일러가 이해하는 선언과 생성 규칙 |
| 그래프 구성 | 개발자가 직접 호출 순서를 관리 | 컴파일러 플러그인과 Android 통합 활용 | 런타임에 모듈을 등록하고 조회하는 구성 사용 | 컴파일 시 그래프 생성 및 검증을 목표로 함 |
| 플랫폼 관점 | 각 플랫폼 진입점에서 직접 조립 | Android 통합이 중심이므로 KMP 공통 사용 여부를 별도 확인 | KMP 지원 모듈과 플랫폼별 선언 범위를 버전별 확인 | KMP 지원 범위와 안정성을 공식 문서에서 확인 |
| 장점 | 단순하고 동작이 명시적 | Android 생태계와 통합된 기능 및 익숙한 패턴 | Kotlin 코드 중심의 선언, 비교적 유연한 구성 | 생성 코드 중심으로 그래프 오류를 일찍 발견할 가능성 |
| 부담 | 객체 수가 늘면 수동 연결 코드가 커짐 | 생성 규칙과 플러그인, Android 개념 학습 필요 | 런타임 조회와 모듈 구성, 실행 중 오류 확인 필요 | 컴파일러 플러그인, 설정 및 성숙도 검토 필요 |

표는 성능 순위가 아니라 비교 기준을 요약한다. 프로젝트 규모와 팀 경험에 따라 선택은 달라진다. “컴파일 시 검증”과 “KMP 지원”의 범위는 도구 버전과 구성에 따라 다르므로 실제 적용 버전에서 확인한다.

### 수동 DI: 코드로 그래프를 직접 조립하기

수동 DI는 프레임워크 없이 생성자를 호출해 객체 그래프를 만든다. Android에서는 `Application`이나 화면 생성 지점에서 객체를 조립해 `ViewModel`에 전달할 수 있다.

```kotlin
class ProfileGraph(
    api: ProfileApi,
) {
    val profileRepository: ProfileRepository =
        DefaultProfileRepository(api)
}

class ProfileViewModelFactory(
    private val repository: ProfileRepository,
) : ViewModelProvider.Factory {
    @Suppress("UNCHECKED_CAST")
    override fun <T : ViewModel> create(modelClass: Class<T>): T {
        if (modelClass.isAssignableFrom(ProfileViewModel::class.java)) {
            return ProfileViewModel(repository) as T
        }
        error("지원하지 않는 ViewModel: ${modelClass.name}")
    }
}
```

이 코드는 Android 화면 모델을 위한 부분 예제다. `ViewModelProvider.Factory`의 구현은 프로젝트의 AndroidX 버전에 맞춘다. `ProfileRepository` 계약과 플랫폼 독립 구현은 `commonMain`에 둘 수 있지만, AndroidX의 `ViewModelProvider`를 사용하는 코드는 Android 소스셋에 둔다.

수동 구성은 생성 흐름이 코드에 드러나고 별도 플러그인이나 컨테이너가 필요 없다. 작은 객체 그래프에서 구조를 익히기 좋다. 객체가 늘면 생성 코드와 중복이 증가하고 범위 관리도 직접 해야 한다. 따라서 그래프가 작거나 프레임워크 없이 시작하려는 프로젝트에 적합하다.

### Hilt: Android 애플리케이션 구성 요소와 연결하기

Hilt는 Dagger 기반 Android DI 도구다. 애노테이션으로 객체 생성 규칙과 Android 구성 요소별 범위를 선언하면 빌드 과정에서 Dagger 구성 코드를 생성한다. 앱 진입점에는 `@HiltAndroidApp`, 주입 대상 Android 클래스에는 `@AndroidEntryPoint`, 모듈에는 `@Module`과 `@InstallIn`을 사용한다.

```kotlin
@Module
@InstallIn(SingletonComponent::class)
abstract class ProfileModule {
    @Binds
    abstract fun bindProfileRepository(
        implementation: DefaultProfileRepository,
    ): ProfileRepository
}

class DefaultProfileRepository @Inject constructor(
    private val api: ProfileApi,
) : ProfileRepository {
    override suspend fun loadProfile(): Profile = api.loadProfile()
}
```

예제에서 `ProfileApi`를 제공하는 방법과 바인딩 범위는 생략했다. Hilt를 실제 적용할 때는 이를 모듈에 선언하고, `ViewModel`에는 Hilt의 ViewModel 바인딩 규칙을 적용한다. Hilt는 Android 진입점과 생명주기에 맞춘 표준 구성 요소를 제공한다. KMP에서는 공통 계약과 Android 전용 Hilt 구성을 분리하거나, 공통 코드에서 사용할 DI 도구를 별도로 검토한다.

### Koin: Kotlin 선언으로 모듈 구성하기

Koin은 모듈에 객체 정의를 작성하고 앱 시작 시 등록한다. 다음은 기본 DSL로 객체 그래프를 선언하는 예다.

```kotlin
val profileModule = module {
    single<ProfileRepository> { DefaultProfileRepository(get()) }
    factory { ProfileViewModel(get()) }
}
```

예제는 앞 절의 `ProfileRepository`와 `DefaultProfileRepository` 선언을 사용한다. KMP에서는 `commonMain`에 공통 `initKoin()` 함수를 두고 각 플랫폼 시작 지점에서 호출할 수 있다. Android 전용 ViewModel 통합이나 의존성 artifact는 사용하는 Koin 버전의 공식 설정을 확인한다.

Koin은 Kotlin DSL이나 애노테이션으로 객체를 선언한다. 기본 DSL은 런타임에 객체를 연결한다. 선택형 Compiler Plugin은 빌드 시 그래프를 검증한다. 비교할 때는 사용한 방식과 플러그인 적용 여부를 기록하고 누락 바인딩의 발견 시점, 테스트 검증, 객체 범위를 확인한다.

### Metro: 컴파일 시점 생성 접근 검토하기

Metro는 Kotlin 컴파일러 플러그인으로 의존성 그래프를 생성하는 멀티플랫폼 DI 도구다. 선언한 바인딩을 컴파일하며 검증하고 생성 코드를 만든다. 프로젝트의 Kotlin 버전과 타깃이 지원되는지 확인한다.

Metro에서는 그래프 인터페이스에 `@DependencyGraph`를 붙이고 프로퍼티나 `@Provides` 함수로 바인딩을 선언한다. 생성자에는 `@Inject`를 사용하고 `createGraph<GraphType>()`로 그래프를 만든다. 다음은 앞 절의 예제를 적용한 부분 코드다. Metro Gradle 플러그인과 Kotlin 버전은 [공식 호환성 문서](https://zacsweers.github.io/metro/compatibility/)에 맞춘다.

```kotlin
@DependencyGraph
interface ProfileGraph {
    val profileRepository: ProfileRepository

    @Provides
    fun provideProfileApi(): ProfileApi = ProfileApi()
}

@Inject
class DefaultProfileRepository(
    private val api: ProfileApi,
) : ProfileRepository {
    override suspend fun loadProfile(): Profile = api.loadProfile()
}

val graph = createGraph<ProfileGraph>()
val repository = graph.profileRepository
```

Metro 공식 저장소는 그래프 오류를 컴파일 시점에 검출하고 JVM, Android, JS, Wasm, Native 타깃을 지원한다고 안내한다. 프로젝트 버전의 호환성 표와 실제 빌드로 지원 여부를 확인한다. 컴파일러 플러그인과 빌드 도구를 함께 관리해야 하는 점도 고려한다.

## 3. 해 본 일: 동일한 객체 그래프를 KMP 예제에 적용하기

도구를 공정하게 비교하려면 같은 객체 그래프와 요구사항을 사용한다. 예제는 `ProfileRepository`, `DefaultProfileRepository`, `ProfileApi`, `ProfileViewModel`로 구성한다. 플랫폼 독립 계약과 구현은 공통 코드에, Android 프레임워크에 의존하는 연결 코드는 Android 소스셋에 둔다. 각 도구의 공식 KMP 예제를 참고해 수동 DI, Hilt, Koin, Metro를 구성한다.

실습에서는 KMP 예제에 Metro를 적용하고 Android Hilt 구성과 비교한다. 같은 `Repository` 흐름을 사용하고 도구 버전, Gradle 플러그인 설정, 소스셋 경계를 기록한다. Koin도 같은 그래프로 구성해 공통 모듈과 플랫폼별 시작 방식의 차이를 살펴본다. 아래 표는 실험을 마친 뒤 실제 결과로 채운다.

실험 기록에는 다음 항목을 남긴다.

1. Kotlin, Kotlin Multiplatform 플러그인, Android Gradle Plugin, DI 도구의 버전
2. 빌드한 타깃과 실행 환경
3. `Repository` 바인딩과 `ViewModel` 생성 코드가 위치한 소스셋
4. 의존성 누락 또는 중복 바인딩을 의도적으로 만들었을 때 오류가 발견되는 시점
5. Android와 공통 모듈 빌드 성공 여부 및 오류 메시지
6. 설정, 빌드, 디버깅에 걸린 시간과 반복 횟수

각 도구에 같은 기능을 적용하고 공식 샘플과 의존성 선언을 확인한 뒤 빌드한다. 실패하면 오류 메시지와 재현 단계를 남겨 설정 오류와 플랫폼 미지원을 구분한다. 한 번의 빌드 성공으로 유지보수성이나 성능을 판단하지 않는다. 빌드 시간은 같은 머신과 캐시, 프로젝트 조건에서 여러 번 측정한다.

## 4. 결과와 근거: 무엇이 달라졌나

각 결과에는 사용한 버전과 빌드 기록을 붙인다. 예를 들어 Metro의 컴파일 오류 검출을 주장하려면 누락 바인딩 예제와 빌드 로그를 제시한다. Koin의 빌드가 더 빠르다고 쓰려면 같은 조건에서 반복한 측정값을 제시한다. 근거가 없으면 결과를 추정하지 않고 기능과 구성 방식만 비교한다.

실험 뒤 아래 표를 실제 결과와 근거로 채운다.

| 항목 | 수동 DI | Hilt | Koin | Metro |
|---|---|---|---|---|
| Android 구성 성공 | 실험 후 기록 | 실험 후 기록 | 실험 후 기록 | 실험 후 기록 |
| `commonMain`에서 가능한 범위 | 공식 문서와 빌드 결과 | 공식 문서와 빌드 결과 | 공식 문서와 빌드 결과 | 공식 문서와 빌드 결과 |
| 누락 의존성 발견 시점 | 관찰 결과 | 관찰 결과 | 관찰 결과 | 관찰 결과 |
| 설정 및 빌드 단계 | 기록 | 기록 | 기록 | 기록 |
| 빌드 시간 | 동일 조건에서 반복 측정 | 동일 조건에서 반복 측정 | 동일 조건에서 반복 측정 | 동일 조건에서 반복 측정 |
| 확인한 제한 사항 | 근거와 함께 기록 | 근거와 함께 기록 | 근거와 함께 기록 | 근거와 함께 기록 |

결과를 해석할 때는 Android 통합, 공통 코드 지원, 오류 검출 시점, 빌드 설정을 기준으로 삼는다. Koin은 기본 DSL과 Compiler Plugin을 나눠 평가한다. Metro는 플러그인 호환성과 프로젝트 타깃을 확인한다. 검증한 근거를 바탕으로 독자 프로젝트의 조건에 맞는 선택지를 정리한다.

도구를 고를 때 다음 질문을 확인한다.

- DI 그래프를 공통 소스셋에 둘 필요가 있는가, 아니면 플랫폼별로 나누어도 되는가?
- 현재 팀이 유지보수할 수 있는 플러그인과 DSL은 무엇인가?
- 의존성 누락을 빌드 시점과 실행 시점 중 언제 발견해도 괜찮은가?
- 플랫폼별 객체 범위와 생명주기를 명확히 표현할 수 있는가?
- 공식 문서, 샘플, 이슈 대응이 프로젝트가 요구하는 수준으로 제공되는가?
- 도구 설정을 유지하는 비용이 수동 생성 코드보다 실제로 낮은가?

## 5. 주의할 점: 지원 범위와 측정 조건 확인하기

### KMP 지원은 라이브러리 이름만으로 판단하지 않는다

KMP 지원이 모든 API와 타깃의 지원을 뜻하지는 않는다. Android `ViewModel` 연동, 앱 시작 API, 컴파일러 플러그인 지원 타깃을 각각 확인한다. Kotlin, Gradle, Android Gradle Plugin, DI 도구 버전을 고정하고 실제 타깃에서 빌드한다. Android 전용 구현은 `commonMain`에서 참조하지 않도록 경계를 나눈다.

### 빌드 성공과 유지보수성은 서로 다른 근거다

예제가 빌드되어도 유지보수하기 쉽다는 뜻은 아니다. 객체가 늘었을 때 생성 코드와 스코프를 관리하기 쉬운지, 테스트에서 구현을 바꾸기 쉬운지, 팀이 설정을 이해할 수 있는지 확인한다. 한 프로젝트의 결과를 모든 Android와 KMP 프로젝트에 일반화하지 않는다.

### 컴파일 시간 비교는 조건을 맞춘다

한 번의 빌드 시간은 캐시, Gradle 데몬, 의존성 다운로드, 병렬 처리의 영향을 받는다. 같은 머신과 JDK를 사용하고 캐시 조건을 맞춰 여러 번 측정한다. 측정 횟수와 평균 또는 범위를 기록한다. 조건을 통제하지 못했다면 도구 간 속도를 비교하지 않는다.

### 라이브러리 안정성과 팀 비용도 선택 기준이다

API와 플러그인은 업데이트되므로 안정성 표기, 릴리스 기록, 공식 예제의 버전을 확인한다. 새 기능이 많아도 팀의 유지보수 비용이 낮아지는 것은 아니다. 장기 운영 프로젝트라면 업데이트와 빌드 문제를 맡을 사람도 정한다.

### DI가 구조 설계 문제를 대신 해결하지 않는다

DI 프레임워크는 객체 책임이나 모듈 경계를 대신 설계하지 않는다. 먼저 의존할 계약과 방향을 정하고, 생성 코드의 반복이나 범위 관리 문제가 확인된 뒤 도구를 선택한다. 작은 객체 그래프에 큰 프레임워크를 도입하면 설정 부담이 구현보다 커질 수 있다.

## 결론

Android에서 익숙한 DI 방식이 KMP 공통 코드에 그대로 맞지는 않는다. 수동 DI는 단순한 객체 그래프를 명시적으로 조립하기 좋고, Hilt는 Android 구성 요소와 통합된다. Koin은 Kotlin 선언 방식과 선택형 컴파일 검증을 제공하며, Metro는 멀티플랫폼 컴파일 시점 그래프 생성에 초점을 둔다.

같은 객체 그래프를 각 도구로 구성해 공통 코드 지원, 오류 발견 시점, 빌드 설정, 팀의 유지보수 부담을 확인한다. 지원 범위와 성능은 버전과 환경에 따라 달라지므로 공식 문서와 재현 가능한 빌드 결과로 판단한다. DI 도구는 생성 책임과 사용 책임을 분리해 변경과 테스트를 돕는다. 프로젝트와 팀이 감당할 수 있는 복잡도에 맞춰 선택한다.

## 참고 자료

- Android Developers, Dependency injection in Android: https://developer.android.com/training/dependency-injection
- Hilt documentation: https://dagger.dev/hilt/
- Koin documentation: https://insert-koin.io/docs/intro
- Kotlin Multiplatform documentation: https://kotlinlang.org/docs/multiplatform.html
- Metro 공식 문서: https://zacsweers.github.io/metro/
- Metro 공식 저장소: https://github.com/ZacSweers/metro
