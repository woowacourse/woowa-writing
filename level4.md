# **WorkManager로 배우는 안정적인 백그라운드 작업 설계**

## 서론

안드로이드 앱 개발에서는 사용자 몰래 백그라운드에서 실행돼야 하는 작업들이 많다. 예를 들어 정기적으로 서버와 데이터를 동기화하거나, 앱이 완전히 종료되어도 완료되어야 하는 작업들이 있다.이 글에서는 **지연 실행이 필요한 백그라운드 작업**을 안정적으로 처리하는 Jetpack 라이브러리, **WorkManager**를 소개한다. WorkManager가 등장한 배경과 기본 개념, 실제 서비스 적용 사례, 코드 예시, 권장 패턴과 한계를 정리하며 백그라운드 작업을 공부해 보자.

<img width="1400" height="480" alt="Image" src="https://github.com/user-attachments/assets/0d7ef6df-a573-47b4-934a-e3645d57f9ff" />

*(그림 0-1: WorkManager는 OS 버전에 맞는 스케줄러를 자동 선택해 백그라운드 작업을 실행한다. 모든 작업은 내부 DB에 저장되어 앱 재시작이나 기기 재부팅 후에도 지속된다.)*

---

## **1. WorkManager의 필요성과 기본 개념**

### **1.1 왜 WorkManager인가?**

안드로이드 OS는 배터리 절약을 위해 **Doze 모드**와 **백그라운드 실행 제한** 등의 정책을 도입했다.

- **Doze 모드란?** 기기가 오랜 시간 미사용 상태가 되면 시스템이 네트워크 접근과 예약 작업 실행을 **지연**시키거나 한꺼번에 처리해 배터리를 절약하는 기능이다. 즉, 화면이 꺼진 채 장시간 대기하면 앱이 즉시 네트워크를 쓰거나 예약 작업을 곧바로 실행하지 못할 수 있다. Doze 활성 시 일정 간격의 작업도 지연된다.
- **백그라운드 실행 제한:** Android 8.0(Oreo)부터는 앱이 백그라운드에서 임의로 **Service**를 시작하는 것이 제한되었다. 예전처럼 Service나 AlarmManager만으로 백그라운드 작업을 처리하면 **OS 버전이나 제조사**에 따라 동작이 들쭉날쭉하거나 아예 실행되지 않을 수 있다.

이러한 환경에서 **“OS 버전과 기기에 상관없이 일관되고 신뢰성 있게 백그라운드 작업을 처리할 방법”** 이 필요했다. 그 해답으로 제시된 것이 **WorkManager**이다. WorkManager는 Jetpack 아키텍처 컴포넌트로서, 내부적으로 OS 버전에 맞는 최적의 메커니즘을 선택해 작업을 스케줄링한다. 예를 들어 API 23 이상에서는 **JobScheduler**를 이용하고, 그 이하에서는 **AlarmManager**+**BroadcastReceiver** 등을 활용하는 식이다. 개발자는 복잡한 분기 처리 없이 WorkManager에 작업만 요청하면 된다. WorkManager는 모든 예약 작업을 내부 **SQLite DB**에 저장하여 앱 프로세스 종료나 기기 재부팅 후에도 작업을 복구하고 이어서 실행한다. 또한 WorkManager는 Doze 모드 등 **시스템 절전 정책을 자동으로 준수**하므로 개발자가 일일이 대응할 필요가 없다.

무엇보다 WorkManager의 목표는 **지연 가능한 작업의 “실행”** 이다. 즉, 지금 당장이 아니라도 조건이 만족되면 **반드시 실행됨이 보장**되는 것이 핵심이다. 앱이 강제 종료되지 않는 한, 일정이나 조건 때문에 미뤄진 작업이라도 언젠가 실행되어야 할 때 WorkManager를 사용한다. 예를 들어 “사진 업로드”, “주기적인 데이터 동기화” 등이 이에 해당한다. WorkManager는 이러한 작업을 OS 환경에 맞게 **큐(queue)** 에 작업을 넣어 두었다가 적절한 시점에 실행함으로써, 개발자가 다양한 기기 환경에서도 동일한 동작을 얻도록 해준다.

> **※ Tip:** WorkManager는 과거의 FirebaseJobDispatcher나 GcmNetworkManager 등의 백그라운드 작업 API를 대체하는 **권장 솔루션**이다. 최신 프로젝트에서는 WorkManager 사용이 표준시 된다.

### **1.2 WorkManager의 핵심 기능**

