# 결합도는 낮추고 확장성은 높인 뱃지 시스템 설계 이야기

## 목차

1. **도입**  
   1.1. 배경: 왜 뱃지 기능을 도입했는가?  
   1.2. 초기 뱃지 정책

2. **아키텍처 설계 및 고민**  
   2.1. 핵심 고민: 도메인 간의 결합도  
   2.2. 잘못된 접근 방식의 예  
   2.3. 설계 원칙: "다른 도메인은 뱃지의 존재를 몰라야 한다"  
   2.4. 해결책: 이벤트 기반 아키텍처 도입

3. **비동기 이벤트 처리 분석**  
   3.1. `BadgeEventListener`의 역할  
   3.2. 데이터 정합성을 위한 선택: `@TransactionalEventListener`  
   3.3. 사용자 경험 최적화: `@Async`를 통한 응답 시간 단축  
   3.4. 트랜잭션 분리: `@Transactional(propagation = REQUIRES_NEW)`

4. **확장성을 고려한 디자인 패턴 적용**  
   4.1. Registry 패턴: 효율적인 정책 탐색  
   4.2. Interface와 OCP: 새로운 뱃지 정책의 유연한 확장  
   4.3. 역할 분리(SRP): `BadgeAwardService`의 도입  
   4.4. `BadgeEvent` Interface 설계

5. **템플릿 메서드 패턴을 이용한 중복 제거**

6. **결론**

---
# 1. 도입

### 1.1. 배경: 왜 뱃지 기능을 도입했는가?

서비스에 뱃지 기능을 추가한 주된 이유는 **사용자 경험(UX)을 향상**시키고, **서비스 참여도를 극대화**하기 위함입니다.  
사용자는 특정 목표를 달성했을 때 시각적인 보상을 받음으로써 성취감을 느끼고,  
이는 서비스에 대한 만족도와 몰입도를 자연스럽게 높이는 요소로 작용합니다.

#### 주요 목표
- **성취감 부여**: 특정 목표를 달성한 사용자에게 시각적인 보상(뱃지)을 제공하여 성취감을 느끼게 합니다.  
  이는 서비스에 대한 긍정적인 경험을 강화합니다.
- **기능 사용 유도**: 뱃지 획득 조건을 통해 사용자들이 평소에 잘 사용하지 않았던 기능(예: 채팅 등)을  
  자연스럽게 탐색하고 사용하도록 유도합니다. 이를 통해 서비스 전반의 활성화를 기대할 수 있습니다.

---

### 1.2. 초기 뱃지 정책

사용자의 핵심적인 활동을 기반으로 다음과 같이 **5개의 뱃지**를 기획했습니다.  
각 뱃지는 사용자의 서비스 여정에서 **의미 있는 이정표**가 되도록 설계되었습니다.

| **뱃지 이름** | **획득 조건** | **부여 의미** |
|:--------------:|:--------------|:---------------|
| 🥇 리드오프 | 최초 회원가입 | 사용자의 서비스 여정 시작을 환영 |
| ⚾ 플레이볼 | 최초 직관 인증 | 서비스 핵심 기능의 첫 사용을 기념 |
| 💬 말문이 트이다 | 최초 채팅 입력 | 커뮤니티 기능에 처음으로 참여 |
| 🗣️ 공포의 주둥아리 | 채팅 100회 입력 | 커뮤니티 기능에 숙련된 사용자임을 인증 |
| 🏆 그랜드 슬램 | KBO 리그 9개 구장 모두 방문 | 최고 수준의 도전 과제 달성을 축하 |

---

# 2. 아키텍처 설계 및 고민

### 2.1. 핵심 고민: 도메인 간의 결합도
기능을 구현하면서 마주한 가장 큰 기술적 고민은 "뱃지(`Badge`) 도메인이 다른 도메인과 어떻게 상호작용 해야하는가?"였습니다.
회원가입은 `Member` 도메인, 직관 인증은 `CheckIn` 도메인 그리고 채팅은 `Chat` 도메인에서 발생합니다.
만약 이 모든 도메인 서비스가 뱃지를 발급하기 위해 `BadgeService`를 직접 호출한다면, 각 도메인은 `Badge` 도메인의 존재를 알아야만 합니다.
이는 **도메인 간의 강한 결합**을 발생시켜 다음과 같은 문제를 야기합니다.
- **유지보수성 저하**: 뱃지 정책이 변경될 때마다 `Member`, `CheckIn`, `Chat` 등 관련된 모든 도메인의 코드를 수정해야 할 수 있습니다.
- **책임 불균형**: `CheckIn` 도메인의 핵심 책임은 '직관 인증'이지, '뱃지 발급'이 아닙니다. 즉, 직관 인증이 뱃지 발급되는 것을 알 필요가 없습니다.

