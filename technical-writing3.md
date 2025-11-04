# SLOW QUERY 성능 개선부터 프로젝트 전체 구조 리팩터링까지

---

## 들어가며

Moment 프로젝트를 진행하면서 slow query를 탐색하고 개선하는 과정에서 쿼리 최적화를 넘어 프로젝트 전체 구조를 리팩터링하게 된 여정을 공유합니다.

**대상 독자**
- 대용량 테스트 데이터를 이용한 쿼리 성능 개선에 관심이 있는 개발자
- 데이터베이스 인덱스를 처음 적용해 보는 개발자
- 레거시 프로젝트 리팩터링을 고민하는 개발자

---

## 1. SLOW QUERY 탐색 및 문제 정의

### 1.1 개선 대상 쿼리 선정

Moment는 사용자의 이야기를 공유하고 댓글과 답글을 통해 공감을 주고받는 서비스입니다. 사용자가 작성한 이야기(Moment)는 태그로 검색할 수 있으며, 다른 사용자에게 무작위로 전달됩니다. 사용자는 마이페이지에서 자신이 주고받은 공감의 기록을 확인할 수 있습니다.

이처럼 Moment를 무작위로 조회하고 공감 기록을 확인하는 기능은 서비스의 핵심입니다.

<img src="image/%E1%84%8C%E1%85%B5%E1%84%8B%E1%85%A7%E1%86%AB%E1%84%89%E1%85%A9%E1%86%A8%E1%84%83%E1%85%A9_%E1%84%8B%E1%85%B2%E1%84%8C%E1%85%A5%E1%84%8B%E1%85%B5%E1%84%90%E1%85%A1%E1%86%AF%E1%84%85%E1%85%B2%E1%86%AF.png" alt="화면 로딩 지연에 따른 유저 이탈률" width="500">

<화면 로딩 지연에 따른 유저 이탈률, 출처: https://www.pingdom.com/blog/page-load-time-really-affect-bounce-rate/>

위 자료에 따르면 화면 로딩 시간이 2초에서 3초로 늘어나면 이탈률이 9%에서 13%로 급격히 증가합니다. 서비스 이용자가 늘어나면서 핵심 API의 응답 속도가 느려진다면 사용자 경험이 크게 저하되고 이탈률이 높아질 것입니다.

API 성능 저하의 원인은 다양하지만, 그중에서도 쿼리 성능이 가장 중요하다고 판단했습니다. 잘못 작성된 쿼리는 데이터 탐색 속도 자체를 떨어뜨리기 때문에 아무리 고성능 서버를 투입해도 개선에 한계가 있습니다. 이에 따라 핵심 기능 API의 쿼리 성능 측정을 진행했습니다.

### 1.2 테스트 데이터 환경 구축

쿼리 성능을 정확히 측정하기 위해 실제 운영 환경과 유사한 대용량 테스트 데이터를 구축했습니다. Moment 서비스는 하나의 Moment에 여러 개의 Comment가 작성되는 구조이므로, 도메인 특성을 반영해 다음과 같이 테스트 데이터를 생성했습니다.

**테스트 데이터 규모**

| 엔티티 | 데이터 개수 |
|--------|-------------|
| User | 약 100건 |
| Moment | 약 100만 건 |
| Comment | 약 1,000만 건 |

**엔티티 간 관계**

| 관계 | 카디널리티 |
|------|------------|
| User - Moment | 1 : M |
| Moment - Comment | 1 : M |
| User - Comment | 1 : M |

테스트 데이터는 Java-Faker 라이브러리를 이용해 실제 데이터와 유사한 형태로 구성했습니다.

<img src="image/%E1%84%90%E1%85%A6%E1%84%89%E1%85%B3%E1%84%90%E1%85%B3%E1%84%83%E1%85%A6%E1%84%8B%E1%85%B5%E1%84%90%E1%85%A5%E1%84%8B%E1%85%B5%E1%84%86%E1%85%B5%E1%84%8C%E1%85%B5.png" alt="테스트 데이터 예시" width="500">

---

## 2. 1차 개선 - EXPLAIN ANALYZE 및 인덱스 최적화

### 2.1 EXPLAIN과 EXPLAIN ANALYZE로 문제 파악하기

테스트 데이터 구성 후, 나의 댓글 조회 기능에서 사용하는 SELECT 쿼리가 평균 32초라는 심각한 지연을 일으키는 것을 확인했습니다.

