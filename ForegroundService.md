# Foreground Service

앱을 만들다 보면, 사용자가 앱을 쓰고 있지 않은 상황에서도 무언가 동작해야 하는 상황이 있다.

이런 작업들을 백그라운드 작업이라고 하는데, 이런 작업을 구현하는 데는 여러 방법이 있다.

이 중에서 한 가지인 Foreground Service에 대해 알아봤다.

Foreground Service에 대해 딱 알기 위해서 아주 간단한 예시를 들어보자면..

유튜브 뮤직에서 상단 알림으로 사용자에게 지금 재생 중인 음악이 뭔지 알려주는 걸 본 적 있을 것이다.
이게 Foreground Service로 구현된 것이다.

백문이 불여일견이니, 만들면서 알아봤다.

## 서비스 만들기

공식 문서에서는 Service 클래스를 상속받으라 설명되어 있다.
그러나 Service는 onCreate, onStartCommand, onBind, onDestroy 등으로 생명주기를 직접 관리해야 한다.
반면에 LifecycleService는 LifecycleOwner 인터페이스를 구현하여 이런 관리를 조금 쉽게 다룰 수 있댜.
(coroutine이 언제 살고 언제 죽어야 하는지 등.. aac viewmodel과 viewmodelscope를 생각해보면 감아 올 것)
자세히 알아보지는 않고, 한번 import하고 적어나 보자.

먼저 gradle에 필요한 의존성을 넣는다.

```xml
implementation(”androidx.lifecycle.lifecycle-service:2.8.6”) // 2024.9.30 기준 최신
```

아래는 예시로 만들어본 LifecycleService.

추가로 작성해야 할 것들이 있으니, 일단 아래와 같이 두려고 한다.

```kotlin

class MyForegroundService: LifecycleService() {}
```

그 후 만든 서비스를 manifest에 등록한다.

서비스를 등록할 때는 내가 백그라운드에서 무슨 작업을 하는지 서비스 타입을 명시해야 한다.

위치 작업을 예시로 들면 아래와 같다.

```xml
 <application....>
    <service
        android:name=".MyForegroundService"
        android:foregroundServiceType="location"
        android:exported="false">
    </service>
 </application>
```

manifest에 추가로 서비스에 대한 권한을 등록하자.

마찬가지로, 작업에 대한 권한도 같이 요청해야 한다. 위치 작업에 관련된 권한과 서비스 권한들이다.

이것들은 공식 홈페이지에 친절하게 설명되어 있으므로 자세한 설명은 생략한다.

여기에 서비스를 시작해 주는 코드를 더하면…

```kotlin
//MainActivity.onCreate
...
    requestPermission()
    startService()
...
private fun startService() {
    val intent = Intent(applicationContext, MyForegroundService::class.java)
    startForegroundService(intent)
}
```

끝인 줄 알았지? 아직 남았다.

Foreground Service의 경우 유저에게 서비스가 실행 중이라는 것을 알려줄 notification이 필요하다.

그래서 startForegroundService(intent) 메서드를 호출해 준 후에는 notification을 표시하고 서비스를 foreground로 올려주는 startForeground 메서드를 5초 이내에 호출해 주어야 한다. 

어찌 됐던 이 글을 읽는 사람은 notification보단 백그라운드 작업을 어떻게 할지에 관심 있을 것 같으니,

notification을 만드는 코드를 아래와 같이 MyForegroundService에 적당히 작성해 주자.

```kotlin
class MyForegroundService: Service {
...
    private fun getNotification(): Notification {
        val channelName = "CHANNEL_NAME"
        val descriptionText = "Foreground Service Channel"
        val importance = NotificationManager.IMPORTANCE_DEFAULT

        val mChannel = NotificationChannel("CHANNEL_ID", channelName, importance)
        mChannel.enableLights(true);
        mChannel.lightColor = Color.BLUE;
        mChannel.description = descriptionText

        val notificationManager = getSystemService(NOTIFICATION_SERVICE) as NotificationManager
        notificationManager.createNotificationChannel(mChannel)
        val notification: Notification = Notification.Builder(this, "CHANNEL_ID")
            .setContentTitle("Foreground Service") //알림 제목
            .setContentText("Foreground Service Running") //알림 내용
            .setSmallIcon(R.drawable.ic_launcher_foreground) //알림 아이콘
            .build()
        
        return notification
    }
...
}
```

