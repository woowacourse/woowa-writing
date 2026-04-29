## 들어가며

개발을 하다 보면 한 번쯤은 Hibernate SQL 로그를 유심히 들여다보게 되는 순간이 있습니다.

분명 API 한 번만 호출했는데, 콘솔에는 비슷한 쿼리들이 수십 줄씩 반복되어 찍히는 걸 보면서 “왜 이렇게 많은 쿼리가 호출되지?”라는 궁금증이 생깁니다.

저희 프로젝트에서도 같은 일이 있었습니다.

GET /contents/keyword API 한 번 호출에 298개의 쿼리가 실행되고 있었습니다. 원인은 JPA의 지연 로딩(Lazy Loading)으로 인해 발생한 N+1 문제였습니다.

이 글은 그 문제를 추적하고, Fetch Join → Batch Size → @EntityGraph로 쿼리 개수를 줄이며 성능을 개선해 나간 과정을 기록한 글입니다.

## 문제 상황

운영 환경에서 API 응답 속도를 모니터링하던 중, 특정 엔드포인트(GET /contents/keyword)의 응답 시간이 다른 API보다 유독 길게 측정되는 현상을 발견했습니다.

처음에는 단순히 데이터 양이나 정렬 조건 때문이라고 생각했지만, 평균 응답 시간이 꾸준히 높게 유지되는 것을 보고 쿼리 레벨에서의 문제를 의심했습니다. 그래서 실제로 쿼리 개수를 측정해보기로 했습니다.

```java
"duration":"633ms","method":"GET","uri":"/contents/keyword"
```

### 쿼리 개수 측정

쿼리 개수를 확인하기 위해 Hibernate에서 제공하는 `StatementInspector` 인터페이스를 구현했습니다. 이 인터페이스의 `inspect()` 메서드는 하이버네이트가 실행할 SQL문을 가로채서 실행 전 후처리를 할 수 있도록 도와주는 기능을 제공합니다.

```java
@Component
public class QueryCountInspector implements StatementInspector {

    private final ThreadLocal<QueryCounter> queryCount = new ThreadLocal<>();

    public void startCounter() {
        queryCount.set(new QueryCounter(0L, System.currentTimeMillis()));
    }

    public QueryCounter getQueryCount() {
        return queryCount.get();
    }

    public void clearCounter() {
        queryCount.remove();
    }

    @Override
    public String inspect(String sql) {
        QueryCounter queryCounter = queryCount.get();
        if (queryCounter != null) {
            queryCounter.increaseCount();
        }
        return sql;
    }
}
```

이렇게 하면 하이버네이트가 실행하는 모든 쿼리의 개수를 측정할 수 있습니다.

이를 활용해 하나의 API 요청에서 10개 이상의 쿼리가 발생할 경우 경고 로그를 남기도록 설정했습니다.

```java
if (QueryCountInspector.getCount() > 10) {
    log.warn("하나의 요청에 쿼리가 10번 이상 발생했습니다. QUERY_COUNT: {}", QueryCountInspector.getCount());
}
```

### 측정 결과

쿼리 카운터를 적용한 후 로그를 확인해보니 GET /contents/keyword 요청 한 번에 298개의 쿼리가 실행되고 있었습니다.

```
"duration":"633ms","method":"GET","uri":"/contents/keyword"
"message":"하나의 요청에 쿼리가 10번 이상 발생했습니다. QUERY_COUNT: 298","method":"GET","uri":"/contents/keyword"
```

쿼리가 이렇게 많이 발생한 이유는 JPA의 기본 동작 방식인 지연 로딩 때문이었습니다.

Content → Place → PlaceCategory 로 이어지는 연관관계가 각 엔티티마다 별도의 쿼리를 발생시키며 N+1 문제를 만들고 있었던 것입니다.

## N+1 문제란?

