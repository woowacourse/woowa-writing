# SLOW QUERY 성능 개선부터 시작해터 프로젝트 전체 구조 리펙터링까지 가게된 사연

---

## 문서 주제

moment 프로젝트를 진행하면서 slow query를 탐색 및 개선하기 위해 여러 방법을 시도하면서 최종적으로 프로젝트 전체 구조를 리펙토링 하게 된 전 과정을 소개합니다.

---

## 대상 독자

- 대용량 테스트 데이터를 이용한 쿼리 성능 개선에 관심이 있는 사람
- DB에 인덱스를 처음 걸어보는 사람
- 프로젝트 리팩토링에 관심이 있는 사람

---

## 문서 활용 계획

우아한 기술 블로그 공유/ 개인 블로그 업로드/ 프로젝트 팀원에게 공유

---

## 1. SLOW QUERY 탐색 및 문제 정의

### 1.1 개선 대상 쿼리 선정

Moment 프로젝트는 사용자의 이야기를 공유하고 댓글과 답글을 통해서 공감을 주고 받는 서비스입니다.

Moment라고 불리는 사용자의 이야기는 설정된 태그에 따라서 검색이 가능하며 랜덤으로 사용자에게 하나씩 분배가 되는 구조를 가지고 있습니다.

그리고 이렇게 상대방의 이야기에 공감하고 감사를 표현한 기록들을 마이페이지에서 모아볼 수 있는데요.

이렇게 Moment 랜덤으로 조회하고 상대방과 주고받은 공감의 기록들을 조회하는 것이 Moment 서비스의 핵심 기능이라고 할 수 있습니다.

<img src="image/%E1%84%8C%E1%85%B5%E1%84%8B%E1%85%A7%E1%86%AB%E1%84%89%E1%85%A9%E1%86%A8%E1%84%83%E1%85%A9_%E1%84%8B%E1%85%B2%E1%84%8C%E1%85%A5%E1%84%8B%E1%85%B5%E1%84%90%E1%85%A1%E1%86%AF%E1%84%85%E1%85%B2%E1%86%AF.png" alt="지연속도_유저이탈률" width="300">

<화면 로딩 지연에 따른 유저 이탈률>

출처 : https://www.pingdom.com/blog/page-load-time-really-affect-bounce-rate/

만약 서비스의 이용자가 늘어남에 따라서 핵심 기능 제공하는 API 들의 속도가 저하된다면 사용자 경험이 급감하며 유저 이탈률이 증가될 것입니다.

API 성능을 저하시키는 이유들을 여러가지가 있겠지만 그중 가장 중요한 것은 쿼리 성능이라고 생각합니다.

잘못 작성된 쿼리로 인하여 데이터를 탐색하는 속도 자체가 줄어든다면 아무리 고성능 자원을 부여한다 하여도 API 개선에는 한계가 존재합니다.

그렇기 때문에 핵심 기능 API에 대한 쿼리 성능 측정을 진행하게되었습니다.


### 테스트 데이터 환경 구축

먼저 쿼리 성능 측정을 위해서 유의미한 테스트 데이터를 구축합니다.

Moment 서비스는 1개의 Moment에 여러 개의 Comment가 작성되는 구조를 가지고 있기 때문에 도메인 특성에 최대한 가깝게 테스트 데이터 크기를 설정하였습니다.

쿼리 성능 측정을 위해서 생성한 테스트 데이터의 수는 아래와 같습니다.

| 엔티티 | 데이터 개수 |
|--------|-------------|
| User | 약 100 건 |
| Moment | 약 100만 건 |
| Comment | 약 1000만 건 |

각 엔티티 간의 관계는 다음과 같습니다.

| 관계 | 카디널리티 |
|------|------------|
| User — Moment | 1 : M |
| Moment — Comment | 1 : M |
| User — Comment | 1 : M |


테스트 데이터는 Java-Faker를 이용하여 실제 데이터와 유사한 형태로 구성하였습니다.

<img src="image/%E1%84%90%E1%85%A6%E1%84%89%E1%85%B3%E1%84%90%E1%85%B3%E1%84%83%E1%85%A6%E1%84%8B%E1%85%B5%E1%84%90%E1%85%A5%E1%84%8B%E1%85%B5%E1%84%86%E1%85%B5%E1%84%8C%E1%85%B5.png" alt="테스트_데이터" width="300">