### 2.2. 잘못된 접근 방식의 예
만약 이러한 고민 없이 기능을 구현했다면, `CheckInService`는 아래와 같이 `BadgeService`를 직접 의존하는 형태가 되었을 것입니다.
```
@Service
@RequiredArgsConstructor
public class CheckInService {

    private final CheckInRepository checkInRepository;
    private final BadgeService badgeService; // 😱 BadgeService에 직접 의존

    @Transactional
    public void createCheckIn(final Long memberId, final CreateCheckInRequest request) {
        // ... 직관 인증 관련 핵심 로직 ...
        CheckIn checkIn = new CheckIn(game, member, team);
        checkInRepository.save(checkIn);

        // 이벤트 발행 대신, '직관 인증 시 뱃지를 줘야 한다'는 자신의 핵심 책임이 아닌 일에 대해 알고 있어야 합니다.
        badgeService.awardStadiumVisitBadges(member, stadiumId);
    }
}
```

### 2.3. 설계 원칙: "다른 도메인은 뱃지의 존재를 몰라야 한다"
위 문제를 해결하기 위해 **다른 도메인들은 뱃지의 존재를 몰라야 한다**는 도메인 분리 원칙을 세웠습니다.

### 2.4. 해결책: 이벤트 기반 아키텍처 도입
도메인 간의 결합을 끊어내기 위해 `ApplicationEventPublisher`를 사용한 이벤트 기반 아키텍처를 도입했습니다.
이제 `CheckInService`는 자신의 핵심 로직을 처리한 후, `BadgeService`를 직접 호출하는 대신 **경기장에 방문했다**라는 사실(Event)만 발행(Publish)합니다.
뱃지 발급에 대한 책임은 이 이벤트를 구독(Subscribe)하는 별도의 리스너에게 위임됩니다.
```
@Service
@RequiredArgsConstructor
public class CheckInService {

    private final CheckInRepository checkInRepository;
    // BadgeService 대신 이벤트 발행자를 주입받습니다.
    private final ApplicationEventPublisher eventPublisher;

    @Transactional
    public void createCheckIn(final Long memberId, final CreateCheckInRequest request) {
        // ... 직관 인증 관련 핵심 로직 ...
        checkInRepository.save(checkIn);

        // 인증이 발생했다는 이벤트만 알려줌! Badge의 존재를 전혀 모름.
        eventPublisher.publishEvent(new StadiumVisitEvent(member, stadiumId));
    }
}
```
이 구조를 통해 `CheckIn`도메인은 `Badge`도메인으로부터 분리되어 자신의 책임에만 집중할 수 있게 되었습니다.

---

# 3. 비동기 이벤트 처리 분석

### 3.1. `BadgeEventListener`의 역할
`BadgeEventListener`는 `CheckIn`, `Chat`등 다른 도메인에서 발행된 이벤트를 수신하여, 해당 이벤트가 특정 뱃지 발급 조건에 부합하는지 판단하고 실제 발급 로직을 트리거하는 책임을 가집니다.
이 과정에서 데이터 정합성, 성능, 트랜잭션 관리를 위해 다음과 같은 어노테이션 전략을 사용했습니다.
```
@Component
public class BadgeEventListener {

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    @Async
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void handleBadgeEvent(final BadgeEvent event) {
        // ... 뱃지 발급 로직 ...
    }
}
```
### 3.2. 데이터 정합성을 위한 선택: `@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)`
- **선택 이유**: 이벤트를 발생시킨 원래 작업의 트랜잭션이 성공적으로 커밋되어 DB에 완전히 저장된 후에만 뱃지 로직을 실행하기 위해 사용했습니다.
- **기대 효과**: 만약 이 옵션이 없다면, 직관 인증 정보가 DB에 저장되기 전에 뱃지가 먼저 발급되는 **데이터 불일치 문제**가 발생할 수 있습니다.
default 값이 `phase = TransactionPhase.AFTER_COMMIT`인데, 명시한 이유는 가독성이 높아지기 때문입니다.

