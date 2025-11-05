
# `@Transactional`, 왜 조심해야 할까?

## 들어가기 전

<img width="818" height="392" alt="image" src="https://github.com/user-attachments/assets/e9cc018a-c08b-443b-8797-c56cbdcacecc" />

스프링에 입문하여 JPA를 처음 다룰 때, 코드래빗의 리뷰를 통해 `@Transactional`의 존재를 알게 되었다.

대충 찾아봤을 때는 일관성 보장 등 장점밖에 없는 것 같았고, “이걸 안 쓸 이유가 있을까?”라는 생각으로 무조건 붙이곤 했다. 특히 여러 작업을 한 번에 처리하는 경우엔 반드시 사용해야 한다고 생각했다. 하지만 다양한 상황을 겪으면서, **아무 생각 없이 `@Transactional`을 사용하는 것이 얼마나 위험할 수 있는지** 깨달았다. 

이 글은 필자처럼 Spring의 입문 단계에 있는 개발자들이 시행착오를 겪지 않도록 미리 알아두면 좋을 내용들을 정리한 글이다. Spring Data JPA의 Entity Manager에 대한 배경 지식이 있다면 더욱 이해하기 쉬울 것이다.

## 1️⃣ 트랜잭션과 `@Transactional`

### ✔️ 트랜잭션?
트랜잭션은 **db의 상태를 변경시키는 작업의 단위**다. 가장 와닿았던 설명은, 트랜잭션 작업 단위는 모두 성공하거나 모두 실패해야 한다는 것이었다.

가장 흔하게 사용되는 예시는 송금 기능이다. A 통장에서 B 통장으로 송금하는 과정은 A 통장에서 출금, B 통장에 입금 두 단계로 이루어진다. 그런데 만약 A 통장에서 출금만 성공하고 B 통장에 입금에 실패한다면 명백한 오류 상황이다. 출금, 입금은 모두 실패하거나 모두 성공해야 하고, 이를 하나의 트랜잭션 단위라고 할 수 있다.

### ✔️ 트랜잭션 특성 - ACID
트랜잭션이 존재하는 이유는, 그 핵심 원칙인 **ACID**에서 찾을 수 있다. 이 네 가지 원칙이 **데이터의 신뢰성과 안정성을 보장하는 기반**이 된다.

- **Atomicity (원자성)** - 트랜잭션에 포함된 작업은 **모두 성공하거나, 모두 실패해야 한다.**  일부만 반영되는 상태는 허용되지 않는다.
- **Consistency (일관성)** - 트랜잭션이 끝난 후 데이터는 항상 **신뢰할 수 있는, 일관된 상태**를 유지해야 한다. 예를 들어, 은행 송금이라면 돈이 한쪽에서 빠졌으면 다른 쪽에는 반드시 입금되어야 한다.
- **Isolation (격리성)** - 동시에 여러 트랜잭션이 실행되더라도, **서로 간섭하지 않아야 한다.** 각 트랜잭션은 마치 혼자 실행되는 것처럼 독립적으로 동작해야 한다.
- **Durability (지속성)** - 트랜잭션이 한 번 **성공적으로 커밋되면**, 그 결과는 전원 장애나 시스템 오류가 나더라도 **영구히 보존**되어야 한다.

### ✔️ DB 트랜잭션

위에서 작성한 송금 과정을 하나의 트랜잭션으로 묶는 방법은 아래와 같다.

```sql
BEGIN;

UPDATE account SET balance = balance - 10000 WHERE id = 1;
UPDATE account SET balance = balance + 10000 WHERE id = 2;

COMMIT; -- 성공 시 반영
-- ROLLBACK; -- 실패 시 복구
```

`BEGIN`으로 트랜잭션을 시작하면 DB는 자동 커밋 모드를 해제하고, 이후 실행되는 모든 SQL은 같은 트랜잭션 컨텍스트 안에서 수행된다. 두 `UPDATE` 연산이 모두 성공했을 때만 `COMMIT`을 수행하여 변경 내용을 영구 반영한다. 중간에 예외가 발생하면 `ROLLBACK`을 호출하여 두 쿼리 모두 취소되고, 송금 전 상태로 복구된다.

이렇게 하면 원자성(Atomicity)이 보장되어, 한쪽 계좌만 변경되는 불일치 상태를 방지할 수 있다.

