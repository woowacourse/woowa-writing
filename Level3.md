# Android Context, 앱의 맥락을 이해하기

## 1장. 서론: 문맥

사람과 대화를 할 때 우리는 항상 문맥을 이해하며 대화를 이어갑니다. 문맥(context)란 어떤 사건이나 대화가 발생하는 주변 상황과 배경을 의미합니다. 문맥을 제대로 이해하지 못하면 대화가 끊기거나 상대방이 의미를 제대로 파악하지 못할 수 있습니다.

안드로이드 개발에서도 마찬가지입니다. 개발자가 앱을 만들 때 앱의 현재 상태와 흐름을 이해하고 있어야 다양한 기능을 올바르게 구현할 수 있습니다. 이때 필요한 것이 바로 **안드로이드 컨텍스트(Context)**입니다.

안드로이드 컨텍스트는 주로 리소스를 가져오거나, 화면에 메시지를 출력할 때 사용됩니다. 그뿐만 아니라 데이터베이스 접근, 시스템 서비스 호출 등 앱 개발에서 매우 자주 사용되는 개념입니다.

이 글에서는 안드로이드 컨텍스트가 무엇인지, 종류는 무엇이 있는지, 그리고 왜 중요한지 자세히 설명합니다.

<img width="768" height="283" alt="스크린샷 2025-10-14 오후 4 22 45" src="https://github.com/user-attachments/assets/6ce3846a-cecb-4842-8f49-9b0f64ad4e52" />


## 2장. 안드로이드 Context

안드로이드 공식 문서에서는 컨텍스트를 다음과 같이 정의합니다.

> “앱 환경에서 전역 정보를 제공하는 인터페이스이다. 이는 안드로이드 시스템에서 제공하는 추상 클래스이며, 앱 특정 리소스와 클래스에 접근할 수 있도록 해주며, 액티비티(Activity)를 실행하거나 브로드캐스트(Broadcast)를 보내고, 인텐트(Intent)를 수신하는 등 앱 수준의 작업을 수행할 수 있게 해준다.”
> 

쉽게 풀면, 컨텍스트는 앱이 실행되는 환경 정보와 상태를 나타내는 객체라고 이해할 수 있습니다.

즉, 컨텍스트는 앱이 현재 어디에서, 어떤 상태로 실행되고 있는지 알려주고, 개발자가 그 상태를 기반으로 기능을 수행하도록 돕는 역할을 합니다.

## 3장. **컨텍스트가 제공하는 기능**

안드로이드 컨텍스트(Context)를 사용하면 앱 개발에서 필요한 다양한 기능을 수행할 수 있습니다. 컨텍스트가 제공하는 기능을 이해하면, 앱을 안정적이고 효율적으로 만들 수 있습니다.

1. **앱 리소스 접근**: 앱 내에 정의된 문자열, 이미지, 색상 등 리소스(Resource)를 가져올 수 있습니다.
    
    ```kotlin
    // 문자열 리소스 접근
    val appName = context.getString(R.string.app_name)
    
    // 이미지 리소스 접근
    val logoDrawable = ContextCompat.getDrawable(context, R.drawable.ic_logo)
    ```
    
2. **시스템 서비스 접근**: 시스템 서비스란 센서, 알람, 위치 서비스, 네트워크 상태 등 **운영체제가 제공하는 기능**을 말하며, 이 시스템 서비스에 접근할 수 있습니다.
    
    ```kotlin
    // 알람 서비스 사용
    val alarmManager = context.getSystemService(Context.ALARM_SERVICE) as AlarmManager
    val intent = Intent(context, AlarmReceiver::class.java)
    val pendingIntent = PendingIntent.getBroadcast(context, 0, intent, PendingIntent.FLAG_UPDATE_CURRENT)
    alarmManager.setExact(AlarmManager.RTC_WAKEUP, System.currentTimeMillis() + 60000, pendingIntent)
    ```
    
3. **화면 전환**: 컨텍스트를 사용하면 새로운 액티비티를 실행하거나 화면을 전환할 수 있습니다.
    
    ```kotlin
    val intent = Intent(context, MainActivity::class.java)
    context.startActivity(intent)
    ```
    