### 3.3. `@Async`를 통한 응답 시간 단축
- **문제점**: 동기 처리로 인한 응답 시간 저하가 발견되었습니다. 직관 인증 API 호출 시, 사용자의 핵심 요청인 **인증 처리**와는 별개인 **뱃지 부여**로직까지 동기적으로 실행되었습니다.
이로 인해 사용자는 즉시 알 필요 없는 후속 작업이 끝날 때까지 대기해야 했습니다. Postman으로 측정한 결과, **평균 1.4s** (**1400ms**)의 응답 시간이 소요되어 직접적인 원인이 되고 있었습니다.

- **해결 방안**: `@Async` 어노테이션을 도입하여 뱃지 발급 로직을 별도의 스레드에서 비동기적으로 처리하도록 아키텍처를 개선했습니다.
이를 통해 인증 요청에 대한 응답은 사용자에게 즉시 전송하고, 시간이 소요될 수 있는 뱃지 부여 작업은 독립적으로 처리합니다.

- **성과**: 비동기 처리 도입 후, 동일한 API의 응답 시간은 **평균 144ms**로 단축되었습니다. 이는 기존 대비 약 10배 향상된 속도이며, 사용자가 체감하는 대기 시간을 **90%** 이상 감소시킨 결과입니다.

**스레드 풀 구성으로 안정성 확보**

`@Async`도입 후, "기본 `Async`는 매번 새로운 스레드를  생성하여 리소스 낭비가 발생할 수 있지 않을까?"라는  추가적인 고민이 들었습니다.
이 문제를 해결하고 스레드를 효율적으로 재사용하며, 시스템 안정성까지 확보하기 위해 별도의 스레드 풀을 직접 설정했습니다.
```
@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean(name = "badgeAsyncExecutor")
    public Executor badgeAsyncExecutor() {
        ThreadPoolTaskExecutor taskExecutor = new ThreadPoolTaskExecutor();
        taskExecutor.setCorePoolSize(2);
        taskExecutor.setMaxPoolSize(4);
        taskExecutor.setQueueCapacity(100);
        taskExecutor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        taskExecutor.setThreadNamePrefix("Badge-Thread-");
        taskExecutor.initialize();
        return taskExecutor;
    }
}
```
- **설정 의도**
  - `CorePoolSize`/`MaxPoolSize`: 운영 서버의 CPU 코어 수에 맞춰 평상시와 요청 급증 시에 스레드를 유연하게 사용하도록 설정했습니다.
  - `QueueCapacity`: 핵심 스레드가 모두 바쁠 때, 최대 100개의 작업을 대기시킬 수 있는 버퍼를 만들었습니다.

**🤔 고민: 스레드 풀이 가득 찼을 때**

"모든 스레드가 일하고 있고, 대기 큐까지 꽉 찼을 때 새로 들어온 이벤트는 어떻게 되지?"
기본 정책대로라면 이 이벤트는 거부되어 **데이터가 손실될 위험**이 있었습니다. 처음에는 "손실되는 이벤트는 로그로 남겨서 개발자가 수동으로 처리하자"
라고 생각했지만, `EventListener`가 이 이벤트를 받기도 전에 스레드 풀에서 거부되어서 로그조차 남길 수 없다는 것을 깨달았습니다.

그래서, "거부하지 말고, 그 요청만큼은 동기로 처리하면 어떨까?"라는 아이디어를 떠올렸습니다.

**해결책**: `CallerRunsPolicy`와 데이터 무결성 보장
`setRejectedExecutionHandler`에 `CallerRunsPolicy`정책을 적용했습니다. 이 정책은 스레드 풀이 포화 상태일 때, 새로운 작업을 버리는 대신
**이벤트를 발행하는 원래 스레드(Caller)가 직접 그 작업을 처리**하도록 만듭니다.

이 방식은 어떤 상황에서도 이벤트가 버려지지 않고 반드시 처리되도록 보장하여 데이터 무결성을 지켜줍니다. 흥미로운 점은, 대기 큐에서 100번째로 기다리던
작업보다, 큐가 꽉 찬 직후에 들어온 101번째 작업이 '새치기'처럼 먼저 처리될 수 있다는 점입니다. 대기 중인 100개의 작업은 스레드의 순서를 기다리지만,
101번째 작업은 메인 스레드가 즉시 동기 방식으로 처리하기 때문입니다.

