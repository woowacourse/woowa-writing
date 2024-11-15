# 실제 프로젝트 사례로 공부하는 도커

도커는 개발자들이 컨테이너 애플리케이션을 구축, 공유, 실행할 수 있도록 돕는 플랫폼이다. 어떤 운영 체제라도 도커가 설치되어 있다면 편하게 도커 컨테이너를 사용해서 애플리케이션 실행 환경을 구축할 수 있다. 올해 들어 도커를 많이 사용하고 공부하면서 도커의 장점을 몸소 느꼈고 경험을 공유하고 싶었다.

## 내 경험으로 비추어 본 도커를 공부해야 하는 이유

### 1. 이식성이 좋다.

한 번 애플리케이션 실행 환경을 구축해 놓으면(도커 이미지를 빌드해 놓으면) 언제든지 다른 환경에서도 실행할 수 있다. 나는 특히 다른 파트 개발자들과 협업할 때나 서비스 인프라가 변경되었을 때 이 장점을 체감했다.

최근 진행한 프로젝트의 로컬 개발 환경은 아래와 같다.
<img width="641" alt="스크린샷 2024-10-01 오후 10 58 54" src="https://github.com/user-attachments/assets/92fb1525-3e68-4874-bba2-1092f503cd78">

안드로이드 개발자들은 로컬에서 api 연결 테스트를 해야 하기 때문에 개발용 서버가 필요하다. 따라서 운영 서버와 개발 서버에서 사용하는 이미지를 레지스트리에 저장하고 안드로이드 개발자가 이를 다운로드하여 실행시킬 수 있도록 했다.

이 방식으로 안드로이드 개발자들은 백엔드 개발/운영 서버 배포 환경에 상관없이 로컬호스트로 테스트할 수 있다. 프로젝트 초기나 개발 서버 인프라가 불안정(변경될 때 등)할 때 특히 도움 되었다. 웹 클라이언트 개발자들과 협업할 경우도 유용한데, 로컬호스트로 테스트하며 same site, secure와 같은 복잡한 브라우저의 HTTP 보안 정책에서 자유롭게 개발할 수 있다.

백엔드 개발자들도 로컬에서 같은 조회용 데이터베이스를 사용하기 위해서 도커 레지스트리에 데이터베이스 이미지를 올려 두고 함께 사용했다. CI 파이프라인에서도 적용했다. 또한 프로젝트에서 데이터베이스를 MySQL에서 MongoDB로 교체할 때 도커를 활용해 개발 및 운영 환경에 영향을 주지 않고 데이터베이스 전환을 원활하게 수행할 수 있었다.

### 2. 개발 서버 비용을 절약할 수 있다.

로컬 개발 환경, CI 등에서 도커 이미지를 활용하니 개발 서버 비용을 줄일 수 있다.

### 3. 애플리케이션 실행 환경을 편하게 공부할 수 있다.

도커를 공부하면 자동으로 애플리케이션 실행 환경을 공부하게 된다. 네트워크와 저장소를 직접 설정하고 고민할 수 있다. 이번 글에서 자세히 언급하지 않지만 도커 스웜을 활용한다면 오케스트레이션도 구축해볼 수 있다. 이전에 인프라에 대한 이해가 부족했던 나는 도커를 활용해 프로젝트 인프라를 구축하면서 개념을 익히고 실전 CS 지식을 쌓을 수 있었다.

## 도커의 아키텍쳐

<img width="641" src ="https://github.com/user-attachments/assets/bc6c3999-1ae9-4cc9-8d36-e1140e9e7636">

사진처럼 도커는 클라이언트-서버 아키텍처를 사용한다.

우리가 도커를 사용하기 위해 터미널에 직접 치는 명령어(CLI)가 클라이언트다.
도커 데몬(dockerd)은 서버로서 클라이언트와 통신하고 컨테이너를 빌드, 실행, 배포하는 등의 작업을 수행한다.
서버와 클라이언트가 통신할 때 UNIX 소켓 또는 네트워크 인터페이스 위에서 Restful API를 사용한다. CLI 말고도 도커 컴포즈를 도커 클라이언트로 사용할 수 있다.

