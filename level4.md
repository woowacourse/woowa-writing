# Docker CI/CD 파이프라인 속도 개선하기
### 목차

1. **문제 정의와 목표**
    
    - 1.1. 생산성을 저해하는 느린 CI
        
    - 1.2. 목표: 빠르고 효율적인 CI 파이프라인 구축
        
2. **개선 전략 1: Docker Layer Cache 활용하기**
    
    - 2.1. Docker 이미지
        
    - 2.2. Docker Layer Cache의 작동 방식
        
    - 2.3. CI/CD 환경
        
    - 2.4. 해결책: BuildKit과 원격 캐시
        
3. **개선 전략 2: 멀티 아키텍처 빌드 최적화**
    
    - 3.1. 멀티 아키텍처 빌드의 함정: QEMU 에뮬레이션
        
    - 3.2. 해결책: 네이티브 러너(Native Runner) 활용
        
    - 3.3. 성능 비교: QEMU vs Native
        
4. **CI/CD 최종 적용 사례**
    
    - 4.1. 최종 워크플로우 구조
        
    - 4.2. 적용 결과 요약
        
    - 4.3. 구조적 이점
        
5. **결론**
## 1. 문제 정의와 목표

### 1.1. 생산성을 저해하는 느린 CI

애플리케이션을 개발하고 배포하는 과정에서 CI/CD 파이프라인을 이용할 수 있습니다. 코드를 변경하고 원격 저장소에 푸시하면, 자동으로 테스트, 빌드, 배포가 진행되는 과정은 개발 생산성을 크게 향상시킵니다.

하지만 이 과정에서  Docker 이미지를 빌드하는 시간이 길어진다면 어떨까요? 저희 팀은 최근 프로젝트에서 새로운 코드를 푸시할 때마다 CI 파이프라인이 완료되기까지 평균 7분에서 9분의 시간을 기다려야 했습니다.

빌드 시간이 길면 다음과 같은 문제가 발생합니다.

- 피드백의 지연: 코드 변경이 정상적으로 빌드되고 테스트를 통과하는지 확인하는 데 오랜 시간이 걸립니다. 수정 사항을 확인하기 위해 9분을 기다리는 과정이 반복되면 개발의 흐름이 끊기고 병목이 발생합니다.
    
- 긴급 대응의 어려움: 운영 환경에서 발생한 긴급한 버그를 수정해야 할 때, 빌드 시간은 치명적인 약점이 됩니다. 1분 1초가 급한 상황에서 빌드에만 10분 가까이 소요된다면, 문제 해결 시간은 그만큼 길어지고 서비스 장애로 인한 피해가 커질 수 있습니다.
    

### 1.2. 목표: 빠르고 효율적인 CI 파이프라인 구축

이러한 문제들을 해결하기 위해 우리는 Docker 이미지 빌드 시간을 단축한다는 목표를 설정했습니다.

이 목표를 달성함으로써 우리는 다음과 같은 효과를 기대할 수 있습니다.

- 신속한 피드백: 코드 변경 결과를 거의 실시간으로 확인하고 빠르게 다음 단계로 나아갈 수 있습니다.
- 개발자 경험 향상: 불필요한 대기 시간을 줄여 개발자들이 코드 작성이라는 본질적인 업무에 더 집중할 수 있는 환경을 제공합니다.
- 안정적이고 빠른 배포: 긴급 상황 발생 시 신속하게 대응하여 서비스 안정성을 높입니다.

## 2. 개선 전략 1: Docker Layer Cache 활용하기

빌드 속도를 개선하는 방법중 하나는 캐시를 이용하는 것입니다. 매번 CI마다 모든 과정을 처음부터 다시 수행하는 것은 비효율적입니다. 따라서 Docker Layer Cache를 이해하고 활용하면 빌드 시간을 단축할 수 있습니다.

#### 2.1 Docker 이미지

Docker의 이미지는 Dockerfile에 명시된 명령어 하나하나가 독립된 레이어로 구성됩니다. 각 레이어는 이전 레이어의 변경 사항을 담고 있습니다.

