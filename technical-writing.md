# Android 인앱 업데이트(In-App Update)

## 1. 인앱 업데이트란 무엇인가

### 인앱 업데이트의 개념

인앱 업데이트(In-App Update)는 사용자가 앱을 사용하는 도중에 Google Play Store로 이동하지 않고, 앱 내부에서 직접 업데이트를 진행할 수 있는 기능입니다. Google Play Core Library를 통해 제공되며, 앱의 최신 버전 유지와 사용자 경험 향상을 동시에 달성할 수 있습니다.

### 기존 업데이트 방식과의 차이

**기존 방식 (Manual Update)**

- 사용자가 직접 Play Store를 열어야 함
- 업데이트 알림을 무시하기 쉬움
- 앱 실행 → Play Store 이동 → 업데이트 → 앱 재실행의 복잡한 과정
- 업데이트 완료율이 낮음

**인앱 업데이트 방식**

- 앱 내에서 업데이트 프로세스가 시작됨
- 개발자가 업데이트 UX를 제어할 수 있음
- 중요한 업데이트를 강제할 수 있음
- 사용자 이탈을 최소화하며 업데이트 완료율 향상

### 인앱 업데이트의 필요성

**보안 취약점 신속 대응**

- 심각한 보안 이슈 발견 시 즉각적인 업데이트 강제 가능
- 사용자 데이터 보호 및 앱 안정성 확보

**서버 API 호환성 유지**

- 백엔드 API 버전 변경 시 클라이언트 강제 업데이트
- 구버전 클라이언트 사용으로 인한 서비스 장애 방지

**사용자 경험 개선**

- 새로운 기능을 빠르게 전파
- 버그 수정을 신속하게 배포
- 앱 실행 시 자연스러운 업데이트 흐름 제공

## 2. 인앱 업데이트의 작동 원리

### Google Play Core Library 소개

Google Play Core Library는 Google Play의 다양한 기능을 앱에 통합할 수 있도록 제공되는 라이브러리입니다. 인앱 업데이트 외에도 Dynamic Feature Module, In-App Review 등의 기능을 포함합니다.

```kotlin
dependencies {
    implementation 'com.google.android.play:app-update:2.1.0'
    implementation 'com.google.android.play:app-update-ktx:2.1.0'
}
```

### 인앱 업데이트 작동 흐름

```
[앱 시작]
    ↓
[AppUpdateManager 초기화]
    ↓
[appUpdateInfo 요청] ──→ Google Play Store와 통신
    ↓                      (패키지명, 현재 버전 코드 전송)
    ↓
[Play Store 응답 수신]
    ↓
[업데이트 정보 분석]
    - 최신 버전 존재 여부
    - 업데이트 타입 허용 여부
    - 업데이트 우선순위
    ↓
[업데이트 플로우 시작] ──→ Play Store UI 오버레이 표시
    ↓                      (다운로드 및 설치 진행)
    ↓
[결과 콜백 수신]
    - RESULT_OK: 업데이트 완료
    - RESULT_CANCELED: 사용자 취소
    - 기타: 에러 발생
```

### AppUpdateManager의 역할

`AppUpdateManager`는 인앱 업데이트의 핵심 컴포넌트로, 다음과 같은 역할을 수행합니다:

- **업데이트 정보 조회**: Play Store에서 최신 버전 정보 가져오기
- **업데이트 플로우 시작**: 사용자에게 업데이트 UI 표시
- **업데이트 상태 모니터링**: 다운로드 진행 상황 추적
- **업데이트 설치 완료**: 다운로드된 업데이트 적용

```kotlin
val appUpdateManager: AppUpdateManager = AppUpdateManagerFactory.create(context)
```

### AppUpdateInfo 클래스 분석

`AppUpdateInfo` 객체는 현재 앱의 업데이트 상태에 대한 모든 정보를 담고 있습니다.

```kotlin
val appUpdateInfoTask: Task<AppUpdateInfo> = appUpdateManager.appUpdateInfo

appUpdateInfoTask.addOnSuccessListener { appUpdateInfo ->
    // 업데이트 가능 여부
    val updateAvailability = appUpdateInfo.updateAvailability()
    
    // 사용 가능한 최신 버전 코드
    val availableVersionCode = appUpdateInfo.availableVersionCode()
    
    // 현재 설치 상태
    val installStatus = appUpdateInfo.installStatus()
    
    // 특정 업데이트 타입 지원 여부
    val isImmediateAllowed = appUpdateInfo.isUpdateTypeAllowed(AppUpdateType.IMMEDIATE)
    val isFlexibleAllowed = appUpdateInfo.isUpdateTypeAllowed(AppUpdateType.FLEXIBLE)
}
```

