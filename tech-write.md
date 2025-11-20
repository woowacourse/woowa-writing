# MySQL 쿼리 성능 측정 방법

# 목차

- 개요
- 쿼리 성능 측정 필요성과 방법
- Spring Data JPA 실제 쿼리 확인
- 쿼리 성능 측정 방법 EXPLAIN ANALYZE
- 쿼리 실행 계획 확인 방법 EXPLAIN
- SHOW PROFILING

# 개요

Spring Data JPA와 MySQL 데이터베이스를 사용하는 프로젝트에서 데이터베이스 쿼리 성능 측정 방법을 정리한 글입니다. 대상 독자는 MySQL 성능 최적화 필요 여부를 확인하고 싶은 사람, MySQL 성능 최적화가 잘 이루어졌는지 확인하고 싶은 사람입니다. MySQL의 SELECT, INSERT와 같은 데이터 조작 언어(DML)와 인덱스(Index), Spring Data JPA를 경험 후 읽기를 권장합니다.

사용된 프로그램의 버전은 다음과 같습니다.

- Java: JDK21
- Spring Boot: 3.5.3
- MySQL 서버: 8.0.43

# 쿼리 성능 측정 필요성과 방법

서비스에 사용자가 생기고, 데이터베이스에 데이터가 쌓이니 문자열 조회 쿼리가 느려졌습니다. 데이터베이스에 데이터 양이 증가하여 쿼리 성능이 낮아졌다고 생각했습니다. 이 예측을 확인하려면 데이터베이스에 데이터를 넣고 쿼리 성능을 측정해야 합니다. 데이터베이스에 가짜 데이터는 SQL 프로시저나 프로그래밍 언어로 삽입할 수 있습니다. 이 글에서 가짜 데이터 생성은 다루지 않습니다. 다양한 쿼리 측정 방식이 있지만 일정하지 않은 네트워크 소요 시간을 배재한 순수 쿼리 성능 측정을 위해 MySQL에서 제공하는 `EXPLAIN`과 `EXPLAIN ANALYZE` 를 사용합니다.

# Spring Data JPA 실제 쿼리 확인

데이터베이스에서 쿼리 성능을 측정하려면 대상이 되는 쿼리가 필요합니다. 성능 문제가 발생한 서비스에서 사용하는 쿼리는 Spring Data JPA가 생성합니다. 기본 설정의 Spring Data JPA는 생성하는 쿼리를 콘솔에 출력하지 않습니다. Spring Data JPA가 생성하는 쿼리를 콘솔에 출력하려면 Spring 설정을 변경해야 합니다. 콘솔 출력은 시스템 자원을 소모하므로 운영 환경에서는 출력 관련 설정을 끄는 것을 권장합니다.

```yaml
spring:
  jpa:
    show-sql: true
    properties:
      hibernate:
        format_sql: true          # 자동 줄바꿈 + 들여쓰기
        use_sql_comments: true
```

> - show-sql: true 
>   - Spring Data JPA 구현체 `하이버네이트(Hibernate)`가 생성하는 모든 SQL 쿼리를 콘솔에 출력합니다.
> - format_sql: true 
>   - 콘솔에 출력하는 SQL 문을 보기 좋게 줄바꿈과 들여쓰기를 사용하여 포맷(format)합니다.
> - use_sql_comments: true 
>   - 하이버네이트(Hibernate)가 생성하는 SQL에 주석을 추가합니다.

설정을 적용하고 API를 호출하면 콘솔에 Spring Data JPA가 생성한 SQL이 출력됩니다.

```sql
select
    l1_0.id 
from
    lineup l1_0 
left join
    festival f1_0 
        on f1_0.id=l1_0.festival_id 
        and (f1_0.deleted = 0) 
where
    (
        l1_0.deleted = 0
    ) 
    and f1_0.id=? 
    and l1_0.performance_at=? 
limit
    ?
```

이 방법으로 서비스에서 사용하는 Spring Data JPA 생성 쿼리를 확인했습니다. 다음 단계에서 방금 확인한 쿼리를 사용하여 성능을 측정하겠습니다. 

