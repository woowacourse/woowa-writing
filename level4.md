# R8으로 앱 최적화하기

## 소개

안드로이드 앱을 빌드할 때 R8이라는 도구를 사용해 앱을 최적화할 수 있습니다. **앱 최적화**(app optimization)는 앱의 코드에서 기능적으로 필요하지 않은 부분을 제거하고 단순화해 앱을 더 작고 빠르게 만드는 과정입니다.

이 글을 통해 R8을 사용한 안드로이드 앱 최적화를 소개하고, 적용 시 발생할 수 있는 오류와 트러블슈팅 방법을 간단히 설명하고자 합니다.

### 앱이 커지는 이유

우리가 안드로이드 앱을 만들 때 작성하는 코드는 필요 이상으로 큽니다.

그 이유 중 하나는 외부 라이브러리를 사용한다는 점입니다. 외부 라이브러리를 사용하면 라이브러리의 기능 중 일부만 사용하더라도 라이브러리 전체가 앱에 포함되게 됩니다. 예를 들어 앱을 만들 때 코틀린을 사용하기만 해도 자바로 만든 똑같은 앱에 비해 더 커지게 됩니다. Kotlin Standard Library가 통째로 앱에 포함되기 때문입니다.

또다른 이유는 가독성과 유지보수성을 위해 일부러 더 긴 코드를 쓴다는 것입니다. 프로퍼티와 함수에 의미있는 이름을 붙여주거나, 주석을 추가하거나, 빌더 패턴을 적용하는 경우를 예시로 들 수 있습니다. 좋은 개발 습관일 수는 있지만 코드가 더 길어지게 만들고, 앱의 동작에 필수적인 코드는 아닙니다. 사람이 읽고 쓰기 더 쉬우라고 하는거지, 기계 입장에서는 좌표를 나타내는 변수의 이름이 `coordinate`이든 `a`이든 상관 없으니까요.

### 앱 최적화를 적용하는 이유

앱 최적화를 적용하면:

- APK의 크기를 줄여 더 빠르게 다운로드 및 설치될 수 있게 합니다.
- 런타임 성능을 개선하고 ANR이 발생할 확률을 줄여 더 좋은 사용자 경험을 만들어줍니다.
- 우리의 앱이 삭제당할 확률이 줄여줍니다(기기에 용량이 부족해지면 큰 앱부터 삭제하고 싶어지기 마련이니까요).

