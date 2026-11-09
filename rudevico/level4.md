# HTTP 200이어도 실패하는 테스트: REST Docs로 문서 누락 잡기

책췍 도서 검색 API에 Spring REST Docs를 적용한 경험

남효윤 루드비코

## 1 REST Docs 도입 배경과 비교 범위

저는 책 감상을 기록하고 공유하는 서비스 ‘책췍’에 REST Docs를 적용했어요. 당시 도서 검색 API에는 요청과 응답을 확인하는 `MockMvc` 테스트가 있었어요. 컨트롤러에 문서 설명을 추가하지 않고, 기존 테스트가 주고받는 요청과 응답에 설명을 붙이기로 했어요.

문서화를 추가하면서 검색 성공 테스트는 38줄에서 46줄로, OpenAPI 명세까지 연결한 뒤에는 60줄로 늘었어요. 이후에는 CD에서 테스트를 생략했다가 문서 생성에 필요한 스니펫이 없어 빌드가 실패하기도 했어요. 문서화를 테스트에 연결하는 선택은 테스트 코드뿐 아니라 배포 과정에도 영향을 줬어요.

이 글에서는 초기 도서 검색 API를 따라 문서 생성부터 Swagger UI 배포까지 살펴볼 거예요. 응답 필드의 문서화 정의를 빠뜨리면 실패하지만 설명의 뜻이 틀려도 통과하는 이유는 별도 재현 예제로, 배포 빌드에 생긴 변화는 실제 수정 기록으로 확인할게요. Spring MVC, `MockMvc` 테스트, Swagger를 이용한 문서화 경험이 있는 개발자를 대상으로 해요.

### 도서 검색 API의 REST Docs 도입 이력

