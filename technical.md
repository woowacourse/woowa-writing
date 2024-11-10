# OSIV, DB 복제 환경에서 왜 문제였나? 제대로 이해해보기 

## 1. 개요

이 글에서는 일상 기록 서비스 `Staccato`의 단일 데이터베이스 환경에서 복제 환경으로 마이그레이션하는 과정에서 `OSIV(Open Session In View)` 설정과 관련된 문제를 다룹니다.<br>
OSIV가 무엇인지 학습하고, 실제 문제 상황을 재현하고 분석하는 과정을 통해 문제의 본질을 이해하고 해결책을 제시합니다.

이 글은 특히 OSIV의 개념을 잘 모르는 독자를 대상으로 OSIV의 동작 방식과 설정에 따른 장단점을 체계적으로 설명하고, 비슷한 상황에서 문제 파악과 해결을 논리적으로 할 수 있도록 돕는 것을 목표로 합니다.

## 2. 문제 상황
### 2.1 Writer-Reader Replication 환경으로의 마이그레이션

데이터베이스의 성능과 확장성을 높이기 위해 싱글 DB 인스턴스 구조에서 Writer-Reader 복제 환경으로 마이그레이션 작업을 진행했습니다.
기존에는 단일 EC2 인스턴스에 MySQL을 설치하여 사용했고, 마이그레이션 과정에서 Writer와 Reader로 구성된 2대의 RDS를 사용하는 구조로 변경했습니다.

### 2.2 Spring Boot에서 다중 데이터 소스 구성 과정

Spring Boot는 하나의 데이터소스를 사용하는 경우 AutoConfiguration으로 `DataSource`, `EntityManager`, `TransactionManager` 등을 설정합니다.
하지만, 2개 이상의 데이터소스를 정의하면 Spring Boot의 기본 AutoConfiguration은 비활성화되고, 개발자가 직접 코드로 두 개의 `DataSource`를 명시해야 합니다.

따라서 애플리케이션 코드에서 `AbstractRoutingDataSource`를 사용하여, 쓰기 트랜잭션은 Writer로, 읽기 트랜잭션은 Reader로 보내도록 설정했습니다.

``` java
protected Object determineCurrentLookupKey() {
    if (isCurrentTransactionReadOnly()) {
        return READER;
    }
    return WRITER;
}
```
특정 조건에 맞는 데이터 소스를 명시하지 않으면 기본적으로 Writer를 사용하도록 설정했습니다.
``` java
@Configuration
public class DataSourceConfig {
    @DependsOn({WRITER_DATA_SOURCE, READER_DATA_SOURCE})
    @Bean
    public DataSource routingDataSource(
            @Qualifier(WRITER_DATA_SOURCE) DataSource writer,
            @Qualifier(READER_DATA_SOURCE) DataSource reader
    ) {
        DynamicRoutingDataSource routingDataSource = new DynamicRoutingDataSource();
        Map<Object, Object> dataSourceMap = new HashMap<>();
        dataSourceMap.put(WRITER, writer);
        dataSourceMap.put(READER, reader);
        routingDataSource.setTargetDataSources(dataSourceMap);
        routingDataSource.setDefaultTargetDataSource(writer);
        return routingDataSource;
    }
    ...
}
```
### 2.3 문제 발생과 해결 과정

위의 작업을 완료한 후, RDS로 DataSource를 변경하는 작업을 진행했습니다.<br>
그리고 여기에서 다음과 같은 문제가 발생했습니다.

![](image/img.png)

해결 방법은 간단했습니다.<br>
`application.yml`에서 OSIV 설정을 비활성화로 바꾸면 해결할 수 있었습니다.
``` properties
spring.jpa.open-in-view=false
```
하지만 문제의 본질은 이해하지 못하고 있었습니다.

**애초에 OSIV가 무엇이며, 왜 문제가 된 것일까요?**