```
# --- Build stage ---
# 1. 베이스 이미지 설정 (Layer 1)
FROM amazoncorretto:21 AS build

# 2. 작업 디렉토리 설정 (Layer 2)
WORKDIR /workspace

# 3. 전체 소스 복사 (Layer 3)
COPY . .

# 4. gradlew 실행 준비 (Layer 4)
RUN sed -i 's/\r$//' gradlew && chmod +x gradlew

# 5. 애플리케이션 빌드 (Layer 5)
RUN ./gradlew clean bootJar --no-daemon

# 6. 런타임 베이스 이미지 설정 (Layer 6)
FROM amazoncorretto:21-alpine

# 8. 작업 디렉토리 설정 (Layer 7)
WORKDIR /app

# 9. 빌드 결과 복사 (Layer 8)
COPY --from=build /workspace/build/libs/*SNAPSHOT.jar /app/app.jar

# 10. 애플리케이션 실행 설정 (최종 메타데이터)
ENTRYPOINT ["sh","-c","java $JAVA_OPTS -jar /app/app.jar"]
```

이 Dockerfile로 이미지를 빌드하면, docker는 각 명령어를 순서대로 실행하며 레이어를 쌓아올립니다.

#### 2.2 Docker Layer Cache의 작동 방식

Docker는 이미지를 저장할 때 각 레이어를 만들고 그 결과물들을 로컬 캐시에 저장합니다. 그리고 빌드를  실행할때 명령어의 내용이 이전과 완전히 동일하다면 새롭게 레이어를 만드는 대신 캐시에 저장된 레이어를 그대로 재사용합니다.

캐시가 작동하는 조건은 다음과 같습니다.
- RUN: 실행하는 커맨드 문자열이 완전히 동일해야 합니다.
- COPY, ADD: 복사하는 파일의 내용(체크섬)이 동일해야 합니다.

만약 특정 레이어에서 조건이 맞지 않아 Cache Miss가 발생하면 이후의 모든 레이어는 캐시를 사용하지 못합니다.

따라서 Dockerfile 작성시 명령어의 순서가 캐시 효율에 중요합니다. 변경이 잘 일어나지 않는 부분은 위쪽에, 자주 변경되지 않는 부분을 아래쪽에 배치하는 것이 중요합니다.

아래와 같이 COPY 명령어를 먼저 작성하는 것은 캐시 효율에 좋지 않습니다.
```
COPY .. # 소스코드가 바뀌면 캐시가 깨짐
RUN npm install # 위 레이어에서 캐시가 깨졌으므로 npm install을 다시 실행함
```

의존성 파일을 먼저 복사하여 설치하고, 이후에 소스코드를 나중에 복사하는 경우 Cache hit 비율이 올라갑니다
```
**

COPY package*.json ./  # package.json이 변경될 때만 캐시가 깨짐

RUN npm install       # 소스 코드 변경 시에도 이 레이어의 캐시가 재사용됨

COPY . .              # 자주 바뀌는 소스 코드는 맨 마지막에 복사

**
```

### 2.3  CI/CD 환경

대부분의 CI/CD 환경은 보안과 독립성을 위해 매번 깨끗한 가상 환경 작업을 시작합니다. 이는 이전 빌드에서 생성된 Docker의 로컬 캐시가 다음 빌드에서는 존재하지 않는다는 것을 의미합니다. 결국 CI 환경에서는 Dockerfile 순서를 아무리 잘 최적화해도 캐시의 이점을 누릴 수 없습니다.

#### 2.4 해결책: BuildKit과 원격 캐시
BuildKit이 이 문제를 해결하기 위한 기술입니다. BuildKit은 병렬 빌드, 향상된 성능, 그리고 원격 캐시 저장소 기능을 지원하는 빌드 엔진입니다.

BuildKit을 사용하면 캐시를 로컬이 아닌 외부 저장소에 저장할 수 있습니다. 덕분에 깨끗한 가상 환경에서도 캐시를 이용할 수가 있습니다. 