```sql
-- 문제의 쿼리
SELECT c1_0.id, c1_0.commenter_id, ... 
FROM comments c1_0 
JOIN moments m1_0 ON m1_0.id = c1_0.moment_id 
JOIN users m2_0 ON m2_0.id = m1_0.momenter_id 
WHERE (c1_0.deleted_at IS NULL) AND c1_0.commenter_id = 1 
ORDER BY c1_0.created_at DESC, c1_0.id DESC 
LIMIT 11;
```

이 쿼리는 클라이언트에게 적절한 시간 내에 응답을 보내지 못해 재시도 요청이 발생했고, 최대 90초까지 처리 시간이 늘어났습니다. 쿼리 처리가 지연되면서 DB 커넥션 풀(Connection Pool)이 빠르게 고갈되어 전체 서비스에 장애가 발생하는 상황까지 이어졌습니다.

이러한 심각한 지연의 원인을 파악하기 위해 쿼리 실행 계획을 분석했습니다. MySQL 옵티마이저(Optimizer)가 수립한 실행 계획은 다음과 같았습니다.

**EXPLAIN 결과**

| id | select_type | table | type | possible_keys | key | key_len | ref | rows | filtered | Extra |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| 1 | SIMPLE | c1_0 | ref | fk_comments_moments, fk_comments_users | fk_comments_users | 8 | const | 192,696 | 10 | Using where; Using filesort |
| 1 | SIMPLE | m1_0 | eq_ref | PRIMARY, fk_moments_users | PRIMARY | 8 | moment_database.c1_0.moment_id | 1 | 100 | null |
| 1 | SIMPLE | m2_0 | eq_ref | PRIMARY | PRIMARY | 8 | moment_database.m1_0.momenter_id | 1 | 100 | null |

EXPLAIN 결과를 보면 Comments 테이블에서 'Using where'와 'Using filesort'가 발생하는 것을 확인할 수 있습니다. 이는 현재 인덱스(fk_comments_users)가 commenter_id만 포함하고 있어, deleted_at 필터링과 created_at 정렬을 위해 테이블 행을 직접 읽어야 하기 때문입니다.

이러한 실행 계획은 원하는 결과를 비효율적으로 탐색하고 있다는 증거입니다. 데이터 필터와 정렬에 필요한 컬럼을 포함한 인덱스를 추가하면 더 효율적인 실행 계획을 도출할 수 있을 것으로 판단했습니다.

더 정확한 문제를 진단하기 위해 EXPLAIN ANALYZE를 실행해 실제 수행 시간과 처리량을 확인했습니다.

**EXPLAIN ANALYZE 결과**

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

EXPLAIN ANALYZE 결과에서 들여쓰기가 깊을수록 먼저 실행된 작업입니다. 실행 순서를 보면 다음과 같습니다.

1. commenter_id가 1인 레코드 91,724개를 인덱스로 탐색 (42,952ms)
2. deleted_at IS NULL 필터링으로 87,254개로 축소 (42,962ms)
3. created_at, id 기준으로 정렬 (43,063ms)
4. 최종적으로 11개 레코드 반환

결국 commenter_id가 1이면서 deleted_at이 NULL이고 created_at 기준으로 내림차순 정렬된 테이블에서 11개의 레코드만 필요한 상황입니다. 이 조건들을 모두 포함하는 복합 인덱스를 생성하면 필터링과 정렬 작업에 소요되는 시간을 대폭 줄일 수 있을 것으로 판단했습니다.

### 2.2 복합 인덱스 적용과 JOIN으로 인한 실행 계획 불안정성

EXPLAIN ANALYZE를 분석한 내용을 바탕으로 복합 인덱스를 적용해 나의 댓글 조회 쿼리를 최적화했습니다.

복합 인덱스의 대상 컬럼은 commenter_id, created_at, id로 정했습니다. 필터 조건에 deleted_at도 포함되어 있지만, deleted_at의 NULL 비율이 90% 이상으로 매우 높고, 이를 인덱스에 추가하지 않아도 약 12~15개 행만 추가로 읽으면 되므로 성능 영향이 미미하다고 판단해 제외했습니다.

```sql
CREATE INDEX idx_comments_commenter_created_id 
ON comments (commenter_id, created_at DESC, id DESC);
```

복합 인덱스를 적용한 후 EXPLAIN을 확인했을 때 예상과 다른 결과가 나타났습니다.