## 3. OSIV
### 3.1 OSIV의 기본 개념
`OSIV(Open Session In View)`는 `JPA/Hibernate`에서 사용되는 개념으로, 영속성 컨텍스트를 뷰까지 열어두는 것을 의미합니다.
즉, View에서도 엔티티가 영속 상태로 유지되기 때문에 지연 로딩과 같은 영속성 컨텍스트의 특징을 사용할 수 있습니다.

> 참고: OSIV에서 Session이란 Hibernate에서 영속성 컨텍스트를 가리키는 용어입니다.

### 3.2 OSIV의 목적

스프링 컨테이너는 **트랜잭션 범위의 영속성 컨텍스트 전략**을 기본으로 사용합니다. 
그리고 같은 트랜잭션 안에서는 항상 같은 영속성 컨텍스트에 접근합니다.

![img_3.png](image/img_3.png)

스프링 프레임워크를 사용한다면 보통 비즈니스 로직을 시작하는 Service 계층에 `@Transactional` 어노테이션을 선언하여 트랜잭션을 시작합니다. 
그리고 서비스 계층이 끝나는 시점에 트랜잭션이 종료되면서 영속성 컨텍스트도 함께 종료됩니다.

![img_4.png](image/img_4.png)

`Service`와 `Repository` 계층에서는 조회한 엔티티는 영속성 컨텍스트에서 관리되면서 영속 상태를 유지하지만, `Presentation` 계층(`Controller`, `View`)에서는 준영속 상태가 됩니다.<br>
즉, `Presentation` 계층에서는 더 이상 영속성 컨텍스트의 기능을 사용할 수 없습니다.<br>
그리고 지연 로딩 기능이 동작하지 않는다는 점은 문제가 되기도 합니다.

`Presentation` 계층에서 지연 로딩으로 설정된 연관된 엔티티를 프록시 객체로 조회했다고 가정해 보겠습니다.<br>
아직 초기화되지 않은 프록시 객체를 사용하면, 실제 데이터를 불러오는 시도를 할 것입니다.<br>
하지만, 더 이상 영속성 컨텍스트의 관리 대상이 아니기 때문에 Hibernate 구현체 기준 `org.hibernate.LazyInitializationException`이 발생하게 됩니다.

이와 같이 **엔티티가 `Presentation` 계층에서 준영속 상태이기 때문에 발생**하는 문제를 해결하기 위해 OSIV를 사용할 수 있습니다.

### 3.3 OSIV의 동작

가장 단순한 구현 방법은 클라이언트의 요청이 들어오자마자 서블릿 필터나 스프링 인터셉터에서 트랜잭션을 시작하고, 마치는 것입니다.<br>
이를 **요청 당 트랜잭션 방식의 OSIV**라고 합니다.

![img_5.png](image/img_5.png)

이 방식은 트랜잭션이 종료된 후에도 영속성 컨텍스트 내의 엔티티에 접근할 수 있고 지연 로딩을 포함한 다양한 JPA 연산이 가능합니다. <br>
하지만 `Service` 계층처럼 비즈니스 로직 실행 시 데이터가 변경되는 것이 아닌 `Presentation` 계층에서 데이터를 수정했을 때 실제 데이터베이스까지 변경이 반영된다는 문제점이 있습니다. 
그렇기 때문에 최근에는 거의 사용하지 않는 방법입니다.

최근에는 이러한 문제점을 어느 정도 보완해서 **비즈니스 계층에서만 트랜잭션을 유지하는 방식**의 OSIV를 사용합니다.

![img_6.png](image/img_6.png)

HTTP 요청이 들어올 때 영속성 컨텍스트를 열고, 요청이 끝날 때까지 이를 유지합니다.<br>
따라서, `Presentation` 계층에서도 영속성 컨텍스트 내의 엔티티에 접근할 수 있고 지연 로딩을 포함한 다양한 JPA 연산이 가능합니다.<br>
트랜잭션의 범위는 영속성 컨텍스트의 범위와 다르게 `Service` 계층에서 시작되고, 종료됩니다.
트랜잭션 범위 밖에서는 영속성 컨텍스트 내 엔티티를 조회만 할 수 있습니다.<br>

따라서, 요청 당 트랜잭션 방식의 OSIV는 `Presentation` 계층에서 데이터를 변경할 수 있다는 문제를 극복할 수 있습니다.