**GitHub Actions에 원격 캐시 적용하기**

GitHub Actions에서는 docker/build-push-action을 사용하여 매우 간단하게 원격 캐시를 설정할 수 있습니다. 가장 대표적인 두 가지 방식은 gha 캐시와 registry 캐시입니다.

**gha** **캐시**: GitHub Actions가 제공하는 캐시 저장소를 활용합니다. 설정이 매우 간편하지만, 저장 용량과 범위에 제한이 있을 수 있습니다.  ****

```
jobs:
  ci:
    if: github.event.pull_request.merged == true
    runs-on: ubuntu-latest
    steps:
      - name: Build and push (with GHA cache when CI)
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: ${{ env.IMAGE }}:dev-latest
        
          # --- GHA 캐시 설정 ---
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

**registry** **캐시**: Docker 이미지를 저장하는 레지스트리 자체를 캐시 저장소로 활용합니다. gha보다 유연하고 강력하며, 다른 CI 환경과도 캐시를 공유할 수 있는 장점이 있습니다.

```
- name: Build and push (with Registry cache)
  uses: docker/build-push-action@v4
  with:
    context: .
    push: true
    tags: ${{ env.IMAGE }}:dev-latest
    # --- Registry 기반 캐시 ---
    cache-from: type=registry,ref=${{ env.IMAGE }}:buildcache
    cache-to: type=registry,ref=${{ env.IMAGE }}:buildcache,mode=max
```


이처럼 원격 캐시를 적용하면, CI 환경에서도 두 번째 빌드부터는 변경되지 않은 레이어들을 캐시에서 가져와 사용하게 되므로, 의존성 설치와 같이 시간이 오래 걸리는 작업들을 건너뛸 수 있습니다. 그 결과, 빌드 시간이 단축될 수 있습니다.

### 2.5. 성능 비교: 원격 캐시 적용 전후

|항목|원격 캐시 미적용|원격 캐시 적용|
|---|---|---|
|첫 빌드 시간|약 8분|약 8분|
|두 번째 빌드 시간 (의존성 변경 없음)|약 8분 (모든 레이어 재빌드)|**약 3분 30초** (의존성 레이어 캐시 재사용)|
|빌드 시간 단축 효과|없음|**약 55% 이상 단축**|
위 표에서 볼 수 있듯이, 원격 캐시를 적용하지 않으면 CI 환경의 특성상 매 빌드마다 처음부터 모든 과정을 다시 실행해야 합니다. 반면, 원격 캐시를 적용하면 두 번째 빌드부터는 변경되지 않은 의존성 관련 레이어를 캐시에서 가져와 재사용하므로, 시간이 오래 걸리는 `gradlew bootJar`와 같은 단계를 건너뛸 수 있습니다. 결과적으로 소스 코드 변경에만 집중하여 빌드 시간을 크게 단축시키는 효과를 얻을 수 있습니다.

### 개선 전략 2:  **##  멀티 아키텍처 빌드 최적화**

최근 개발 환경은 점점 더 다양해지고 있습니다. 따라서 amd64(Intel/AMD)와 arm64 아키텍처를 모두 지원하는 멀티 아키텍처 Docker 이미지를 빌드해야 하는 경우가 많아졌습니다. 하지만 멀티 아키텍처 빌드는 잘못 접근하면 오히려 빌드 시간을 크게 늘리는 주범이 될 수 있습니다.
#### 3.1. 멀티 아키텍처 빌드의 함정: QEMU 에뮬레이션
저희 ec2는 arm64 아키텍처를 사용중입니다. 하지만 docker 이미지를 빌드하는 GitHub Actions의 ubuntu-latest 러너는 amd64 기반입니다. 이 환경에서 arm64 이미지를 빌드하려면 어떻게 해야 할까요? Docker는 QEMU 라는 기술을 사용하여 이 문제를 해결합니다.ㅁ

QEMU는 현재 실행 중인 CPU 아키텍처에서 다른 아키텍처의 명령어를 실행할 수 있도록 실시간으로 번역(에뮬레이션)해주는 소프트웨어입니다. docker/setup-qemu-action은 바로 이 QEMU를 설정해주는 역할을 합니다.

```
- name: Set up QEMU
  uses: docker/setup-qemu-action@v3
  with:
    platforms: arm64