- **일회성 vs 주기성 작업:** **일회성(One-Time)** 작업과 **주기적(Periodic)** 작업을 모두 지원한다. 예를 들어 한 번 실행하고 끝낼 작업, 혹은 15분마다 또는 하루 한 번 등 **반복 실행**이 필요한 작업을 선언적으로 예약할 수 있다.
- **유연한 작업 조건(Constraints):** 작업 실행에 필요한 조건을 지정할 수 있다. 네트워크 연결 유형(예: Unmetered Wi-Fi), 충전 중 여부, 배터리 상태(예: 배터리 부족 시 실행 안 함), 기기 유휴 상태(Idle), 저장공간 여유 등 조건을 만족할 때만 작업을 실행하도록 할 수 있다. 조건이 충족될 때까지 WorkManager가 자동으로 실행을 지연시켜 주므로, 개발자가 직접 조건 변화를 감지하지 않아도 된다. 여러 조건을 지정하면 **모두 충족되어야 실행**되며, 실행 중 조건이 깨지면 일시 중단 후 재시도된다.
- **체이닝(Chaining):** 여러 작업을 **순서대로 또는 병렬로 연결**하여 복잡한 흐름을 구성할 수 있다. 예를 들어 “**다운로드 → 가공 → 업로드**”처럼 앞 단계가 **성공해야** 다음 단계가 실행되도록 시퀀스를 정의할 수 있다. WorkManager는 체인으로 연결된 작업들 간에 **의존 관계**를 관리하여, 이전 작업의 **출력 데이터를 다음 작업의 입력으로 전달**하는 것도 자동으로 처리한다. 체이닝은 beginWith()와 then() 메서드 체인을 통해 선언적으로 구현한다.
- **지수 백오프 재시도:** 작업이 실패했을 때 자동 재시도를 설정할 수 있다. 기본 값은 **지수(backoff)** 증가 정책으로, 초기 지연 후 재시도할 때마다 지연 시간이 배로 늘어난다. (또는 선형 증가로 변경 가능) 또한 재시도 간 최소 대기 시간(기본 10초) 등을 설정해 **급작스런 연속 실행**을 피할 수 있다.
- **고유 작업(Unique Work):** 고유 이름을 부여하여 **중복 등록**을 방지하거나, 이미 등록된 작업을 새 요청으로 **대체**하는 등의 정책을 지원한다. 예를 들어 뉴스 동기화 작업에 "SyncWork"라는 고유 이름을 쓰면, enqueue 시 기존 "SyncWork"가 있다면 새로 추가하지 않을지(KEEP), 기존 것을 취소하고 교체할지(REPLACE) 정할 수 있다. 주기적 작업의 경우 enqueueUniquePeriodicWork()로 유사하게 관리한다.
- **Expedited (즉시 실행) 작업:** WorkManager 2.7.0부터 **Expedited Work** 개념이 도입되어, **중요한 작업을 가능한 한 즉시 실행**하도록 요청할 수 있다. Expedited 작업은 “사용자에게 중요하고 몇 분 내로 완료될 짧은 작업”에 적합하며, Android 12+에서는 **Foreground 서비스 없이**도 백그라운드에서 바로 실행된다. 다만 **시스템 자원 할당량(quota)** 을 지켜야 하고, 오래 걸리면 자동으로 일반 작업으로 강등되므로 긴 작업엔 적합하지 않다. Android 12 미만 기기에서는 Expedited로 요청하면 자동으로 Foreground Service를 사용하므로 개발자는 getForegroundInfo() 메서드를 구현해 알림을 제공해야 한다.
- **상태 모니터링 및 LiveData:** WorkManager는 각 작업의 상태나 결과를 조회/관찰할 수 있는 API를 제공한다. WorkInfo 객체를 통해 ENQUEUED(대기 중) / RUNNING(실행 중) / SUCCEEDED(성공) / FAILED(실패) / CANCELLED(취소) 등의 **상태**와 **출력 데이터를 확인**할 수 있다. 특히 WorkManager.getWorkInfoByIdLiveData()나 getWorkInfosForUniqueWorkLiveData()를 사용하면 작업 상태를 **LiveData**로 쉽게 관찰하여 UI에 반영할 수 있다. (예: 작업 진행률을 ProgressBar로 표시, 완료 시 UI 갱신 등)
- **내구성(Persistence):** WorkManager는 요청한 작업과 그 상태를 앱의 내부 DB에 저장해 둔다. 덕분에 앱이 종료되거나 기기가 재부팅되어도 WorkManager 초기화 시 **이전 작업들을 복원**하여 이어서 수행한다. 이러한 내구성으로 **작업 실행 보장**을 달성하며, OS도 WorkManager의 백그라운드 작업을 신뢰할 수 있는 스케줄로 취급한다.

---

## **2. WorkManager 사용 예제와 실무 활용**