### 업데이트 가능 여부 확인 로직

- updateAvailability() 상태 값

```kotlin
when (appUpdateInfo.updateAvailability()) {
    UpdateAvailability.UPDATE_AVAILABLE -> {
        // 업데이트 가능: 새 버전이 Play Store에 존재
    }
    
    UpdateAvailability.UPDATE_NOT_AVAILABLE -> {
        // 업데이트 불가: 이미 최신 버전 사용 중
    }
    
    UpdateAvailability.DEVELOPER_TRIGGERED_UPDATE_IN_PROGRESS -> {
        // 강제 업데이트가 진행 중이었으나 중단됨 (앱 재시작 시 재개 필요)
    }
    
    UpdateAvailability.UNKNOWN -> {
        // 알 수 없는 상태 (네트워크 오류 등)
    }
}
```

- isUpdateTypeAllowed() 검증
    - 업데이트가 가능하더라도 특정 업데이트 타입이 허용되는지 확인해야 합니다.

```kotlin
if (appUpdateInfo.updateAvailability() == UpdateAvailability.UPDATE_AVAILABLE) {
    if (appUpdateInfo.isUpdateTypeAllowed(AppUpdateType.IMMEDIATE)) {
        // Immediate 업데이트 가능
    } else if (appUpdateInfo.isUpdateTypeAllowed(AppUpdateType.FLEXIBLE)) {
        // Flexible 업데이트만 가능
    }
}
```

## 3. 인앱 업데이트의 유형

Google Play In-App Update는 두 가지 업데이트 방식을 제공합니다.

### Immediate Update (즉시 업데이트)

- 전체 화면 업데이트 UI가 표시됨
- 업데이트를 완료하기 전까지 앱 사용 불가
- 업데이트가 완료되면 앱이 자동으로 재시작됨
- 사용자가 취소할 경우 앱을 계속 사용할 수 없음 (개발자가 제어 가능)

```
┌─────────────────────┐
│   앱 시작 (onCreate) │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ AppUpdateInfo 조회   │
└──────────┬──────────┘
           │
           ▼
      업데이트 필요?
           │
    ┌──────┴──────┐
    │             │
   YES            NO
    │             │
    ▼             ▼
┌─────────┐  ┌─────────┐
│Immediate│  │  정상    │
│ Update  │  │  실행    │
│  시작    │  └─────────┘
└────┬────┘
     │
     ▼
┌─────────────────────┐
│   전체 화면 업데이트    │
│   다이얼로그 표시       │
└──────────┬──────────┘
           │
           ▼
       사용자 선택
           │
    ┌──────┴──────┐
    │             │
  업데이트         취소
    │             │
    ▼             ▼
┌─────────┐  ┌─────────┐
│ 다운로드  │  │  앱 종료  │
│ 및 설치  │   │  (강제)  │
└────┬────┘  └─────────┘
     │
     ▼
┌─────────────────────┐
│   앱 자동 재시작       │
└─────────────────────┘
```

### Flexible Update (유연한 업데이트)

- 백그라운드에서 업데이트 다운로드
- 다운로드 중에도 앱 사용 가능
- 사용자가 원하는 시점에 설치 가능
- 스낵바 또는 커스텀 UI로 진행 상황 표시

```
┌─────────────────────┐
│   앱 시작 (onCreate) │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ AppUpdateInfo 조회   │
└──────────┬──────────┘
           │
           ▼
      업데이트 필요?
           │
    ┌──────┴──────┐
    │             │
   YES            NO
    │             │
    ▼             ▼
┌─────────┐  ┌─────────┐
│Flexible │  │  정상    │
│ Update  │  │  실행    │
│  시작    │  └─────────┘
└────┬────┘
     │
     ▼
┌─────────────────────┐
│   업데이트 권장        │
│   다이얼로그 표시       │
└──────────┬──────────┘
           │
           ▼
    사용자 선택
           │
    ┌──────┴──────┐
    │             │
  다운로드        나중에
    │             │
    ▼             ▼
┌─────────┐  ┌─────────┐
│ 백그라운드│   │  앱 계속 │
│ 다운로드  │  │  사용    │
└────┬────┘  └─────────┘
     │
     ▼
┌─────────────────────┐
│   앱 사용 가능         │
│   (진행률 표시)        │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│   다운로드 완료 알림    │
└──────────┬──────────┘
           │
           ▼
    사용자 [설치] 클릭
           │
           ▼
┌─────────────────────┐
│    앱 재시작 및 설치    │
└─────────────────────┘
```

## 4. 업데이트 우선순위 관리