**복합 인덱스 적용 후 EXPLAIN 결과**

| id | select_type | table | type | possible_keys | key | key_len | ref | rows | filtered | Extra |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | SIMPLE | m2_0 | ALL | PRIMARY | null | null | null | 109 | 10 | Using where; Using temporary; Using filesort |
| 1 | SIMPLE | m1_0 | ref | PRIMARY, fk_moments_users | fk_moments_users | 8 | moment_database.m2_0.id | 10,989 | 10 | Using where |
| 1 | SIMPLE | c1_0 | ref | PRIMARY, fk_comments_moments, fk_comments_users, idx_comments_commenter_created_id | fk_comments_moments | 8 | moment_database.m1_0.id | 10 | 0.49 | Using where |

EXPLAIN 결과를 보면 옵티마이저가 User 테이블을 먼저 스캔(ALL)하는 것을 확인할 수 있습니다.

원래 의도는 Comment 테이블에서 commenter_id = 1인 레코드를 먼저 필터링해 탐색 범위를 최소화한 후 Moment와 User를 JOIN하는 것이었습니다. 하지만 실제 실행 계획은 이와 달랐습니다.

**복합 인덱스 적용 후 EXPLAIN ANALYZE 결과**

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
                -> Filter: ((c1_0.commenter_id = 1) and (c1_0.deleted_at is null) and (...))  (cost=9.21 rows=0.05) (actual time=0.0297..0.0308 rows=0.09 loops=921594)
                    -> Index lookup on c1_0 using fk_comments_moments (moment_id=m1_0.id)  (cost=9.21 rows=10.2) (actual time=0.0157..0.03 rows=9.5 loops=921594)
```

EXPLAIN ANALYZE 결과를 보면 실제로는 User → Moment → Comment 순서로 JOIN이 진행되고 있습니다. Comment 인덱스 조회가 92만 번(loops=921,594) 반복되면서 여전히 31초가 소요되어 성능이 개선되지 않았습니다.

MySQL 옵티마이저가 잘못된 조인 순서를 선택한 것이 주요 원인으로 파악되어 추가 최적화가 필요한 상황이었습니다.

### 2.3 JOIN으로 인한 실행 계획 불안정성 해결 방법 탐색

옵티마이저가 의도한 조인 순서대로 실행 계획을 수립하도록 여러 해결책을 시도했습니다.

**첫 번째 시도: 복합 인덱스에 deleted_at 추가**

deleted_at을 복합 인덱스에 추가하면 옵티마이저가 더 정확한 통계를 바탕으로 실행 계획을 수립할 수 있을 것으로 기대했습니다. 하지만 deleted_at을 추가해도 여전히 옵티마이저는 실행 계획을 고정하지 못했고 근본적인 해결책이 되지 못했습니다.

**두 번째 시도: Force Index 또는 Straight Join**

Force Index는 옵티마이저가 특정 인덱스를 사용하도록 강제하고, Straight Join은 작성된 테이블 순서대로 조인을 실행하도록 제약을 거는 방법입니다.

이 방법들을 사용하면 옵티마이저가 예상한 방향대로 실행 계획을 고정할 수 있습니다. 하지만 데이터 분포가 변경되면 옵티마이저의 통계 기반 최적화를 무시하게 되어 오히려 비효율적인 실행 계획이 선택될 수 있습니다.

힌트를 사용한 강제 최적화보다는 쿼리 구조 자체를 개선하거나 적절한 인덱스 설계를 통해 옵티마이저가 올바른 판단을 내릴 수 있도록 유도하는 것이 더 나은 접근 방법이라고 판단했습니다.

**세 번째 시도: 쿼리 구조 개선**

인덱스 개선이 아닌 쿼리 구조 개선을 통해 성능 최적화를 시도했습니다. 옵티마이저가 Comments 테이블을 먼저 탐색해 범위를 줄일 수 있도록 서브쿼리 방식을 적용했습니다.

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

FROM절에 서브쿼리를 작성해 Comments 테이블에서 commenter_id를 기준으로 11개 행을 먼저 조회하도록 했습니다. EXPLAIN ANALYZE를 통해 확인한 결과, 정의한 복합 인덱스를 정상적으로 사용하는 것을 확인했습니다.

하지만 서브쿼리를 사용하려면 Native Query를 작성해야 했고, Comment, Moment, User의 모든 컬럼을 조회하는 JOIN 쿼리였기 때문에 JPA 엔티티(Entity)로 직접 매핑할 수 없어 별도의 DTO(Data Transfer Object)를 생성해야 했습니다.

이 방법은 성능 개선 효과가 있지만, 현재 시점에서는 JPQL(Java Persistence Query Language) 기반의 일관된 코드 스타일을 유지하는 것이 프로젝트 전반의 유지보수성 측면에서 더 중요하다고 판단해 보류했습니다.

### 2.4 지연 조인 전략(Deferred JOIN) 도입

여러 시도 끝에 최종적으로 선택한 방법은 지연 조인 전략입니다.

**지연 조인으로 분리된 쿼리**

```java
@Query("""
    SELECT c.id
    FROM Comment c
    WHERE c.commenter = :commenter
    ORDER BY c.createdAt DESC, c.id DESC
""")
List<Long> findFirstPageCommentIdsByCommenter(@Param("commenter") User commenter, Pageable pageable);