**결론: 두 마리 토끼를 잡는 유연한 비동기 전략**

이러한 스레드 풀 설정을 통해 두 마리 토끼를 모두 잡을 수 있었습니다. 🐇🐇
평상시에는 비동기 처리의 이점을 최대한 활용하여 **사용자에게 즉각적인 응답**을 제공합니다. 동시에, 트래픽이 폭주하여 스레드 풀이 포화가 되는 예외적인 상황에서는
`CallerRunsPolicy`를 통해 **동기 방식으로 유연하게 전환**됩니다.

이를 통해 이벤트 유실 없이 데이터 무결성을 보장할 수 있었습니다. 흥미롭게도 이 방식은 단순히 데이터를 지키는 것을 넘어, 대기 큐에서
자신의 차례를 기다리는 작업보다 오히려 더 빠르게 처리될 가능성까지 열어두었습니다.

결론적으로, 비동기로 처리할 수 있는 부분은 사용자에게 빠른 응답을 줌과 동시에, 스레드가 포화 상태일 때는 적절하게 동기 방식을 택해
데이터 무결성을 지키고 처리 지연을 초소화하는 시스템을 구축할 수 있었습니다.

### 3.4. 트랜잭션 분리: `@Transactional(propagation = Propagation.REQUIRES_NEW)`
- **문제 상황**: `TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)`으로 리스너를 실행하면, 리스너는 이벤트를 발생시킨 기존 트랜잭션의 영속성 컨텍스트는 물려받지만,
해당 트랜잭션은 이미 커밋되어 **'읽기 전용 상태'**가 됩니다. 이 상태에서 뱃지 정보를 DB에 쓰기 작업하려고 시도하자 쓰기 작업 실패가 발생했습니다.

- **첫 번째 해결 시도와 한계**:
각 뱃지 정책(`BadgePolicy`)의 `award`메서드에 `@Transactional(propagation = REQUIRES_NEW)`를 붙이는 방법입니다. 이 방법은 동작은 했지만,
하나의 이벤트로 여러 뱃지(예: 첫 채팅, 100회 채팅)가 발급되는 경우, **불필요하게 여러 개의 새로운 트랜잭션이 생성되고 커밋**되는 비효율성이 있었습니다.
또한, 트랜잭션 관리 책임이 여러 클래스에 분산되는 문제도 있었습니다.

- **최종 결정**: **리스너를 트랜잭션 경계로 설정**:
"이벤트 처리"라는 전체 과정을 하나의 작업 단위로 보고, 이 전체를 감싸는 단 하나의 트랜잭션을 만드는 것이 더 올바른 설계라고 판단했습니다.
따라서 `BadgeEventListener`의 `handleBadgeEvent` 메서드에 `@Transactional(propagation = REQUIRES_NEW)`를 적용했습니다.

- **최종 구조의 이점**:
   - **단일 트랜잭션**: 리스너가 호출될 때 쓰기 가능한 새로운 트랜잭션이 단 한 번 생성됩니다.
   - **중앙화된 관리**: 트랜잭션의 시작과 끝이 이벤트 리스너로 명확하게 중앙화되어, 코드의 구조가 단순해지고 전체 작업의 원자성을 보장하기 쉬워졌습니다.

---

# 4. 확장성을 고려한 설계

### 4.1. Registry 패턴: 효율적인 정책 탐색
- **문제점**: 초기에는 `BadgeEventListener`가 모든 `BadgePolicy` 구현체들을 `List`로 주입받아,
이벤트가 발생할 때마다 반복문을 통해 "이 이벤트를 처리할 수 있니?"를 일일이 물어봐야 했습니다.
이는 최악의 경우 모든 정책을 순회해야 하는 **비효율성**을 야기합니다.