---

## 2. 1차 개선 - EXPLAIN ANALYZE 및 INDEX 최적화

### 2.1 EXPLAIN & EXPLAIN ANALYZE

테스트 데이터 구성 후 핵심 API 중 나의 코멘트 조회 기능에서 사용하는 SELECT 쿼리에서 평균 32s라는 심각한 지연이 발생하는 것을 확인했습니다.

```sql
-- 문제의 쿼리
select c1_0.id, c1_0.commenter_id, ... 
from comments c1_0 
join moments m1_0 on m1_0.id=c1_0.moment_id 
join users m2_0 on m2_0.id=m1_0.momenter_id 
where (c1_0.deleted_at IS NULL) and c1_0.commenter_id=1 
order by c1_0.created_at desc, c1_0.id desc limit 11;
```

문제 쿼리로 인하여 클라이언트에게 적절한 시간에 응답을 보내지 못하여 재시도 요청이 다시 도달하고 이에 따라 최대 약 90s 까지 쿼리 처리 속도가 저하되는 것을 확인하였습니다.

또한 쿼리 처리 속도가 늦어짐에 따라 DB Connection Pool도 빠르게 고갈되었도 전체 서비스에 장애가 발생하는 상황까지 확인하게 되었습니다.

이렇게 큰 지연을 발생시키는 쿼리를 개선하기 위해 원인 분석이 필요했으며 이를 위해 쿼리해 대한 EXPLAIN 를 확인하였습니다.

옵티마이저는 아래와 같은 실행 계획을 세우고 있었습니다.

#### EXPLAIN

| id | select\_type | table | partitions | type | possible\_keys | key | key\_len | ref | rows | filtered | Extra |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| 1 | SIMPLE | c1\_0 | null | ref | fk\_comments\_moments,fk\_comments\_users | fk\_comments\_users | 8 | const | 192696 | 10 | Using where; Using filesort |
| 1 | SIMPLE | m1\_0 | null | eq\_ref | PRIMARY,fk\_moments\_users | PRIMARY | 8 | moment\_database.c1\_0.moment\_id | 1 | 100 | null |
| 1 | SIMPLE | m2\_0 | null | eq\_ref | PRIMARY | PRIMARY | 8 | moment\_database.m1\_0.momenter\_id | 1 | 100 | null |

먼저 explain을 확인해 보았을 때, Comments 테이블에서 'Using where'와 'Using filesort'가 발생하는 것을 확인할 수 있습니다.

이는 현재 인덱스(fk_comments_users)가 commenter_id만 포함하고 있어, deleted_at 필터링과 created_at 정렬을 위해 테이블 행을 직접 읽어야 하기 때문입니다.

이런 실행 계획 원하는 결과를 비효율적으로 탐색하고 있다는 증거이기 때문에 데이터 필터에 필요한 칼럼을 대상으로 index를 부여하여 더 효율적인 explain을 도출할 필요가 있다고 생각했습니다.

더 정확한 문제를 진단하기 위해서 Explain Analyze를 실행하여 현재 Explain에 대한 정확한 결과를 확인하였습니다 

```
-> Limit: 11 row(s)  (cost=255291 rows=11) (actual time=43063..43063 rows=11 loops=1)
    -> Nested loop inner join  (cost=255291 rows=192696) (actual time=43063..43063 rows=11 loops=1)
        -> Nested loop inner join  (cost=231204 rows=192696) (actual time=43063..43063 rows=11 loops=1)
            -> Sort: c1_0.created_at DESC, c1_0.id DESC  (cost=192684 rows=192696) (actual time=43063..43063 rows=11 loops=1)
                -> Filter: (c1_0.deleted_at is null)  (cost=192684 rows=192696) (actual time=12.6..42962 rows=87254 loops=1)
                    -> Index lookup on c1_0 using fk_comments_users (commenter_id=1)  (cost=192684 rows=192696) (actual time=12.6..42952 rows=91724 loops=1)
            -> Single-row index lookup on m1_0 using PRIMARY (id=c1_0.moment_id)  (cost=0.999 rows=1) (actual time=0.00733..0.00735 rows=1 loops=11)
        -> Single-row index lookup on m2_0 using PRIMARY (id=m1_0.momenter_id)  (cost=0.25 rows=1) (actual time=0.00256..0.00259 rows=1 loops=11)
```