### ✔️ JPA에서 트랜잭션  

SQL 레벨에서의 트랜잭션은 `BEGIN`, `COMMIT`, `ROLLBACK`과 같은 명령으로 제어했지만, 실제 애플리케이션에서는 이러한 명령을 코드로 직접 다루지 않는다. DB와의 연결은 JDBC나 JPA 같은 **영속성 계층(persistence layer)** 이 대신 관리해주기 때문이다.

따라서 자바 애플리케이션에서도 트랜잭션의 경계를 지정해줄 방법이 필요한데, 스프링에서는 이를 `@Transactional` 어노테이션으로 관리할 수 있다. 즉, **SQL의 `BEGIN`과 `COMMIT`을 코드 차원에서 추상화한 것이 `@Transactional`** 이다.

JPA에서는 EntityManager를 통해 직접 트랜잭션을 제어할 수도 있다.

```java
EntityManagerFactory entityManagerFactory = Persistence.createEntityManagerFactory("jpa-example");
EntityManager entityManager = entityManagerFactory.createEntityManager();

try {
    entityManager.getTransaction().begin(); // 트랜잭션 시작
    
    entityManager.persist(firstEntity);
    entityManager.persist(secondEntity);
    
    entityManager.getTransaction().commit(); // 성공 시 반영
} catch (Exception e) {
    entityManager.getTransaction().rollback(); // 실패 시 복구
}
```
### ✔️ `@Transactional`

트랜잭션이 필요한 모든 작업 단위마다 위의 코드를 작성해야 한다면 상당히 번거로울 것이다.

**Spring Data JPA**를 사용하면 `@Transactional` 애노테이션 하나만 붙여주면 된다. 스프링은 `@Transactional`이 붙은 메서드가 호출되면 트랜잭션을 시작하고, 정상적으로 종료되면 **커밋**, 예외가 발생하면 **롤백**을 수행해준다.

주의할 점은, **스프링의 트랜잭션은 기본적으로 언체크 예외가 발생했을 때만 롤백된다는 것**이다. 체크 예외는 개발자가 **예상하고 처리할 수 있는 예외**로 간주되므로, 스프링은 이를 자동으로 롤백하지 않는다. 필요하다면 `rollbackFor` 속성을 사용해 체크 예외에 대해서도 명시적으로 롤백을 설정할 수 있다.

```java
@Transactional(rollbackFor = Exception.class) 
public void processOrder() {     
	... 
}
```

### ✔️ 프록시

어떻게 메서드 실행 전, 후에 트랜잭션 로직을 수행할 수가 있을까?

트랜잭션은 여기저기 중복으로 사용되는 기능이다. 트랜잭션을 관리하는 코드를 재사용한다면, 비즈니스 로직에 대한 코드에서 트랜잭션 관련 코드를 분리할 수 있다. 스프링은 이를 **프록시**를 이용해 구현한다.

```java
@Component  
public class ImProxy {  
  
    @Transactional  
    public void imTransactionalMethod() {  
  
    }  
}
```

```java
@SpringBootTest  
public class TransactionTest {  
  
    @Autowired  
    private ImProxy imProxy;  

    @Test  
    @DisplayName("@Transactional이 붙은 클래스는 프록시 빈이 주입된다")  
    void transactionTest1() {  
        System.out.println(imProxy.getClass());  
    }
}
```

```
class test.transaction.ImProxy$$SpringCGLIB$$0
```

`@Transactional`이 붙어있는 메서드가 존재하거나 클래스에 `@Transactional`이 붙어있다면, 스프링은 **해당 클래스를 프록시로 생성해서 주입**한다. 따라서 우리가 트랜잭션이 걸린 로직을 호출하면, 핵심 로직을 실행하기 전에 트랜잭션을 시작하고, 로직이 끝난 후 커밋 혹은 롤백을 수행할 수 있다.

그래서 private 메서드에는 `@Transactional`을 붙일 수 없다. 왜냐하면 한 프록시 객체 내부에서 다른 메서드를 호출하면, 트랜잭션이 적용된 프록시 말고 실제 클래스의 로직이 호출되기 때문이다. 필자는 처음에 이 사실을 모르고 private 메서드에 `@Transactional` 붙이면 컴파일 에러가 발생하길래, 그냥 public으로 바꾸면 되는 줄 알았었다.

