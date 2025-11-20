# CI/CD, 왜 해야 할까? (ft. GitHub Actions로 완성하는 Android 자동 배포)

## CI/CD는 왜 필요할까?

CI/CD(지속적 통합/지속적 전달 또는 지속적 배포)는 소프트웨어 개발 라이프 사이클의 통합, 테스트 및 배포 단계를 자동화하는 방법론입니다. 이 체계적인 접근 방식은 자동화된 파이프라인을 통해 빈번한 코드 변경
사항을 안정적으로 테스트하고 검증하며 프로덕션 환경에 배포할 수 있도록 지원합니다.  
CI/CD의 주요 목표는 개발 작업을 자동화하여 더 나은 소프트웨어를 더 빠르게 제공하는 것입니다. 이를 통해 다음과 같은 상당한 이점을 얻을 수 있습니다.

- **시장 출시 기간 단축 및 운영 효율성 증대**: 반복 작업을 자동화하여 릴리즈 주기를 단축하고, 소프트웨어 제작을 간소화하여 운영 효율성을 높입니다.
- **코드 품질 향상 및 문제점 조기 발견**: 개발자는 코드 변경 사항을 공유 저장소에 하루에 여러 번 정기적으로 커밋하도록 권장되며, 각 커밋은 자동화된 테스트를 실행하여 결함이나 불일치를 식별합니다. 이
  방법론은
  사소한 오류가 개발 후반부에 심각한 문제로 누적되는 것을 방지하고, 소프트웨어 성능에 영향을 미치기 전에 오류를 발견 및 수정하여 더 높은 품질의 소프트웨어를 제공할 수 있도록 합니다.
- **투명성 및 책임성 증대**: CI/CD 파이프라인은 지속적인 피드백을 통해 전체 소프트웨어 개발 프로세스를 비즈니스 측면에서 투명하게 처리하며, 프로젝트 상태를 한눈에 확인하고 책임 소재를 추적할 수
  있습니다.
- **성능 향상**: 성숙한 CI/CD 관행을 활용하는 우수한 DevOps 팀은 코드 커밋부터 프로덕션 배포까지 리드 타임이 단축되고, 코드 배포 빈도가 증가하며, 변경 실패율이 훨씬 낮습니다 (2024 DORA
  보고서 기준).
  <br>

## CI/CD 도구

CI/CD 파이프라인을 지원하는 주요 플랫폼 및 도구는 다음과 같습니다

Jenkins

- 특징: 무료, 방대한 사용자 기반 및 정보. 다양한 OS에 설치 용이, 1,800개 이상의 플러그인을 가진 광범위한 생태계 보유
- 통합 및 환경: Windows, macOS, Linux 지원

Bamboo

- 특징: Atlassian에서 개발, 유료 서비스. JIRA 및 Confluence와 같은 Atlassian 제품과 통합이 우수함
- 통합 및 환경: Windows, macOS, Linux 지원

Travis CI

- 특징: 인터넷 기반 CI 서비스, 간결하고 직관적인 웹 인터페이스. 오픈 소스(공개) 프로젝트에 대한 유료 버전 제공, Github와 통합하여 커밋/푸시 및 풀 요청 이벤트에 CI를 자동 실행
- 통합 및 환경: Github 통합

Circle CI

- 특징: 제한된 크레딧/빌드 시간이 있는 무료 계층 제공. Git Push 시 자동으로 테스트를 실행하고, 문제가 없으면 서버 배포까지 자동 진행하는 기능이 있음
- 통합 및 환경: Linux, macOS, Android, Windows 지원

GitHub Actions

- 특징: CI/CD 파이프라인의 자동화를 지원하는 도구.
- 통합 및 환경: GitHub 통합

GitLab CI

- 특징: 이미 GitLab을 코드 저장소로 사용하는 경우 합리적인 선택
- 통합 및 환경: GitLab 통합

Red Hat OpenShift Pipelines / Tekton

- 특징: 클라우드 공급업체 및 선호하는 개발 전략에 따라 CI/CD 구현을 지원
- 통합 및 환경: Red Hat OpenShift, AWS 통합
  <br><br>

## CI

CI는 개발자들이 만든 새로운 코드나 바뀐 코드를 중앙 저장소에 '자주', 그리고 '자동으로' 합치는 개발 방식입니다.