Explain Analyze 를 확인해보면 현재 어느 지점에서 가장 시간이 많이 소요되고 있는지 확인할 수 있습니다. 

Explain Analyze 은 들여쓰기가 깊을수록 먼저 실행된 것인데, 실행 순서대로 확인을 해보면 먼저 commenter_id가 1인 91724개의 레코드를 탐색한 것을 확인할 수 있습니다.

이어서 deleted_null에 대한 filter 작업이 수행되며, create_at 정렬 작업이 이어지는 것을 확인할 있습니다.

결국 마지막으로 commenter_id가 1이면서 deleted_at이 null이고 created_at이 내림차순으로 정렬된 테이블에서 11개의 레코드가 필요한 것이므로 복합 인덱스를 설정하여 준다면 가장 오랜 시간을 차지하고 있는 filter 실행 시간을 대폭 감소하여 쿼리 성능을 개선할 수 있다고 보았습니다.


### 2.2 복합 인덱스 적용 및 JOIN으로 인한 실행 계획 불안정성

Explain Analyze 을 확인하고 설계한 내용대로 복합 인덱스를 사용하여 나의 모멘트 쿼리에 대한 쿼리 최적화를 진행하였습니다.

먼저 복합 인덱스의 대상 칼럼은 commenter_id, created_id, id로 정했습니다. filter에 deleted_at과 관련된 조건도 들어가 deleted_at 칼럼도 복합 인덱스에 추가해주는 것이 더 빠른 성능을 낼 수도 있습니다.

하지만 deleted_at의 NULL 비율이 90% 이상으로 높고, deleted_at을 복합 인덱스에 추가하지 않아도 약 12~15개 행만 추가로 읽으면 되므로 성능 영향이 미미하여 제외하였습니다.

```sql
CREATE INDEX idx_comments_commenter_created_id ON comments (commenter_id, created_at DESC, id DESC);
```

이렇게 복합 인덱스를 적용하여 Explain을 확인해 보았을 때, 아래와 같은 결과가 나오는 것을 확인할 수 있었습니다.

| id | select\_type | table | partitions | type | possible\_keys | key | key\_len | ref | rows | filtered | Extra |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | SIMPLE | m2\_0 | null | ALL | PRIMARY | null | null | null | 109 | 10 | Using where; Using temporary; Using filesort |
| 1 | SIMPLE | m1\_0 | null | ref | PRIMARY,fk\_moments\_users | fk\_moments\_users | 8 | moment\_database.m2\_0.id | 10989 | 10 | Using where |
| 1 | SIMPLE | c1\_0 | null | ref | PRIMARY,fk\_comments\_moments,fk\_comments\_users,idx\_comments\_commenter\_created\_id | fk\_comments\_moments | 8 | moment\_database.m1\_0.id | 10 | 0.49 | Using where |

Explain을 보면 user 테이블에 대한 scan을 먼저 진행하는 것을 확인할 수 있었습니다.

쿼리를 작성했을 때 의도한 것은 Comment 테이블에서 commenter_id=1인 레코드를 먼저 필터링하여 탐색 범위를 최소화한 후, Moment와 User를 JOIN하는 것이었습니다.