- name: Build and push (QEMU multi-arch)
  uses: docker/build-push-action@v4
  with:
    context: .
    push: true
    tags: ${{ env.IMAGE }}:dev-latest
    platforms: linux/amd64,linux/arm64
    # Registry 캐시 사용(선택)
    cache-from: type=registry,ref=${{ env.IMAGE }}:buildcache
    cache-to: type=registry,ref=${{ env.IMAGE }}:buildcache,mode=max
```


에뮬레이션은 매우 강력한 기능이지만, 치명적인 단점이 있습니다. 바로 성능 저하입니다. 네이티브 CPU에서 명령어를 직접 실행하는 것과 달리, QEMU는 모든 명령어를 한 단계 거쳐 번역해야 하므로 상당한 오버헤드가 발생합니다. 이로 인해 arm64 빌드는 amd64 빌드보다 느려질 수밖에 없습니다.

결국 멀티 아키텍처 빌드의 전체 시간은 가장 느린 arm64 에뮬레이션 빌드가 끝날 때까지 기다려야 하므로, CI 시간은 다시 길어지게 됩니다.

#### 3.2. 해결책: 네이티브 러너(Native Runner) 활용

에뮬레이션을 피하는 방법은 각 아키텍처 환경에 맞는 러너를 구성하는 것입니다. 이는 네이티브하게 빌드하므로 빠른 빌드 속도를 가질 수 있습니다. 즉 arm64 이미지를 빌드할 때 러너의 환경을 arm64로 구성하는 것입니다.

GitHub Actions는 runs-on 속성을 통해 다양한 러너 환경을 선택할 수 있으며, 최근에는 ARM 기반 러너에 대한 지원도 확대되고 있습니다. 다음과 같이 러너의 아키텍처를 arm으로 구성할 수 있습니다.

```
jobs:
  ci:
    runs-on: ubuntu-22.04-arm
```

**

### 3.3. 성능 비교: QEMU vs Native

| 항목          | QEMU 에뮬레이션          | 네이티브 러너 (Matrix)          |
| ----------- | ------------------- | ------------------------- |
| 워크플로우 구조    | 단일 작업에서 모든 아키텍처 빌드  | 아키텍처별로 병렬 작업 실행           |
| amd64 빌드 시간 | 약 5분                | 약 5분                      |
| arm64 빌드 시간 | 약 15분 (에뮬레이션 오버헤드)  | 약 6분 (네이티브 속도)            |
| 총 CI 완료 시간  | 약 15분 (가장 느린 작업 기준) | 약 6분 (병렬 작업 중 가장 느린 것 기준) |

위 표에서 볼 수 있듯이, 네이티브 빌드는 에뮬레이션으로 인한 성능 저하를 원천적으로 제거하여 arm64 빌드 시간을 크게 단축합니다. 전체 CI 완료 시간은 개별 작업 중 가장 오래 걸리는 시간을 따라갑니다. 결과적으로 QEMU 방식 대비 2배 이상의 속도 향상을 기대할 수 있습니다.

### **4. CI/CD 최종 적용 사례**

  
  앞서 살펴본 두 가지 핵심 전략 ― **(1) Docker Build Cache 최적화**와 **(2) 멀티 아키텍처 빌드 가속화** ― 를 실제 GitHub Actions 환경에 통합하여 개선된 CI/CD 파이프라인을 구성했습니다.
#### **4.1. 최종 워크플로우 구조**

아래는 최종적으로 적용한 GitHub Actions 설정의 핵심 구조입니다.

이 워크플로우는 **GHA 캐시**와 **레지스트리 캐시**, **네이티브 러너 병렬 빌드**를 결합하여 효율성을 극대화합니다.
```
name: YaguBogu Backend Release & Deploy (Optimized)

