제목: 서비스 중단 없이 DB 스키마 변경하기

# 문제 상황

백엔드 개발자라면 한 번쯤 ALTER TABLE 구문을 실행하다가 WAS가 먹통이 되는 경험을 해봤을 것이다. 테이블이 잠기면서 쿼리들이 대기 상태에 빠지고, 타임아웃이 연쇄적으로 발생하는 상황 말이다.  
물론 잠시 점검 시간을 갖고 서비스를 중단한 뒤 DDL을 실행하면 간단하다. 하지만 대규모 서비스를 운영 중인데, 상대적으로 작은 스키마 변경을 위해 전체 서비스를 중단해야 한다면? 그 시간 동안 발생하는 손실은 상상하기 어렵다.  
  
**그렇다면 서비스 중단 없이 안전하게 스키마를 변경할 방법은 없을까?**  
  
> **이 글은 스키마 변경으로 서비스를 중단해본 경험이 있거나, 지금 당장 스키마를 변경해야 하지만 서비스 중단이 부담스러운 개발자를 위한 실전 가이드다.** 단순한 개념 소개가 아닌, 실무에 바로 적용할 수 있는 구체적인 방법을 다룬다.

## 두 가지 문제

ALTER TABLE이 서비스에 영향을 주는 이유는 크게 두 가지다.  
  
**1. 데이터베이스 계층의 문제: 테이블 락**
- DDL 실행 중 테이블에 락이 걸려 DML이 차단된다
- 대용량 테이블일수록 락 시간이 길어진다
- ex) 1억 row 테이블의 컬럼 수정 시 수십 분간 락 발생
  
**2. 애플리케이션 계층의 문제: 스키마 불일치**
- DDL 실행 시점과 애플리케이션 배포 시점의 시간차
- 변경된 테이블 구조와 기존 애플리케이션 코드의 불일치
- ex) 컬럼 추가 후 배포 전까지 신규 데이터의 해당 컬럼이 NULL로 저장됨
  
각 문제를 해결하기 위해서는 **데이터베이스 계층**과 **애플리케이션 계층** 양쪽 모두의 대응이 필요하다. 하나씩 살펴보자.



# 데이터베이스 계층: 테이블 락 문제 해결

서비스를 무중단으로 유지하려면 DDL 실행 중 테이블 락을 최소화해야 한다. MySQL은 5.6 버전부터 Online DDL 기능을 제공하여 이 문제를 해결하고 있다.


## DDL 실행 알고리즘

MySQL은 DDL을 실행할 때 내부적으로 다음 세 가지 알고리즘 중 하나를 사용한다.  
  
**INSTANT(MySQL 8.0.12+)**  
메타데이터만 수정하고 테이블 데이터는 건드리지 않는다. 거의 즉시 완료된다.  
```sql
-- 예: 컬럼 추가 (끝에)
ALTER TABLE users ADD COLUMN email VARCHAR(255);
-- 실행 시간: 0.01초 (테이블 크기 무관)
```
  
**INPLACE**  
테이블을 제자리에서 수정한다. 임시 테이블을 만들지 않아 상대적으로 빠르고, 대부분 락이 걸리지 않는다.  
```sql
-- 예: 인덱스 추가
ALTER TABLE users ADD INDEX idx_email (email);
-- 실행 시간: 테이블 크기에 비례하지만 락 없음
```
  
**COPY**  
임시 테이블을 생성하고 데이터를 전부 복사한 뒤 원본과 교체한다. 실행 중 테이블이 잠긴다.  
```sql
-- 예: 컬럼 타입 변경
ALTER TABLE users MODIFY COLUMN name VARCHAR(200) NOT NULL;
-- 실행 시간: 오래 걸림 + 락 발생
```

## 락(LOCK) 수준
DDL 실행 시 테이블에 걸리는 락의 수준도 알고리즘과 함께 결정된다. 락 수준에 따라 DML 작업의 허용 범위가 달라진다.  
  