@Query("""
    SELECT c
    FROM Comment c
    LEFT JOIN FETCH c.moment m
    LEFT JOIN FETCH m.momenter
    WHERE c.id IN :ids
    ORDER BY c.createdAt DESC, c.id DESC
""")
List<Comment> findCommentsWithDetailsByIds(@Param("ids") List<Long> ids);
```

지연 조인 전략은 쿼리를 두 단계로 나누어 실행합니다.

1. **ID만 조회**: 필요한 Comment의 ID만 먼저 조회합니다.
2. **상세 데이터 조회**: 조회한 ID 목록으로 실제 데이터와 연관 엔티티를 조회합니다.

Comment의 ID만 조회하는 `findFirstPageCommentIdsByCommenter()` 메서드의 성능 최적화를 위해 Comments 테이블에 commenter_id, created_at, id를 포함하는 복합 인덱스를 설정했습니다.

**성능 개선 결과**

지연 조인 전략을 적용한 결과 쿼리 성능이 크게 개선되었습니다. 기존 32초에서 **평균 0.5초 이내**로 응답 시간이 단축되었으며, JPQL 기반의 일관된 코드 스타일도 유지할 수 있었습니다.

---

## 3. 2차 개선 - 아키텍처 리팩터링을 통한 근본 문제 해결

### 3.1 지연 조인 전략의 성공과 새로운 의문

지연 조인 전략 도입으로 MySQL 옵티마이저가 조인 순서를 매번 다르게 선택하던 문제를 해결했고, 쿼리 성능을 안정화하는 데 성공했습니다.

하지만 쿼리 최적화를 진행하면서 한 가지 근본적인 의문이 들었습니다.

**"이 문제는 정말 쿼리만의 문제였을까?"**

현재 Moment 프로젝트 코드 중 일부를 살펴보겠습니다.

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

문제가 되었던 Comment 도메인만 살펴봐도 CommentService는 14개의 서로 다른 서비스와 컴포넌트에 의존하고 있었습니다.

Repository 직접 조회, QueryService 호출, 다른 도메인 Service 의존이 명확한 기준 없이 혼재되어 있었습니다. 이는 프로젝트 초반 명확한 도메인 경계 설정 없이 기능 구현에만 집중했던 결과였습니다.

이러한 구조적 문제가 결국 복잡한 다중 테이블 JOIN으로 이어졌고, 쿼리 성능 불안정성이라는 형태로 나타났습니다.

어떤 쿼리 최적화 기법을 적용하더라도 이 구조적 문제가 해결되지 않으면 근본적인 불안정성은 계속될 것이라고 판단했습니다. 이러한 인식을 바탕으로 단순히 쿼리를 최적화하는 것을 넘어 프로젝트 전체 아키텍처를 재설계하기로 결정했습니다.

### 3.2 Facade 패턴 도입 및 3계층 아키텍처 구축

#### 리팩터링 이전의 문제점

리팩터링 이전 구조는 복잡한 다중 테이블 JOIN 외에도 다음과 같은 문제를 가지고 있었습니다.

- **책임의 불명확성**: 하나의 서비스가 여러 도메인의 Repository와 Service에 직접 의존
- **높은 결합도**: 도메인 간 경계가 모호해 변경 시 영향 범위가 불명확
- **복잡한 매핑 로직**: DTO 응답 생성을 위한 매핑 로직이 여러 곳에 분산
- **테스트 어려움**: 과도한 의존성으로 인해 단위 테스트 작성이 어려움

#### 3계층 구조 설계

이러한 문제를 해결하기 위해 다음과 같은 3계층 구조를 설계했습니다.

```text
┌─────────────────────────────────────┐
│   Facade Layer (컨트롤러와 통신)     │
│   - MomentFacadeService             │
│   - 여러 Application Service 조합   │
│   - DTO 반환                        │
└─────────────────────────────────────┘
              ↓