4. **데이터 저장 및 접근:** 앱 데이터를 안전하게 저장하고 가져올 수 있습니다. 대표적으로 Shared Preferences와 데이터베이스 접근이 있습니다.
    
    ```kotlin
    // Shared Preferences 생성
    val sharedPref = context.getSharedPreferences("user_prefs", Context.MODE_PRIVATE)
    
    // 값 저장
    sharedPref.edit().putString("username", "뭉치").apply()
    
    // 값 읽기
    val username = sharedPref.getString("username", "기본값")
    ```
    
    데이터베이스 접근도 컨텍스트를 기반으로 수행합니다. 예를 들어 Room 데이터베이스 초기화 시 컨텍스트를 전달해야 합니다.
    
    ```kotlin
    val db = Room.databaseBuilder(context, AppDatabase::class.java, "app_db").build()
    ```
    
5. **사용자 메시지 출력**: 사용자에게 Toast 메시지 전달
    
    ```kotlin
    Toast.makeText(context, "토스트메시지가 출력됐습니다.", Toast.LENGTH_SHORT).show()
    ```
    

정리하면, 앱 리소스 접근, 시스템 서비스 사용, 화면 전환, 데이터 저장, 사용자 메시지 출력 등 안드로이드의 핵심 기능 대부분은 컨텍스트 없이는 수행할 수 없습니다.

컨텍스트는 앱이 실행되는 환경 정보를 제공하고, 개발자가 앱을 원하는 대로 제어할 수 있도록 돕는 핵심 객체입니다.

## **4장. 컨텍스트의 구성**

안드로이드의 핵심 구성 요소는 모두 컨텍스트(Context)를 기반으로 동작합니다.

컨텍스트가 앱의 환경 정보를 제공하고 기능을 수행하게 하기 때문에, 액티비티(Activity)나 애플리케이션(Application)도 내부적으로 컨텍스트를 포함하고 있습니다.

- 액티비티에서 상속하는 AppCompatActivity 의 경우
    
    AppCompatActivity → FragmentActivity → activity/ComponenetActivity → app/ComponentActivity → Activity → ContextThemeWrapper → ContextWrapper → Context
    
    를 통해 결국은 Context 를 조합하고 있습니다.
    
- 애플리케이션의 경우
    
    Application → ContextWrapper → Context
    
    를 통해 결국은 Context 를 조합하고 있습니다.
    
    <img width="1234" height="760" alt="context" src="https://github.com/user-attachments/assets/f2b4f0e1-846a-4e14-94f9-36b0d35d5c8a" />

    

따라서, 액티비티 혹은 애플리케이션에서 this 를 통해 context에 접근할 수 있게 됩니다.

## **5장. ContextWrapper란?**

안드로이드에서 ContextWrapper는 다른 컨텍스트를 감싸는 래퍼(Wrapper) 클래스입니다.

즉, 실제 컨텍스트의 기능을 그대로 전달하면서, 필요에 따라 기능을 확장하거나 수정할 수 있게 해주는 객체입니다.

쉽게 말하면, 컨텍스트를 그대로 쓰되, 약간 변형하거나 추가 기능을 붙이고 싶을 때 사용하는 장치라고 이해하면 됩니다.

### 1. 사용 이유

ContextWrapper를 사용하는 주요 이유는 다음과 같습니다.

1. 기존 컨텍스트 기능 확장
    - 기존 컨텍스트의 기능을 그대로 사용하면서, 추가 기능을 덧붙일 수 있습니다.
    - 예: 특정 화면에서만 리소스를 다르게 적용하거나, 테마를 바꾸고 싶을 때
2. 기능 제한
    - 컨텍스트 일부 기능만 노출하거나 제한하고 싶은 경우 사용합니다.
    - 예: 보안 상 일부 서비스 접근을 막거나, 특정 액티비티 범위에서만 동작하도록 제한

### 2. 구조
<img width="1179" height="458" alt="스크린샷 2025-10-14 오후 4 23 38" src="https://github.com/user-attachments/assets/ab785b1b-8a1c-46dc-a025-0060dd20221c" />


- ContextWrapper는 기본 컨텍스트(Context)를 내부에 저장합니다.
- 모든 컨텍스트 호출은 내부의 실제 컨텍스트로 전달됩니다.
- 필요한 경우 일부 메서드를 재정의하여 기능을 수정할 수 있습니다.

