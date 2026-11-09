# 들어가며

기존에는 성능 개선을 할 때 주로 코드를 먼저 살펴보았습니다. 코드에서 반복 쿼리와 같이 성능에 영향을 줄 수 있는 부분을 발견하면 이를 개선하고, 전후 성능을 비교하는 방식이었습니다.

하지만 이 방식으로는 **발견한 문제가 실제 서비스에서도 개선이 필요한 문제인지**, 그리고 **개선 결과가 운영 환경에서도 유효한지** 설명하기 어려웠습니다. 그래서 우테코 방학 동안 운영 중인 개인 블로그에 모니터링을 구축했습니다. 운영 환경에서 수집한 데이터를 근거로 개선할 문제를 고르고, 개선 효과도 같은 데이터로 확인하기 위해서였습니다.

모니터링을 연결하자 이전에는 보이지 않던 문제가 보이기 시작했습니다.

그중 하나가 게시물 목록을 조회하는 `GET /api/v1/posts`였습니다. 평균 응답 시간은 453ms였지만, 느린 쪽 상위 5% 요청의 경계값인 p95 응답 시간은 1.62초였습니다. 두 값 모두 Grafana 대시보드에서 확인한 운영 환경의 응답 시간 지표입니다.

느린 요청의 Trace를 따라가 보니 게시물 수에 비례해 동일한 조회가 반복되고 있었습니다. 이를 일괄 조회하도록 변경한 결과, 배포 전후 운영 환경에서 평균 응답 시간은 **449ms → 45.2ms**, p95는 **1.51초 → 94.3ms**로 감소했습니다. 처음 문제를 발견했을 때와 배포 전후 비교는 집계 구간이 달라 개선 전 수치가 조금 다릅니다.

코드에서 문제를 먼저 찾은 것이 아니라, **모니터링을 통해 실제 운영 환경의 문제를 발견하고 원인을 추적한 뒤 개선 결과까지 다시 확인할 수 있었습니다.**

이번 글에서는 관측 가능성을 간단히 살펴본 뒤, 모니터링에서 발견한 API의 성능 문제를 어떻게 정의하고 원인을 찾아 개선했는지 적어보려고 합니다. 모니터링을 구축했지만 대시보드를 어떻게 문제 해결로 이어가야 할지 막막한 분, 또는 N+1과 같은 반복 조회를 데이터로 찾아 검증하는 과정이 궁금한 분께 도움이 되었으면 합니다.

---

# 1. 관측 가능성이란?

개발 과정에서 충분히 테스트하더라도 운영 환경에서는 예상하지 못한 문제가 발생합니다. 실제 사용자의 요청 패턴이나 데이터의 양, 인프라의 상태는 개발 환경과 다르기 때문입니다.

모든 문제를 사전에 방지할 수 없다면 문제가 발생했을 때 이를 발견하고 원인을 분석할 수 있어야 합니다. 이때 필요한 것이 **관측 가능성(Observability)**입니다.

관측 가능성은 시스템이 외부로 내보내는 데이터를 바탕으로 내부에서 어떤 일이 발생하고 있는지 이해할 수 있는 능력을 의미합니다.

예를 들어 응답 시간 그래프를 통해 특정 API가 느려졌다는 사실을 알 수 있습니다. 하지만 이것만으로는 DB 조회가 느린 것인지, 외부 API의 응답을 기다리는 것인지, 동일한 작업이 불필요하게 반복되고 있는지 알기 어렵습니다.

미리 정의된 지표를 통해 시스템의 상태를 확인하는 것이 모니터링이라면, 관측 가능성은 **"왜 이런 문제가 발생했을까?"라는 질문에 수집된 데이터를 바탕으로 답할 수 있는 상태**라고 이해했습니다.

저는 관측 가능성을 다음 질문으로 판단할 수 있다고 생각했습니다.

> 예상하지 못한 문제가 발생했을 때, 코드를 수정해 계측을 추가하지 않고도 이미 수집한 데이터로 원인을 분석할 수 있는가?

여기서 "코드를 수정하지 않는다"는 것은 아무런 준비 없이도 원인을 알 수 있다는 뜻이 아닙니다. Metrics와 Trace 같은 데이터를 **평소에 미리 수집해 두어서**, 문제가 생긴 뒤에 로그를 추가하고 다시 배포한 다음 문제가 재현되기를 기다리지 않아도 된다는 뜻입니다.