도커 데스크탑(Docker Desktop)은 도커를 실행할 때 사용하는 도구로, Linux 기반의 VM 안에서 도커가 실행된다. 맥과 윈도우를 사용한다면 Docker desktop으로 편하게 도커를 실행할 환경을 구성할 수 있다. 도커 데몬, 도커 클라이언트, 도커 컴포즈, Content Trust, 쿠버네티스, Credential Helper를 포함한다.

## 도커의 저장소

### 컨테이너별도의 저장소가 필요한 이유

기본적으로 도커 컨테이너는 파일들을 컨테이너의 writable layer에 저장한다. 이는 무엇을 의미하냐면

1. 데이터가 영구적으로 저장되지 않는다. 컨테이너가 존재하지 않으면 데이터 역시 존재하지 않는다.
2. 다른 프로세스가 데이터가 필요하더라도 컨테이너 밖으로 꺼내기 어렵다.
3. 데이터가 호스트 머신에 종속적이어서 다른 머신이나 환경으로 이동하기 어렵다.
4. writable layer에 파일을 쓰는 것은 컨테이너의 파일시스템을 관리하기 위해 storage driver가 필요하다. storage driver은 리눅스 커널을 사용하며 통합된 파일시스템을 제공한다. 이 부가적인 추상화 과정이 호스트의 파일시스템에 직접 파일을 쓰는 것보다 성능 저하를 일으킨다.

따라서 이 단점들을 해소하기 위해서 도커에서 제공하는 저장 방식을 사용할 수 있다.

<img width="641" src ="https://github.com/user-attachments/assets/267b4ee0-f845-4faa-899f-a092a34377b1">

호스트의 디스크에 파일을 저장하는 방법으로 volume과 bind mount가 있다. 호스트의 메모리에도 저장할 수 있는데, tmpfs mount를 사용하면 된다.

### Volume의 장점

Volume은 데이터를 도커에 의해서 관리되는 영역에 저장한다. 리눅스라면 /var/lib/docker/volumes/ 라는 위치이다. 하지만 Bind mount는 호스트의 어디든지 저장할 수 있는 방식이다. 다른 프로세스들이 해당 데이터를 수정할 수도 있다. 따라서 bind mount는 보안적인 문제가 발생할 수 있고 호스트의 디렉토리 구조와 OS에 의존적이다. 대체로 volume이 더 선호된다.

Volume의 상대적 장점을 비교해 보면 아래와 같다.

1. 백업과 마이그레이션이 쉽다.
2. 도커의 CLI 명령어와 API로 관리할 수 있다.
3. 리눅스와 윈도우 컨테이너에서 모두 동작한다.
4. 여러 컨테이너들에서 더 안전하게 공유될 수 있다.
5. 여러 컨테이너가 동시에 동일한 볼륨을 읽기-쓰기 또는 읽기 전용으로 마운트 할 수 있다.
6. Volume driver를 사용하면 원격 호스트나 클라우드에 데이터들을 저장할 수 있고 데이터를 암호화하거나 기타 기능을 추가할 수 있다.
7. Mac과 Window에서 Docker Desktop을 사용한다면 성능이 훨씬 뛰어나다.

6번은 Docker Desktop이 VM 위에서 실행된다는 점을 생각하면 이해가 쉽다. Volume은 VM 내에 저장하지만 Bind mount는 호스트 머신에 저장한다. 즉, Bind mount를 사용하면 VM에서 실행되는 도커 프로세스가 VM 외부인 호스트의 파일시스템에 데이터를 쓰는 IO 작업이 추가로 발생한다.

### Volume의 활용

도커 컴포즈 파일을 아래와 같이 구성해보자.

```
services:
  backend:
    image: example/database
    volumes:
      - db-data:/etc/data

  backup:
    image: backup-service
    volumes:
      - db-data:/var/lib/backup/data

volumes:
  db-data:

```

backend 서비스의 데이터는 backup 서비스에 의해 주기적으로 백업될 수 있다. 도커 문서에 나와 있는 예시인데, 호스트에 백업하는 게 아니라 따로 백업용 컨테이너를 사용하고 있다. 호스트에 종속적이지 않고 백업 데이터 역시 컨테이너와 이미지로 관리할 수 있어 좋다.

### Bind mount의 활용

Bind mount가 적절한 사례도 있다.