이것은 개발 방법론인 데브옵스(DevOps) 와 애자일(Agile)의 가장 기본이 되는 단계이며, 코드가 만들어져서 사용자에게 전달되는 과정을 빠르고 매끄럽게 만듭니다.

CI 시스템이 코드를 합치는 과정을 효율적으로 만들기 위해 주로 하는 일은 다음과 같습니다.

- 자주, 조금씩 통합하기
    - 개발자들은 하루에도 여러 번, 작은 단위로 코드 변경 사항을 중앙 저장소(예: Git)에 올립니다.
    - 자주 합칠수록 문제가 생겼을 때 원인을 빠르게 찾을 수 있고, 큰 문제가 쌓이는 것을 막을 수 있습니다.

- 합치는 과정을 자동화하기 (빌드와 테스트)
    - 코드가 저장소에 올라올 때마다, CI 서버(예: Jenkins, GitHub Actions)가 이 변경 사항을 자동으로 감지합니다.
    - 자동으로 코드를 컴파일(빌드)하고, 테스트(단위 테스트, 통합 테스트 등)를 실행하여 코드가 정상적으로 작동하는지 검증합니다.

- 빠른 피드백 받기
    - 자동 빌드나 테스트에서 문제가 발견되면, 개발자에게 즉시 알림이 전송됩니다.
    - 이처럼 오류를 개발 과정 초기에 바로 찾아 고칠 수 있어, 나중에 심각한 버그로 커지는 것을 막아줍니다.

요약: CI는 자주 합치고, 자동으로 검사해서, 문제가 생기면 바로 알려주는 시스템을 구축하는 것입니다.
<br><br>

## CD

CI 파이프라인의 후속 단계인 CD는 자동화를 통해 배포를 처리하며, CI가 선행되어야 합니다.

- **지속적 전달 (Continuous Delivery, CD)**
    - 정의: 프로덕션에 릴리스하기 위한 코드 변경이 자동으로 준비되는 소프트웨어 개발 방식입니다.
    - 자동화 수준: 코드가 자동으로 프로덕션과 유사한 환경으로 이동하여 추가 테스트 및 품질 보증을 받지만, 성공적인 테스트 이후 프로덕션으로 이동하기 위해서는 인간의 개입(수동 승인)이 필요합니다.
    - 목표: 최소한의 노력으로 새 코드를 배포하는 동시에, 라이브 전환 전에 어느 정도의 인간 감독을 허용하는 데 중점을 둡니다.

- **지속적 배포 (Continuous Deployment, CD)**
    - 정의: 자동화 수준이 가장 높은 방식입니다. 코드가 테스트를 통과하면, 프로덕션으로의 배포가 자동으로 이루어지며 인간의 승인이 필요하지 않습니다.
    - 자동화 수준: 코드가 테스트를 통과하면 최종 사용자에게 코드 변경 사항이 자동으로 릴리즈됩니다.
    - 목표: 자동화 수준을 극대화하여 인간의 개입 없이 신속한 릴리즈를 목표로 합니다 (예: 넷플릭스는 이 방식을 통해 하루에 수천 번 코드를 배포).
      <br><br>

## CI/CD와 외부 도구 연동

CI/CD를 구축할 때, 단순히 코드를 빌드하고 배포하는 것만으로는 부족합니다. 개발팀이 상황을 즉시 알고 대응할 수 있도록 외부 협업 도구와 연동하는 것이 중요합니다.

1. 실시간 알림 받기 (Slack, Discord, 이메일)
   목적: 빌드나 배포가 성공했는지 실패했는지 팀 전체에 실시간으로 공유하여, 문제가 발생했을 때 즉시 대응할 수 있도록 합니다.

- Slack
    - GitHub Actions의 전용 액션(slackapi/slack-github-action)을 사용합니다.
    - 메시지에 성공/실패 여부(job.status)와 실행 기록 링크를 포함하여 즉시 확인 가능하도록 합니다.
      <img width="516" height="267" alt="스크린샷 2025-11-12 15 14 48" src="https://github.com/user-attachments/assets/2e0e25f5-8b88-4b5a-a1ae-0e4202c4db86" />

- Discord
    - Webhook URL을 GitHub 시크릿(Secret)에 안전하게 저장한 뒤, curl 명령어 또는 전용 액션을 사용하여 메시지를 보냅니다.
    - 보안을 위해 민감 정보(Webhook URL)는 노출되지 않도록 관리합니다.