```
-> Limit: 11 row(s)  (actual time=31772..31772 rows=11 loops=1)
    -> Sort: c1_0.created_at DESC, c1_0.id DESC, limit input to 11 row(s) per chunk  (actual time=31772..31772 rows=11 loops=1)
        -> Stream results  (cost=164479 rows=599) (actual time=664..31693 rows=82940 loops=1)
            -> Nested loop inner join  (cost=164479 rows=599) (actual time=664..31473 rows=82940 loops=1)
                -> Nested loop inner join  (cost=41935 rows=11978) (actual time=0.465..2953 rows=921594 loops=1)
                    -> Filter: (m2_0.deleted_at is null)  (cost=11.2 rows=10.9) (actual time=0.0277..0.413 rows=109 loops=1)
                        -> Table scan on m2_0  (cost=11.2 rows=109) (actual time=0.0269..0.335 rows=109 loops=1)
                    -> Filter: (m1_0.deleted_at is null)  (cost=2757 rows=1099) (actual time=0.28..26.5 rows=8455 loops=109)
                        -> Index lookup on m1_0 using fk_moments_users (momenter_id=m2_0.id)  (cost=2757 rows=10989) (actual time=0.28..25.6 rows=8900 loops=109)
                -> Filter: ((c1_0.commenter_id = 1) and (c1_0.deleted_at is null) and ((c1_0.created_at < TIMESTAMP'2025-09-20 10:30:00') or ((c1_0.created_at = TIMESTAMP'2025-09-20 10:30:00') and (c1_0.id < 100))))  (cost=9.21 rows=0.05) (actual time=0.0297..0.0308 rows=0.09 loops=921594)
                    -> Index lookup on c1_0 using fk_comments_moments (moment_id=m1_0.id)  (cost=9.21 rows=10.2) (actual time=0.0157..0.03 rows=9.5 loops=921594)

```

하지만 Explain Analyze를 확인해보면 실제로는 User → Moment → Comment 순서로 JOIN이 진행되고 있습니다. 

이로 인해 Comment 인덱스 조회가 92만 번(loops=921594) 반복되면서 여전히 성능이 개선되지 않고 여전히 31초가 소요되는 결과가 발생하였습니다.

MySQL 옵티마이저가 잘못된 조인 순서를 선택한 것이 주요 원인으로 파악되어 추가 최적화가 필요한 상황입니다.


### 2.3 JOIN으로 인한 실행 계획 불안정성을 해결하기 위한 여러 문제 해결 방법


옵티마이저가 잘못된 조인 순서로 실행 계획을 세우는 것을 바로 잡기 위해서 여러 해결책을 세우고 성능 개선을 시도하였습니다.

첫번째, 복합 인덱스 deleted_at 추가 

해당 문제를 증명하기 위해서 deleted_at을 이용한 복합 인덱스를 사용하여 성능 개선 여부를 확인하였습니다.

deleted_at을 복합 인덱스에 지정하여도 여전히 옵티마이저는 실행 계획을 고정하지 못하였고 근본적인 해결책이 되지 못하였습니다.

두번째, Force Index 또는 Straight Join

위 방법들은 옵티마이저가 설정한 Index를 사용하도록 강제하거나 예상한 Join을 우선적으로 실행하도록 제약 조건을 걸어주는 방법입니다.

해당 방법을 사용하면 옵티마이저는 실행 계획을 기존에 예상한 방향대로 고정시킬 수 있지만, 데이터 분포가 변경되면 옵티마이저의 통계 기반 최적화를 무시하게 되어 오히려 최적의 실행 계획을 선택하는데 문제가 발생할 수 있습니다.

그렇기 떄문에 힌트를 사용한 강제 최적화보다는 쿼리 구조 자체를 개선하거나 적절한 인덱스 설계를 통해 옵티마이저가 올바른 판단을 내릴 수 있도록 유도해야 합니다.


세번째, 쿼리 구조 개선

인덱스 개선이 아닌 쿼리 구조 개선을 통해서 쿼리 성능 최적화를 시도하였습니다.

먼저 옵티마이저가 comments 테이블을 먼저 바라보고 탐색 범위를 줄일 수 있는 방법에 대해서 고민하였고, 서브쿼리 방식을 이용하여 성능 개선을 시도하였습니다.

```sql
SELECT
    c1_0.id, c1_0.commenter_id, c1_0.content, c1_0.created_at, c1_0.deleted_at, c1_0.moment_id,
    m1_0.id, m1_0.content, m1_0.created_at, m1_0.deleted_at, m1_0.is_matched, m1_0.momenter_id,
    m2_0.id, m2_0.available_star, m2_0.created_at, m2_0.deleted_at, m2_0.email, m2_0.exp_star, m2_0.level, m2_0.nickname, m2_0.password, m2_0.provider_type, m1_0.write_type
FROM
    (SELECT *
     FROM comments
     WHERE commenter_id = 10 AND deleted_at IS NULL
     ORDER BY created_at DESC, id DESC
     LIMIT 11) AS c1_0
JOIN
    moments m1_0 ON m1_0.id = c1_0.moment_id AND m1_0.deleted_at IS NULL
JOIN
    users m2_0 ON m2_0.id = m1_0.momenter_id AND m2_0.deleted_at IS NULL
ORDER BY
    c1_0.created_at DESC, c1_0.id DESC;
```

