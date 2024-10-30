# Github Actions와 Docker로 실현하는 지속적 배포

우리는 지속적 배포(Continuous Deployment)를 알아보고 GitHub Actions와 Docker를 사용해 직접 지속적 배포를 해볼 것입니다. 이 글은 Java, 스프링 부트 프로젝트를 기반으로 작성되었습니다.

## 1. 지속적 배포(CD, Continuous Deployment)

### 1.1. 지속적 배포란 무엇인가?

지속적 배포는 소프트웨어 개발 전략 중 하나입니다. 이름에서 알 수 있듯 끊김 없이 배포를 한다는 뜻입니다. 즉, 새로운 기능을 추가하거나 리팩터링하는 등 코드에 변경점이 생기면 변경 사항이 자동으로 배포되는 환경을 뜻합니다. 완성도 있는 지속적 배포는 자동화된 테스트와 빌드를 포함합니다. 이는 새로운 기능을 더 자주, 안정적으로 배포할 수 있게합니다. 그렇다면 왜 지속적 배포를 해야 할까요?

### 1.2. 지속적 배포를 왜 해야 하는가?

지속적 배포를 하면 코드가 변경될 때마다 빠르게 배포됩니다. 그래서 사용자의 피드백을 신속하게 반영할 수 있습니다. 빠른 피드백 사이클은 사용자의 만족도를 높입니다.
지속적 배포 시, 자동화된 테스트와 빌드 과정을 거치기에 오류를 사전에 방지할 수 있습니다. 지속적 배포가 없다면 개발자는 코드가 변경될 때마다 수동으로 패키징하고 배포해야 합니다. 지속적 배포는 개발자가 수동 배포에 할애할 시간을 아껴 핵심 기능 개발에 집중할 수 있게 합니다.

또한, 수동 배포를 하면 개발자가 실수를 하거나 다른 개발자가 배포할 경우 중간 프로세스가 달라질 가능성이 있습니다. 반면에 지속적 배포는 동일한 프로세스가 보장되기 때문에 일관된 품질을 유지할 수 있습니다.

지속적 배포는 소규모 업데이트를 빈번하게 할 수 있게 합니다. 대규모 업데이트를 하면 버그가 발생할 가능성이 높아지고 운영이 중단될 수 있습니다. 소규모 업데이트는 문제를 추적하고 해결하기 더 쉽게 합니다. 게다가 소규모 업데이트 시 문제가 발생하더라도 롤백이나 핫픽스를 빠르게 적용할 수 있습니다.

## 2. GitHub Actions

### 2.1. GitHub Actions란 무엇인가?

GitHub Actions는 빌드, 테스트 및 배포 파이프라인을 자동화할 수 있는 CI/CD(지속적 통합 및 지속적 배포) 플랫폼입니다. GitHub 레파지토리에 이벤트가 발생할 때 특정 일을 자동으로 실행합니다. 예를 들어, 레파지토리에 새 이슈를 만들 때 레이블을 자동으로 추가할 수 있습니다.
GitHub Actions 이외에도 다양한 지속적 배포 플랫폼이 있습니다. 그렇다면 대표적인 플랫폼 중 하나인 Jenkins와 GitHub Actions는 어떻게 다를까요?

### 2.2. GitHub Actions vs Jenkins

Jenkins는 역사가 오래된 오픈 소스 CI/CD 도구입니다. 다양한 플러그인이 있어 확장성이 좋고 복잡한 파이프라인 설정이 편리하고 대규모 프로젝트에 적합한 유연성을 제공합니다. 하지만 서버에 직접 설치하거나 클라우드에서 실행해야 합니다. 간단한 프로젝트임에도 불구하고 사용자가 서버 설정과 유지 보수와 같은 번거로운 작업을 해야 합니다.

GitHub Actions는 GitHub에 내장된 CI/CD 도구로 GitHub 레파지토리와 긴밀하게 통합되어 있습니다. GitHub의 클라우드 인프라에서 실행되기 때문에 별도의 서버 관리나 유지 보수가 필요하지 않습니다. Jenkins만큼 플러그인이 많지 않지만, GitHUb 레파지토리와 통합이 매우 간편합니다. 하지만 복잡한 파이프라인을 설정할만큼 유연하지 않기 때문에 대규모 프로젝트에는 적합하지 않을 수 있습니다.

### 2.3. GitHub Actions 구성요소

<image src="https://docs.github.com/assets/cb-25535/mw-1440/images/help/actions/overview-actions-simple.webp" />