## 1.1 Metrics, Logs, Traces

관측 가능성을 확보하기 위해 대표적으로 수집하는 데이터는 Metrics, Logs, Traces입니다.

| 구분 | 확인할 수 있는 정보 | 이번 사례에서의 역할 |
| --- | --- | --- |
| Metrics | 응답 시간, 오류율, 요청 수 등 시스템 상태의 변화 | 성능 문제 발견 및 개선 효과 확인 |
| Logs | 특정 시점의 이벤트, 예외, 처리 결과 | 직접 사용하지 않음 |
| Traces | 요청이 거친 경로와 작업별 소요 시간 | 반복 조회 원인 추적 |

각 데이터는 서로 다른 정보를 제공합니다.

Metrics로 API의 응답 시간이 증가했다는 사실을 알 수 있지만, 개별 요청에서 어떤 작업이 오래 걸렸는지는 확인하기 어렵습니다.

반면 Trace를 사용하면 하나의 HTTP 요청을 처리하는 과정에서 발생한 작업들을 Span 단위로 확인할 수 있습니다. 어떤 Repository 메서드가 호출되었고, SQL이 몇 번 실행되었으며, 각 작업에 얼마나 시간이 걸렸는지 분석할 수 있습니다.

이번 사례에서는 **Metrics를 통해 문제를 발견하고, Trace로 원인을 추적한 뒤, 다시 Metrics를 통해 개선 효과를 검증**했습니다.

## 1.2 블로그 모니터링 구성

개인 블로그의 모니터링은 다음과 같이 구성했습니다.

| 데이터 | 수집 | 저장 |
| --- | --- | --- |
| Metrics | Prometheus | Prometheus |
| Traces | Grafana Alloy | Tempo |
| Logs | Grafana Alloy | Loki |

Spring Boot 애플리케이션에는 Actuator와 Micrometer Prometheus Registry를 적용했습니다. 이를 통해 HTTP 요청 수와 응답 시간 등 Metrics를 수집했습니다.

Trace 계측에는 OpenTelemetry Java Agent를 사용했습니다.

```bash
-javaagent:/opt/opentelemetry-javaagent.jar
```

별도의 Trace 샘플링 비율을 지정하지 않고 기본 샘플러인 `parentbased_always_on`을 사용했습니다. 즉, 들어오는 모든 요청의 Trace를 수집했습니다.

수집된 Trace는 Grafana Alloy와 Tempo를 통해 저장 및 조회할 수 있도록 구성했습니다.

이렇게 구성한 모니터링을 통해 API의 응답 시간뿐 아니라 실제 요청을 처리하는 과정에서 어떤 작업이 발생하는지도 확인할 수 있게 되었습니다.

---

# 2. 평균 453ms와 p95 1.62초의 차이

2026년 9월 7일부터 Grafana 대시보드를 통해 운영 환경의 API 성능을 분석하기 시작했습니다.

그 과정에서 개선이 필요한 API를 확인할 수 있었습니다.

