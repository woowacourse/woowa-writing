# 팀 인프라 소개

## 목차  
  
- 전반적인 인프라 소개
- 가비아를 통한 DNS 설정
	- DNS 설정
	- 레코드의 종류
- 프론트엔드 인프라 구조
	- S3
	- CloudFront
- 백엔드 인프라 구조
	- ALB
	- EC2
	- RDS
- 기타
	- WAF
- 참고 자료

## 전반적인 인프라 소개

<img align="center" src="./image/ddangkong-infra-structure.png">

 위 구성은 크게 프론트 부분과 백엔드 부분으로 나눌 수 있다. 프론트엔드는 모든 유저에게 똑같이 제공되는 정적 파일을 담당한다. S3에서 정적 파일을 보관하며, CloudFront에서 정적 파일을 쉽게 이용할 수 있도록 도와준다. 그리고 유저, 상황마다 바뀌는 정보는 백엔드에게 요청한다.

 백엔드는 유저, 상황마다 바뀌는 정보를 다룬다. 서버로 보내는 각 요청이 DB의 데이터를 조회하거나 수정하는 역할을 한다. ALB(Application Load Balancer)는 두 서버의 로드 밸런싱을 담당하며, EC2는 WAS를 실행을 담당한다. 데이터베이스는 RDS로 구성하였으며 소스와 레플리카를 따로 두었다.

추가적으로 백엔드 요청에서 ALB에 도달하기 전, WAF(Web Application Firewall)에서 원치 않는 트래픽으로부터 네트워크를 보호하며, CloudWatch에서는 전반적인 상태 모니터링 및 로그 확인을 담당한다. 각 구성에 대해 알아보기 전 DNS 설정에 대해 먼저 알아보자.

## 가비아를 통한 DNS 설정

 DNS(Domain Name System)란 사람이 읽을 수 있는 도메인 이름(예: ddangkong.kr)을 머신이 읽을 수 있는 IP 주소(예: 192.0.2.44)로 변환하는 서버를 말한다. 사람이 인지하기 쉬운 문자를 통해 숫자 4개로 되어있는 서버 주소로 변환하도록 하여, 사용자는 IP 주소를 알지 않아도 원하는 사이트에 접속할 수 있다. 그리고 IP 주소가 바뀌더라도 DNS를 설정해준다면, 같은 도메인 이름을 통해 언제든지 같은 서비스에 접속할 수 있다.

 가비아(Gabia)는 IT 환경을 필요로 하는 기업을 대상으로 클라우드, 그룹웨어, 보안, 도메인, 호스팅에 이르는 통합적인 인프라 서비스를 제공하는 기업이다. 도메인, 웹호스팅, 홈페이지 서비스에서 나아가 자체 기술 스텍으로 개발한 IaaS(Infrastructure-as-a-Service) ‘G-클라우드’와 클라우드 운영에 꼭 필요한 전반의 서비스를 제공한다. 우리 서비스에서는 도메인 구매 및 DNS 설정을 위해 해당 서비스를 사용했다. 그렇다면 가비아를 통해 DNS를 설정하는 방법에 대해 알아보자.

### DNS 설정