- **해결책**: **Registry 패턴 도입:**
이런 문제를 해결하기 위해, 어떤 정책이 어떤 구현체에 의해 처리되는지 미리 등록해두고 찾아주는 관리자 역할의 `BadgePolicyRegistry`를 도입했습니다.
이는 우아한테크코스 Lv4 MVC 미션 중 Spring의 `DispatcherServlet`이 `HandlerMapping`을 통해 요청을 처리할 컨트롤러를 찾는 원리에서 아이디어를 얻었습니다.
```
// Registry: 정책과 구현체를 매핑하여 관리
@Component
public class BadgePolicyRegistry {
    private final Map<Policy, BadgePolicy> policyMap = new HashMap<>();

    public void register(final Policy policy, final BadgePolicy badgePolicy) {
        policyMap.put(policy, badgePolicy);
    }

    public BadgePolicy getPolicy(final Policy policy) { // O(1) 조회
        return policyMap.get(policy);
    }
}
```
각 `BadgePolicy` 구현체는 Spring Bean으로 생성된 직후 `@PostConstruct`를 통해 자기 자신을 `BadgePolicyRegistry`에 등록합니다.
- **도입 효과:**
  - **성능 향상**: 불필요한 반복문이 사라지고 `Map`을 이용한 **O(1)** 조회로 변경되었습니다.
  - **책임 분리**: 정책 클래스는 순수하게 자신의 비즈니스 로직에만 집중하게 되었습니다.
  - **유지보수 용이성**: 새로운 뱃지 정책이 추가되어도 `BadgeEventListener`의 코드는 변경할 필요가 전혀 없습니다.

### 4.2. Interface와 OCP: 새로운 뱃지 정책의 유연한 확장
뱃지 기능은 확장될 가능성이 매우 높은 기능입니다. 따라서 설계 단계에서부터 **OCP**를 최우선으로 고려했습니다.

- **문제 상황 (OCP 위반):**
만약 `BadgePolicy`를 인터페이스로 만들지 않고, `BadgeEventListener`가 구체적인 클래스 타입을 다루게 된다면 새로운 정책이 추가될 때마다 리스너 코드를 수정해야 합니다.
```
// Bad Case: OCP 위반
public void handleBadgeEvent(final BadgeEvent event) {
    Object policyObject = badgePolicyRegistry.getPolicy(event.policy());

    if (policyObject instanceof ChatBadgePolicy) {
        // ... ChatBadgePolicy에 맞는 처리 ...
    } else if (policyObject instanceof CheckInBadgePolicy) {
        // ... CheckInBadgePolicy에 맞는 처리 ...
    }
    // 새로운 정책이 추가될 때마다 이곳에 else if를 계속 추가해야 함
}
```

- **해결책 (OCP 준수):**
`BadgePolicy`라는 인터페이스를 통해 모든 정책을 동일한 타입으로 바라보도록 설계했습니다.
`BadgeEventListener`는 상대방이 `ChatBadgePolicy`인지 `CheckInBadgePolicy`인지 전혀 알 필요 없이, 오직 `BadgePolicy`라는 규격에 맞춰 메시지만 전달할 뿐입니다.
```
// OCP 준수 (현재 설계 구조)
public void handleBadgeEvent(final BadgeEvent event) {
    // 리스너는 상대가 BadgePolicy라는 사실 외에는 아무것도 모름
    BadgePolicy policy = badgePolicyRegistry.getPolicy(event.policy());

    // 어떤 종류의 정책이 오든, 리스너의 코드는 동일함
    BadgeAwardCandidate candidate = policy.determineAwardCandidate(event);

    if (candidate != null) {
        badgeAwardService.award(candidate);
    }
}
```
이 구조 덕분에, 향후 수십 개의 새로운 뱃지 정책이 추가되더라도 `BadgeEventListener`의 코드는 **단 한 줄도 수정할 필요가 없습니다.**

---

### 4.3. 역할 분리: `BadgeAwardService`의 도입

- **문제 발견:**  
  초기 설계에서는 "뱃지 발급 자격 판단" 로직과 "실제 뱃지 발급" 로직이 모두 `BadgePolicy` 구현체 내부에 있었습니다.  
  하지만 실제 코드를 작성해보니, '판단' 로직은 정책마다 달랐지만, '실행(발급)' 로직은 모든 정책에서 완전히 동일했습니다.  
  이로 인해 각 `BadgePolicy` 클래스에 똑같은 `award` 메서드 코드가 중복되는 문제가 있었습니다.

- **리팩터링:**  
  SRP(단일 책임 원칙)에 따라 '판단'과 '실행'이라는 두 가지 다른 책임을 분리하기로 결정했습니다.  
  모든 정책에 공통된 '실행' 로직을 별도의 `BadgeAwardService`로 추출했습니다.