![GET /api/v1/posts API 응답 시간 모니터링](http://api.yeo-li.com/api/v1/images/01dcafbf1811466084acf5ccc3e60d02.png)

`GET /api/v1/posts`는 확인한 집계 구간에서 1,682번 호출되었는데, 평균 응답 시간이 453ms인 반면 p95 응답 시간은 1.62초였습니다.

> **p95란?**
>
> 전체 요청의 응답 시간을 작은 값부터 큰 값 순으로 나열했을 때 95번째 백분위에 위치하는 값입니다.
>
> p95가 1.62초라면 전체 요청의 약 95%가 1.62초 이내에 처리되었다는 의미입니다.

평균만 봤다면 단순히 조금 느린 API라고 생각할 수 있었습니다. 하지만 p95까지 함께 확인하니 일부 요청의 응답 시간이 크게 증가하고 있다는 사실을 확인할 수 있었습니다.

1,682건의 요청에서 평균 응답 시간은 453ms였지만 p95는 1.62초로 약 3.6배 차이가 있었습니다.

물론 이 수치만으로 성능 문제라고 단정할 수는 없습니다.

따라서 **요청별 응답 시간의 편차가 왜 발생하는지 확인하기 위해 느린 요청의 Trace를 분석해보기로 했습니다.**

---

# 3. 게시물 수에 비례하는 반복 조회

## 3.1 Trace에서 발견한 반복 조회

`GET /api/v1/posts`를 자세히 확인하기 위해 Trace를 관측해보았습니다.

먼저 2026년 9월 11일 발생한 느린 요청입니다. 해당 요청의 응답 시간은 1.92초였습니다.

![1.92초가 소요된 게시물 목록 조회 Trace](http://api.yeo-li.com/api/v1/images/dc57dfa61d5f4415915f62db12ba5531.png)

확인 결과 개별 Query의 실행 시간에서 특별한 병목은 발견되지 않았습니다.

대신 `PostTagRepository.findPostTagsByPost_Id`와 `PostRepository.findPostsByCategoryAndIsPublishedTrueOrderByPublishedAtDesc`, 그리고 카테고리를 조회하는 SQL이 반복 실행되고 있었습니다.

그렇다면 빠르게 처리된 요청은 어떨까요?

2026년 9월 12일 발생한 동일 API의 요청을 확인했습니다. 해당 요청의 응답 시간은 228.74ms였습니다.

![228.74ms가 소요된 게시물 목록 조회 Trace](http://api.yeo-li.com/api/v1/images/8ee22258f8144a269c0e3d86c352159e.png)

느린 요청보다 8배 이상 빠르게 처리되었지만, 여기에서도 게시물 수에 비례해 동일한 조회 메서드가 반복 호출되고 있었습니다.

즉, 느린 요청과 빠른 요청 모두에서 **게시물 수에 비례해 동일한 조회가 반복되는 구조**를 확인할 수 있었습니다.

## 3.2 일부 요청만 느렸던 이유

그렇다면 반복 조회가 발생하는 것은 동일한데, 왜 어떤 요청은 1.92초가 걸리고 다른 요청은 228.74ms 만에 처리되었을까요?

두 Trace를 비교해보았습니다.

먼저 Span 수는 느린 요청이 685개, 빠른 요청이 691개였습니다. 요청에서 처리한 작업의 개수는 큰 차이가 없었습니다.

차이는 개별 작업의 소요 시간에 있었습니다.

| 작업 | 빠른 요청 | 느린 요청 |
| --- | ---: | ---: |
| `PostRepository.findAllPosts` | 8.14ms | 28.26ms |
| 첫 번째 `PostTagRepository.findPostTagsByPost_Id` | 2.78ms | 20.22ms |
| 요청 전체 | 228.74ms | 1.92초 |

특정 작업 하나가 전체 지연의 대부분을 차지하는 것이 아니라, 여러 작업에서 조금씩 시간이 더 소요되고 있었습니다.

또한 느린 요청에서 `findAllPosts`는 28.26ms가 걸렸지만, 내부 SQL 실행 Span은 3.34ms였습니다. `findPostTagsByPost_Id` 역시 20.22ms 중 SQL 실행 Span 두 개의 합은 약 6.3ms였습니다.

SQL 실행 자체보다 Repository 메서드를 처리하는 과정에서 더 많은 시간이 소요되고 있었습니다.

다만 각 작업이 느려진 직접적인 원인까지는 확인하지 못했습니다. 분석에 필요한 데이터가 남아 있지 않았기 때문인데, 이 부분은 6.1에서 다시 다룹니다.

하지만 한 가지는 분명했습니다.

**반복 조회가 개별 작업의 지연을 전체 응답 시간으로 누적시키는 구조**였습니다.

게시물 수가 증가할수록 같은 작업이 반복되기 때문에 작은 지연도 여러 번 누적될 수 있었습니다.

따라서 API의 속도 저하 문제를 다음과 같이 정의했습니다.

> **속도 저하 문제 정의**
>
> 게시물 목록을 조회할 때 각 게시물마다 태그와 카테고리의 게시물 수를 개별적으로 조회하면서, 게시물 수에 비례해 추가 쿼리가 반복 실행되고 있다.

## 3.3 코드에서 확인한 반복 조회의 원인

Trace에서 발견한 반복 조회가 실제 코드에서는 어떻게 발생하고 있는지 확인해보았습니다.

전체 게시물을 반환하는 메서드입니다.

```java
public List<PostResponse> getAllPosts() {
    List<Post> allPosts = postRepositoryFacade.findAllPosts();
    return convertPostResponse(allPosts);
}
```

전체 게시물을 조회한 뒤 `convertPostResponse()`를 호출합니다.

`convertPostResponse()`는 `List<Post>`를 클라이언트에 반환할 `List<PostResponse>`로 변환하는 역할을 합니다. 반복 조회가 발생하는 줄에는 `// ①`~`// ③` 주석을 달아 두었습니다.

```java
public List<PostResponse> convertPostResponse(List<Post> posts) {
    List<PostResponse> postResponses = new ArrayList<>();

    for (Post post : posts) {
        // ① 게시물마다 태그 조회 쿼리 실행
        List<Tag> tags = postTagService.findTagByPostId(post.getId());
        List<String> tagNames = new ArrayList<>();

        for (Tag tag : tags) {
            tagNames.add(tag.getTagName());
        }

        postResponses.add(new PostResponse(
            post.getId(),
            post.getTitle(),
            post.getSummary(),
            post.getAuthor(),
            post.getContent(),
            post.getViews(),
            post.getIsPublished(),
            post.getPublishedAt(),
            CategoryResponse.builder()
                .categoryId(post.getCategory().getId())
                // ② 카테고리 엔티티 지연 로딩
                .categoryName(post.getCategory().getCategoryName())
                .categoryColor(post.getCategory().getCategoryColor())
                .categoryDescription(post.getCategory().getCategoryColor())
                // ③ 게시물마다 카테고리별 게시물 수 조회 쿼리 실행
                .postCount(postRepositoryFacade.countPostByCategory(post.getCategory()))
                .build(),
            tagNames
        ));
    }

    return postResponses;
}
```

문제는 `for`문 내부에 있었습니다.

`convertPostResponse()`에서 모든 게시물을 순회하면서 다음 두 Repository 메서드가 반복 호출되고 있었습니다.

1. ① `PostTagRepository.findPostTagsByPost_Id()`: 게시물의 태그 조회
2. ③ `PostRepository.findPostsByCategoryAndIsPublishedTrueOrderByPublishedAtDesc()`: 카테고리별 게시물 수 조회

또한 ② `post.getCategory().getCategoryName()`에 접근할 때 카테고리 엔티티의 지연 로딩도 발생하고 있었습니다.

특히 카테고리별 게시물 수를 구하는 메서드를 확인해보니 다음과 같이 구현되어 있었습니다.

```java
public int countPostByCategory(Category category) {
    List<Post> posts =
        postRepository.findPostsByCategoryAndIsPublishedTrueOrderByPublishedAtDesc(
            category);

    return posts.size();
}
```

게시물의 개수를 얻기 위해 `COUNT(*)`를 사용하는 것이 아니라, 카테고리에 해당하는 게시물 목록을 모두 조회한 뒤 `size()`로 개수를 계산하고 있었습니다.

즉, **게시물 수 하나를 구하기 위해 본문까지 포함한 게시물 데이터를 불필요하게 조회**하고 있었습니다.

2026년 9월 12일 이전에 생성되어 현재 DB에 남아 있는 게시물은 102개였습니다.

게시물 102개를 기준으로 목록 조회 한 번과 게시물마다 세 종류의 조회가 모두 실행된다고 가정하면, 최대 307회에 해당하는 조회 구조였습니다. 실제 Trace에서는 약 220회의 SQL이 확인됐습니다.

게시물이 증가할수록 추가 조회 횟수도 함께 증가하는 구조였습니다.

Trace에서 발견한 **"게시물 수에 비례해 동일한 조회가 반복된다"**는 현상이 코드에서도 확인되었습니다.

모니터링을 통해 정의한 문제와 코드에서 발견한 원인이 일치했으므로, 이제 해당 부분의 반복 조회를 제거해보기로 했습니다.

---

# 4. 일괄 조회를 통한 쿼리 호출 횟수 감소

개선 방향은 **필요한 데이터를 반복해서 하나씩 조회하지 않고 한 번에 가져오는 것**이었습니다.

두 가지 방법을 고민했습니다.

## 4.1 개선 방안

1. Post를 조회할 때 Tag도 함께 가져오는 쿼리를 만들어 사용한다.
2. `List<Long> postIds` 등을 이용해 필요한 태그와 카테고리별 게시물 수를 한 번에 조회한 뒤 `Map`으로 변환한다.

저는 두 번째 방법을 선택했습니다.

Post를 조회할 때 항상 태그까지 가져오도록 변경하기보다, 현재 응답을 만드는 데 필요한 데이터를 별도의 일괄 조회로 가져오는 편이 기존 조회 로직을 크게 변경하지 않으면서 문제를 해결하기 적절하다고 판단했습니다.

또한 첫 번째 방법으로는 태그 조회를 개선할 수 있지만, 카테고리별 게시물 수를 조회하는 문제는 별도로 해결해야 했습니다.

반면 두 번째 방법은 태그와 카테고리별 게시물 수를 각각 일괄 조회할 수 있었습니다.

기존에는 게시물 수가 증가할수록 추가 조회 횟수도 함께 증가했지만, 일괄 조회를 사용하면 **게시물 수에 비례하던 추가 쿼리 호출 횟수를 고정된 횟수로 줄일 수 있습니다.** 게시물 수를 N이라고 하면, 쿼리 호출 횟수가 O(N)에서 O(1)로 줄어드는 것입니다.

> 여기서 쿼리 호출 횟수가 O(N)에서 O(1)로 줄었다는 것은 전체 API의 시간복잡도가 O(1)이 되었다는 의미가 아닙니다.
>
> 태그와 카테고리별 게시물 수를 조회하는 DB 쿼리 호출 횟수가 고정된 횟수가 되었다는 의미입니다. 조회된 게시물을 순회하고 응답 객체를 만드는 작업은 여전히 N에 비례합니다.

## 4.2 개선 코드

```java
public List<PostResponse> convertPostResponse(List<Post> posts) {
    List<PostResponse> postResponses = new ArrayList<>();
    List<Long> postIds = new ArrayList<>();
    List<Category> categories = new ArrayList<>();

    for (Post post : posts) {
        postIds.add(post.getId());
        categories.add(post.getCategory());
    }

    Map<Long, List<String>> tagNamesByPostIds =
        postTagService.findTagNamesByPostIds(postIds);

    Map<Long, Integer> postCountByCategoryIds =
        postRepositoryFacade.countPostsByCategory(categories);

    for (Post post : posts) {
        List<String> tagNames =
            tagNamesByPostIds.getOrDefault(post.getId(), new ArrayList<>());

        postResponses.add(new PostResponse(
            post.getId(),
            post.getTitle(),
            post.getSummary(),
            post.getAuthor(),
            post.getContent(),
            post.getViews(),
            post.getIsPublished(),
            post.getPublishedAt(),
            CategoryResponse.builder()
                .categoryId(post.getCategory().getId())
                .categoryName(post.getCategory().getCategoryName())
                .categoryColor(post.getCategory().getCategoryColor())
                .categoryDescription(post.getCategory().getCategoryDescription())
                .postCount(
                    postCountByCategoryIds.getOrDefault(
                        post.getCategory().getId(),
                        0
                    )
                )
                .build(),
            tagNames
        ));
    }

    return postResponses;
}
```

기존에는 `for`문을 돌 때마다 필요한 데이터를 DB에서 조회했습니다.

개선 후에는 먼저 게시물 ID와 카테고리를 모은 뒤, 필요한 태그와 카테고리별 게시물 수를 **반복문 밖에서 일괄 조회**하도록 변경했습니다.

이후 반복문에서는 조회 결과를 저장한 `Map`에서 필요한 값을 가져오도록 했습니다.

참고로 기존 코드에서는 `categoryDescription`에 카테고리 색상(`getCategoryColor()`)이 잘못 들어가고 있었는데, 이번에 함께 `getCategoryDescription()`으로 수정했습니다.

## 4.3 실제 일괄 조회 쿼리

### 태그 일괄 조회

먼저 태그 조회를 개선했습니다.

기존에는 게시물마다 `findPostTagsByPost_Id()`를 호출했지만, 개선 후에는 게시물 ID 목록을 이용해 한 번에 조회하도록 변경했습니다.

`PostTagRepository`에 다음 JPQL을 추가했습니다.

```java
@Query("""
    SELECT new com.yeo_li.yeol_post.domain.post_tag.PostTagName(
        pt.post.id,
        pt.tag.tagName
    )
    FROM PostTag pt
    WHERE pt.post.id IN :postIds
    """)
List<PostTagName> findTagNamesByPostIds(
    @Param("postIds") List<Long> postIds
);
```

`IN` 조건으로 여러 게시물의 태그를 한 번에 조회하고, 게시물 ID와 태그 이름만 반환하도록 했습니다.

조회된 결과는 `Map<Long, List<String>>`으로 변환해 게시물별로 사용할 수 있도록 했습니다.

### 카테고리별 게시물 수 일괄 조회

카테고리별 게시물 수 조회도 개선했습니다.

기존에는 해당 카테고리의 게시물 목록을 조회한 뒤 `size()`로 개수를 계산했습니다.

이를 `COUNT()`와 `GROUP BY`를 사용해 DB에서 직접 집계하는 방식으로 변경했습니다.

```java
@Query("""
    SELECT new com.yeo_li.yeol_post.domain.category.CategoryPostCount(
        p.category.id,
        COUNT(p.id)
    )
    FROM Post p
    WHERE p.category IN :categories
      AND p.isPublished = true
    GROUP BY p.category.id
    """)
List<CategoryPostCount> countPublishedPostsByCategories(
    @Param("categories") List<Category> categories
);
```

이제 게시물 전체를 메모리로 가져오지 않고도 카테고리별 게시물 수를 조회할 수 있습니다.

조회 결과는 `Map<Long, Integer>`로 변환해 응답 객체를 생성할 때 사용했습니다.

이렇게 태그와 카테고리별 게시물 수를 일괄 조회하도록 변경하면서 게시물 수에 비례해 반복되던 Repository 호출을 제거했습니다.

## 4.4 개선 후 Trace 확인

실제 Trace에서도 변화가 나타났습니다.

개선 후 확인한 게시물 목록 조회 Trace에서는 전체 10개의 Span과 3회의 SQL 실행이 기록됐습니다.

| 조회 작업 | SQL 실행 시간 |
| --- | ---: |
| 게시물 목록 조회 | 2.73ms |
| 태그 일괄 조회 | 1.56ms |
| 카테고리별 게시물 수 집계 | 1.40ms |

게시물 목록 조회, 태그 일괄 조회, 카테고리별 게시물 수 집계가 각각 한 번씩 실행됐으며, 개별 카테고리 조회는 나타나지 않았습니다.

반복 조회를 제거하면서 요청을 처리하는 작업도 크게 줄었습니다.

하지만 반복 쿼리를 제거했다고 해서 실제 성능까지 개선되었다고 볼 수는 없습니다.

따라서 코드의 변경이 실제 응답 시간에도 영향을 주었는지 확인해보았습니다.

---

# 5. 로컬과 운영 환경에서의 성능 검증

코드에서 반복 조회를 제거했지만, 코드가 바뀌었다는 사실만으로 성능이 개선되었다고 판단할 수는 없습니다.

그래서 검증에 앞서 무엇을 근거로 개선되었다고 판단할지 먼저 정했습니다.

| 단계 | 판단 기준 | 이 기준을 정한 이유 |
| --- | --- | --- |
| 1. 로컬 검증 | 같은 환경·같은 데이터에서 평균과 p95 응답 시간이 감소하는가 | 환경 차이 없이 코드 변경의 효과만 확인하기 위해 |
| 2. 운영 검증 | 배포 후 운영 환경의 p95가 감소하고, 평균과 p95의 격차가 줄어드는가 | 문제를 처음 발견한 계기가 평균과 p95의 큰 차이였기 때문에 |

먼저 **동일한 로컬 환경에서 개선 전후를 비교**하고, 실제 배포 이후에는 **운영 환경의 지표가 어떻게 변했는지 확인**했습니다.

## 5.1 로컬에서 확인한 80% 이상의 응답 시간 감소

배포 전에 운영 환경과 동일한 데이터를 준비하고 k6를 이용해 개선 전후의 성능을 측정했습니다.

VU를 1로 설정해 요청을 순차적으로 전송했기 때문에 동시 요청 상황은 측정에 포함되지 않습니다.

### 측정 조건

| 항목 | 조건 |
| --- | --- |
| 환경 | MacBook M3 |
| DB | MySQL |
| 데이터 | 운영 환경과 동일 |
| VU | 1 |
| 요청 횟수 | 1,000회 |

### 측정 결과

| 지표 | 개선 전 | 개선 후 | 변화 |
|---|---:|---:|---:|
| **평균 응답 시간** | 50.67ms | **9.98ms** | **80.3% 감소** |
| **p95 응답 시간** | 73.97ms | **12.86ms** | **82.6% 감소** |
| **최대 응답 시간** | 352.08ms | **154.74ms** | **56.0% 감소** |
| **요청 성공률** | 100% | **100%** | 동일 |

로컬에서 실행했기 때문에 실제 운영 서버와 비교하면 전체적인 응답 시간은 빠르게 측정되었습니다.

따라서 로컬과 운영 환경의 절대적인 응답 시간을 비교하기보다는, **동일한 로컬 환경에서 개선 전후가 얼마나 달라졌는지**를 확인하는 데 초점을 맞췄습니다.

그 결과 평균 응답 시간은 약 80.3%, p95는 약 82.6% 감소했습니다. 최대 응답 시간 역시 약 56% 감소했습니다.

동일한 조건에서 반복 쿼리를 제거한 뒤 응답 시간이 감소했음을 확인할 수 있었습니다.

하지만 로컬 환경에서의 결과만으로 실제 운영 환경에서도 동일하게 개선되었다고 판단할 수는 없습니다.

실제 운영 환경에서도 같은 변화가 나타나는지 확인해보았습니다.

## 5.2 운영에서 확인한 p95 93.8% 감소

개선한 코드를 운영 서버에 배포한 뒤 모니터링을 통해 실제 API 응답 시간이 어떻게 달라졌는지 확인했습니다.

운영 환경은 Vultr의 단일 컴퓨팅 인스턴스에 Spring Boot 애플리케이션과 MySQL을 함께 실행하는 구조입니다.

### 개선 전 API 요청 속도

![개선 전 API 응답 시간 모니터링](http://api.yeo-li.com/api/v1/images/a956faa3515140a9afe9cc14204be5f7.png)

### 개선 후 API 요청 속도

![개선 후 API 응답 시간 모니터링](http://api.yeo-li.com/api/v1/images/154460e742b04cc0a3d849b84c2a375c.png)

### 배포 전후 모니터링 결과

![배포 전후 API 응답 시간 비교](http://api.yeo-li.com/api/v1/images/79b9bbe7e7474380bf422dd336a88d05.png)

실제 운영 지표에서도 뚜렷한 변화가 나타났습니다.

그래프를 보면 개선 전 구간에 비해 개선 후 구간에서 응답 시간 지표가 전반적으로 낮은 수준으로 형성된 것을 확인할 수 있습니다.

개선 이후에도 일부 구간에서 응답 시간이 상승하고 있지만, 개선 전과 비교하면 전체적으로 낮은 수준을 유지하고 있습니다.

추가로 배포 전후의 메트릭을 이용해 평균과 p95 값을 비교했습니다. 2장에서 본 수치(평균 453ms, p95 1.62초)와는 집계 구간이 달라 개선 전 값이 조금 다릅니다.

| 지표 | 개선 전 | 개선 후 | 개선량 | 감소율 |
|---|---:|---:|---:|---:|
| **평균 응답 시간** | 449ms | 45.2ms | **403.8ms 감소** | **89.9% 감소** |
| **p95 응답 시간** | 1.51초 | 94.3ms | **1,415.7ms 감소** | **93.8% 감소** |

측정값을 비교하면 평균 응답 시간은 449ms에서 45.2ms로 약 89.9% 감소했고, p95 응답 시간은 1.51초에서 94.3ms로 약 93.8% 감소했습니다.

특히 처음 문제를 발견하는 계기가 되었던 p95가 크게 감소했습니다.

개선 전에는 평균 응답 시간이 449ms인 상황에서 p95가 1.51초였지만, 개선 이후에는 평균 45.2ms, p95 94.3ms로 감소했습니다.

또한 평균과 p95의 차이도 함께 줄었습니다. 개선 전에는 평균 449ms와 p95 1.51초 사이에 약 3.4배의 차이가 있었지만, 개선 이후에는 평균 45.2ms와 p95 94.3ms로 약 2.1배의 차이를 보였습니다.

평균적인 요청뿐만 아니라 상대적으로 느리게 처리되던 요청의 응답 시간도 함께 감소했음을 확인할 수 있었습니다.

성능 개선 폭은 로컬보다 운영 환경에서 더 컸습니다.

평균 응답 시간 기준으로 로컬은 80.3%, 운영은 89.9% 감소했습니다. 이를 통해 반복 조회를 제거한 개선이 실제 운영 환경에서도 효과가 있음을 확인할 수 있었습니다.

---

# 6. 이번 개선에서 확인한 한계

이번 개선을 통해 반복 조회 문제는 해결했지만, 분석과 검증 과정에서 아쉬운 부분도 있었습니다.

## 6.1 느린 요청의 직접적인 원인을 확인하지 못했습니다

Trace를 통해 반복 조회가 발생한다는 사실은 확인했지만, 일부 요청에서 개별 작업의 소요 시간이 증가한 직접적인 이유까지 확인하지는 못했습니다.

CPU 사용률이나 GC 일시 정지 시간, 커넥션 풀 대기 시간 등의 데이터를 함께 확인할 수 있었다면 원인을 더 자세히 분석할 수 있었을 것입니다.

이번에는 해당 시점의 데이터가 남아 있지 않아 추가 분석을 진행하지 못했습니다.

따라서 앞으로는 JVM과 커넥션 풀 관련 지표도 함께 수집하고, 분석에 필요한 기간만큼 보관할 필요가 있다고 생각했습니다.

## 6.2 동시 요청 상황은 검증하지 못했습니다

로컬 성능 테스트는 VU 1로 진행했습니다.

따라서 여러 사용자가 동시에 요청하는 상황에서의 커넥션 풀 경합이나 DB 부하까지 검증하지는 못했습니다.

반복 조회가 제거되면서 개별 요청에서 발생하는 DB 작업은 줄었지만, 동시 요청 상황에서 어느 정도의 성능 개선이 나타나는지는 별도의 부하 테스트가 필요합니다.

## 6.3 게시물 수가 증가하면 응답 크기는 계속 커집니다

현재 `GET /api/v1/posts`는 페이지네이션 없이 게시물 목록을 반환하고 있으며, 응답에는 게시물 본문(`content`)까지 포함됩니다.

이번 개선으로 반복 쿼리 호출 횟수는 줄었지만, 게시물이 증가할수록 조회하는 데이터의 양과 응답 크기는 계속 커집니다.

앞으로는 페이지네이션을 적용하거나 목록 조회 응답에서 본문을 제외하는 방법도 검토할 예정입니다.

---

# 마치며

이전까지의 성능 개선은 주로 **코드의 정적 분석**에서 시작했습니다.

하지만 그 방식으로는 개선이 정말 필요한 문제였는지, 운영 환경에서도 효과가 있었는지 설명하기 어려웠습니다.

그래서 이번엔 **운영 환경에서 수집한 데이터를 출발점으로 성능 문제를 분석했습니다.**

Metrics에서 평균과 p95 응답 시간의 차이를 발견했고, Trace를 통해 게시물 수에 비례하는 반복 조회를 확인했습니다. 이후 코드를 개선하고 로컬과 운영 환경에서 성능 변화를 검증했습니다.

특히 Trace를 통해 Repository 메서드의 반복 호출뿐 아니라, 코드만 봐서는 쉽게 발견하기 어려웠던 카테고리 지연 로딩까지 확인할 수 있었습니다.

단순히 코드에서 문제처럼 보이는 부분을 개선한 것이 아니라, **실제 서비스에서 발생한 문제를 데이터로 발견하고 원인을 설명한 뒤, 개선 효과까지 확인한 점**에서 이전보다 더 의미 있는 성능 개선이었다고 생각합니다.

동시에 모니터링을 구축했다고 해서 모든 문제를 분석할 수 있는 것은 아니라는 점도 알게 되었습니다. 이번에는 반복 조회라는 구조적인 원인은 발견했지만, 일부 요청이 유독 느려진 직접적인 이유까지는 확인하지 못했습니다.

앞으로는 답하지 못했던 질문을 해결할 수 있도록 수집하는 데이터를 보완해 나가려고 합니다.

다음에는 지금 수집하고 있는 모니터링 데이터가 충분히 신뢰할 수 있는지, 이번 개선으로 인해 다른 병목이나 문제가 생기지는 않았는지도 확인해보려고 합니다.