스프링 프레임워크가 제공하는 OSIV는 `비즈니스 계층에서 트랜잭션을 사용하는 OSIV`입니다.

**그렇다면 스프링 프레임워크에서 OSIV는 어떻게 동작하고 있을까요?**

### 3.4 Spring의 OSIV 동작 원리
스프링 ORM에서는 다양한 OSIV 클래스를 제공합니다. <br>
OSIV를 서블릿 필터 적용할지 스프링 인터셉터에서 적용할지에 따라 원하는 클래스를 선택하여 사용할 수 있습니다.
> Hibernate 기준
> - OpenSessionInViewFilter
> - OpenSessionInViewInterceptor

Spring JPA의 OSIV 옵션이 true일 때 동작 과정을 `OpenSessionInViewInterceptor`의 코드와 함께 살펴봅시다.

1. **요청이 들어오면 `Spring Interceptor`에서 세션(영속성 컨텍스트)을 생성합니다.**
   ``` java
   @Override
   public void preHandle(WebRequest request) throws DataAccessException {
       String key = getParticipateAttributeName();
       AsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
       if (asyncManager.hasConcurrentResult() && applySessionBindingInterceptor(asyncManager, key)) {
           return;
	   }

       if (TransactionSynchronizationManager.hasResource(obtainSessionFactory())) {
           // Do not modify the Session: just mark the request accordingly.
           Integer count = (Integer) request.getAttribute(key, WebRequest.SCOPE_REQUEST);
           int newCount = (count != null ? count + 1 : 1);
           request.setAttribute(getParticipateAttributeName(), newCount, WebRequest.SCOPE_REQUEST);
	   }
       else {
           logger.debug("Opening Hibernate Session in OpenSessionInViewInterceptor");
           Session session = openSession();
           SessionHolder sessionHolder = new SessionHolder(session);
           TransactionSynchronizationManager.bindResource(obtainSessionFactory(), sessionHolder);
           AsyncRequestInterceptor asyncRequestInterceptor = new AsyncRequestInterceptor(obtainSessionFactory(), sessionHolder);
           asyncManager.registerCallableInterceptor(key, asyncRequestInterceptor);
           asyncManager.registerDeferredResultInterceptor(key, asyncRequestInterceptor);
       }
   }
   ```
   `OpenSessionInViewInterceptor`는 영속성 컨텍스트를 요청 범위 전체에 걸쳐 유지하는 역할을 합니다. 
   이는 트랜잭션 범위에서만 영속성 컨텍스트를 열고 닫는 기본 방식과 차이가 있습니다. 
   이 인터셉터는 클라이언트 요청이 들어오면 `preHandle` 메서드를 통해 영속성 컨텍스트를 생성하고 요청의 끝까지 유지함으로써, View 계층에서도 지연 로딩을 사용할 수 있게 합니다.
2. **Transaction AOP 혹은 `begin`을 이용하여 트랜잭션을 시작할 때 앞서 생성한 영속성 컨텍스트를 사용하여 트랜잭션을 시작합니다.**
   ``` java
   TransactionSynchronizationManager.bindResource(obtainSessionFactory(), sessionHolder);
   ```
   앞서 생성한 영속성 컨텍스트를 `TransactionSynchronizationManager`에 바인딩하는 작업을 `prehandle`메서드에서 진행함을 확인할 수 있습니다.
3. **`Service` 클래스에서 `Transaction AOP`를 이용한다면, `Service` 로직이 완료되었을 때 트랜잭션을 커밋하고 종료합니다.**
4. **영속성 컨텍스트는 아직 유지하고 있습니다. 즉, 데이터베이스와의 커넥션이 종료되지 않았습니다. `Controller`와 `View`까지 영속성 컨텍스트가 유지됩니다.**
5. **`Spring Interceptor`로 로직이 다시 돌아오면 세션(영속성 컨텍스트)을 종료합니다.**
   ``` java
   @Override
   public void afterCompletion(WebRequest request, @Nullable Exception ex) throws DataAccessException {
      if (!decrementParticipateCount(request)) {
         SessionHolder sessionHolder = (SessionHolder) TransactionSynchronizationManager.unbindResource(obtainSessionFactory());
         logger.debug("Closing Hibernate Session in OpenSessionInViewInterceptor");
         SessionFactoryUtils.closeSession(sessionHolder.getSession());
      } 
   }
   ```
   이때 `flush`가 아닌 `close` 메서드가 호출됩니다. 따라서, 엔티티의 변경 내용을 반영하지 않고 종료합니다.

