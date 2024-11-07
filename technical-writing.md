# 우아한테크코스 오디 안드로이드 팀의 백그라운드 작업 도입기
### 목차
- 안드로이드 백그라운드 작업이란
  - 안드로이드 백그라운드 작업
    - Immediate
    - Long-Running
    - Deferrable
  - 안드로이드 백그라운드 API
    - Service
    - JobScheduler
    - AlarmManager
    - WorkManager
  - 상황에 따른 백그라운드 API 선택하기
- 우리 팀의 이야기
  - 어떤 기능을 백그라운드 작업으로 실행하려고 하는가
  - 왜 WorkManager를 선택했는가
- WorkManager
  - WorkManager의 특징
  - WorkManger의 구성 요소
    - WorkManager
    - Work
    - WorkRequest
  - WorkManager의 동작 방식
- WorkManager의 한계점
  - Foreground Service + AlarmManager 마이그레이션
  - Foreground Service + AlarmManager의 단점 보완하기
- 참고 자료

<br>

## 안드로이드 백그라운드 작업이란
안드로이드에서 백그라운드 작업이란, 앱이 화면에서 보이지 않는 상태에서도 동작하는 작업을 말한다. 대표적인 예로 음악 재생, 파일 다운로드, 위치 정보 수집 등이 있다.  
잘못된 API를 선택하면 앱의 성능이 저하되어 배터리가 소모되고, 사용자 기기 전체의 성능이 저하될 수 있다. 경우에 따라 Play Store에 앱이 등록되지 않을 수도 있다.  
일단 백그라운드 작업의 종류에는 어떤 것이 있는지, 백그라운드 작업을 구현할 수 있는 API 중에는 어떤 것이 있는지 살펴보자.  

<br>

### 안드로이드 백그라운드 작업 종류
<img width="808" alt="image" src="https://github.com/user-attachments/assets/f4e0e105-0902-4024-a256-46a4f35adb4a">

#### Immediate
백그라운드에서 실행되는 작업 중에서도 우선순위가 매우 높은 작업을 말한다. 예를 들어 아래와 같은 작업들이 있다.
- 푸시 알림
- 파일 다운로드
- 데이터 동기화  
이 작업이 백그라운드에서 계속해서 실행될 경우, 배터리 수명과 성능에 영향을 미칠 수 있다.안드로이드에서 제한하고 있는 정책을 적용시키고, 필요한 경우에만 실행되도록 주의해야 한다.  

#### Long-Running
작업을 완료하는데 10분 이상이 걸릴 것으로 예상되는 작업들이다. 예를 들어 용량이 매우 큰 파일을 한 번에 다운로드 해야 하는 작업이다.  

#### Deferrable
Immediate 작업과는 달리 백그라운드에서 실행될 때 사용자에게 직접적으로 영향을 주지 않는 작업들이다.  
사용자에게 영향을 주지 않으므로 우선순위가 낮은 작업들이다. 사용자가 앱을 벗어났을 때만 실행될 수 있는 작업이다.  
예를 들어 데이터베이스에서 데이터를 업데이트하거나, 로그 수집, 데이터 백업 등의 작업들이다.

<br>

### 안드로이드 백그라운드 API
안드로이드에서 제공하는 백그라운드 작업을 수행할 수 있는 API는 여러가지가 있다. 여기서는 가장 일반적으로 사용하는 `Service`, `JobScheduler`, `AlarmManager`, `WorkManager`를 간단히 소개하려고 한다.  

#### Service
Service는 안드로이드 4대 컴포넌트 중 하나다. Service의 종류에는 아래와 같은 세가지가 있다.  
- Foreground Service
  - 사용자에게 백그라운드 작업이 실행 중임을 인지시켜야 한다.
  - 즉, 반드시 사용자에게 Notification을 표시해야 한다.
  - 백그라운드 작업의 우선순위가 높다. 
  - ex: 음악 재생
- Background Service
  - 사용자에게 보이지 않는 작업을 수행한다.