## 2️⃣ `@Transactional`을 사용할 때 고려해야 할 점

### ✔️ `@Transactional` 기본 옵션

`@Transactional`은 기본 옵션일 때 **더티 체킹**과 **flush**가 일어난다. 관련 내용은 JPA의 EntityManager를 공부하면 알 수 있는데, 간단히 설명하자면 영속성 컨텍스트에 등록된 엔티티의 값의 변경을 감지(더티 체킹)하여 실제 db에도 반영(flush)해준다는 것이다. 그리고 이는 결국 CPU와 메모리를 사용하는 연산 작업이다. 트랜잭션 내에서 값 변경이 일어나지 않는 경우, 즉 읽기 작업만 수행하는 경우에는 더티 체킹과 flush는 불필요한 오버헤드가 될 것이다.

```java
@Service
@RequiredArgsConstructor
public class MemberService {

    private final MemberRepository memberRepository;

    @Transactional
    public List<Member> findAllMembers() {
        // 10만 명의 회원을 한 번에 조회
        // 단순히 조회만 하지만, 모든 엔티티가 "변경 감시 대상"으로 등록됨
        List<Member> members = memberRepository.findAll();
		return members;
    }
}
```

위의 예시 코드를 살펴보자. 조회된 모든 Member 엔티티가 영속성 컨텍스트에 등록될 것이다. 그리고 Hibernate는 각 엔티티의 변경 여부를 감지하기 위해  **초기 상태를 스냅샷 형태로 복제해두며**, 트랜잭션이 종료되어 flush가 일어나기 전까지 **일정량의 메모리를 점유**한다.

예를 들어, `Member` 엔티티가 10만 개 존재한다고 가정해보자.
```java
@BeforeAll
static void setup(@Autowired MemberRepository memberRepository) {
    if (memberRepository.count() == 0) {
        for (int i = 0; i < 100_000; i++) {
            memberRepository.save(new Member("name" + i, "email" + i, "password" + i));
        }
    }
}
```

`findAllMembers()` 메서드를 호출했을 때, 각 `Member` 객체가 영속성 컨텍스트에 로드되고, 기본 트랜잭션에서는 각 엔티티의 **초기 상태를 스냅샷 형태로 복제**한다. 이 경우 얼마만큼의 메모리 용량을 차지하는지 유추해보자.

- `Member` 객체의 평균 메모리 크기: 약 **350~400바이트** (필드 + 객체 헤더 포함)
- 10만 개 엔티티 × 400B = 약 **38MB**

스냅샷까지 복제되면 **약 70~80MB 이상**의 힙 메모리를 점유한다.

이 작업이 짧은 순간이라면 문제가 없지만, 이 로직이 **트래픽이 많은 서비스에서 자주 호출**된다면 GC 빈도 증가와 **OutOfMemoryError**로 이어질 수 있다.

### ✔️ DB 자원 잠금 문제

트랜잭션이 시작되면 DataSource에서 DB 커넥션(Connection)을 빌려오고, 트랜잭션이 커밋되거나 롤백될 때까지 이 커넥션을 점유한다. HikariCP 같은 커넥션 풀은 한정적인 수의 커넥션을 관리하고, 기본 설정은 보통 10개다. 남은 커넥션이 존재하지 않을 경우에는 요청이 대기 큐에서 기다리게 되고, 설정된 `connectionTimeout`(기본 30초)을 초과하면 **타임아웃 예외**가 발생한다. 

```java
@Transactional
public void pay() {
	
	...
	// 평균적으로 5초 소요되는 결제 api 호출
	paymentClient.confirmPayment(10_000);
	
	...
}
```

그런데 만약 트랜잭션 내부에 **오래 걸리는 로직**이 존재한다면, 그 시간 동안 커넥션이 계속 점유되어 다른 요청들이 커넥션을 얻지 못하고 대기한다. 이 대기가 길어지면 결국 **응답 지연**이나 **타임아웃 에러**로 이어질 수 있다.

또한 트랜잭션은 DB의 **락(Lock)** 과도 밀접하게 관련된다. 트랜잭션이 테이블이나 행 단위로 락을 오랫동안 유지하면, 다른 트랜잭션이 해당 자원에 접근하지 못하게 되고, 결국 **지연 전파(Lock Contention)** 가 발생한다.

