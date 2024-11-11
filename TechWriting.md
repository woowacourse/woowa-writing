# GitHub Actions를 이용한 프론트엔드 애플리케이션의 CI/CD 구축

## CI/CD의 개념과 필요성

애플리케이션 개발 과정에서 코드 변경은 필연적입니다. 변경된 코드가 실제 서비스에 반영되려면 코드 컨벤션 검토, 기능 테스트, 배포 등 여러 단계를 거쳐야 하는데요. 2주 단위 스프린트처럼 빠른 개발-배포 주기를 반복하다 보면 다음과 같은 문제점들이 발생하게 됩니다.

- 팀 코드 컨벤션과 어긋나는 작업물 발생
- 새로운 컴포넌트와 훅(hook)의 검증을 로컬 환경의 수동 테스트에만 의존
- 수동 배포 과정에서 발생하는 시간 낭비와 휴먼 에러 위험

이러한 문제들을 해결해주는 것이 바로 **CI/CD**입니다. 저는 최근 [우아한테크코스](https://woowacourse.io)에서 참여한 팀 프로젝트 과정에서 ['크루루'라는 리크루팅 관리 플랫폼](https://www.cruru.kr)의 프론트엔드 CI/CD 파이프라인을 구축했습니다. 테스트와 배포 같은 반복적인 수동 작업들을 자동화하고 나니 팀의 개발 생산성이 눈에 띄게 개선되는 것을 실감할 수 있었습니다.

이 글에서는 CI/CD의 개념을 살펴보고, GitHub Actions를 활용해 프론트엔드 애플리케이션의 CI/CD 파이프라인을 구축하는 과정을 다루겠습니다.

### CI/CD란?

CI/CD는 지속적 통합(Continuous Integration)과 지속적 제공/배포(Continous Delivery/Deployment) 개념을 합친 용어입니다. 각 개념을 살펴보면 다음과 같습니다.

![CI/CD Diagram](assets/images/TechWriting/ci-cd-diagram.png)
<span style="font-size:0.9rem;display:block;text-align:center;">(이미지 출처: RedHat 공식 문서 "CI/CD란?")</span>

#### CI(Continuous Integration)

CI는 개발자들의 코드 변경 사항을 지속적으로 통합하는 과정입니다. 코드가 변경될 때마다 자동화된 빌드와 테스트를 실행하여, 언제든 배포 가능한 상태를 유지하는 것이 핵심입니다.

#### CD(Continuous Delivery/Deployment)

CD는 아래의 두 가지 자동화 과정을 포함합니다. 요약하자면, CD는 프로젝트 저장소와 운영 환경에 새로운 코드를 자동으로 반영하는 연속적인 과정이라고 할 수 있습니다.

- **지속적 제공(Continuous Delivery)**: 코드 변경 사항을 저장소에 릴리즈
- **지속적 배포(Continuous Deployment)**: 업데이트된 코드를 운영 환경에 배포

### CI/CD가 왜 필요할까?

기능 브랜치(`feat`)의 코드를 개발 브랜치(`dev`)로 병합하는 과정을 생각해보겠습니다. 이 과정에서는 코드 문법 검사, 기능 검증, 빌드 테스트 등 여러 검증 단계가 필요합니다.

이러한 테스트들을 자동화하고 이를 통과한 코드만 병합되도록 만든다면, 코드의 신뢰성과 애플리케이션의 안정성을 함께 높일 수 있겠지요. 더불어 개발자는 반복적인 검증 작업에서 벗어나 코드 품질 향상에만 집중할 수 있게 됩니다. 이처럼 잘 구축된 CI/CD 파이프라인은 코드 검증부터 배포까지의 자동화를 통해 애플리케이션의 안정성과 개발팀의 생산성을 함께 높여줍니다.

## GitHub Actions 소개

GitHub Actions는 GitHub에서 제공하는 무료 CI/CD 플랫폼입니다. 코드 저장소와 통합된 환경에서 빌드, 테스트, 배포 파이프라인을 만들 수 있도록 도와주지요. 다른 여러 도구들에 비해 GitHub Actions가 가지는 이점은 다음과 같습니다.

1. **원활한 GitHub 통합**: 외부 서비스 연동 없이 GitHub 환경 내에서 완결된 파이프라인 구성
2. **유연한 작업 흐름 설정**: YAML 파일만으로 프로젝트에 최적화된 트리거 조건과 작업 설정 가능
3. **무료 사용**: 개인 및 소규모 팀 프로젝트에 적합한 비용 효율성

GitHub Actions를 효과적으로 활용하기 위한 핵심 개념들을 먼저 살펴보겠습니다. 아래의 다이어그램 이미지를 함께 참고해주세요.

![GitHub Actions Workflow Diagram](assets/images/TechWriting/workflow-diagram.png)
<span style="font-size:0.9rem;display:block;text-align:center;">(이미지 출처 : spacelift.io)</span>

### Workflow

**자동화된 작업의 기본 단위**로, 하나 이상의 Job으로 구성됩니다. GitHub 저장소의 특정 이벤트(예: push, pull request)에 의해 트리거되며, `.github/workflows/` 디렉토리 내의 YAML 파일로 정의됩니다.

하나의 Workflow는 하나 이상의 Job과, Job 내부의 개별 태스크를 의미하는 하나 이상의 Step들로 구성됩니다. Job과 Step의 개념에 대해서는 아래에서 다시 소개해 드리겠습니다.

### Event

**Workflow의 트리거 조건**입니다. 코드 push, pull request 생성, issue 등록 등 GitHub 저장소에서 발생하는 다양한 이벤트를 지정할 수 있습니다.

### Job

**실행되어야 할 각 작업 단계(Step)들의 실행 단위**입니다. 각 Job은 독립된 Runner 인스턴스에서 실행됩니다. 따라서 Job을 구성할 때엔 실행 환경과 의존성을 명시적으로 정의해주셔야 합니다. Job은 다음과 같은 특징을 가집니다:

- 의존성이 없는 Job들은 병렬 실행 가능
- Job 간 의존성 설정을 통해 실행 순서 제어 가능

### Step

**Job 내부의 개별 실행 프로세스**입니다. 쉘 명령어(`npm install`, `npm run build` 등) 또는 Action을 이 단계에 추가하여 실행할 수 있습니다. 기본적으로 하나의 Step이 실패하면 해당 Job이 중단됩니다. 필요하다면 `continue-on-error: true` 옵션으로 이를 방지할 수 있습니다.

### Action

**여러 Step들을 조합하여 만든 작업 단위**입니다. 이렇게 정의한 Action은 여러 Workflow에서 재사용 가능합니다. 프론트엔드의 재사용 가능한 컴포넌트와 유사한 개념으로 접근할 수 있겠습니다.

### Runner

**Workflow를 실행하는 실행 환경**입니다. GitHub에서는 아래의 두 가지 유형을 지원합니다.

- **GitHub-hosted Runner**: GitHub가 제공하는 기본 실행 환경(Ubuntu Linux, Windows, macOS)
- **Self-hosted Runner**: 사용자가 직접 관리하는 실행 환경으로, 더 높은 커스터마이징이 가능

## 프론트엔드 애플리케이션을 위한 CI/CD 파이프라인 설계하기

이제 GitHub Actions를 활용해 프론트엔드 애플리케이션의 CI/CD 파이프라인을 구축해보겠습니다. 앞서 소개한 ['크루루' 서비스](https://www.cruru.kr)의 프론트엔드 애플리케이션 배포 과정을 예시로 하되, 최종 배포 타겟을 AWS 대신 GitHub Pages로 변경하여 파이프라인을 설계할 것입니다.

무언가를 자동화하려면, 우선 자동화할 작업의 흐름을 먼저 정의해야 합니다. 아래는 `fe/develop` 브랜치에서 `fe/main` 브랜치로 코드가 병합되었을 때 작동시킬 CI/CD 파이프라인의 구조입니다.

![프론트엔드 CI-CD 파이프라인 구현 다이어그램](assets/images/TechWriting/cruru-fe-ci-cd-diagram.png)

위의 파이프라인 구조를 풀어서 설명드리자면 다음과 같습니다.

1. `fe/develop` 브랜치에서 `fe/main`으로 코드가 병합됩니다.
2. CI/CD Workflow가 실행됩니다. 우선 코드 실행에 필요한 의존성 설치가 진행됩니다.
3. 단계별 테스트가 자동으로 순차 실행됩니다.
   1. ESLint를 이용한 코드 문법 검사
   2. Jest를 이용한 기능 테스트 (React 기반 애플리케이션일 경우 React Testing Library 포함)
   3. Storybook 빌드를 이용한 컴포넌트 단위 UI 코드 검증
   4. 애플리케이션 전체 빌드 테스트
4. 모든 테스트 통과 시, 빌드 결과물을 Artifact로 저장합니다.
5. 저장된 애플리케이션 빌드 결과물을 GitHub Pages로 배포합니다.

## CI 단계의 Workflow 만들기

이제 정의된 작업 흐름을 GitHub Actions Workflow로 구현해보겠습니다. 먼저 테스트와 빌드를 포함하는 CI 단계부터 시작하겠습니다.

### 1. 실행 조건 명시

```YAML
on:
  push:
    branches:
      - fe/develop
```

트리거 조건을 `on` 키워드로 정의합니다. 여기서는 `fe/develop` 브랜치에 대한 push 이벤트를 감지하도록 설정했습니다.

### 2. Job에 부여할 Runner 인스턴스 설정

```YAML
jobs:
  run-test-pr-opened:
    if: startsWith(github.head_ref, 'fe-')
    runs-on: ubuntu-22.04
    defaults:
      run:
        working-directory: ./frontend
    env:
      API_URL: ${{ secrets.API_URL }}
      API_VERSION: ${{ secrets.API_VERSION }}
```

Job 설정의 주요 구성 요소는 다음과 같습니다.

- **Job 식별자**: `jobs.<Job 이름>` 형태로 지정하며, 이 이름으로 워크플로우 내 의존성 표현과 실행 현황을 추적할 수 있습니다.
- **실행 조건**: `if` 구문으로 Job 실행 조건을 정의합니다. 여기서는 `fe-` 접두어를 가진 브랜치의 PR을 지정했습니다.
- **Runner 설정**: `runs-on`으로 실행할 Runner를 지정합니다. 여기서는 `ubuntu-22.04` 환경의 GitHub-hosted Runner를 사용합니다.
- **기본 설정**: `defaults`로 모든 Step에 적용될 기본 실행 경로입니다. 여기서는 저장소의 `./frontend` 디렉토리를 선택했습니다.
- **환경 변수**: `env`로 애플리케이션 실행에 필요한 환경 변수를 삽입합니다. 여기서 사용한 Secrets의 개념과 사용법에 대해서는 아래의 공식 문서를 참고하세요.
  - [Using secrets in GitHub Actions - GitHub Docs](https://docs.github.com/en/actions/security-for-github-actions/security-guides/using-secrets-in-github-actions)

### 3. CI 실행 환경 설정

```YAML
steps:
  - name: 저장소 checkout
    uses: actions/checkout@v4

  - name: Node.js 셋업
    uses: actions/setup-node@v4
    with:
      node-version: 20.x
```

`steps` 아래에 실행할 작업들을 순차적으로 정의합니다. 각 Step에는 `name` 키워드로 작업의 내용을 표기할 수 있습니다. `name` 키워드를 활용하면, GitHub 저장소의 Actions 탭에서 Workflow 실행 결과를 살펴볼 때 어떤 작업에서 문제가 있었는지 쉽게 알 수 있습니다.

![GitHub 저장소의 Actions 탭 > 개별 Workflow 실행 결과 화면](assets/images/TechWriting/cruru-fe-ci-steps.png)

각 Step에는 Action 또는 명령어를 실행하도록 정의할 수 있습니다. 둘의 차이는 다음과 같습니다.

- **Action 사용**: `uses` 키워드로 GitHub 또는 커뮤니티에서 미리 정의된 Action을 불러와 실행합니다. 저장소 체크아웃, Node.js 환경 설정, 의존성 캐싱 등의 작업을 간편하게 사용할 수 있습니다.

- **명령어 실행**: `run` 키워드로 직접 명령어를 실행합니다. 이 명령어는 앞서 `defaults`로 설정된 Runner 인스턴스의 기본 경로에서 실행됩니다.

### 4. 의존성 캐싱 설정

```YAML
  - uses: actions/cache@v4
    id: npm-cache
    with:
      path: |
        frontend/node_modules
        ~/.npm
      key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}

  - name: 애플리케이션 의존성 항목들 설치
    if: steps.npm-cache.outputs.cache-hit != 'true'
    run: npm ci
```

**프론트엔드 애플리케이션 구동에 필요한 의존성을 캐싱**하는 부분입니다.

Workflow가 실행될 때마다 의존성 설치가 반복된다면 불필요한 시간 낭비가 발생할 것입니다. 만약 의존성에 영향이 없는 코드 변경 사항이라면, `actions/cache@v4` 액션으로 캐싱을 설정하여 의존성 설치 단계를 생략할 수 있습니다.

위의 YAML 코드는 `npm`을 통해 `node_modules` 안에 설치된 패키지들을 `key`로 지정된 식별자와 함께 GitHub 환경의 캐시에 저장시키는 내용입니다. 여기서 `key`의 내용에 주목해주세요. Runner 인스턴스의 OS와 `package-lock.json` 파일에 대한 hash 값이 포함되어 있음을 알 수 있습니다. Runner 구동 환경이나 애플리케이션의 의존성에 변화가 생기지 않는 한, 위의 `key`는 항상 그대로 유지될 것입니다.

CI 실행 단계에서 이전 실행 때와 동일한 `key`가 확인된다면, 의존성 항목들을 설치하는 작업이 GitHub 환경의 캐시로부터 해당 패키지들을 내려받는 과정으로 대체됩니다. 오직 의존성이 변했을 경우에만 `npm ci`로 패키지 재설치 작업이 이루어집니다. 이처럼 간단한 캐싱 설정으로, 반복되는 CI 작업의 시간 효율을 더 높일 수 있는 것이지요.

### 5. 단계별 테스트 추가

이제 본격적인 테스트 실행 단계들을 Workflow에 추가합니다. 또한 마지막으로 빌드된 결과물을 `fe-dev-dist`라는 이름의 Artifact로 업로드하여 후속 Job에서 사용할 수 있도록 합니다.

```YAML
- name: 코드 문법 테스트
  run: npm run lint

- name: 애플리케이션 기능 테스트
  run: npm run test -- --passWithNoTests

- name: Storybook 빌드 테스트
  run: npm run build-storybook

- name: 애플리케이션 빌드
  run: npm run build

- name: 빌드된 파일을 artifact로 업로드
  uses: actions/upload-artifact@v4
  with:
    name: fe-dev-dist
    path: frontend/dist
```

## CD 단계의 Workflow 만들기

AWS와 같은 클라우드 환경에서 CD 단계의 Workflow를 구현하려면 IAM 설정, 네트워크 및 인프라 구축 등 복잡한 사전 작업이 필요합니다. 이 글의 목적은 GitHub Actions를 이용한 CI/CD 구현 예시에 초점을 두고 있으므로, 별도의 인프라 설정 없이 바로 배포가 가능한 GitHub Pages를 예시로 사용하겠습니다.

### 1. 새로운 Job 구성

```YAML
deploy-to-gh-pages:
  needs: run-test-pr-opened
  runs-on: ubuntu-22.04
```

배포를 위한 새로운 Job을 정의합니다. `needs` 키워드를 통해 이전 테스트 Job(`run-test-pr-opened`)의 성공적인 완료를 전제 조건으로 설정합니다.

### 2. 배포 단계별 작업 설정

```YAML
steps:
  - name: 저장소 checkout
    uses: actions/checkout@v4

  - name: 빌드된 파일 다운로드
    uses: actions/download-artifact@v4
    with:
      name: fe-dev-dist
      path: ./dist

  - name: GitHub Pages 배포
    uses: peaceiris/actions-gh-pages@v4
    with:
      github_token: ${{ secrets.GITHUB_TOKEN }}
      publish_dir: ./dist
      publish_branch: gh-pages
```

배포 프로세스는 다음의 세 단계로 진행됩니다.

1. 저장소 체크아웃
2. 이전 Job에서 생성한 빌드 파일(`fe-dev-dist`) 다운로드
3. GitHub Pages 배포 (`peaceiris/actions-gh-pages@v4` 커뮤니티 액션 활용)

이렇게 설정하면 빌드된 파일이 GitHub Pages의 기본 브랜치인 `gh-pages`에 자동으로 배포됩니다.

## CI/CD Workflow 최종본 확인

위와 같은 과정을 거쳐 작성한 CI/CD Workflow YAML 파일의 최종본은 다음과 같습니다. 파일 최상단에 Workflow의 이름을 추가한 부분에 유의해 주세요.

```YAML
name: FE/CI-CD - Development 테스트, 빌드 및 배포

on:
  push:
    branches: - fe/develop

jobs:
  run-test-pr-opened:
    if: startsWith(github.head_ref, 'fe-')
    runs-on: ubuntu-22.04
    defaults:
      run:
        working-directory: ./frontend
    env:
      API_URL: ${{ secrets.API_URL }}
      API_VERSION: ${{ secrets.API_VERSION }}
    steps:
      - name: 저장소 checkout
        uses: actions/checkout@v4

      - name: Node.js 셋업
        uses: actions/setup-node@v4
        with:
          node-version: 20.x

      - uses: actions/cache@v4
        id: npm-cache
        with:
          path: |
            frontend/node_modules
            ~/.npm
          key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}

      - name: 애플리케이션 의존성 항목들 설치
        if: steps.npm-cache.outputs.cache-hit != 'true'
        run: npm ci

      - name: 코드 문법 테스트
        run: npm run lint

      - name: 애플리케이션 기능 테스트
        run: npm run test -- --passWithNoTests

      - name: Storybook 빌드 테스트
        run: npm run build-storybook

      - name: 애플리케이션 빌드
        run: npm run build

      - name: 빌드된 파일을 artifact로 업로드
        uses: actions/upload-artifact@v4
        with:
          name: fe-dev-dist
          path: frontend/dist

  deploy-to-gh-pages:
    needs: run-test-pr-opened
    runs-on: ubuntu-22.04
    steps:
      - name: 저장소 checkout
        uses: actions/checkout@v4

      - name: 빌드된 파일 다운로드
        uses: actions/download-artifact@v4
        with:
          name: fe-dev-dist
          path: ./dist

      - name: GitHub Pages 배포
        uses: peaceiris/actions-gh-pages@v4
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./dist
          publish_branch: gh-pages
```

## CI/CD 실행 결과 확인

이렇게 작성한 YAML 파일을 `./github/workflows/` 경로에 `fe-ci-cd.yaml`로 저장하여 `fe/develop` 브랜치에 올리면 모든 작업이 끝납니다. 이제 `fe/develop` 브랜치에 `push` 이벤트가 발생할 때마다 이 Workflow가 동작하게 될 것입니다.

실행 결과는 프로젝트 브랜치의 Actions 탭에서 아래 스크린샷과 같이 확인하실 수 있습니다.

![크루루 FE CI/CD Workflow 실행 결과 내역](assets/images/TechWriting/cruru-fe-ci-cd-history.png)

## CI/CD 구현이 가져다 준 성과

코드 변화를 감지하여 테스트와 배포를 자동화하는 CI/CD 파이프라인 구현은 ['크루루'](https://www.cruru.kr) 프론트엔드 팀에게 예상 이상의 성과를 안겨주었습니다. 지난 7월 프로젝트 시작 이후 100일 동안 기록된 정량적인 성과는 다음과 같습니다.

- **하루 평균 7.2회, 총 650회 이상**의 코드 병합 테스트 실행
- **총 170회 이상**의 프론트엔드 앱 빌드 및 배포

하지만 CI/CD를 통해 얻은 진정한 가치는 이런 숫자들 너머에 있었습니다. 팀의 코드 컨벤션이 일관되게 유지되어 유지보수가 수월해졌고, 예기치 않은 빌드-배포 실수가 줄어들면서 안정적인 개발 사이클이 자리잡았습니다. 덕분에 모든 팀원이 더 나은 제품을 만드는 데에만 집중할 수 있게 되었죠. **협업 과정의 실수를 줄이고 효율을 높이는 것.** 이것이 우리가 CI/CD를 통해 얻은 진정한 성과였습니다.

## 참고자료

- [CI/CD란? - RedHat 공식 문서](https://www.redhat.com/ko/topics/devops/what-is-ci-cd)
- [GitHub Actions Tutorial: Getting Started & Examples - spacelift.io](https://spacelift.io/blog/github-actions-tutorial)
- [Workflow file YAML Syntax - korgithub.com](https://www.korgithub.com/Ch4.GitHub%20Actions/02.workflow/02.workflow_yaml_syntax.html)
- [Using secrets in GitHub Actions - GitHub Docs](https://docs.github.com/en/actions/security-for-github-actions/security-guides/using-secrets-in-github-actions)
- [워크플로 속도를 높이기 위한 종속성 캐싱 - GitHub Docs](https://docs.github.com/ko/actions/writing-workflows/choosing-what-your-workflow-does/caching-dependencies-to-speed-up-workflows)