┌─────────────────────────────────────┐
│   Application Layer (도메인 조합)    │
│   - MomentApplicationService        │
│   - 관련 도메인 서비스 조합          │
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

Facade 계층은 컨트롤러와 직접 통신하는 최상위 계층으로, 여러 도메인의 ApplicationService를 조합해 API 응답을 생성합니다.

Facade 계층에서는 Repository에 직접 접근하지 않도록 규칙을 정했습니다. 대신 각 도메인의 ApplicationService를 통해 독립적으로 데이터를 조회하고, 이를 메모리에서 조합합니다.

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

ApplicationService 계층에서는 Entity가 아닌 DTO를 반환해 상위 계층에 도메인이 노출되는 것을 방지합니다. 이를 통해 도메인 모델의 변경이 상위 계층에 영향을 주지 않도록 하고, 비즈니스 로직이 도메인 계층에서만 실행되도록 보장합니다.

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

Domain 계층은 단일 Repository와만 통신하며 해당 도메인의 CRUD 및 비즈니스 로직 처리를 담당합니다.

#### 리팩터링 성과

리팩터링을 진행한 후 각 도메인은 관련된 도메인에 대한 책임만 가지도록 완벽히 분리되었으며, 과도하게 집중되어 있던 의존성이 도메인별 계층의 책임에 맞게 고루 분포되었습니다.

**의존성 구조 개선**

```
리팩터링 전:
CommentService - 14개 의존성
├─ UserQueryService
├─ MomentQueryService
├─ CommentRepository
├─ EchoQueryService
├─ RewardService
└─ ... (총 14개)

리팩터링 후:
MyCommentPageFacadeService - 3개 의존성
├─ CommentApplicationService - 4개 의존성
│   ├─ UserService (단일 Repository)
│   ├─ CommentService (단일 Repository)
│   ├─ CommentImageService (단일 Repository)
│   └─ EchoService (단일 Repository)
├─ MomentApplicationService
└─ NotificationApplicationService
```

하나의 거대한 서비스를 책임에 따라 분리해 각 계층의 복잡도가 크게 감소했으며, 계층의 역할과 의존성이 명확해져 변경 영향 범위를 예측할 수 있게 되었습니다. 또한 단일 책임을 가진 서비스로 분리되어 단위 테스트 작성이 용이해졌습니다.

### 3.3 지연 조인 전환 및 쿼리 분리

아키텍처 리팩터링과 함께 쿼리 구조도 개선되었습니다.

#### 지연 조인 전략 적용

기존의 복잡한 JOIN 쿼리를 2단계로 분리했습니다.

**1단계: ID만 조회** (CommentService에서 실행)

```java
@Query("""
    SELECT c.id
    FROM Comment c
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
    FROM Comment c
    WHERE c.id IN :ids
    ORDER BY c.createdAt DESC, c.id DESC
""")
List<Comment> findCommentsByIds(@Param("ids") List<Long> ids);
```

- 1단계에서 찾은 ID 목록으로 정확히 필요한 행만 조회
- 불필요한 JOIN 제거

#### 간접 참조를 통한 도메인 경계 명확화

각 도메인 영역별로 간접 참조를 사용해 예상하지 못한 N+1 쿼리 발생을 방지하고 도메인 간 경계를 명확하게 나누었습니다.

**리팩터링 전: 직접 참조**

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

**리팩터링 후: 간접 참조**

```java
public class Comment {
    @Column(name = "moment_id")
    private Long momentId;  // ID만 보관 (간접 참조)

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "commenter_id", insertable = false, updatable = false)
    private User commenter;  // 조회용으로만 유지
}
```

간접 참조를 사용하면 "언제, 무엇을, 얼마나 조회할지"를 개발자가 명확하게 코드로 작성할 수 있습니다.

**직접 참조 사용 시**

```java
Comment comment = commentRepository.findById(1L);
String momentContent = comment.getMoment().getContent();
// ❌ moment를 조회할지 안 할지 코드만 봐서는 불명확
// ❌ 언제 쿼리가 나가는지 예측 어려움 (Lazy Loading)
// ❌ N+1 문제 발생 가능
```