세션이 생성되는 시점과 트랜잭션을 시작하는 시점을 다르게 가져가는 방법은 이해했습니다. <br>
**그렇다면, 트랜잭션 범위 밖에서 세션(영속성 컨텍스트) 내 엔티티에 생긴 변경 사항은 어떻게 반영되지 않는 걸까요?**

세션을 생성하는 과정을 조금 더 자세히 보겠습니다.<br>
`OpenSessionInViewInterceptor`내에 있는 `openSession()` 메서드를 살펴봅시다.

`openSession` 메서드는 `SessionFactory`에서 새로운 `Session`을 열어 반환합니다. <br>
Spring의 `OpenSessionInViewInterceptor`는 이 메서드를 통해 트랜잭션이나 `Presentation` 계층에서 사용할 세션을 생성합니다.
``` java
protected Session openSession() throws DataAccessResourceFailureException {
   try {
      Session session = obtainSessionFactory().openSession();
      session.setHibernateFlushMode(FlushMode.MANUAL);
      return session;
   }
   catch (HibernateException ex) {
      throw new DataAccessResourceFailureException("Could not open Hibernate Session", ex);
   }
}
```
이 메서드 내에서 생성된 세션은 `FlushMode.MANUAL`로 플러쉬 모드가 설정됩니다.<br>
이 설정 덕분에 트랜잭션 밖의 변경 사항이 실제 데이터베이스 상에 반영되지 않습니다.<br>
OSIV에서 세션이 열려있더라도 트랜잭션이 끝난 후에는 데이터 무결성을 유지할 수 있는 이유입니다.

>`FlushMode`는 플러쉬 전략을 의미하고, `MANUAL`, `COMMIT`, `AUTO`, `ALWAYS` 모드가 존재합니다. <br>
그 중 `MANUAL` 모드는 `Session.flush()`가 명시적으로 호출되었을 때만 플러쉬 작업이 발생하여 변경 사항이 DB에 동기화됩니다.
`@Transactional(read-only=true)`일 때에도 `MANUAL` 모드를 사용합니다.

지금까지 OSIV의 개념과 동작 원리를 살펴보았습니다.<br>
이제 실제 환경에서 OSIV 설정이 어떤 문제를 유발할 수 있는지 구체적으로 살펴보겠습니다.

## 4. 문제 상황 분석하기
### 4.1 환경 구성
실서버에서 겪었던 문제 상황을 재현하기 위해서 저는 로컬 환경에 Docker를 사용하여 Writer-Reader 설정하여 문제 상황을 재현했습니다.
``` yaml
## application.yml
spring:
  ...
  datasource:
    writer:
      driver-class-name: com.mysql.cj.jdbc.Driver
      jdbcUrl: jdbc:mysql://localhost:3307/staccato
      username: staccato
      password: 1234
    reader:
      driver-class-name: com.mysql.cj.jdbc.Driver
      jdbcUrl: jdbc:mysql://localhost:3308/staccato
      username: staccato
      password: 1234
  jpa:
    ...
    open-in-view: false
```

### 4.2 가설 설정
> **가설**<br>
> 모든 API는 인증을 거친다.<br>
> 인증을 거칠 때 획득한 Read 트랜잭션을 OSIV로 인해 요청이 끝날 때까지 사용하게 되면서 Write 작업임에도 Read db에서 시도하여 오류가 발생한다.