| 단계 | 커밋 | 변경 내용 |
| --- | --- | --- |
| 도입 직전 | [6466101](https://github.com/woowacourse-teams/2026-chaekchaek/commit/6466101) | 도서 검색 `MockMvc` 테스트 존재 |
| REST Docs 추가 | [b351d7d](https://github.com/woowacourse-teams/2026-chaekchaek/commit/b351d7d) | 기존 테스트에 요청과 응답 설명 추가 |
| UI 연결 완료 | [c3397bc](https://github.com/woowacourse-teams/2026-chaekchaek/commit/c3397bc) | OpenAPI 생성과 Swagger UI 제공 |

첫 두 커밋 사이에서는 `BookControllerTest.java`만 바뀌었어요. 검색 기능은 그대로 둔 채 문서화를 추가한 비교예요. 이 API는 처음부터 Swagger 애노테이션 없이 구현됐고, 도입 과정에서도 컨트롤러는 그대로였어요. 따라서 이 글에서 비교하는 것은 기존 테스트에 문서화를 연결하기 전후예요. Swagger 애노테이션을 사용하던 코드를 REST Docs로 전환한 사례는 아니에요.[^1][^2]

## 2 컨트롤러의 요청 처리 코드와 문서화 정보

도서 검색은 검색어와 페이지 번호를 받아 결과를 반환해요. 도입 전후에 유지된 실제 검색 메서드는 다음과 같아요.[^1]

```java
@GetMapping("/api/v1/books")
public ResponseEntity<BookSearchResponse> search(
        @RequestParam @NotBlank String query,
        @RequestParam @Positive int page
) {
    return ResponseEntity.ok(bookSearchService.search(query, page));
}
```

요청 매핑, 입력값 검증, 서비스 호출을 연달아 읽을 수 있어요. `@NotBlank`와 `@Positive`는 요청을 검증하므로 컨트롤러에 남아요. 검색어의 뜻이나 응답 필드의 의미를 설명하는 문서화 정보는 테스트에 추가했어요.

Swagger라는 이름으로 묶어 부르던 도구도 구분할 필요가 있어요. `springdoc-openapi`는 Spring 코드와 애노테이션을 바탕으로 OpenAPI 명세를 만들고, Swagger UI는 그 명세를 화면에 보여 줘요. REST Docs를 사용하더라도 생성한 명세를 Swagger UI에 연결할 수 있어요. 책췍에서는 테스트 실행 결과를 바탕으로 명세를 만들고, 그 명세를 Swagger UI로 제공했어요. Swagger UI를 사용하는 독자라면 문서를 읽는 화면보다, 명세를 만드는 과정에 응답 필드 검증을 연결한다는 차이에 주목하면 좋아요.[^2][^5]

## 3 REST Docs 선택 근거와 웹 계층 테스트 구성

책췍에는 이미 도서 검색 컨트롤러를 확인하는 `MockMvc` 테스트가 있었어요. 그래서 별도 요청 사례를 처음부터 관리하기보다, 이 테스트가 주고받은 요청과 응답에 설명을 붙이는 방식을 선택했어요.

Apidog나 Postman에서 문서와 요청을 별도로 관리하는 방식도 있지만, 그 경우에는 서버 변경과 문서 갱신을 연결할 절차를 정해야 해요. 책췍에서는 기존 테스트 실행에 필드 문서화를 연결할 수 있다는 점에 집중했어요. 우아한형제들의 2018년 REST Docs 적용 글 역시 코드와 별도로 관리하던 wiki 문서가 어긋난 경험을 다뤄요.[^8]

초기 도입 커밋의 테스트 구성은 다음과 같아요.[^1]

```java
@WebMvcTest(BookController.class)
@AutoConfigureRestDocs
class BookControllerTest {
    @Autowired
    private MockMvc mockMvc;

    @MockitoBean
    private BookSearchService bookSearchService;
}
```

`@WebMvcTest`로 웹 계층을 구성하고, 검색 서비스는 Mockito 대역으로 바꿔요. 테스트는 HTTP 요청 매핑, 응답 직렬화와 필드 설명에 집중해요. 대역이 반환할 값을 정한 뒤 컨트롤러가 내보내는 결과를 확인하는 범위예요. 외부 도서 API 연동이나 DB 조회의 정확성은 별도 통합 테스트에서 확인해야 해요.

`@AutoConfigureRestDocs`는 `MockMvc`와 REST Docs를 연결해요. 책췍은 슬라이스 테스트를 사용했지만, 전체 컨텍스트를 사용하는 테스트에 연결할 수도 있어요. 테스트 범위를 먼저 정하고 그 결과를 문서화해야, 생성된 문서가 무엇을 근거로 하는지도 분명해져요.[^4]

## 4 요청과 응답 문서화, 스니펫 생성

기존 검색 성공 테스트는 검색 서비스의 반환값을 준비하고, `query=마션`과 `page=1`로 요청한 뒤 상태 코드와 JSON 값을 확인해요. REST Docs를 도입하면서 이 검증을 지우지 않았어요. 마지막에 `document()`를 연결해 문서 조각인 스니펫을 생성했어요. 아래는 추가한 부분을 중심으로 줄인 코드예요.[^1]

```java
mockMvc.perform(get("/api/v1/books")
        .param("query", "마션").param("page", "1"))
    .andExpect(status().isOk())
    .andExpect(jsonPath("$.totalCount").value(11))
    .andExpect(jsonPath("$.nextPage").value(2))
    // 실제 테스트의 나머지 응답 값 검증은 생략
    .andDo(document("book-search",
        queryParameters(
            parameterWithName("query").description("검색할 도서명"),
            parameterWithName("page").description("1부터 시작하는 페이지 번호")
        ),
        responseFields(BOOK_SEARCH_RESPONSE_FIELDS)
    ));
```

`queryParameters()`에는 쿼리 파라미터 설명을, `responseFields()`에는 응답 본문의 필드 설명을 전달해요. `BOOK_SEARCH_RESPONSE_FIELDS`는 다음과 같은 `FieldDescriptor` 배열이에요. 아래에는 일부만 발췌했으며, 실제 테스트에는 11개 필드를 모두 정의했어요.

```java
private static final FieldDescriptor[] BOOK_SEARCH_RESPONSE_FIELDS = {
    fieldWithPath("totalCount").type(JsonFieldType.NUMBER)
        .description("검색 결과의 전체 도서 수"),
    fieldWithPath("nextPage").type(JsonFieldType.NUMBER)
        .description("다음 페이지 번호. 마지막 페이지라면 null"),
    fieldWithPath("items").type(JsonFieldType.ARRAY)
        .description("검색된 도서 목록"),
    fieldWithPath("items[].title").type(JsonFieldType.STRING)
        .description("도서 제목")
    // 표지, 저자, 옮긴이, 출판일, ISBN, 분류, 출판사도 기술
};
```

`items[].title`은 배열 안의 도서 제목을 가리켜요. `jsonPath()`의 값 검증과 필드 문서화는 역할이 달라요. 앞의 검증은 준비한 값이 응답에 담겼는지 확인하고, 뒤의 문서화는 실행된 응답과 필드 목록을 대조해요. 설명 문구는 개발자가 직접 작성해요.

해당 커밋에서 `backend` 디렉터리로 이동해 `./gradlew test --tests '*BookControllerTest'`를 실행하면, 기본 출력 위치인 `build/generated-snippets/book-search` 아래에 요청, 응답, 쿼리 파라미터와 응답 필드 스니펫을 생성하는 구성이에요. 전체 클래스와 당시 Gradle 설정은 참고 자료의 고정 커밋에서 확인할 수 있어요. 예시의 패키지와 의존성은 Java 21, Spring Boot 4.1.0 기반 코드이므로 다른 버전에 옮길 때는 그대로 복사하기보다 버전에 맞춰 확인해야 해요.[^1][^2]

## 5 필드 문서화의 검증 범위와 한계

문서화 테스트가 통과하면 어디까지 믿을 수 있을까요? 이를 구분하려고 책췍의 초기 검색 응답 구조와 11개 필드 설명을 사용한 별도 재현 예제를 실행했어요. 고정 JSON을 반환하는 `MockMvc` 독립 컨트롤러에 문서화를 연결했으며, 운영 서버나 실제 검색 서비스를 호출한 결과는 아니에요. 기준 사례에서는 11개 필드를 모두 정의했어요. 두 번째 사례는 `fieldWithPath("items[].publisher")`로 시작하는 정의를 제외했고, 세 번째는 목록을 복원한 뒤 `title`의 `.description("도서 제목")`만 `.description("도서 가격")`으로 바꿨어요. 각 사례는 같은 고정 응답을 별도로 문서화했어요.

실행 환경은 Java 21, Spring Framework 7.0.8, Spring REST Docs 4.0.1이에요. [부록의 재현 코드](#부록)에서 고정 응답과 세 가지 변경을 확인할 수 있어요. 상태 코드 200을 확인한 뒤 `queryParameters()`와 `responseFields()`를 연결하고, 동일한 응답에 대해 필드 정의를 제거하거나 설명 문구를 바꿨을 때의 결과를 비교했어요.

기준 사례에서 생성된 `response-fields.adoc`의 일부예요. 필드 정의에 적은 경로, 타입, 설명이 문서의 각 열로 이어져요.

```asciidoc
|===
|Path|Type|Description

|`+items[].title+`
|`+String+`
|도서 제목

|`+items[].publisher+`
|`+String+`
|출판사
|===
```

같은 응답에 대해 필드 정의를 빼거나 설명을 바꾸면 어떻게 될까요?

| 실행한 변경 | 결과 | 확인한 범위 |
| --- | --- | --- |
| 11개 필드를 모두 문서화 | 통과 | 기준 응답을 문서화할 수 있음 |
| `items[].publisher`의 필드 정의 제거 | 실패 | 응답에 남은 미문서화 필드를 발견 |
| `title` 설명을 ‘도서 가격’으로 변경 | 통과 | 설명 문장의 의미는 검증하지 않음 |

`publisher`의 필드 정의를 제외한 두 번째 사례에서는 상태 코드 200 검증을 통과했지만, 문서를 생성하는 단계에서 아래 `SnippetException`이 발생했어요. 실패 메시지는 문서화되지 않은 필드가 어디인지 보여 줬어요.

```text
SnippetException:
The following parts of the payload were not documented:
{
  "items" : [ {
    "publisher" : "알에이치코리아(RHK)"
  } ]
}
```

세 번째 경우는 달랐어요. `title`의 설명을 ‘도서 가격’으로 바꿔도 문서 생성은 성공했어요. 필드 경로와 타입이 맞는다고 설명의 뜻까지 맞는 것은 아니었어요. 따라서 문서화 테스트의 통과 여부와 설명 문장의 정확성은 나눠 확인해야 해요. 설명 문구는 필드 정의와 함께 리뷰하고, 실제 검색 결과와 도서 수 계산은 해당 로직의 테스트로 확인해야 해요.

설정도 검증 범위를 바꿔요. 기본 `responseFields()`는 문서에 없는 응답 필드나 `optional`로 표시하지 않은 필수 필드의 누락을 실패로 처리해요. `relaxedResponseFields()`를 사용하면 미문서화 필드가 있어도 허용해요. `optional()`은 해당 필드가 없는 경우를 허용하므로, ‘키가 항상 존재하며 값만 `null`일 수 있다’는 계약을 이것만으로 보장할 수는 없어요.[^6]

초기 도입 테스트의 `nextPage`는 숫자 2였고, 뒤의 OpenAPI 연결 커밋에서는 `null` 사례와 `optional()`이 추가됐어요. 문서에 ‘마지막 페이지라면 `null`’이라고 적는 것과 그 상황을 실제로 테스트하는 것은 별개예요. 마지막 페이지와 다음 페이지가 있는 경우를 각각 준비하고, 필요한 키 존재 여부와 값 조건도 별도로 검증하는 편이 좋아요.[^1][^2]

## 6 스니펫의 OpenAPI 변환과 Swagger UI 배포

스니펫 파일만 생성하면 클라이언트 개발자가 읽을 문서 화면은 아직 없어요. 책췍에서는 ePages의 `restdocs-api-spec`을 사용해 스니펫의 정보를 OpenAPI 3.0.1 명세로 모으고, 그 명세를 Swagger UI에 연결했어요. Spring REST Docs의 스니펫 생성과 OpenAPI 변환은 서로 다른 단계예요.[^2][^7]

[PR #23](https://github.com/woowacourse-teams/2026-chaekchaek/pull/23)에서는 `document()`의 `import`를 ePages의 `MockMvcRestDocumentationWrapper`로 바꾸고, `resource()`에 요약과 태그 등의 정보를 추가했어요. 응답 필드 배열을 재사용하면서 기존 `responseFields()`도 유지했어요. 아래는 해당 `resource()` 부분의 핵심만 발췌한 코드예요.

```java
import static com.epages.restdocs.apispec
    .MockMvcRestDocumentationWrapper.document;

// document()에 전달하는 스니펫 중 일부
resource(ResourceSnippetParameters.builder()
    .summary("도서 검색")
    .description("도서명과 페이지 번호로 도서를 검색한다")
    .tag("도서")
    // queryParameters 설정은 생략
    .responseFields(BOOK_SEARCH_RESPONSE_FIELDS)
    .build())
```

Gradle에는 `com.epages.restdocs-api-spec` 플러그인과 테스트용 `restdocs-api-spec-mockmvc` 의존성 0.20.1을 추가했어요. `openapi3` 설정에는 문서 이름과 출력 형식을 지정했고, 생성된 YAML은 `build/api-spec/openapi3.yaml`에 두었어요. 각 태스크가 만드는 파일과 다음 단계에서 사용하는 입력은 다음과 같아요.

| 단계 | 입력 또는 역할 | 다음 단계에 넘기는 것 |
| --- | --- | --- |
| `test` | 문서화 테스트 실행 | `generated-snippets` |
| `openapi3` | 스니펫 정보 집계 | `openapi3.yaml` |
| `verifyOpenApi` | 생성 명세의 경로와 응답 등 확인 | 검증을 통과한 명세 |
| `stageOpenApiSpec` | 명세를 정적 리소스 경로에 복사 | `static/docs/openapi3.yaml` |
| `bootJar` | UI와 명세를 실행 JAR에 포함 | 배포할 애플리케이션 |

Gradle의 태스크는 빌드 작업의 단위예요. 다른 태스크에 의존하도록 설정하면 그 작업을 먼저 마친 뒤 실행해요. 이 구성에서 `bootJar`는 `stageOpenApiSpec`에, `stageOpenApiSpec`은 `verifyOpenApi`에, `verifyOpenApi`는 `openapi3`에 의존해요. ePages 플러그인은 `openapi3`가 `check`에 의존하도록 설정하고, Java 플러그인의 `check`에는 `test`가 연결돼 있어요. 그래서 `bootJar`를 호출해도 테스트와 명세 생성, 검증이 먼저 실행돼요.[^9]

이 구성에서는 명세를 생성하는 것뿐 아니라, 생성 결과에 검색 경로와 200, 400, 500, 502 응답이 들어 있는지도 `verifyOpenApi`로 확인했어요. 오류 응답이 `application/problem+json`과 `ProblemDetail` 스키마를 사용하도록 검사한 것도 프로젝트에서 추가한 검증이에요. REST Docs를 도입하면 저절로 생기는 검증은 아니에요.[^2]

`backend` 디렉터리에서 `./gradlew clean test verifyOpenApi`를 실행하면 테스트와 명세 생성, 검증까지 확인할 수 있어요. 이 명령에는 JAR를 만드는 작업은 포함되지 않아요. 문서를 포함한 배포용 JAR까지 만들려면 `./gradlew clean bootJar`를 실행해요. 앞서 연결한 태스크들이 함께 실행되므로 테스트부터 다시 확인해요. 이 JAR를 배포하면 `/docs/swagger-ui.html`에서 문서를 읽을 수 있어요. 독자가 따라 적용할 때도 테스트 성공, 명세 생성, 최종 배포물 포함 여부를 나눠 확인해야 해요. 문서 화면이 열리는지만 보지 말고, 수정한 필드와 예시가 명세에 반영됐는지도 함께 확인해 보세요.

## 7 문서화 코드 증가와 배포 시 테스트 재실행

REST Docs를 적용하면서 컨트롤러는 그대로였지만, 테스트에는 요청과 응답을 설명하는 코드가 추가됐어요.[^1][^2]

| 측정 대상 | 도입 전 | REST Docs 추가 | OpenAPI 연결 |
| --- | --- | --- | --- |
| 검색 성공 테스트 메서드 | 38줄 | 46줄 | 60줄 |
| 검색 컨트롤러 파일 전체 | 26줄 | 26줄 | 26줄 |

문서화 코드는 한 번 작성하고 끝나지 않아요. 응답 필드를 추가하면 테스트 데이터, 값 검증, 필드 설명을 함께 살펴야 해요. 테스트 응답에 새 필드가 실제로 포함됐는데 설명이 없다면 엄격한 필드 문서화가 실패해요. 설명을 작성할 일은 남지만, 빠뜨렸다는 사실을 테스트 결과에서 확인할 수 있어요. 반면 뜻이 틀린 설명은 앞의 재현처럼 리뷰가 필요해요.

### 스니펫 누락에 따른 CD 빌드 실패와 수정

문서화의 영향은 테스트 파일에서 끝나지 않았어요. [PR #62](https://github.com/woowacourse-teams/2026-chaekchaek/pull/62)에는 CD에서 `bootJar`를 만들다가 `openapi3` 단계가 실패한 기록이 있어요. 당시 CI는 GitHub가 제공하는 실행 환경에서, CD는 팀이 관리하는 별도 실행 환경에서 동작했어요. CI에서 업로드한 파일은 테스트 보고서였고, CD에는 CI가 만든 스니펫이나 JAR를 내려받는 단계가 없었어요.[^10] 그런데 기존 CD 명령은 CI에서 테스트를 했다는 이유로 테스트를 생략했어요. 그 결과 CD 빌드 환경에는 명세를 생성할 스니펫이 없었어요. 테스트 성공 여부와 문서 생성에 필요한 파일의 존재는 서로 다른 조건이었어요.[^3]

```bash
# 변경 전
./gradlew clean bootJar -x test

# PR #62에서 변경한 명령
./gradlew clean bootJar --no-daemon
```

[PR #62](https://github.com/woowacourse-teams/2026-chaekchaek/pull/62)에서는 CD에서도 테스트를 실행해 스니펫을 만들도록 바꿨어요. 테스트가 검증뿐 아니라 문서 생성의 입력을 만드는 단계이기도 했기 때문이에요. CI에서 한 번 통과했어도 그때 만든 파일을 CD로 전달하지 않으면 다음 빌드에서 사용할 수 없어요. 이 수정으로 문서 생성에 필요한 입력을 마련했지만, PR에는 CI와 CD의 중복 테스트 실행이 단점으로 남았어요.

PR #62에서는 CD에서 필요한 파일을 다시 만드는 쪽을 선택했어요. 중복 실행을 줄이려면 CI에서 검증한 배포물을 CD에 전달하는 방법도 검토할 수 있어요.

| 구분 | CD가 사용하는 파일 | 중복 실행과 관리할 일 |
| --- | --- | --- |
| PR #62에서 적용한 방식 | CD에서 테스트와 문서 생성을 다시 실행해 만든 JAR | CI에서 수행한 테스트를 다시 실행해요 |
| 후속 개선으로 검토할 방식 | CI에서 문서를 포함해 만들고 검증한 JAR | JAR를 다시 만들기 위한 테스트 실행을 줄일 수 있지만, 배포할 코드와 일치하는 산출물을 전달해야 해요 |

문서 생성이 빌드에 연결되면, 테스트 성공 여부와 함께 그 실행에서 만든 파일을 다음 단계에서 어떻게 사용할지도 정해야 해요.

## 8 REST Docs 도입 판단과 남은 배포 과제

이미 웹 계층 테스트를 작성하고 있으며 응답 변경 때 문서 갱신을 놓치는 문제가 있다면, 대표 API 하나에 연결해 볼 만해요. 반대로 테스트 기반이 거의 없거나 구현 전 명세 합의가 중심인 팀이라면, 테스트를 새로 마련하는 비용과 문서가 필요한 시점을 먼저 살펴야 해요. REST Docs가 모든 팀에서 가장 싼 문서화 방식인 것은 아니에요.

처음 적용할 때는 응답 구조가 단순하고 정상 응답과 오류 응답을 모두 준비할 수 있는 API를 고르면 좋아요. 기존 값 검증을 유지한 채 필드 설명을 추가하고, 응답에 있는 필드의 문서화 정의 하나를 일부러 제외해 실패하는지 확인해 보세요. 이어서 생성된 명세와 최종 문서 화면에 같은 요청과 응답이 나타나는지 확인하면, 코드 작성부터 배포까지 필요한 작업이 드러나요.

책췍에 REST Docs를 적용하면서 응답 필드의 문서화 정의를 빠뜨리면 테스트에서 발견할 수 있게 됐어요. 설명의 뜻은 여전히 사람이 검토해야 하지만, 응답과 문서의 필드 목록을 대조하는 일은 테스트에 맡길 수 있었어요.

배포 과정에서는 테스트가 문서 생성에 필요한 입력을 만드는 단계이기도 하다는 점을 확인했어요. CD에서도 테스트를 실행해 스니펫 누락을 해결했지만, CI에서 통과한 테스트를 다시 실행하는 비용이 남았어요.

다음 개선 과제는 이 중복 실행을 줄이는 거예요. CI에서 문서를 포함한 JAR를 만들고 검증한 뒤, 그 산출물을 그대로 배포하는 방식을 검토할 수 있어요. 코드와 문서가 같은 변경을 반영하도록 유지하면서, 이미 확인한 결과를 다음 단계에서도 활용하는 방향이에요.

## 부록

<details>
<summary>5장의 독립 재현 코드와 실행 결과</summary>

아래 코드는 글을 작성하며 실행한 별도 예제예요. 책췍의 서비스 구현을 호출하지 않고 고정 JSON을 반환해요. Spring REST Docs의 MockMvc 모듈과 Spring MVC 테스트 의존성을 사용하는 Java 21 프로그램이며, `main()`에서 세 사례를 실행해요. 스니펫은 실행 위치의 `work/probe-snippets`에 생성돼요.

```java
import java.util.*;
import org.springframework.restdocs.ManualRestDocumentation;
import org.springframework.restdocs.payload.*;
import org.springframework.restdocs.snippet.SnippetException;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.setup.MockMvcBuilders;
import org.springframework.web.bind.annotation.*;

import static org.springframework.restdocs.mockmvc.MockMvcRestDocumentation.*;
import static org.springframework.restdocs.payload.PayloadDocumentation.*;
import static org.springframework.restdocs.request.RequestDocumentation.*;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

public class DocsProbe {

    @RestController
    static class SearchController {
        @GetMapping(value="/api/v1/books", produces="application/json")
        String search() {
            return """
            {"totalCount":11,"nextPage":2,"items":[{
              "title":"마션","coverImageUrl":"https://image.aladin.co.kr/martian.jpg",
              "authors":["앤디 위어"],"translators":["박아람"],"publishedDate":"2026-07-01",
              "isbn13":"9788925568683","category":"국내도서>소설>과학소설","publisher":"알에이치코리아(RHK)"
            }]}
            """;
        }
    }

    static List<FieldDescriptor> fields(boolean omitPublisher, boolean wrongDescription) {
        var result = new ArrayList<FieldDescriptor>();
        result.add(fieldWithPath("totalCount").type(JsonFieldType.NUMBER).description("검색 결과의 전체 도서 수"));
        result.add(fieldWithPath("nextPage").type(JsonFieldType.NUMBER).description("다음 페이지 번호. 마지막 페이지라면 null"));
        result.add(fieldWithPath("items").type(JsonFieldType.ARRAY).description("검색된 도서 목록"));
        result.add(fieldWithPath("items[].title").type(JsonFieldType.STRING).description(wrongDescription ? "도서 가격" : "도서 제목"));
        result.add(fieldWithPath("items[].coverImageUrl").type(JsonFieldType.STRING).description("표지 이미지 URL"));
        result.add(fieldWithPath("items[].authors").type(JsonFieldType.ARRAY).description("저자 이름 목록"));
        result.add(fieldWithPath("items[].translators").type(JsonFieldType.ARRAY).description("옮긴이 이름 목록"));
        result.add(fieldWithPath("items[].publishedDate").type(JsonFieldType.STRING).description("출판일"));
        result.add(fieldWithPath("items[].isbn13").type(JsonFieldType.STRING).description("ISBN-13"));
        result.add(fieldWithPath("items[].category").type(JsonFieldType.STRING).description("도서 카테고리"));
        if (!omitPublisher) result.add(fieldWithPath("items[].publisher").type(JsonFieldType.STRING).description("출판사"));
        return result;
    }

    public static void main(String[] args) throws Exception {
        var docs = new ManualRestDocumentation("work/probe-snippets");
        MockMvc mvc = MockMvcBuilders.standaloneSetup(new SearchController())
            .apply(documentationConfiguration(docs)).build();
        String[] names = {"baseline", "missing-publisher", "wrong-description"};
        for (int i=0; i<names.length; i++) {
            docs.beforeTest(DocsProbe.class, names[i]);
            boolean failed = false;
            try {
                mvc.perform(get("/api/v1/books").param("query", "마션").param("page", "1"))
                    .andExpect(status().isOk())
                    .andDo(document(names[i], queryParameters(
                        parameterWithName("query").description("검색할 도서명"),
                        parameterWithName("page").description("1부터 시작하는 페이지 번호")),
                        responseFields(fields(i==1, i==2))));
                System.out.println(names[i] + ": PASS");
            } catch (SnippetException ex) {
                failed = true;
                System.out.println(names[i] + ": FAIL " + ex.getClass().getSimpleName() + " " + ex.getMessage());
            } finally { docs.afterTest(); }
            if (failed != (i==1)) throw new AssertionError("Unexpected result: " + names[i]);
        }
    }
}
```

실행 결과:

```text
baseline: PASS
missing-publisher: FAIL SnippetException The following parts of the payload were not documented:
{
  "items" : [ {
    "publisher" : "알에이치코리아(RHK)"
  } ]
}
wrong-description: PASS
```

</details>

[^1]: [책췍 도서 검색 REST Docs 도입 전후 비교](https://github.com/woowacourse-teams/2026-chaekchaek/compare/64661016078a8973291d4c4d824cae9fd5c9b15d...b351d7d7e59e4490f60c5f5b0163d21ad5eddf8b) 테스트 원문과 추가한 11개 응답 필드 설명. 도입 직후 커밋 b351d7d.

[^2]: [책췍 REST Docs 기반 Swagger 문서 제공 PR 23](https://github.com/woowacourse-teams/2026-chaekchaek/pull/23) 본문 기준 리비전 c3397bc. `backend/build.gradle`과 `BookControllerTest.java` 참고.

[^3]: [책췍 CD에서 테스트 스킵 제거 PR 62](https://github.com/woowacourse-teams/2026-chaekchaek/pull/62) 문서 생성 입력이 없어 배포 빌드가 실패한 사례와 수정 명령.

[^4]: [Spring Creating API Documentation with Restdocs](https://spring.io/guides/gs/testing-restdocs/) 테스트 구성과 스니펫 생성 방식. 사용 중인 버전의 문서와 함께 확인.

[^5]: [springdoc openapi 공식 문서](https://springdoc.org/) Spring 코드와 애노테이션을 바탕으로 OpenAPI 명세를 만드는 방식.

[^6]: [Spring REST Docs Request and Response Payloads](https://docs.spring.io/spring-restdocs/reference/documenting-your-api/request-response-payloads.html) 응답 필드 경로, `optional`과 relaxed 모드의 검증 범위.

[^7]: [ePages restdocs api spec](https://github.com/ePages-de/restdocs-api-spec) REST Docs 스니펫을 OpenAPI로 변환하는 확장 도구. 프로젝트 사용 버전은 0.20.1.

[^8]: [Spring Rest Docs 적용, 우아한형제들 기술블로그](https://techblog.woowahan.com/2597/) 2018-12-28

[^9]: [책췍의 Gradle 태스크 설정](https://github.com/woowacourse-teams/2026-chaekchaek/blob/c3397bc116d7e9c6211f6d715adb466b341efa50/backend/build.gradle), [ePages 0.20.1의 공통 태스크 설정](https://github.com/ePages-de/restdocs-api-spec/blob/0.20.1/restdocs-api-spec-gradle-plugin/src/main/kotlin/com/epages/restdocs/apispec/gradle/RestdocsApiSpecPlugin.kt). 프로젝트 설정과 플러그인이 연결한 선행 태스크 참고.

[^10]: PR #62 시점의 [CI 설정](https://github.com/woowacourse-teams/2026-chaekchaek/blob/a20c0f83aa7a5ce0a4c52623c60f3bba39b4c9ac/.github/workflows/backend-ci.yml)과 [CD 설정](https://github.com/woowacourse-teams/2026-chaekchaek/blob/a20c0f83aa7a5ce0a4c52623c60f3bba39b4c9ac/.github/workflows/backend-cd.yml). 별도 실행 환경, 테스트 보고서 업로드, CD의 별도 빌드 구성 참고.