notification을 보여주려면 권한도 필요한데, 마찬가지로 자세한 설명은 생략한다.

알람에 관한 작업 끝.

주의해야할 점은 사용자에게 알람에 대한 권한을 받지 않았을 경우 Foreground Service가 켜져있더라도 알람이 보여지지 않는다는 점.

이후에 서비스의 onStartCommand에서 startForeground를 호출해 주면 된다.

### onStartCommand의 리턴값

onStartCommand의 리턴값은, 만약 시스템에 의해 우리 서비스가 종료될 경우 (배터리가 부족하거나, 메모리가 모자라거나 할 때) 어떻게 대처할지를 정한다고 보면 된다. 이 리턴값은 몇 가지가 있는데,

- START_STICKY: 강제 종료된 후 재시작하나, onStartCommand의 Intent에 null을 전달.
- START_NOT_STICKY: 강제 종료된 후 재시작하지 않기.
- START_REDELIVER_INTENT는 강제 종료되면 null이 아닌 기존의 Intent를 다시 전달한다.

다만 START_STICKY의 경우 “서비스 다시 켜줘”라고 시스템에 보내는 요청일 뿐 반드시 받아들여지는가는 보장을 못 하는 것 같다. (예: 서비스가 비정상적으로 반복해서 종료되는 경우)

```kotlin
//MyForegroundService
override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
    super.onStartCommand(intent, flags, startId)

    val notification = getNotification()
    startForeground(1, notification)

    return START_STICKY
}
```

마지막으로 텍스트를 누르면 서비스가 시작되도록 만들자.

```kotlin
//MainActivity
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.activity_main)
    requestPermission()
    findViewById<TextView>(R.id.tv_hello).setOnClickListener {
        startService()
    }
}
```

버튼을 눌러보면..

<img width="414" alt="스크린샷 2024-10-01 17 40 44" src="https://github.com/user-attachments/assets/727c6f6a-ea0c-4317-82e3-071fc4ad7fd9">
<br>
잘 된다.

## 생명주기

<img width="207" alt="lifecycleservcie" src="https://github.com/user-attachments/assets/be44c19e-7498-4d35-a6a6-a5b5b5a0c37a">

보통 서비스의 생명주기는 onCreate → onStartCommand → onDestroy 순서로 진행된다.

그런데 서비스 내부를 작성하며 몇 가지 궁금증이 생겼었다.

### 1. onCreate와 onStartCommand의 차이가 뭐지?

onCreate는 만약 서비스가 이미 돌아가고 있다면 호출되지 않는다. 그러니까, 진짜 초기화를 위한 작업을 여기에 넣으면 된다.

onStartCommand는 외부(액티비티 등)에서 startService로 서비스를 시작할 때 호출된다. 서비스가 이미 돌아가고 있어도 호출된다. 

    “아니 근데, 이미 시작한 서비스를 어떻게 다시 시작한다는 거지? 새로 만들어지는건가?“

새로 만들어지는 건 아니고, 이미 만들어진 서비스에 Intent만 다시 전달한다고 보면 될 것 같다. (startService로 Intent를 보내고 onStartCommand에서 Intent를 받고..)



### 2. 왜 onStartCommand에서 startForeground를 호출하는 거야?

startForegroundService가 호출된 뒤 5초 안에 startForeground가 호출되어야 한다고 했었다. startService시에 바로 실행되는 onStartCommand에서 startForeground를 하는 게 아무래도 상식에 맞겠지? 거기에 더해 이 작업을 서비스를 호출하는 곳마다 작성한다면, 같은 로직인데 여러 군데 분산되니 관리하기 힘들다는 이유도 있다.

서비스의 생명주기 콜백들(onCreate, onDestroy, onStartCommand)에 로그를 남겨 어떻게 돌아가는지도 직접 확인해 보자.

버튼을 눌러 서비스를 시작했을 때 onCreate, onStartCommand 순으로 잘 만들어지는 걸 볼 수 있고

<img width="456" alt="firstclick" src="https://github.com/user-attachments/assets/a1178604-e86e-4674-9668-c8046e8a4649">

버튼을 재차 눌렀을 때 onCreate가 다시 실행되는 것이 아닌, onStartCommand 만이 호출된다는 걸 알 수 있다.