**LOCK=NONE**  
락이 걸리지 않는다. DDL 실행 중에도 읽기/쓰기 모두 가능하다.  
```sql
ALTER TABLE users ADD COLUMN email VARCHAR(255), LOCK=NONE;
-- 실행 중: SELECT ✓ / INSERT ✓ / UPDATE ✓ / DELETE ✓
```
  
**LOCK=SHARED**  
읽기 락만 허용한다. 읽기는 가능하지만 쓰기는 차단된다.  
```sql
ALTER TABLE users ..., LOCK=SHARED;
-- 실행 중: SELECT ✓ / INSERT ✗ / UPDATE ✗ / DELETE ✗
```
  
**LOCK=EXCLUSIVE**  
배타적 락이 걸린다. 모든 읽기/쓰기가 차단된다.  
```sql
ALTER TABLE users ..., LOCK=EXCLUSIVE;
-- 실행 중: SELECT ✗ / INSERT ✗ / UPDATE ✗ / DELETE ✗
```
  
>**ALGORITHM과 LOCK**  
>  
>MySQL은 DDL 종류별로 사용 가능한 ALGORITHM과 LOCK 수준이 정해져 있다.  
>  
> | ALGORITHM | 주로 사용되는 LOCK | DML 허용 | 의미 |
> |-----------|------------------|---------|------|
> | INSTANT | NONE | ✓ 전부 가능 | **Online DDL** |
> | INPLACE | NONE | ✓ 전부 가능 | **Online DDL** |
> | COPY | SHARED/EXCLUSIVE | ✗ 차단 | **Offline DDL** |

## Online DDL vs Offline DDL

이제 위 알고리즘을 바탕으로 Online과 Offline DDL을 구분해보자.  
  
### Online DDL

DDL 실행 중에도 테이블에 대한 읽기/쓰기가 가능하다. 락이 최소한으로 걸리기 때문에 서비스 중단 없이도 스키마를 변경할 수 있다.  
```sql
-- Online DDL: 컬럼 추가
ALTER TABLE users ADD COLUMN email VARCHAR(255);
-- ALGORITHM=INSTANT, LOCK=NONE
```
  
**특징**
- 실행 중 DML(SELECT, INSERT, UPDATE, DELETE) 가능
- 테이블 크기와 무관하게 서비스 영향 최소
- INSTANT 또는 INPLACE 알고리즘 사용

### Offline DDL

DDL 실행 중 테이블에 락이 걸려 DML이 차단된다. 테이블을 통째로 복사하는 방식이라 시간이 오래 걸리며, 운영 환경에서 실행 시 서비스 장애로 이어질 수 있다.  
```sql
-- Offline DDL: 컬럼 타입 변경
ALTER TABLE users MODIFY COLUMN age VARCHAR(10) NOT NULL;
```
  
**특징**
- 실행 중 테이블에 락 발생
- 임시 테이블 생성 -> 데이터 복사 -> 테이블 교체
- 데이터가 많을수록 락 시간 증가
- COPY 알고리즘 사용

## 판단 기준

