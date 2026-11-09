# HTTP 200이어도 실패하는 테스트: REST Docs로 문서 누락 잡기

책췍 도서 검색 API에 Spring REST Docs를 적용한 경험

남효윤 루드비코

## 1 REST Docs 도입 배경

저는 책 감상을 기록하고 공유하는 서비스 ‘책췍’에 REST Docs를 적용했습니다. 컨트롤러에는 요청 처리 코드를 두고, API 문서 설명은 분리하고 싶었습니다. 또 응답 필드를 추가하거나 바꾸면서 문서화를 빠뜨리면 테스트에서 발견하고 싶었습니다.

도서 검색 API에는 이미 `MockMvc` 테스트가 있었습니다. `MockMvc`는 실제 서버를 띄우지 않고 Spring MVC의 요청 처리와 응답을 테스트하는 도구입니다.

REST Docs를 이 테스트에 연결하면 문서 설명을 테스트에 작성하고, 테스트 응답과 문서화한 필드 목록을 대조할 수 있었습니다. 그래서 기존 `MockMvc` 테스트에 REST Docs를 적용하기로 했습니다.[^1][^2]

문서화를 추가하면서 테스트 코드가 늘었고, 배포 과정도 달라졌습니다. CD에서 테스트를 생략했다가 문서 조각인 스니펫이 없어 빌드가 실패하기도 했습니다.

이 글에서는 도서 검색 API의 문서 생성부터 Swagger UI 배포까지 살펴봅니다. 응답 필드의 문서화 누락을 잡는 과정과 검증의 한계, 문서 생성을 배포에 연결하며 겪은 문제를 다룹니다. Spring MVC와 `MockMvc` 테스트, Swagger를 이용한 문서화 경험이 있는 개발자를 대상으로 합니다.

## 2 컨트롤러의 요청 처리 코드와 문서화 정보

도서 검색은 검색어와 페이지 번호를 받아 결과를 반환합니다. 다음은 문서화 이후에도 유지된 검색 메서드입니다.[^1]

```java
@GetMapping("/api/v1/books")
public ResponseEntity<BookSearchResponse> search(
        @RequestParam @NotBlank String query,
        @RequestParam @Positive int page
) {
    return ResponseEntity.ok(bookSearchService.search(query, page));
}
```

컨트롤러에는 요청 처리와 입력값 검증을 두고, 요청과 응답의 설명은 테스트에 작성했습니다.

`springdoc-openapi`는 Spring 코드와 애노테이션을 바탕으로 OpenAPI 명세를 만들고, Swagger UI는 그 명세를 화면에 보여 줍니다. 책췍에서는 테스트 결과를 바탕으로 만든 명세를 Swagger UI로 제공했습니다.[^2][^5]

## 3 웹 계층 테스트 구성

초기 도입 커밋의 테스트 구성은 다음과 같습니다.[^1]

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

웹 계층만 구성하고, 검색 서비스는 Mockito 대역으로 바꿉니다. 대역이 반환할 값을 정한 뒤 컨트롤러의 요청 처리와 응답 직렬화 결과를 확인합니다. 외부 도서 API 연동이나 DB 조회의 정확성은 별도 통합 테스트에서 확인해야 합니다.

`@AutoConfigureRestDocs`는 이 테스트의 요청과 응답을 문서화할 수 있도록 `MockMvc`와 REST Docs를 연결합니다.[^4]

## 4 요청과 응답 문서화, 스니펫 생성

기존 검색 성공 테스트는 검색 서비스의 반환값을 준비하고, `query=마션`과 `page=1`로 요청한 뒤 상태 코드와 JSON 값을 확인합니다. REST Docs를 도입할 때는 기존 검증을 유지한 채 마지막에 `document()`를 연결해 스니펫을 생성했습니다. 아래는 추가한 부분을 중심으로 줄인 코드입니다.[^1]

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