이제 WorkManager를 실제 코드에서 어떻게 활용하는지 살펴보자. 간단한 사용 시나리오 두 가지(일회성 작업과 주기적 작업)를 통해 기본적인 사용법과 유의사항을 설명한다. 이어서 실무 환경에서 고려해야 할 점들을 정리한다.

### **2.1 일회성 작업 예제: 앱 위젯 데이터 갱신**

**시나리오:** 사용자의 홈 화면에 있는 **App Widget**에 오늘의 진행률이나 목표 달성 데이터를 표시해야 한다고 가정하자. 위젯은 AppWidgetProvider를 통해 업데이트되는데, AppWidgetProvider는 **BroadcastReceiver**의 일종이라 그 안에서 바로 네트워크나 DB 작업을 하면 ANR 위험이 있다. 또한 위젯 업데이트는 앱이 실행 중이지 않아도 일정 주기로 이루어져야 한다.

**해결:** WorkManager의 **일회성 OneTimeWorkRequest**를 사용하여 필요한 데이터를 백그라운드에서 로딩한 뒤, 완료 시 위젯 UI를 갱신한다. 위젯 업데이트 요청을 받을 때마다 백그라운드 작업을 WorkManager에 enqueue하고, 결과를 LiveData로 관찰하여 위젯 갱신 코드(AppWidgetManager.updateAppWidget())를 호출하는 구조다.

<img width="1536" height="436" alt="Image" src="https://github.com/user-attachments/assets/5cd5b185-0b70-4bd5-97de-befc04bf02dc" />

*(그림 2-1: AppWidgetProvider에서 WorkManager로 일회성 작업을 enqueue하고, Worker에서 데이터 로드 후 LiveData로 결과를 받아 위젯을 업데이트하는 흐름)*

**구현 흐름:**

1. **위젯 브로드캐스트 수신 (AppWidgetProvider.onReceive())** – 위젯이 추가되었거나, 사용자 갱신 액션이 발생하면, onReceive()에서 WorkManager에 백그라운드 작업을 의뢰한다. 이때 위젯 ID 등의 데이터를 WorkRequest의 inputData로 전달할 수 있다.
2. **작업 요청 생성 (OneTimeWorkRequest)** – 실제 백그라운드 로직은 WidgetInfoWorker라는 Worker 또는 CoroutineWorker로 구현하고, 이를 OneTimeWorkRequest로 빌드하여 WorkManager에 **enqueue**한다. 작업에 필요한 입력값이 있다면 setInputData(...)로 설정한다.
3. **백그라운드 작업 실행 (CoroutineWorker.doWork())** – 워커 내부에서 네트워크나 DB를 통해 위젯에 보여줄 데이터를 불러온다. 데이터 로드에 성공하면 Result.success(outputData)로 결과를 반환하고, 실패하면 Result.retry()로 지정하여 자동 재시도하게 할 수 있다. outputData에는 위젯 업데이트에 필요한 데이터를 키-값 쌍으로 담는다.
4. **결과 관찰 및 위젯 UI 갱신** – WorkManager에 저장된 작업의 WorkInfo를 **LiveData**로 관찰하여, 작업 상태가 SUCCEEDED에 도달하면 outputData를 꺼내 위젯 UI를 갱신한다. 이때 observeForever를 사용했다면 업데이트 후 반드시 removeObserver()로 관찰을 해제하여 메모리 누수를 방지한다.

**예시 코드:**

```kotlin
class WidgetInfoWorker(
    appContext: Context,
    params: WorkerParameters
) : CoroutineWorker(appContext, params) {
    override suspend fun doWork(): Result {
        // 예: 오늘의 건강 데이터 로드 (네트워크/DB)
        val data = repository.fetchTodayHealthData()
        return if (data != null) {
            Result.success(
                workDataOf(
                    "KEY_STEPS" to data.steps,
                    "KEY_HEART" to data.heartRate
                )
            )
        } else {
            Result.retry()
        }
    }
}

// 위젯 브로드캐스트 수신 시 실행
val work = OneTimeWorkRequestBuilder<WidgetInfoWorker>()
            .setInputData(workDataOf("WIDGET_ID" to widgetId))
            .build()
WorkManager.getInstance(context).enqueue(work)

// 작업 상태 관찰 후 UI 업데이트
WorkManager.getInstance(context)
    .getWorkInfoByIdLiveData(work.id)
    .observeForever { info ->
        if (info?.state == WorkInfo.State.SUCCEEDED) {
            val steps = info.outputData.getInt("KEY_STEPS", 0)
            updateAppWidgetUI(steps)  // 위젯 UI 갱신
            WorkManager.getInstance(context)
                .getWorkInfoByIdLiveData(work.id)
                .removeObserver(this)  // 관찰자 해제
        }
    }
```