이렇게 문제가 될 수 있는 긴 트랜잭션을 유발하는 원인은 아래의 경우가 있을 것이다.
- `@Transactional` 메서드 안에서 외부 api 호출, 파일 입출력 수행
- **사용자 입력 혹은 검증 대기**
- 대량 조회 후 **스트림/루프 처리**로 수 초 동안 객체 변환 및 가공
- 컨트롤러, 서비스, **뷰 렌더링까지 트랜잭션 유지**

이러한 경우들은 트랜잭션 경계를 **불필요하게 확장**시켜, 커넥션과 락이 장시간 점유되는 주요 원인이 된다. 따라서 DB 조회, 수정, 저장 같은 **순수 데이터 조작 로직만 트랜잭션 안에서 수행**하고, 그 외의 오래 걸리는 로직은 트랜잭션 외부로 분리할 필요가 있다.

### ✔️ 외부 시스템과의 불일치 문제

사용자는 아래와 같은 **어이없는 상황**을 겪을 수 있다.

```java
@Transactional
public void payAndSaveOrder() {
    // 결제 API 호출
    paymentClient.confirmPayment(10_000);

    // 주문 데이터 저장
    orderRepository.save(new Order(...));

    // 이후 로직 중 예외 발생 -> 롤백
    throw new RuntimeException("주문 처리 중 오류 발생");
}
```

이 코드에서 결제 API는 외부 네트워크를 통해 호출되기 때문에, 사용자의 결제는 이미 승인된 상태가 된다. 문제는 그다음이다. 결제는 성공했지만, 우리 서비스 내부 로직에서 예외가 발생하면 TransactionManager가 전부 롤백시켜버린다. 결과적으로 사용자의 입장에서는 "돈은 빠져나갔으나 주문은 실패한" 상황이 발생하게 된다.

이처럼 **개발자의 통제 범위를 벗어난 외부 시스템(결제, 문자, 이메일 등)** 을 트랜잭션 내부에서 호출할 때는 각별히 주의해야 한다. DB는 롤백으로 되돌릴 수 있지만, 한 번 전송된 네트워크 요청은 **되돌릴 수 없기 때문이다.**

## 3️⃣ 그럼 어떻게 사용해야 할까?

### ✔️ 조회만 할 때는 `readOnly = true`

```java
@Transactional(readOnly = true)
public List<Member> findAllMembers() {
    return memberRepository.findAll();
}
```
`@Transactional`에는 `readOnly` 옵션이 존재한다. 이는 말 그대로 **읽기 전용 트랜잭션**을 의미하며, **데이터를 변경하지 않는 조회 로직**이라면 반드시 설정해주는 것이 좋다.

`readOnly = true`를 감지하면, 쓰기 작업이 없을 예정이라는 힌트가 전달된다. 그 결과, 더티 체킹을 위한 스냅샷이 생성되지도 않으며, db 변화가 일어나지 않기에 flush되지 않는다. 따라서 CPU와 메모리 부담이 줄어들고, 트랜잭션 경량화 효과를 얻을 수 있다. 또한 조회 전용 로직을 의도했으나 실수로 엔티티의 값을 변경하는 경우 db에는 반영되지 않기 때문에 실수를 방지할 수도 있다.

### ✔️ 오래 걸리는 작업을 처리하는 방법

앞서 살펴봤듯, 트랜잭션 내부에서 시간이 오래 걸리는 로직을 수행하면 그동안 DB 커넥션이 점유되어 **지연, 락 경합, 타임아웃**으로 이어질 수 있다.

이 문제를 해결하는 가장 깔끔한 방법 중 하나는 스프링이 제공하는 **이벤트 기반 비동기 처리(Event Driven Processing)** 방식이다.

**동기 요청**은 요청과 응답이 순차적으로 처리되고, 응답이 올 때까지 기다리는 방식이다. 반면 비동기 요청은, “처리는 나중에 알아서 해” 라는 식으로 일단 요청은 보내놓고, 다른 작업을 병렬로 진행할 수 있다.

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepository;
    private final ApplicationEventPublisher eventPublisher;

    @Transactional
    public void createOrder(OrderRequest request) {
        // 1️⃣ 결제 요청 이벤트 발행
        eventPublisher.publishEvent(new PaymentEvent(order.getId(), request.getAmount()));
        
	    // 2️⃣ 주문 생성 및 저장
        orderRepository.save(new Order(...));
    }
}
```

```java
@Component
@RequiredArgsConstructor
public class PaymentEventHandler {