- Bound Service
  - 클라이언트-서버 인터페이스를 제공한다.
  - 하나의 서비스에 여러 컴포넌트(Activity, Fragment 등)이 연결될 수 있다.
  - 앱이 살아있을 때만 실행된다.

Service를 사용하면서 주의할 점은, 기본적으로 안드로이드의 Main Thread (= UI Thread)에서 수행된다는 점이다. ANR 오류를 방지하기 위해서는 Service 내에서 별도의 스레드를 생성해야 한다.  

#### JobScheduler
특정 조건을 만족할 때 백그라운드 작업을 수행할 수 있다. 네트워크 타입, 배터리 충전 상태, 특정 앱의 `ContentProvider` 갱신과 같은 조건을 명시한다.  

#### AlarmManager
특정 시간/기간마다 `Intent`를 실행한다. `Intent`에 예약한 시간에 수행할 작업을 정의할 수 있다. Doze 모드에 제약을 받고, 기기 재부팅 시 예약해둔 작업이 취소된다는 특징이 있다.  
이 때 Doze 모드란, 기기를 오랫동안 사용하지 않는 경우 앱의 백그라운드 CPU 및 네트워크 활동을 지연시켜 배터리 소모를 줄이는 것이다.  

#### WorkManager
지속적인 작업에 권장되는 API다. 앱이 다시 시작되거나 기기가 재부팅될 때 예약된 작업이 사라지지 않고 남아있다.  
대부분의 백그라운드 작업은 지속적인 작업이므로 구글에서 가장 권장하는 백그라운드 처리 API다.  

<br>

### 상황에 따른 백그라운드 API 선택하기
안드로이드 공식 문서에서는 아래와 같이 백그라운드 작업에 고려할 두 가지 시나리오를 정의하고 있다.
- 사용자가 직접 시작한 작업인지
- 특정 이벤트에 대한 트리거 작업인지 

#### 사용자가 직접 시작한 작업인지
<img width="300" alt="image" src="https://github.com/user-attachments/assets/9d049acd-1146-4f30-ae1e-eab0d3af73c4">  

앱이 백그라운드 작업을 실행해야 하고, 사용자가 직접 작업을 시작하는 경우 위 그림과 같은 질문에 대답하며 올바른 API를 선택한다.  

- 앱이 백그라운드에 있는 동안 작업을 계속 실행해야 하는가?  
앱이 백그라운드에 있는 동안 작업을 계속 실행할 필요가 없다면 비동기 작업을 사용한다.  
예를 들어 Kotlin Coroutines와 Java Thread를 사용할 수 있다. 중요한 점은 앱이 종료된 경우에 작업이 중지된다는 것이다.  

- 작업이 지연되거나 중단되면 사용자 경험이 저하되는가?  
작업이 지연되거나 중단되어도 사용자 경험에 영향이 없는 경우가 있다.  
예를 들어 앱에서 데이터를 백업해야 하는 경우 사용자가 백그라운드에서 작업을 수행하고 있다는 것을 알아채지 않아도 된다.  
이러한 경우 `WorkManager`, `JobScheduler`와 같은 백그라운드 작업 API를 사용한다.  

- 짧고 중요한 작업인가?  
작업을 지연할 수 없고 빠르게 완료되는 경우 `shortService` 유형의 Foreground Service를 사용한다.
단, 3분 이내에 백그라운드 작업이 완료되어야 한다.  

