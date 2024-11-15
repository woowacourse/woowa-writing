# MySQL vs PostgreSQL: 장단점 분석 및 선택 가이드

<br>

## 0. 주제를 선택한 이유

서비스에서 데이터베이스는 데이터를 저장하고 관리하는 역할을 합니다. 이는 서비스의 성능, 확장성, 보안에 큰 영향을 미칩니다. 따라서 데이터를 효율적으로 관리할 수 있는 데이터베이스 관리 시스템(DBMS)을 선택하는 것은 서비스의 핵심 결정 사항입니다.

오픈소스 DBMS인 MySQL과 PostgreSQL 중 어떤 DBMS을 사용할지는 프로젝트 요구 사항, 애플리케이션 특성, 그리고 개발팀의 경험과 선호도에 따라 달라집니다. 그럼에도 불구하고 많은 사람들이 MySQL과 PostgreSQL 중 어떤 것이 자신의 서비스에 더 적합한지 판단하기 어려워합니다.

이 글에서 두 데이터베이스를 간략하게 소개하고 주요 차이점을 비교하여 선택 가이드를 제공하고자 합니다.

이 글은 기본적인 데이터베이스 개념과 MySQL 문법에 대한 이해를 전제하에 MySQL 관련 개념은 간단하게 설명하고 넘어갑니다.

<br>

## 1. MySQL이란?