    private final PaymentClient paymentClient;

    // 트랜잭션 커밋 후 실행되도록 설정
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    @Async  // 별도 스레드에서 비동기 실행
    public void handlePaymentEvent(PaymentEvent event) {
        // 트랜잭션과 무관하게 결제 처리 수행
        paymentClient.confirmPayment(event.amount());
    }
}
```

결제 api의 응답을 기다리지 않고, 일단 요청을 보내둔 채로 주문을 생성하기 때문에, 커넥션 점유 시간이 줄어든다. `@TransactionalEventListener`의 phase를 `AFTER_COMMIT`으로 설정하면, DB 커밋이 완료된 후에 이벤트가 트리거된다.

다만, 이 경우 **두 가지 불일치 상황**이 발생할 수 있다. 

첫째, 주문 데이터는 정상적으로 insert되어 커밋되었지만, 결제 이벤트가 실패하여 실제 결제가 이루어지지 않는 경우가 있다. 둘째, 결제는 성공했지만 주문 데이터 생성이 실패하여 트랜잭션이 롤백되는 경우다. 이러한 불일치 상황을 처리하기 위해서는 별도의 보상 로직이 필요하다.

### ✔️ 보상(Compensation) 트랜잭션

트랜잭션 내에 롤백할 수 없는 로직이 존재한다면 어떻게 해야할까?

```java
@Transactional
public void payAndSaveOrder() {
    // 결제 API 호출
    paymentClient.confirmPayment(10_000);

    // 주문 데이터 저장
    orderRepository.save(new Order(...));

    // 이후 로직 중 예외 발생 -> 롤백
    throw new RuntimeException("주문 처리 중 오류 발생");
}
```

예를 들어, 이미 결제가 진행되어 돈이 빠져나간 상태에서 **결제에 대한 롤백**은 불가능하다.

이럴 때 사용하는 것이 바로 **보상 트랜잭션(Compensating Transaction)** 이다. 보상 트랜잭션은 말 그대로 “이미 커밋되어 되돌릴 수 없는 작업을 논리적으로 되돌리는 반대 작업(Undo Logic) 으로 복구하는 것”이다. 실패를 없던 일로 만드는 게 아니라, **되돌린 것처럼 보이게 만드는 복원 로직**을 작성하면 된다.

예를 들어, 결제가 실패했는데 롤백이 필요한 경우에는 **환불 api**를 호출해서 논리적으로 롤백할 수 있다.

```java
@Transactional
public void payAndSaveOrder() {
    try {
        paymentClient.confirmPayment(10_000);    // 결제 승인
        orderRepository.save(new Order(...));    // 주문 저장
        // 이후 로직 ...
    } catch (Exception e) {
        // 예외 발생 시 결제 취소(보상 트랜잭션)
        paymentClient.cancelPayment(10_000);
        throw e;
    }
}
```

다만 여기서 주의할 점은, 보상 트랜잭션조차 실패할 수 있다는 것이다. 결제 취소 API 호출이 실패하면, 결국 **데이터 불일치**가 남는다. 이 경우 재시도 로직을 구현하거나, 실패한 요청을 이벤트로 발행시키는 DLQ(Dead Letter Queue) 등을 반드시 고려해야 한다.

보상 트랜잭션을 구현하는 경우에는, 트랜잭션 내부의 실행 순서도 고려해야 한다. 되돌릴 때 문제가 발생할 수 있는 경우를 구분하고, 그 순서를 전략적으로 정해야 한다.

문제 상황을 확인하기 위해 아래의 입출금에 대한 코드를 확인해보자.

```java
@Transactional
public void transfer(Long fromId, Long toId, int amount) {
    try {
        // 1️⃣ 입금 처리 (외부 송금 API)
        bankClient.deposit(toId, amount);

        // 2️⃣ 출금 처리 (외부 출금 API)
        bankClient.withdraw(fromId, amount);

        // 3️⃣ 거래 내역 저장
        transferRepository.save(new Transfer(fromId, toId, amount));

    } catch (Exception e) {
        // 예외 발생 시 입금 취소(보상 트랜잭션)
        bankClient.withdraw(toId, amount);
        throw e;
    }
}
```

입금은 성공했는데 출금 과정에 오류가 발생해서, 입금에 대한 보상 트랜잭션으로 입금 취소, 즉 출금을 시켜야 하는 상황을 생각해보자. 만약 그 사이에 사용자의 통장의 잔고가 부족해진다면, 출금이라는 보상 트랜잭션을 수행할 수 없다.

정리하자면, 트랜잭션 내에서는 되돌릴 수 있는 작업부터 실행하고, 되돌리기 어려운 작업은 **마지막에** 실행하도록 설계해야 한다.

#### 💡 Saga 패턴

보상 트랜잭션이라는 개념이 등장한 배경은 마이크로서비스 환경에서의 롤백 상황이다. 예를 들어 외화 서비스와 원화 서비스가 독립적으로 운영되는 경우, "환전"이라는 기능을 구현하려면 두 서비스를 하나의 작업 단위로 관리해야 한다.

그러나 한 서비스의 작업이 실패하더라도, 이미 커밋이 완료된 다른 서비스의 작업은 되돌릴 수 없다. 따라서 전통적인 롤백 대신, **실패한 작업을 반대로 되돌리는 보상(Compensation) 로직**을 도입하게 되었고, 이 개념을 체계적으로 정립한 것이 **Saga 패턴**이다.

한 서비스가 실패하더라도 이미 커밋이 완료된 다른 서비스의 작업은 되돌릴 수 없다. 따라서 전통적인 롤백이 아니라, 실패한 작업을 반대로 되돌리는 보상(Compensation) 로직을 도입하게 되었고, 이를 체계화한 것이 Saga 패턴이다. 

하지만 Saga 패턴을 안정적으로 운용하기 위해서는 고려해야 할 변수가 훨씬 많다. 예를 들어, 보상 트랜잭션이 실패했을 때 이를 기록하기 위해 메시지를 브로커(Kafka, RabbitMQ 등)에 남길 수 있다. 그런데 이 메시지 전송조차 실패할 수 있다. 그렇다면 메시지를 재전송해야 하고, 보상 트랜잭션이 실제로 실행되었는지 검증하는 로직도 필요하다. 하지만 그 검증 로직 자체도 실패할 가능성이 있다. 혹은 메시지 서버 자체에 장애가 발생한 상황에는 메시지 손실이 발생할 수 있다.

이처럼 보상 트랜잭션은 단순히 실패를 되돌리는 방법이 아니라, 실패하더라도 시스템이 일관성을 유지하도록 만드는 복잡한 전략이다. 그만큼 알아야 할 것도, 고려해야 할 것도 많다. 그래서 MSA 환경에서의 트랜잭션은 단순한 기술이 아니라 “실패를 설계하는 기술”이고, 더 깊이 공부해야 할 주제라고 생각한다.

## 마무리

이 글에서는 `@Transactional`의 기본 개념부터 시작해서, 무분별하게 사용할 때 발생할 수 있는 문제점들을 다뤘다. 불필요한 더티 체킹으로 인한 메모리 낭비, 긴 트랜잭션으로 인한 DB 커넥션 고갈, 외부 시스템과의 불일치 문제 등이 대표적이다. 그리고 이러한 문제들을 해결하기 위해 `readOnly` 옵션 활용, 비동기 이벤트를 통한 작업 분리, 보상 트랜잭션 패턴까지 살펴봤다.

내가 사용하는 기술이 어떤 문제를 유발할 수 있는지는, 직접 문제 상황을 겪어보기 전까지는 알기 어렵다. 필자 역시 아무 생각 없이 `@Transactional` 메서드 안에서 외부 API를 호출했었고, 누군가 지적해주지 않았다면 그것이 문제라는 사실조차 깨닫지 못했을 것이다.

이 글을 접한 사람들이 나와 같은 시행착오를 겪지 않고 이러한 문제점을 인지하고 있는다면, 더욱 안정적인 서비스를 운영할 수 있을 것이다.

이 글에서는 다루지 않았지만, 트랜잭션의 **격리 수준(Isolation Level)** 이나 **전파 옵션(Propagation)** 등  
알아두면 도움이 되는 내용이 많다. 본인이 스프링 코드에서 `@Transactional`을 사용하고 있다면, 한 번쯤은 이 개념들을 깊이 있게 공부해보기를 권한다.