**설계 포인트:** UI 코드(AppWidgetProvider)는 작업을 **의뢰만 하고**, 실제 데이터 로드/처리는 Worker에서 수행함으로써 **UI와 백그라운드 로직을 분리**했다. 입력 및 출력 데이터의 **키 이름을 상수로 관리**하여 여러 클래스에서 일관되게 사용했다. 또한 observeForever를 쓴 경우 반드시 removeObserver로 제거하여야 함을 강조한다. 만약 LifecycleOwner 범위 내에서 관찰할 수 있다면 (예: Activity나 Service) observe(owner, Observer)를 쓰는 것이 더 안전하다.

### **2.2 주기적 작업 예제: 2시간마다 데이터 동기화**

**시나리오:** 사용자 건강 데이터를 2시간마다 서버와 동기화해야 한다고 가정하자. 배터리 소모를 고려해 **Wi-Fi에 연결되고, 배터리가 충분할 때만** 동기화를 실행해야 한다. 앱이 처음 설치된 후에는 바로 동기화하지 말고 15분 정도 있다가 첫 실행되면 좋겠다. 또한 중복 예약을 피하고, 필요 시 작업을 취소할 수 있어야 한다.

**해결:** WorkManager의 **주기적 PeriodicWorkRequest**를 사용한다. 주기적 작업은 enqueueUniquePeriodicWork() 로 등록한다. 이미 같은 이름의 작업이 있다면, **ExistingPeriodicWorkPolicy.KEEP** 을 적용해 중복을 방지한다. Constraints를 사용해 **네트워크=Unmetered(무료 Wi-Fi)**, **배터리 충분** 조건을 걸고, setInitialDelay로 **15분 지연 후 첫 실행**되도록 설정한다. 다음은 코틀린 코드 예시다.

```kotlin
val constraints = Constraints.Builder()
    .setRequiredNetworkType(NetworkType.UNMETERED)    // Wi-Fi 연결 시에만
    .setRequiresBatteryNotLow(true)                  // 배터리가 충분할 때만
    .build()

val periodic = PeriodicWorkRequestBuilder<HealthSyncWorker>(
    2, TimeUnit.HOURS,    // 2시간 간격
    30, TimeUnit.MINUTES  // 유연 실행 윈도우(flex interval)
)
    .setConstraints(constraints)
    .setInitialDelay(15, TimeUnit.MINUTES)
    .build()

WorkManager.getInstance(context).enqueueUniquePeriodicWork(
    "HealthDataSync",      // 고유 작업 이름
    ExistingPeriodicWorkPolicy.KEEP,
    periodic
)
```

위 코드에서는 HealthSyncWorker가 2시간마다 실행되며, 실행 조건을 만족할 때에만 실제로 동작한다. flex interval 30분을 준 것은 2시간 주기 안에서도 최대 30분까지 **유연한 실행 시간**을 두어, 시스템이 약간 앞뒤로 조정할 수 있게 한 것이다. (예: 정확히 120분마다가 아니라 120~150분 사이에 실행될 수 있음) 이렇게 하면 Doze 모드 등의 영향으로 약간 지연되더라도 다음 실행 간격을 조정할 여유가 생긴다.

**주의 사항:**

- **주기 최소 간격 15분:** WorkManager의 PeriodicWorkRequest는 **최소 15분 주기**로 동작하며, 그보다 짧게 설정할 수 없다. 따라서 15분 미만 간격이 필요하면 WorkManager로는 구현할 수 없다. (이 경우 포그라운드 서비스나 정확한 알람 사용을 검토해야 한다.)
- **정확한 실행 시간 보장 없음:** 주기 작업은 지정한 간격마다 정확히 실행되는 것이 아니다. 앞서 설정한 조건 미충족이나 Doze 모드 진입 등의 이유로 **지연될 수 있으며**, 심하면 어떤 주기는 건너뛰고 다음 주기로 넘어갈 수도 있다. 예를 들어 2시간마다 실행이지만 기기가 4시간째 Doze 상태라면, 그 동안의 한 번은 건너뛰고 4시간 후에 한 번 실행될 수 있다. **WorkManager는 일정한 주기보다는 “실행됨”에 초점을 맞춘 모델**임을 기억하자. (정확한 시각에 실행해야 하는 작업은 AlarmManager의 setExact()를 사용해야 한다.)
- **중복 등록 방지:** enqueueUniquePeriodicWork를 사용해 **중복 예약**을 피했다. 이미 "HealthDataSync"라는 이름의 작업이 스케줄된 경우 KEEP 정책으로 새 요청을 버리고 기존 것을 유지한다. (다른 정책으로 REPLACE를 지정하면 기존 작업을 취소하고 새로 등록할 수도 있다.)
- **작업 취소:** 주기 작업을 더 이상 수행할 필요가 없으면 언제든 WorkManager.cancelUniqueWork("HealthDataSync")로 해당 이름의 작업을 취소할 수 있다. 그러면 향후 예약과 재시도가 모두 중단된다. 이처럼 **UniqueWork를 쓰면 취소 관리도 용이**하다.

