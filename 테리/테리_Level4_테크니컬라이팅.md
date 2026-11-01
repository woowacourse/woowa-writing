# SSH 없이 EC2에 배포하기

저희 팀은 소규모 독서 모임을 위한 온라인 교환 독서 서비스 ‘여백’을 만들었습니다. 초기 배포를 구성하면서 GitHub Actions로 빌드부터 배포까지 이어 보려고 했는데요. 빌드도 되고 Docker Hub에 이미지도 잘 올라가는데, 정작 서버에 접속하는 단계에서 막혔습니다.

`ssh: connect to host *** port 22: Connection timed out`

우아한테크코스 네트워크에서 접속할 때는 문제가 없던 서버였습니다. 같은 서버에 같은 명령을 실행하려는데, GitHub Actions에서는 응답이 없었습니다. 서버의 SSH 접근을 특정 IP로 제한해 두었기 때문입니다.

저희는 SSH 접근 범위를 넓히는 대신 EC2에 GitHub Actions 러너를 설치했습니다. 이 방식을 고른 이유와 설치하면서 겪은 오류를 정리했습니다. EC2 배포와 GitHub Actions, Docker 이미지 빌드를 한 번씩 해 보셨다면 읽기 수월할 것 같습니다.

## 우테코에서는 되던 SSH 접속

당시에는 EC2 한 대에서 Spring Boot 애플리케이션과 MySQL을 실행하고 있었습니다. 컨테이너는 Docker Compose로 관리했고, 호스트에 설치한 Nginx가 사용자 요청을 애플리케이션으로 전달했습니다.

GitHub Actions에서는 여러 단계를 **잡**(job)으로 묶고, 잡을 **러너**(runner)에서 실행합니다. 처음에는 GitHub가 제공하는 **GitHub-hosted runner** 하나에서 아래 작업을 이어서 수행하도록 했습니다.

1. 소스를 가져와 Gradle 빌드를 수행합니다.
2. Docker 이미지를 빌드하고 Docker Hub에 올립니다.
3. EC2에 SSH로 접속합니다.
4. EC2에서 새 이미지를 내려받고 Docker Compose로 실행합니다.

앞의 두 단계는 끝났지만 세 번째 단계에서 멈췄습니다. SSH 명령은 연결 시간 초과와 함께 종료 코드 `255`를 남겼습니다.

### SSH 키를 확인하기 전에

`Connection timed out`은 지정한 호스트와 포트에 제한 시간 안에 연결하지 못했다는 뜻입니다. SSH 키로 인증하기도 전에 실패했으니 네트워크 연결부터 살펴봐야 했습니다.