N+1 문제란, 첫 번째 쿼리(1)의 결과로 N개의 데이터를 가져온 뒤, 연관된 데이터를 조회하기 위해 N개의 추가 쿼리가 실행되는 현상을 말합니다.

즉, 한 번의 요청으로 1 + N개의 쿼리가 발생하는 구조입니다. 데이터 양이 많아질수록 이 문제는 기하급수적으로 성능 저하를 일으키게 됩니다.

## 원인 분석

저희 프로젝트에서는 장소(`Place`)와 장소의 카테고리(`PlaceCategory`)가 아래와 같이 `@OneToMany` 관계로 설정되어 있고, 지연 로딩이 기본값으로 동작하고 있었습니다.

```java
@Entity
public class Place {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    // @OneToMany의 기본 Fetch 전략은 지연 로딩이다.
    @OneToMany(mappedBy = "place", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<PlaceCategory> placeCategories = new ArrayList<>();
    // ...
}
```

문제의 API 로직은 다음과 같은 흐름을 가지고 있었습니다.

1. Content 목록을 조회
2. 각 Content에 포함된 Place 목록을 조회
3. 각 Place의 카테고리(PlaceCategory)를 불러와 화면에 표시

코드는 단순했지만, 실행 시점에는 다음과 같은 쿼리 패턴이 발생했습니다.

```sql
// 1. 메인 컨텐츠에 포함된 Place 목록을 조회 (쿼리 1개)
select p1_0.id, p1_0.name from place p1_0 where p1_0.id in (?, ?, ...);

// 2. 조회된 각 Place(N개)의 Category를 얻기 위해 추가 쿼리 발생 (N개)
Hibernate:
    select pc1_0.place_id, ... from place_category pc1_0 where pc1_0.place_id=?
Hibernate:
    select pc1_0.place_id, ... from place_category pc1_0 where pc1_0.place_id=?
Hibernate:
    select pc1_0.place_id, ... from place_category pc1_0 where pc1_0.place_id=?
... (N번 반복)
```

즉, Place 목록을 한 번에 가져온 뒤, place.getPlaceCategories()가 호출되는 시점마다 추가 쿼리 N개가 실행되었습니다.

이것이 바로 전형적인 N+1 쿼리 패턴입니다.

## Fetch 전략

엔티티의 각 필드에는 JPA가 제공하는 데이터 조회 전략(Fetch) 설정이 가능합니다.

Fetch 전략에는 Lazy, Eager 2가지가 있습니다.

### Eager(즉시 로딩)

엔티티를 조회할 때 연관된 모든 데이터를 한 번에 가져옵니다. `@ManyToOne`, `@OneToOne`의 기본값입니다. 연관된 데이터가 항상 로드되어 있으므로 나중에 추가 쿼리가 발생하지 않습니다. 하지만 사용하지 않는 데이터까지 조회하여 비효율적일 수 있습니다.

### Lazy(지연 로딩)

연관된 데이터를 실제로 사용할 때까지 조회를 미룹니다. `@OneToMany`, `@ManyToMany`의 기본값입니다. 연관된 데이터를 실제로 조회하기 전에는 프록시(가짜) 객체로 가지고 있고, 실제 사용 시 추가 조회 쿼리가 발생하게 됩니다. 

예를 들어 Place 엔티티를 조회하면, placeCategories 필드는 실제 데이터가 아닌 프록시 객체로 채워집니다. 이후 `place.getPlaceCategories()`처럼 이 필드에 실제로 접근하는 코드가 실행될 때 데이터베이스에 조회를 요청합니다.

효율적이지만, 잘못 사용하면 위에서 본 N+1 문제를 유발합니다.

### 어떤 전략을 선택해야 할까?