### **2.3 실무 적용 시 고려사항**

실제 서비스에 WorkManager를 도입할 때는 단순한 예제 외에 OS별 특성, 제조사별 제약, 사용 패턴 등을 고려해야 한다. 다음은 WorkManager를 운영하며 겪는 대표적인 이슈와 해결책, 그리고 모범적인 사용 지침들이다.

1. **OS 및 제조사별 제약 차이:** 안드로이드 순정 OS 외에 제조사 커스텀 OS(Xiaomi, Oppo, 삼성 등)는 백그라운드 작업에 추가 제약을 가하기도 한다. 예를 들어 **일부 제조사 기기는 사용자가 앱을 최근 목록에서 스와이프 제거하면 프로세스를 강제 종료하면서 WorkManager 작업도 모두 취소**해버린다. 또 agressive한 **배터리 절전 모드**에서는 WorkManager 작업이 몇 시간 이상 지연되거나 실행되지 않을 수 있다. 개발자는 이러한 현상이 발생할 수 있음을 사용자에게 안내하여, 필요한 경우 **배터리 최적화 예외**(doze 모드 예외Whitelist)를 설정하게 하거나, 제조사별 전원 관리 메뉴에서 앱을 보호하도록 유도할 수 있다. 실제로 WorkManager 공식 이슈 트래커에도 “중국산 기기에서 WorkManager 주기 작업이 실행 안됨”과 같은 보고가 있으며, **근본적으로 WorkManager만으로는 해결 불가**하다는 답변이 있다. 그러므로 **“DontKillMyApp”** 과 같은 사이트 정보를 참고하여 기기별 예외 설정 방법을 알려주는 것도 방법이다.
2. **앱의 강제 종료(Force Stop) 상황:** WorkManager는 앱 프로세스가 정상 종료되거나 시스템에 의해 kill된 경우 작업을 복구하지만, **사용자가 강제로 앱을 종료(설정에서 “강제 종료” or 최근앱 목록에서 제거)** 하면 예약한 작업도 모두 사라진다. 이는 OS가 해당 앱의 모든 스케줄을 취소해버리기 때문이다. 따라서 사용자 강제 종료 후 다시 앱을 열었을 때 **중요 주기 작업은 재등록**해주는 것이 안전하다. 예를 들어 앱이 실행될 때 WorkManager.getWorkInfosByTag()나 ...ByUniqueWork()로 작업 존재를 확인하고, 없으면 다시 enqueue하는 로직을 넣을 수 있다. (백그라운드 작업이 중요한 앱은 아예 강제 종료되지 않도록 사용자 교육이나 설정 유도도 고민해볼 수 있다.)
3. **Foreground 서비스로의 전환 기준:** WorkManager의 백그라운드 작업은 **제한된 시간** 내에 끝나야 한다. Android 12 기준으로 백그라운드 작업은 수 분 내 완료가 권장되며, 그 이상 걸리면 시스템이 앱에 제재를 가할 수 있다. 따라서 **오래 걸리는 작업**(예: 대용량 파일 업로드, AI 연산 등)은 WorkManager 내부에서 setForegroundAsync()를 호출해 **포그라운드 서비스**로 승격시킬 수 있다. 이렇게 하면 알림 아이콘이 나타나고 작업이 진행 중임을 사용자에게 표시해야 하지만, 백그라운드 제약 시간을 넘어 계속 실행할 수 있다. 사용자에게 진행 상황을 보여줄 필요가 있는 작업 (예: 데이터 동기화 진행률)도 Foreground로 전환하면 좋다. 단, 불필요하게 모든 WorkManager 작업을 Foreground로 만들면 오히려 사용자 경험을 해칠 수 있으므로, **진행 표시나 즉시성 요구가 있는 작업만 선별적으로** 적용한다. (Expedited 작업의 경우 일정 시간 내 완료가 전제이므로 foreground 전환 없이도 즉시 실행 가능하지만, **quota** 초과 시 지연될 수 있음에 유의한다.)
4. **리소스 남용 주의:** WorkManager를 남용하면 오히려 배터리나 자원에 악영향을 줄 수 있다. 예를 들어 5분마다 동기화하도록 WorkManager를 쓰려고 보니 최소 15분 간격 제한이 있어, 이를 우회하려고 여러 OneTimeWork를 체인으로 걸어 무한 루프를 돌리는 것은 바람직하지 않다. 또한 10개의 다른 OneTimeWork를 한꺼번에 enqueue하면 내부적으로도 여러 작업이 대기하게 되므로, **유사한 작업은 가급적 하나의 Worker에서 처리**하고, 필요하면 입력 데이터를 분기하여 처리하는 편이 좋다. WorkManager는 각 작업을 개별 스레드로 처리하므로, DB나 네트워크에 동시에 다량 접근하면 리소스 경합이 일어날 수 있다. 이때는 **태그(Tag)** 를 활용해 같은 태그를 가진 작업은 한 번에 하나만 실행되도록 제어하거나 (WorkManager.enqueueUniqueWork + APPEND와 ExistingWorkPolicy, 혹은 Constraints에 DeviceIdle 사용) 또는 **체이닝**을 통해 순차 실행되게끔 조정한다.
5. **디버깅과 모니터링:** WorkManager는 안드로이드 스튜디오의 **Background Task Inspector**(작업 인스펙터)를 통해 현재 대기 중/실행 중인 작업과 과거 작업 이력 등을 시각적으로 보여준다. 이를 활용하면 어떤 작업이 언제 실행되었고, 어떤 이유로 실패했는지 등을 파악하기 쉽다. 예를 들어 작업 인스펙터에서는 작업별 제약 조건, 현재 상태, 실행 시간, 출력 데이터, 체인 구조 등을 그래프로 확인할 수 있다. 개발 단계에서 이 도구를 사용해 제대로 스케줄링됐는지, 연쇄 실행 흐름이 원하는 대로 이루어지는지 확인하자. 또한 WorkInfo를 통해 작업 진행 상태 변화를 로그로 남기거나, Worker의 onStopped()를 오버라이드하여 작업 중단 시 리소스 해제를 확실히 하는 등의 방식을 취해 **운영 중 모니터링과 장애 대응**을 할 수 있다. 작업 실패가 잦다면 backoff 설정을 조정하고, 특정 시간대에 작업이 몰려있다면 스케줄을 분산시키는 등 튜닝도 가능하다.

