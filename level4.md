## 해당 주제를 선택하게 된 이유

프로젝트 중 `읽기용 DB - 쓰기용 DB` 분리를 진행하던 중
`java.sql.SQLException: The MySQL server is running with the --read-only option so it cannot execute this statement`
해당 에러를 해결하기 위해 10시간 정도 삽질하다가 `open-in-view` 설정이 문제임을 깨달아서 자세히 알아보기 위해 선택했습니다.

## 사전 지식

해당 내용들은 OSIV 에 필수적인 지식은 아니나 혼돈을 줄 수도 있기 때문에 설명하고 넘어갑니다.

### JPA 기본 제공 함수의 트랜잭션

```java
@Override  
public AuthInfo resolveArgument(final MethodParameter parameter, final ModelAndViewContainer mavContainer,  
								final NativeWebRequest webRequest, final WebDataBinderFactory binderFactory) {  
	
	final String name = webRequest.getHeader("Authorization");  
	
	final Member member = memberRepository.findById(1L)  
			.orElseThrow(() -> new IllegalArgumentException(String.format("Not Exist Member : %s", name)));    
	return AuthInfo.from(member);  
}
```

본격적인 OSIV 심층 조사에 앞서
이와 같은 함수는 READ , WRITE 중 어떤 DB 에 연결을 할까요? 
( Class Level 에도 `Transactional` 은 없습니다. )

<img width="500" alt="image" src="https://i.imgur.com/RCazs1x.jpeg">

정답은 READ DB 로 향하게 됩니다.