즉시 로딩을 사용하면 문제를 피할 수 있을 것 같지만, 실제로는 완벽한 해결책이 아닙니다. 데이터를 조회하면 연관관계가 있는 엔티티는 신경 쓰지 않고, 조회 대상이 되는 엔티티만 즉시 가져옵니다. 조회 대상 엔티티를 가져온 이후 연관된 엔티티가 있다면 그때 연관 엔티티를 즉시 로딩합니다. 지연 로딩과 다르게 프록시 객체를 사용하진 않지만, 결국 N개의 추가 쿼리가 발생하게 됩니다.

결국 중요한 것은, 언제 어떻게 연관 데이터를 함께 가져올지를 명시적으로 제어하는 것입니다. 스프링 공식 문서에서도 즉시 로딩보다 지연 로딩을 기본으로 사용하고, 필요에 따라 연관 데이터를 함께 조회하는 방식을 권장합니다.

### 지연 로딩의 N+1 문제 해결법

### 1. Fetch Join

Fetch Join은 연관된 엔티티를 한 번의 쿼리로 함께 가져오도록 지정하는 JPQL 문법입니다. 

> JPA 공식 문서(Hibernate)의 권장 사항
"Performance tuning is about removing the additional selects. The best way to do that is a JOIN FETCH..." (성능 튜닝은 추가적인 select 구문을 제거하는 것입니다. 이를 위한 최고의 방법은 JOIN FETCH입니다.)
> 

하지만 `@OneToMany` 관계에 Fetch Join을 사용하면 카테시안 곱(Cartesian Product) 문제가 발생할 수 있습니다.

### 카데시안 곱이란?

데이터베이스에서 `JOIN`을 수행할 때, 1(One)에 해당하는 테이블의 행이 N(Many)에 해당하는 테이블의 행 개수만큼 복제되어 결과가 부풀려지는 현상을 말합니다.

예를 들어 다음과 같은 테이블이 있다고 가정합니다.

Content 테이블

| ID | 제목 |
| --- | --- |
| 1 | 부산 여행 |
| 2 | 서울 여행 |
| 3 | 제주 여행 |

ContentPlace 테이블

| ID | 장소명 | content_id |
| --- | --- | --- |
| 101 | 해운대 | 1 |
| 102 | 광안리 | 1 |
| 201 | 경복궁 | 2 |
| 202 | 남산타워 | 2 |
| 203 | 명동 | 2 |
| 301 | 한라산 | 3 |

이때 `Content`를 기준으로 `ContentPlace`를 `JOIN FETCH` 하는 JPQL을 실행하면, 데이터베이스는 내부적으로 다음과 같은 결과를 만듭니다. (총 2+3+1 = 6행)

이제 페이지 크기(size)를 2로 설정하여 첫 페이지를 조회한다고 가정해 봅시다. 개발자의 의도는 `Content` 2개, 즉 "부산 여행"과 "서울 여행"을 받아보는 것입니다.

하지만 데이터베이스는 `LIMIT 2`와 같은 페이징 쿼리를 위 `JOIN FETCH` 결과 테이블에 그대로 적용합니다.

이 결과를 받은 JPA(하이버네이트)는 `content.id`가 1인 `Content` 객체 하나와, 그에 속한 `ContentPlace` 2개를 만들어 애플리케이션에 반환합니다.

결과적으로, 개발자는 `Content` 2개를 기대했지만 실제로는 `Content` 1개("부산 여행")만 받게 됩니다. 이처럼 `@OneToMany` Fetch Join에서 페이징을 사용하면 데이터가 누락되는 문제가 발생합니다.

이러한 위험을 인지하고 있는 하이버네이트는, `@OneToMany` Fetch Join과 페이징을 함께 사용할 경우 경고 로그를 출력합니다.

```
WARN: HHH000104: firstResult/maxResults specified with collection fetch; applying in memory!

```

이 경고는 "컬렉션 fetch와 페이징을 함께 사용했으므로, DB 페이징을 포기하고 모든 결과를 메모리에 올린 후 애플리케이션 레벨에서 페이징하겠다"는 의미입니다. 위 예시에서는 6행의 데이터를 모두 DB에서 가져온 후, 메모리에서 2개의 `Content`를 추려내는 방식입니다.