---

## **3. WorkManager의 의의와 한계**

이제 WorkManager를 효율적으로 사용하기 위한 권장 패턴과, 알아두어야 할 한계점을 정리해보자. 어떤 경우에 WorkManager를 쓰고, 어떤 경우에 다른 대안을 검토해야 하는지도 함께 다룬다.

### **3.1 권장 패턴 및 모범 사례**

- **고유 작업 이름 부여 및 정책 활용:** 중요 작업에는 가급적 enqueueUniqueWork 또는 enqueueUniquePeriodicWork를 사용하여 **중복 실행을 방지**한다. 고유 이름과 ExistingWorkPolicy를 조합해, 기존 작업 유지(KEEP), 교체(REPLACE), 이어붙이기(APPEND) 중 시나리오에 맞게 선택하자. 예를 들어 사용자가 동기화 버튼을 연속으로 누르면 이전 동기화가 끝난 후 다시 하게 하려면 APPEND 체인을 쓰고, 무조건 최신 것만 수행하려면 REPLACE를 사용할 수 있다.
- **체이닝으로 작업 흐름 구성:** 여러 하위 작업으로 나뉘는 큰 작업은 chain으로 **작업 순서를 선언적**으로 구성하는 것이 관리에 좋다. 예를 들어 파일 다운로드 후 압축 해제, 업로드까지 이어지는 경우 WorkManager.beginWith(A).then(B).then(C).enqueue() 형태로 쓰면 A 작업 성공 시 자동으로 B 실행, B 성공 시 C 실행이 보장된다. 앞 작업이 실패하면 체인 전체가 중단되므로 **오류 전파**도 자연스럽게 이뤄진다. 또한 then() 대신 combine(Array<Work>) 등을 사용하면 병렬 작업 결과를 모아 다음 작업을 실행하는 것도 가능하다.
- **입출력 데이터 명확화:** Worker 간 전달해야 하는 데이터는 Data 객체로 주고받는다. 키 문자열을 상수로 정의하고, 필요한 경우 Data를 직렬화하여 객체 대신 기본 타입으로 보내도록 한다. 출력 Data 크기가 제한(약 10KB 이하 권장)이 있으므로 큰 데이터는 파일이나 DB로 공유하고 경로만 전달하는 편이 좋다.
- **백오프 및 제한 횟수 설정:** 일시적 실패(예: 네트워크 단절)는 Result.retry()로 대응하되, **백오프 정책**을 명시적으로 설정해두자. setBackoffCriteria(BackoffPolicy.EXPONENTIAL, 30, TimeUnit.SECONDS) 식으로 설정하면 기본 값을 덮어쓸 수 있다. 또한 재시도 무한 반복을 방지하려면 Worker 내부에서 재시도 횟수를 체크해 일정 횟수 이상일 경우 Result.failure()로 종료하는 로직도 고려한다.
- **포그라운드 서비스 사용 기준 문서화:** 팀 개발 시 WorkManager 작업 중 어느 경우에 setForegroundAsync()를 써서 포그라운드로 전환할지 기준을 정해두면 좋다. 예를 들어 *“2초 이내 완료되는 작업은 Expedited로 처리하고, 10초를 넘어갈 가능성이 있으면 Foreground 알림을 띄운다”*, *“사용자 동작 직접 유발 작업은 Expedited, 주기 작업은 일반”* 등의 규칙을 정해 일관성을 유지하자.
- **작업 취소와 정리:** WorkManager는 앱이 종료되어도 남아있는 작업이 있을 수 있다. 불필요하게 남은 작업은 cancelAllWorkByTag나 cancelUniqueWork 등으로 적절히 취소해 리소스를 확보한다. 또한 작업 취소 시 ListenableWorker.onStopped()에서 현재 진행 중인 일을 중단하고 열린 리소스를 닫는 처리를 하여 **깨끗하게 정리**하도록 한다.
- **테스트 및 시뮬레이션:** WorkManager용 Testing 라이브러리(androidx.work:work-testing)를 사용하면 가상환경에서 작업을 바로 실행하거나, 일정 시간을 경과시킨 것처럼 시뮬레이션할 수 있다. 이를 활용해 작업 체인의 성공/실패/재시도 시나리오별 동작을 단위 테스트로 검증하자. 예를 들어 Worker의 doWork가 특정 조건에서 실패를 잘 반환하는지, 여러 작업을 UniqueWork로 넣었을 때 정책대로 동작하는지 등을 테스트하면 신뢰도를 높일 수 있다.
- **운영 모니터링:** WorkManager의 WorkInfo를 수집해 통계를 내면 유용하다. 예를 들어 Firebase Crashlytics의 Custom Log나 Analytics를 통해 작업 실패율, 평균 지연 시간, retry 횟수 분포 등을 기록해두면, 특정 버전이나 기기에서 비정상 패턴을 감지할 수 있다. 또한 Android 11부터 추가된 WorkManager.getStopReason() API를 통해 작업이 중단된 이유(예: 요구 조건 변화, 앱 종료 등)를 파악할 수 있으니 활용해보자 .