1. 호스트 머신에서 컨테이너로 설정 파일을 공유할 때
   예를 들어 도커는 기본적으로 /etc/resolv.conf를 호스트 머신에서 각 컨테이너로 마운트 하여 컨테이너에 DNS resolution을 제공한다. 컨테이너가 기본 설정인 Bridge network를 사용할 때 호스트와 같은 DNS 서버를 사용하기 위해서 필요한 과정이다.
2. 개발 환경에서 호스트와 컨테이너 간에 소스 코드 또는 빌드 산출물을 공유할 때

   예를 들어 호스트에서 gradle 프로젝트를 빌드할 때마다 컨테이너는 새로 빌드된 산출물에 접근할 수 있다. 개발 환경에서 쓰기 편하다. 하지만 운영 환경에서는 아래 도커파일처럼 COPY해서 사용하는 것이 안전하다.


```
FROM openjdk:17
ARG JAR_FILE=/build/libs/*.jar
COPY ${JAR_FILE} app.jar
ENTRYPOINT ["java","-jar", "/app.jar"]

```

### Tmpfs mount

위 두 가지 방식과 성질이 아주 다르다. 호스트의 메모리에 저장하는 것이기 때문에 영구 저장이 되지 않는다. 보안상의 이유로 디스크에 저장하지 않아야 하는 데이터나 대량의 비영속 애플리케이션 데이터를 사용할 때 유용하다.

## 도커의 네트워크

도커 컨테이너 여러 대로 개발, 운영 환경을 구축할 때 네트워크 설정은 필수이다. 직관적인 이해를 위해 프로젝트 개발, 운영 환경에서 구성한 도커 네트워크를 예시로 들겠다.

<img width="641" alt="스크린샷 2024-10-01 오후 10 58 54" src="https://github.com/user-attachments/assets/92fb1525-3e68-4874-bba2-1092f503cd78">

### Docker Network Driver

### Bridge

기본 드라이버다. 사용자 정의로 만들어서 컨테이너 간 통신을 할 수 있다. 도커 컴포즈는 기본적으로 자동으로 사용자 정의 네트워크를 생성한다. 네트워크 layer 2의 스위치와 유사하다.

<img width="641" src ="https://github.com/user-attachments/assets/8365fc41-8dc3-4b98-89b0-c1af0eaf0608">

안드로이드 개발자들이 로컬에서 사용하는 도커 컴포즈 파일에서 Bridge Network Driver를 사용한다.

```
version: '3'
services:

  server:

    depends_on:
      db:
        condition: service_healthy

    container_name: server
    image: {Springboot 이미지 이름}
    expose:
      - 8080
    ports:
      - 8080:8080
    tty: true
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://db:3306/{데이터베이스 이름과 기타 설정}
      SPRING_DATASOURCE_USERNAME:
      SPRING_DATASOURCE_PASSWORD:
      SPRING_PROFILES_ACTIVE:

  db:

    healthcheck:
      test: [ 'CMD-SHELL', 'mysqladmin ping -h 127.0.0.1 -u root --password=$$MYSQL_ROOT_PASSWORD' ]
      interval: 10s
      timeout: 2s
      retries: 20

    container_name: db
    image: {Test DB 이미지 이름}
    ports:
      - "23306:3306"
    environment:
      MYSQL_ROOT_PASSWORD:

```

도커 컴포즈 파일로 묶여 있는 db 컨테이너와 server 컨테이너가 같은 네트워크 Bridge 안에 있다. db가 먼저 실행되는 상태에서 springboot 애플리케이션이 실행되어야 하므로 `depends_on`과 `healthcheck`를 설정해 주었다.

`ports:`가 published ports를 지정하는 부분이다. 왼쪽이 호스트 포트, 오른쪽이 컨테이너 포트이다. db 컨테이너의 경우 호스트 포트를 23306으로 설정하고 컨테이너 내부 포트는 3306이다. 따라서 같은 네트워크 안에 있는 server 컨테이너가 데이터 소스 url의 포트를 3306으로 사용한다.

또한 데이터 소스 url에서 db 컨테이너의 내부(private) ip 주소 대신 db 컨테이너의 이름을 적어준 걸 볼 수 있다. 커스텀 bridge network를 구성했다면 컨테이너는 도커의 내장된 DNS 서버를 사용한다. 도커 네트워크 내에서 해결되지 않은 DNS 쿼리는 호스트 DNS 서버로 전달한다. 따라서 컨테이너의 이름만으로 컨테이너의 내부 ip 주소를 찾을 수 있다.