# 쿼리 성능 측정 방법 EXPLAIN ANALYZE

`EXPLAIN ANALYZE`는 MySQL 8.0 부터 제공되는 쿼리의 실행 시간을 분석하는 도구입니다. [MySQL 공식 문서: EXPLAIN ANALYZE](https://dev.mysql.com/blog-archive/mysql-explain-analyze)

`EXPLAIN ANALYZE`는 쿼리를 실제로 실행하고 쿼리의 각 영역별 동작 시간을 분석합니다. 사용 방법은 쿼리의 접두로 `EXPLAIN ANALYZE`를 붙입니다.

```sql
# EXPLAIN ANALYZE 예시
EXPLAIN ANALYZE
select l1_0.id
from lineup l1_0 left join festival f1_0 
    on f1_0.id = l1_0.festival_id and (f1_0.deleted = 0)
where (l1_0.deleted = 0)
  and f1_0.id = 2911
  and l1_0.performance_at = '2026-08-13 06:30:45.000000' limit 1;
```

### 쿼리 동작 순서

`EXPLAIN ANALYZE`는 계층 구조로 이루어진 데이터를 응답합니다. 각 쿼리의 영역별로 계층 구조가 나뉘며, 계층 구조의 안쪽부터 바깥쪽 순서로 수행됩니다. 

다음 결과는 `Index lookup...` -> `Filter: ...` -> `Limit: 1 row(s)...` 순서로 작업했다는 의미입니다.

```sql
-> Limit: 1 row(s)  (cost=7.09 rows=0.45) (actual time=0.0262..0.0262 rows=1 loops=1)
    -> Filter: ((l1_0.performance_at = TIMESTAMP'2026-08-13 06:30:45') and (l1_0.deleted = 0))  (cost=7.09 rows=0.45) (actual time=0.0255..0.0255 rows=1 loops=1)
        -> Index lookup on l1_0 using FK_LINEUP_ON_FESTIVAL (festival_id=2911)  (cost=7.09 rows=9) (actual time=0.0236..0.0236 rows=1 loops=1)

```

### 예상 비용과 실제 시간

데이터베이스가 해당 작업을 처리하는데 예상한 비용(Cost)과 실제 시간을 확인해보겠습니다. 여기서 말하는 비용(Cost)은 같은 작업을 처리하는 다양한 방법이 있을 때, 우선순위를 가리기 위해 필요한 자원을 추상적으로 수치화한 값입니다. 따라서 저희가 관심 있어하는 실제 시간을 주로 보면 됩니다.

```sql
-> Index lookup on l1_0 using FK_LINEUP_ON_FESTIVAL (festival_id=2911)  (cost=7.09 rows=9) (actual time=0.0236..0.0236 rows=1 loops=1)
```

살펴볼 곳은 `(cost=7.09 rows=9) (actual time=0.0236..0.0236 rows=1 loops=1)` 입니다. 

예상 비용 부분은 `(cost=7.09 rows=9)` 입니다. 데이터베이스는 예상 필요 자원 비용을 7.09, 예상되는 결과의 행의 개수를 9개로 판단했습니다.

실제 시간 부분은 `(actual time=0.0236..0.0236 rows=1 loops=1)` 입니다. `actual time의 왼쪽` 값 0.0236은 결과 대상의 첫번째 행을 찾는데 걸린 시간입니다. `actual time의 오른쪽` 값 0.0236은 모든 결과 대상의 행을 찾는데 걸린 시간입니다. `rows=1`는 실제 결과의 행의 개수입니다. 예제는 1개의 행을 반환했기에 왼쪽과 오른쪽의 시간이 동일합니다.
`loops=1`는 해당 계층이 동작한 수입니다. loops=2라면 두 번 작업이 되었다는 의미이며, `actual time의 왼쪽`과 `actual time의 오른쪽` 값은 이 반복된 작업들의 평균 시간을 나타냅니다. 따라서 `actual time의 오른쪽` * loops 가 해당 계층이 동작하는데 걸린 총 시간입니다.

### EXPLAIN ANALYZE 주의사항

`EXPLAIN ANALYZE` 명령어는 쿼리가 실제로 실행되고 반영됩니다. 따라서 `EXPLAIN ANALYZE`를 INSERT, UPDATE, DELETE와 같은 쓰기 작업과 데이터 정의어(DDL)과 같은 오토 커밋(Auto Commit)이 발생하는 쿼리에 사용하면 실제로 반영됩니다. 쓰기 작업의 성능을 측정하려면 트랜잭션을 사용하여 데이터베이스에 반영 되지 않도록 해야 합니다.

```sql
# 트랜잭션 시작
START TRANSACTION;

EXPLAIN ANALYZE
INSERT INTO `lineup` (
    `created_at`,
    `updated_at`,
    `deleted`,
    `deleted_at`,
    `festival_id`,
    `name`,
    `image_url`,
    `performance_at`
) VALUES (
    '1970-01-01 00:00:00.000000',
    '2025-08-22 13:04:21.068731',
    0,
    NULL,
    1,
    '제임스',
    'https://kr.object.ncloudstorage.com/matilda/4.png',
    '2025-10-13 20:00:00.000000'
    );

# 롤백으로 반영하지 않음
ROLLBACK;
```

### EXPLAIN ANALYZE 정리

쿼리를 실제로 실행하여 쿼리의 각 영역별 실제 소요시간을 사용하는 `EXPLAIN ANALYZE`을 알아봤습니다. 이를 통해 쿼리의 특정 구간에 큰 병목 지점을 확인하거나, 목표 응답 시간 달성 여부를 확인할 수 있습니다.

# 쿼리 실행 계획 확인 방법 EXPLAIN

`EXPLAIN` 명령어는 MySQL이 쿼리를 실행하는 방법에 대한 정보를 제공합니다. [MySQL 공식문서: EXPLAIN](https://dev.mysql.com/doc/refman/8.0/en/explain-output.html)

`EXPLAIN` 사용 방법은 쿼리 접두에 EXPLAIN을 붙이면 됩니다.

```sql
# EXPLAIN 예시
EXPLAIN
SELECT * FROM festabook.lineup;
```

`EXPLAIN` 접두가 붙은 쿼리는 실제로 동작하지 않습니다.

따라서 INSERT, UPDATE, DELETE 쓰기 작업 쿼리가 데이터베이스에 영향을 미치지 않습니다.

`EXPLAIN` 명령어는 쿼리가 각 테이블에 접근하는 방식을 행(Row) 단위로 보여주며, 정해진 열(Column)을 통해 실행 계획을 설명합니다

![image.png](img/2.png)

성능과 밀접하게 연관된 열은 type과 Extra입니다.

## Type

`EXPLAIN` 결과 열(Column)의 type 값은 데이터에 접근하는 방식을 알려줍니다.

type 값을 확인하면 인덱스를 잘 사용하는지, 개선이 필요한지 파악할 수 있습니다.

### Type 값

- system: 테이블에 데이터가 하나 뿐인 경우. 

- const: 쿼리로 탐색되는 행이 최대 한개인 경우.  
PRIMARY KEY, UNIQUE INDEX를 사용한 경우 주로 나타남.

- eq_ref: 드라이빙 테이블에서 읽은 하나의 행에 대해 조인되는 테이블에서 정확히 하나의 행만 찾는 경우.  
PRIMARY KEY나 UNIQUE NOT NULL 인덱스를 사용해야 한다.

- ref: 인덱스를 사용하여 조건에 맞는 여러 행을 찾는 경우.  
JOIN이나 WHERE 절에서 인덱스를 잘 사용했다는 의미이다.

- range: 인덱스를 사용해서 특정 범위의 행을 찾는 경우.

- index_merge: 여러 인덱스를 동시에 사용하여 결과를 병합하는 경우. (개선 고려)

- index: 풀 인덱스 스캔이 발생한 경우.  
커버링 인덱스이거나, ORDER BY, GROUP BY를 사용한 경우 발생함. (개선 필요)

- ALL: 모든 데이터를 확인하는 경우. 풀 테이블 스캔이라고도 불린다. (개선 필요)

## Extra

`EXPLAIN` 결과 열(Column)의 extra 값은 쿼리가 효율적으로 실행되는지 판단하는 정보를 알려줍니다.

extra 값으로 개선이 필요한지 파악할 수 있습니다.

### Extra 값

인덱스를 사용했는지, 비용이 큰 작업이 발생했는지 확인할 수 있습니다.

- Using index: 커버링 인덱스를 사용한 경우.

- Using index condition: 인덱스 사용하여 클러스터링 인덱스의 데이터 접근을 줄임.

- Using where: 인덱스만으로 조건을 처리할 수 없어서 클러스터링 인덱스에 접근한 경우. (개선 필요)

- Using filesort: ORDER BY 정렬을 수행함. (개선 필요)

- Using temporary: GROUP BY, ORDER BY, DISTINCT  작업을 위해 임시 테이블 생성 (개선 필요)

- Using join buffer (Block Nested Loop): 조인 데이터가 커서 메모리를 사용함 (개선 필요)

## EXPALIN 정리

실제 사용되는 쿼리를 `EXPLAIN` 명령어로 분석하면 MySQL에서 쿼리가 효율적으로 동작하는지 파악할 수 있습니다.

type과 extra에 개선이 필요한 값이 나타나면 쿼리에 적합한 인덱스를 생성하거나 쿼리를 수정하여 성능을 개선해야 합니다.

# SHOW PROFILING

`EXPLAIN ANALYZE`는 쿼리 실행 시간을 보여줍니다.

만약 쿼리 파싱, 최적화, 락(Lock)등 쿼리 시작부터 응답까지의 전체 소요 시간을 확인하려면 `SHOW PROFILING`을 사용해야 합니다.

`EXPLAIN ANALYZE`는 쿼리 자체의 효율의 개선을 확인하는 지표로 사용합니다.

`SHOW PROFILING`는 MySQL에서 쿼리 실행 전후의 부가적인 단계에 병목이 없는지 확인하는 지표로 사용합니다.

`SHOW PROFILING`는 다음 명령어를 사용합니다.
- `SET profiling = 1;`
- `SET profiling = 0;`
- `SHOW PROFILES;` 

```sql
# 쿼리 기록 시작 (이전 쿼리 기록이 사라집니다.)
SET profiling = 1;

select l1_0.id
from lineup l1_0
         left join festival f1_0 on f1_0.id = l1_0.festival_id and (f1_0.deleted = 0)
where (l1_0.deleted = 0)
  and f1_0.id = 2911
  and l1_0.performance_at = '2026-08-13 06:30:45.000000' limit     1;

# 쿼리 기록을 종료합니다.
SET profiling = 0;
```

```sql
# 쿼리 기록을 확인합니다.
SHOW PROFILES;
```

![image.png](img/2.png)

가장 중요한 열(Column) 값은 Duration 입니다.

Duration은 해당 쿼리에 소요된 전체 시간 값을 나타내며, 값의 단위는 초(Second)입니다.

`SHOW PROFILING` 기록을 지우고 새로 측정하려면, 
기록 종료 후 다시 기록을 시작하면 됩니다.

```sql
# 쿼리 기록 종료
SET profiling = 0;

# 새로운 쿼리 기록 시작 (이전 쿼리 기록 삭제)
SET profiling = 1;
```

## SHOW PROFILING 정리

`SHOW PROFILING`은 네트워크 소요시간과 웹 어플리케이션 서버 처리 시간을 제외한 순수 쿼리 전체 동작 속도를 측정할 수 있습니다.

# 마무리

EXPLAIN, EXPLAIN ANALYZE, SHOW PROFILES를 사용하여 개선할 쿼리를 찾고, 동작 시간을 파악하여 개선하는 방법을 알아봤습니다. 

MySQL 내부에서 동작하는 시간을 확인했습니다.

실제 웹 어플리케이션 응답 시간을 확인하기 위해서는 K6, JMeter 와 같은 웹 성능 분석 도구를 사용해야 합니다.