# 파산을 막기 위한 동시성 문제 해결 방법 총망라!

> 예제 코드는 [이곳](https://github.com/le2sky/spring-atom/tree/main/spring-concurrency-coupon)에서 확인하실 수 있습니다!

동시성은 성능을 높이는 기술이기도 하지만, 제대로 알고 사용하지 않으면 독이 되기도 합니다. 저는 우아한테크코스 6기 데벨업 프로젝트를 진행하면서 이 동시성 때문에 난항을 겪었었는데요. 설명의 편의를 위해 모두에게
친숙한 **쿠폰 발급 예제로 동시성 문제를 어떻게 해결해 볼 수 있는지 이야기**해보려 합니다. 문장의 간결함을 위해 높임말은 생략할게요. 그리고, 데벨업 프로젝트에서는 MySQL 8.0을 사용했기 때문에 이 글의
데이터베이스와 관련된 이야기는 MySQL 8.0 InnoDB를 기준으로 작성되었다는 점 참고 부탁드립니다. 😀

## 1. 쿠폰 발급 API와 동시성 문제

### 1.1 동시성 문제란 무엇인가?

동시성(Concurrency)이란 여러 작업들이 빠르게 전환되면서 실행되어 마치 동시에 실행되는 것처럼 보이는 것을 일컫는다.
예를 들어, 손은 2개이지만 저글링을 하면 3개의 공을 한 번에 다룰 수 있는 것과 비슷한 이치이다.
동시성은 스레드로 달성할 수 있다. 동시에 여러 스레드가 실행되는 경우 데이터 정합성이 맞지 않는 문제가 발생할 수 있는데 이를 동시성 문제라고 한다.

### 1.2 동시성 문제 사례 - 쿠폰 발급 API

쿠폰 발급 기능을 구현하려고 한다. 쿠폰 발급 기능의 가장 중요한 요구사항은 1명의 사용자는 1개의 쿠폰만 발급받을 수 있다는 것이다.
만약, 그렇지 않는다면 쿠폰을 발급한 회사는 **파산**할 것이다.
<p align="center">
    <img src="./tech-coupon-intro.png"/>
</p>

쿠폰 발급 API는 다음과 같이 쿠폰 발급 서비스의 기능을 사용한다.
그리고 쿠폰 발급 서비스는 사용자가 1개의 쿠폰만 발급받을 수 있도록 다음과 같이 검증 메서드인 validateAlreadyIssued 를 구현했다.

```java

@RestController
@RequiredArgsConstructor
class MemberCouponApi {

    private final MemberCouponService memberCouponService;

    @PostMapping("/member-coupon")
    public ResponseEntity<Void> issueCoupon(@RequestBody IssueCouponRequest request) {
        memberCouponService.issue(request.memberId(), request.couponId());

        return ResponseEntity.noContent().build();
    }
}
```

```java

@Service
@RequiredArgsConstructor
public class MemberCouponService {

    // ... 중략 ...

    @Transactional
    public Long issue(Long memberId, Long couponId) {
        validateAlreadyIssued(memberId, couponId);
        Member member = memberRepository.findById(memberId).orElseThrow();
        Coupon coupon = couponRepository.findById(couponId).orElseThrow();
        MemberCoupon memberCoupon = MemberCoupon.issue(member, coupon);
        memberCouponRepository.save(memberCoupon);

        return memberCoupon.getId();
    }

    private void validateAlreadyIssued(Long memberId, Long couponId) {
        if (memberCouponRepository.existsMemberCouponByMemberIdAndCouponId(memberId, couponId)) {
            throw new IllegalStateException("해당 사용자는 이미 쿠폰을 발급했습니다.");
        }
    }
}
```

하지만, 사용자가 동시에 API에 요청을 보내게 된다면 1명의 사용자는 1개 이상의 쿠폰을 발급 받을 수 있게 된다.

### 1.3 쿠폰 발급 API 원인 간단하게 알아보기!

<p align="center">
    <img src="./tech-coupon-reason.png"/>
</p>

만약 사용자가 동시에 API에 요청을 보내게 된다면, 1개 이상의 스레드가 동시에 MemberCouponService의 issue 메서드를 읽게 된다.
이때, 각 스레드는 DB 트랜잭션을 커밋하기 이전이기 때문에 각 스레드는 모두 검증에 통과하고 결과적으로 1개 이상의 쿠폰을 발급 받을 수 있게 되는 것이다.

## 2. 해결책을 저울질하자

위 문제를 해결하기 위해서는 동시성을 희생시키는 모든 방식을 고려해볼 수 있다.
하지만, 여러 방식 중에서 현재 상황에 맞는 가장 효율적인 방식을 선택하는 것이 중요하다.
이를 위해서 다양한 접근 방식을 생각해 보고 비교해 볼 필요성이 있다.

### 2.1 처리율 제한를 이용한 해결 방식

처리율 제한 장치의 장점은 다음과 같다.

- DOS 공격에 의한 자원 고갈과 서버 과부하(사용자의 잘못된 사용 패턴, 봇 트래픽)를 방지한다.
- 서드파티 API 사용료 증가를 예방한다.

동시성 문제는 서버의 동시 처리 능력과 연관이 있다. 극단적으로 생각했을 때 서버의 스레드를 한 개로 제한하면 동시성 문제는 발생하지 않는다.  
처리율 제한 장치를 이용하여 특정 API의 동시 처리 능력을 희생시키면 동시성 문제가 발생하지 않고 처리율 제한의 이점도 얻어갈 수 있을 것이다.

#### 적용

```java

@RestController
@RequiredArgsConstructor
class MemberCouponApi {

    private final RateLimiter rateLimiter = RateLimiter.create(1);
    private final MemberCouponService memberCouponService;

    @PostMapping("/member-coupon")
    public ResponseEntity<Void> issueCoupon(@RequestBody IssueCouponRequest request) {
        if (rateLimiter.tryAcquire()) {
            memberCouponService.issue(request.memberId(), request.couponId());
            return ResponseEntity.noContent().build();
        }

        return ResponseEntity.status(HttpStatus.TOO_MANY_REQUESTS).build();
    }
}
```

Guava 라이브러리의 처리율 제한 기능을 사용했다. 100개 스레드에 같은 사용자, 같은 쿠폰의 ID로 동시에 쿠폰 발급 API에 요청했다.  
처리율 제한에 막히는 경우 429(Too Many Requests) 응답을 내려주고 요청을 무시한다. 따라서, 사용자는 쿠폰을 단 한 번만 발급할 수 있게 된다.

#### 장점과 한계

**장점** :

- 이미 시스템에 처리율 제한 장치가 있다면 적용이 유리할 수 있다. 예를 들어, 한 시스템에서 처리율 제한 장치가 이미 존재한다고 가정하자. 사용자 IP 별로 1초에 1번만 요청하도록 처리율을 제한할 수 있다.
- DB 커넥션을 점유하지 않고 동시성 문제를 해결한다.

**한계** :

- Guava를 이용한 구현의 경우, 분산된 서버에서 동시 요청이 들어온다면 첫 요청은 A 서버, 두번째 요청은 B서버로 가는 경우 동시성 문제가 발생한다.
- 단순 동시성 문제를 해결하기 위해서 도입하기에는 애매한 지점이 있다.
- 처리율 제한 장치 설계에 대한 고려가 필요하다.
- 동시성 문제 해결을 위한 처리율 제한 수치와 근본적으로 사용해야하는 처리율 제한의 수치가 다를 수 있다.

### 2.2 자바 동기화 도구를 이용한 해결 방식

자바에서는 sysncronized 키워드나 ReetrantLock와 같은 동기화 도구를 제공한다.
동기화 도구를 사용하면 한 스레드가 어떤 행위를 수행하고 있을 때, 다른 스레드를 대기시킬 수 있다.
따라서, 동시성 문제를 해결할 수 있는 대안이 될 수 있다.

#### 적용

```java

@Service
@RequiredArgsConstructor
public class MemberCouponService {

    // ... 중략 ...

    public synchronized Long issue(Long memberId, Long couponId) {
        return memberCouponIssuer.issue(memberId, couponId);
    }
}
```

기존 쿠폰 발급 로직을 MemberCouponIssuer 내부로 위임했다.
그리고, MemberCouponService issue 메서드에 synchronized 키워드를 추가했다.
이렇게 변경한 이유는 DB 트랜잭션의 커밋한 이후에 잠금을 해제하기 위함이다.

<p align="center">
    <img src="./tech-coupon-aop-proxy.png"/>
</p>

`@Transactional` 어노테이션이 추가된 메서드는 프록시 기반으로 동작한다.
우선 MemberCouponService issue 메서드를 호출하면 프록시가 요청을 받아 DB 트랜잭션을 진행하고 실제 객체를 호출한다.
따라서, 잠금이 해제되고 DB 트랜잭션이 커밋되므로 동시성 문제가 다시 발생할 수 있다. 이를 해결하기 위해서는 객체를 분리하거나 `@Transactional` 어노테이션을 제거할 수 있다.

#### 장점과 한계

**장점** :

- synchronized를 사용하는 경우 잠금 해제에 대한 고민을 하지 않아도 된다.
- 트랜잭션 없는 상위 계층에서 잠금 획득을 시도하기 때문에 DB 커넥션을 점유하지 않고 스레드가 대기한다.

**한계** :

- 분산 서버 환경에서 동시성 문제가 발생할 수 있다.

### 2.3 트랜잭션 격리 수준(READ_UNCOMMITED)을 이용한 해결 방식

다른 DB 트랜잭션이 커밋하기 이전에 검증을 통과하기 때문에 사용자는 쿠폰을 1개 이상 발급받을 수 있었다.
트랜잭션 격리 수준 READ_UNCOMMITED에서 발생하는 더티 리드를 이용하면 다른 스레드가 삽입한 데이터를 읽을 수 있기 때문에 동시성 문제를 해결할 수 있다.

#### 적용

```java

@Service
@RequiredArgsConstructor
public class MemberCouponService {

    // .. 중략 ..

    @Transactional(isolation = Isolation.READ_UNCOMMITTED)
    public Long issue(Long memberId, Long couponId) {
        Member member = memberRepository.findById(memberId).orElseThrow();
        Coupon coupon = couponRepository.findById(couponId).orElseThrow();
        MemberCoupon memberCoupon = MemberCoupon.issue(member, coupon);

        // 1. INSERT 쿼리가 발생한다.
        memberCouponRepository.save(memberCoupon);
        validateAlreadyIssued(memberId, couponId);

        return memberCoupon.getId();
    }

    // 2. 더티 리드를 이용해서 다른 트랜잭션에서 삽입한 데이터 알 수 있다.
    private void validateAlreadyIssued(Long memberId, Long couponId) {
        try {
            // 3. 가장 먼저 INSERT를 수행하고 조회를 한 스레드는 1개를 반환할 것이고, 나머지는 그 이상의 데이터를 반환하니 예외가 발생한다.
            memberCouponRepository.findByMemberIdAndCouponId(memberId, couponId);
        } catch (IncorrectResultSizeDataAccessException e) {
            // 4. 예외가 발생한 트랜잭션은 롤백된다.
            throw new IllegalStateException("해당 사용자는 이미 쿠폰을 발급했습니다.");
        }
    }
}
```

`@Transactional` 어노테이션에 isolation 속성을 READ_UNCOMMITED로 설정하고, 연산의 순서를 변경했다.
INSERT 쿼리를 가장 먼저 수행하고 그 이후에 검증 로직을 실행한다.
만약 1개 이상의 스레드가 동시에 데이터를 삽입하고 검증을 수행하면 충돌된 모든 스레드는 예외가 발생한다.

#### 장점과 한계

**장점** :

- 분산 서버 환경에서도 동시성 문제를 해결한다.

**한계** :

- 충돌한 모든 스레드가 실패한다.
- INSERT 쿼리가 바로 데이터베이스로 전송되어야 한다.
- 연산의 순서가 일반적이지 않아 다른 개발자에게 혼란을 야기한다.

### 2.4 MySQL 스토리지 잠금을 이용한 해결 방식

MySQL의 `select .. for update, shared`, 그리고 트랜잭션 격리 수준 SERIALIZABLE을 사용하면 MySQL 스토리지 잠금을 사용한다.  
이 잠금을 사용하면 다른 트랜잭션을 대기시킬 수 있으므로 동시성 문제를 해결할 수 있는 대안이 될 수 있다.

#### 적용

```java
public interface MemberCouponRepository extends JpaRepository<MemberCoupon, Long> {

    @Lock(LockModeType.PESSIMISTIC_READ)
    @QueryHints({@QueryHint(name = "javax.persistence.lock.timeout", value = "10000")})
    boolean existsMemberCouponByMemberIdAndCouponId(Long memberId, Long couponId);
}
```

`@Transactional`의 격리 레벨 설정을 SERIALIZABLE로 변경하거나 공유 혹은 배타 잠금 쿼리를 작성한다.
MySQL의 SERIALIZABLE은 내부적으로 단순 조회의 경우에도 잠금을 수행한다.
따라서 명시적으로 공유 잠금과 배타 잠금을 사용하는 쿼리를 사용하는 것이 상대적으로 나은 성능을 보인다.
따라서, 위 예시에서는 `@Lock` 어노테이션을 이용하여 공유 잠금을 설정했다.

위 메서드는 다음과 같은 쿼리가 발생한다.

```postgresql
select mc.id
from member_coupon mc
where mc.member_id = 2
  and mc.coupon_id = 2
limit 1 for share
```

이 경우 S, GAP 잠금이 확인할 수 있다. 만약 member_coupon 테이블에 member_coupon(1, 2), member_coupon(6, 2) 조합인 두 개의 레코드가 존재한다면,
member_id가 2보다 큰 6을 기준으로 S, GAP 잠금이 걸린다. 따라서 member_id가 1부터 5인 INSERT 쿼리를 대기시키므로 동시성 문제를 해결할 수 있다.

#### 장점과 한계

**장점** :

- 분산 서버 환경에서도 동시성 문제를 해결한다.
- 다른 방식에 비해 적용이 편리하다.

**한계** :

- 불필요한 공간까지 잠금하기 때문에 상대적인 성능 저하가 발생한다.
- 데드락을 야기할 수 있다.

### 2.5 유니크 제약 조건을 이용한 해결 방식

유니크 제약 조건을 사용하면 중복 저장이 불가능하다. 따라서, 동시 호출이 발생하더라도 실제 테이블에는 1건만 저장될 수 있으니 동시성 문제를 해결할 수 있는 대안이 될 수 있다.

#### 적용

```java

@Entity
@Getter
@Table(uniqueConstraints = {
        @UniqueConstraint(
                name = "unique_member_coupon",
                columnNames = {"member_id", "coupon_id"}
        )
})
@AllArgsConstructor
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class MemberCoupon {

    // ... 중략 ...
}
```

JPA에서는 위와 같이 인덱스 제약을 추가할 수 있다. 이 경우 중복 저장이 불가하므로 한 사용자는 쿠폰을 한 번만 받을 수 있다.

#### 장점과 한계

**장점** :

- 분산 서버 환경에서도 동시성 문제를 해결한다.
- 적용이 간단하다.
- 잠금을 관리하기 위한 추가적인 작업이 필요하지 않다.

**한계** :

- 기획적인 한계가 있을 수 있다. 가령, 한 사용자는 5개 쿠폰을 발급받을 수 있다면 유니크 인덱스는 적절하지 않을 수 있다.
- 비즈니스 제약 조건이 DB에 의존적인 것은 단점일 수 있다.

### 2.6 분산 잠금(MySQL 네임드락)을 이용한 해결 방식

```mysql
select get_lock('mylock', 2); # n 초동안 잠금 획득을 시도한다.
select is_free_lock('mylock'); # 잠금을 획득할 수 있는지 확인한다.
select is_used_lock('mylock'); # 사용되고 있는 잠금인지 체크한다.
select release_lock('mylock'); # 특정 잠금을 해제한다.
select release_all_locks(); # 세션에서 획득한 모든 잠금을 해제한다.
```

MySQL의 네임드락 기능을 활용하여 분산 잠금을 구현할 수 있다. 네임드락은 임의의 문자열에 잠금을 거는 기능이며, 위와 같이 사용할 수 있다.
네임드락은 몇 가지 특징이 있다. 우선 한 세션에서 잠금을 유지하고 있으면, 다른 세션에서 해당 잠금을 획득할 수 없다.
그리고, 획득한 잠금은 트랜잭션이 종료되어도 해제되지 않는다.
해제의 경우는 현재 세션에서 획득한 잠금만 릴리즈할 수 있다. 이러한 네임드락의 특징을 활용하여 분산 잠금을 구현하면 동시성 문제를 해결할 수 있다.

#### 적용

```java

@Service
@RequiredArgsConstructor
public class MemberCouponService {

    // ... 중략 ...

    public Long issue(Long memberId, Long couponId) {
        String key = memberId + "-" + couponId;

        DataSource lockDataSource = getLockDataSource();
        try (Connection connection = lockDataSource.getConnection()) {
            distributedLock.tryLock(connection, key, 3);
            try {
                return memberCouponIssuer.issue(memberId, couponId);
            } finally {
                distributedLock.releaseLock(connection, key);
            }
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }

    private DataSource getLockDataSource() {
        return applicationContext.getBean(DataSourceConfig.LOCK_DATA_SOURCE, DataSource.class);
    }
}
```

```java

@Component
public class MySqlDistributedLock {

    public void tryLock(Connection connection, String key, int timeout) {
        String sql = "select get_lock(?, ?)";

        try (PreparedStatement preparedStatement = connection.prepareStatement(sql)) {
            preparedStatement.setString(1, key);
            preparedStatement.setInt(2, timeout);
            preparedStatement.execute();
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }

    public void releaseLock(Connection connection, String key) {
        String sql = "select release_lock(?)";

        try (PreparedStatement preparedStatement = connection.prepareStatement(sql)) {
            preparedStatement.setString(1, key);
            preparedStatement.execute();
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
}
```

MySqlDistributedLock 클래스는 잠금을 획득하고 해제하는 역할을 가진다.
이때 tryLock, releaseLock 메서드에서 커넥션을 주입받도록 구현했다. 왜냐하면, 잠금을 획득한 커넥션으로 잠금을 해제해야 하기 때문이다. 
만약, 그렇지 않는다면 잠금 해제가 실패할 것이고 더 나아가 커넥션 풀링을 하는 경우에는 다른 스레드가 잠금을 획득한 커넥션을 사용하여 예기치 못한 상황이 발생할 수 있다.

MemberCouponService 내부를 확인하면 DataSource 또한 분리했다. MemberCouponIssuer의 issue 메서드에서 하나의 커넥션을 점유한다. 그리고, 잠금을 획득하는 부분에서도 하나의 커넥션을 사용한다. 
만약, 10개의 커넥션이 풀에 존재한다고 가정하자. 10개의 스레드가 동시에 잠금을 획득하는 커넥션 획득하면 각 스레드가 issue 메서드를 실행할 커넥션을 얻을 수 없기 때문에 커넥션 풀 데드락이 발생한다. 
이를 예방하기 위해서 DataSource를 분리했다.

또한, 잠금을 수행하기 위해서는 대기가 필요할 수 있다. 이때 커넥션을 점유하고 대기를 수행하기 때문에 커넥션 풀이 고갈이 발생하여 전체 서비스의 장애로 이어질 수 있다. 
따라서 리소스를 분리하여 서비스 전체의 장애로 퍼지는 것을 어느 정도 완화할 수 있다.

번외로 Spring Data Jpa를 이용하면 아래와 같이 더욱 간단하게 구현할 수 있다. 
하지만, OSIV(Open Session In View) 옵션이 비활성화되어 있는 경우에는 잠금을 점유한 커넥션과 다른 커넥션으로 잠금을 해제할 수 있기 때문에 주의해야한다.

```java

@Service
@RequiredArgsConstructor
public class MemberCouponService {

    // ... 중략 ...

    public Long issue(Long memberId, Long couponId) {
        // lock 메서드는 MysqlLockRepository를 사용한다.
        memberCouponIssueLock.lock(memberId, couponId);
        try {
            return memberCouponIssuer.issue(memberId, couponId);
        } finally {
            memberCouponIssueLock.unlock(memberId, couponId);
        }
    }
}
```

```java
public interface MySqlLockRepository extends JpaRepository<MemberCoupon, Long> {

    @Query(value = "select get_lock(:key, 3000)", nativeQuery = true)
    void getLock(String key);

    @Query(value = "select release_lock(:key)", nativeQuery = true)
    void releaseLock(String key);
}
```

#### 장점과 한계

**장점** :

- 분산 서버 환경에서도 동시성 문제를 해결한다.
- 상대적으로 작은 잠금의 범위를 가진다.
- 기존에 MySQL을 운영하고 있는 경우 추가 비용 없이 구축이 가능하다.

**한계** :

- MySQL 기능에 의존적인 방식이며, 다른 DB로 변경되는 경우 한계가 있다.
- DB 커넥션을 점유하고 스레드가 대기하는 비효율이 생긴다.
- 잠금 획득과 해제를 위한 데이터베이스 추가 요청이 발생한다.

### 2.7 분산 잠금(Redis)을 이용한 해결 방식

Redis는 분산 잠금과 아토믹 연산을 지원한다. 따라서, Redis를 활용하여 동시성 문제를 해결할 수 있다. Redis 클라이언트 별로 분산 잠금을 사용하는 양상이 다르다. 
분산 잠금을 사용하기 위한 Redis 클라이언트는 대표적으로 Lettuce와 Redission이 있다. Lettuce의 경우에는 따로 지원해 주는 것이 없기 때문에 SETNX 명령어를 이용해 직접 구현해야 하며, 
Redisson의 경우에는 RLock이라는 클래스를 통해서 분산 잠금을 사용할 수 있도록 지원한다.

#### 적용

Lettuce 구현 방식은 다음과 같다. 잠금을 획득하는데 필요한 타임아웃을 직접 구현해야 한다. 
스핀락 방식으로 Redis에 부하를 주니 Thread.Sleep을 추가했다. 
잠금 해제의 경우 키에 해당되는 값을 제거하면 된다.

```java

@Component
@RequiredArgsConstructor
class LettuceMemberCouponIssueLock implements MemberCouponIssueLock {

    // ... 중략 ...

    @Override
    public void lock(Long memberId, Long couponId) {
        int tryCount = 10;

        tryLockWithSpin(memberId, couponId, tryCount);
    }

    private void tryLockWithSpin(Long memberId, Long couponId, int tryCount) {
        while (!requestLock(memberId, couponId)) {
            if (tryCount-- == 0) {
                // lock 획득 실패 처리
                throw new RuntimeException();
            }

            try {
                // redis에 너무 많은 부하를 주지 않기 위해 sleep을 설정
                Thread.sleep(100);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        }
    }

    private boolean requestLock(Long memberId, Long couponId) {
        Boolean result = redisTemplate
                .opsForValue() // opsForX는 커맨드를 호출할 수 있는 기능을 모은 인터페이스를 반환
                .setIfAbsent(generateKey(memberId, couponId), "empty", Duration.ofSeconds(3));

        return Boolean.TRUE.equals(result);
    }
}
```

Redisson은 RLock 클래스를 제공한다. 이는 타임아웃과 같은 설정을 지원하며 Lettuece 방식에 비해 편리하다. 
Pub/Sub 방식으로 락이 해제되면 잠금을 구독하는 클라이언트에게 신호를 전달하는 방식으로 작동한다.

```java

@Component
@RequiredArgsConstructor
class RedissonMemberCouponIssueLock implements MemberCouponIssueLock {

    @Override
    public void lock(Long memberId, Long couponId) {
        RLock lock = redissonClient.getLock(generateKey(memberId, couponId));
        try {
            boolean acquired = lock.tryLock(5, TimeUnit.SECONDS);
            if (!acquired) {
                // lock 획득 실패 처리
                throw new RuntimeException();
            }
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
    }

    @Override
    public void unlock(Long memberId, Long couponId) {
        RLock lock = redissonClient.getLock(generateKey(memberId, couponId));
        lock.unlock();
    }

    private String generateKey(Long memberId, Long couponId) {
        return "memberCoupon-" + memberId.toString() + couponId.toString();
    }
}
```

#### 장점과 한계

**장점** :

- 분산 서버 환경에서도 동시성 문제를 해결한다.
- 기존에 Redis를 운영하고 있는 경우 추가 비용 없이 구축이 가능하다.
- DB 커넥션을 점유하고 대기하지 않아도 된다.
- MySQL 분산 잠금에 비해서 신경 써야 할 부분이 적다.

**한계** :

- Redis에 대한 인프라 비용, 유지보수 비용이 추가로 발생하므로 단순 동시성 문제 해결을 위한 방법으로는 적합하지 않을 수 있다.

## 3. 고민해 볼 지점

동시성 문제를 해결하기 위해서 여러 대안을 생각해 봤다. 하지만, 동시성 문제를 해결하기 위해서는 몇 가지 추가적으로 고민해 볼 부분들이 존재한다.

- 추가적인 인프라 구축 비용을 발생시킬 수 있다.
- 병목지점을 만들 수 있다.
- 상황에 따라서 데드락을 발생 시킬 수 있다.
- 코드의 복잡도를 증가시킬 수 있다.
- 막지 않아도 괜찮을 수도 있다.

위와 같은 부분들을 충분히 고민해 봤는데도 꼭 막아야 하는 경우도 있을 것이다.
이러한 경우에는 오늘 접근해 본 방식보다 나은 대안이 있을 것이라 생각하고 끊임없이 탐구하는 자세가 필요하다.

## 참고

도서

- 데이터베이스 개론
- Real Mysql 8.0
- 자바 병렬 프로그래밍
- 자바 성능 튜닝 이야기
- 자바의 정석
- 자바 ORM 표준 JPA 프로그래밍
- 가상 면접 사례로 배우는 대규모 시스템 설계 기초

공식 문서

- MySQL 공식 문서
- Guava 공식 문서
- Spring Data Redis 공식 문서

기술 블로그

- [와디즈 기술 블로그 - 분산 환경 속에서 ‘따닥’을 외치다](https://blog.wadiz.kr/%EB%B6%84%EC%82%B0-%ED%99%98%EA%B2%BD-%EC%86%8D%EC%97%90%EC%84%9C-%EB%94%B0%EB%8B%A5%EC%9D%84-%EC%99%B8%EC%B9%98%EB%8B%A4/)
- [채널톡 기술 블로그 - Distributed Lock 구현 과정](https://channel.io/ko/blog/distributedlock_2022_backend)
- [요기요 기술 블로그 - DB Concurrency 어디까지 알고 있니](https://techblog.yogiyo.co.kr/db-concurrency-%EC%96%B4%EB%94%94%EA%B9%8C%EC%A7%80-%EC%95%8C%EA%B3%A0-%EC%9E%88%EB%8B%88-559bfc4f59ee)
- [우아한 기술 블로그 - MySQL을 이용한 분산락으로 여러 서버에 걸친 동시성 관리](https://techblog.woowahan.com/2631/)
- [우아한 기술 블로그 - WMS 재고 이관을 위한 분산 락 사용기](https://techblog.woowahan.com/17416/)
- [우아한 기술 블로그 - HikariCP Dead lock에서 벗어나기 (이론편)](https://techblog.woowahan.com/2664/)
- [우아한 기술 블로그 - HikariCP Dead lock에서 벗어나기 (실전편)](https://techblog.woowahan.com/2663/)
- [당근 기술 블로그 - MySQL Gap Lock 다시 보기](https://medium.com/daangn/mysql-gap-lock-%EB%8B%A4%EC%8B%9C%EB%B3%B4%EA%B8%B0-7f47ea3f68bc)
- [당근 기술 블로그 - MySQL Gap Lock (두 번째 이야기)](https://medium.com/daangn/mysql-gap-lock-%EB%91%90%EB%B2%88%EC%A7%B8-%EC%9D%B4%EC%95%BC%EA%B8%B0-49727c005084)
- [컬리 기술 블로그 - 풀필먼트 입고 서비스팀에서 분산락을 사용하는 방법 - Spring Redisson](https://helloworld.kurly.com/blog/distributed-redisson-lock/)
- [하이퍼커넥트 기술 블로그 - 레디스와 분산 락(1/2) - 레디스를 활용한 분산 락과 안전하고 빠른 락의 구현](https://hyperconnect.github.io/2019/11/15/redis-distributed-lock-1.html)