다음은 [런세권](https://play.google.com/store/apps/details?id=io.coursepick.coursepick) 프로젝트를 진행하면서 R8 최적화를 적용하기 전과 후의 APK 파일을 비교한 것입니다. APK 파일의 크기가 60.1 MB에서 45.2 MB로 줄었음을 확인할 수 있습니다.

|적용 전|적용 후|
|-|-|
|<img width="880" height="506" alt="478680380-7a10fbf9-72db-4ccc-8eca-8b543d9a71b2" src="https://github.com/user-attachments/assets/a7d2c880-9877-4f3a-838d-3f2778117c1b" />|<img width="880" height="506" alt="478680376-0f047251-aeae-4fa4-9650-f5c15ea18904" src="https://github.com/user-attachments/assets/9dc37954-25d2-4441-a37a-e6df200b62fe" />|

### 안드로이드 빌드 과정

R8가 앱을 최적화하는 과정을 이해하기 위해 안드로이드 빌드 과정 및 관련 용어를 간단히 짚고 넘어가겠습니다.

안드로이드 앱은 일반적으로 자바 또는 코틀린 코드로 작성됩니다. 자바/코틀린 코드는 자바 바이트코드로 컴파일되는데, 안드로이드는 자바 바이트코드를 직접 실행시키지 않습니다. 대신 Dalvik bytecode로 컴파일돼서 안드로이드가 실행할 수 있는 Dalvik executable(DEX) 파일에 저장됩니다. Dalvik bytecode를 생성하는 과정은 덱싱(dexing)이라고 부릅니다.

### ProGuard vs R8

R8는 위에 설명된 빌드 과정에서 코드를 최적화해줍니다. 그런데 안드로이드 앱 최적화에 대한 자료를 찾아보면 R8뿐만 아니라 ProGuard라는 이름도 자주 보실 수 있을 겁니다.

[ProGuard](https://www.notion.so/Making-Android-app-smaller-with-ProGuard-R8-2521d20ce1e9802b9597fc5943d3f354?pvs=21)는 Guardsquare 사에서 개발한 코드 최적화 툴로, 자바 바이트코드에 별도의 후처리를 적용하는 방식입니다. ProGuard 최적화의 결과물은 여전히 자바 바이트코드이기 때문에 덱싱을 거쳐야 안드로이드가 실행할 수 있게 됩니다.

R8은 구글이 개발한 컴파일러로, 코드 최적화와 덱싱을 하나의 과정으로 통합시킵니다. 참고로 R8은 릴리즈 빌드용 컴파일러이며, 디버그 빌드는 D8이라는 컴파일러가 담당합니다.

Android Studio 3.4 및 Android Gradle Plugin 3.4를 기점으로 기본 코드 최적화 툴이 ProGuard에서 R8으로 교체됐습니다. 현재는 특별한 이유가 없다면 R8을 사용하는 것이 권장됩니다.

## 앱 최적화 과정

ProGuard와 R8은 크게 세 가지 과정을 거쳐 앱을 최적화합니다. **코드 축소**, **코드 최적화**, 그리고 **코드 난독화**입니다.

### 코드 축소

**코드 축소**(code shrinking)는 코드를 정적으로 분석해 사용되지 않는 코드를 제거합니다.

지정된 몇 가지 진입점을 기준으로 도달할 수 있는 모든 코드를 확인하고, 그 어떤 진입점에서 시작해도 도달할 수 없다고 판단되는 나머지 코드는 제거합니다. 이 진입점은 R8의 **keep rule**이라는 규칙에 의해 정의되는데, keep rule은 앱을 안전하게 최적화하기 위해 꼭 알아야 하는 개념이니 아래에서 더 자세히 설명하겠습니다.

### 코드 최적화

**코드 최적화**(code optimization)는 코드의 런타임 성능을 개선하기 위해 다양한 기법을 적용하는 과정입니다. 내부적으로 굉장히 다양한 최적화를 해주지만, 사용되는 기법 중 대표적인 것으로는 **필드 인라이닝**과 **메서드 인라이닝**이 있습니다.

**필드 인라이닝**은 상수 필드의 값이 사용되는 곳에 직접 삽입되도록 코드를 변경합니다. 예를 들어 Intent에 extra를 get/put하기 위해 사용할 키 값들을 다음과 같이 상수로 선언했다면, R8에 의해 다음과 같이 인라인 처리될 수 있습니다.

<table>
<tr>
<th>필드 인라이닝 전</th>
<th>필드 인라이닝 후</th>
</tr>
<tr>
<td>
<pre><code class="language-kotlin">

    object IntentKeys {
        const val INTENT_KEY_PLACE_LATITUDE = "place_latitude"
        const val INTENT_KEY_PLACE_LONGITUDE = "place_longitude"
    }
    
    class SearchActivity : ComponentActivity() {
        private fun submit(place: Place) {
            val resultIntent =
                Intent().apply {
                    putExtra(DataKeys.DATA_KEY_PLACE_LATITUDE, place.latitude)
                    putExtra(DataKeys.DATA_KEY_PLACE_LONGITUDE, place.longitude)
                }
            setResult(RESULT_OK, resultIntent)
            finish()
        }
        // ...
    }


</code></pre>
</td>
<td>
<pre><code>

    class SearchActivity : ComponentActivity() {
        private fun submit(place: Place) {
            val resultIntent =
                Intent().apply {
                    putExtra("place_latitude", place.latitude)
                    putExtra("place_longitude", place.longitude)
                }
            setResult(RESULT_OK, resultIntent)
            finish()
        }
        // ...
    }

</code></pre>
</td>
</tr>
</table>

**메서드 인라이닝**은 함수를 호출하는 대신, 함수가 수행하는 로직을 호출 위치에 바로 삽입하는 방식으로 변경해 코드를 최적화합니다.

<table>
<tr>
<th>메서드 인라이닝 전</th>
<th>메서드 인라이닝 후</th>
</tr>
<tr>
<td>
<pre><code class="language-kotlin">

    class SearchActivity : ComponentActivity() {
        companion object {
            fun intent(context: Context): Intent = Intent(context, SearchActivity::class.java)
        }
    }
    
    class CoursesActivity : AppCompatActivity() {
        override fun search() {
            val intent = SearchActivity.intent(this)
            val query: String? = viewModel.state.value?.query
            intent.putExtra(DataKeys.DATA_KEY_PLACE_NAME, query)
            searchLauncher.launch(intent)
        }
        // ...
    }

</code></pre>
</td>
<td>
<pre><code>

    class CoursesActivity : AppCompatActivity() {
        // ...
        override fun search() {
            val intent = Intent(this, SearchActivity::class.java)
            val query: String? = viewModel.state.value?.query
            intent.putExtra(DataKeys.DATA_KEY_PLACE_NAME, query)
            searchLauncher.launch(intent)
        }
        // ...
    }

</code></pre>
</td>
</tr>
</table>

코드 최적화 기법에 대한 더 자세한 예시는 [Android Dev Summit 2019 발표 영상](https://youtu.be/uQ_yK8kRCaA?si=s4-qyAkCTFPpbZ7v&t=562)(09:22 ~ 15:30)에 잘 정리돼있으니 참고하시는 것을 추천합니다.

### 코드 난독화

앞서 언급했듯 우리가 패키지, 클래스, 함수, 변수 등에 붙이는 "의미있는" 이름들은 사람이 볼 때만 의미가 있습니다. 컴파일된 코드는 사람이 읽기 좋을 필요가 없고, 오히려 사람이 읽기 안 좋은 쪽이 보안상 더 나을 수 있습니다.

**코드 난독화**(code obfuscation)는 패키지 안에 패키지가 들어있는 구조를 평탄화하고, 클래스 및 클래스 멤버의 이름을 줄입니다. 예를 들어 `com.example.app` 패키지의 `ExampleClass` 클래스에 `exampleFunction()` 함수가 있다면, `k` 패키지의 `a1` 클래스의 `d()` 함수처럼 의미 없는 이름으로 바뀌게 됩니다.

여기서 "난독화"라는 표현 때문에 오해가 생길 수 있는데, ProGuard와 R8은 보안 툴이 아닙니다. [ProGuard 공식 메뉴얼](https://www.guardsquare.com/manual/configuration/usage)에서는 다음과 같이 설명하고 있습니다.

> Both ProGuard and R8 were designed for app optimization, and although they employ minimal obfuscation techniques, they are not security tools and do not harden applications effectively against reverse engineering and tampering.

> ProGuard와 R8는 앱 최적화를 위해 설계됐으며, 간단한 난독화 기법을 사용하지만 보안 툴이 아니고, 리버스 엔지니어링이나 변조로부터 어플리케이션을 효과적으로 견고하게 만들어주지 않습니다.

앱 최적화 과정에서 코드가 난독화되는 건 부수 효과일 뿐이지, 코드의 보안을 높이는 것이 목적은 아닙니다. 때문에 앱 최적화의 맥락에서는 code obfuscation 대신 **identifier renaming**(식별자 이름 변경)이라는 표현이 사용되기도 합니다.

## 사용법

R8을 활성화하는 것 자체는 간단합니다. `build.gradle` 파일에 다음 코드를 추가해주기만 하면 됩니다.

```kotlin
buildTypes {
    release {
        isMinifyEnabled = true  // R8 활성화
        proguardFiles(getDefaultProguardFile("proguard-android-optimize.txt"))
        // ...
    }
}
```

앱 최적화를 적용하면 빌드 시간이 상당히 길어지기 때문에, 특별한 이유가 없다면 릴리즈 빌드에만 적용하는 것이 권장됩니다.

...하지만 실전은 그렇게 간단하지 않습니다. 저도 *너무 간단한데?*라고 생각하고 런세권에 바로 적용해보았을 때, 릴리즈 빌드에서 런타임 에러를 발생하는 현상을 겪었습니다.
### 주의할 점

앱 최적화는 리플렉션과 궁합이 좋지 않습니다. R8은 코드를 정적으로 분석해서 코드를 최적화하는데, 컴파일 타임에는 불필요한 것처럼 보이는 코드가 사실은 리플렉션에 의해 런타임에 사용될 수 있기 때문입니다.

또한 `Class.getDeclaredMethod()` 등 이름으로 특정 멤버를 찾아내는 코드가 있을 경우, R8의 코드 난독화로 인해 멤버의 이름이 바뀌기 때문에 더이상 찾을 수 없게 됩니다. 우리가 함수에 `exampleFunction()`이라는 이름을 붙여서 `getDeclaredMethod("exampleFunction")`으로 찾으려고 해도, 빌드 시 R8에 의해 `d()`같은 의미없는 이름으로 바뀌어 `"exampleFunction"`이라는 이름은 사라지기 때문입니다.

때문에 앱 최적화를 리플렉션과 함께 사용할 경우 분명 존재했던 코드를 런타임에서는 찾지 못하게 되어 `NoSuchElementException` 같은 오류가 발생할 수 있습니다. 이를 방지하려면 적절한 **keep rule**이라는 것을 적용해야 합니다.

### Keep rule

**Keep rule**은 ProGuard와 R8한테 "이 코드는 건들면 안 돼"라고 알려주는 규칙입니다. 앱 최적화가 문제를 일으켜서 커스텀 keep rule을 추가해야 한다면, 앱 모듈의 루트 디렉터리에 `proguard-rules.pro`파일을 만들어서 keep rule을 추가하고, 빌드 설정에 다음과 같이 해당 파일을 사용하겠다고 명시해주면 됩니다. 일반적으로는 안드로이드 프로젝트를 생성할 때 자동으로 해당 파일도 생성됩니다.

```kotlin
buildTypes {
    release {
        isMinifyEnabled = true
        proguardFiles(
            getDefaultProguardFile("proguard-android-optimize.txt"),
            "proguard-rules.pro",  // Keep rule 추가
        )
    }
}
```

Keep rule 문법은 필요에 따라 [공식 메뉴얼](https://www.guardsquare.com/manual/configuration/usage)을 참고하는 것을 추천합니다.

참고로 안드로이드 앱이 빌드될 때, AAPT2가 안드로이드의 4대 컴포넌트에 필요한 keep rule을 자동으로 추가해주기 때문에 안드로이드 컴포넌트에 대한 keep rule은 별도로 추가해주지 않아도 됩니다. 또한 R8의 keep rule은 ProGuard의 keep rule과 호환되도록 설계되었습니다.

### 외부 라이브러리를 사용할 경우

> 내 코드에서는 리플렉션 안 썼는데요?

우리가 직접 작성한 코드가 리플렉션을 쓰지 않더라도 안심할 수는 없습니다. 사용하는 외부 라이브러리 중 리플렉션을 쓰는 것이 있을 수 있으니까요. 외부 라이브러리도 앱 코드의 일부가 되기 때문에, 적절한 keep rule을 추가해주지 않으면 앱 축소로 인한 문제가 발생할 수 있습니다.

물론 [OkHttp](https://square.github.io/okhttp/features/r8_proguard/#r8-proguard)나 [Retrofit](https://square.github.io/retrofit/download/#r8--proguard)처럼 필요한 keep rule이 내장돼있어 개발자가 직접 규칙을 추가해주지 않아도 되는 친절한 라이브러리도 있습니다. 혹은 공식 문서에서 어떤 규칙을 추가해야 하는지 안내해주는 라이브러리도 있고요. 하지만 라이브러리가 애초에 안드로이드를 염두하지 않고 제작됐을 수도 있고, 그 외에도 어떤 이유로든 라이브러리가 R8 친화적이지 않아서 개발자가 직접 적절한 keep rule을 추가해줘야 할 수 있습니다.

때문에 [안드로이드 공식 문서](https://developer.android.com/topic/performance/app-optimization/choose-libraries-wisely)는 라이브러리를 사용할 때도, 개발할 때도 코드 최적화와의 호환성을 유의할 것을 권장하고 있습니다. 예를 들어 직렬화 라이브러리가 필요하다면, 리플렉션을 쓰는 Gson 대신 kotlinx.serialization을 사용하는 것이 권장됩니다.

리플렉션을 사용하는 라이브러리를 부득이하게 사용해야 한다면, "특정 인터페이스의 구현체" 등 특정할 수 있는 범위에서만 리플렉션을 사용하는 라이브러리가 권장됩니다. 이런 경우 라이브러리의 대부분 또는 전체를 최적화에서 제외시킬 필요 없이, 필요한 작은 부분만 keep rule에 추가해서 제외시키면 나머지는 안전하게 최적화할 수 있기 때문입니다.

## 트러블슈팅 팁

앱 최적화로 인해 오류가 발생할 경우, 어떤 keep rule을 추가해줘야 하는지 알아내야 합니다. 런세권을 개발하면서 원인을 파악할 때 도움이 됐던 방법 몇 가지를 공유합니다.

### `NoSuchSomething` 오류 확인하기

앱 최적화 후 리플렉션으로 인한 문제가 생길 경우, `NoSuchElementError`, `NoSuchFieldError` 등 특정 요소를 찾지 못 하는 오류가 발생하고 있을 확률이 높습니다. 해당 오류가 발생하는 위치를 찾아서 keep rule을 적용해야 할 대상을 좁힐 수 있습니다.

예를 들어 카카오맵 SDK를 사용하는 앱에서 적절한 keep rule을 추가하지 않고 앱 최적화를 활성화한다면 다음과 같은 오류가 발생할 수 있습니다.

```kotlin
java.lang.NoSuchFieldError: no "I" field "x" in class "Lcom/kakao/vectormap/internal/RenderViewOptions;"
java.lang.NoSuchMethodError: no static method "Lcom/kakao/vectormap/camera/CameraPosition;.from(DDIDDD)Lcom/kakao/vectormap/camera/CameraPosition;"
```

다음과 같이 `com.kakao.vectormap`부터 시작해서 R8가 필요한 코드를 잘못 제거하지 않도록 적절한 keep rule을 찾아갈 수 있습니다.

```kotlin
-keep class com.kakao.vectormap.** { *; }
-keep interface com.kakao.vectormap.**
```

### 일부 최적화 과정 비활성화하기

`proguard-rules.pro` 파일에 다음 규칙을 추가하고 하나씩 번걸아가며 제거함으로써 세 가지 최적화 과정 중 무엇이 문제를 일으키는지 확인할 수 있습니다. 특히 코드 난독화는 로그를 읽기 어렵게 만들기 때문에, 난독화 자체가 문제의 원인은 아니더라도 잠시 비활성화하는 것이 트러블슈팅에 도움이 될 수 있습니다.

```bash
-dontshrink     # 코드 축소 비활성화
-dontoptimize   # 코드 최적화 비활성화
-dontobfuscate  # 코드 난독화 비활성화
```

### 최적화 관련 정보 출력해서 확인하기

`proguard-rules.pro` 파일에 다음 규칙을 추가해서 빌드 시 앱 최적화와 관련된 정보를 출력하도록 만들 수 있습니다.

```bash
-printconfiguration  # 적용된 모든 규칙을 출력
-printmapping        # 난독화 전/후 이름에 대한 매핑을 출력
-printusage          # 코드 축소 과정에서 제거된 클래스 및 멤버를 출력
-printseeds          # 코드 축소 과정에서 제거되지 않은 클래스 및 멤버를 출력
```

## TL;DR

- R8은 코드 축소, 코드 최적화, 코드 난독화를 통해 앱을 더 작게 만들어주는 도구입니다.
- R8을 사용할 때는 리플렉션과의 충돌을 주의해야 합니다(직접 쓴 코드든 라이브러리든).
- R8과 리플렉션이 충돌하지 않도록 적절한 keep rule을 사용해 앱을 안전하게 최적화할 수 있습니다.