데이터가 적을 때는 문제가 없어 보일 수 있지만, `Content`가 1,000개이고 각각 10개의 `ContentPlace`를 가진다면 총 10,000개의 행이 메모리에 로드되어 `OutOfMemoryError` 가 발생하게 됩니다.

### 2. Batch Size

이 때 `Batch Size` 옵션을 고려할 수 있습니다. `Batch Size` 옵션은 하이버네이트의 어노테이션을 사용하여 한 번에 조회할 데이터의 크기를 설정하는 방법입니다.

지연 로딩된 데이터를 초기화할 때, 한 번에 하나의 엔티티가 아닌, 지정된 `size`만큼의 ID를 모아 `IN` 절 쿼리로 한 번에 조회합니다.

- `application.yml`에 글로벌 설정을 추가하거나, 엔티티 필드에 `@BatchSize` 어노테이션을 붙입니다.
    
    ```yaml
    spring:
      jpa:
        properties:
          hibernate:
            default_batch_fetch_size: 100
    
    ```
    
- `size=100`일 때, 20개의 `Place`에 대한 `PlaceCategory`가 필요하다면, `WHERE place_id IN (?, ?, ...)` 쿼리 한 개로 모든 카테고리 정보를 가져옵니다.
- Fetch Join의 페이징 문제를 완벽하게 해결하면서, N개의 쿼리를 단 1개의 추가 쿼리로 줄여줍니다.

대부분의 주요 데이터베이스(MySQL, PostgreSQL 등)는 IN 절에 수천 개 이상의 값을 허용하기 때문에 Batch Size는 일반적으로 10에서 100사이 값을 많이 사용합니다.

## 적용하기

`FetchType.LAZY`옵션을 지정하여 조회 전략을 지연 로딩으로 변경하고, `default_batch_fetch_size` 를 100으로 설정한 후 다시 API를 호출하니 쿼리 개수가 7개로 줄어들었습니다.

```java
"QUERY_COUNT: 7", "method":"GET","uri":"/contents/keyword"
"duration":"216ms","method":"GET","uri":"/contents/keyword"
```

```java
// 1. 특정 Content에 속한 모든 ContentPlace 목록을 조회합니다.
Hibernate: 
    select
        cp1_0.id,
        cp1_0.content_id,
        ...
    from
        content_place cp1_0 
    left join
        content c1_0 
            on c1_0.id=cp1_0.content_id 
    where
        c1_0.id=?

// 2. 위에서 조회된 ContentPlace 목록(N개)에 대해 연관된 Place 정보를 조회합니다.
Hibernate: 
    select
        p1_0.id,
        p1_0.address,
        ... 
    from
        place p1_0 
// default_batch_fetch_size=100 설정으로 인해, N개의 쿼리가 아닌 한 개의 'IN'절 쿼리로 모든 Place 정보를 한 번에 가져옵니다.
    where
        p1_0.id in (?, ?, ?, ? ... ?, ?, ?, ?, ?)   
```

## @EntityGraph

저희 프로젝트 대부분의 화면에서는 Content를 조회할 때 항상 Creator와 City 정보가 함께 필요했습니다.
이 경우, `batch_fetch_size` 설정만으로는, `Content` 목록을 조회한 후 `creator`와 `city` 정보에 접근할 때 매번 Lazy Loading으로 인해 IN 쿼리가 두 번씩 발생했죠.

```java
@Entity
public class Content {
    // ...
    @ManyToOne(fetch = FetchType.LAZY)(fetch =FetchType.LAZY)
    @JoinColumn(name = "creator_id")
    private Creator creator;

    @ManyToOne(fetch = FetchType.LAZY)(fetch =FetchType.LAZY)
    @JoinColumn(name = "city_id")
    private City city;
    // ...
}
```