먼저 위와 같이 FROM절에 서브쿼리를 작성해주어 comments 테이블에서 commenter id를 기준으로 11개의 행을 조회하는 쿼리가 먼저 실행되도록 해주려 하였습니다. 이 쿼리를 DB에서 EXPLAIN ANALYZE를 통해 실행해보아 저희가 정의한 복합 인덱스를 정상적으로 사용하는 것을 확인하였습니다.

하지만 서브쿼리 사용을 위해서는 Native Query를 작성해야 했고, Comment, Moment, User의 모든 컬럼을 조회하는 JOIN 쿼리였기 때문에 JPA Entity로 직접 매핑할 수 없어 별도의 DTO를 생성해야 했습니다. 

해당 방법은 성능 개선 효과가 있지만, 현재 시점에서는 JPQL 기반의 일관된 코드 스타일을 유지하는 것이 프로젝트 전반의 유지보수성 측면에서 더 중요하다고 판단하여 이 방법은 보류하였습니다.


### 2.4 지연 조인 전략(Deferred JOIN)의 도입

앞서 문제를 해결하기 위한 세가지 전략들이 있었지만 최종적으로 쿼리 성능 개선을 위해서 선택한 방법은 지연 조인 전략입니다.

## 지연 조인으로 분리된 쿼리 성능 측정하기

```java
 @Query("""
                SELECT c.id
                FROM comments c
                WHERE c.commenter = :commenter
                ORDER BY c.createdAt DESC, c.id DESC
            """)
    List<Long> findFirstPageCommentIdsByCommenter(@Param("commenter") User commenter, Pageable pageable);

    @Query("""
            SELECT c
            FROM comments c
            LEFT JOIN FETCH c.moment m
            LEFT JOIN FETCH m.momenter
            WHERE c.id IN :ids
            ORDER BY c.createdAt DESC, c.id DESC
            """)
    List<Comment> findCommentsWithDetailsByIds(@Param("ids") List<Long> ids);
```

이 전략은 쿼리를 두 단계로 나누어 먼저 필요한 Comment의 ID만 조회하고 그 ID들로 실제 데이터와 연관 엔티티를 조회하였습니다.

Comment의 ID만 조회하는 findFirstPageCommentIdsByCommenter() 메서드 쿼리의 성능 최적화를 위하여 Comments 테이블에 commemter_id, created_at, id를 사용하는 복합 인덱스 설정하였습니다.

결론적으로 성능은 개선되었고, 애플리케이션에서 JPQL 기반의 일관된 코드 스타일 또한 유지할 수 있었습니다.

!!!!!!!!!!! 쿼리 성능 재측정하여 첨부하기 !!!!!!!!!!!!!

---

## 3. 2차 개선 - 아키텍처 리팩토링을 통한 근본 문제 해결

### 3.1 지연 조인 전략의 성공과 새로운 의문

지연 조인 전략의 도입으로 MySQL 옵티마이저가 조인 순서를 매번 다르게 선택하던 문제를 해결하였고, 쿼리 성능을 안정화하는 데 성공하였습니다.

하지만 쿼리 최적화를 진행하면서 한 가지 근본적인 의문이 들었습니다.

"이 문제는 정말 쿼리만의 문제였을까?"

현재 Moment 프로젝트의 코드 중 일부를 살펴보겠습니다.

```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class CommentService {
    private final UserQueryService userQueryService;
    private final MomentQueryService momentQueryService;
    private final CommentRepository commentRepository;
    private final EchoQueryService echoQueryService;
    private final CommentQueryService commentQueryService;
    private final RewardService rewardService;
    private final NotificationFacade notificationFacade;
    private final CommentImageService commentImageService;
    private final MomentImageService momentImageService;
    private final NotificationQueryService notificationQueryService;
    private final MomentTagService momentTagService;
    private final ReportService reportService;
    private final EchoService echoService;
    private final PushNotificationSender pushNotificationSender;
}
```