아래의 설정값을 변경하며, 문제 상황을 분석해 보도록 하겠습니다.
``` properties
spring.jpa.open-in-view=false
```
### 4.3 테스트 방법
간단한 시나리오로 OSIV를 활성화했을 때와 비활성화했을 때의 결과를 분석해 보고자 합니다.
>**시나리오**<br>
> Staccato 서비스를 사용하는 회원(`Member`)이 추억(`Memory`)을 생성한다.<br>
>
> ** `Memory`: 회원의 기록을 담는 일종의 카테고리

**1. 특정 `Member`의 권한으로 새로운 `Memory` 생성을 시도합니다.**
   ``` shell
   curl --location 'localhost:8080/memories' \
   --header 'Authorization: token' \
   --header 'Content-Type: application/json' \
   --data '{
      "memoryThumnailUrl": "https://example.com/memorys/geumohrm.jpg",
      "memoryTitle": "2023 여름 휴가",
      "description": "친구들과 함께한 여름 휴가 여행",
      "startAt": "2023-07-01",
      "endAt": "2023-07-10"
   }'
   ```
   위의 API를 호출했을 때, 다음과 같은 `Controller`가 호출됩니다.
   ``` java
   @PostMapping
   public ResponseEntity<MemoryIdResponse> createMemory(
           @Valid @RequestBody MemoryRequest memoryRequest,
           @LoginMember Member member
           ) {
           MemoryIdResponse memoryIdResponse = memoryService.createMemory(memoryRequest, member);
           return ResponseEntity.created(URI.create("/memories/" + memoryIdResponse.memoryId())).body(memoryIdResponse);
   }
   ```
**2. 내부적으로는 해당 메서드들이 호출될 때, OSIV 설정에 따라 사용하는 영속성 컨텍스트와 커넥션 정보를 비교 분석합니다.**
   ``` java
   @Transactional(readOnly = true)
   public Member extractFromToken(String token)
   ```
   ``` java
   @Transactional
   public MemoryIdResponse createMemory(MemoryRequest memoryRequest, Member member)
   ```

### 4.4 spring.jpa.open-in-view=false
특정 `Member`의 권한으로 새로운 `Memory` 생성을 시도하면, `ArgumentResolver`에 의해 인증 작업이 수행됩니다. <br>
인증을 시도할 때, `extractFromToken()`을 통해 `Member`의 정보를 조회합니다.<br>
따라서, 해당 메서드가 호출될 때는 Reader 데이터베이스의 커넥션이 획득됩니다.

![img_9.png](image/img_9.png)

해당 메서드가 종료될 때, 트랜잭션이 종료되면서 영속성 컨텍스트 또한 종료됩니다.

이후, `createMemory()`를 호출되면, 새로운 트랜잭션이 시작됨과 동시에 앞서와는 별개의 영속성 컨텍스트가 생성됩니다.<br>
해당 메서드에서는 Writer 데이터베이스의 커넥션을 획득합니다.

![img_7.png](image/img_7.png)

마찬가지로 메서드가 종료되면서 트랜잭션이 종료됨과 동시에 영속성 컨텍스트 또한 종료됩니다.

OSIV가 비활성화되어있다면, `Presentation` 계층에서는 영속성 컨텍스트가 유지되지 않습니다.<br>
따라서, 사용하던 커넥션을 유지할 필요가 없으므로 Reader 데이터베이스에 쓰기 작업 시도가 발생하지 않습니다.

### 4.5 spring.jpa.open-in-view=true
특정 `Member`의 권한으로 새로운 `Memory` 생성을 시도하면, `ArgumentResolver`에 의해 인증 작업이 수행됩니다. <br>
인증을 시도할 때, `extractFromToken()`을 통해 `Member`의 정보를 조회합니다.
따라서, 해당 메서드가 호출될 때는 Reader 데이터베이스의 커넥션이 획득됩니다.

![img_10.png](image/img_10.png)

해당 메서드가 종료될 때 트랜잭션은 종료되지만, 영속성 컨텍스트는 종료되지 않습니다.

즉, **사용 중이던 Reader 데이터베이스의 커넥션이 반환되지 않습니다.**