**간접 참조 사용 시**

```java
Comment comment = commentRepository.findById(1L);
// moment가 필요하면 명시적으로 조회
Moment moment = momentRepository.findById(comment.getMomentId());
String momentContent = moment.getContent();
// ✅ moment를 조회하는 코드가 명확하게 보임
// ✅ 언제 쿼리가 나가는지 정확히 알 수 있음
// ✅ 필요 없으면 아예 조회하지 않음
```

**실전 예시**

```java
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
```

간접 참조를 통해 연관된 엔티티를 조회하는 시점과 범위를 명시적으로 코드에 표현할 수 있어, 성능 문제를 사전에 파악하고 제어할 수 있습니다.

### 3.4 최종 성능 개선 결과

**쿼리 성능 개선**

- 기존: 평균 32초 (최대 90초)
- 개선 후: 평균 0.5초 이내 (약 64배 개선)

**아키텍처 개선**

- 서비스 의존성: 14개 → 계층별 3~4개로 분산
- 도메인 경계 명확화: Repository 직접 접근 규칙 정립
- 테스트 용이성: 단일 책임 원칙 적용으로 단위 테스트 작성 가능

---

## 4. 회고 및 배운 점

### 4.1 쿼리 최적화를 넘어 아키텍처 개선으로

이번 프로젝트를 통해 단순히 쿼리 성능만 개선하는 것이 아니라 근본적인 구조적 문제를 함께 해결해야 한다는 것을 배웠습니다.

처음에는 EXPLAIN ANALYZE를 통해 쿼리 실행 계획을 분석하고 인덱스를 최적화하는 것에 집중했습니다. 하지만 복합 인덱스를 적용해도 옵티마이저가 의도한 실행 계획을 선택하지 않는 문제를 겪으면서, 쿼리 자체보다 애플리케이션 구조가 복잡한 JOIN을 유발하고 있다는 것을 깨달았습니다.

결국 지연 조인 전략을 도입하고 3계층 아키텍처로 리팩터링하면서 쿼리 성능과 코드 품질을 동시에 개선할 수 있었습니다.

### 4.2 명확한 계층 분리의 중요성

Facade, Application, Domain 계층으로 명확하게 분리하면서 다음과 같은 이점을 얻을 수 있었습니다.

- **책임의 명확화**: 각 계층이 자신의 역할에만 집중
- **의존성 관리**: Repository 직접 접근 규칙으로 예측 가능한 데이터 흐름
- **테스트 용이성**: 단일 책임 원칙 적용으로 단위 테스트 작성 가능
- **변경 영향 최소화**: 도메인 경계가 명확해 변경 범위 예측 가능

### 4.3 간접 참조의 장점

JPA의 연관관계(@ManyToOne, @OneToMany) 대신 간접 참조(ID 값)를 사용하면서 다음과 같은 이점을 발견했습니다.

- **명시적 쿼리**: 언제, 무엇을, 얼마나 조회할지 코드로 명확하게 표현
- **N+1 문제 예방**: Lazy Loading으로 인한 예상치 못한 쿼리 발생 방지
- **도메인 독립성**: 각 도메인이 다른 도메인에 직접 의존하지 않음

### 4.4 성능 최적화의 순서

이번 프로젝트를 통해 성능 최적화를 다음과 같은 순서로 접근하는 것이 효과적이라는 것을 배웠습니다.

1. **문제 정의**: 실제 성능 저하가 발생하는 지점 파악
2. **데이터 수집**: EXPLAIN ANALYZE로 정확한 원인 분석
3. **인덱스 최적화**: 적절한 복합 인덱스 설계
4. **쿼리 구조 개선**: 지연 조인, 서브쿼리 등 전략 선택
5. **아키텍처 개선**: 근본적인 구조 문제 해결

### 4.5 마치며

Slow Query 개선은 단순히 쿼리만의 문제가 아니었습니다. 명확한 도메인 경계 설정 없이 기능 구현에만 집중했던 초기 설계가 복잡한 JOIN을 유발했고, 이것이 성능 문제로 이어졌습니다.

이번 경험을 통해 처음부터 명확한 아키텍처 설계의 중요성을 깨달았으며, 단순히 동작하는 코드를 넘어 확장 가능하고 유지보수하기 쉬운 구조를 만드는 것이 얼마나 중요한지 배울 수 있었습니다.