MySQL은 세계에서 가장 널리 사용되는 오픈 소스 관계형 데이터베이스 관리 시스템(RDBMS)입니다. [DB-Engines](https://db-engines.com/en/ranking)에 따르면 MySQL은 Oracle Database의 뒤를 이어 두 번째로 널리 사용되는 데이터베이스입니다. Facebook, Twitter, Netflix 등 세계적인 애플리케이션들이 MySQL을 사용하고 있습니다.

![image](https://github.com/user-attachments/assets/c9d0b345-f808-4ce2-b438-b257ac1508eb)

*그림 1 DB-Engines Ranking*

### 1.1. MySQL의 주요 기능

MySQL은 성능과 안정성 측면에서 다양한 주요 기능을 제공하여 개발자들이 효율적인 데이터 관리와 확장성을 구현할 수 있도록 돕습니다. 또한 MySQL의 여러 도구와 인터페이스는 사용 편의성을 보장합니다. 

이제 MySQL의 주요 기능들을 살펴보겠습니다.

- **멀티스레딩 지원**: MySQL은 커널 스레드를 사용하여 여러 CPU를 효율적으로 활용할 수 있습니다.
- **트랜잭션 및 비 트랜잭션 스토리지 엔진**: 다양한 스토리지 엔진을 제공하여 유연한 데이터 처리 기능을 지원합니다.
- **확장성**: 수십억 개의 데이터 행과 수십만 개의 테이블을 관리할 수 있어 대규모 데이터베이스 운영에 적합합니다.
- **다양한 연결 옵션**: TCP/IP 소켓, Unix 도메인 소켓 등 다양한 연결 방식을 통해 폭넓은 환경에서 사용할 수 있습니다.

<br>

## 2. PostgreSQL란?

PostgreSQL은 고급 기능을 제공하는 객체 관계형 데이터베이스 관리 시스템(ORDBMS)입니다. ORDBMS는 RDBMS와 ODBMS의 결합으로 데이터를 객체로 표현할 수 있는 시스템입니다. PostgreSQL은 이러한 ORDBMS 특성으로 데이터를 더 자연스럽고 유연하게 관리할 수 있으며, 복잡한 애플리케이션의 요구에 맞게 확장할 수 있습니다.

### 2.1. ORDBMS란?

DBMS는 관계형 데이터베이스 관리 시스템(RDBMS), 객체 지향 데이터베이스 관리 시스템(ODBMS), 객체-관계형 데이터베이스 관리 시스템(ORDBMS) 등 다양한 형태로 존재합니다.

ORDBMS는 RDBMS와 ODBMS의 장점을 결합한 시스템입니다. 이를 통해 데이터베이스는 객체의 속성, 상속, 다형성 같은 개념을 지원하며, 보다 유연한 데이터 관리가 가능합니다. 특히 복잡한 데이터 구조를 다루거나 비정형 데이터를 처리할 때 ORDBMS는 더 자연스러운 표현과 관리를 제공합니다.

### 2.2. PostgreSQL의 주요 기능

PostgreSQL은 복잡한 데이터 워크로드를 안전하게 저장할 수 있는 다양한 기능을 제공합니다. 또한 PostgreSQL의 강력한 기능들은 개발자들이 데이터베이스를 효율적으로 관리할 수 있도록 돕습니다. 

이제 PostgreSQL의 주요 기능들을 살펴보겠습니다.

- **고급 데이터 유형:** JSON, 배열, 사용자 정의 데이터 타입을 지원합니다.
- **트랜잭션 처리:** ACID 준수로 안정적인 데이터 무결성을 보장합니다.
- **확장성:** 대규모 데이터 처리에 적합하며, 다양한 복제 및 샤딩 옵션을 제공합니다.
- **고급 쿼리 기능:** 서브쿼리, 윈도우 함수, CTE(Common Table Expressions)를 지원합니다.
- **사용자 정의 함수:** 다양한 프로그래밍 언어로 함수를 작성할 수 있습니다.

<br>

## 3. MySQL VS PostgreSQL

[Stack Overflow](https://survey.stackoverflow.co/2022/?utm_source=results#most-popular-technologies-database)와 [JetBrains](https://www.jetbrains.com/lp/devecosystem-2022/databases/)의 설문조사에 따르면 두 데이터베이스 모두 개발자들이 선호하는 데이터베이스임을 알 수 있습니다. Stack Overflow에서 진행한 설문조사를 보면 MySQL 46.85%, PostgreSQL 43.59%로 막상막하의 점유율을 보여주고 있습니다.

![image](https://github.com/user-attachments/assets/82c696b1-dc73-44f5-bfe3-a631dad68a6d)

*그림 2 JetBrains에서 진행한 설문 조사 (모든 응답자)*

심지어 전문 개발자만 투표한 결과는 PostgreSQL 46.48%, MySQL 45.68%로 PostgreSQL가 1위 자리를 차지했습니다. 

![image](https://github.com/user-attachments/assets/b6c77194-7008-4d6e-9334-d8f2e405cc77)

*그림 3 JetBrains에서 진행한 설문 조사 (전문 개발자)*

그럼 두 종류의 DBMS 중에 무엇을 선택할지 고민하기 전에 이들의 공통점과 차이점을 살펴봅시다.

<br>

### 3.1. MySQL과 PostgreSQL의 공통점

어떤 DBMS를 사용할지 고민하는 이유는 두 데이터베이스가 여러 공통점을 가지고 있으며 그 사용 방식이 유사하기 때문입니다. 

따라서 먼저 이들의 공통점을 살펴봅시다.

- **구조화된 쿼리 언어(SQL)**: 두 데이터베이스 모두 SQL을 인터페이스로 사용하여 데이터를 읽고 편집할 수 있습니다.
- **오픈 소스**: MySQL과 PostgreSQL은 모두 오픈 소스이며, 강력한 개발자 커뮤니티의 지원을 받습니다.
- **데이터 백업 및 복제**: 데이터 백업, 복제 및 액세스 제어 기능이 내장되어 있습니다.
- **ACID 준수**: 두 데이터베이스 모두 트랜잭션의 원자성, 일관성, 격리성, 지속성을 보장합니다.
- **확장성**: MySQL과 PostgreSQL 모두 대규모 데이터베이스 운영에 적합하며, 수많은 데이터 행과 테이블을 관리할 수 있습니다.
- **플랫폼 독립성**: 다양한 운영 체제에서 사용할 수 있어 유연성이 높습니다.
- **다양한 클라이언트 라이브러리**: 다양한 프로그래밍 언어를 지원하는 클라이언트 라이브러리를 제공하여 쉽게 통합할 수 있습니다.

두 DBMS의 공통점에 대해 더 알고 싶다면 SQL 키워드 학습을 추천합니다. [» SQL에 대해 읽어보기](https://aws.amazon.com/what-is/sql/)  
  
<br>

### 3.2. MySQL과 PostgreSQL의 차이점

이렇듯 MySQL과 PostgreSQL는 개념적으로는 유사합니다. 하지만 두 DBMS는 구현하기 전에 고려해야 할 많은 차이점이 있습니다.

MySQL은 데이터를 행과 열이 있는 테이블로 저장할 수 있는 RDBMS입니다. 그리고 많은 웹 애플리케이션, 동적 웹 사이트 및 임베디드 시스템을 지원하는 널리 사용되는 시스템입니다.
PostgreSQL은 MySQL보다 더 많은 기능을 제공하는 ORDBMS입니다. 데이터 유형, 확장성, 동시성 및 데이터 무결성에 있어 유연성이 더 뛰어납니다.

이제 이들의 주요 차이점에 대해 살펴봅시다.

|                  | **MySQL**             | **PostgreSQL**                 |
| ---------------- | --------------------- | ------------------------------ |
| **DBMS 유형**      | RDBMS                 | ORDBMS                         |
| **성능**           | 읽기 위주의 웹 애플리케이션에 적합   | 복잡한 쿼리와 분석 워크로드에 적합            |
| **ACID 규정 준수**   | MyISAM 제외, 지원         | 모든 구성에서 완벽 지원                  |
| **동시성 제어(MVCC)** | 지원하지만 제한적             | 고도화된 MVCC 지원                   |
| **인덱스 유형**       | B-트리, R-트리 지원         | 트리, 표현식 인덱스, 부분 인덱스, 해시 인덱스 지원 |
| **데이터 유형**       | 표준 SQL 데이터 유형만 지원     | JSON, 배열 등 확장 가능               |
| **트리거 지원**       | AFTER, BEFORE 트리거만 지원 | INSTEAD OF 트리거까지 지원            |


#### 성능

MySQL은 역사적으로 읽기 중심의 워크로드에 최적화되어 있어 웹 애플리케이션과 웹사이트에서 널리 사용됩니다. 반면, PostgreSQL의 아키텍처는 복잡한 쿼리 및 분석 워크로드에 더 적합하여 고급 SQL 기능이 필요한 시나리오에서 우수한 성능을 발휘합니다.

#### ACID 규정 준수

원자성, 일관성, 격리성, 지속성(ACID)은 데이터베이스가 예상치 못한 오류 발생 후에도 유효한 상태를 유지하도록 보장하는 속성입니다. 예를 들어 많은 수의 행을 업데이트하는 도중 시스템이 실패하면 수정된 행이 남아서는 안 됩니다.

MySQL은 대부분의 스토리지 엔진에서 ACID 규정 준수를 제공하지만 MyISAM은 ACID를 지원하지 않습니다. 반면, PostgreSQL은 모든 구성에서 ACID와 완벽하게 호환됩니다.

#### 동시성 제어

다중 버전 동시성 제어(MVCC)는 레코드의 중복 사본을 생성하여 동일한 데이터를 병렬로 안전하게 읽고 업데이트할 수 있게 하는 고급 데이터베이스 기능입니다. MVCC를 통해 여러 사용자가 데이터 무결성을 손상하지 않고 동일한 데이터를 동시에 읽고 수정할 수 있습니다.   

PostgreSQL은 동시 트랜잭션을 허용하는 MVCC 기능을 구현한 최초의 DBMS입니다. 최신 버전의 MySQL도 MVCC를 지원하지만 일반적으로 MVCC에서 PostgreSQL이 더 우수한 성능을 보입니다.   

#### 인덱스 유형

데이터베이스는 인덱스를 사용하여 데이터를 더 빠르게 검색합니다. 자주 액세스하는 데이터를 다른 데이터와 다르게 정렬하고 저장하도록 데이터베이스 관리 시스템을 구성함으로써, 빠른 검색이 가능해집니다.   

MySQL은 B-트리 및 R-트리 인덱싱을 지원하여 계층적으로 인덱스 된 데이터를 저장합니다. 반면, PostgreSQL은 트리, 표현식 인덱스, 부분 인덱스, 해시 인덱스 등 다양한 인덱스 유형을 제공하여 데이터베이스 성능 요구 사항을 세밀하게 조정할 수 있는 옵션을 더 많이 제공합니다.

#### 데이터 유형

MySQL은 순수 관계형 데이터베이스로 표준 데이터 유형만을 지원합니다. 이에 비해 PostgreSQL은 객체 관계형 데이터베이스로 속성을 가진 객체로 저장할 수 있습니다. 이러한 객체는 Java와 같은 여러 프로그래밍 언어에서 일반적으로 사용되는 데이터 유형이며 상위-하위 관계 및 상속과 같은 개념을 지원합니다.   
 
PostgreSQL은 데이터베이스 개발자에게 더 직관적인 경험을 제공하며 배열 및 XML과 같은 추가 데이터 유형도 지원합니다.   

#### 트리거 지원

트리거는 데이터베이스 관리 시스템에서 특정 이벤트가 발생할 때 자동으로 실행되는 저장 프로시저입니다.   

MySQL에서는 `INSERT`, `UPDATE`, `DELETE` 문에 대해 `_AFTER_` 및 `_BEFORE_` 트리거만 사용할 수 있습니다. 즉, 사용자가 데이터를 수정하기 전이나 후에 트리거가 실행됩니다.   

반면, PostgreSQL은 `_INSTEAD OF_` 트리거를 지원하여 함수를 사용해 복잡한 SQL 문을 실행할 수 있는 유연성을 제공합니다.   

<br>

### 3.3. PostgreSQL의 고급 기능에도 불구하고, 왜 여전히 MySQL을 선택할까?

풍부한 기능을 제공하는 PostgreSQL은 개발자들로부터 많은 사랑을 받고 있습니다. 그러나 특정 사용 사례에서는 MySQL의 단순함, 사용 편의성, 그리고 안정성이 더욱 적합할 수 있습니다. 다음과 같은 이유로 MySQL이 여전히 선택되는 경우가 많습니다.

- **단순성**: MySQL은 구조가 간단하고 직관적이어서 초기 설정과 유지 관리가 용이합니다. 이에 따라 작은 프로젝트나 스타트업에서 많이 선호됩니다.
    
- **사용 편의성**: MySQL은 많은 사용자에게 익숙하며 다양한 도구와 라이브러리가 지원되어 개발 속도를 높일 수 있습니다.
    
- **안정성**: MySQL은 오랜 역사와 많은 사용 사례를 보유하고 있어 안정성이 검증되었습니다. 이러한 특성으로 인해 대규모 웹 애플리케이션에서 널리 사용됩니다.
    
- **커뮤니티와 지원**: MySQL은 방대한 사용자 커뮤니티와 다양한 리소스를 통해 빠른 문제 해결과 지원을 받을 수 있습니다.
    

따라서 MySQL과 PostgreSQL은 각기 다른 영역에서 탁월한 성능을 발휘하며 프로젝트의 요구 사항과 환경에 따라 선택하는 것이 중요합니다. 더 자세한 내용은 *[4. 선택 가이드]* 에서 확인할 수 있습니다.

<br>

## 4. 선택 가이드

[JetBrains](https://www.jetbrains.com/lp/devecosystem-2022/databases/)의 설문조사에 따르면 MySQL과 PostgreSQL은 개발자들이 가장 많이 사용하는 두 주요 데이터베이스로, 서로 직접적인 경쟁 관계에 있습니다. 흥미롭게도 MySQL은 PostgreSQL 사용자들 사이에서 덜 선호되며, 그 반대의 경우도 마찬가지입니다. 그런데도 응답자의 19%는 두 데이터베이스를 모두 사용한다고 응답하였습니다. 이는 각 데이터베이스의 강점과 특성에 따라 프로젝트에 적합한 DBMS를 선택하는 경향을 보여줍니다. 이러한 선택의 다양성은 개발자들이 특정 요구 사항에 맞춰 최적의 솔루션을 찾고 있음을 시사합니다.

![image](https://github.com/user-attachments/assets/7c02ef71-5859-4367-8f0d-e0d3d31bc5a6)

*그림 4 ‘지난 12개월 동안 어떤 데이터베이스를 사용하셨나요?’ 설문 결과*

이제 어떤 상황에서 어떤 DBMS가 더 적절한지 일반적인 사용 사례와 함께 이야기해 봅시다.


### 4.1. MySQL이 더 적합한 상황

MySQL을 사용해야 하는 상황은 다음과 같습니다.

- **스토리지 엔진 유연성**: 다양한 스토리지 엔진을 지원하여 여러 테이블 유형의 데이터를 유연하게 관리할 수 있습니다.
- **속도와 안정성**: 단순한 구조로 높은 속도와 안정성을 제공합니다. 특히 읽기 전용 작업에 탁월하지만 복잡한 쿼리가 많은 경우에는 PostgreSQL이 더 적합할 수 있습니다.
- **사용 용이성**: 설정이 간편하고, 경험 있는 관리자를 찾기 쉬우며 다양한 도구와 함께 사용자 친화적인 환경을 제공합니다.
- **간단한 솔루션 필요**: 기술적 복잡성이 낮고, 빠른 구축이 필요한 경우 MySQL이 적합합니다.

### 4.2. PostgreSQL이 더 적합한 상황

PostgreSQL을 사용해야 하는 상황은 다음과 같습니다.

- **ORDBMS 필요**: 객체 지향 프로그래밍과 관계형 데이터베이스의 결합으로 인해 복잡한 데이터 구조를 처리할 수 있습니다.
- **복잡한 읽기-쓰기 작업**: 유효성 검사가 필요한 복잡한 읽기-쓰기 작업을 수행할 때 적합합니다.
- **초대형 데이터베이스 관리**: 데이터베이스 크기에 제한이 없으며 페타바이트(PB) 단위의 데이터를 처리할 수 있습니다.
- **MVCC 지원**: 다중 버전 동시성 제어(MVCC)를 이용해 여러 사용자가 동시에 안전하게 데이터를 읽고 쓸 수 있습니다.
- **ACID 준수**: 모든 트랜잭션에서 데이터 무결성을 보장하며 ACID 규정을 완벽하게 준수합니다.

이러한 상황들을 통해 각각의 데이터베이스가 어떤 환경에서 더 적합한지 명확하게 이해할 수 있습니다. 필요에 따라 적절한 데이터베이스를 선택하는 데 도움이 될 것입니다.

<br>

## 5. 결론

결론적으로, PostgreSQL과 MySQL 중에서 하나를 선택하려면 다음 질문으로 정리할 수 있습니다.

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