<img width="375" height="444" alt="스크린샷 2025-11-12 15 18 52" src="https://github.com/user-attachments/assets/68347ff1-9cf2-4ad1-b9b2-ef6ea7ae6a8d" /> (출처: Bottari 앱)

- 이메일
    - SMTP 기반의 액션(dawidd6/action-send-mail)을 사용합니다.
    - 개발팀 외에도 QA 담당자나 프로젝트 매니저(PO) 등 필요한 분들에게 CI 결과를 직접 전달할 때 유용합니다.

2. 이슈 트래커 연동 (Jira, GitHub Issues)
   목적: 빌드/배포 실패 시 자동으로 할 일(이슈)을 생성하고, 성공 시에는 그 상태를 업데이트하여 품질 관리 프로세스를 자동화합니다.

- Jira
    - REST API를 직접 사용하거나, appleboy/jira-action 같은 전용 액션을 사용합니다.

- GitHub Issues
    - GitHub API를 호출하여 이슈를 생성합니다.
      <br><br>

## Android CI/CD

### GitHub Actions를 활용한 CI

1. 환경 설정 및 코드 체크아웃 (Setup & Checkout): CI를 실행할 환경을 준비하는 단계입니다.
    - 코드 체크아웃: 최신 소스 코드를 가져옵니다. (워크플로의 actions/checkout@v4 단계)
    - 빌드 환경 설정: Android 프로젝트 빌드에 필요한 JDK(Java Development Kit), Android SDK 및 Gradle 환경을 설정합니다. (워크플로의 set up JDK 21 단계)

```yaml
- uses: actions/checkout@v4
- name: set up JDK 21
  uses: actions/setup-java@v4
  with:
    java-version: '21'
    distribution: 'temurin'
    cache: gradle
```

2. 의존성 및 캐시 관리 (Dependencies & Caching): 빌드 속도를 최적화하고 불필요한 다운로드를 줄입니다.
    - Gradle 캐시 복원 및 저장: 이전 CI 실행에서 사용했던 Gradle 의존성 캐시를 복원하여 빌드 시간을 단축합니다. 성공적으로 빌드되면 다음 실행을 위해 새 캐시를 저장합니다. (워크플로의
      Restore Gradle cache 단계)

```yaml
- name: Restore Gradle cache
  uses: actions/cache@v4
  with:
    path: |
      ~/.gradle/caches
      ~/.gradle/wrapper
    key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
    restore-keys: |
      ${{ runner.os }}-gradle-
```

3. 보안 정보 및 환경 변수 설정 (Secrets & Environment): 빌드 시 필요한 민감한 정보(Secret)나 환경 변수를 설정합니다.
    - Gradle 실행 권한 부여: gradlew 스크립트에 실행 권한을 부여합니다. (워크플로의 Grant execute permission for gradlew 단계)
    - 설정 파일 생성: CI 환경 변수에 저장된 BASE_URL 및 google-services.json과 같은 중요한 설정 정보를 프로젝트 파일(.properties 파일 또는 JSON 파일)로 생성하여
      빌드에 사용합니다. (워크플로의 Add Local Properties, Get Google Services JSON 단계)

```yaml
- name: Grant execute permission for gradlew
  run: chmod +x gradlew

- name: Add Local Properties
  run:
    echo "BASE_URL=$BASE_URL" > ./local.properties

- name: Get Google Services JSON
  run:
    echo "$GOOGLE_SERVICES_JSON" > ./app/google-services.json
```

4. 코드 품질 및 정적 분석 (Code Quality & Static Analysis): 코드 스타일 가이드 및 잠재적인 문제를 검사하여 코드 품질을 유지합니다.
    - Ktlint / Lint 검사: Kotlin/Android 프로젝트의 코드 스타일(컨벤션)을 확인하고 잠재적인 버그나 성능 문제를 감지하는 정적 분석 도구를 실행합니다. (워크플로의 Run ktlint
      Check 단계)

```yaml
- name: Run ktlint Check
  run: ./gradlew ktlintCheck
```