- 대체할 API가 있는가?
Foreground Service는 많은 기기 리소스를 사용할 수 있기 때문에, 시스템이 제한을 많이 둔다. 이 경우 대체할 API를 사용한다.  
이 경우에 대해서는 [링크](https://developer.android.com/develop/background-work/background-tasks?hl=ko#is_there_an_alternative_api_just_for_this_purpose)를 참고한다. 


#### 특정 이벤트에 대한 트리거 작업인지 
<img width="300" alt="image" src="https://github.com/user-attachments/assets/40cc6a4a-d74f-4133-94c6-76c41ea519be">  

<br>
<br>

## 우리 팀의 이야기
### 어떤 기능을 백그라운드 작업으로 실행하려고 하는가
<img width="600" alt="image" src="https://github.com/user-attachments/assets/2a73becf-04b8-45c7-a0e0-29353f0b785a">  

백그라운드 작업 도입 과정을 설명하기 전에, 우리 앱은 어떤 앱인지와 어떤 기능을 백그라운드 작업으로 실행하려고 하는지를 말해야 할 것 같다.  
먼저 우리 앱 오디를 한마디로 정의하자면 “원만한 친구 사이를 위한 약속 지킴이 서비스"다.  
앱 기능 중 “도착 현황” 기능이 있다.  


<img width="300" alt="image" src="https://github.com/user-attachments/assets/b6b127be-16f9-43d3-9375-082bba33ac6e">  <img width="300" alt="image" src="https://github.com/user-attachments/assets/cc616d76-4816-449a-8f5c-74082f18792b">  

위 화면처럼, 하나의 약속에 참여한 친구들이 몇시에 도착하는지(`10분 내 도착`, `28분 후 도착`)를 알 수 있고, 지각 여부(`지각 위기`, `지각`, `도착` 등)도 알 수 있다.  
이 기능은 약속 시간 30분 전부터 약속 시간 당일까지 활성화된다. 또한 사용자 위치를 기반으로 도착 예정 시간 등을 계산하고 있다.  
즉, 약속 시간 30분 전부터 사용자의 위치 정보를 서버에게 10초 간격으로 계산해야 했다. 또한 약속에 참여한 모든 사용자의 현황을 확인해야 하기 때문에, 앱을 사용하고 있지 않아도 이 작업이 수행되어야 한다.  

<br>

### 왜 `WorkManager를` 선택했는가
위에서 말했듯이 약속에 참여한 사용자들끼리 하나의 정보를 공유하는 기능이다. 즉, 약속에 참여한 사용자들끼리 정보의 간극이 없는 것이 가장 중요하다.  
`WorkManager는` Doze 모드에 대한 제약이 없고, 기기 재부팅 시에도 예약한 작업이 남아있다. 개발 제약 사항들을 개발자가 신경쓰지 않아도 된다는 점에서 편리하다고 생각했다.  

또한 A 화면에서 작업을 예약하고, B 화면에서 작업의 실행 결과를 화면상에 표시해야 했다. `WorkManager`는 작업의 결과를 `LiveData`와 `Flow`로 반환해주는 기능을 제공한다. `WorkManager`를 사용하면 비동기적으로 UI를 갱신하기에 편리할 것이라 판단했다.  

<br>

## WorkManager
### WorkManager의 특징
- WorkManager는 앱이 종료되거나 기기가 다시 시작되어도 안정적으로 실행되는 비동기 작업을 처리하는데 적합하다.
- 앱이나 기기가 다시 시작되는 경우에도 실행이 보장된다.
- Doze 모드와 같은 절전 기능을 지원한다.
- API 14 이상부터 사용할 수 있다.
- 한 번 또는 반복적으로 작업을 예약할 수 있다.
- 작업에 실패하면 다시 시도하는 백오프 정책을 지원한다.
- [작업 체이닝](https://developer.android.com/topic/libraries/architecture/workmanager/how-to/chain-work)이 가능하여 작업을 순차적으로 처리할 수 있다.

<br>

### WorkManager의 구성요소
#### WorkManager
```kotlin
  WorkManager.getInstance(applicationContext)
```
- 처리해야 하는 작업을 자신의 큐에 넣고 관리한다.  
- 싱글톤으로 구현이 되어있기 때문에 `getInstance()`로 `WorkManager`의 인스턴스를 받아 사용한다.  

#### Work
```kotlin
class EtaWorker(context: Context, workerParameters: WorkerParameters) :
    CoroutineWorker(context, workerParameters) {
    override suspend fun doWork(): Result {
        // 백그라운드에서 수행해야 할 작업을 구현
        return Result.success()
    }
}
```
- 처리해야 하는 백그라운드 작업의 처리 코드를 이 클래스를 상속받아 `doWork()` 메서드를 오버라이드하여 작성한다.
- `doWork()`
  - 작업을 완료하면 `Result`의 값 중 하나를 반환한다. (`Result.success()`, `Result.failure()`, `Result.retry()`)

#### WorkRequest
```kotlin
    // 일회성 작업
    OneTimeWorkRequestBuilder<EtaWorker>()
        .setInputData(inputData) // Worker에게 데이터를 전달할 수 있다.
        .addTag(meetingId.toString()) // Worker에 tag를 부여한다.
        .setInitialDelay(initialDelay, TimeUnit.MILLISECONDS) // Worker가 실행될 때까지의 delay를 설정한다.
        .build()

    // 반복 작업
    PeriodicWorkRequestBuilder<EtaDashboardWorker>(Duration.ofHours(1)) // 생성자 파라미터로 작업 반복 주기를 전달한다.
        .build()
```
- `WorkManager`를 통해 실제 요청하게 될 개별 작업이다.
- 처리해야 할 작업인 `Work`와 작업 반복 여부, 작업 제약 사항 등에 대한 정보가 담겨있다.
- `OneTimeWorkRequest`
  - 한 번만 실행할 작업의 요청을 나타낸다.
- `PeriodicWorkRequest`
  - 여러번 실행할 작업의 요청을 나타낸다.

#### WorkInfo
```kotlin
    val liveData: LiveData<WorkInfo> = workManager.getWorkInfosByTagLiveData(tag)
    val recentWorkInfo =
        liveData
            .filter { it.state == WorkInfo.State.SUCCEEDED }
            .maxByOrNull { it.initialDelayMillis }
    }
```
- `WorkRequest`의 id, 상태 등을 담고 있는 클래스다.
- `BLOCKED`, `CANCELLED`, `ENQUEUED`, `FAILED`, `RUNNING`, `SUCCEEDED`의 6개 State를 가진다.
  - `BLOCKED`: 선행 조건이 완료되지 않아 `WorkRequest`가 차단
  - `CANCELLED`: `WorkRequest`가 취소
  - `ENQUEUED`: `WorkRequest`가 큐에 대기 중
  - `FAILED`: `WorkRequest`가 실패 상태에서 완료
  - `RUNNING`: 현재 `WorkRequest`가 실행 중임
  - `SUCCEEDED`: `WorkRequest`가 성공적으로 완료
 
<br>

### WorkManager의 동작 방식
<img width="800" alt="image" src="https://github.com/user-attachments/assets/9f6fd7db-d8ee-4925-a08a-7ff95d1a5acc">  

1. Worker에 실행되어야 할 작업을 작성한다.  
2. Worker를 가지는 WorkRequest를 생성한다.  
3. WorkManager에 WorkRequest를 `enqueue()`한다.  
4. WorkManager에서 id 또는 tag로 작업 결과를 받아온다.  

<br>

우리 서비스에서는 아래와 같은 Flow로 구성되어 있다.
1. 사용자가 약속에 참여한다.
2. Worker가 약속 시간 30분 전에 수행될 수 있도록 큐에 등록한다.
3. 약속 시간 30분 전애 Worker가 실행되어, 서버와 폴링 방식으로 통신한다.
   3-1. 사용자가 도착 현황 화면에 진입하면, Worker의 가장 최근 결과값을 `LiveData`로 받아온다.
4. 약속 시간이 지나면 Worker를 중단한다.

<br>

## WorkManager의 한계점
WorkManager로 기능을 개발하고 앱을 테스트하며, 예상치 못한 버그가 발생했다. 예약한 시간에 작업이 제대로 수행되지 않았다.
그런데 작업이 수행되지 않는 상황과 되지 않는 상황을 명확히 정의할 수 없었다. 
큐에 여러 작업이 쌓이면, 불규칙적으로 작업이 수행되지 않았다.

<br>
<img width="800" src="https://github.com/user-attachments/assets/5cfea512-492d-4af5-834f-45c9bab29827">

WorkManager에서 스케줄링할 수 있는 작업의 개수를 지정할 수 있는 `setMaxSchedulerLimit()` 함수가 있다.  
[안드로이드 공식 문서](https://developer.android.com/reference/androidx/work/Configuration.Builder#setMaxSchedulerLimit(kotlin.Int))에 따르면, 최대 작업의 개수는 50개다.

<br>

이를 해결하기 위해 아래와 같은 방식을 시도해 보았다. 결과적으로는 모두 실패다.    
한 약속 당 적은 수의 Worker를 생성해서 예약한 시간이 Worker가 실행된다고 해도, 약 30분 동안 폴링 작업을 수행해야 하기 때문에 Worker가 중간에 취소되는 문제가 있었다.   
WorkManager는 신속하게 처리되어야 하거나 일정 주기마다 처리되어야 하는 작업, 몇 분 안에 끝나는 짧은 작업에 적합하도록 설계되어 있다.   
리팩터링 과정은 [노션 페이지](https://sly-face-106.notion.site/eta-cc11715e571d42e5b4153cbcc6d0ca77?pvs=74)에 자세히 정리해 두었으니, 궁금하신 분들은 참고해도 좋다.   

- 한 약속 당 10초 간격의 모든 폴링 작업을 WorkManager 큐에 쌓아둔다.
- 한 약속 당 하나의 Worker를 WorkManager 큐에 쌓아둔다. 하나의 Worker가 실행되면 그 안에서 10초 간격으로 폴링 작업을 수행한다.
- WorkManager와 AlarmManager를 함께 사용한다.

<br>

### Foreground Service + AlarmManager 마이그레이션
결국 해당 기능이 Foreground Service에서 동작할 수 있도록 마이그레이션했다.   
위에서 말했듯, Foreground Service로 수행하는 작업은 우선순위가 높다.   

Foreground Service는 특정 시간에 Service를 시작할 수 있도록 예약하는 기능이 없기 때문에, AlarmManager를 함께 사용했다.   
마이그레이션 후 Flow는 아래와 같다.   
1. 사용자가 약속에 참여한다.
2. AlarmManager에 약속 시간 30분 전 Service 시작 Alarm을 예약한다.
3. AlarmManager에 약속 시간 이후 Service 중단 Alarm을 예약한다.
4. 약속 시간 30분 전에 작업이 Foreground Service가 실행되어, 서버와 폴링 방식으로 통신한다.
5. 약속 시간이 지나면 Foreground Service가 중단된다.

<br>

### Foreground Service + AlarmManager의 단점 보완하기
AlarmManager는 WorkManager에 비해 개발자가 더 고려해야 할 사항들이 많다. 최대한 WorkManager를 사용하는 것과 기능적 차이가 없도록 구현하고자 노력했다.    
AlarmManager의 단점에는 어떤 것이 있고, 어떻게 보완했는지 알아보자.   

<br>

1. **작업 결과를 LiveData로 받아와 비동기적으로 UI 데이터 갱신하기**    

WorkManager에는 Worker가 각각 있고, 이 Worker에 대한 결과를 LiveData나 Flow로 가져올 수 있었다.   
반면에 AlarmManager는 작업 결과를 따로 저장하지 않는다.   
작업이 수행될 때마다 결과를 저장하기 위해, 안드로이드 로컬 데이터베이스 중 하나인 Room을 사용했다.   
```kotlin
@Dao
interface MateEtaInfoDao {
    @Upsert
    suspend fun upsert(mateEtaInfoEntity: MateEtaInfoEntity)

    @Query("SELECT mate_etas FROM eta_info WHERE meetingId = :meetingId")
    fun getMateEtas(meetingId: Long): LiveData<List<MateEta>>
}
```
결과를 저장하고 가져오는 Dao 코드 중 일부다.   
`upsert()`를 통해 하나의 약속에 해당하는 작업 수행 결과를 저장한다.   
`getMateEtas()`를 통해 하나의 약속에 해당하는 작업 수행 결과를 가져온다. 이 때, 반환타입이 LiveData<T> 형태다.   
반환타입을 LiveData나 Flow로 감싸면, Room 내부적으로 비동기적으로 데이터를 갱신한다.  
사용자에게 보여지는 화면에서도 비동기적으로 데이터를 갱신할 수 있었다.  

<br>

2. **로그아웃 시 Alarm을 제거하기**    
로그아웃 시에는 AlarmManager에 예약된 Alarm들이 수행되지 않아야 한다.  
이미 앱에서 로그아웃을 한 상태인데, 로그인 후 이용할 수 있는 기능을 위한 예약을 수행할 필요가 없기 때문이다.   

```kotlin
        fun logout() {
            viewModelScope.launch {
                matesEtaRepository.clearEtaReservation(isReservationPending = true)
            }
        }

        fun loginWithKakao(context: Context) {
            viewModelScope.launch {
                loginRepository.login(context)
                    .onSuccess {
                        // ...
                        matesEtaRepository.reserveAllEtaReservation()
                    }
            }
        }
```
로그아웃 시 AlarmManager에 예약된 Alarm을 모두 삭제했다.  
이후 재로그인 했을 때는 정상적으로 Alarm이 수행되어야 하므로, 완전히 삭제하지는 않고 Room에 저장해둔 Alarm들을 다시 AlarmManager에 등록하는 과정을 거쳤다.  

하지만 이 방식에도 한계점이 있다. 하나의 기기에서 로그아웃 하고 다른 계정으로 다시 로그인했을 때, 이전 계정의 Alarm들이 수행된다는 것이다.  
개선하기 위해서는 서버에 사용자 별 Alarm들을 저장해두고, 로그인 시마다 사용자의 Alarm을 서버에서 받아올 수 있겠다.  
물론 폴링 작업과 백그라운드 예약을 개선하기 위해서는, 서버와 웹 소켓으로 통신하는 방식이 최선일 것이다.  

<br>

3. **탈퇴 시 Alarm을 제거하기**   
로그아웃과 마찬가지로 탈퇴 시에도 AlarmManager에 예약된 Alarm들이 수행되지 않아야 한다.  
하지만 로그아웃과 다른 점은, 재로그인 했을 때도 Alarm이 수행될 수 없다.
탈퇴 시에는 Room에 저장해둔 Alarm까지 삭제하는 방식으로 구현했다.

<br>

# 마무리 하며
안드로이드 백그라운드는 배터리 수명을 최적화하기 위해서 제약 사항이 많다.  
글 초반에 말했듯이 우선순위가 높은 백그라운드 작업과 우선순위가 낮은 백그라운드 작업이 있다.

내가 개발해야 하는 기능은 항상 실행되어야 할, 우선순위가 높은 백그라운드 작업이었다.
하지만 WorkManager는 상황에 따라 지연될 수 있는 백그라운드 작업에 적합한 API다.  
즉, 적절하지 않은 API를 선택했고 그래서 예약한 시간에 작업이 실행되지 않는 문제가 발생했다.
개발 마감일에 쫓기느라 백그라운드 작업 API를 꼼꼼히 비교하지 않고 선택한 점이 아쉽고, 상황에 따라 적절한 백그라운드 API를 사용해야 함을 몸소 느꼈다.

<br>

# 참고 자료
https://developer.android.com/develop/background-work/services?hl=ko  
https://small-stepping.tistory.com/1050  
https://developer.android.com/develop/background-work/background-tasks?hl=ko  
https://developer.android.com/training/monitoring-device-state/doze-standby?hl=ko   