GitHub Actions는 YAML 파일로 작성되며, 크게 Workflow, Event, Job, Step으로 구성됩니다. 하나의 YAML에서 특정 이벤트가 발생할 때 실행될 한 가지 Workflow를 정의할 수 있습니다. 그리고 해당 Workflow를 어떤 이벤트가 발생할 때 실행할지 Event 옵션을 통해 정의합니다. 그리고 하나의 Workflow는 여러개의 Job으로 구성되어 있고, Job은 또 여러개의 Step을 가질 수 있습니다. 자세한 내용은 [Github Actions 공식 문서](https://docs.github.com/ko/actions/about-github-actions/understanding-github-actions)에서 확인할 수 있습니다.

## 3. Docker

### 3.1. Docker란 무엇인가?

Docker는 컨테이너 기반의 오픈소스 플랫폼입니다. 애플리케이션을 개발, 배포, 실행할 수 있는 환경을 제공합니다. Docker를 사용하면 애플리케이션과 실행 환경을 컨테이너라는 독립된 단위로 관리할 수 있습니다. 컨테이너는 필요한 라이브러리, 종속성, 설정 파일 등을 포함하고 있어, 어디서든 동일한 환경에서 실행할 수 있게 돕습니다. 즉, 개발 환경과 실제 프로덕션 환경을 동일하게 유지할 수 있습니다.

Docker는 컨테이너를 띄우는데 컨테이너로 서버를 구조화하면 여러 장점을 갖습니다.

첫째로 격리된 환경을 제공하여 서로 다른 프로그램 간의 충돌을 방지하고, 예상치 못한 문제를 예방할 수 있습니다. 또한, 동일한 도커 이미지를 사용함으로써 일관된 환경을 보장하여 개발, 테스트, 프로덕션 환경에서 동일한 설정을 유지할 수 있습니다. 이를 통해 애플리케이션의 배포와 관리 과정의 효율성을 높일 수 있습니다.
이러한 구조 변경에 필요한 구성 요소로는 먼저 스프링 애플리케이션을 도커 이미지로 만들어야 하는데, 이 때 `Dockerfile`을 사용합니다. `Dockerfile`은 애플리케이션을 컨테이너화하여 동일한 환경에서 실행되게 합니다. 또한, 여러 컨테이너를 관리하고 실행하기 위한 `Docker Compose`를 사용해 단일 명령어로 모든 컨테이너를 시작, 중지, 재시작할 수 있습니다. 마지막으로, CD 파이프라인 파일도 도커 환경에 맞게 조정하여 자동화된 빌드 및 배포 프로세스를 구현할 수 있습니다.

## 4. 이제 직접 해봅시다.

먼저 도커를 사용해서 우리 애플리케이션을 실행할 수 있는 환경을 만들겠습니다. 서버에는 도커가 설치되어 있어야 합니다. 도커 설치는 [도커 공식 문서](https://docs.docker.com/engine/install/)에서 확인할 수 있습니다.

### 4.1. Dockerfile 작성하기

`Dockerfile`은 도커 이미지를 생성하는 명령어를 정의한 스크립트 파일입니다. 애플리케이션을 배포하기 위한 일련의 단계를 코드로 명시해, 누구나 동일한 이미지를 생성할 수 있습니다. 일관된 환경 세팅을 제공해 배포 및 개발 과정의 효율성을 높입니다.
물론 명령어를 직접 사용해 이미지를 생성할 수도 있지만, Dockerfile을 사용하면 git과 같은 버전 관리 시스템을 통해 이미지 생성 과정을 체계적으로 관리할 수 있습니다. 이는 명령어를 읽는 것보다 가독성이 좋고, 관리가 용이하다는 장점을 제공합니다.

```dockerfile
# 베이스 이미지로 사용할 이미지
# 특정 레지스트리를 명시하지 않았기 때문에 Docker 이미지의 기본 공개 저장소인 Docker Hub에서 이미지를 다운로드 합니다.
FROM openjdk:21

# JAR_FILE이라는 이름의 인수 정의
ARG JAR_FILE=build/libs/*.jar
# 정의한 JAR_FILE을 Docker 이미지의 루트 디렉토리(`/`)에 `app.jar`라는 이름으로 복사
COPY ${JAR_FILE} app.jar

# *Docker 컨테이너 사용할 포트 명시(단순 문서화 역할)
EXPOSE 8080

# Docker 컨테이너가 시작될 때 실행할 명령어
ENTRYPOINT ["java", "-Dspring.profiles.active=${SPRING_PROFILE}", "-jar", "/app.jar"]
```

Dockerfile에서 EXPOSE라는 명령어를 사용했습니다.

> The EXPOSE instruction doesn't actually publish the port. It functions as a type of documentation between the person who builds the image and the person who runs the container, about which ports are intended to be published. To publish the port when running the container, use the -p flag on docker run to publish and map one or more ports, or the -P flag to publish all exposed ports and map them to high-order ports.

도커 공식문서에 따르면 `EXPOSE`는 실제로 포트를 개방하는 역할을 하지 않습니다. 이미지 빌더와 이미지를 사용하는 실행자에게 8080 포트가 외부에 공개될 것이라는 걸 문서화할 뿐입니다.
그리고 `ENTRYPOINT`에서 `"-Dspring.profiles.active=${SPRING_PROFILE}"`와 같이 프로덕션 서버, 개발 서버에 따라 달라질 수 있는 스프링 프로필을 환경 변수로 받아 설정할 수 있게 했습니다. 이 환경 변수는 이후에 작성할 `compose.yml`에서 `Dockerfile`로 전달합니다.

### 4.2. compose.yml 작성하기

`Docker Compose`는 여러 컨테이너를 단일 명령어로 쉽게 관리할 수 있도록 도와주는 도구입니다. 이를 통해 모든 컨테이너를 한 번에 시작, 중지, 재시작할 수 있으며, 개발, 테스트, 프로덕션 환경에서 동일한 설정을 사용하여 일관된 환경을 유지할 수 있습니다. Docker Compose를 통해 복잡한 애플리케이션 환경을 간편하게 관리하고, 설정의 일관성을 보장할 수 있습니다. 기존에는 매번 직접 입력하던 명령어를 파일로 관리함으로써 가독성 향상 및 유지보수 편의성까지 제공합니다.

```yml
services: # Docker Compose가 관리할 여러 컨테이너를 정의하는 섹션
  application:
    image: ${BACKEND_APP_IMAGE_NAME} # 사용할 백엔드 애플리케이션 이미지 (환경 변수로 설정됨) -> GitHub Actions에서 넘겨줍니다.
    ports:
      - "8080:8080"
environment: # 4
      TZ: "Asia/Seoul" # 컨테이너 내에서 사용할 시간대 설정
      SPRING_PROFILE: dev # Spring 애플리케이션의 프로파일 설정
    restart: always # 컨테이너가 종료되면 항상 재시작
container_name: {컨테이너 이름} # 5. 컨테이너의 이름 설정
```

- environment

```yml
environment: # 4
  TZ: 'Asia/Seoul' # 컨테이너 내에서 사용할 시간대 설정
```

`environment` 옵션을 통해 컨테이너의 환경 변수를 설정할 수 있습니다. TZ은 컨테이너에서 사용할 시간대를 설정해줍니다. 그리고 SPRING_PROFILE은 우리가 Dockerfile을 작성할 때 `"-Dspring.profiles.active=${SPRING_PROFILE}"`라고 명시해 둔 부분에 적용됩니다.

`container_name` 옵션은 말 그대로 컨테이너의 이름을 지정해주는 옵션입니다. 사실 명시하지 않아도 Docker는 자동으로 생성된 컨테이너 이름을 부여하는데요. 보통 `<프로젝트명>*<서비스명>\_<번호>` 형식으로 부여됩니다. 우리가 compose.yml에서 services의 바로 하단에 application으로 명시했지만 그 이름으로 생성되지는 않습니다. 하지만 `container_name` 옵션으로 명시하면 그 이름 그대로 컨테이너가 생성됩니다.

## 4.3. GitHub Actions CD 파일 작성하기

GitHub Actions 파일은 레파지토리 최상단의 .github 디렉토리 안의 workflows 디렉토리에 YAML 파일로 저장되어야 합니다.
우선 완성된 코드는 다음과 같습니다. 코드를 먼저 보고 사용된 옵션들을 하나씩 살펴보겠습니다.

```yml
name: Backend CD

on:
  workflow_dispatch:
  push:
    branches:
      - dev

jobs:
  build:
name: 🏗️ Build Jar and Upload Docker Image
runs-on: ubuntu-latest

    steps:
      - name: 🏗️ Set up JDK 21
        uses: actions/setup-java@v4
        with:
          distribution: 'corretto'
          java-version: 21

      - name: 🏗️ Set up Gradle
        uses: gradle/actions/setup-gradle@v3

      - name: 🏗️ Build with Gradle
        run: ./gradlew clean bootJar

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Docker Image Build and Push
        uses: docker/build-push-action@v6
        with:
push: true
          tags: ${{ secrets.DOCKER_REPOSITORY_NAME }}:${{ github.sha }}
          platforms: linux/arm64

  deploy:
name: 🚀 Server Deployment
needs: build
    runs-on: [ self-hosted ]
    env:
      BACKEND_APP_IMAGE_NAME: ${{ secrets.DOCKER_REPOSITORY_NAME }}:${{ github.sha }}
      MYSQL_DATABASE: ${{ secrets.MYSQL_DATABASE }}
      MYSQL_ROOT_PASSWORD: ${{ secrets.MYSQL_ROOT_PASSWORD }}
      MYSQL_ROOT_HOST: ${{ secrets.MYSQL_ROOT_HOST }}
      HOST_NAME: 'DEV_SERVER'

    steps:
      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Docker Compose up
        run: docker compose -f compose.dev.yml up -d

      - name: Clean Unused Image
        run: docker image prune -af
```

- name

  Workflow의 이름을 정하는 옵션입니다. name을 지정하지 않아도 워크플로우는 정상적으로 실행됩니다. name 필드는 워크플로우나 job의 가독성을 높이고, GitHub Actions 인터페이스에서 더 쉽게 식별할 수 있도록 돕습니다.

- on

```yml
on:
  workflow_dispatch:
  push:
    branches:
      - dev
```

`on`은 GitHub Actions 워크플로우가 언제 트리거(trigger)될지 정의하는 부분입니다. 특정 조건에 맞는 이벤트가 발생할 때만 워크플로우가 실행되도록 설정할 수 있습니다.
workflow_dispatch는 GitHub Actions 워크플로우를 수동으로 트리거할 수 있게 하는 옵션입니다. 이 옵션을 명시하면 사용자가 GitHub 레포지토리의 `Actions` 탭에서 워크플로우를 직접 실행할 수 있습니다. (단, 메인으로 설정한 브랜치만 해당됩니다.)
push 이벤트는 특정 브랜치에 코드가 푸시될 때 자동으로 트리거됩니다. 이 외에도 여러 이벤트가 있는데 [GitHub Actions 공식 문서 - 워크플로를 트리거하는 이벤트](https://docs.github.com/ko/actions/writing-workflows/choosing-when-your-workflow-runs/events-that-trigger-workflows)에서 확인할 수 있습니다.
branches는 워크플로우가 트리거될 브랜치를 지정하는 부분입니다. 우리 코드에서는 dev 브랜치에 변경사항이 푸시되었을 때만 이 워크플로우가 실행됩니다.

우리 워크플로우는 build와 deploy라는 두 개의 잡을 포함하고 있습니다.
jobs는 여러 job을 포함하는 블록으로, 각 job은 특정 작업을 수행합니다. build는 첫 번째 job의 이름입니다. 이 job은 애플리케이션을 빌드하고 Docker 이미지를 업로드합니다.
name은 job의 이름입니다. runs-on은 이 job이 실행될 운영 체제를 정의합니다. 빌드 과정에서는 굳이 우리 서버를 이용할 필요가 없기 때문에 GitHub에서 제공하는 ubuntu-latest 러너를 사용하도록 지정했습니다.
steps는 이 job 내에서 수행할 단계들의 목록입니다. `uses: actions/checkout@v4`는 GitHub 레포지토리의 코드를 체크아웃하는 액션입니다. 이 액션은 서브모듈을 포함하여 코드를 가져오며, token을 통해 인증을 처리합니다.

name: 🏗️ Set up JDK 21은 Java Development Kit (JDK) 21을 설정하는 단계입니다. actions/setup-java@v4 액션을 사용하여 JDK를 설치합니다. name: 🏗️ Set up Gradle은 Gradle을 설정하는 단계입니다. gradle/actions/setup-gradle@v3 액션을 사용합니다. name: 🏗️ Build with Gradle은 Gradle을 사용하여 애플리케이션을 빌드하는 단계입니다. ./gradlew clean bootJar 명령어를 실행하여 빌드를 수행합니다. name: Login to Docker Hub는 Docker Hub에 로그인하는 단계입니다. docker/login-action@v3 액션을 사용하여 인증 정보를 입력합니다. name: Docker Image Build and Push는 Docker 이미지를 빌드하고 푸시하는 단계입니다. docker/build-push-action@v6 액션을 사용하여 backend 디렉토리에서 이미지를 빌드하고, 푸시합니다.
deploy는 두 번째 잡으로, 서버 배포를 담당합니다. 배포는 우리 서버에서 진행되어야 하므로 runs-on에서 self-hosted runner를 사용하겠습니다.

self-hosted runner 설치 방법은 [GitHub Actions 공식 문서](https://docs.github.com/ko/actions/hosting-your-own-runners/managing-self-hosted-runners/adding-self-hosted-runners)에서 확인할 수 있습니다.

name: Login to Docker Hub는 Docker Hub에 다시 로그인하는 단계입니다. name: Docker Compose up은 Docker Compose를 사용하여 서버를 실행하는 단계입니다. compose.dev.yml 파일을 사용하여 애플리케이션을 실행합니다. name: Clean Unused Image는 사용하지 않는 Docker 이미지를 정리하는 단계입니다. docker image prune -af 명령어를 실행하여 불필요한 이미지를 삭제합니다.

이제, 도커와 관련된 작업을 하겠습니다.

1.  도커허브 로그인

    먼저 우리 이미지를 도커허브에 푸시하기 위해서는 로그인을 해야합니다. docker/login-action@v3를 액션을 사용해서 로그인을 해줍니다. 도커허브에 로그인하기 위해서는 username과 token이 필요합니다. token은 도커허브 홈페이지에서 다음과 같은 순서로 받을 수 있습니다.
    ![image](https://github.com/user-attachments/assets/5879e7ab-99e5-4787-bb9c-a311dfc156a9)
    ![image](https://github.com/user-attachments/assets/47ecab7b-53f1-448f-966a-84cdf67f9908)

2.  도커 이미지 빌드 및 푸시

```yml
  - name: 🐳 Docker Image Build and Push
        uses: docker/build-push-action@v6
        with:
          context: ./backend
          push: true
          tags: ${{ secrets.DOCKER_REPOSITORY_NAME }}:${{ github.sha }}
          platforms: linux/arm64
```

도커 이미지 빌드 및 푸시는 별도 명령어를 사용하지 않고 docker/build-push-action@v6 액션을 사용했습니다. context는 default로 git checkout을 해주기 때문에. 추가 설정이 필요하지 않다면 git checkout을 할 필요가 없습니다.

3. 도커 컴포즈에 전달될 환경 변수 정의

```yml
env:
  BACKEND_APP_IMAGE_NAME: ${{ secrets.DOCKER_REPOSITORY_NAME }}:${{ github.sha }}
```

compose.yml파일에서 application(스프링) 이미지 이름을 환경 변수로 받아서 사용할거라고 했었는데 바로 여기서 설정합니다. 그 환경변수는 다음과 같이 env로 명시해주면 `docker compose up`명령어로 도커 컨테이너들을 띄울 때 전달 됩니다.

4. 도커 컴포즈 업

```yml
- name: 🐳 Docker Compose up
  run: docker compose -f compose.yml up -d
```

이제 드디어 도커 컨테이너를 실행합니다! `-f` 옵션은 도커 컴포즈 파일의 경로를 지정합니다. 기본적으로 compose.yml 파일을 찾아 실행해주지만 프로덕션 서버에서 compose.yml 파일을 사용한다는 것을 명시하기 위해 지정해주었습니다. 그리고 -d 옵션은 컨테이너를 백그라운드 모드로 실행하는 옵션입니다. 즉, 터미널 세션이 종료되거나 터미널을 다른 작업에서 사용하는 것과 무관하게 실행할 수 있게 해주는 옵션입니다.

5. 사용하지 않는 이미지 정리

```yml
- name: 🐳 Clean Unused Image
  run: docker image prune -af
```

다음은 도커에서 사용하지 않는 이미지를 정리해줍니다. 불필요한 이미지를 갖고 있으면 디스크 공간을 차지하기 때문에 삭제해주는 과정입니다. -a 옵션은 모든 사용하지 않는 이미지를 삭제하고, -f는 삭제 작업을 진행하기 전에 사용자 확인을 생략하고 강제로 실행하는 옵션입니다. 만약 중요한 이미지가 있는 경우는 사용하지 않는게 좋습니다.

## 마치며

이제 모든 과정이 끝났습니다. Docker와 GitHub Actions를 이용한 지속적 배포로 여러분의 프로젝트가 더욱 신속하고 안정적으로 성장하길 바랍니다.

### references

- [GitHub Actions 공식 문서](https://docs.github.com/ko/actions/about-github-actions/about-continuous-deployment-with-github-actions)
- [IBM](https://www.ibm.com/topics/continuous-deployment)
- [Docker 공식 문서](https://docs.docker.com/get-started/docker-overview/)