### **3.2 WorkManager의 한계와 고려해야 할 대안**

아무리 WorkManager가 편리해도, 모든 배경 작업 상황에 만능은 아니다. 아래는 WorkManager로 해결하기 어려운 시나리오와 그에 대한 대안을 정리한 것이다.

- **즉각 실행이 필요한 작업:** WorkManager는 지연 가능한 작업에 적합하며, **정확한 타이밍이나 즉시 실행 보장**을 목표로 하지는 않는다. 사용자 입력에 즉각 반응해야 하는 작업(예: 버튼 클릭 시 파일 업로드 시작) 등은 WorkManager에 맡기지 말고, **Coroutine/Thread** 등을 활용해 곧바로 처리하거나, 꼭 백그라운드 유지가 필요하면 **Foreground Service**로 전환하는 편이 낫다. WorkManager의 Expedited도 100% 즉시성을 보장하진 않으며, 연속 여러 번 쓰면 제약이 생긴다. 따라서 “사용자가 지금 당장 결과를 봐야 하는 일”은 WorkManager보다는 다른 수단을 쓰도록 하자.
- **정확한 시각에 실행해야 하는 작업:** 예를 들어 매일 **정해진 오전 9시**에 알림을 보내야 한다면, WorkManager의 주기 작업으로는 구현이 애매하다. WorkManager 주기는 최소 15분이고, Doze 상황에서는 9시를 훌쩍 넘겨 실행될 수도 있다. 이럴 땐 신뢰성을 조금 포기하더라도 **AlarmManager의 정확 알람**(setExactAndAllowWhileIdle())을 써야 한다. 정확 알람은 배터리 효율이 떨어지므로 남발하면 안 되지만, **캘린더 이벤트나 정시 알림** 등에는 필요한 API다.
- **긴 작업 또는 무기한 지속 작업:** WorkManager로 시작한 작업도 안드로이드의 **백그라운드 실행 제한**을 완전히 벗어나진 못한다. Android 11 기준 백그라운드 작업은 몇 분 내로 완료되어야 하며, 그 이상 장시간 걸릴 경우 시스템이 중단시킬 수 있다. 따라서 예상치 못하게 오래 걸릴 수 있는 일은 WorkManager에서 **부분 작업으로 쪼개 체인**으로 구현하거나, 아예 **Foreground 서비스로 전환**해 지속성을 확보해야 한다. 예를 들어 대용량 파일 100개 업로드는 한 번에 하지 말고, 10개씩 나눠 WorkManager 체인으로 처리하면서 중간중간 완료 상태를 기록하거나, 사용자가 보고 있다면 Foreground로 처리하는 식이다.
- **반복 주기 15분 미만 작업:** 앞서 언급했듯 WorkManager의 Periodic 작업은 15분보다 짧은 간격을 지원하지 않는다. 5분, 1분 또는 초단위 주기 작업은 WorkManager로 구현이 불가능하며, 이런 경우 **Foreground 서비스 + Handler/Timer**로 직접 주기 구현을 하거나 **Exact 알람**을 반복 등록하는 편법이 필요하다. 다만 짧은 간격의 작업은 배터리 소모를 크게 하므로 앱 설계 자체를 재고해 보는 것이 좋다.
- **앱이 완전히 종료된 상황에서의 작업 보장:** WorkManager는 앱 프로세스가 살아있지 않아도 작업을 예약/실행해주지만, **사용자가 앱 정보를 지우거나(데이터 초기화)** 혹은 **앱을 제거**한 경우에는 방법이 없다. 또한 앞서 말한 제조사 강제종료 이슈 등 사용자가 의도적으로 앱을 종료하는 상황에서는 WorkManager도 속수무책이다. 이러한 경우 *“작업 실행 보장”* 약속은 깨질 수 있으므로, **가능한 한 앱을 강제 종료하지 않도록** 안내하거나, 중요한 작업은 **서버 측과 협업**하여 일정 기간 내 수신이 없으면 재요청하도록 설계하는 등 **보완책**이 필요하다.