당장 문제가 되었던 Comment 도메인만 살펴봐도, CommentService는 14개의 서로 다른 서비스와 컴포넌트에 의존하고 있었습니다. 

명확한 기준 없이 Repository 직접 조회, QueryService 호출, 다른 도메인 Service 의존이 혼재되어 있었고, 이는 프로젝트 초반 명확한 도메인 경계 설정 없이 기능 구현에만 집중했던 결과였습니다.

그리고 이러한 구조적 문제가 결국 복잡한 다중 테이블 JOIN으로 이어졌고, 쿼리 성능 불안정성이라는 형태로 나타나게 된 것입니다.

결론적으로 어떤 쿼리 최적화 기법을 적용하더라도 이 구조적 문제가 해결되지 않으면 근본적인 불안정성은 계속될 것이며 이러한 인식을 바탕으로, 단순히 쿼리를 최적화하는 것을 넘어 프로젝트 전체 아키텍처를 재설계하기로 결정했습니다.

### 3.2 Facade 패턴 도입 및 3계층 아키텍처 구축

#### 리팩토링 이전의 문제점

리팩토링 이전의 구조적 문제로 인하여 복잡한 다중 테이블 JOIN이 이어지는 것 이외에도 몇가지 문제를 더 찾아볼 수 있었습니다.

- 책임의 불명확성 : 하나의 서비스가 여러 도메인의 Repository와 Service에 직접 의존
- 높은 결합도 : 도메인 간 경계가 모호하여 변경 시 영향 범위가 불명확
- 복잡한 매핑 로직 : DTO 응답 생성을 위한 매핑 로직이 여러 곳에 분산
- 테스트 어려움 : 과도한 의존성으로 인해 단위 테스트 작성이 어려움

#### 3계층 구조 설계

이러한 문제를 해결하기 위해 다음과 같은 3계층 구조를 설계하게 되었습니다

```text
┌─────────────────────────────────────┐
│   Facade Layer (Controller와 통신)  │
│   - MomentFacadeService             │
│   - 여러 Application Service 조합   │
│   - DTO 반환                        │
└─────────────────────────────────────┘
              ↓
┌─────────────────────────────────────┐
│   Application Layer (도메인 조합)    │
│   - MomentApplicationService        │
│   - 관련 도메인 서비스들 조합        │
│   - Moment + MomentTag + MomentImage│
└─────────────────────────────────────┘
              ↓
┌─────────────────────────────────────┐
│   Domain Layer (단일 책임)          │
│   - MomentService                   │
│   - 자신의 Repository만 접근         │
│   - CRUD 및 비즈니스 로직           │
└─────────────────────────────────────┘
```

#### Facade 계층

```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class MyCommentPageFacadeService {

    private final CommentApplicationService commentApplicationService;
    private final MomentApplicationService momentApplicationService;
    private final NotificationApplicationService notificationApplicationService;

    public MyCommentPageResponse getMyCommentsPage(String nextCursor, int limit, Long commenterId) {
        // 1. Comment 조합 정보 조회
        CommentCompositions commentCompositions = commentApplicationService.getMyCommentCompositions(
                new Cursor(nextCursor), new PageSize(limit), commenterId);

        // 2. Moment 조합 정보 조회
        List<Long> momentIds = extractMomentIds(commentCompositions);
        List<MomentComposition> myMomentCompositions =
            momentApplicationService.getMyMomentCompositionsBy(momentIds);

        // 3. 알림 정보 조회
        Map<Long, List<Long>> unreadNotifications =
            notificationApplicationService.getNotificationsByTargetIds(...);

        // 4. 최종 DTO 조합 후 반환
        return MyCommentPageResponse.of(commentCompositions, myMomentCompositions, unreadNotifications);
    }
}
```

Facade 계층은 컨트롤러와 직접 통신하는 최상위 계층으로, 여러 도메인의 ApplicationService를 조합하여 API 응답을 생성합니다.

