# 테스트가 느려!

한창 백엔드 애플리케이션을 개발하던 중 동료가 불만을 제시했습니다.

> *AI가 너무 느려! 쉬운 작업인데 왜 이렇게 오래 걸리는 거야?*

저도 의아했습니다.
설계와 계획은 이미 끝났고, 지금 AI가 하는 작업은 길어도 **30분이면 끝날 단순 구현 작업**이었기 때문입니다.

AI는 총 50분 35초나 걸려서 작업을 마쳤고, 저는 AI에게 "왜 이렇게 작업이 오래 걸린 거야?"를 물어봤습니다.

![image.png](https://techcourse-project-2026.s3.ap-northeast-2.amazonaws.com/rilog/images/originals/52-0243f177-e494-47b0-a6b7-688a9bebedd2.png)

> *전체 작업 50분 35초 중, 테스트 실행에만 23분 11초를 소모했습니다.*

AI의 답변은 충격적이었습니다.
저는 **작업 속도가 느린 원인이 AI의 성능이 아니라 코드 구조에 있다고** 판단했고, 즉시 테스트 코드를 살펴봤습니다.

# 의도는 좋으나 성능은 나쁜 코드

```java
@DataJpaTest
@Import(BookService.class)
class BookServiceTest {
    // ...
}

@DataJpaTest
@Import(CommentService.class)
class CommentServiceTest {
    // ...
}

@DataJpaTest
@Import(ClubService.class)
class ClubServiceTest {
    // ...
}
```

저희의 서비스 테스트는 위처럼 `@DataJpaTest`와 `@Import`를 사용하고 있었습니다.
테스트에 필요한 빈만 최소한으로 등록해, **테스트가 무관한 빈에 의존하지 않게 만들겠다**는 의도였죠.

이때, **`@Import`로 서로 다른 빈을 추가**하는 게 문제였습니다.

# 왜 느렸죠?

![image.png](https://techcourse-project-2026.s3.ap-northeast-2.amazonaws.com/rilog/images/originals/52-1e805fb7-76da-42ba-b3fe-3e8bf8553f06.png)

이 도식은 OpenAI 이미지 생성 도구로 만들었습니다.

결론부터 말하자면, 테스트끼리 `ApplicationContext`를 공유하지 못해서 **여러 개의** `ApplicationContext`**를 생성**해야 했기 때문이었습니다.

아래에서 자세히 알아보겠습니다.

## Context Cache Key

> Once the TestContext framework loads an `ApplicationContext` (or `WebApplicationContext`) for a test, **that context is cached and reused for all subsequent tests that declare the same unique context configuration within the same test suite.**
>
> 테스트에서 TestContext 프레임워크가 `ApplicationContext`(또는 `WebApplicationContext`)를 한 번 로드하면, 해당 컨텍스트는 캐시에 저장되며 **동일한 고유 컨텍스트 설정을 사용하는 이후의 모든 테스트에서 캐시된 컨텍스트를 재사용합니다.**
>
> - [Spring 공식 문서 - 컨텍스트 캐싱](https://docs.spring.io/spring-framework/reference/testing/testcontext-framework/ctx-management/caching.html)

Spring이 애플리케이션을 실행하려면 애플리케이션에 필요한 객체를 만들고 관리하는 공간이 필요합니다.
이 공간을 `ApplicationContext`라고 부릅니다.
Spring은 `ApplicationContext`를 생성할 때 설정을 읽고 필요한 객체를 모두 생성해 의존 관계를 연결하는 초기화 작업을 수행하므로, `ApplicationContext`를 생성하는 작업은 다른 작업에 비해 무겁습니다.

Spring의 TestContext 프레임워크는 **같은 구성을 사용하는 테스트마다 `ApplicationContext`를 한 번만 만들고 재사용**합니다.
덕분에, `ApplicationContext`를 띄우는 무거운 작업은 한 번만 수행하면서 여러 테스트를 실행할 수 있죠.

여기서 "같은 구성을 사용하는가?"를 결정하는 기준을 **Context Cache Key**라고 부르고, Context Cache Key는 다음과 같은 설정들로 구성됩니다.

| 구성 요소 | 설명 |
| --- | --- |
| `locations` | 컨텍스트를 구성할 XML 등 설정 리소스의 위치 |
| `classes` | 컨텍스트를 구성할 설정 클래스 목록 |
| `contextInitializerClasses` | 컨텍스트 초기화에 사용할 초기화 클래스 목록 |
| **`contextCustomizers`** | **컨텍스트 구성을 추가·변경하는 커스터마이저. `@Import`로 추가한 빈 구성도 여기에 반영** |
| `contextLoader` | 컨텍스트를 생성하고 로드하는 로더 |
| `parent` | 계층 구조에서 상위 컨텍스트의 구성 |
| `activeProfiles` | 테스트에서 활성화할 프로파일 |
| `propertySourceDescriptors` | 테스트용 프로퍼티 리소스의 위치와 로딩 설정 |
| `propertySourceProperties` | 테스트에 직접 지정한 프로퍼티 값 |
| `resourceBasePath` | 웹 애플리케이션 리소스의 기준 경로 |

문제는, `@Import`**로 어떤 빈을 추가했는지까지 Context Cache Key에 반영**된다는 거였습니다.

```java
@DataJpaTest
// BookService 빈만 Import한 ApplicationContext 필요
@Import(BookService.class)
class BookServiceTest {
    // ...
}

@DataJpaTest
// CommentService 빈만 Import한 ApplicationContext 필요
@Import(CommentService.class)
class CommentServiceTest {
    // ...
}

@DataJpaTest
// ClubService 빈만 Import한 ApplicationContext 필요
@Import(ClubService.class)
class ClubServiceTest {
    // ...
}
```

저희는 서비스 테스트 클래스마다 서로 다른 빈을 `Import`하고 있었으니, 전체 테스트를 돌릴 때마다 **서비스 테스트 클래스 개수만큼의** `ApplicationContext`**가 추가로 생성**되고 있었습니다.

저희의 경우, 서비스 테스트를 위해 8개의 `ApplicationContext`를 추가로 생성하고 있었습니다.

# ApplicationContext 생성이 이렇게 느린가?

여기까지만 보면 이런 의문이 생길 수 있습니다.

> `ApplicationContext`를 몇 번 더 만드는 것만으로 테스트가 이렇게까지 느려질까?

저희의 통합 테스트는 Docker의 MySQL 컨테이너를 사용하고 있었습니다.
이 때문에, `ApplicationContext`에 실제 MySQL을 실행하는 Testcontainers 설정이 포함되어 있어 `ApplicationContext` 생성이 더 오래 걸렸습니다.

```java
@TestConfiguration
class TestcontainersConfiguration {

    @Bean
    @ServiceConnection
    MySQLContainer<?> mysqlContainer() {
        return new MySQLContainer<>("mysql:8.4");
    }
}
```

실제 테스트 구성을 단순화하면 다음과 같습니다.

```java
@DataJpaTest
@Import({
    TestcontainersConfiguration.class,
    BookService.class
})
class BookServiceTest {
    // ...
}
```

새로운 `ApplicationContext`가 만들어질 때마다 `MySQLContainer` 빈도 새로 만들어졌습니다.
그 결과 Spring Bean만 다시 등록한 것이 아니라, MySQL 컨테이너를 생성하는 작업까지 수행됐습니다.

즉, 서비스마다 다른 `@Import` 설정 때문에 **매번 MySQL 테스트 환경 전체를 다시 준비**하고 있었습니다.

# 어떻게 해결할 수 있을까?

## 1. MySQL 컨테이너를 재사용하면 되지 않을까?

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
abstract class MySqlJpaTest {

    private static final MySQLContainer<?> MYSQL = new MySQLContainer<>("mysql:8.4");

    static {
        MYSQL.start();
    }

    @DynamicPropertySource
    static void configureDataSource(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", MYSQL::getJdbcUrl);
        registry.add("spring.datasource.username", MYSQL::getUsername);
        registry.add("spring.datasource.password", MYSQL::getPassword);
    }
}
```

`ApplicationContext`와 별개로, MySQL 컨테이너는 한 번만 생성해 재사용할 수 있습니다.
이를 Singleton Container Pattern이라고 합니다.

이러면 `ApplicationContext`를 새로 생성해도 MySQL 컨테이너는 재사용하니, 지금 `ApplicationContext` 생성이 오래 걸리는 원인 중 하나는 제거할 수 있습니다.
하지만, 저희는 애당초 `ApplicationContext`를 불필요하게 여러 번 생성하는 게 근본적인 문제라고 판단했기에 Singleton Container Pattern을 고려하지 않았습니다.

## 2. DB를 바꾸면 해결되지 않을까?

저희는 우선 "반드시 MySQL로 테스트해야 할까?"를 고민했습니다.
H2를 통해 인메모리 DB로 테스트하면, MySQL 테스트 환경을 준비하느라 드는 시간을 없앨 수 있기 때문이죠.

이점과 한계를 비교해보면 다음과 같습니다.

* 이점
  * 테스트의 Docker 의존성이 감소한다.
  * 테스트 속도가 MySQL보다 빠르다.
* 한계
  * MySQL에서의 동작을 검증할 수 없다.

H2도 매력적이었지만, 저희는 계속 MySQL로 테스트하기로 결정했습니다.
테스트 환경과 운영 환경의 데이터베이스 관리 시스템(DBMS)이 달라지면 테스트로 운영 안정성을 보장할 수 없다고 판단했기 때문입니다.

## 3. `@Import`를 통일하자

테스트마다 `@Import` 설정이 달라 `ApplicationContext`를 재사용하지 못하는 게 근본적인 문제니, `@Import` 설정을 통일해야 했습니다.

저희는 다음 2가지 방법 중 어떤 방법을 선택할지 고민했습니다.

### 3-1. `@SpringBootTest`로 통일하기

```java
@SpringBootTest
class BookServiceTest {
    // ...
}

@SpringBootTest
class CommentServiceTest {
    // ...
}
```

서비스 테스트가 `@DataJpaTest` 대신 `@SpringBootTest`를 사용하도록 통일하는 방법입니다.

가장 구성이 단순하지만, 테스트와 무관한 빈까지 항상 로드한다는 단점이 있습니다.

### 3-2. 필요한 `@Import`를 공통 애노테이션으로 묶기

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@DataJpaTest
@Import({
    BookService.class,
    CommentService.class,
    ClubService.class
})
public @interface ServiceJpaTest {
}
```

`@DataJpaTest`는 유지하면서, 서비스 테스트들이 사용할 빈들을 모두 공통 애노테이션에 등록합니다.

```java
@ServiceJpaTest
class BookServiceTest {
    // ...
}

@ServiceJpaTest
class CommentServiceTest {
    // ...
}
```

각 서비스 테스트에서는 공통 애노테이션만 사용합니다.

서비스 테스트에 필요한 빈들만 로드하면서도, 서비스 테스트끼리는 하나의 `ApplicationContext`를 재사용할 수 있습니다.

### 3-3. 비교하기

| 비교 항목 | `@SpringBootTest` | 공통 애노테이션 |
| --- | --- | --- |
| 유지보수 (새로운 서비스 테스트가 추가된다면?) | 기존 테스트를 수정할 필요 없음 | 공통 애노테이션을 수정해야 함 |
| 복잡도 (개발자의 인지부하) | 단순히 `@SpringBootTest` 사용 | 서비스 테스트에만 사용하는 공통 애노테이션을 인지해야 함 |
| 테스트 초기화 비용 | 빈 로딩 범위가 넓어, 상대적으로 큼 | 로드하는 빈의 범위가 좁아 초기화 비용이 상대적으로 작음 |
| 무관한 빈의 영향 | 무관한 빈의 초기화가 실패하면 서비스 테스트도 실패함 | 무관한 빈에 영향받지 않음 |

저희는 애플리케이션을 이제 막 개발하기 시작한 상태였습니다.
따라서, "**앞으로의 유지보수에 어떤 방식이 유리한가?**"를 가장 중요한 가치로 설정했습니다.

이에 따라, **구조가 가장 간단하고 유지보수가 간편한 `@SpringBootTest` 방식을 선택**했습니다.
이때, 무관한 빈에 의해 문제가 발생할 가능성은 낮다고 판단했기에, 단점이 크지 않다고 판단했습니다.

# 개선 전후 비교

`@SpringBootTest`로 구성을 통일하면 실제로 얼마나 빨라질까요?
같은 13개 클래스의 64개 테스트를 변경 전후 각각 10번씩 실행해 비교해봤습니다.

| 비교 항목 | 개선 전 | 개선 후 |
| --- | --- | --- |
| `ApplicationContext` 생성 횟수 | 9번 | 2번 |
| 실행 시간 중앙값 | **144.3초** | **50.9초** (**약 64.7% 감소**) |

단순히 `ApplicationContext`를 재사용할 수 있게 만들었을 뿐인데, 테스트 시간이 144초에서 50초로 약 64% 감소했습니다.

# 마치며

저희의 사고 흐름을 요약하면 다음과 같습니다.

1. 테스트 속도가 느리다
2. `ApplicationContext`를 재사용하지 못하고 매번 생성하는 게 문제다
3. `ApplicationContext`가 생성될 때 MySQL 컨테이너까지 생성하니 더 오래 걸린다
4. 인메모리 DB를 사용해볼까? → 운영 환경과 테스트 환경을 최대한 일치시켜야 하니 사용하지 않는다
5. `@Import`를 통일하자 → 구조가 간단하고 유지보수하기 쉬운 `@SpringBootTest`로 통일하자

다음 내용은 의도적으로 다루지 않았습니다.
흥미가 있다면 따로 조사해 보세요.

1. `@SpringBootTest`는 테스트 간 DB 격리까지 담당하지 않습니다. 테스트 간 DB를 어떻게 격리할지는 별도로 고민해야 합니다.
2. `@SpringBootTest`로 통일하더라도, 모든 테스트가 같은 `ApplicationContext`를 사용하진 않습니다. `@Import` 외에도 다양한 요인에 의해 Context Cache Key가 달라질 수 있습니다.