5. 자동화된 테스트 (Automated Testing): 코드 변경 사항이 기존 기능을 손상시키지 않는지 자동으로 확인합니다. CI의 핵심 목적 중 하나입니다.
    - 유닛 테스트 실행: 로직 검증을 위해 Gradle의 test 작업을 실행합니다. (워크플로의 Run Unit Tests 단계)
    - 더 복잡한 CI에서는 에뮬레이터를 사용하여 UI 테스트나 계측 테스트(androidTest)를 추가로 실행하기도 합니다

```yaml
- name: Run Unit Tests
  run: ./gradlew test
```

6. 빌드 (Build Artifact Generation): 테스트가 통과되면 실제 배포 가능한 앱 파일을 생성합니다.
    - APK/AAB 생성: 디버그 또는 릴리즈용 APK(Android Package)나 AAB(Android App Bundle) 파일을 생성합니다. (워크플로의 Build Debug APK 단계)

```yaml
- name: Build Debug APK
  run: ./gradlew assembleDebug
```

7. 결과 알림 (Notification): CI 실행 결과를 팀원들에게 자동으로 알려줍니다.
    - Slack 알림: 빌드 성공 또는 실패 여부에 따라 Slack과 같은 메신저에 알림을 보냅니다. 이는 팀이 문제 발생 시 신속하게 대응할 수 있도록 돕습니다. (워크플로의 If Success, Send
      notification on Slack, If Fail, Send notification on Slack 단계)

```yaml
- name: If Success, Send notification on Slack
  if: ${{success()}}
  uses: rtCamp/action-slack-notify@v2
  env:
    SLACK_COLOR: '#60E0C5'
    SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK_URL }}
    SLACK_TITLE: 'Hearit PR Checker have passed ✅'
    MSG_MINIMAL: true
    SLACK_USERNAME: Hearit Android
    SLACK_MESSAGE: 'Hearit Android PR Check Success 🎉'

- name: If Fail, Send notification on Slack
  if: ${{failure()}}
  uses: rtCamp/action-slack-notify@v2
  env:
    SLACK_COLOR: '#FF0000'
    SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK_URL }}
    SLACK_TITLE: 'Hearit PR Checker have failed ❌'
    MSG_MINIMAL: true
    SLACK_USERNAME: Hearit Android
    SLACK_MESSAGE: 'Hearit Android PR Check Failure - Should Check Up'
```

### Firebase App Distribution (CD)

Firebase App Distribution은 모바일 앱의 사전 출시 버전(Pre-release)을 신뢰할 수 있는 테스터 그룹에게 쉽고 빠르게 배포할 수 있도록 도와주는 Google Firebase 플랫폼의
도구입니다. CI/CD 파이프라인에서 빌드가 완료된 앱을 Play Store에 올리기 전에 QA(품질 보증) 목적으로 내부 팀이나 외부 베타 테스터에게 전달할 때 주로 사용됩니다.

구성 요소

- Firebase App ID, 서비스 계정 JSON, 배포 대상 그룹(e.g. qa-team).
- wzieba/firebase-app-distribution GitHub Action 사용.

```yaml
- name: Upload APK to Firebase App Distribution
  uses: wzieba/Firebase-Distribution-Github-Action@v1.7.1
  with:
    appId: ${{ secrets.FIREBASE_APP_ID }}
    serviceCredentialsFileContent: ${{ secrets.CREDENTIAL_FILE_CONTENT }}
    groups: hearit-test
    releaseNotesFile: ./android/app/whatsnew/whatsnew-ko-KR
    file: ./android/app/build/outputs/apk/debug/hEARit-dev-${{ env.VERSION_CODE }}.apk
```

appId: 배포할 Firebase 앱 식별자

- 배포하려는 특정 안드로이드 앱을 Firebase 프로젝트 내에서 식별하는 고유 ID입니다. 이 ID를 통해 Firebase는 업로드된 APK/AAB가 어떤 앱에 속하는지 정확히 알 수 있습니다.

serviceCredentialsFileContent: Firebase API 접근 권한

- GitHub Actions와 같은 외부 시스템이 Firebase 서비스(여기서는 App Distribution API)에 접근하고 파일을 업로드할 수 있도록 인증하는 데 사용되는 서비스 계정 키(Service
  Account Key) 파일의 내용입니다.
- 민감한 정보이므로 파일 자체를 저장하는 대신, 파일의 전체 JSON 내용을 텍스트 형식으로 GitHub Secret (CREDENTIAL_FILE_CONTENT)에 저장하여 사용합니다.