- DNS 설정 방법
	1. [가비아 메인 화면](https://www.gabia.com/)으로 이동
	2. Gmail 땅콩 계정으로 로그인
	3. My 가비아 > 이용 중인 서비스 > 도메인 으로 이동
	4. ddangkong.kr의 관리 버튼 클릭하여 ddangkong.kr의 도메인 관리 페이지로 이동
	5. 하단의 DNS 정보 > 도메인 연결 > 설정 버튼을 클릭하여 DNS 관리 페이지로 이동
	6. ddangkong.kr의 DNS 정보 > 설정 버튼을 클릭하여 ddangkong.kr의 DNS 관리 페이지로 이동

- 현재 설정 (24.10.01 기준)
	<img align="center" src="./image/dns-setting.png">

- prod 관련 도메인 설정
	- @ : ddangkong.kr로 연결된 값되는 주소를 의미한다. prod 환경으로 구현된 프론트 CloudFront에 접근하는 주소이다.
	- api : api.ddangkong.kr로 연결되는 주소를 의미한다. WAS와 연결되어 있는 ELB의 주소와 연결되어 있다.
- dev 관련 도메인 설정
	- dev : dev.ddangkong.kr 에서 연결되는 주소를 의미한다. dev 환경의 프론트 CloudFront에 접근하는 주소이다.
	- api.dev : api.dev.ddangkong.kr 에서 연결되는 주소를 의미한다. WAS의 public ip가 입력되어 있다.

### 레코드의 종류

- A: 특정 도메인 이름을 IP 주소에 매핑하는 DNS 레코드
	- 작동 방식: 사용자가 웹 브라우저에 example.com을 입력하면, DNS 서버는 A 레코드를 참조하여 해당 도메인을 192.0.2.1 IP 주소로 변환하고, 그 IP 주소로 트래픽을 라우팅한다. 
	- api.dev의 경우 EC2의 public IP를 값으로 넣기 위해 A 레코드를 사용했다.

- CNAME: 하나의 도메인 이름을 다른 도메인 이름으로 매핑하는 DNS 레코드 
	- 작동 방식: CNAME 레코드는 다른 도메인의 별칭(Alias)을 정의하는 데 사용된다. 예를 들어, www.example.com 을 example.com 으로 리디렉션하고자 할 때 CNAME 레코드를 사용한다. 즉, www.example.com 으로 요청이 오면 DNS 서버는 이를 example.com으로 변환하고, 그 후에 example.com에 대한 A 레코드 등을 참조하여 최종 IP 주소로 연결된다.

- 기타 레코드 종류
	- AAAA 레코드 : 도메인 이름을 IPv6 주소로 매핑
	- MX 레코드 (Mail Exchange Record) : 이메일을 수신할 메일 서버를 지정
	- TXT 레코드 (Text Record): 도메인에 대한 추가적인 텍스트 정보를 제공 (주로 인증 및 정책 정보를 저장)
	- NS 레코드 (Name Server Record) : 도메인을 관리하는 네임서버를 지정합니다.
	- SOA 레코드 (Start of Authority Record) : 도메인에 대한 권한이 있는 네임서버와 관련된 정보를 제공 (일반적으로 도메인의 관리 정보와 리프레시 타임, 재시도 타임 등을 포함)

## 프론트엔드 인프라 구조

### S3 (Amazon Simple Storage Service)

 S3란 어디서나 원하는 양의 데이터를 저장하고 검색할 수 있도록 구축된 객체 스토리지이다. 업계 최고 수준의 내구성, 가용성, 성능, 보안 및 거의 무제한의 확장성을 아주 저렴한 요금으로 제공하고 있습니다. 그리고 언제든지 어디서나 원하는 양의 데이터를 저장하고 검색하는 데 사용할 수 있는 간편한 웹 서비스 인터페이스를 제공한다.

  S3는 정적 웹사이트를 직접 호스팅할 수 있는 기능을 제공한다. React 애플리케이션은 주로 HTML, CSS, JavaScript로 구성된 정적 파일이므로 별도의 서버 없이도 S3만으로 호스팅이 가능합니다. S3는 사용한 만큼만 비용을 지불하는 구조로, 초기 비용 부담 없이 작은 프로젝트부터 큰 프로젝트까지 확장할 수 있습니다. 추가적으로 글로벌 인프라를 사용하여 사용자가 갑자기 늘어나더라도 추가 설정 없이 트래픽을 자동으로 처리할 수 있다.

### CloudFront

 CloudFront란 HTML, CSS, JavaScript 및 이미지 파일과 같은 정적 및 동적 웹 콘텐츠를 사용자에게 더 빨리 배포하도록 지원하는 웹 서비스이다. CloudFront는 S3와 와 쉽게 연동되어 글로벌 사용자에게 빠른 콘텐츠 전달이 가능하다. CloudFront를 사용하면 전 세계의 사용자에게 빠르고 안정적인 성능을 제공할 수 있어 React 애플리케이션의 사용자 경험이 좋아진다. 추가적으로 CloudFront의 캐싱 기능을 통해 S3로의 직접적인 요청 수를 줄여 S3에서 발생하는 데이터 전송 비용을 절감할 수 있다.

## 백엔드 인프라 구조

### ALB (Applivation Load Balancing)

ELB(Elastic Load Balancing)은 둘 이상의 가용 영역에서 EC2 인스턴스, 컨테이너, IP 주소 등 여러 대상에 걸쳐 수신되는 트래픽을 자동으로 분산한다. 등록된 대상의 상태를 모니터링하면서 상태가 양호한 대상으로만 트래픽을 라우팅한다. 그 중에서 ALB는 7계층의 로드 밸런서이다. HTTP 및 HTTPS 트래픽을 처리하며 URL 경로 또는 호스트 기반의 라우팅을 지원하여 특정 요청을 다양한 백엔드 서비스로 보낼 수 있다.

그렇다면 실시간 서비스에서 로드 밸런서가 필요한 이유는 무엇일까? 서비스가 중단 없이 지속적으로 운영되고, 안정적이며 효율적인 서비스를 제공하기 위해서는 고가용성 설정이 필수적이다. 이를 달성하기 위해, 우리는 시스템의 부하를 분산하고 장애 발생 시에도 서비스가 중단되지 않도록 다중화 로드 밸런서가 필요하다. 그래서 손쉽게 로드 밸런싱을 구현해주는 ALB를 사용했다.

###  EC2 (Amazon Elastic Compute Cloud)

 EC2란 클라우드에서 온디맨드 확장 가능 컴퓨팅 용량을 제공하는 서비스이다. 원하는 만큼 필요한 만큼 사용할 수 있기 때문에 하드웨어 비용이 절감된다. 간편한 설정을 통해 컴퓨터를 빌릴 수 있어 애플리케이션을 더욱 빠르게 배포할 수 있다. 또한 복잡한 명령어 없이 간단하게 네트워크를 설정(예: 인바운드 규칙, 아웃바운드 규칙)할 수 있어 개발자는 개발 자체에 더욱 집중할 수 있다.

EC2를 간단히 말하자면, OS가 설치된 하나의 컴퓨터를 대여해주는 것과 유사하다. WAS를 EC2에서 구동하기 위해서는 여러 설정들이 필요하다. 아래는 땅콩 팀에서 설정해 준 작업 일부이다.

#### ip table 설정

현재 ELB에서 들어온 HTTPS 요청을 EC2에게 80 포트로 요청을 넘긴다. 그리고 WAS는 EC2 내에서 8422 포트에서 실행되어고 있다. 그래서 80 포트로 들어온 요청을 8422 포트로 포드 포워딩 해야 한다. 포트 포워딩에는 여러 방법이 있지만, 땅콩 조에서는 명령어를 통해 ip table을 설정하는 방법을 사용했다.

```bash
iptables -A PREROUTING -t nat -i eth0 -p tcp --dport 80 -j REDIRECT --to-port 8422
```

그리고 아래 명령어를 통해 포트 포워딩이 잘 설정되었는지 확인할 수 있다.
```bash
sudo iptables -t nat -L PREROUTING -n -v
```

#### 스왑 메모리 설정

EC2는 사용하는 CPU, 메인 메모리에 따라 사용 가격이 다르기 때문에, 효율적으로 사용하기 위해서는 스왑 메모리 설정이 필수다. 스왑 메모리는 컴퓨터의 메인 메모리가 부족할 때, 일시적으로 하드 디스크나 SSD 같은 저장 장치를 마치 메모리처럼 사용하는 공간이다. 하드 디스크나 SSD를 사용하기 때문에 속도는 매우 느려질 수 있으나 메인 메모리가 부족할 때 여유 공간을 두고 활용할 수 있다.

스왑 메모리는 ubuntu 24.04 기준 아래와 같이 설정한다. 현재 사용하는 EC2의 메인 메모리 크기의 2배인 2GB로 설정하는 명령어이다.

```bash
sudo dd if=/dev/zero of=/swapfile bs=128M count=16
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

추가적으로 `/etc/fstab` 파일에 아래의 문구를 추가해 주어야 한다.
```plain text
/swapfile swap swap defaults 0 0
```

스왑 메모리가 잘 설정되었는지 아래의 명령어로 확인할 수 있다.
```shell
free -h
```

#### 헬스 체크 설정

ELB는 현재 어플리케이션이 정상 작동하고 있는지 WAS에게 주기적으로 헬스 체크 요청을 보낸다. WAS에서 헬스 체크 요청에 응답을 해야 ELB는 해당 서버가 정상적으로 응답하고 있다고 인지한다. 아래와 같은 설정을 통해 헬스 체크 설정을 할 수 있다.

- Spring Project에서 build.gradle 파일 설정
	```gradle
	dependencies { 
		implementation 'org.springframework.boot:spring-boot-starter-actuator'
		// ...
	}
	```
- application.yml 설정
	```yml  
	management:   
	  endpoints:   
	    web:   
	      exposure:   
	        include: "health"  
	``````

### RDS (Amazon Realtional Database Service)

 RDS란 클라우드에서 간편하게 데이터베이스를 설치, 운영 및 규모 조정할 수 있는 관리형 서비스를 말한다. 대부분의 데이터베이스 관리 작업을 담당하며 번거로운 수동 프로세스(ex. 백업, 소프트웨어 패치, 자동 장애 감지 및 복구)를 처리할 필요가 없어 애플리케이션과 사용자에게 집중할 수 있다. 
 
 또한 DB의 읽기 전용 복제본(Replica)을 설정하여 애플리케이션에서 읽기 요청을 복제본으로 분산시켜 각 DB 요청 부하를 줄였고, 마스터 데이터베이스에 장애가 발생할 경우, 복제본이 즉시 새로운 마스터 역할을 맡아 데이터베이스의 가용성을 유지한다. 또한 복제본을 통해 데이터를 복구하거나, 백업을 생성할 수 있어 데이터 손실 위험을 줄일 수 있다. 

## 기타

### WAF (AWS Web Application Firewall)

WAF란 고객이 정의한 조건에 따라 웹 요청을 허용, 차단 또는 모니터링(계수)하는 규칙을 구성하여 공격으로부터 웹 애플리케이션을 보호하는 웹 애플리케이션 방화벽이다. 예를 들어 특정 IP 주소, HTTP 헤더, HTTP 본문 또는 URI 문자열 만의 요청을 받아들이고 이외의 요청은 차단할 수 있다.

현재의 인프라 구조에서는 따로 WAF에서 처리하는 역할은 없다. 현재 서버의 요청 URI는 모두 `/api`로 시작하기 때문에, 추후 설정을 통해 `/api`로 시작하지 않는 URI 요청은 차단 할 예정이다.

## 참고 자료

- 가비아를 통한 DNS 설정
	- [DNS란 무엇입니까?](https://aws.amazon.com/ko/route53/what-is-dns/) (검색 일자 : 24.10.01)
	- [가비아 회사 소개](https://company.gabia.com/introduce/corporate) (검색 일자 : 24.10.01)
- 프론트엔드 인프라 구조
	- [Amazon S3란 무엇인가요?](https://docs.aws.amazon.com/ko_kr/AmazonS3/latest/userguide/Welcome.html) (검색 일자 : 24.10.01)
	- [Amazon CloudFront란 무엇입니까?](https://docs.aws.amazon.com/ko_kr/AmazonCloudFront/latest/DeveloperGuide/Introduction.html) (검색 일자 : 24.10.01)
- 백엔드 인프라 구조
	- [Elastic Load Balancing이란 무엇인가요?](https://docs.aws.amazon.com/ko_kr/elasticloadbalancing/latest/userguide/what-is-load-balancing.html) (검색 일자 : 24.10.01)
	- [Amazon EC2란 무엇인가요?](https://docs.aws.amazon.com/ko_kr/AWSEC2/latest/UserGuide/concepts.html) (검색 일자 : 24.10.01)
	- [Amazon Relational Database Service(Amazon RDS)란 무엇입니까?](https://docs.aws.amazon.com/ko_kr/AmazonRDS/latest/UserGuide/Welcome.html) (검색 일자 : 24.10.01)
- 기타
	- [AWS WAF FAQ](https://aws.amazon.com/ko/waf/faq/) (검색 일자 : 24.10.01)
	- [Amazon CloudWatch란 무엇인가요?](https://docs.aws.amazon.com/ko_kr/AmazonCloudWatch/latest/monitoring/WhatIsCloudWatch.html) (검색 일자 : 24.10.01)
- thanks to : 우테코 6기 땅콩 (프린의 사진, 이든의 가비아 DNS 설명)