요청이 차단됐다면 곧바로 실패할 것 같지만, 연결 요청이 중간에서 버려지면 클라이언트는 응답이 올 때까지 기다립니다. 결국 제한 시간이 지나고 나서야 연결 시간 초과로 끝납니다. 반대로 연결 대상이 명시적으로 거절 응답을 보내면 [`Connection refused`](https://repost.aws/knowledge-center/ec2-linux-resolve-ssh-connection-errors)가 나타날 수 있습니다.

물론 연결 시간 초과가 모두 보안 그룹 때문인 것은 아닙니다. [주소나 라우팅, 서브넷의 접근 규칙에 문제가 있어도](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/TroubleshootingInstancesConnecting.html) 연결하지 못할 수 있습니다. 저희에게는 접속 위치에 따라 결과가 다르다는 단서가 있었습니다. 우아한테크코스 네트워크에서는 접속할 수 있었고, 보안 그룹의 22번 포트가 특정 IP에만 열려 있었습니다. GitHub-hosted runner에서 보낸 요청은 이 허용 범위에 들어가지 않았습니다.

보안 그룹은 EC2로 들어오는 **인바운드 트래픽**과 나가는 **아웃바운드 트래픽**에 각각 허용 규칙을 적용합니다. 저희 환경에서는 외부 러너가 시작하는 SSH 연결은 막혀 있었지만, EC2에서 GitHub나 Docker Hub로 나가는 통신은 허용되어 있었습니다.

![보안 그룹의 인바운드·아웃바운드 규칙. 구체적인 IP와 리소스 식별자는 가림 처리](./보안_그룹_규칙.png)

*보안 그룹 규칙 일부. IP 주소와 리소스 식별자는 가렸습니다.*

### 22번 대신 다른 포트를 쓰면 어떨까요?

SSH가 꼭 22번 포트를 써야 하는 것은 아닙니다. 22번은 기본값이고, [서버의 `Port` 설정](https://man.openbsd.org/sshd_config#Port)을 바꾸면 다른 포트를 사용할 수 있습니다.

포트를 바꾸더라도 보안 그룹에서 외부 러너의 요청을 허용해야 합니다. 저희는 접속 가능한 출발지의 범위를 그대로 두고 싶었으므로, SSH 없이 서버에서 명령을 실행할 방법을 찾아봤습니다.

## 명령 두 개를 실행하려고 어디까지 바꿔야 할까

서버에서 할 일은 이미 정해져 있었습니다. `docker compose pull`로 새 이미지를 받고, `docker compose up -d`로 컨테이너를 실행하면 됩니다. 이 명령들을 누가, 어떻게 실행하게 할지가 고민이었습니다.

후보는 AWS Systems Manager(SSM)의 Run Command, self-hosted runner, AWS CodeDeploy, Watchtower였습니다. **self-hosted runner**는 저희가 준비한 서버에 러너 프로그램을 설치해서 GitHub Actions의 잡을 실행하는 방식입니다.

가장 원했던 것은 빌드와 배포 로그를 한곳에서 보는 것이었습니다. 배포가 실패하면 어느 단계에서 막혔는지 GitHub Actions에서 확인하고 싶었습니다. 기존 Compose 명령도 그대로 쓰고 싶었고요. 서버를 추가할 계획은 없었으므로, 현재 EC2에 무엇을 더 설치하고 관리해야 하는지도 함께 비교했습니다.

각 방식을 쓰려면 무엇을 준비해야 할지 비교했습니다.

| 방법 | 명령을 실행하는 방식 | 저희가 기대한 점 | 더 준비하거나 관리할 것 |
| --- | --- | --- | --- |
| SSM Run Command | EC2의 SSM Agent가 명령을 받아 실행 | 기존 셸 명령 재사용 | 에이전트·권한 설정, 명령의 완료 상태와 출력을 Actions에 연결 |
| self-hosted runner | EC2의 러너가 GitHub Actions 잡을 받아 실행 | 기존 워크플로에서 명령 실행과 로그 확인 | 러너의 실행 상태·권한·업데이트·디스크 관리 |
| AWS CodeDeploy | EC2의 에이전트가 지정한 버전과 배포 절차를 실행 | 배포 단계 관리, 자동 롤백 | 배포 파일 저장소·권한·배포 절차·대상 서버 그룹 설정 |
| Watchtower | 레지스트리의 이미지 변경을 확인하고 컨테이너 갱신 | 이미지 변경에 따른 컨테이너 교체 자동화 | 별도 빌드·푸시, Actions 실행과 컨테이너 갱신 결과를 연결해서 확인 |

SSM Run Command를 쓰면 명령을 보낸 뒤 API나 명령줄 도구로 [완료 여부를 조회하고](https://docs.aws.amazon.com/systems-manager/latest/userguide/monitor-commands.html), 그 결과를 GitHub Actions의 성공·실패에 반영해야 했습니다. 저희에게는 기존 잡을 EC2에서 실행하는 방식이 더 익숙했습니다.

CodeDeploy는 [자동 롤백](https://docs.aws.amazon.com/codedeploy/latest/userguide/deployments-rollback-and-redeploy.html)까지 지원했습니다. 다만 저희가 검토한 구성에서는 Amazon S3에 배포 파일을 저장하고, IAM 권한과 AppSpec 배포 절차를 설정해야 했습니다. EC2 한 대에서 Compose 명령을 실행하려는 데 비해 준비할 것이 많다고 느꼈습니다. 당시에는 자동 롤백을 필수 조건으로 두지 않았고, 필요해지면 별도 스크립트로 보완하기로 했습니다.

[Watchtower](https://containrrr.dev/watchtower/)는 이미지 변경을 감지해 컨테이너를 갱신하고 알림도 보낼 수 있습니다. 하지만 빌드와 푸시를 마친 Actions 실행 기록에 컨테이너 갱신 결과까지 연결해서 보려면 추가 구성이 필요했습니다.

그래서 기존 EC2에 self-hosted runner를 두기로 했습니다. 러너를 직접 관리하는 수고는 들지만, 사용하던 워크플로와 배포 명령을 유지할 수 있었습니다.

## 밖에서 들어갈 수 없다면 안에서 연결하기

외부인이 들어올 수 없는 사무실이라도 안에서 전화를 걸어 업무를 전달받을 수는 있습니다. self-hosted runner도 EC2에서 먼저 GitHub에 연결합니다.

기존에는 GitHub-hosted runner가 EC2의 SSH 포트로 접속했습니다. self-hosted runner는 EC2에서 GitHub로 아웃바운드 HTTPS 연결을 맺고 잡을 받아옵니다. 이를 위해 [443번 포트의 아웃바운드 HTTPS 통신과 필요한 도메인 접근](https://docs.github.com/en/actions/reference/runners/self-hosted-runners)이 허용되어 있어야 합니다.

![외부 러너의 SSH 접속 방식에서 EC2의 self-hosted runner 방식으로 바꾼 배포 구조](./배포_구조_변경_전후.png)

*배포 방식 변경 전후. Spring Boot와 MySQL은 컨테이너로 실행합니다. 사용자 요청을 받는 Nginx 경로는 생략했습니다.*

그림의 점선은 러너가 먼저 맺은 연결로 작업을 전달받는 흐름입니다. 이를 위해 인바운드 포트를 새로 열 필요는 없습니다. 보안 그룹은 [허용된 아웃바운드 요청의 응답](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-security-groups.html)을 인바운드 규칙과 무관하게 허용합니다. 다만 서브넷의 접근 규칙이나 프록시에서도 통신이 가능해야 합니다.

EC2에 러너 프로그램을 설치하고 GitHub 저장소에 등록한 뒤, 배포용 러너에 `ec2-prod` 라벨을 붙였습니다. 워크플로에서는 이 라벨로 배포할 러너를 지정합니다.

러너는 작업을 받을 수 있도록 계속 실행되어 있어야 하므로 서비스로 등록해 둡니다. 워크플로의 셸 명령은 러너를 실행하는 계정으로 수행됩니다. 이 계정에 Docker 명령을 실행할 권한도 필요합니다.

SSH는 최초 러너 설치나 장애 대응처럼 관리자가 직접 접속할 때 계속 사용합니다. 자동 배포에서는 더 이상 사용하지 않습니다.

## 빌드까지 우리 서버에서 해야 할까요?

러너를 EC2에 두면 기존 잡을 통째로 옮기는 방법도 있습니다. 그러면 Gradle 빌드부터 Docker 이미지 제작, 컨테이너 실행까지 한 서버에서 처리하게 됩니다.

하지만 그 서버에서는 이미 Spring Boot와 MySQL이 실행 중입니다. 여기에 컴파일, 테스트, 정적 분석, 이미지 빌드까지 얹으면 서비스와 빌드가 자원을 나눠 쓰게 됩니다. 빌드는 GitHub-hosted runner에 남기고, EC2는 만들어진 이미지를 받아 실행하도록 했습니다.

| 실행 환경 | 맡긴 작업 |
| --- | --- |
| GitHub-hosted runner | Gradle 빌드, 테스트·정적 분석, Docker 이미지 빌드와 Docker Hub 업로드 |
| EC2의 self-hosted runner | Docker 이미지 다운로드, Compose 실행, 이미지 정리 |

물론 이미지를 내려받고 압축을 풀고 컨테이너를 시작하는 데도 자원은 듭니다. 여기에 빌드 작업까지 더하지 않기로 했습니다.

### 잡을 나누고 순서 정하기

워크플로는 `build`와 `deploy` 두 잡으로 나눴습니다. 아래 코드에서는 빌드 단계의 세부 명령을 생략했습니다.

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    # steps: 소스 체크아웃, Gradle 빌드, 이미지 빌드·푸시

  deploy:
    needs: build
    runs-on: [self-hosted, ec2-prod]
    steps:
      - name: Pull & restart
        run: |
          cd ~/yeobaek
          docker compose pull
          docker compose up -d
          docker image prune -f
```

`build`는 `ubuntu-latest`로 지정해서 GitHub가 제공하는 Ubuntu 러너에서 실행합니다. `deploy`에는 `[self-hosted, ec2-prod]`를 지정했습니다. 두 라벨이 모두 붙어 있는 러너에서 배포하겠다는 뜻입니다.

[`needs: build`](https://docs.github.com/en/actions/how-tos/write-workflows/choose-what-workflows-do/use-jobs)는 이미지가 준비된 뒤 배포를 시작하도록 넣었습니다. 위 설정에서는 빌드 잡이 성공해야 배포 잡이 실행되고, 빌드가 실패하거나 건너뛰어지면 배포도 건너뜁니다.

이때 `needs`가 빌드 결과물까지 전달해 주는 것은 아닙니다. 두 잡은 서로 다른 서버에서 실행되므로 파일도 따로 있습니다. 저희는 Docker Hub를 통해 이미지를 전달했습니다. 빌드 잡에서 이미지를 올려 두면, 배포 잡이 `docker compose pull`로 가져오는 식입니다.

배포 잡은 EC2에 미리 준비한 `~/yeobaek/docker-compose.yml`과 `.env`를 사용하므로 소스를 다시 체크아웃하지 않습니다. Compose 설정이나 환경 변수를 바꿀 때는 서버에 있는 파일도 별도로 갱신해야 합니다.

## 설치하고 나서야 보인 것들

러너 프로그램을 실행하고 저장소에 등록하는 과정부터 순탄하지 않았습니다.

### Linux용 파일인데 왜 실행이 안 될까

EC2에 러너를 내려받고 설정 스크립트를 실행했더니 이런 오류가 나왔습니다.

```text
./config.sh: line 80: ./bin/Runner.Listener: cannot execute binary file: Exec format error
```

설치 디렉터리의 파일 이름을 보니 `actions-runner-linux-x64-…tar.gz`였습니다. Linux용은 맞지만 x64용 파일이었습니다. 저희 EC2는 ARM64 기반이어서 실행할 수 없었습니다.

서버의 아키텍처는 다음 명령으로 확인할 수 있습니다. x64는 x86_64라고도 부르며, Docker에서는 amd64로 표기합니다.

```bash
uname -m
```

결과가 `aarch64`라면 ARM64용 러너를, `x86_64`라면 x64용 러너를 선택하면 됩니다. 아키텍처에 맞는 파일로 설치를 진행한 뒤에는 `Connected to GitHub` 로그를 확인할 수 있었습니다.

### `ec2-prod`는 이름이 아니라 라벨이었습니다

프로그램을 실행하고 나니 이번에는 등록 과정에서 막혔습니다. 러너 그룹을 묻는 곳에 `ec2-prod`를 입력했더니, 그런 그룹을 찾을 수 없다는 오류가 나왔습니다.

```text
Could not find any self-hosted runner group named "ec2-prod".
```

워크플로에 적어 둔 `ec2-prod`는 라벨인데 그룹 이름으로 입력했던 것입니다. 등록 화면의 세 항목은 각각 쓰임이 달랐습니다.

| 항목 | 하는 일 | 저희 설정에서의 의미 |
| --- | --- | --- |
| 이름 | 관리 화면과 실행 기록에서 러너를 구분 | 러너 이름은 `ec2` |
| 라벨 | 잡을 배정할 러너의 조건 | 배포 잡에 필요한 라벨은 `self-hosted`, `ec2-prod` |
| 그룹 | 러너를 묶어서 접근 범위 관리 | 등록 중 입력한 `ec2-prod`라는 그룹은 존재하지 않았음 |

앞서 작성한 `runs-on: [self-hosted, ec2-prod]`의 값은 라벨입니다. 러너의 이름을 `ec2-prod`로 짓는 것만으로는 이 조건을 만족하지 않습니다. 러너에 `ec2-prod`라는 사용자 정의 라벨을 추가해야 합니다. 일반적인 등록 과정에서는 [`self-hosted`, 운영체제, 아키텍처 라벨](https://docs.github.com/en/actions/how-tos/manage-runners/self-hosted-runners/use-in-a-workflow)이 기본으로 붙습니다.

같은 라벨을 붙인 러너가 여러 대여도 잡은 조건에 맞는 한 대에만 배정됩니다. 저희는 EC2 한 대에 배포했지만, 서버가 늘어난다면 각 서버에 배포하는 절차도 따로 필요합니다.

### 러너를 고쳤더니 이번에는 이미지가 문제였습니다

잡을 나눈 직후에는 컨테이너가 `exec format error`를 내며 재시작을 반복했습니다. 러너 파일에 이어 Docker 이미지도 아키텍처가 맞지 않았습니다. GitHub-hosted runner에서 만든 이미지는 amd64용이었고, 배포할 EC2는 ARM64였습니다.

배포 로그에도 이미지의 플랫폼은 `linux/amd64`인데 호스트는 `linux/arm64/v8`라는 경고가 나왔습니다.

![배포 로그에 표시된 linux/amd64 이미지와 linux/arm64/v8 호스트의 플랫폼 불일치 경고](./Actions_ARM_아키텍처_경고.jpg)

*[수정 전 배포 잡](https://github.com/woowacourse-teams/2026-yeobaek/actions/runs/31368847746/job/93393932500#step:2:109)의 플랫폼 불일치 경고입니다. 컨테이너 내부의 `exec format error` 로그와는 별개의 메시지입니다.*

해결을 위해 Docker Buildx로 amd64와 arm64용 이미지를 함께 만들었습니다. Buildx는 여러 플랫폼의 이미지 빌드를 지원하는 도구입니다. 다른 CPU 아키텍처의 실행 환경을 에뮬레이션할 수 있도록 QEMU도 준비했습니다. 다음 단계들은 소스 체크아웃, Gradle 빌드, Docker Hub 로그인 이후에 실행합니다.

```yaml
- name: Set up QEMU
  uses: docker/setup-qemu-action@v3

- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v3

- name: Build and push image (amd64+arm64)
  run: |
    IMAGE="${{ secrets.DOCKERHUB_USERNAME }}/yeobaek-backend"
    docker buildx build --platform linux/amd64,linux/arm64 \
      -t "$IMAGE:latest" -t "$IMAGE:${{ github.sha }}" --push .
```

`--platform linux/amd64,linux/arm64`로 두 아키텍처를 지정했습니다. Docker는 [멀티 플랫폼 이미지](https://docs.docker.com/build/building/multi-platform/)를 내려받을 때 호스트에 맞는 것을 선택하므로, EC2에서는 ARM64용 이미지를 받을 수 있습니다. 배포 대상이 ARM64뿐이라면 ARM64용 이미지만 빌드해도 됩니다.

## 두 잡이 모두 초록색이 됐습니다

이미지의 아키텍처를 수정한 뒤 [빌드와 배포를 다시 실행했습니다](https://github.com/woowacourse-teams/2026-yeobaek/actions/runs/31369922821). GitHub-hosted runner가 이미지를 올린 뒤 EC2의 러너가 받아서 Compose 명령을 실행했고, 두 잡 모두 성공했습니다.

![멀티 아키텍처 수정 후 build와 deploy 잡이 모두 성공한 GitHub Actions 실행 화면](./Actions_빌드_배포_성공.jpg)

*멀티 아키텍처 빌드를 적용한 뒤의 실행 결과입니다. `build` 다음에 `deploy`가 실행됐고, 두 잡 모두 성공했습니다.*

워크플로에서 SSH 명령과 SSH 키를 사용하는 부분도 지웠습니다. 빌드 로그와 EC2에서 실행한 Compose 명령의 로그는 GitHub Actions에서 함께 확인할 수 있었습니다.

그런데 앞서 아키텍처 경고가 나왔던 배포도 잡 상태는 성공이었습니다. [`docker compose up -d`](https://docs.docker.com/reference/cli/docker/compose/up/)는 컨테이너를 백그라운드로 실행하므로, 명령이 끝나도 Spring Boot는 기동 중이거나 직후 오류로 종료될 수 있습니다. 초록불만으로 애플리케이션의 정상 동작까지 판단할 수는 없었습니다.

MySQL에는 `healthcheck`를 설정해 DB가 준비될 때까지 애플리케이션 시작을 기다리게 했습니다. 하지만 정작 애플리케이션이 요청에 응답하는지는 검사하지 않고 있었습니다.

애플리케이션 기동을 기다렸다가 HTTP 요청을 보내고, 응답 검사에 실패하면 잡도 실패하도록 연결하는 일이 남았습니다.

## 러너를 들여놓고 나면 챙겨야 할 일

서비스와 같은 EC2에서 러너를 실행하므로, 러너가 어떤 코드를 실행하는지와 실행 상태를 직접 챙겨야 합니다.

### 이 서버에서 실행해도 되는 코드인지

러너가 실행하는 코드에 악성 명령이 들어 있으면 [서버 파일이나 자격 증명에 접근할 수 있습니다](https://docs.github.com/en/actions/concepts/security/compromised-runners). SSH 접근 제한으로는 서버 안에서 실행되는 이 명령을 막을 수 없습니다.

그래서 외부 풀 리퀘스트(PR)의 코드를 검증하는 작업을 운영 러너에서 실행해서는 안 됩니다. 워크플로를 누가 수정할 수 있는지, 배포 브랜치는 어떻게 보호하는지, 어떤 외부 Action을 사용하는지도 확인해야 합니다.

실행할 코드를 충분히 통제하기 어렵다면 서비스와 러너를 같은 서버에 두는 구성부터 다시 검토해야 합니다.

### 배포가 시작되지 않는다면

애플리케이션이 정상이어도 러너가 멈추면 새 배포는 대기 상태에 머뭅니다. GitHub Actions가 [잡을 배정할 온라인 러너](https://docs.github.com/en/actions/reference/runners/self-hosted-runners)를 찾지 못하기 때문입니다.

이때는 러너가 온라인인지, 필요한 라벨이 붙어 있는지, GitHub와 통신할 수 있는지부터 확인합니다. 잡이 시작된 뒤 실패했다면 해당 단계의 로그를 살펴보면 됩니다.

러너 업데이트도 직접 챙겨야 합니다. EC2 자체가 중단되면 러너를 통한 복구도 불가능하므로, 관리자 접근 경로를 별도로 남겨 둬야 합니다.

### 이미지를 지우기 전에

배포를 반복하면 이전 이미지가 쌓이므로 스크립트 마지막에 [`docker image prune -f`](https://docs.docker.com/reference/cli/docker/image/prune/)를 넣었습니다. 태그가 없고 컨테이너에서도 참조하지 않는 dangling 이미지를 정리합니다. `-f`는 확인 질문을 생략하는 옵션입니다. 태그가 남은 이미지의 정리와 보관 기간은 별도로 정해야 합니다.

롤백할 이미지도 보관해야 합니다. 앞의 빌드 코드에서는 이미지에 `latest`와 커밋 식별자(SHA) 태그를 함께 붙였습니다. SHA 태그로 어떤 커밋에서 만든 이미지인지 구분할 수 있지만, EC2의 Compose는 계속 갱신되는 `latest`를 사용했습니다. 이전 버전으로 되돌리려면 보관한 이미지의 태그를 지정해 다시 실행해야 합니다.

같은 이유로 `needs: build`만으로는 방금 빌드한 이미지가 배포된다고 보장할 수 없습니다. `needs`는 빌드의 성공을 기다릴 뿐, Compose가 사용하는 `latest`를 그 빌드에 고정하지는 않습니다. 예를 들어 별도 워크플로나 수동 푸시가 `latest`를 갱신하면 배포 잡에서 다른 이미지를 받을 수 있습니다. 이런 경로가 생긴다면 빌드와 배포가 같은 이미지를 가리키도록 보완해야 합니다.

SHA 태그나 이미지 내용의 식별자인 digest를 배포 잡에 전달하고, Compose가 그 값을 사용하게 하면 배포할 이미지를 지정할 수 있습니다.

### 롤백도 직접 만들게 된다면

HTTP 응답 검사에 실패했을 때 이전 이미지를 다시 실행하려면, 그 절차도 워크플로에 작성해야 합니다. 러너가 자동으로 이전 버전을 선택해 복구해 주지는 않습니다.

또한 지금처럼 애플리케이션 컨테이너 하나를 교체하는 방식에서는 서비스가 중단되는 시간이 생길 수 있습니다. 무중단 배포가 필요하다면 새 버전을 실행하고 정상 동작을 확인한 뒤 트래픽을 넘기는 절차가 필요합니다.

자동 롤백이나 여러 서버에 순서대로 배포하는 기능까지 필요해진다면 CodeDeploy 같은 도구를 다시 비교할 만합니다. 처음에는 Compose 명령만 실행하면 됐지만, 직접 관리할 배포 코드가 늘어나면 도구를 도입하는 쪽이 나을 수도 있습니다.

## 마치며

배포를 위해 EC2로 들어가는 대신, EC2가 GitHub에 연결해 작업을 받도록 바꿨습니다. SSH 접근 범위를 유지하면서 빌드와 배포를 한 워크플로에서 실행할 수 있었습니다.

서버 한 대에 Compose로 배포하던 저희에게는 기존 설정을 크게 바꾸지 않아도 된다는 점이 좋았습니다. 대신 서비스 서버에 러너를 두면서 실행할 코드와 러너 상태를 직접 관리하게 됐습니다. 같은 구성을 고려한다면 이 부분도 함께 살펴보시면 좋겠습니다.