groups: 앱을 받을 테스터 그룹

- 업로드된 앱 빌드를 다운로드할 수 있도록 초대할 테스터 그룹의 이름입니다. Firebase App Distribution에서는 테스터 목록을 미리 그룹으로 묶어 관리할 수 있습니다.

releaseNote: 릴리즈 노트 파일 경로

- 테스터에게 앱 업데이트와 함께 전달할 버전 변경 내용 또는 릴리즈 노트가 포함된 텍스트 파일의 로컬 경로입니다.

- file: 배포할 빌드 파일의 경로

- CI 워크플로에서 최종적으로 빌드된 APK 또는 AAB 파일이 저장된 로컬 파일 시스템 내의 경로입니다.

### Google Play Store 자동 배포

내부 테스트 / 알파 / 프로덕션 트랙에 자동 업로드하여 출시 주기를 단축할 수 있습니다.

구성 요소

- Fastlane (supply) 또는 r0adkll/upload-google-play@v1

필수 설정

- Google Play Developer API 활성화
- 서비스 계정 생성 및 JSON 키 관리
- 앱 패키지명, 트랙 지정(internal/alpha/production)

```yaml
- name: Deploy AAB On Google Play Console
  uses: r0adkll/upload-google-play@v1.1.3
  with:
    serviceAccountJsonPlainText: ${{ secrets.GOOGLE_SERVICE_ACCOUNT_KEY }}
    packageName: com.onair.hearit
    releaseFiles: ./android/app/build/outputs/bundle/release/app-release.aab
    track: alpha
    whatsNewDirectory: ./android/app/whatsnew
```

serviceAccountJsonPlainText: Google Play Developer API 인증

- GitHub Actions가 사용자를 대신하여 Google Play Developer API를 호출하고 앱 파일을 업로드할 수 있도록 인증하는 데 필요한 서비스 계정 키의 내용입니다. Firebase App
  Distribution에서 사용한 serviceCredentialsFileContent와 유사하지만, 이 키는 Google Play Developer Console과의 통신에 사용됩니다.
- 이 값 역시 민감한 정보이므로, JSON 파일 내용 전체를 GitHub Secret (GOOGLE_SERVICE_ACCOUNT_KEY)에 평문(Plain Text)으로 저장하여 사용합니다.

packageName: 앱의 패키지 이름

- Google Play Store에 등록된 안드로이드 앱의 고유 식별자입니다. 프로젝트의 build.gradle 파일에 설정된 applicationId와 동일합니다.

releaseFiles: 배포할 파일의 경로

- 이전 Gradle 빌드 단계(bundleRelease)에서 생성된 릴리즈 AAB(Android App Bundle) 파일의 로컬 경로입니다.
- (예: ./android/app/build/outputs/bundle/release/app-release.aab)

track: 배포할 트랙 지정: Google Play Console에서 앱을 배포할 대상 그룹(트랙)을 지정합니다.
<br>역할: 배포된 앱이 어떤 사용자 그룹에게 공개될지 결정합니다.

- 주요 트랙의 종류:
    - alpha (알파): 내부 테스터나 QA 팀에게 가장 먼저 배포.
    - beta (베타): 더 넓은 범위의 외부 베타 테스터에게 배포.
        - production (프로덕션): 일반 사용자에게 공식적으로 출시. (예: track: alpha)

whatsNewDirectory: 릴리즈 노트 디렉토리
<br>역할: 업로드 시 해당 버전의 릴리즈 노트 내용을 Play Store에 자동으로 등록합니다.

- Play Store에 표시될 새로운 기능/변경 사항(릴리즈 노트)을 담고 있는 텍스트 파일들이 저장된 디렉토리입니다. 이 디렉토리에는 언어 코드별로 파일이 있어야 합니다 (예: whatsnew-ko-KR).
- (예: ./android/app/whatsnew)

## 결론

CI/CD 파이프라인을 GitHub Actions 기반으로 구축하고 Slack, Discord, 이메일, 이슈 트래커와 통합하면 팀은 빌드 품질을 즉각적으로 인지하고, 실패 원인 추적과 이슈 대응을 자동화할 수
있습니다.

Firebase App Distribution과 Google Play 자동 배포를 추가하면, Android 앱 배포 사이클을 완전히 자동화할 수 있습니다.