백엔드 개발자의 로컬 환경에서는 데이터 소스 url을 어떻게 지정해주어야 할까? 같은 도커 네트워크 내에 서버와 데이터베이스가 존재하지 않는다.(데이터베이스 컨테이너만 존재한다.) 따라서 서버에서 데이터베이스를 연결할 때 23306 포트로 연결해주어야 한다. ip 주소 역시 db 대신 localhost로 지정해 준다.

```
jdbc:mysql://localhost:23306/{데이터베이스 이름과 기타 설정}

```

### Host

Host 네트워크 드라이버는 컨테이너를 띄우는 호스트 컴퓨터와 컨테이너가 같은 네트워크를 쓰는 방식으로 동작한다.

아래는 프로젝트 초반 백엔드 운영 환경에서 사용했던 도커 컴포즈 파일이다. Springboot와 Nginx 컨테이너는 Bridge 드라이버로, Fail2ban 컨테이너는 host 드라이버로 설정했다.

```
version: '3'

services:
  server:
    container_name: server
    image: {Springboot 이미지 이름}
    ports:
      - 8080:8080

  nginx:
    container_name: nginx
    image: {Nginx 이미지 이름}
    ports:
      - 80:80
      - 443:443
    depends_on:
      - server
    volumes:

  fail2ban:
    container_name: fail2ban
    image: {Fail2ban 이미지 이름}
    privileged: true
    network_mode: 'host' # 주목할 부분
    volumes:
      - ./log/nginx:/log/nginx/ # Nginx 로그를 바탕으로 요청을 추적한다

```

Fail2ban은 짧은 시간 안에 많은 요청을 한(디도스 공격) ip를 차단할 수 있는 프로그램이다. ip 주소를 iptable 같은 방화벽에 추가해서 차단하는 것이기 때문에 호스트(이를테면 ec2 우분투 서버)와 동일한 네트워크 스택을 가져가야 한다.

### None

None 네트워크 드라이버는 다른 컨테이너들 및 호스트 컴퓨터와 통신하지 않는 방식이다.

### ETC

이 외에도 Overlay, Ipvlan, Macvlan가 있다. 하지만 위 세 개가 가장 기본적이다. Overlay는 다른 호스트에서 컨테이너끼리 통신할 때 사용한다. 도커 스웜 모드에서 사용해 볼 수 있겠다. Ipvlan은 IPv4, IPv6 주소 할당을 완전히 제어할 수 있고 Macvlan는 컨테이너에 맥 주소를 할당한다.

### 도커 컨테이너의 내부 IP 주소는 어떻게 할당될까?

컨테이너는 연결된 네트워크에 대해 IP 서브넷 범위 안에서 주소를 할당받는다. 도커 데몬이 동적으로 서브넷팅과 IP 주소 할당을 관리한다.

각 네트워크는 기본 서브넷 마스크와 게이트웨이도 갖고 있다. 기본적으로 컨테이너의 호스트네임은 Docker에서 컨테이너의 ID이다. 도커를 실행할 때 명령어에서 `--hostname` 플래그로 변경할 수 있다.

`docker inspect {네트워크이름}`명령어로 위에서 언급됐던 안드로이드 개발자의 로컬 개발 환경에서 컨테이너 내부 IP 주소를 확인해보자.

```
[
    {
        "Name": "local-network",
        "Id": "8e82",
        "Created": "2024-08-05T03:58:41.201090169Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.23.0.0/16",
                    "Gateway": "172.23.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "02d": {
                "Name": "db",
                "EndpointID": "1e9b2f",
                "MacAddress": "생략",
                "IPv4Address": "172.23.0.2/16",
                "IPv6Address": ""
            },
            "740d": {
                "Name": "server",
                "EndpointID": "dd91fe",
                "MacAddress": "생략",
                "IPv4Address": "172.23.0.3/16",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {
            "com.docker.compose.network": "local-network",
            "com.docker.compose.project": "pokerogue",
            "com.docker.compose.version": "2.12.2"
        }
    }
]

```

Gateway(172.23.0.1)가 브릿지 네트워크가 외부와 통신할 때 트래픽을 처리한다. 서브넷을 보면 16비트 서브넷 마스크가 지정되어 있고 그 안에서 db 컨테이너와 server 컨테이너의 IP 주소를 할당받는다.