이러한 경우 `@EntityGraph` 를 사용할 수 있습니다. 

`@EntityGraph`는 특정 쿼리를 실행할 때, 지연 로딩으로 설정된 연관관계를 즉시 로딩하도록 만들어줍니다. JPQL의 `JOIN FETCH`와 유사한 역할을 하지만, 더 간결하고 재사용성이 높다는 장점이 있습니다.

```java
@EntityGraph(attributePaths = {"creator", "city"}, type = EntityGraph.EntityGraphType.FETCH)
Slice<Content> findByCityName(...);
```

- `@EntityGraph(attributePaths = {"creator", "city"})`: `Content`를 조회할 때, `creator`와 `city` 필드를 함께 `JOIN`하여 가져오도록 지정합니다.
- `type = EntityGraph.EntityGraphType.FETCH`: `attributePaths`에 명시된 속성은 EAGER로, 나머지 속성은 엔티티에 명시된 기본 Fetch 전략(LAZY)을 따릅니다.
- * `type = EntityGraph.EntityGraphType.LOAD`는 명시된 속성만 EAGER, 나머지는 기본 EAGER 전략을 따릅니다.

`@EntityGraph` 적용 후, Hibernate는 단 하나의 `JOIN` 쿼리를 생성하여 모든 정보를 한 번에 가져옵니다.

```sql
-- @EntityGraph 적용 후 실행되는 쿼리
SELECT c.id,
       c.title, ...,     -- Content 필드
    cr.id, cr.name, ..., -- Creator 필드
    ci.id, ci.name, ...  -- City 필드
    FROM
    content c
    LEFT OUTER JOIN
    creator cr
ON c.creator_id=cr.id
    LEFT OUTER JOIN
    city ci ON c.city_id=ci.id
WHERE
    c.city_name = ?
  AND c.id
    < ?
ORDER BY
    c.id DESC

```

## 부하테스트

@BatchSize와 @EntityGraph 적용 전후의 성능 변화를 수치로 확인하기 위해 k6로 부하 테스트를 진행했습니다.

테스트는 동일한 조건(100 VUs, 10초 동안, 요청 100회)에서 수행되었습니다.

```java
import http from "k6/http";
import { sleep } from "k6";

export const options = {
	vus: 100, // 동시에 100명 가상 유저
	duration: "10s", // 총 10초 동안 실행
	iterations: 100, // 총 100번만 실행 (유저 1번씩)
};

export default function () {
	let params = {
		headers: {
			"...",
		},
	};
	http.get("...", params);
	sleep(1);
}
```

테스트 결과는 평균 응답시간(avg), 95% 응답시간(p95), 초당 요청 수(throughput) 를 중심으로 비교했습니다.

| 구분 | 평균 응답시간 (avg) | 95% 응답시간 (p95) | 초당 요청 수 (http_reqs/s) |
| --- | --- | --- | --- |
| 개선 전 | 715.55 ms | 1.14 s | 46.46 /s |
| Batch Size, Lazy Loading 적용 | 492.73 ms | 798.49 ms | 55.38 /s |
| @EntityGraph 적용 | 270.84 ms | 418.03 ms | 69.99 /s |

즉, N+1 문제를 해결하면서 단순히 쿼리 수만 줄어든 것이 아니라, 전체 API 응답 속도와 서버 처리 효율이 모두 개선되었음을 확인할 수 있었습니다.

## 마무리하며

즉시 로딩(EAGER)은 단기적으로 편해 보이지만, 결국 불필요한 데이터 로드로 성능을 악화시킵니다. 반대로 지연 로딩(LAZY)은 효율적이지만, 무의식적인 접근 한 줄이 수백 개의 쿼리를 만들 수 있습니다.

N+1 문제는 단순히 쿼리가 많이 나가는 현상이 아니라, 데이터를 언제, 어떻게 불러올지에 대한 설계의 영역이 될 수 있다는 걸 느꼈습니다.