### 앱 버전 코드를 활용한 업데이트 전략

Android에서는 `versionCode`를 통해 앱의 버전을 관리합니다. 효과적인 업데이트 전략을 위해서는 체계적인 버전 코드 관리가 필수입니다.

‘야구보구’ 프로젝트에서는 Semantic Versioning 기반 버전 코드를 활용하는 전략을 사용했습니다.

```kotlin
// build.gradle.kts
android {
    defaultConfig {
        // Major.Minor.Patch 형식을 정수로 변환
        // 예: 1.0.1 → 1_00_01
        versionCode = 1_00_01
        versionName = "1.0.1"
    }
}
```

### VersionInfo 데이터 클래스 구현

```kotlin
data class VersionInfo private constructor(
    val major: Int,
    val minor: Int,
    val patch: Int,
) {
    companion object {
        fun of(versionCode: Int): VersionInfo {
		        // 10001 → Major: 1, Minor: 0, Patch: 1
            val major: Int = versionCode / 10000
            val minor: Int = (versionCode % 10000) / 100
            val patch: Int = versionCode % 100
            return VersionInfo(major, minor, patch)
        }
    }
}
```

### 업데이트 타입 결정 로직

현재 버전과 최신 버전을 비교하여 적절한 업데이트 타입을 결정하는 전략을 적용했습니다. 

```kotlin
enum class InAppUpdateType {
    IMMEDIATE, // 업데이트 강제
    FLEXIBLE,  // 업데이트 권장
    NONE,      // 업데이트 불필요
    ;

    companion object {
        fun determine(
            currentVersionInfo: VersionInfo,
            availableVersionInfo: VersionInfo,
        ): InAppUpdateType =
            when {
				// Major 버전이 증가한 경우 → 강제 업데이트
                availableVersionInfo.major > currentVersionInfo.major -> IMMEDIATE
				// Minor 버전이 증가한 경우 → 권장 업데이트
                availableVersionInfo.minor > currentVersionInfo.minor -> FLEXIBLE
				// Patch만 증가하거나 동일한 경우 → 업데이트 불필요
                else -> NONE
            }
    }
}
```

## 5. 인앱 업데이트 테스트

### 내부 앱 공유로 테스트하기

인앱 업데이트 기능은 Google Play Store를 통해서만 동작하므로, 개발 중에 테스트하기 위해서는 내부 앱 공유(Internal App Sharing)를 활용해야 합니다.

1. 테스트 기기에 설치된 앱 버전이 인앱 업데이트를 지원하고 내부 앱 공유 URL을 사용하여 설치되었는지 확인합니다.
2. Play Console 안내에 따라 앱을 내부적으로 공유합니다. 테스트 기기에 이미 설치된 버전 코드보다 높은 버전 코드를 사용하는 앱 버전을 업로드합니다.
3. 테스트 기기에서 업데이트된 앱 버전의 내부 앱 공유 링크를 클릭합니다. 단, 링크를 클릭한 후 표시되는 Google Play 스토어 페이지에서 앱을 설치하지 않습니다.
4. 기기의 앱 검색 창이나 홈 화면에서 앱을 엽니다. 이제 앱에서 업데이트를 사용할 수 있으며 인앱 업데이트 구현을 테스트할 수 있습니다.

### 테스트 시나리오

Immediate Update 테스트

```
준비:
1. 버전 1.0.0 (versionCode: 1_00_00) 빌드 및 내부 앱 공유로 업로드
2. 디바이스에 설치
3. 버전 2.0.0 (versionCode: 2_00_00) 빌드 및 내부 앱 공유로 업로드

테스트 절차:
1. 앱 실행
2. Immediate Update 다이얼로그 표시 확인
3. "업데이트" 버튼 클릭
4. 다운로드 진행 확인
5. 앱 자동 재시작 확인
6. 새 버전(2.0.0)으로 실행되는지 확인
```

Flexible Update 테스트

```
준비:
1. 버전 1.0.0 설치
2. 버전 1.1.0 (versionCode: 1_01_00) 업로드

테스트 절차:
1. 앱 실행
2. Flexible Update 다이얼로그에서 "다운로드" 클릭
3. 앱을 계속 사용하면서 진행률 UI 확인
4. 다운로드 완료 후 스낵바 표시 확인
5. "재시작" 버튼 클릭
6. 앱 재시작 및 새 버전 적용 확인
```

## 6. 참고자료

### 공식 문서

- [Google Play In-App Updates 공식 문서](https://developer.android.com/guide/playcore/in-app-updates)
- [Play Core Library 레퍼런스](https://developer.android.com/reference/com/google/android/play/core/appupdate/package-summary)