1. 공식 문서 확인  
    - MySQL 공식 문서에서 각 DDL이 어떤 알고리즘을 사용하는지 확인할 수 있다.  
    - [[MySQL] Online DDL Operations](https://dev.mysql.com/doc/refman/8.4/en/innodb-online-ddl-operations.html)  
  
2. 직접 테스트
    - 확실하게 확인하려면 로컬/개발 환경에서 알고리즘을 명시해서 실행해보자.  
  
```sql
-- ALGORITHM과 LOCK을 명시
ALTER TABLE users
ADD COLUMN email VARCHAR(255),
ALGORITHM=INPLACE, LOCK=NONE;

-- 성공 -> Online DDL 가능, 그대로 실행됨.
-- 에러 -> Offline DDL만 가능. 명시한 조건으로는 실행 안 됨.

-- 에러 예시
-- ERROR 1846 (0A000): ALGORITHM=INPLACE is not supported. 
-- Reason: Cannot change column type INPLACE. Try ALGORITHM=COPY.
```
  
ALGORITHM과 LOCK을 명시하지 않으면 MySQL이 자동으로 COPY 알고리즘을 선택할 가능성이 있다. 따라서 운영 환경에서 Online DDL을 실행할 경우 `ALGORITHM=INPLACE, LOCK=NONE`을 명시하여 의도치 않은 Offline DDL 실행을 방지하자.


## Offline DDL의 Online Migration

앞서 Offline DDL은 실행 중 모든 DML을 차단한다고 했다. 그럼 Offline DDL은 Online Migration이 불가능한 것일까?  
  
결론부터 말하자면, 그렇지 않다. Offline DDL 실행 중 테이블에 락이 걸리는 이유가 무엇일까?  

COPY 알고리즘을 사용하는 Offline DDL은 내부적으로 스키마 변경사항이 반영된 신규 테이블을 만들고, 여기에 기존 테이블 데이터를 복사한다. 문제는 이 복사 과정에서 발생한다. 데이터를 복사하는 동안 
기존 테이블에 데이터 변경(INSERT, UPDATE, DELETE)이 발생하면 신규 테이블에는 이 변경사항이 반영되지 않는다. 이렇게 되면 데이터 일관성이 깨지기 때문에 MySQL은 테이블에 락을 걸어 모든 쓰기 작업을 차단한다.  

그렇다면 복사 중 발생하는 변경사항을 신규 테이블에도 실시간으로 반영할 수 있다면 어떨까? 락을 걸지 않아도 일관성을 유지할 수 있을 것이다. 이 아이디어를 통해 Offline DDL의 Online Migration을 실현할 수 있다.  

### Offline DDL Online migration 도구

시중에는 Offline DDL online migration 도구가 여럿 있다. 여기서는 pt-online-schema-change와 gh-ost를 간단하게 알아보자.  
  
**pt-online-schema-change**  
2011년 Perocona Toolkit에서 출시한 도구로, 복제 테이블에서 작업하며 변경사항은 트리거에 의해 반영된다. 즉 원본 테이블에 락이 걸리지 않아 Offline DDL 실행 간 DML을 허용한다.  
  
동작 과정  
1. 임시 테이블 생성 후 스키마 변경사항 반영
2. 대상 테이블 -> 임시 테이블 트리거 생성
3. 대상 테이블 -> 임시 테이블 데이터 복사
   - 복사 간 대상 테이블 변경사항은 트리거에 의해 실시간 반영
4. 대상 테이블 <-> 임시 테이블 교체
  
**gh-ost**  
2016년 Github에서 출시한 도구로, pt-online-schema-change와 같이 임시 테이블(복제본)에서 작업하며 원본 테이블의 락을 회피한다. 하지만 이를 구현한 방법이 다른데, 다른 도구는 트리거 기반으로 변경사항을 반영하는 반면 gh-ost는 Binary Log Stream을 기반으로 변경사항을 반영한다. 덕분에 트리거로 인해 생기는 여러 제약과 위험을 회피했다.  

동작 과정  
1. 임시 테이블 생성 후 스키마 변경사항 반영
2. MySQL 데이터베이스에 자신을 레플리카로 위장하여 등록
- 마스터의 모든 변경사항을 Binary Log Streaming으로 수신하여 대상 테이블에 관한 내용만 필터링
3. 대상 테이블 -> 임시 테이블 데이터 복사
- 복사 간 대상 테이블 변경사항은 Binary Log Streaming에 의해 수신 및 반영
4. 대상 테이블 <-> 임시 테이블 교체
  
**pt-online-schema-change vs gh-ost**  
  
![](https://i.imgur.com/qsv2UK3.png)  
  
대부분의 상황에서 gh-ost의 성능이 더 좋지만, FK를 지원하지 않는다는 치명적인 단점이 있다. 따라서 실행하고자 하는 Offline DDL의 특성을 파악하여 어울리는 스키마 마이그레이션 도구를 선택해야 한다.  



# 애플리케이션 계층: 스키마 불일치 문제 해결

이번에는 애플리케이션 계층 문제를 살펴보자.  
  
![](https://i.imgur.com/EGvVGx3.png)  
  
위 그림은 first_name, last_name을 full_name으로 통합하는 상황으로, 애플리케이션 코드는 기존 스키마에 맞춰 작성되어 있다. DDL로 테이블에 full_name이 추가되더라도, 애플리케이션은 여전히 first_name과 last_name만 사용한다. 이처럼 스키마 변경 이후부터 애플리케이션 재배포 사이의 시간 갭 동안 불일치가 발생하며, 이는 서비스 중단으로 이어진다.  
  
DDL을 Online으로 실행하든 Offline으로 실행하든 스키마 불일치 문제를 해결하지 않으면 서비스 중단은 피할 수 없다.  
  
이 문제를 해결하기 위해 우리는 확장 축소 패턴을 사용할 수 있다.  


## 확장 축소 패턴

![](https://i.imgur.com/VbICbDG.png)  
  
확장 축소 패턴(Expand Contract Pattern)은 스키마 변경 시 확장에 대한 부분을 먼저 반영하고, 이후에 기존 스키마를 축소하는 패턴이다. 이렇게 순서를 구분하면 뭐가 달라질까?  
  
![](https://i.imgur.com/FVbhXtp.png)  
  
이번에는 확장 이후의 스키마에 대응하도록 애플리케이션 코드를 작성했다. 이는 기존 스키마와 신규 스키마에 모두 대응되는 애플리케이션 코드이기도 하다. 이렇게 애플리케이션에서 양 쪽 스키마에 모두 대응하면 스키마 변경 완료 후 애플리케이션 코드를 재배포하지 않아도 이미 대응이 되는 상태라 문제가 없다.  
  
하지만 여기서 고려해야 할 부분이 두 가지 있다.  

### 쓰기

확장 단계에서 기존/신규 스키마 양 쪽에 모두 대응하는 애플리케이션을 배포한 시점부터는 신규 스키마에도 데이터가 함께 반영되어야 한다. 데이터를 읽는 대상이 기존 스키마에서 신규 스키마로 넘어갈 때 데이터 일관성을 보장하기 위해서이다.  
  
즉, 이 단계에서 애플리케이션은 기존/신규 스키마 양 쪽에 동시에 쓰기를 해야 한다. 이를 이중 쓰기(Dual Write) 기법이라 한다.  
  
![](https://i.imgur.com/WnLxNRz.png)  
  
> **💡 DDL 실행 중 변경사항 반영과 Dual Write의 차이**  
>  
> 앞서 `데이터베이스 계층: 테이블 락 문제`를 다룰 때 신규 테이블에 데이터 변경사항을 반영하기 위해 트리거나 바이너리 로그를 활용한다고 언급한 바 있다. 하지만 헷갈리지 말자.  
> 해당 문제는 DDL 실행 간 발생하는 변경사항을 반영하기 위한 것이고, Dual Write는 DDL 종료 이후 애플리케이션 재배포 시점부터 발생하는 변경사항을 반영하기 위한 것이다. 둘 다 데이터 일관성 보장을 위한 것은 맞지만 그 시점과 계층이 완전히 다르다.  

### 읽기

Dual Write를 적용한 애플리케이션은 기존 스키마에서 데이터를 읽어와야 한다. 바로 뒤에 언급하겠지만 Dual Write 애플리케이션 배포 시점에는 아직 기존 데이터가 신규 스키마로 마이그레이션되지 않았다. 따라서 신규 스키마에서 읽으면 NULL 값이 반환된다. 그래서 이 단계에서는 기존 스키마로부터 데이터를 읽어와야 한다.


## 데이터 마이그레이션

DB 스키마를 확장하고 Dual Write 애플리케이션을 배포했다면 이제 뭘 해야 할까?  
  
기존 데이터를 신규 스키마 구조에 맞게 마이그레이션해야 한다. 앞에서의 예시를 다시 살펴보자.  
  
![](https://i.imgur.com/VbICbDG.png)  
  
first_name + last_name을 full_name으로 바꾸는 스키마 변경이지만 확장 축소 패턴을 적용하면서 세 컬럼이 동시에 존재하게 된다. 확장 단계에서는 full_name이라는 비어있는 컬럼이 추가되었고, 축소 단계에서는 first_name과 last_name 기존 컬럼들이 제거된다. 그럼 기존의 first_name과 last_name 데이터들은 언제 full_name으로 마이그레이션될까? 그 시점은 Dual Write 애플리케이션이 배포된 이후이다. 백그라운드에서 데이터를 채운다고 하여 이 단계를 백필(Back Fill)이라 부른다.  
  
백필은 데이터 규모에 따라 긴 시간이 걸리는 작업이기 때문에 보통 배치 작업으로 수행한다.  


## 점진적 읽기 전환

데이터 마이그레이션까지 끝나면 변경된 스키마를 읽도록 변경한 애플리케이션을 재배포해야 한다. 그런데 만약 이 애플리케이션에서 버그가 발생하면 어떻게 될까? 모든 사용자가 에러를 겪을 수밖에 없다. 대규모 애플리케이션에서는 이 피해를 최소화하기 위해 Feature Flag 도입을 고려할 수 있다.  
  
Feature Flag는 애플리케이션 재배포 없이도 특정 기능을 켜고 끌 수 있는 스위치이다. 점진적 배포를 도와주는 도구로 활용할 수 있는데, 이를 활용하면 신규 스키마로의 읽기 전환을 1% -> 10% -> ... -> 100%와 같이 점진적으로 진행할 수 있다. 또한 재배포 없이 이 수치를 수정할 수 있기 때문에 신규 버전에서 버그를 확인하는 즉시 기존 버전으로 롤백할 수 있다.  


### Feature Flag와 카나리 배포의 차이

얼핏 보면 비슷해 보이지만, 중요한 차이가 있다.  
  
실제로 Feature Flag와 카나리 배포는 점진적 전환이라는 동일한 목적을 가지고 있지만, 계층의 차이가 존재한다.  
  
카나리 배포는 인프라 계층에서의 점진적 전환으로, 여러 요청을 보내면 기본적으로 서로 다른 WAS로 요청이 전달될 수 있다. 반면 Feature Flag는 DB/애플리케이션 계층에서의 점진적 전환으로, 식별자를 기준으로 flag를 평가하기 때문에 여러 요청을 보내도 항상 일관적인 응답을 반환한다.  
즉, Feature Flag는 카나리 배포와 달리 일관된 사용자 경험을 제공해줄 수 있다.  
  
이렇게 보면 Feature Flag가 무조건 카나리 배포보다 좋아보일 수 있지만 그렇지 않다.  
  
카나리 배포도 Sticky Session이나 hash 기반 라우팅 정책 설정 등 로드밸런서 설정을 수정하여 유사한 기능을 구축할 수 있다. 또한 Feature Flag를 사용하면 애플리케이션에 점진적 전환 관련 코드 추가가 불가피하다. 전환이 완료되면 Feature Flag 코드를 제거한 애플리케이션을 재배포해야 한다.  


# 정리

전체 흐름을 요약하면 다음과 같다.  
  
![](https://i.imgur.com/PgM3icr.png)  
  
각 단계를 구체적으로 살펴보자.  
  
[실습 저장소](https://github.com/songsunkook/db-migration-test)에 단계별 PR이 있으니 직접 확인해보면 이해에 도움이 될 것이다.  
  
**1. Expand: 스키마 변경 대상 추가(DDL)**
- 스키마 변경에 대해 확장(추가)만 하고 기존 스키마는 유지한다.
- Offline DDL이라면 Offline DDL Online Migration Tool 사용을 고려할 수 있다.
- 신규 데이터는 아직 NULL 상태이다.
- 애플리케이션은 아직 기존 스키마를 사용한다.
- https://github.com/songsunkook/db-migration-test/pull/1

```sql
ALTER TABLE users ADD COLUMN email VARCHAR(255);
```

**2. Dual Write: 기존/신규 스키마 쓰기 대응 애플리케이션 배포(APP)**
- 기존/신규 스키마 양쪽에 모두 쓰기 작업을 하는 애플리케이션을 배포한다.
- 읽기는 아직 기존 스키마를 사용한다.
- 신규 데이터는 양쪽에 동시 저장된다.
- https://github.com/songsunkook/db-migration-test/pull/2

```java
public void updateName(String firstName, String lastName) {
    this.firstName = firstName;
    this.lastName = lastName;
    this.fullName = firstName + " " + lastName;
}
```

**3. Back Fill: 기존 데이터 마이그레이션(BATCH)**
- 기존 데이터를 신규 스키마로 마이그레이션한다.
- Spring Batch 등을 통해 배치 처리한다.
- NULL인 신규 스키마를 실제 값으로 채운다.
- https://github.com/songsunkook/db-migration-test/pull/3

```sql
UPDATE users 
    SET full_name = CONCAT(first_name, ' ', last_name) 
    WHERE full_name IS NULL;
```

**4. Read Conversion: 읽기 전환 애플리케이션 배포(APP)**
- Feature Flag로 신규 스키마 읽기를 점진적 전환한다.
- 쓰기는 여전히 Dual Write를 유지한다.
- https://github.com/songsunkook/db-migration-test/pull/4

```java
public String getDisplayName() {
    if (featureFlag.isEnabled("use_full_name", id)) {
        return fullName; // 신규
    }
    return firstName + " " + lastName; // 기존 (Fallback)
}
```

**5. Clean Up: 코드 정리 애플리케이션 배포(APP)**
- Feature Flag를 제거한다.
- Dual Write를 제거한다.(신규 스키마만 Write한다)
- 기존 스키마 관련 코드를 제거한다.
- https://github.com/songsunkook/db-migration-test/pull/5

```java
public String getFullName() {
    return fullName; // Feature Flag와 Fallback 로직 제거
}
```

**6. Contract: 기존 스키마 대상 제거(DDL)**
- 기존 스키마를 제거한다.
- Offline DDL이라면 Offline DDL Online Migration Tool 사용을 고려할 수 있다.
- 애플리케이션이 더이상 기존 스키마를 참조하지 않는다.
- 데이터베이스와 애플리케이션의 스키마 마이그레이션이 완료된다.
- https://github.com/songsunkook/db-migration-test/pull/6

```sql
ALTER TABLE users 
    MODIFY COLUMN full_name VARCHAR(255) NOT NULL;
```


## 참고 자료

25년 10월 14일 기준 참고 자료  
  
[[MySQL Docs]15.1.9 ALTER TABLE Statement - Performance and Space Requirements](https://dev.mysql.com/doc/refman/8.0/en/alter-table.html#:~:text=%ED%83%9C%EB%B8%94%EB%A6%BF%20%ED%85%8C%EC%9D%B4%EB%B8%94%EC%9D%84%20%EC%BF%BC%EB%A6%AC%ED%95%98%EC%8B%AD%EC%8B%9C%EC%98%A4.-,Performance%20and%20Space%20Requirements,-%EC%84%B1%EB%8A%A5%20%EB%B0%8F%20%EA%B3%B5%EA%B0%84)  
[[MySQL Docs]17.12.1 Online DDL Operations](https://dev.mysql.com/doc/refman/8.4/en/innodb-online-ddl-operations.html)  
[[Percona Toolkit Docs]pt-online-schema-change](https://docs.percona.com/percona-toolkit/pt-online-schema-change.html)  
[[Github]gh-ost Repository](https://github.com/github/gh-ost?tab=readme-ov-file)  
[[Bytebase]gh-ost vs pt-online-schema-change in 2025](https://www.bytebase.com/blog/gh-ost-vs-pt-online-schema-change/)  