- **최종 구조의 이점:**
    - `BadgePolicy`: "이 사용자에게 뱃지를 줄지 말지 **판단**만 할게"
    - `BadgeAwardService`: "결정된 내용을 받아서 DB에 **저장**만 할게"  
      이 리팩터링을 통해 코드 중복이 사라져 유지보수성이 향상되었습니다.  
      만약 나중에 뱃지를 발급할 때마다 사용자에게 알림을 보내는 기능이 추가된다면, 여러 정책 클래스를 수정할 필요 없이 `BadgeAwardService` 단 한 곳만 수정하면 됩니다.

---

### 4.4. `BadgeEvent` Interface 설계

- **도메인 간 결합도 분리:**  
  이벤트를 발행하는 `CheckIn`이나 `Chat` 도메인이 `Badge` 도메인의 내부 사정을 알 필요가 없도록,  
  `Badge` 패키지에 공통 규격인 `BadgeEvent` 인터페이스를 두었습니다.  
  각 도메인은 자신의 패키지 안에서 구체적인 DTO(예: `CheckInEvent`)를 정의하고, `BadgeEvent` 인터페이스를 구현하기만 하면 됩니다.

```
// badge 패키지에 위치한 '공통 규격'
public interface BadgeEvent {
    Member member();
    Policy policy();
}

// checkin 패키지에 위치한 '구체적인 이벤트'
public record CheckInEvent(Member member) implements BadgeEvent {
    @Override
    public Policy policy() {
        return Policy.CHECK_IN;
    }
}
```

- **리스너의 유연성 확보:**
이 설계 덕분에 `BadgeEventListener`는 어떤 구체적인 이벤트가 발생하든 상관없이, `BadgeEvent`라는 단일 타입으로 모든 이벤트를 처리할 수 있게 되었습니다.

---

# 5. 템플릿 메서드 패턴을 이용한 중복 제거
- **문제점:** `GrandSlamBadgePolicy`를 제외한 나머지 단순 뱃지 정책들의 `determineAwardCandidate`(자격 판단) 메서드 로직이 거의 중복되고 있었습니다.

- **해결책:** 템플릿 메서드 패턴 도입
공통 로직은 추상 부모 클래스(`AbstractBadgePolicy`)에 두고, 변화하는 부분(사전 조건 검사)만 하위 클래스에서 `protected` 메서드(`isAwardable`)를 오버라이드하여 구현하도록 구조를 변경했습니다.
```
public abstract class AbstractBadgePolicy implements BadgePolicy {
    // ...
    @Override
    public final BadgeAwardCandidate determineAwardCandidate(final BadgeEvent event) {
        if (!isAwardable(event)) { // 2. 각 정책의 고유한 사전 조건을 여기서 확인
            return null;
        }
        // 1. 모든 정책에 공통으로 적용되는 로직 (템플릿)
        // ...
    }

    // 3. 하위 클래스에서 필요에 따라 재정의할 수 있는 메서드
    protected boolean isAwardable(final BadgeEvent event) {
        return true;
    }
}
```
- **효과:** 이 구조는 현재는 단순해 보이는 정책 클래스들이 미래에 "도배성 채팅은 카운트하지 않는다", "채팅 길이가 10자 미만은 카운트하지 않는다"
와 같은 고유한 로직을 갖게 될 때, 다른 정책에 영향을 주지 않고 해당 정책의 `isAwardable` 메서드만 수정하면 되므로 미래의 확장성을 확보하고 코드의 명확성을 높이는 효과를 가져옵니다.

---

# 6. 결론

뱃지 기능을 도입하면서 단순한 기능 구현을 넘어, **도메인 간 결합도**, **확장성**, **유지보수성**이라는 현실적인 문제들을 해결해야 했습니다.

우리는 **이벤트 기반 아키텍처**를 통해 도메인 간 의존을 제거하고, `@Async`를 활용해 사용자 경험을 해치지 않으면서 비동기적으로 이벤트를 처리했습니다.  
또한, **OCP**, **SRP** 같은 객체지향 원칙을 지키며 **Registry**, **Template Method 패턴** 등을 적용해, 새로운 정책이 추가되더라도 기존 코드를 수정하지 않아도 되는 구조를 만들었습니다.

결과적으로, 뱃지 시스템은 더 이상 특정 도메인에 묶이지 않고 독립적으로 확장 가능한 형태로 발전했습니다.  
이번 설계는 "변경에 유연한 구조"가 얼마나 중요한지를 다시 한번 확인할 수 있었던 좋은 경험이었습니다.  
