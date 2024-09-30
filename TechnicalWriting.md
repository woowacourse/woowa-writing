# 0. 주제를 선택한 이유

데이터베이스 관리 시스템(DBMS)은 소프트웨어 애플리케이션의 핵심 구성 요소 중 하나이다. 데이터베이스는 응용 프로그램에서 데이터를 저장, 검색, 수정 및 관리하는 데 필수적인 역할을 한다. 따라서 데이터베이스 선택은 애플리케이션의 성능, 확장성, 보안 등에 직접적인 영향을 미친다.

오픈 소스 데이터베이스 관리 시스템인 MySQL과 PostgreSQL은 서로 다른 특징을 지닌다. 이 둘 사이의 선택은 프로젝트의 요구 사항, 애플리케이션의 특성, 개발팀의 경험 및 선호도에 따라 결정된다. 각 DBMS는 특정 시나리오에서 더 우수한 성능을 발휘할 수 있다.

이 글에서 두 데이터베이스를 간략하게 소개하고, 주요 차이점을 비교하여 선택 가이드를 제공하고자 한다.

이 글은 기본적인 데이터베이스 개념과 MySQL 문법에 대한 이해를 전제하에 설명한다.

<br>

## 1. MySQL란?


MySQL은 세계에서 가장 널리 사용되는  오픈 소스 관계형 데이터베이스 관리 시스템(RDBMS)이다. [DB-Engines](https://db-engines.com/en/ranking)에 따르면 MySQL은 Oracle Database의 뒤를 이어 두 번째로 널리 사용되는 데이터베이스로 선정되었다. Facebook, Twitter, Netflix, Uber, Airbnb, Shopify, Booking.com 등 세계적인 애플리케이션이 MySQL을 사용하고 있다.

![](https://i.imgur.com/kR4wQHJ.png)

*그림 1 DB-Engines Ranking*

### 1.1. MySQL의 주요 기능

MySQL은 빠르고 신뢰할 수 있으며, 확장성과 사용의 용이성을 제공한다. 다음은 MySQL의 주요 기능이다.

-      **멀티스레딩 지원**: MySQL은 커널 스레드를 사용하여 여러 CPU를 효율적으로 사용할 수 있다.
-      **트랜잭션 및 비트랜잭션 스토리지 엔진**: 다양한 스토리지 엔진을 제공하여 유연한 데이터 처리 기능을 지원한다.
-      **확장성**: 수십억 개의 데이터 행과 수십만 개의 테이블을 관리할 수 있어 대규모 데이터베이스 지원에 적합하다.
-      **다양한 연결 옵션**: TCP/IP 소켓, Unix 도메인 소켓 등 다양한 연결 방식을 통해 폭넓은 환경에서 사용할 수 있다.

<br>

## 2. PostgreSQL란?

PostgreSQL은 고급 기능을 제공하는 객체 관계형 데이터베이스 관리 시스템(ORDBMS)이다. ORDBMS는 관계형 데이터베이스(RDBMS)와 객체 지향 데이터베이스의 결합으로, 데이터를 객체로 표현할 수 있는 시스템이다. PostgreSQL은 이러한 ORDBMS 특성을 통해 데이터를 더 자연스럽고 유연하게 관리할 수 있으며, 복잡한 애플리케이션의 요구에 맞게 확장할 수 있다.

### 2.1. ORDBMS란?

DBMS는 관계형 데이터베이스 관리 시스템(RDBMS), 객체 지향 데이터베이스 관리 시스템(ODBMS), 객체-관계형 데이터베이스 관리 시스템(ORDBMS) 등 다양한 형태로 존재한다.

ORDBMS는 관계형 데이터베이스(RDBMS)와 객체 지향 데이터베이스(ODBMS)의 장점을 결합한 시스템이다. 이를 통해 데이터베이스는 객체의 속성, 상속, 다형성 같은 개념을 지원하며, 보다 유연한 데이터 관리가 가능하다. 특히 복잡한 데이터 구조를 다루거나, 비정형 데이터를 처리할 때 ORDBMS는 더 자연스러운 표현과 관리를 제공한다.

PostgreSQL은 이러한 ORDBMS의 특성을 활용하여, 데이터를 유연하고 직관적으로 관리할 수 있다.

### 2.2. PostgreSQL의 주요 기능

PostgreSQL은 복잡한 데이터 워크로드를 안전하게 저장할 수 있는 다양한 기능을 제공한다. 다음은 PostgreSQL의 주요 기능이다.

-      **고급 데이터 유형:** JSON, 배열, 사용자 정의 데이터 타입을 지원한다.
-      **트랜잭션 처리:** ACID 준수로 안정적인 데이터 무결성을 보장한다.
-      **확장성:** 대규모 데이터 처리에 적합하며, 다양한 복제 및 샤딩 옵션을 제공한다.
-      **고급 쿼리 기능**: 서브쿼리, 윈도우 함수, CTE(Common Table Expressions) 지원한다.
-      **사용자 정의 함수:** 다양한 프로그래밍 언어로 함수 작성할 수 있다.

<br>

## 3. MySQL VS PostgreSQL

[Stack Overflow](https://survey.stackoverflow.co/2022/?utm_source=results#most-popular-technologies-database)와 [JetBrains](https://www.jetbrains.com/lp/devecosystem-2022/databases/)의 설문조사에 따르면 두 데이터베이스는 개발자들이 선호하는 데이터베이스임을 알 수 있다. 대표적으로 Stack Overflow 진행한 설문조사를 보면 MySQL 46.85%, PostgreSQL 43.59%로 막상막하의 점유율을 보여주고 있다.

![](https://i.imgur.com/H5KSqN8.png)

*그림 2 JetBrains에서 진행한 설문 조사 (모든 응답자)*

심지어 전문 개발자만 투표한 결과는 PostgreSQL 46.48%, MySQL 45.68%로 PostgreSQL가 1위 자리를 차지했다. 두 DBMS 중에 무엇을 선택할지 고민하기 전에 이들의 공통점과 차이점을 살펴보자.

![](https://i.imgur.com/60FYrZJ.png)

*그림 3 JetBrains에서 진행한 설문 조사 (전문 개발자)*

<br>

### 3.1. MySQL과 PostgreSQL의 공통점

MySQL과 PostgreSQL 중 하나를 선택하는 고민은 두 데이터베이스가 여러 공통점을 가지고 있으며, 그 사용 방식이 유사하기 때문에 발생한다. 따라서 먼저 이들의 공통점을 살펴보자.

·        관계형 데이터베이스 관리 시스템이다. 공통 열값을 통해 서로 관련된 테이블에 데이터를 저장한다.

- 구조화된 쿼리 언어(SQL)를 인터페이스로 사용하여 데이터를 읽고 편집할 수 있다.
- 오픈 소스이며 강력한 개발자 커뮤니티 지원 제공한다.
- 데이터 백업, 복제 및 액세스 제어 기능이 내장되어 있다.

더 알고 싶다면 SQL 키워드 학습을 추천한다. [» SQL에 대해 읽어보기](https://aws.amazon.com/what-is/sql/)  
  
<br>

### 3.2. MySQL과 PostgreSQL의 차이점

MySQL 및 PostgreSQL는 개념적으로는 유사하지만 구현하기 전에 고려해야 할 많은 차이점이 있다.

MySQL은 데이터를 행과 열이 있는 테이블로 저장할 수 있는 관계형 데이터베이스 관리 시스템이다. 많은 웹 애플리케이션, 동적 웹 사이트 및 임베디드 시스템을 지원하는 널리 사용되는 시스템이다. PostgreSQL은 MySQL보다 더 많은 기능을 제공하는 객체 관계형 데이터베이스 관리 시스템이다. 데이터 유형, 확장성, 동시성 및 데이터 무결성에 있어 유연성이 더 뛰어나다.

이제 이들의 주요 차이점에 대해 살펴보자.

#### 성능

MySQL은 역사적으로 읽기가 많은 워크로드에 선호되어 웹 애플리케이션과 웹사이트에 널리 사용된다.

PostgreSQL의 아키텍처는 복잡한 쿼리 및 분석 워크로드에 더 적합합니다. 고급 SQL 기능이 필요한 시나리오에서 우수한 성능을 발휘한다.

#### ACID 규정 준수

원자성, 일관성, 격리성, 지속성(ACID)은 예상치 못한 오류가 발생한 후에도 데이터베이스를 유효한 상태로 유지하는 데이터베이스 속성이다. 예를 들어, 많은 수의 행을 업데이트했는데 중간에 시스템이 실패하는 경우 행을 수정해서는 안 됩니다.

MySQL은 대부분의 엔진은 ACID 규정 준수를 제공하지만 MyISAM은 ACID를 지원하지 않는다.

PostgreSQL은 모든 구성에서 ACID와 완벽하게 호환된다.

#### 동시성 제어

다중 버전 동시성 제어(MVCC)는 레코드의 중복 사본을 생성하여 동일한 데이터를 병렬로 안전하게 읽고 업데이트하는 고급 데이터베이스 기능이다. MVCC를 사용하면 여러 사용자가 데이터 무결성을 손상시키지 않고 동일한 데이터를 동시에 읽고 수정할 수 있다.

PostgreSQL은 동시 트랜잭션을 허용하는 MVCC 기능을 구현한 최초의 DBMS이다. 최신 버전의 MySQL도 MYVCC를 제공하지만, 일반적으로 MVCC에는 PostgreSQL이 가장 적합하다.

#### 인덱스

데이터베이스는 인덱스를 사용하여 데이터를 더 빠르게 검색한다. 자주 액세스하는 데이터를 다른 데이터와 다르게 정렬 및 저장하도록 데이터베이스 관리 시스템을 구성하여 자주 액세스하는 데이터를 인덱싱할 수 있다.

MySQL은 계층적으로 인덱싱된 데이터를 저장하는 B-트리 및 R-트리 인덱싱을 지원한다.

PostgreSQL 인덱스 유형에는 트리, 표현식 인덱스, 부분 인덱스 및 해시 인덱스가 포함된다. 크기를 확장할 때 데이터베이스 성능 요구 사항을 세밀하게 조정할 수 있는 더 많은 옵션이 있다.

#### 기본 SQL 데이터 유형

MySQL은 순수 관계형 데이터베이스로 표준 데이터 유형만 지원한다.

PostgreSQL은 객체 관계형 데이터베이스이다. 즉, PostgreSQL에서는 데이터를 속성을 가진 객체로 저장할 수 있다. 객체는 Java 및 .NET과 같은 여러 프로그래밍 언어의 일반적인 데이터 유형이다. 객체는 상위-하위 관계 및 상속과 같은 패러다임을 지원한다.

PostgreSQL은 데이터베이스 개발자에게 더 직관적인 경험을 제공한다. PostgreSQL은 배열 및 XML과 같은 다른 추가 데이터 유형도 지원한다.

#### 트리거

트리거는 데이터베이스 관리 시스템에서 관련 이벤트가 발생할 때 자동으로 실행되는 저장 프로시저이다.

MySQL은 SQL _INSERT_, _UPDATE_ 및 _DELETE_ 문에 _AFTER_ 및 _BEFORE_ 트리거만 사용할 수 있다. 즉, 사용자가 데이터를 수정하기 전이나 후에 트리거가 실행된다.

PostgreSQL은 _INSTEAD OF_ 트리거를 지원하므로 함수를 사용하여 복잡한 SQL 문을 실행할 수 있다.

<br>

### 3.3. 차이점 정리

|                  | **MySQL**             | **PostgreSQL**      |
| ---------------- | --------------------- | ------------------- |
| **DBMS 유형**      | RDBMS                 | ORDBMS              |
| **성능**           | 읽기 위주의 웹 애플리케이션에 적합   | 복잡한 쿼리와 분석 워크로드에 적합 |
| **ACID 규정 준수**   | MyISAM 제외, 지원         | 모든 구성에서 완벽 지원       |
| **동시성 제어(MVCC)** | 지원하지만 제한적             | 고도화된 MVCC 지원        |
| **데이터 유형**       | 표준 SQL 데이터 유형만 지원     | JSON, 배열 등 확장 가능    |
| **트리거 지원**       | AFTER, BEFORE 트리거만 지원 | INSTEAD OF 트리거까지 지원 |

<br>

### 3.4. PostgreSQL의 고급 기능에도 불구하고, 왜 여전히 MySQL을 선택할까?

풍부한 기능을 제공하는 PostgreSQL은 개발자들로부터 많은 사랑을 받고 있다. 그러나 특정 사용 사례에서는 MySQL의 단순함, 사용 편의성, 그리고 안정성이 훨씬 더 적합할 수 있다. 이러한 점에서 MySQL과 PostgreSQL은 각각 다른 영역에서 탁월한 성능을 발휘한다.

더 자세한 내용은 *[4. 선택 가이드]* 에서 확인할 수 있다.

<br>

### 3.5. 성능 테스트

차이점에 적혔듯이 읽기 위주의 웹 애플리케이션에서는 MySQL이 적합할까?라는 개념에 확신을 얻기 위해 간단한 성능 테스트를 진행하였다.

#### 3.5.1. 성능 테스트 초기 설정

##### 1) WSL(Windows Subsystem for Linux) 설치
Windows PowerShell을 관리자 권한으로 실행한다. 다음 명령어로 WSL을 설치한다.
```bash
wsl –install
```

##### 2) Sysbench 설치

```bash
sudo apt update
sudo apt install sysbench
```


##### 3) Sysbench 사용 : CPU 성능 테스트를 실행

```bash
sysbench cpu run
```

![](https://i.imgur.com/CBCgMzK.png)


#### 3.5.2. 읽기 성능 테스트 : MySQL 성능 평가

##### 1) 테이블 준비

데이터베이스에 필요한 테이블을 생성한다. oltp_read_only.lua 스크립트를 사용하여 읽기 전용 테이블을 준비할 수 있다.

```bash
sysbench /usr/share/sysbench/oltp_read_only.lua \
	--mysql-host=127.0.0.1 \
	--mysql-port=[PORT] \
	--mysql-user=[USER] \
	--mysql-password=[PASSWORD] \
	--mysql-db=[DATABASE] prepare
```

##### 2) 성능 평가 실행

```bash
sysbench /usr/share/sysbench/oltp_read_only.lua \
--mysql-host=127.0.0.1 \
--mysql-port=[PORT] \
--mysql-user=[USER] \
--mysql-password=[PASSWORD] \
--mysql-db=[DATABASE] run
```

![](https://i.imgur.com/OZctGzS.png)


응답 시간 (Latency) 평균 (avg): 34.57 ms

처리량 (Throughput)

- 초당 쿼리 수 (QPS): 4640 (462.45 쿼리/초)
- 트랜잭션 수: 290 (28.90 트랜잭션/초)

##### 결과

MySQL의 읽기 성능을 평가하는 데 유용하며, 응답 시간이 짧고 처리량이 높다는 것은 데이터베이스가 높은 성능을 유지하고 있음을 알 수 있다.

<br>

## 4. 선택 가이드

[JetBrains](https://www.jetbrains.com/lp/devecosystem-2022/databases/)의 설문조사에 따르면 MySQL과 PostgreSQL은 개발자들이 많이 사용하는 두 개의 주요 데이터베이스로, 서로 직접적인 경쟁 관계에 있다. 흥미롭게도, MySQL은 PostgreSQL 사용자들 사이에서 덜 선호되며, 그 반대도 마찬가지이다. 하지만 응답자의 19%는 두 데이터베이스를 모두 사용하는 것으로 나타났다. 이는 각 데이터베이스의 강점과 특성에 따라 프로젝트에 적합한 DBMS를 선택하는 경향을 보여준다.

![](https://i.imgur.com/vHDzgmB.png)

*그림 4 ‘지난 12개월 동안 어떤 데이터베이스를 사용하셨나요?’ 설문 결과*

그럼 이제 어떤 상황에서 어떤 SQL이 더 적절한지 일반적인 사용 사례와 함께 이야기하려고 한다.


### 4.1. MySQL이 더 적합한 상황

MySQL을 사용해야 하는 상황은 다음과 같다.

-      스토리지 엔진 유연성: 다양한 스토리지 엔진을 지원하여 여러 테이블 유형의 데이터를 유연하게 관리할 수 있다.
-      속도와 안정성: 단순한 구조로 높은 속도와 안정성을 제공한다. 특히 읽기 전용 작업에 탁월하지만, 복잡한 쿼리가 많은 경우 PostgreSQL이 더 적합할 수 있다.
-      사용 용이성: 설정이 간편하고, 경험 있는 관리자를 찾기 쉽다. 다양한 도구와 함께 사용자 친화적인 환경을 제공한다.
-      간단한 솔루션 필요 시: 기술적 복잡성이 낮고, 빠른 구축이 필요한 경우 MySQL이 적합하다.

### 4.2. PostgreSQL이 더 적합한 상황

PostgreSQL을 사용해야 하는 상황은 다음과 같다.

-      ORDBMS 필요 시: 객체 지향 프로그래밍과 관계형 데이터베이스의 결합을 통해 복잡한 데이터 구조를 처리할 수 있다.
-      복잡한 읽기-쓰기 작업: 유효성 검사가 필요한 복잡한 읽기-쓰기 작업을 수행할 때 적합하다.
-      초대형 데이터베이스 관리: 데이터베이스 크기에 제한이 없으며, 페타바이트(PB) 단위의 데이터를 처리할 수 있다.
-      MVCC 지원: 다중 버전 동시성 제어(MVCC)를 통해 여러 사용자가 동시에 안전하게 데이터를 읽고 쓸 수 있다.
-      ACID 준수: 모든 트랜잭션에서 데이터 무결성을 보장하며, ACID 규정을 완벽하게 준수한다.

<br>

## 5. 결론

결론적으로, PostgreSQL과 MySQL 중에서 하나를 선택하려면 다음 질문으로 정리할 수 있다.

### 처리 작업에 따른 선택 기준

- 주로 처리해야 할 작업이 읽기 작업인가? - MySQL
- 주로 처리해야 할 작업이 복잡한 읽기-쓰기 작업인가? - postgreSQL

### 데이터베이스 규모에 따른 선택 기준

- 다루어야 할 데이터베이스의 크기가 상대적으로 작은 데이터 세트를 다루는가? - MySQL
- 다루어야 할 데이터베이스의 크기가 초대형인가? - PostgreSQL

### 우선순위에 따른 선택 기준

- 속도와 유연성을 더 중시하는가? - MySQL
- 트랜잭션 처리에서 높은 ACID 준수가 중요한가? – PostgreSQL

### 기타 고려 사항

- 설정 및 관리가 쉽고 이해도가 높은 더 간단한 데이터베이스가 필요한가? - MySQL
- 데이터베이스에서 객체 지향적인 접근 방식을 사용할 필요가 있는가? - PostgreSQL

<br>

## 6. 참고

Oracle docs - [https://docs.oracle.com/cd/E17952_01/mysql-5.7-en/what-is-mysql.html](https://docs.oracle.com/cd/E17952_01/mysql-5.7-en/what-is-mysql.html)
PostgreSQL: Up and Running, 3rd Edition
PostgreSQL docs -  [https://www.postgresql.org/about/](https://www.postgresql.org/about/)
ORDBMS란 - [https://database.guide/what-is-an-ordbms/](https://database.guide/what-is-an-ordbms/)
AWS : PostgreSQL과 MySQL 비교 - [https://aws.amazon.com/ko/compare/the-difference-between-mysql-vs-postgresql/](https://aws.amazon.com/ko/compare/the-difference-between-mysql-vs-postgresql/)
Integrate.io : PostgreSQL과 MySQL 비교 - [https://www.integrate.io/ko/blog/postgresql-vs-mysql-which-one-is-better-for-your-use-case-ko/](https://www.integrate.io/ko/blog/postgresql-vs-mysql-which-one-is-better-for-your-use-case-ko/)
DB-Engines Ranking - [https://db-engines.com/en/ranking](https://db-engines.com/en/ranking)
Stackoverflow DB 관련 설문 조사 -  
[https://survey.stackoverflow.co/2022/?utm_source=results#most-popular-technologies-database](https://survey.stackoverflow.co/2022/?utm_source=results#most-popular-technologies-database)
Jetbrains DB 관련 설문 조사 - [https://www.jetbrains.com/lp/devecosystem-2022/databases/](https://www.jetbrains.com/lp/devecosystem-2022/databases/)