### **3. 예시 코드**

```kotlin
class LoggingContext(base: Context) : ContextWrapper(base) {

    override fun getSystemService(name: String): Any? {
        Log.d("LoggingContext", "getSystemService called: $name")
        return super.getSystemService(name)
    }
}

// 사용처
val wrappedContext = LoggingContext(this) // this는 Activity 컨텍스트
val sensorManager = wrappedContext.getSystemService(Context.SENSOR_SERVICE)
```

위 코드에서 LoggingContext는 기존 컨텍스트 기능을 그대로 제공하면서, 시스템 서비스 호출 시 로그를 남깁니다.

이처럼 ContextWrapper를 사용하면 **기존 컨텍스트를 안전하게 확장**할 수 있습니다.

### 4. **ContextWrapper와 ContextThemeWrapper의 차이**

<img width="800" height="500" alt="context" src="https://github.com/user-attachments/assets/f2b4f0e1-846a-4e14-94f9-36b0d35d5c8a" />

- ContextThemeWrapper는 테마(Theme) 적용 기능이 추가된 ContextWrapper입니다.
- Activity에서 setTheme()나 getTheme()을 호출할 때 내부적으로 ContextThemeWrapper를 통해 처리됩니다.

### 5. 구조가 복잡한 이유

ContextWrapper는 컨텍스트에 마치 옷을 입혀 필요한 기능만 추가하거나 감추는 장치입니다.

이 때문에 안드로이드에서는 액티비티나 애플리케이션이 단일 컨텍스트를 직접 사용하는 것이 아니라, 여러 단계의 래퍼(wrapper)를 거쳐 기능을 확장하거나 제한하는 구조를 가지게 되었습니다.

Activity가 ContextWrapper를 직접 상속하지 않고 ContextThemeWrapper를 거쳐 

즉, 다양한 기능과 확장성을 위해 컨텍스트 구조가 복잡해진 것입니다.

## **6장. 컨텍스트와 메모리 누수**

컨텍스트를 잘못 사용하면 메모리 누수(Memory Leak)가 발생합니다.

메모리 누수란 가비지 컬렉터(Garbage Collector)가 메모리를 해제하지 못하는 상황을 말합니다.

예시: 액티비티가 종료된 후에도 액티비티 컨텍스트(Activity Context)를 참조하면 메모리가 해제되지 않고 앱이 느려지거나 강제 종료될 수 있습니다.

- `메모리 릭`:  메모리 누수
    - GC 에 의해 메모리가 제거되어야 하는데, 메모리에서 해제되어야 할 참조가 걸려있을 경우 GC가 메모리를 해제하지 않습니다.
    - e.g. activity가 destroyed 된 후에도 activity context 를 참조하고 있으면 메모리가 해제되지 않아 메모리 릭이 발생합니다.

1. 액티비티 컨텍스트 참조로 인한 메모리 누수
가장 흔한 메모리 누수 사례는 액티비티(Activity) 컨텍스트를 오래 참조하는 경우입니다.

```kotlin
class MySingleton private constructor(private val context: Context) {

    companion object {
        private var instance: MySingleton? = null

        fun getInstance(context: Context): MySingleton {
            if (instance == null) {
                // 액티비티 컨텍스트를 그대로 저장하면 메모리 누수 발생
                instance = MySingleton(context)
            }
            return instance!!
        }
    }
}
```

위 코드에서 액티비티 컨텍스트를 싱글턴(Singleton) 객체에 저장하면, 해당 액티비티가 종료되어도 컨텍스트가 참조되므로 메모리가 해제되지 않습니다.

결과적으로 앱이 점점 느려지거나, 최악의 경우 강제 종료될 수 있습니다.

**2. 안전한 컨텍스트 사용 방법**

```kotlin
class MySingleton private constructor(private val context: Context) {

    companion object {
        private var instance: MySingleton? = null

        fun getInstance(context: Context): MySingleton {
            if (instance == null) {
                // Application 컨텍스트 사용
                instance = MySingleton(context.applicationContext)
            }
            return instance!!
        }
    }
}
```

`context.applicationContext`를 사용하면, 액티비티의 생명주기와 무관하게 앱 전체에서 안전하게 공유할 수 있습니다. 또한, 액티비티 종료와 상관없이 메모리 누수 걱정 없이 사용할 수 있습니다.