Facade 계층에서는 Repository에 직접 접근하지 않도록 규칙을 세웠습니다. 대신 각 도메인의 ApplicationService를 통해 독립적으로 데이터를 조회하고, 이를 메모리에서 조합합니다.

이는 한 번에 모든 데이터를 JOIN으로 가져오지 않고 필요한 것만 단계적으로 조회한 후 조합한다는 점에서 지연 조인 전략과 유사한 철학을 가지며, 이를 명확한 계층 구조로 구현한 것입니다.


#### Application 계층

```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class CommentApplicationService {

    private final UserService userService;
    private final CommentService commentService;
    private final CommentImageService commentImageService;
    private final EchoService echoService;

    public CommentCompositions getMyCommentCompositions(Cursor cursor, PageSize pageSize, Long commenterId) {
        // 1. 도메인 서비스를 통한 데이터 조회
        User commenter = userService.getUserBy(commenterId);
        List<Comment> comments = commentService.getCommentsBy(commenter, cursor, pageSize);

        // 2. 관련 엔티티 조회
        Map<Comment, CommentImage> commentImages = commentImageService.getCommentImageByComment(comments);
        Map<Comment, List<Echo>> echoes = echoService.getEchosOfComments(comments);

        // 3. DTO 조합 (Entity를 DTO로 변환)
        return mapToCommentCompositions(comments, commentImages, echoes);
    }
}
```

ApplicationService 계층은 단일 도메인 내 여러 서비스를 조합하는 계층으로, 동일한 도메인 영역 내의 서비스만 의존하도록 규칙을 정했습니다.
ApplicationService 계층에서는 Entity가 아닌 DTO를 반환하여 상위 계층에 도메인이 노출되는 것을 방지합니다. 
이를 통해 도메인 모델의 변경이 상위 계층에 영향을 주지 않도록 하고, 비즈니스 로직이 도메인 계층에서만 실행되도록 보장합니다.


#### Domain 계층

```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class CommentService {

    private final CommentRepository commentRepository;

    public List<Comment> getCommentsBy(User commenter, Cursor cursor, PageSize pageSize) {
        PageRequest pageable = pageSize.getPageRequest();

        if (cursor.isFirstPage()) {
            List<Long> commentIds = commentRepository.findFirstPageCommentIdsByCommenter(commenter, pageable);
            return commentRepository.findCommentsByIds(commentIds);
        }
        List<Long> commentIds = commentRepository.findNextPageCommentIdsByCommenter(
            commenter, cursor.dateTime(), cursor.id(), pageable);
        return commentRepository.findCommentsByIds(commentIds);
    }
}
```

단일 Repository와만 통신하며 해당 도메인의 CRUD 및 비즈니스 로직 처리를 담당합니다.

#### 리펙토링 성과

리팩토링은 진행한 후 각각의 도메인은 관련된 도메인에 대한 책임만 가지도록 완벽히 분리되었으며, 과도하게 집중되어 있었던 의존성이 도메인 별 계층의 책임에 맞게 고루 분포될 수 있었습니다.

```
리팩토링 전 :
CommentService - 14개 의존성
├─ UserQueryService
├─ MomentQueryService
├─ CommentRepository
├─ EchoQueryService
├─ RewardService
└─ ... (총 14개)

리팩토링 후 :
MyCommentPageFacadeService - 3개 의존성
├─ CommentApplicationService - 4개 의존성
│   ├─ UserService (단일 Repository)
│   ├─ CommentService (단일 Repository)
│   ├─ CommentImageService (단일 Repository)
│   └─ EchoService (단일 Repository)
├─ MomentApplicationService
└─ NotificationApplicationService
```

또한 하나의 거대한 서비스를 책임에 따라 분리하여 각 꼐층의 복잡도가 크게 감소하였으며, 계층의 역할과 의존성이 명확해짐으로 변경 영향 범위가 예측이 가능해였습니다.
그리고 단일 책임을 가진 서비스로 분리됨으로 단위 테스트 작성이 용이해졌습니다.

### 3.3 지연 조인 전환 및 쿼리 분리

아키텍처 리펙토링과 함꼐 쿼리 구조도 개선되었습니다. 

#### 지연 조인 전략 적용