`queryParameters()`에는 쿼리 파라미터 설명을, `responseFields()`에는 응답 본문의 필드 설명을 전달합니다. `BOOK_SEARCH_RESPONSE_FIELDS`는 다음과 같은 `FieldDescriptor` 배열입니다. 아래에는 일부만 발췌했으며, 실제 테스트에는 11개 필드를 모두 정의했습니다.

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

`items[].title`은 배열 안의 도서 제목을 가리킵니다. `jsonPath()`는 준비한 값이 응답에 담겼는지 확인하고, 필드 문서화는 실행된 응답과 문서화한 필드 목록을 대조합니다. 설명 문구는 개발자가 직접 작성합니다.

`backend` 디렉터리에서 `./gradlew test --tests '*BookControllerTest'`를 실행하면 `build/generated-snippets/book-search` 아래에 스니펫이 생성됩니다.[^1]

아래는 같은 응답 구조를 사용한 재현 예제에서 생성한 `response-fields.adoc`의 일부입니다.

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

## 5 필드 문서화의 검증 범위와 한계

책췍의 검색 응답을 고정 JSON으로 반환하는 별도 예제에서 응답은 그대로 두고 문서화 정의만 바꿔 검증 범위를 확인했습니다. 코드와 실행 환경은 [부록](#부록)에 정리했습니다.

| 실행한 변경 | 결과 | 확인한 범위 |
| --- | --- | --- |
| 11개 필드를 모두 문서화 | 통과 | 기준 응답을 문서화할 수 있음 |
| `items[].publisher`의 필드 정의 제거 | 실패 | 응답에 남은 미문서화 필드를 발견 |
| `title` 설명을 ‘도서 가격’으로 변경 | 통과 | 설명 문장의 의미는 검증하지 않음 |

`publisher`의 필드 정의를 제외한 두 번째 사례에서는 상태 코드 200 검증을 통과했지만, 문서를 생성하는 단계에서 아래 `SnippetException`이 발생했습니다. 실패 메시지는 문서화되지 않은 필드가 어디인지 보여 줬습니다.

```text
SnippetException:
The following parts of the payload were not documented:
{
  "items" : [ {
    "publisher" : "알에이치코리아(RHK)"
  } ]
}
```

`title`의 설명을 ‘도서 가격’으로 바꿔도 문서 생성은 성공했습니다. REST Docs는 설명 문장의 뜻까지 검증하지 않으므로, 설명 문구는 필드 정의와 함께 리뷰해야 합니다.

기본 `responseFields()`는 문서에 없는 응답 필드나 `optional`로 표시하지 않은 필수 필드의 누락을 실패로 처리합니다. `relaxedResponseFields()`를 사용하면 미문서화 필드가 있어도 허용합니다. `optional()`은 해당 필드가 없는 경우를 허용하므로, ‘키가 항상 존재하며 값만 `null`일 수 있다’는 계약을 이것만으로 보장할 수는 없습니다.[^6]

초기 도입 테스트의 `nextPage`는 숫자 2였고, 뒤의 OpenAPI 연결 커밋에서는 `null` 사례와 `optional()`이 추가됐습니다. 문서에 ‘마지막 페이지라면 `null`’이라고 적는 것과 그 상황을 실제로 테스트하는 것은 별개입니다. 마지막 페이지와 다음 페이지가 있는 경우를 각각 준비하고, 필요한 키 존재 여부와 값 조건도 별도로 검증하는 편이 좋습니다.[^1][^2]

## 6 스니펫의 OpenAPI 변환과 Swagger UI 배포

스니펫 파일만 생성한 상태에서는 클라이언트 개발자가 읽을 문서 화면이 아직 없습니다. 책췍에서는 ePages의 `restdocs-api-spec`을 사용해 스니펫의 정보를 OpenAPI 3.0.1 명세로 모으고, 그 명세를 Swagger UI에 연결했습니다. Spring REST Docs의 스니펫 생성과 OpenAPI 변환은 서로 다른 단계입니다.[^2][^7]

[PR #23](https://github.com/woowacourse-teams/2026-chaekchaek/pull/23)에서는 `document()`의 `import`를 ePages의 `MockMvcRestDocumentationWrapper`로 바꾸고, `resource()`에 요약과 태그 등의 정보를 추가했습니다. 응답 필드 배열을 재사용하면서 기존 `responseFields()`도 유지했습니다. 아래는 해당 `resource()` 부분의 핵심만 발췌한 코드입니다.

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

생성된 스니펫을 `openapi3` 태스크로 모아 `build/api-spec/openapi3.yaml`을 만들고, Swagger UI와 함께 JAR에 포함했습니다. 문서 생성부터 JAR 패키징까지의 흐름은 다음과 같습니다.[^2]

| 단계 | 입력 또는 역할 | 다음 단계에 넘기는 것 |
| --- | --- | --- |
| `test` | 문서화 테스트 실행 | `generated-snippets` |
| `openapi3` | 스니펫 정보 집계 | `openapi3.yaml` |
| `verifyOpenApi` | 생성 명세에 도서 검색 경로와 응답 정보가 포함됐는지 확인 | 검증을 통과한 명세 |
| `stageOpenApiSpec` | 명세를 정적 리소스 경로에 복사 | `static/docs/openapi3.yaml` |
| `bootJar` | UI와 명세를 실행 JAR에 포함 | 배포할 애플리케이션 |

이 프로젝트에서 `bootJar`를 실행하면 테스트와 명세 생성, 검증을 거친 뒤 JAR를 만듭니다.[^9]

`backend` 디렉터리에서 `./gradlew clean test verifyOpenApi`로 테스트와 명세 검증을 수행할 수 있습니다. 문서를 포함한 JAR를 만들 때는 `./gradlew clean bootJar`를 실행합니다. 이 JAR를 배포하면 `/docs/swagger-ui.html`에서 문서를 읽을 수 있습니다.

## 7 문서화 코드 증가와 배포 시 테스트 재실행

REST Docs를 적용하면서 컨트롤러는 그대로였지만, 테스트에는 요청과 응답을 설명하는 코드가 추가됐습니다.[^1][^2]

| 측정 대상 | 도입 전 | REST Docs 추가 | OpenAPI 연결 |
| --- | --- | --- | --- |
| 검색 성공 테스트 메서드 | 38줄 | 46줄 | 60줄 |
| 검색 컨트롤러 파일 전체 | 26줄 | 26줄 | 26줄 |

응답 필드를 추가할 때마다 테스트 데이터, 값 검증, 필드 설명을 함께 살펴야 합니다.

### 스니펫 누락에 따른 CD 빌드 실패와 수정

CD에서 `bootJar`를 만들다가 `openapi3` 단계가 실패한 적도 있습니다.[^3] 당시 CI와 CD는 서로 다른 실행 환경에서 동작했습니다. CI에서 업로드한 파일은 테스트 보고서였고, CD에는 스니펫이나 JAR를 내려받는 단계가 없었습니다.[^10]

그런데 CD에서는 CI에서 테스트를 마쳤다는 이유로 `-x test` 옵션을 사용했습니다. 스니펫을 전달받지도, 테스트로 새로 만들지도 않은 상태라 명세 생성에 필요한 입력이 없었습니다.

```bash
# 변경 전
./gradlew clean bootJar -x test

# PR #62에서 변경한 명령
./gradlew clean bootJar --no-daemon
```

[PR #62](https://github.com/woowacourse-teams/2026-chaekchaek/pull/62)에서 CD도 테스트를 실행해 스니펫을 생성하도록 바꾸면서 빌드 실패를 해결했습니다. 다만 CI에서 통과한 테스트를 CD에서 다시 실행하게 됐습니다.

## 8 적용 결과와 남은 배포 과제

응답에 문서화하지 않은 필드가 있으면 테스트에서 발견할 수 있게 됐습니다. 다만 설명의 뜻은 여전히 사람이 검토해야 합니다.

배포에서는 스니펫 누락을 해결하려고 CD에서도 테스트를 실행하면서 중복 실행이 생겼습니다. 이를 줄이기 위해 CI에서 문서를 포함한 JAR를 만들고 검증한 뒤, 같은 산출물을 CD에서 배포하는 방식을 검토할 수 있습니다.

## 부록

### 도서 검색 API의 REST Docs 도입 이력

| 단계 | 커밋 | 변경 내용 |
| --- | --- | --- |
| 도입 직전 | [6466101](https://github.com/woowacourse-teams/2026-chaekchaek/commit/6466101) | 도서 검색 `MockMvc` 테스트 존재 |
| REST Docs 추가 | [b351d7d](https://github.com/woowacourse-teams/2026-chaekchaek/commit/b351d7d) | 기존 테스트에 요청과 응답 설명 추가 |
| UI 연결 완료 | [c3397bc](https://github.com/woowacourse-teams/2026-chaekchaek/commit/c3397bc) | OpenAPI 생성과 Swagger UI 제공 |

### 프로젝트 설정과 실행 환경

본문의 프로젝트 예시는 Java 21, Spring Boot 4.1.0 기준입니다. 전체 테스트와 Gradle 설정은 REST Docs 도입 커밋과 OpenAPI 연결 커밋에서 확인할 수 있습니다.[^1][^2]

OpenAPI 변환에는 `com.epages.restdocs-api-spec` 플러그인과 테스트용 `restdocs-api-spec-mockmvc` 의존성 0.20.1을 사용했습니다. 생성 명세는 OpenAPI 3.0.1 형식입니다.[^2][^7]

### 재현 코드와 실행 환경

재현 예제는 Java 21, Gradle 9.5.1, Spring Framework 7.0.8, Spring REST Docs 4.0.1에서 실행했습니다.

<details>
<summary>5장의 독립 재현 코드와 실행 결과</summary>

고정 JSON을 반환하는 `MockMvc` 컨트롤러에 문서화를 연결한 예제입니다. `main()`에서 세 사례를 실행하고, `work/probe-snippets`에 스니펫을 생성합니다.

Java 21과 Gradle 9.5.1이 설치된 환경에서 빈 디렉터리에 다음 `build.gradle`을 만듭니다.

```groovy
plugins {
    id 'application'
}

repositories {
    mavenCentral()
}

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(21)
    }
}

dependencies {
    implementation 'org.springframework.restdocs:spring-restdocs-mockmvc:4.0.1'
}

application {
    mainClass = 'DocsProbe'
}
```

아래 Java 코드를 `src/main/java/DocsProbe.java`에 저장합니다.

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

`build.gradle`이 있는 디렉터리에서 실행합니다.

```bash
gradle run
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

[^9]: [책췍의 Gradle 태스크 설정](https://github.com/woowacourse-teams/2026-chaekchaek/blob/c3397bc116d7e9c6211f6d715adb466b341efa50/backend/build.gradle), [ePages 0.20.1의 공통 태스크 설정](https://github.com/ePages-de/restdocs-api-spec/blob/0.20.1/restdocs-api-spec-gradle-plugin/src/main/kotlin/com/epages/restdocs/apispec/gradle/RestdocsApiSpecPlugin.kt). 프로젝트 설정과 플러그인이 연결한 선행 태스크 참고.

[^10]: PR #62 시점의 [CI 설정](https://github.com/woowacourse-teams/2026-chaekchaek/blob/a20c0f83aa7a5ce0a4c52623c60f3bba39b4c9ac/.github/workflows/backend-ci.yml)과 [CD 설정](https://github.com/woowacourse-teams/2026-chaekchaek/blob/a20c0f83aa7a5ce0a4c52623c60f3bba39b4c9ac/.github/workflows/backend-cd.yml). 별도 실행 환경, 테스트 보고서 업로드, CD의 별도 빌드 구성 참고.