또한, 액티비티 컨텍스트를 싱글턴에 넣으면 액티비티가 종료되어도 GC가 회수하지 못해 앱 메모리가 점점 증가합니다. 반대로 ApplicationContext는 앱 전체에서 공유되는 싱글턴이므로 안전합니다.

> 올바른 컨텍스트 사용은 앱 성능과 안정성을 지키는 핵심 요소입니다.
> 

## **7장. Application 컨텍스트와 Activity 컨텍스트**

<img width="753" height="530" alt="스크린샷 2025-10-14 오후 4 24 49" src="https://github.com/user-attachments/assets/1496e556-2726-4f07-bc64-c703a540c2ed" />


| **구분** | **특징** | **사용 예시** |
| --- | --- | --- |
| Application Context | 앱 전체에서 공유되는 싱글턴(Singleton) 인스턴스, 앱 런칭부터 종료까지 유지 | 데이터베이스 접근, 앱 전체 리소스 접근 |
| Activity Context | 특정 액티비티 범위에서만 존재, 액티비티 라이프사이클 공유 | 화면 구성(UI), LayoutInflater, Toast 메시지 |

LayoutInflater, Toast, Dialog 등은 액티비티 컨텍스트에서만 정확히 동작합니다. ApplicationContext를 쓰면 예외가 발생하거나 UI가 제대로 렌더링되지 않을 수 있습니다.

Compose에서도 context 사용 시 `LocalContext.current`로 얻은 컨텍스트는 액티비티 컨텍스트(Activity Context)를 반환합니다.
``` kotlin
@Composable
fun GreetingToast() {
    val context = LocalContext.current // ActivityContext 반환
    Button(onClick = {
        Toast.makeText(context, "안녕하세요!", Toast.LENGTH_SHORT).show()
    }) {
        Text("토스트 출력")
    }
}
```

### 그리고 Base Context

- ContextWrapper 에서 사용하는 context 로 실제로는 Context 의 실제 구현체
- 언제 쓰는가?
    - 개발 시에 잘 쓰지 않고, activityContext 혹은 applicationContext 의 내부 구현체에서 사용하고 있습니다.
- [baseContext 사용을 지양해야 하는 이유](https://medium.com/@ali.muzaffar/which-context-should-i-use-in-android-e3133d00772c): 여러 context 를 참조하고 있기에 어떤 반환값인지 모호합니다.

## **8장. 컨텍스트 사용 시 주의 사항**

1. 액티비티 컨텍스트는 화면 관련 작업에서만 사용합니다.
    - 예: LayoutInflater, UI 구성, Toast
2. 애플리케이션 컨텍스트는 앱 전체에서 공유합니다.
    - 예: 데이터베이스 접근, 네트워크 서비스, 싱글턴 객체 초기화
3. Base Context 직접 참조하지 않습니다.
4. 액티비티 컨텍스트를 오래 참조하지 않습니다.
    - 액티비티 컨텍스트를 오래 참조하면 메모리 누수 발생

## **9장. 정리**

1. 컨텍스트는 앱의 현재 상태와 환경 정보를 제공하는 객체입니다.
2. 안드로이드 핵심 기능과 구성 요소는 모두 컨텍스트 기반으로 동작합니다.
3. Application 컨텍스트와 Activity 컨텍스트를 목적에 맞게 사용하면 메모리 누수 위험을 줄일 수 있습니다.
4. ContextWrapper와 Base Context는 내부 구현체이므로 직접 참조는 피해야 합니다.
5. 올바른 컨텍스트 사용은 앱 안정성과 성능을 높이는 핵심 요소입니다.

# 참고 자료

[테코톡-context](https://www.youtube.com/watch?v=yoU-3ks7e5Q)

[Context in Android - A Deep Dive 영상](https://www.youtube.com/watch?v=S22NlX4iXJU)

[Context 공식문서](https://developer.android.com/reference/android/content/Context)

[ContextWrapper 공식 문서](https://developer.android.com/reference/android/content/ContextWrapper)

[Context In Android Application 포스팅](https://outcomeschool.com/blog/context-in-android-application)

[Which Context should I use in Android? 포스팅](https://medium.com/@ali.muzaffar/which-context-should-i-use-in-android-e3133d00772c)