기존의 복잡한 JOIN 쿼리를 2단계로 분리했습니다:
**1단계: ID만 조회** (CommentService에서 실행)
```java
@Query("""
    SELECT c.id
    FROM comments c
    WHERE c.commenter = :commenter
    ORDER BY c.createdAt DESC, c.id DESC
""")
List<Long> findFirstPageCommentIdsByCommenter(@Param("commenter") User commenter, Pageable pageable);
```
- 복합 인덱스 `idx_comments_commenter_created_id (commenter_id, created_at DESC, id DESC)` 활용
- 최소한의 데이터만 읽어 빠른 필터링 가능

**2단계: 실제 데이터 조회** (CommentService에서 실행)
```java
@Query("""
    SELECT c
    FROM comments c
    WHERE c.id IN :ids
    ORDER BY c.createdAt DESC, c.id DESC
""")
List<Comment> findCommentsByIds(@Param("ids") List<Long> ids);
```
- 1단계에서 찾은 ID 목록으로 정확히 필요한 행만 조회
- 불필요한 JOIN 제거

그리고 각 도메인 영역 별 간접 참조를 사용하여 예상하지 못한 N+1 쿼리가 발생되는 것을 방지하며 각 도메인 간 경계를 명확하게 나누었습니다.
그리고 필요한 데이터만 명시적으로 조회가 가능하

**리팩토링 전: 직접 참조**
```java
public class Comment {
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "moment_id")
    private Moment moment;  // 직접 참조

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "commenter_id")
    private User commenter;  // 직접 참조
}
```

**리팩토링 후: 간접 참조**
```java
public class Comment {
    @Column(name = "moment_id")
    private Long momentId;  // ID만 보관 (간접 참조)

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "commenter_id", insertable = false, updatable = false)
    private User commenter;  // 조회용으로만 유지
}
```

간접 참조를 사용하면 "언제, 무엇을, 얼마나 조회할지"를 개발자가 명확하게 코드로 작성할 수 있다는 의미입니다.

직접 참조 (연관관계) 사용 시

public class Comment {
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "moment_id")
private Moment moment;  // 직접 참조
}

// 사용 시
Comment comment = commentRepository.findById(1L);
String momentContent = comment.getMoment().getContent();
// ❌ moment를 조회할지 안 할지 코드만 봐서는 불명확
// ❌ 언제 쿼리가 나가는지 예측 어려움 (Lazy Loading)
// ❌ N+1 문제 발생 가능

간접 참조 사용 시

public class Comment {
@Column(name = "moment_id")
private Long momentId;  // ID만 보관 (간접 참조)
}

// 사용 시
Comment comment = commentRepository.findById(1L);
// moment가 필요하면 명시적으로 조회
Moment moment = momentRepository.findById(comment.getMomentId());
String momentContent = moment.getContent();
// ✅ moment를 조회하는 코드가 명확하게 보임
// ✅ 언제 쿼리가 나가는지 정확히 알 수 있음
// ✅ 필요 없으면 아예 조회하지 않음

실전 예시

// 직접 참조: 무엇을 조회하는지 불명확
List<Comment> comments = commentRepository.findAll();
// 이 시점에 moment도 함께 가져올까? User도 가져올까?
// LAZY라서 안 가져온다? 그럼 나중에 N+1 발생?
// → 예측 불가능

// 간접 참조: 필요한 것만 명시적으로 조회
List<Comment> comments = commentRepository.findAll();  // Comment만 조회
List<Long> momentIds = comments.stream()
.map(Comment::getMomentId)
.toList();
List<Moment> moments = momentRepository.findAllById(momentIds);  // Moment도 필요하면 명시적으로 조회
// → 코드만 봐도 무엇을 조회하는지 명확함

그리고 각 도메인 간 간접 참조를 사용하여 예상하지 못한 N+1 쿼리 발생을 방지하고, 도메인 간 경계를 명확하게 나누었습니다.

간접 참조를 통해 연관된 엔티티를 조회하는 시점과 범위를 명시적으로 코드에 표현할 수 있어, 성능 문제를 사전에 파악하고 제어할 수 있습니다.


### 3.4 최종 성능 개선 결과

최종 성능은 아직 리펙토링 후 측정하지 못하여 추가 할 예정입니다

---

## 4. 회고 및 배운 점