따라서, 이후에 `createMemory()`를 호출되었을 때 기존의 영속성 컨텍스트를 재사용함에 따라 사용 중이었던 커넥션을 그대로 재사용합니다.

![img_11.png](image/img_11.png)

**그 과정에서 쓰기 작업을 Reader 데이터베이스에 시도하게 되면서, 권한 문제로 인하여 작업을 실패하게 됩니다.**

### 4.6 결과
Spring의 OSIV는 `Presentation` 계층의 변경 감지 권한은 허용하지 않으면서, 지연 로딩을 할 수 있게 하는 방법으로 **영속성 컨텍스트는 유지하되 트랜잭션을 종료**시키는 방법을 택했습니다.<br>
**하지만, 영속성 컨텍스트를 유지함에 따라 한 번 얻은 커넥션은 요청이 끝날 때까지 반환하지 않습니다.**<br>
그렇기 때문에, DB Replication 환경에서는 **인증으로 인해 Reader 데이터베이스의 커넥션을 가져왔을 때, 해당 Connection을 반납하지 않고 재사용하면서 쓰기 작업을 시도해 오류가 발생**하는 것을 확인했습니다.

따라서, OSIV 설정을 비활성화하여 문제를 쉽게 해결할 수 있었습니다.

그렇다면, OSIV의 기본 설정 값이 활성화인 이유가 있지 않았을까요?

### 4.7 OSIV 활성화 VS OSIV 비활성화
OSIV를 사용할 때와 사용하지 않을 때의 단점을 생각해봅시다.

OSIV를 사용한다면 다음과 같은 단점이 있습니다.

**(1) 데이터베이스의 커넥션 점유 시간 증가** 
   - 데이터베이스의 서비스 특성(ex: 실시간)에 따라 커넥션이 부족하여 장애가 발생할 수 있습니다.

**(2) Presentation 계층에서 엔티티 수정 후 Service 계층 호출 문제**
   - Spring OSIV의 경우, `Presentation` 계층에서 엔티티를 수정한 직후에 트랜잭션을 시작하는 `Service` 계층을 호출하면 문제가 발생합니다.<br>
   - Spring OSIV는 같은 영속성 컨텍스트를 여러 트랜잭션이 공유할 수 있으므로 이와 같은 문제가 발생할 수 있습니다.

**(3) 지연 로딩에 따른 성능 문제**
   - `Presentation` 계층에서 지연 로딩에 의한 SQL이 실행되므로, 성능 튜닝 시 확인해야 할 부분이 넓습니다.

그렇다면, 사용하지 않았을 때의 단점은 어떤 것들이 있을까요?

(1) 지연로딩을 트랜잭션 밖에서 사용할 수 없습니다.<br>
(2) 트랜잭션마다 다른 영속성 컨텍스트를 사용하므로, 발생하는 쿼리가 더 많습니다.

단점을 비교해보았을 때, OSIV를 비활성화했을 때의 단점은 관심사만 명확하게 분리해도 쉽게 극복 가능하다고 생각했습니다.
오히려 관심사 분리를 명확하게 할 수 있다는 장점으로 보이기도 합니다.

따라서, **굳이 OSIV를 활성화해야만 하는 이유는 없다**고 생각합니다.

## 5. 마무리
지금까지 Staccato 서비스를 `Writer-Reader Replication` 환경으로 마이그레이션하면서 `OSIV(Open Session In View)` 설정과 관련된 문제를 어떻게 마주했는지 재현하고, OSIV의 개념과 동작 방식을 분석해 보았습니다.<br>
이 과정에서 OSIV가 왜 문제를 발생시켰는지 이해하고, OSIV 설정을 비활성화하는 방식으로 문제를 해결했습니다.<br>
마지막으로, OSIV가 활성화 여부에 따른 장단점을 함께 비교 분석하며 OSIV의 비활성화가 문제 해결에 적합한지 판단해보았습니다.

이 글이 OSIV의 개념을 이해하고, 유사한 문제 상황에서 해결 방안을 찾는 데 도움이 되기를 바랍니다.

## 참고자료
- 자바 ORM 표준 JPA 프로그래밍 (김영한 저)