<img width="448" alt="secondslick" src="https://github.com/user-attachments/assets/f38400c4-ec02-4b55-978f-a4c52c98252a">

이후에 onDestroy의 로그를 확인하기 위해 앱을 종료했으나, 호출되지 않았다. 

onDestroy는 시스템에 의해 서비스가 종료되거나, 서비스 외부에서 stopService로 서비스를 종료하는 경우, 서비스 내부에서 작업을 마치고 stopSelf로 자기 자신을 종료하는 경우에 호출되기 때문이다.

서비스를 켜자마자 종료시켜 onDestroy가 호출되는지 보자.

```kotlin
//MyForegroundService
override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
    super.onStartCommand(intent, flags, startId)

    val notification = getNotification()
    startForeground(1, notification)
    Log.d("MyForegroundService", "onStartCommand")
    stopSelf()
    return START_STICKY
}
    
override fun onDestroy() {
    super.onDestroy()
    Log.d("MyForegroundService", "onDestroy")
    job.cancel()
}
```

다음과 같이 onDestroy가 호출되는 것을 볼 수 있다.

<img width="131" alt="ondest" src="https://github.com/user-attachments/assets/bb4e40a5-563b-483c-a8e1-0a7de6817587">


## Foreground Service는 언제까지 살아있을까

Foreground Service는 사용자가 종료하거나 stopService, stopSelf 하기 전까지는 계속 살아있는 상태로 돌아갈까?

https://developer.android.com/develop/background-work/services/fg-service-timeout

dataSync 타입과 mediaProjection 타입의 Foreground service는 시간제한이 있는 게 확실해 보인다. 하지만 다른 타입의 경우 공식 문서에서 시간제한이 있는지 명확히 확인하지 못했다.

> While an app is in the foreground, it can create and run both foreground and background services freely. When an app goes into the background, it has a window of several minutes in which it is still allowed to create and use services. At the end of that window, the app is considered to be *idle*. At this time, the system stops the app's background services, just as if the app had called the services' [`Service.stopSelf()`](https://developer.android.com/reference/android/app/Service#stopSelf()) methods.


- https://developer.android.com/about/versions/oreo/background
- 번역하기 뭔가 애매해서 원문으로 가져왔다. 앱이 백그라운드에 있으면 서비스를 만들고 사용할 수 있는 몇 분 동안의 window를 가지며 window가 끝날 때가 되면 앱은 idle로 취급되고, 시스템이 앱의 백그라운드 서비스를 멈춰버린다고 한다.
- https://duckssi.tistory.com/52
- 윗글에서 안드로이드 10 버전 이후에는 앱이 백그라운드에 있으면 Foreground Service가 5분 이상 지속되지 않는다고 하는데, 위 내용과 관련된 것 같다.
- (그런데 안드로이드 14에서 내가 만든 Foreground Service는 몇 분동안 살아있던데? 영문을 모르겠다.)

Gemini에 질문해 본 결과 5분은 아니라 하고, 최대 24시간을 넘지 못하고, 최소 시간은 보장하지 못한다고 하는데, 근거 문서를 찾지 못했다.

이후에 직접 실험을 하든 해서 정확한 정보를 얻은 다음 공유하고 싶다.

## 구성이 애매해서 적당히 던져둔 기타 몇 가지 정보들.

- WorkManager는 Worker Thread( background thread)에서 동작하고, Foreground Service는 Main Thread( Ui thread)에서 동작한다. Foreground는 단지 notification을 보여주고 아니고의 차이만은 아니다. 따라서 Foreground service를 쓸 땐 ANR이 돌지 않도록 해야한다. 만약 다른 쓰레드에서 돌리고 싶다면 쓰레드를 따로 만들어서 돌리던, 코루틴을 다른 Dispatcher에서 돌리든 해야 한다.
- Foreground service는 기기가 sleep에 들어갈 경우 동작이 느려지는 문제점이 있다. 이것을 wake lock을 검으로써 해결할 수 있다.
    - https://stackoverflow.com/questions/55170819/android-slows-down-foreground-service-when-device-sleeps
    - https://stackoverflow.com/questions/67618284/android-kotlin-foreground-service-stops-after-some-time
- 안드로이드 13부터 유저는 Foreground service와 연동된 notification을 없앨 수 있는 게 기본 설정이다. 이전에는 foreground service가 사라지기 전까지 없앨 수 없었다.
