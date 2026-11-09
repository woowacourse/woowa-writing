# PostgreSQL에서 MySQL로 전환한 이유

## Abstract

> 우리는 **KISS(Keep It Simple, Stupid) + YAGNI(You Aren't Gonna Need It)** 원칙에 따라, 현재 요구사항을 충족하는 가장 단순한 선택을 우선하기로 했다. 그 결과, **기존에 사용하던 PostgreSQL 대신 MySQL로 Production DB를 전환하기로 최종 결정했다.**

|   |   |   |   |
|---|---|---|---|
|비교 기준|PostgreSQL|MySQL|Rilog. 판단|
|**격리 수준**|기본 `READ COMMITTED`|InnoDB 기본 `REPEATABLE READ`|둘 다 변경 가능하며 Rilog.는 높은 격리 수준이 핵심 요구사항이 아님 → **선택에 영향 적음**|
|**READ**|B-tree +  <br>**Partial Index**|B-tree + **Composite Index**|**PostgresSQL이 기능적으로 우위**지만, `PUBLISHED` 비율이 높은 Rilog.에서는 Partial Index의 실질적 이점은 작음.|
|**UPDATE**|MVCC → dead tuple → `VACUUM`, `HOT` 최적화 가능|Undo Log → `Purge`|게시글당 평균 약 6회의 수정이 관찰됐으며 최신 데이터 조회 중심 → **MySQL이 조금 더 적합하다고 판단**.|
|`content` **압축**|`TOAST`: 큰 값을  <br>선택적으로 압축 + out-of-line 저장|`DYNAMIC`: 큰 값 off-page / `COMPRESSED`: 압축 가능|`content`만 큰 Rilog. 구조에는 PostgreSQL이 조금 더 자연스러울 수 있지만, **MySQL도 충분히 대응 가능하며 오히려 대부분의 Rilog. 에서 자주 나오는 상황에서는 우위를 보임.**|
|**검색**|`jsonb`, GIN, `tsvector` 등  <br>기능적으로 풍부|`JSON`, `FULLTEXT` 지원|`content`를 검색 대상에서 제외하기로 한 현재 정책에서는 **MySQL로 충분**. 검색 엔진 고도화 필요 시, 향후 ElasticSearch 도입 고려.|

---

## 서론

1차 스프린트를 마친 뒤 회고를 진행하면서, 기술적인 결정부터 사소한 의사 결정까지 전반적으로 문서화가 부족했다는 의견이 팀 내에서 나왔다. 이에 기술 결정, 일반적인 의사 결정, 비즈니스 정책 결정까지 포함해 ADR을 다시 정리하는 시간을 가졌다.

![image.png](https://techcourse-project-2026.s3.ap-northeast-2.amazonaws.com/rilog/images/originals/c434e527-f5bc-4570-b7e0-06347ff9f05c.png)

프로젝트 초기 설정 단계에서 작성한 Production DB에 대한 ADR이다.

프로젝트 초기에는 빠르게 개발 환경을 구성하고 기능 구현을 시작하는 것이 우선이라고 생각했다. 당시에는 ADR을 남기는 것 자체의 중요성이나, 하나의 기술 선택이 이후 개발과 운영에 얼마나 오랫동안 영향을 미치는지 충분히 체감하지 못했다. (갈 길이 멀다고 느꼈기 때문이랄까.. 바쁘다. 바빠)

또한 다른 블로그 플랫폼에서 PostgreSQL을 많이 사용한다는 점을 이미 접한 상태였기 때문에, “블로그 플랫폼이라면 PostgreSQL이 무난하겠지”라는 생각으로 큰 고민 없이 선택했다. 이후 작성한 ADR 역시 여러 선택지를 충분히 비교한 뒤 결론을 내렸다기보다, 이미 내려둔 선택을 뒷받침하는 근거를 정리하는 데 가까웠던 것 같다.

1차 스프린트를 마치고 회고하는 과정에서 기술적인 결정부터 비즈니스 정책에 이르기까지 여러 의사 결정의 근거가 제대로 남아 있지 않다는 점을 발견했다. 시간이 지나자 “왜 이렇게 결정했지?”라는 질문에 나 스스로가 명확하게 답하기 어려운 순간도 생겼고, 이때부터 기술 결정의 결과만큼이나 그 과정과 근거를 기록하는 것이 중요하다는 점을 체감하게 되었다. 지금 다시 당시 ADR을 읽어보면 근거가 충분하지 않고, 제3자의 입장에서 보았을 때도 PostgreSQL을 선택해야 하는 이유가 명확하게 전달되지 않는다. 그래서 기존 결정을 그대로 답습하기보다, 현재 Rilog. 의 요구사항을 기준으로 처음부터 다시 검토해보기로 했다.

이 경험 이후로 나는 **“다들 많이 쓰니까”, “익숙하니까”와 같은 이유만으로 기술을 선택하는 것을 경계하게 되었다.** 기술은 한 번 선택하면 이후의 개발 방식과 운영 비용, 확장성에 지속적으로 영향을 미친다. 따라서 어떤 기술을 선택했는지보다, **왜 그 기술을 선택했는지 스스로 설명할 수 있는가가 더 중요하다고 생각한다.** **설령 그 선택이 주류가 아닐지라도 말이다.**

그래서 이번에는 PostgreSQL을 이미 사용하고 있다는 사실에 기대지 않고, Rilog.의 요구사항에 정말 적합한 선택인지 처음부터 다시 따져보기로 했다.

## 격리 수준

> postgreSQL의 기본 격리 수준은 `READ COMMITTED`, MySQL InnoDB는 `REPEATABLE READ`이다.

Rilog.는 조회 중심의 짧은 트랜잭션이 대부분이며, 하나의 트랜잭션 내에서 동일한 스냅샷을 강하게 유지해야 하는 요구사항이 거의 없다. 따라서 일반적인 작업에는 `READ COMMITTED` 수준으로 충분하다.

동시에 높은 정합성이 필요한 일부 작업은 격리 수준을 서비스 전체에 높게 적용하기보다 `Unique Constraint`, 명시적 Lock, 원자적 Update 등을 통해 개별적으로 보장하는 것이 적절하다고 판단했다.
- 조회수나 각종 Count 값은 일시적인 정합성 오차가 발생하더라도 서비스의 핵심 데이터 정합성에 큰 문제를 일으키지는 않는다.
- 반면 `slug` 중복과 같이 반드시 보장되어야 하는 정합성은 `Unique Constraint`로 보장하고, 좋아요 수나 조회수와 같은 카운트성 데이터는 원자적 `UPDATE`를 사용한다. 그 외 동시성 제어가 필요한 경우에는 `Optimistic Lock`이나 `SELECT ... FOR UPDATE`와 같은 명시적 Lock을 적용해 개별적으로 해결한다.

따라서, 격리 수준은 PostgreSQL의 기본 `READ COMMITTED` 가 Rilog. 에 잘 맞는다. 하지만, 두 DB 모두 트랜잭션 또는 세션 단위로 격리 수준을 변경할 수 있으므로, **PostgreSQL 선택의 핵심 근거라기보다는 운영상 자연스러운 기본값이라는 정도로 평가할 수 있다.**

> 결론: Rilog. 는 높은 격리 수준 자체가 중요한 서비스가 아니므로 이는 DB 선택에 거의 영향을 주지 않는다.

---

## 조회(Read)

### PostgreSQL: Partial Index

**Partial Index** 란?

- PostgreSQL은 `WHERE` 조건을 만족하는 행만 인덱스에 포함하는 **Partial Index**를 지원한다.
- 예를 들어 Rilog에서 실제 사용자 조회는 대부분 `PUBLISHED` 게시글만 대상으로 한다면:
```sql
CREATE INDEX idx_post_published
ON post (blog_id, published_at DESC)
WHERE status = 'PUBLISHED';
```

- `DRAFT`, `DELETED`, `PRIVATE` 등의 데이터는 인덱스에서 제외할 수 있어 **인덱스 크기를 줄이고 조회 효율을 높일 수 있다.**
- PostgreSQL은 B-tree뿐 아니라 GIN, GiST, BRIN 등의 인덱스와 Partial Index를 공식 지원한다.
    - [Index 관련 PostgreSQL 공식 문서](https://www.postgresql.org/docs/18/sql-createindex.html)
    - 이러한 부분도 확장성을 생각했을때, 조회가 중요한 Rilog. 에는 긍정적인 요소라고 볼 수 있다.
- 인덱스 크기가 작아지면 메모리에 올라갈 확률도 높아지고, 인덱스 탐색 및 유지 비용도 줄어들 수 있다.


```text
전체 Post 100만 건
	- PUBLISHED 60만
	- DRAFT     20만
	- PRIVATE   10만

일반 Index → 100만 건 모두 인덱싱
Partial Index → 필요한 60만 건만 인덱싱
```

### MySQL: Composite Index

MySQL은 PostgreSQL과 동일한 형태의 **native Partial Index는 지원하지 않는다.**

- 대신 보통, `INDEX(status, user_id, updated_at)` 처럼 **복합 인덱스**를 사용해서 해결한다.
- Patial Index와의 차이는 다음과 같다.


```sql
복합 인덱스: (blog_id, status, created_at)
- PUBLISHED, DRAFT, PRIVATE 모든 상태의 글 포함
	→ 즉, 모든 행이 인덱스에 존재


Partial Index
(blog_id, created_at)
WHERE status = 'PUBLISHED'

PUBLISHED 글만 포함 (DRAFT, PRIVATE X)
	→ 즉, 일부 행만 인덱스에 존재
```

여기까지만 보면, PostgreSQL 이 제공하는 Patial Index가 굉장히 매력적으로 다가온다. 하지만, 조금 더 Rilog. 서비스의 특성을 고려해 생각해 볼 필요가 있다. **결론부터 말하자면, Partial Index라고 무조건 엄청 빨라지는 것은 아니다.**
- 게시글 조회는 대부분 `PUBLISHED` 상태의 글을 대상으로 이루어질 것으로 예상된다. 그러므로, `PUBLISHED` 게시글을 대상으로 Index 를 거는 것은 자명하다.
- But, `PUBLISHED`, `DRAFT`, `PRIVATE` 비율을 생각해 봐야 한다.
    - e.g. Rilog 게시글의 95%가 `PUBLISHED` 라고 가정하면, Partial Index를 만들어도 전체 데이터의 95%를 인덱싱하는 것이다. -> 큰 이득이 없을 수 있다.
    - PostgreSQL 공식 문서에 의하면, partial index를 전체 테이블 중 유용한 subset만 인덱싱하는 용도로 설명한다. -> **소량의 일부 데이터만을 자주 조회할 때** **Partial Index가 빛난다.**


### Partial Index와 Composite Index 비교 실험

전체 게시글을 1,000,000건으로 가정하고 `PUBLISHED`, `DRAFT`, `PRIVATE`의 데이터 분포에 따른 인덱스 효율을 비교해보았다.

|   |   |   |   |   |
|---|---|---|---|---|
|데이터 분포 (`PUBLISHED:DRAFT:PRIVATE`)|MySQL 복합 인덱스 대상|PostgreSQL Partial Index 대상|MySQL 10회 평균 조회 시간|PostgreSQL 10회 평균 조회 시간|
|`90:7:3`|1,000,000|900,000|0.45ms|0.42ms|
|`50:30:20`|1,000,000|500,000|0.45ms|0.35ms|
|`10:60:30`|1,000,000|100,000|0.46ms|0.25ms|

`PUBLISHED` 게시글이 전체의 90%를 차지하는 경우 PostgreSQL의 Partial Index를 사용하더라도 전체 데이터의 10%만 인덱싱 대상에서 제외된다. 따라서 일반적인 B-tree 기반 복합 인덱스와 비교했을 때 인덱스 크기와 조회 성능 측면에서 얻을 수 있는 이점은 제한적이다.

반면 `PUBLISHED`의 비율이 50% 이하로 감소하면 Partial Index를 통해 관리해야 하는 인덱스의 크기가 크게 줄어들기 시작하며, 10~30% 수준에서는 인덱스 크기와 캐시 효율, 인덱스 갱신 비용 측면에서 PostgreSQL의 Partial Index가 보다 명확한 장점을 가질 수 있다.

그러나 Rilog.와 같은 블로그 플랫폼에서는 장기적으로 저장되는 게시글의 대부분이 `PUBLISHED` 상태일 것으로 예상된다. 또한 주요 조회 역시 특정 블로그의 `PUBLISHED` 게시글을 최신순으로 일부 조회하는 형태이므로, `(blog_id, status, published_at)`과 같이 조회 패턴에 맞게 구성된 **MySQL의 B-tree 기반 복합 인덱스만으로도 충분한 성능을 확보할 수 있다고 판단했다.**

> 결론: Partial Index는 특정 데이터의 비율이 전체 데이터 중 작은 부분을 차지할 때 PostgreSQL이 가지는 분명한 장점이지만, 예상되는 Rilog. 의 데이터 분포(`90:7:3`)에서는 실질적인 차이가 크지 않다.

---

## 수정(Update)

### PostgreSQL: VACUUM

PostgreSQL은 MVCC 때문에 **UPDATE 시 새로운 row version을 만든다.**

```text
Post v1 (기존)
   ↓ UPDATE

Post v1  ← 이전 버전
Post v2  ← 새로운 버전
```

이전 버전이 더 이상 필요 없어지면 dead tuple이 되고 `VACUUM`이 공간을 정리해 재사용할 수 있도록 처리한다.
- PostgreSQL 공식 문서도 VACUUM의 주요 목적 중 하나가 UPDATE/DELETE된 row가 점유하던 공간의 회수/재사용이라고 설명한다.
- [VACCUM 관련 Postgres 공식 문서](https://www.postgresql.org/docs/18/routine-vacuuming.html)


따라서, **UPDATE가 지속적으로 발생하고 VACUUM이 이를 충분히 따라가지 못하는 경우 dead tuple이 누적될 수 있다.** Dead tuple이 많이 쌓이면 테이블과 인덱스의 물리적 크기가 증가하는 bloat가 발생할 수 있으며, 이는 더 많은 page 접근과 낮아진 cache 효율 등으로 이어져 **조회(READ) 성능에도 부정적인 영향을 줄 수 있다.**

다만, PostgreSQL에도 **HOT(Heap-Only Tuple)**라는 최적화가 있어서 인덱스 컬럼을 변경하지 않는 `UPDATE` 등 조건이 맞으면 인덱스 갱신 비용을 줄일 수 있다.

예를 들어 다음과 같이 `id`, `slug`에만 인덱스가 존재한다고 가정한다.
```sql
id              INDEX
slug            INDEX
title
content
```

이때 인덱스가 걸려 있지 않은 `title`만 수정하고 HOT 조건을 만족한다면,
```sql
UPDATE post
SET title = '새 제목'
WHERE id = 1;
```

PostgreSQL은 새로운 row version을 생성하더라도 기존 인덱스 엔트리를 그대로 활용할 수 있다.

반면 인덱스가 걸린 칼럼을 수정하는 경우에는, 기존 인덱스 엔트리를 그대로 사용할 수 없기 때문에 새로운 인덱스 엔트리를 생성해야 한다. 그만큼 인덱스 유지 비용이 추가로 발생한다.

￼

#### 더 알아봐야 할 부분.

개인적으로 Rilog.에서는 많은 `UPDATE`가 HOT 조건을 만족할 가능성이 높다고 본다. 게시글 조회를 위해 `status`, `blogId`, `publishedAt` 등에 인덱스가 구성될 가능성이 높지만, 실제 수정은 `title`, `content`, `thumbnailUrl`처럼 인덱스에 포함되지 않는 컬럼에서 자주 발생할 것으로 예상되기 때문이다. 다만 HOT 적용 여부는 인덱스 컬럼 변경 여부뿐 아니라 동일 Heap Page에 새로운 tuple을 저장할 수 있는 여유 공간이 있는지에도 영향을 받기 때문에 지금 판단하긴 좀 복잡한 것 같다.

- PostgreSQL은 HOT UPDATE 횟수를 통계로 제공하므로, 실제 HOT 적용 비율을 확인해보는 것도 의미 있을 것 같다.
    - [HOT 관련 PostgreSQL 공식 문서](https://www.postgresql.org/docs/18/storage-hot.html?)
### MySQL: Undo Log

MySQL도 MVCC를 사용하지만, 이전 row version을 PostgreSQL처럼 테이블 heap에 계속 두는 방식과는 다르게 **Undo Log를 이용해서 이전 버전을 복원할 수 있게 관리**한다.
```text
현재 row
   ↓
Undo Log
   ↓
과거 version
```

이전 값을 undo 영역에 보관했다가 더 이상 필요 없으면 purge한다. DELETE 역시 바로 물리적으로 삭제하는 게 아니라 delete mark 후 purge하는 구조다.
- [Undo 관련 MySQL 공식 문서](https://dev.mysql.com/doc/refman/8.4/en/innodb-multi-versioning.html)

Undo log는 트랜잭션 롤백뿐 아니라, 오래된 Read View를 가진 트랜잭션에게 과거 버전을 제공하기 위해서도 사용된다. 오래된 트랜잭션이 존재하거나 과거 버전을 조회해야 하는 경우에는 undo chain을 따라 이전 버전을 재구성해야 하므로 추가적인 비용이 발생할 수 있다.
- 하지만 Rilog.의 주요 트랜잭션은 비교적 짧고, 일반적인 게시글 조회는 최신 상태의 데이터를 조회하는 경우가 대부분이다. 장시간 동일한 스냅샷을 유지하면서 과거 버전을 조회해야 하는 요구사항은 많지 않을 것으로 예상된다.

정리해보면, 두 과정 모두 이전 버전을 청소 과정은 필요하다.
```text
# PostgreSQL
UPDATE / DELETE
  → old tuple
	→ VACUUM

# MySQL
UPDATE / DELETE
	→ undo/delete mark
	→ purge
```

### 사용자의 Update 패턴 분석

초기에는 블로그 게시글은 한 번 발행된 이후 수정이 자주 발생하지 않을 것이라고 예상했다. 그러나 **PostHog를 통해 실제 사용 패턴을 관찰한 결과, 총 16개의 게시글에서 게시글 하나당 평균 약 6회의 수정이 발생했다.**

- 수정이 이뤄지는 형태는 다음과 같았다.
    - 단순한 오탈자 수정, 우선 초안에 가까운 상태로 글을 발행한 뒤 내용을 지속적으로 보완하는 경우도 있었다. (아직 private/public 발행 기능이 구현되지 않아 그럴 수도 있다.) 그리고, 제목이나 썸네일을 여러 번 변경하는 사용자도 적지 않았다.

이를 통해 실제 블로그 서비스에서는 게시글이 단순히 “한 번 작성되고 계속 조회되는 데이터”라기보다, 발행 이후에도 "생각보다 자주 수정될 수 있는 데이터라는 점"을 확인할 수 있었다. 따라서 Rilog.에서는 MySQL의 undo chain을 따라 과거 버전을 복원해야 하는 비용보다, 빈번한 게시글 수정으로 인해 PostgreSQL에서 dead tuple과 bloat가 발생하고 이것이 조회 효율에 영향을 줄 가능성을 상대적으로 더 중요하게 고려할 수 있다.

> 결론: 서비스의 핵심 트래픽은 게시글 조회에 집중될 것으로 예상된다. 이러한 데이터 접근 패턴만을 놓고 본다면, 현재 record를 중심으로 조회하고 이전 버전을 undo 영역에서 관리하는 **MySQL InnoDB의 MVCC 구조가 Rilog.의 읽기 중심 워크로드에 적합하다고 판단했다.**

> 다만, 이 차이만으로 MySQL이 항상 PostgreSQL보다 빠르다고 단정할 수는 없다. PostgreSQL의 HOT UPDATE 적용 여부, autovacuum 설정, 실제 dead tuple 비율, 테이블 크기와 cache hit ratio 등에 따라 결과가 달라질 수 있으므로, 이는 데이터베이스 선택의 단독 근거라기보다는 Rilog.의 실제 UPDATE 패턴을 기반으로 고려한 하나의 판단 요소로 보았다.

---

## content 압축

Rilog.의 `Post` 테이블은 대부분의 컬럼은 크기가 작지만, 게시글 본문인 `content`**만 상대적으로 매우 큰 구조**를 가진다. 실제 게시글 본문은 수십 KB 이상까지 커질 수 있다.
- 이처럼 하나의 컬럼만 압도적으로 큰 데이터를 PostgreSQL과 MySQL은 어떻게 저장하는지 비교해보았다.
### PostgreSQL: TOAST

PostgreSQL은 일반적으로 8KB 크기의 page를 사용하며, 하나의 tuple이 여러 page에 걸쳐 직접 저장될 수 없다. 따라서 `TEXT`, `JSONB`, `BYTEA`처럼 큰 값을 가질 수 있는 가변 길이 데이터는 **TOAST(The Oversized-Attribute Storage Technique)** 라는 메커니즘을 통해 처리한다.

PostgreSQL은 저장하려는 row의 크기가 기본적으로 **약 2KB를 넘어가면 TOAST 처리를 자동으로 시도**한다. 이때 큰 값을 압축하거나, 그래도 크다면 별도의 TOAST table로 이동시켜 메인 row의 크기를 줄인다.
- 즉 50KB짜리 JSONB 본문이 있다고 해서 개발자가 직접 `content 압축 -> db 저장 -> 조회 -> 압축 해제` 같은 작업을 할 필요가 없다.
- [TOAST 관련 PostgreSQL 공식 문서](https://www.postgresql.org/docs/18/runtime-config-client.html)

```text
post heap

id = 1
title = "MVCC 정리"
content = [TOAST pointer]
             ↓
       TOAST table
       ┌─────────────────┐
       │ content chunk 1 │
       │ content chunk 2 │
       │ content chunk 3 │
       │ ...             │
       └─────────────────┘
```

TOAST 대상으로 분리된 데이터는 압축될 수 있으며, 여러 chunk로 나뉘어 별도의 TOAST table에 저장된다.

Rilog.의 게시글 목록이나 피드에서는 일반적으로 본문 전체가 필요하지 않다. `content`가 별도의 TOAST 영역으로 빠져 있다면 메인 `post` row에는 비교적 작은 메타데이터만 남는다. 하나의 page에 더 많은 row를 저장할 수 있기 때문에 목록 조회에서는 불필요하게 거대한 본문을 읽지 않아도 된다. 따라서 I/O를 줄이고 buffer cache를 효율적으로 활용할 수 있다는 장점이 있다.

> 즉, 하나의 큰 `content`와 여러 개의 작은 메타데이터 컬럼으로 구성된 Rilog.의 `Post` 구조는 TOAST의 저장 방식과 잘 맞는다고 볼 수 있다.

그렇다면 MySQL은?

### MySQL InnoDB: DYNAMIC Row Format

MySQL InnoDB도 큰 데이터를 무조건 하나의 row 안에 저장하지 않는다. 현재 InnoDB의 기본 Row Format인 `DYNAMIC`에서는 row가 너무 커지면 긴 `VARCHAR`, `TEXT`, `BLOB` 등의 값을 **Overflow Page로 분리할 수 있다.**
- `DYNAMIC`에서는 긴 가변 길이 컬럼을 완전히 off-page로 저장하고, clustered index record에는 20-byte pointer만 남길 수 있다.

```text
Clustered Index Record

id
slug
title
thumbnail
content = [20-byte pointer]
               ↓
        Overflow Page
        ┌─────────────┐
        │ content ... │
        └─────────────┘
               ↓
        Overflow Page
```

즉, PostgreSQL TOAST처럼 MySQL 역시 큰 값 `content`를 메인 row 밖으로 분리할 수 있다. 여기까지 들엇을 때 “본문만 큰 테이블에서 본문을 별도로 저장할 수 있다”는 특성 자체는 PostgreSQL만의 장점이라고 보기 어렵다고 생각될 수 있다. 그렇다면, TOAST와 DYNAMIC 이 둘의 차이에 대한 의문이 들 것이다.

### TOAST와 DYNAMIC의 차이는 무엇인가?

두 방식의 목적은 비슷하지만 동작 방식에는 차이가 있다. 표로 정리해보면 다음과 같다.

|   |   |   |
|---|---|---|
||PostgreSQL TOAST|MySQL InnoDB DYNAMIC|
|큰 데이터 분리|O|O|
|메인 row 밖에 저장|TOAST table|Overflow Page|
|메인 row|TOAST pointer|20-byte pointer|
|큰 값 분할|여러 TOAST chunk|Overflow Page chain|
|자동 압축|가능|DYNAMIC 자체는 압축 기능이 아님|
|주요 목적|큰 Attribute 관리|B-tree page에 적절한 row 크기 유지|

가장 눈에 띄는 차이는 **압축**이다.

PostgreSQL TOAST는 큰 값을 단순히 외부로 분리하는 것뿐 아니라 압축도 시도할 수 있다.
- `50KB content -> compression -> 25KB content! -> TOAST table`

반면 MySQL의 `DYNAMIC`은 기본적으로 **큰 값을 Overflow Page로 옮기는 저장 방식**이지, 그 자체가 데이터를 압축하는 기능은 아니다. 그렇다면, MySQL 은 압축을 지원하지 않는가? 그렇지 않다.

### MySQL: COMPRESSED

MySQL에서도 압축을 사용할 수 있다. 대표적인 방법 중 하나가 `ROW_FORMAT=COMPRESSED`이다.
```sql
CREATE TABLE post (
    id BIGINT PRIMARY KEY,
    title VARCHAR(512),
    content JSON
)
ROW_FORMAT=COMPRESSED;
```

`COMPRESSED`는 `DYNAMIC`이 제공하는 긴 가변 길이 컬럼의 off-page 저장 특성을 가지면서, **테이블과 인덱스 데이터 자체에 압축을 추가**한다. 그렇다면, 압축이 추가되니깐 Toast랑 이젠 같은가? 하는 생각이 드는데 압축 단위가 다르다.
- `TOAST = 큰 컬럼을 선택적으로 압축하고 분리하는 메커니즘`
    - 그 큰 칼럼(e.g. content) **만을** 압축한다.
- `DYNAMIC + COMPRESSED = 큰 컬럼은 분리하면서, 저장 page까지 압축하는 메커니즘`
    - 테이블 전체를 압축해버린다. -> **off-page 뿐만 아니라, 메인 page 까지**

|   |   |   |
|---|---|---|
||PostgreSQL TOAST|MySQL COMPRESSED|
|큰 `content`  <br>밖으로 분리|O|O|
|큰 `content` 압축|O|O 가능(but, 특정 값만 압축은 X)|
|메인 page도 압축?|X|O|
|content 가  <br>필요없는 조회|content decompress 불필요|content는 안 읽지만 메인 page decompress 필요|
|압축 중심|**큰 값(attribute)**|**page/storage**|

압축의 장점은 명확하다.
- 저장 공간 ↓
- Disk I/O ↓
- Buffer Pool 효율 ↑

하지만, 압축된 파일 조회 시, CPU에서 decompress 과정을 거쳐야 한다는 trade-off가 있다. 따라서, 압축은 공짜 성능 향상은 아니다. 압축하려는 대상이 얼마나 많이 조회될 지도 생각해봐야 한다는 의미이다.
```text
Disk I/O 감소
저장 공간 감소

        ↕

CPU 사용량 증가
compression / decompression 비용 증가
```

**I/O가 병목이라면 압축이 오히려 전체 성능을 높일 수 있지만, CPU가 병목인 환경에서는 반대 결과가 나올 수도 있다.**
#### 더 알아봐야 할 부분.

Rilog.는 조회 중심의 블로그 플랫폼이며, 게시글 `content`의 크기가 상대적으로 크다. 따라서 초기에는 **압축 및 압축 해제에 사용되는 CPU 비용보다, 큰 본문 데이터를 Storage에서 읽어오는 I/O와 제한된 Buffer Cache를 얼마나 효율적으로 사용하는지가 더 중요한 병목 요소가 될 가능성이 높다고 판단했다.** 특히 다양한 게시글이 지속적으로 조회되어 전체 게시글 데이터를 메모리에 유지하기 어려워질수록 Cache Miss가 증가하고, Storage I/O의 영향이 커질 수 있다.
- 개인적으로 현재 상황으로서는 PostgreSQL TOAST의 압축으로 인해 추가적인 CPU 비용이 발생하더라도, 실제로 읽어야 하는 데이터의 크기와 저장 공간을 줄일 수 있다는 점이 오히려 전체 성능에 긍정적으로 작용할 수 있다고 생각한다.


반면 특정 인기 게시글이 반복적으로 조회되어 대부분의 데이터가 Buffer Cache에 존재하는 상황에서는 Storage I/O 비용이 줄어들기 때문에, 반복적인 압축 해제에 사용되는 CPU 비용이 상대적으로 더 중요해질 수 있다. 이 경우에는 압축을 수행하지 않는 MySQL InnoDB `DYNAMIC` 방식이 유리할 여지가 있다.

따라서 Rilog.에서는 단순히 압축 해제 비용만을 기준으로 판단하기보다, 실제 Cache Hit Ratio와 Storage I/O, CPU 사용량을 함께 관찰하는 것이 필요하다.

### Rilog. 의 조회 패턴 분석

Rilog.의 실제 조회 패턴을 분석해보자. 메인 피드와 프로필 목록에서는 `content`가 필요하지 않기 때문에 큰 데이터를 밖으로 분리하는 것이 분명 유리하다. 그러나 Rilog.의 핵심 기능은 결국 **게시글을 읽는 것**이다. 따라서, 사용자가 게시글 상세 화면에 진입하면 결국 `content`가 필요하다.

PostgreSQL의 `TOAST`에 저장된 본문을 읽는다면 다음과 같은 과정이 필요하다.

1. Post row 조회
2. TOAST pointer 확인
3. TOAST table 접근
4. 여러 chunk 조회
5. chunk 조립
6. 압축되어 있다면 decompress
7. content 반환

따라서 본문 상세 조회에서는 `TOAST` table에 대한 추가적인 접근과 데이터 조립, 경우에 따라 압축 해제 비용이 발생한다.

MySQL `DYNAMIC` 도 마찬가지다.

1. Clustered Index Record 조회
2. Overflow Pointer 확인
3. Overflow Page 접근
4. 필요한 Page 조회
5. content 반환

`DYNAMIC` 역시 본문이 off-page로 저장되어 있다면 상세 조회에서는 Overflow Page에 추가로 접근해야 한다.

> 결론: **큰 데이터를 밖으로 분리하는 방식은 목록 조회에서는 둘 다 유리하지만, 본문 상세 조회에서는 둘 다 추가적인 데이터 접근 비용을 지불한다.** PostgreSQL은 여기에 압축 해제 비용이 추가될 수 있지만, 반대로 압축되어 있기 때문에 디스크에서 실제로 읽어야 하는 데이터 양이 더 적을 수도 있다.

Rilog. 에서 생각해봐야 할 케이스는 다음과 같이 정리할 수 있다.
1. 해당 게시글이 빈번히 조회된다. (잦은 content 압축 해제 필요)
    1. 대부분 cache hit + CPU 비용이 중요
        - 잘 맞는 방식:
            - MySQL: `DYNAMIC`(Not `COMPRESSED`)
            - 이유: 압축 해제 CPU 비용 없이 큰 본문을 off-page로 관리
    2. cache miss 많음 + Storage I/O가 병목
        - 잘 맞는 방식:
            - PostgreSQL: `TOAST`
            - 이유: CPU 비용 지불에 부담이 없을 땐 오히려 빠를 수 있다.
2. content가 필요없는 조회가 많이 발생한다. (content 압축 해제 필요 X)
    - 잘 맞는 방식:
        - PostgreSQL: `TOAST`
        - MySQL: `DYNAMIC`(Not `COMPRESSED`)
        - 이유: 둘 다 큰 본문을 외부로 빼고 목록에서는 안 읽을 수 있음
3. 거의 조회되지 않는 글.
    - 잘 맞는 방식:
        - PostgreSQL: `TOAST`
        - MySQL: `COMPRESSED`
        - 이유: 압축으로 저장 공간/I/O를 줄이는 이점이 크고, 압축 해제 빈도는 낮음
## 검색

Rilog.의 게시글 `content`는 JSON 형태로 저장된다. 따라서 검색 기능을 비교할 때 MySQL의 `JSON`과 PostgreSQL의 `jsonb`가 검색 측면에서 어떤 차이를 가지는지 먼저 확인할 필요가 있다.
### MySQL: `JSON`

MySQL은 `JSON` 타입을 지원하며, **내부 필드 조회와 JSON Path 기반 검색**이 가능하다.
```sql
SELECT JSON_VALUE(content, '$.version')
FROM post;
```

특정 JSON 내부 값을 자주 조회한다면 expression index나 generated column을 활용해 인덱스를 구성할 수도 있다.
```sql
CREATE INDEX idx_content_version
ON post (
    (JSON_VALUE(content, '$.version' RETURNING UNSIGNED))
);
```

다만 MySQL에서는 `JSON` 컬럼 전체에 일반 인덱스를 직접 걸 수는 없다. JSON 내부에서 검색하고 싶은 특정 값을 추출하여 별도의 인덱스를 구성하는 방식이 필요하다. 또한 MySQL의 `FULLTEXT INDEX`는 `CHAR`, `VARCHAR`, `TEXT` 컬럼에 대해서만 사용할 수 있다. 따라서 `content`를 `JSON` 타입으로 저장한 상태에서는 JSON 컬럼 자체에 바로 `FULLTEXT INDEX`를 구성할 수 없다.

`title`, `description`이 `VARCHAR/TEXT`라면 다음과 같은 검색은 가능하다.
```sql
CREATE FULLTEXT INDEX idx_post_search
ON post (title, description);
```

```sql
SELECT *
FROM post
WHERE MATCH(title, description)
      AGAINST('postgresql');
```

하지만 JSON 형태의 `content`까지 Full Text Search에 포함하고 싶다면, 검색에 사용할 본문 텍스트를 별도의 `TEXT` 컬럼으로 추출하거나 검색용 컬럼을 별도로 관리하는 등의 추가적인 설계가 필요하다.
- 이렇게 까진 할 필요가 없다고 생각한다. 그래서, **MySQL은 Content 내 검색을 배제하고자 한다.**

### PostgreSQL: `jsonb`

PostgreSQL의 `jsonb`는 JSON 데이터를 파싱된 바이너리 형태로 저장한다. PostgreSQL 공식 문서에서는 `json`이 입력 문자열을 그대로 저장하는 반면, `jsonb`는 내부적으로 분해된 binary format으로 저장하기 때문에 입력 시 변환 비용은 조금 더 들지만 이후 처리 속도가 빠르고 인덱싱도 가능하다고 설명한다.
```sql
CREATE TABLE post (
    id BIGINT,
    content JSONB
);
```

PostgreSQL `jsonb`의 큰 특징 중 하나는 **GIN Index**를 사용할 수 있다는 점이다.
#### GIN이란?

GIN은 `Generalized Inverted Index`의 약자로, 하나의 값 안에 여러 요소가 들어 있는 데이터를 검색하기 위한 역색인 구조이다.

일반적인 B-tree가 `값 -> row 위치` 를 찾는 구조라면, GIN은 개념적으로
```text
"java"
→ Post 1
→ Post 8
→ Post 15

"spring"
→ Post 1
→ Post 4
```

처럼 **각 요소가 어느 row에 존재하는지 역으로 저장**한다.

PostgreSQL에서는 `jsonb`의 key/value 검색뿐 아니라 Full Text Search의 `tsvector`에도 GIN을 사용할 수 있다. PostgreSQL은 Full Text Search에서 GIN을 권장되는 인덱스 타입으로 설명한다.
- 예를 들어 `jsonb` 자체에는 다음과 같이 GIN 인덱스를 걸 수 있다.

```sql
CREATE INDEX idx_post_content
ON post
USING GIN (content);
```

그리고:
```sql
SELECT *
FROM post
WHERE content @> '{"type": "code"}';
```

처럼 JSON 내부의 특정 구조나 값을 찾는 데 활용할 수 있다.

다만 여기서 중요한 점은 `jsonb`**에 GIN을 건다고 자동으로 블로그 본문 Full Text Search가 되는 것은 아니라는 점**이다.
- `jsonb` GIN은 JSON의 key, value, containment 등의 검색을 빠르게 하기 위한 것이고, 자연어 본문 검색을 위해서는 PostgreSQL의 Full Text Search 기능인 `tsvector`를 사용해야 한다.
- PostgreSQL은 `jsonb_to_tsvector()`를 제공하기 때문에 JSON 내부의 문자열 값을 검색 문서로 변환할 수도 있다.

개념적으로
- `JSONB content -> jsonb_to_tsvector() -> 검색 가능한 단어 목록 -> GIN Index` 가 된다.


> 즉, PostgreSQL은 `jsonb`로 저장된 본문을 DB 내부 Full Text Search까지 연결하는 기능이 MySQL보다 자연스럽게 제공되는 편이다.

### 결국 핵심은 `content`까지 검색에 포함시켜야 하는가

더 중요한 포인트는 MySQL `JSON`과 PostgreSQL `jsonb`의 기능 차이보다 **검색 대상을 어디까지 포함할 것인가** 이다. 현재는 두 가지 방식을 고려할 수 있다. (MySQL 에서는 `content` 내 검색을 배제하였기 때문이다.)

#### `title + description` vs `title + description + content`

`title + description` 의 장점은 검색해야 하는 데이터의 양이 작고 구조가 단순하다는 점이다. 사용자가 개발 관련 키워드를 검색했을 때 title이나 description에 핵심 키워드가 포함되어 있다면 대부분의 검색 요구를 만족할 수 있다.
- 이것은 글 작성자가 노력해주어야 하는 부분이라는 생각도 들었다. `title`, `description`에 그 글의 핵심을 담은 글이 좋은 글의 조건이라고도 생각된다.
- 또한 MySQL에서도 두 컬럼이 `VARCHAR/TEXT`라면 `FULLTEXT INDEX`를 활용할 수 있기 때문에 구현이 비교적 간단하다.
- 본문 전체를 인덱싱하지 않으므로 인덱스 크기와 게시글 수정 시 검색 인덱스 갱신 비용도 상대적으로 작다.


반면 `title + description`의 단점은 명확하다.  
예를 들어 제목이 `Spring에서 겪었던 문제를 해결하며` 다음과 같고, 본문 중간에만 자세한 내용이 있다고 하면, 검색으로 해당 게시글을 찾지 못할 수 있다.

하지만 `content`를 검색 조건에 포함하지 않는 것이 반드시 단점이라고 보기는 어렵다. 본문 전체를 검색 대상으로 포함하는 순간 검색 결과가 불필요한 키워드에 더 쉽게 노출될 수 있기 때문이다.
- 예를 들어 게시글의 핵심 주제는 검색어와 전혀 관련이 없더라도, 본문 중간에 해당 단어가 한 번 등장했다는 이유만으로 검색 결과에 노출될 수 있다. 이러한 결과가 많아질수록 사용자는 원하는 정보를 찾기 위해 더 많은 결과를 걸러내야 하고, 결과적으로 검색 경험이 저하될 수 있다.

따라서 `content`를 검색 대상으로 포함하는 것은 검색 범위를 넓혀 recall을 높일 수 있다는 장점이 있지만, 동시에 관련성이 낮은 결과까지 포함시켜 precision을 떨어뜨릴 수 있다는 trade-off가 존재한다. **개인적인 경험으로는 검색 결과에서** `content`**에 키워드가 포함되었지만, 관련 없는 글이 노출되는** `False Positive`**가 한 번만 발생해도 사용자 경험이 크게 저하되었다.**

추가적으로, 본문 전체에 검색 인덱스를 구성한다면 게시글을 수정할 때마다 큰 검색 문서를 다시 처리해야 한다는 점도 고려할 필요가 있다.

### Elasticsearch가 추가된다면?

Rilog. 는 검색 기능을 고도화할 경우 Elasticsearch와 같은 별도의 검색 엔진을 도입할 가능성이 있다. (하지만, 굉장히 먼 미래의 이야기라고 생각된다.)
- 만약 그렇게 되면 구조 자체가 달라진다.

```text
PostgreSQL / MySQL
→ 원본 데이터 저장

Elasticsearch
→ 검색 전용 데이터 저장

title
description
content
tags
author
...
↓
검색 인덱스
```

이 경우 `content`가 PostgreSQL `jsonb`인지 MySQL `JSON`인지가 검색 성능에 미치는 영향은 크게 줄어든다. RDB에서는 게시글의 원본 데이터를 관리하고, 검색에 필요한 `title`, `description`, `content` 등을 Elasticsearch에 전달하여 역색인을 구성하면 되기 때문이다.

> 결론: Rilog.에서 초기 검색 범위를 `title + description`으로 한정한다면 PostgreSQL, MySQL 모두 대응 가능하다. 물론, 검색 쪽으로 PostgreSQL이 기능적으로 풍부한 것은 사실이지만, 현재 우리는 그 기능을 충분히 활용하지 않으므로 **MySQL이 더 단순하고 합리적인 선택이라고 판단했다**. 또한, 검색 기능을 고도화할 경우 Elasticsearch를 도입할 가능성이 높다는 점도, 현재 단계에서 이러한 선택을 내리는 데 부담을 줄여주었다.

### 마무리

이번 비교에서 중요했던 건 두 DB의 우열보다, 각자의 장점이 지금 리로그에 얼마나 필요한지였다. PostgreSQL의 Partial Index, TOAST, jsonb는 매력적이지만 현재의 조회 패턴과 검색 범위에서는 결정적인 이유가 되지 않았다. 수정 기록과 인덱스 실험도 함께 검토한 끝에 MySQL로 전환하기로 했다. 다만 발행 상태의 분포나 압축 효과처럼 아직 예상에 기댄 부분은 운영 데이터를 보며 다시 확인해야 한다. 이번에 배운 건 기술의 장점을 나열하는 것보다 우리 서비스에서 그 장점이 필요한 이유를 설명하는 일이 더 중요하다는 점이다.