---

## **맺음말**

WorkManager는 OS 버전과 기기에 상관없이 **지연 실행**이 필요한(Deferrable) 백그라운드 작업을 **보장**하는 표준 도구다. WorkManager는 일회성·주기성 작업, 조건부 실행, 체이닝, 재시도, 고유 이름 관리를 지원해, 앱 종료나 재부팅 이후에도 안정적으로 작업을 이어간다. 실제로 WorkManager는 안드로이드 팀이 권장하는 백그라운드 작업 해법으로, 이전의 여러 API들을 통합한 강력한 라이브러리다.

하지만 **모든 것을 WorkManager로 해결할 수 있는 것은 아니다.** 즉각 반응이 필요한 작업은 여전히 UI 스레드에서 처리하거나 Foreground 서비스가 필요하고, 정확한 시각에 실행될 작업은 AlarmManager가 맡아야 하며, WorkManager 수행 중에도 사용자 경험을 위해 적절히 Foreground 전환을 해주어야 한다. 또한 제조사별 제약이나 사용자 행동에 따른 작업 취소 이슈도 존재한다. 결국 중요한 것은 **작업의 성격을 정확히 파악**하고, WorkManager를 쓸지 말지 결정하는 것이다. WorkManager의 보장성과 편의성이 빛을 발하는 부분에 최대한 활용하고, 한계를 넘는 요구사항은 다른 방식으로 보완하는 것이 현명한 설계다.

백그라운드 작업은 눈에 보이지 않지만, 앱의 신뢰성과 사용자 경험을 동시에 결정하는 핵심 요소다. WorkManager를 통해 이러한 보이지 않는 곳에서도 우리의 앱이 꼼꼼히 일하고 있음을 보여주자. **일관된 경험과 효율적인 자원 활용**, 두 마리 토끼를 잡는 WorkManager 활용으로 한 단계 높은 안드로이드 개발을 달성하기 바란다.

---

## **참고 자료**

1. [Android Developers 공식 문서 – **Task scheduling**](https://developer.android.com/develop/background-work/background-tasks/persistent#:~:text=,with%20a%20server)
2. [Android Developers 공식 문서 – **WorkManager를 사용한 백그라운드 작업 - Kotlin**](https://developer.android.com/codelabs/android-workmanager?hl=ko#0)
3. [Medium 블로그 – **Android WorkManager: Overview, Best Practices, and When to Avoid It**](https://medium.com/@nachare.reena8/android-workmanager-overview-best-practices-and-when-to-avoid-it-5d857977330a)