JPA 의 단순 구현체인 [SimpleJPARepository](https://docs.spring.io/spring-data/data-jpa/docs/current/api/org/springframework/data/jpa/repository/support/SimpleJpaRepository.html) 에 나와있는 것처럼 JPA 에서 제공해주는 함수들은

```java
@Transactional(readOnly = true)  
public class SimpleJpaRepository<T, ID> implements JpaRepositoryImplementation<T, ID> {
	@Override  
	public Optional<T> findById(ID id) {
		...
	}
	
	@Override  
	@Transactional  
	public void deleteAllById(Iterable<? extends ID> ids) {  
	  
	    Assert.notNull(ids, "Ids must not be null");  
	  
	    for (ID id : ids) {  
	       deleteById(id);  
	    }  
	}
}
```
 
와 같이 임의적으로 `Transaction` 을 가지고 있습니다.

### Readonly 동작 원리

```java
@Transactional(readOnly = true)
public MemberResponse getMember(final String name) {  
    final Member member = memberRepository.findByName(name)  
            .orElseThrow(EXCEPTION::get);  
    member.view();  
    return MemberResponse.from(member);  
}
public class Member {

	...
	private long viewCount;
	
	public void view() {  
	    this.viewCount += 1;  
	}
}
```

이와 같은 코드가 있을 때 해당 함수를 호출하면 결과 & 쿼리가 어떻게 될까요?

<img width="500" alt="image" src="https://i.imgur.com/RCazs1x.jpeg">

정답은 

```sql
select
	m1_0.id,
	m1_0.name,
	m1_0.view_count 
from
	member m1_0 
where
	m1_0.name=?
```

와 같이 `UPDATE` 문이 없고 `SELECT` 문만 존재합니다.
분명히, JPA 는 엔티티의 변경을 감지하고 DB에 flush 를 한다고 설명을 들은거 같은데요.(Dirty Check)

```java
public class HibernateJpaDialect extends DefaultJpaDialect {

	...
	
	@Nullable  
	protected FlushMode prepareFlushMode(Session session, boolean readOnly) throws PersistenceException {  
	    FlushMode flushMode = session.getHibernateFlushMode();
		if (readOnly) {  
		    // We should suppress flushing for a read-only transaction.  
		    if (!flushMode.equals(FlushMode.MANUAL)) {  
		       session.setHibernateFlushMode(FlushMode.MANUAL);  
		       return flushMode;  
		    }  
		}
	}
}
```

Hibernate 는  `readOnly` 이면 FlushMode 를 `MANUAL` 로 설정합니다. - [HibernateJpaDialect](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/orm/jpa/vendor/HibernateJpaDialect.html) , [FlushMode](https://docs.jboss.org/hibernate/orm/3.5/javadoc/org/hibernate/FlushMode.html)
MANUAL 은 간단하게 설명하면
- 스냅샷(기존 엔티티를 저장해 상태를 비교하는 용)을 사용하지 않습니다.
- 명시적으로 호출하지(`session.flush`) 않으면 DB 에 어떤 변경 사항도 반영하지 않습니다.

그렇기에, JPA 는 viewCount 가 1 증가한 것을 알지 못하며 반영하지 않습니다.

```java
@Transactional  
public MemberResponse getMember(final String name) {  
    final Member member = memberRepository.findByName(name)  
            .orElseThrow(EXCEPTION::get);  
    member.view();  
    return MemberResponse.from(member);  
}
```

와 같이 `readonly=false` 로 변경하면

```sql
select
	m1_0.id,
	m1_0.name,
	m1_0.view_count 
from
	member m1_0 
where
	m1_0.name=?
	
update
	member 
set
	name=?,
	view_count=? 
where
	id=?
```

`UPDATE` 문이 나가는걸 알 수 있습니다.

## OSIV 로 인한 문제 발생

```java
@Override  
@Transactional(readOnly = true)  
public AuthInfo resolveArgument(final MethodParameter parameter, final ModelAndViewContainer mavContainer,  
								final NativeWebRequest webRequest, final WebDataBinderFactory binderFactory) {  
	final Long id = Long.parseLong(webRequest.getHeader("Authorization"));  
	final Member member = memberRepository.findById(id)  
			.orElseThrow(() -> new IllegalArgumentException(String.format("Not Exist Member : %s", id)));  
	return AuthInfo.from(member);  
}

==============================================


@Transactional  
public MemberResponse getMember(final String name) {  
    final Member member = memberRepository.findByName(name)  
            .orElseThrow(EXCEPTION::get);  
    member.view();  
    return MemberResponse.from(member);  
}
```

로그인을 하고 회원 정보를 가져오는 로직을 예시로 들어보겠습니다.
실행을 하면?

`java.sql.SQLException: The MySQL server is running with the --read-only option so it cannot execute this statement`
이와 같은 에러가 발생합니다. ( 해당 문제 때문에 15시간 정도를 삽질하며 날렸습니다,, 😢 )

분명히, 트랜잭션에 맞게 DB 를 향하게 라우팅 하게 했습니다.
해당 에러는 `OSIV` 때문에 발생한 에러입니다.

### OSIV 란?

> Open Session In View

직역하면 `View 영역까지 세션이 열려있다.` 이며,
좀 더 풀어보자면 하나의 HTTP 요청 동안 **동일한 JPA 영속성 컨텍스트**를 유지하는 것을 의미합니다.

이 말은 두 가지의 특징을 포함합니다.
- 여러 개의 `Transactional` 에서도 하나의 JPA 영속성 컨텍스트를 가져서 사용한다.
- `View` 영역에서도 지연 로딩(`Lazy Loading`) 을 가능하게 해준다.

사진으로 설명하면

<img width="500" alt="image" src="https://i.imgur.com/FXrGnUJ.png">

이와같이 됩니다.

### 실제 확인

위의 말이 사실인지 확인하기 위해 현재 연결된 Connection 의 정보를 가져와보겠습니다.
( 로그를 확인한 방법은 [util 패키지](https://github.com/youngsu5582/open-in-view-study/tree/main/src/main/java/joyson/openinviewtest/util) 를 참고해주세요. )

> 이때 `dataSource.getConnection` 을 통해서는 유의미한 확인을 할 수 없습니다.
> 말 그대로, `Transactional` 에 맞게 새로운 연결을 받아오는 것이므로 기존 연결이 아닙니다.

ArgumentResolver 에서는

```
트랜잭션 활성 상태: true
트랜잭션 읽기 전용: true
트랜잭션 이름: joyson.openinviewtest.auth.LoginMemberArgumentResolver.resolveArgument

Connection URL: jdbc:mysql://localhost:3307/readDB
Managed entity: EntityKey[joyson.openinviewtest.member.Member#1]
```

```
트랜잭션 활성 상태: true
트랜잭션 읽기 전용: false
트랜잭션 이름: joyson.openinviewtest.member.MemberService.getMember

Connection URL: jdbc:mysql://localhost:3307/readDB
Managed entity: EntityKey[joyson.openinviewtest.member.Member#1]
```

처럼 Context 가 유지되는걸 볼 수 있습니다.

### 문제 해결?

```yml
open-in-view: false
```

그러면 OSIV 설정을 꺼보겠습니다.

```
트랜잭션 활성 상태: true
트랜잭션 읽기 전용: true
트랜잭션 이름: joyson.openinviewtest.auth.LoginMemberArgumentResolver.resolveArgument

Connection URL: jdbc:mysql://localhost:3307/readDB
Managed entity: EntityKey[joyson.openinviewtest.member.Member#1]
```

```
트랜잭션 활성 상태: true
트랜잭션 읽기 전용: false
트랜잭션 이름: joyson.openinviewtest.member.MemberService.getMember

Connection URL: jdbc:mysql://localhost:3306/writeDB
No entity keys found
```

와 같이 새로운 Connection 과 컨텍스트가 새로운 것을 볼 수 있습니다. 

```json
{
    "id": 1,
    "name": "joyson",
    "viewCount": 1
}
```
로직도 성공적으로 작동하는데 끝일까요...? 단순, 설정을 끄는게 뭔가 꺼림칙 하지 않나요?
좀 더 알아보겠습니다.

<img width="500" alt="image" src="https://i.imgur.com/g52XJuf.jpeg">

## OSIV 가 죄악인가

스프링을 구동할 때 로그를 자세히 보면

```
[2024-09-29 16:14:42:10834] [main] WARN  [
org.springframework.boot.autoconfigure.orm.jpa.
JpaBaseConfiguration$JpaWebConfiguration.openEntityManagerInViewInterceptor:236] 
- spring.jpa.open-in-view is enabled by default. Therefore, database queries may be performed during view rendering. Explicitly configure spring.jpa.open-in-view to disable this warning

open-in-view 가 활성화 되어 있어서 뷰 렌더링( JSP,Thymeleaf ) 중에서 DB 쿼리가 수행될 수 있습니다.
성능 저하나 예상치 못한 DB 접근을 초래할 수 있습니다.
```

와 같이 경고가 뜨는걸 볼 수 있습니다. 무조건 사용하지 않아야 하는 `Deprecated` 적인 요소일까요?

이에 대해서는 아직도 뜨거울 수도 있는 논쟁입니다. ( 그렇기에 제목으로 어그로도 끌었고요 🙂🙂 ) <br>
( [What is this spring.jpa.open-in-view=true property in Spring Boot?](https://stackoverflow.com/questions/30549489/what-is-this-spring-jpa-open-in-view-true-property-in-spring-boot) - 9년전에 물어봤지만, 1달 전에도 수정이 되고 있습니다. ) <br>
( [A Guide to Spring’s Open Session in View](https://www.baeldung.com/spring-open-session-in-view) OSIV 에 대한 설명이 자세히 담겨 있습니다. )

그러면 반대편의 입장으로 OSIV 를 쓰지 않았을 때 발생한 (`+`가능성이 있는) 문제점들에 대해 살펴보겠습니다.

### 매번 새로운 연결, 새로운 영속성 컨텍스트

극단적으로, 컨트롤러 내 각각 트랜잭션 내에서 작업을 해야 하는 경우가 있다고 가정해보겠습니다.

```java
@GetMapping("/login")  
public ResponseEntity<MemberResponse> login(@LoginMember final AuthInfo authInfo, HttpServletRequest httpServletRequest) {  
	memberService.increaseTodayView(authInfo.id());  
	memberService.increaseTotalView(authInfo.id());  
	memberService.increaseGradeQualification(authInfo.id());  

	return ResponseEntity.ok(memberService.getMember(authInfo.name()));  
}
```

해당 코드는 새로운 커넥션 & 새로운 컨텍스트를 가지게 됩니다.
그렇기에, 당연히 매번 쿼리문을 날리게 됩니다. ( 아래 참고 )

```
HikariProxyConnection@1630316951 wrapping com.mysql.cj.jdbc.ConnectionImpl@53ba7997

No entity keys found

select
	m1_0.id,
	m1_0.name,
	m1_0.view_count 
from
	member m1_0 
where
	m1_0.name=?

update
	member 
set
	...
where
	id=?

HikariProxyConnection@577434038 wrapping com.mysql.cj.jdbc.ConnectionImpl@53ba7997

select
	m1_0.id,
	m1_0.name,
	m1_0.view_count 
from
	member m1_0 
where
	m1_0.id=?
	
update
	member 
set
	name=?,
	view_count=? 
where
	id=?
```

### 영속 관리의 어려움

다른 트랜잭션의 도메인을 실수로 사용할 수 있습니다.
아니 누가 그런 실수를? 라고 할 수 있지만
DDD 또는 더 큰 도메인 개념을 만들어 작업을 할 때 충분히 발생할 수 있는 실수라고 생각합니다.

```java
public MemberResponse updateNickname(@LoginMember final AuthInfo authInfo, @RequestParam final String newNickname) {  
  
    final Member member = memberService.getMemberDomain(authInfo.name());  
    return memberService.changeNickname(member, newNickname);  
}

@Transactional  
public MemberResponse changeNickname(final Member member, final String newNickname) {  
    final Profile profile = member.getProfile();  
    profile.changeNickname(newNickname);  
    return MemberResponse.from(member);  
}

```

와 같이 외부 트랜잭션의 도메인을 가지고 작업시

```
org.hibernate.LazyInitializationException: could not initialize proxy [joyson.openinviewtest.member.Profile#1] - no Session
	at org.hibernate.proxy.AbstractLazyInitializer.initialize(AbstractLazyInitializer.java:165)
```

해당 에러가 발생하게 됩니다.

```java
@GetMapping("/domain")  
public ResponseEntity<MemberResponse> domain(@LoginMember final AuthInfo authInfo) {  
    final Member member = memberService.getMemberDomain(authInfo.name());  
    return ResponseEntity.ok(MemberResponse.from(member));  
```

컨트롤러 단에서 도메인 통해 DTO 변환하는 경우에도 동일합니다.

## OSIV 의 문제점

그러면 OSIV 는 어떤 문제가 있을까요?

### 분기에 따른 DataSource 선택 불가능

```java
@DependsOn({"routeDataSource"})  
@Primary  
@Bean  
public DataSource dataSource(final DataSource routeDataSource) {  
    return new LazyConnectionDataSourceProxy(routeDataSource);  
}
```

와 같은 필요할 때 연결을 하는 `LazyConnectionDataSourceProxy` 사용이 불가능합니다.
( AOP 나 Custom Filter 를 통해 의도적으로 제어는 가능할 수도 있을거 같으나, 다루지 않겠습니다. )

### 긴 Connection Time

`@Transactional` 단위로 커넥션을 반환하지 않고
`View Level` 의 Request - Response 가 끝날때 까지 Connection 을 가지므로 긴 연결 시간을 가집니다.

### 의도치 않은 쿼리 발생

개발자의 실수(의도?) 일 수 있으나 예상치 못한 곳에서 쿼리가 발생하고, 이를 인지 못할수 있습니다. 

> IMO if you have a transactional unit-of-work whose result of execution requires further fetching from database that occurs after the aforementioned unit-of-work has ended I don't know how adding behavior that supports such data access pattern can be described not being an anti-pattern.
> 
> 트랜잭션 단위 작업의 결과가 끝난 후에 데이터베이스에서 추가로 조회가 필요하다면, 
> 그러한 데이터 접근 패턴을 지원하는 동작을 추가하는 것이 어떻게 반패턴이 아닐 수 있는지 이해하기 어렵습니다.

와 같은 의견도 존재하나, 결국은 의도하지 않은 것은 주의를 해야 할 필요가 있다고 생각합니다. 
( N+1 문제, 단순 조회가 아니라 변경 작업이라면 `@Transactional` 내에서만 수행 )

### 동시성 문제 발생

DB 작업의 논리적 단위는 트랜잭션입니다. `ACID` 역시도 트랜잭션의 범위 내에서 보장을 합니다.
하지만, OSIV 는 다른 트랜잭션 내에서 수행한 엔티티 역시도 가지고 있으므로
필요한 곳에서 조회를 하지 않을 수 있습니다.

```
트랜잭션을 시작한다.
1. A 의 계좌를 조회한다.
2. A 의 계좌를 조회해서 10만원을 인출해 B 의 계좌에 송금한다.
트랜잭션을 종료한다.

다른 서버에서 A 의 계좌에 100만원을 입금했다.

A의 계좌에서 추가적으로 20만원을 인출한다.
이때, 기존에 존재한 엔티티로 작업을 수행한다. ( 100만원 입금이 반영되지 않은 ) 
```

이와 같이 개발자가 제어할 수 없는 문제들을 발생시키는 것을 알 수 있습니다.

## 네. 쓰지 맙시다. ( 왠만하면 )

제가 느끼기에, 비교적 ON -> OFF 는 쉬운거 같습니다.
( `LazyInitializationException` 를 다뤄야 하는 곳을 찾아서 DTO 또는 메서드 단으로 변환하면 됩니다.  - 이 마저도, 코드를 잘 작성했으면 문제가 되지 않을껍니다. )


하지만, OFF -> ON 은 어려움이 유발될 수 있습니다.

앞서 말한것과 같은 
- 동적 DB Connection 처리
- 의도치 않은 동시성
- 커넥션으로 인한 성능 저하 
와 같은 문제점들이 발생할 수 있습니다.

그러면 `처음부터 키면 되는거 아닌가?`
라고 생각할 수는 있지만 도움을 주는 `슈가 코드` 가 아닌 `관리 포인트 & 제어할 수 없는 포인트` 만 늘어난다고 생각합니다.

이런 관점도 있으니 참고만 해주세요!

<img width="1148" alt="image" src="https://github.com/user-attachments/assets/842dcf09-0b20-46d2-bdc3-8501973e3654">

이상입니다. 감사합니다!