on:
  push:
    branches: [ "main" ]

env:
  IMAGE: yagubogu/yagubogu-backend

jobs:
  build:
    strategy:
      matrix:
        arch: [amd64, arm64]
    runs-on: ${{ matrix.arch == 'arm64' && 'ubuntu-22.04-arm' || 'ubuntu-latest' }}
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up QEMU (only for amd64)
        if: matrix.arch == 'amd64'
        uses: docker/setup-qemu-action@v3

      - name: Set up Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and Push with Cache
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: ${{ env.IMAGE }}:${{ matrix.arch }}-latest
          platforms: linux/${{ matrix.arch }}
          cache-from: type=registry,ref=${{ env.IMAGE }}:buildcache
          cache-to: type=registry,ref=${{ env.IMAGE }}:buildcache,mode=max
```

#### **4.2. 적용 결과 요약**
| **항목**      | **개선 전**     | **개선 후**        | **효과**         |
| ----------- | ------------ | --------------- | -------------- |
| 평균 빌드 시간    | 8분 30초       | **3분 40초**      | 약 **56% 단축**   |
| 캐시 재활용률     | 0% (매번 새 빌드) | **약 80%**       | 의존성 단계 대부분 재사용 |
| ARM 빌드 속도   | 약 14분 (QEMU) | **약 6분 (네이티브)** | 2.3배 향상        |
| 전체 CI 완료 시간 | 15분 이상       | **6분 이내**       | 병렬화 + 캐시 최적화   |
#### **4.3. 구조적 이점**

- **병렬 아키텍처 빌드:** matrix 전략을 통해 amd64 / arm64를 동시에 빌드함으로써 총 소요 시간을 단축했습니다.
    
- **원격 캐시 재사용:** 레지스트리 캐시를 통해 모든 빌드 환경에서 동일한 캐시를 공유할 수 있습니다.
    
- **자동 버전 관리:** 태그 기반 세미버전 관리와 함께, 캐시된 이미지가 자동으로 다음 빌드에 활용됩니다.
    
- **유연한 확장성:** 동일한 구조를 이용해 스테이징·프로덕션 환경 모두에서 동일한 속도로 이미지를 생성할 수 있습니다.
  
  ## 5. 결론

느린 CI 파이프라인은 개발 생산성을 저해하는 주요 원인이었습니다. 이 문제를 해결하기 위해 **Docker 레이어 캐시 최적화**와 **멀티 아키텍처 빌드의 네이티브 처리**라는 두 가지 핵심 전략을 적용했습니다.

먼저, CI 환경의 휘발성으로 인해 활용되지 못했던 Docker 레이어 캐시 문제를 `BuildKit`과 **원격 레지스트리 캐시**를 도입하여 해결했습니다. 이를 통해 의존성 설치와 같이 반복적이고 시간이 많이 소요되는 작업을 건너뛰어 빌드 시간을 단축했습니다.

다음으로, QEMU 에뮬레이션으로 인해 발생했던 멀티 아키텍처 빌드의 성능 저하 문제를 각 아키텍처에 맞는 **네이티브 러너를 병렬로 실행**하는 방식으로 해결했습니다. 이로써 arm64 빌드 속도를 네이티브 수준으로 끌어올리고 전체 CI 완료 시간을 크게 줄일 수 있었습니다.

최종적으로 이 두 전략을 통합하여 CI 파이프라인을 개선한 결과, 평균 빌드 시간은 **8분 30초에서 3분 40초로 약 56% 단축**되었고, 멀티 아키텍처 빌드를 포함한 전체 CI 완료 시간은 **6분 이내**로 줄었습니다. 이러한 개선은 신속한 피드백, 개발자 경험 향상, 그리고 긴급 상황에 대한 빠른 대응 능력을 확보하는 성과로 이어졌습니다.
