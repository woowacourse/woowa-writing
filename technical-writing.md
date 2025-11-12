# 외부 API의 일관성 대신 지속 가능한 가용성 확보하기

## 예상 독자
- 외부 API 의존도가 높은 서비스를 개발하는 개발자
- 시스템 안정성과 장애 대응에 관심 있는 개발자
- 트랜잭션 아웃박스 패턴과 DLQ 적용이 무엇인지 궁금한 개발자

## 배경
TIL(Today I Learned) 플랫폼에 AI 기반 태그 추출 기능을 도입하면서 외부 API 장애로 인한 데이터 정합성 불일치가 발생했습니다.   
이 글에서는 기능의 데이터 정합성 문제를 해결하기 위한 여러 패턴들을 점진적으로 도입하며 가용성까지 고려한 과정을 소개합니다.

---

## 문제 상황

과거 사용자가 TIL을 작성하면 OpenAI나 Claude 같은 AI 서비스가 적절한 태그를 자동으로 추천하는 기능을 구현했습니다. 그러나 시간이 지나며 다음의 문제가 발생했습니다.

- **AI 서비스 장애 시 태그가 생성되지 않음**
- **태그가 없어 유저 작성 TIL의 태그 기반 검색이 불가능**

조사 결과, 간헐적 실패의 원인은 두 가지였습니다.

1. AI 서비스의 서버 다운
2. API 할당량 초과

![AI 서버 상태](https://velog.velcdn.com/images/praisebak/post/7c8f375d-2dbc-4d9d-8b51-b5ff01cc5cb8/image.png)

특정 외부 API는 예상보다 자주 장애가 발생했고, 이로 인해 서비스를 이용하던 사용자는 관련 기능을 사용할 수 없었습니다.   
이에 기능의 가용성을 확보할 방법이 필요합니다.

---

## 해결 방향

### 핵심 요구사항
- **TIL 저장은 AI 서비스 장애와 무관하게 성공해야 합니다**
- **태그는 TIL 저장과 동시에 저장되진 않더라도 결과적으로는 생성되어야 합니다**
- **실패시 이유 등을 알 수 있어야합니다.**

### 요구사항 요약
위 요구사항을 요약해보겠습니다.   
-  외부 API의 가용성 확보
-  실패시 모니터링 용이

### 가용성 확보
#### 가용성이란?
가용성은 다음을 말합니다.
> 서버와 네트워크, 프로그램 등의 정보 시스템이 정상적으로 사용 가능한 정도

#### 가용성을 위해 시도할 수 있는 방법들
1. 최종적 일관성
- 당장은 아니어도, 나중에라도 기능이 성공하도록 하는 방법입니다.
2. 대체 경로 확보
- 현재 경로가(API 등) 실패하는 경우 대체 경로로 시도하는 방법입니다.

### 실패시 모니터링 용이
- 모든 가용성 방법들이 실패하여 수동으로 개발자가 처리해야하는 경우입니다.   
- 실패한 원인 등을 모니터링할 수 있도록 격리가 필요합니다.   

이제 요구사항에 대해 정리해보았으니 서비스 요구사항에 적용해봅시다!

---

### 다시보는 요구사항
- **1. TIL 저장은 AI 서비스 장애와 무관하게 성공해야 합니다**
- **2. 태그는 TIL 저장과 동시에 저장되진 않더라도 결과적으로는 생성되어야 합니다**
- **3. 실패시 현황을 알 수 있어야합니다.**

## Transactional Outbox Pattern 도입
### Transactional Outbox Pattern이란?
![](https://velog.velcdn.com/images/praisebak/post/0abd72eb-fda4-49bd-bde7-c813cf1f34a2/image.png)

출처 : https://microservices.io/

microservices.io에 따르면 다음을 트랜잭션 아웃박스 패턴이라고 합니다.

> The solution is for the service that sends the message to first store the message in the database as part of the transaction that updates the business entities. A separate process then sends the messages to the message broker.

메시지를 전송하는 서비스가 비즈니스 엔티티를 업데이트하는 트랜잭션의 일부로서, 먼저 메시지를 데이터베이스에 저장합니다.
그 후, 별도의 프로세스가 데이터베이스에 저장된 메시지를 메시지 브로커로 전송합니다.

간단하게는, 메시지 전송(외부 api 호출)시에 메시지 그 자체를 데이터베이스에 저장하고 추후에 전송하는 패턴이라고 설명할 수 있습니다.

### 왜 Transactional Outbox이 필요한가?
- TIL 생성에 성공한뒤 외부 API에 의한 태그 생성이 실패하는 경우 추후에 재시도할 방법이 필요합니다.
- 여기서는 태그 생성에 대해 **최종적 일관성**을 보장하기 위해 트랜잭션 아웃박스 패턴을 도입합니다.

### 재시도로는 최종적 일관성을 확보할 수 없나요?
- 단순 재시도는 서버가 재시작되는 순간 실패한 요청이 완전히 휘발되는 문제가 있었습니다.   
- 최종적 일관성을 보장하려면 재시도 정보가 영구 저장되어야 합니다. 이것이 **Transactional Outbox**를 도입한 이유입니다.

### TagCreationOutboxEvent Entity
이제 본격적으로 트랜잭션 아웃박스 패턴을 구현해보겠습니다.

Transactional Outbox 패턴을 위해 `재시도에 필요한 정보`와, `요청 상태`가 필요합니다.
다음 엔티티로 자세히 알아보겠습니다.

```java
@Entity
@Table(name = "tag_creation_outbox_events")
public class TagCreationOutboxEvent extends BaseEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "til_id", nullable = false)
    private Long tilId;

    @Column(name = "til_content", columnDefinition = "TEXT", nullable = false)
    private String tilContent;

    @Column(name = "user_id", nullable = false)
    private Long userId;

    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false)
    private OutboxEventStatus status;

    private LocalDateTime scheduledAt;
}
```

`TagCreationOutboxEvent` 엔티티의 tilContent 필드는 재시도에 필요합니다.
`status` 필드로 재요청이 실패했는지를 판단합니다.

상세 필드 설명:
- **status**: 이벤트 처리 상태 추적 (PENDING, PROCESSING, COMPLETED, FAILED)
- **scheduledAt**: 언제 스케줄링 되었는지 기록


### 스케줄러 구현

```java
@Scheduled(fixedDelay = 30_MIN)
@Transactional
public void processPendingEvents() {
    List<TagCreationOutboxEvent> pendingEvents = 
        outboxRepository.findPendingEvents(LocalDateTime.now());

    for (TagCreationOutboxEvent event : pendingEvents) {
        try {
            processEvent(event.getId());
        } catch (Exception e) {
            log.error("Failed to process pending event {}", event.getId(), e);
            throw e;
        }
    }
}

```

30초마다 대기 중인 이벤트를 처리하게 합니다.

### 이벤트 저장 구현
태그 생성 이벤트를 Outbox에 저장해줍니다.
이 이벤트는 추후 이전에 구현했던 `processPendingEvents`에서 실행될 것입니다.

```java
/**
* 태그 생성 이벤트를 Outbox에 저장 (트랜잭션 안전)
*/
@Transactional
public void scheduleTagCreation(TilCreatedEvent tilCreatedEvent) {
        TagCreationOutboxEvent outboxEvent = TagCreationOutboxEvent.builder()
                .tilId(tilCreatedEvent.getTilId())
                .tilContent(tilCreatedEvent.getTilContent())
                .userId(tilCreatedEvent.getUserId())
                .status(OutboxEventStatus.PENDING)
                .scheduledAt(LocalDateTime.now())
                .build();

        outboxRepository.save(outboxEvent);

        log.info("Tag creation scheduled for TIL {}", tilCreatedEvent.getTilId());
}
```

- 이제 외부 api가 실패하는 경우에도 안전하게 영구적으로 이벤트를 저장하고
  주기적으로 스케줄링하여 실행함으로써 **최종적 일관성**을 보장할 수 있습니다.

---

## Failover를 통한 대체 경로 확보
지금까지 구현으로는 장기간 API 서버 자체가 다운된 경우 성공까지 많은 시간이 걸릴 것입니다.
이런 **단일 병목,장애 지점**을 없애기 위하여 여러 AI API를 순차적으로 시도하도록 외부 API 시스템에 적용해봅시다.

### Failover란?
![](https://velog.velcdn.com/images/praisebak/post/029ea49b-0b8b-46c3-afb1-a59883377405/image.png)
- Failover는 실패하면, 다른 대체 경로로 요청을 하는 방식을 말합니다.
- 해당 글에서는 OpenAI API 요청이 실패하면, Claude API를 요청하도록 Failover하도록 구현하겠습니다.

사진 출처 : https://velog.io/@zxcvbnm5288/%ED%8E%98%EC%9D%BC%EC%98%A4%EB%B2%84Failover%EC%99%80-%ED%8E%98%EC%9D%BC%EB%B0%B1Failback

### AIClient 인터페이스 설계
먼저 AI 클라이언트를 나타낼 인터페이스를 구현합시다.

```java
public interface AIClient {
    String callAI(List<Map<String, Object>> messages, 
                  Map<String, Object> functionDefinition);
    String getClientName();
    boolean isAvailable();
}
```

### AIService 구현
`AIClient` 인터페이스를 구현한 구현체들을 차례대로 호출하도록 구현합니다.

```java
public String callAIWithSimpleFallback(
    List<Map<String, Object>> messages,
    Map<String, Object> functionDefinition
) {
    for (AIClient client : aiClients) {
        try {
            return client.callAI(messages, functionDefinition);

        } catch (Exception e) {
            log.warn("Failed to call {} API: {}", 
                     client.getClientName(), e.getMessage());
        }
    }

    throw new RuntimeException("No available AI services");
}
```

---

## Dead Letter Queue를 이용한 모니터링 용이
지금까지 구현한 방식으로 가용성을 확보했습니다.
하지만 장기간 장애가 발생한 경우나, 왜 실패했는지 사유 등은 알지 못하는 상태입니다.
**Dead Letter Queue**를 통해 실패한 이벤트를 격리하여 모니터링을 용이하도록 해보겠습니다.

### Dead Letter Queue

Dead Letter Queue(이하 DLQ)는 처리하지 못한 메시지를 별도로 저장하는 특수 큐입니다. 정상 처리되지 않은 메시지를 격리하여 재처리나 분석을 가능하게 합니다.

### Transactional Outbox Pattern과 DLQ의 차이

| 항목 | Transactional Outbox | DLQ |
|---|---|---|
| 목적 | 메시지 처리 일관성 보장 | 실패 메시지 격리 및 분석 |
| 메시지 저장 시점 | 정상 흐름 | 예외 발생 시 |
| 재처리 | 자동 재시도 | 수동 개입 필요 |
| 알람 | X | O |

즉 DLQ는 실패한 이벤트를 잘 관리하기 위한 패턴입니다.
최종적 일관성을 보장해주는 트랜잭션 아웃박스 패턴과는 목적이 다르다고 볼 수 있습니다.

### DLQEvent 엔티티 설계

DLQ 엔티티를 설계할때는, 개발자가 왜 실패했는지를 알게 해주기 위해 에러 정보들을 잘 담는것이 필요합니다.
다음 예제코드에서는 이벤트 타입과 에러 메시지, 스택 트레이스 등을 디버깅용으로 저장하고 있습니다.

```java
@Entity
@Table(name = "dlq_events")
public class DLQEvent extends BaseEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "original_event_type", nullable = false)
    private String originalEventType;

    @Column(name = "original_event_id")
    private Long originalEventId;

    @Column(name = "payload", columnDefinition = "TEXT", nullable = false)
    private String payload;

    @Column(name = "error_message", columnDefinition = "TEXT")
    private String errorMessage;

    @Column(name = "stack_trace", columnDefinition = "TEXT")
    private String stackTrace;

    @Column(name = "alarm_sent", nullable = false)
    @Builder.Default
    private Boolean alarmSent = false;

    public boolean shouldSendAlarm() {
        return !this.alarmSent && 
               this.status == DLQEventStatus.PERMANENTLY_FAILED;
    }
}
```

### DLQ 저장 및 알람 전송
이제 DLQ를 저장했다면, 실패한 상황을 개발자에게 알람을 보내야합니다.

```java
@Transactional
public void sendToDLQ(String originalEventType, Long originalEventId, 
                     String payload, String errorMessage, String stackTrace) {
    
    DLQEvent dlqEvent = DLQEvent.builder()
        .originalEventType(originalEventType)
        .originalEventId(originalEventId)
        .payload(payload)
        .errorMessage(errorMessage)
        .stackTrace(stackTrace)
        .status(DLQEventStatus.PERMANENTLY_FAILED)
        .build();

    dlqEventRepository.save(dlqEvent);
    log.warn("Event sent to DLQ: {} (ID: {})", originalEventType, dlqEvent.getId());
    
    sendAlarmIfNeeded(dlqEvent.getId());
}

@Async
public void sendAlarmIfNeeded(Long dlqEventId) {
    try {
        DLQEvent dlqEvent = dlqEventRepository.findById(dlqEventId)
            .orElseThrow(() -> new IllegalArgumentException("DLQ event not found"));

        if (dlqEvent.shouldSendAlarm()) {
            dlqAlarmService.sendDLQAlarm(dlqEvent);
            dlqEvent.markAlarmSent();
            dlqEventRepository.save(dlqEvent);
        }
    } catch (Exception e) {
        log.error("Failed to send alarm for DLQ event {}", dlqEventId, e);
    }
}
```

### 전송될 Slack 알람 메시지

```
🚨 *DLQ Alert - PROD*

*Event Details:*
• ID: `12345`
• Type: `TAG_CREATION_OUTBOX`
• Status: `PERMANENTLY_FAILED`
• Created: `2024-01-15 14:30:25`

*Error Message:* Connection timeout after 3 retries
*Stack Trace:* ...
```

알람 메시지에는 로그를 확인하지 않고도 디버깅할 수 있도록 stackTrace, 에러 메시지, 이벤트 정보를 포함합니다.

## 결론
지금까지, 실패한 api를 **Transactional Outbox Pattern**과, **Failover**로 최종적 일관성을 보장했습니다.   
추가적으로 **DLQ**를 통해 실패하는 이벤트들을 개발자가 모니터링할 수 있게 하였습니다.


# 문제

## 오래된 이벤트들 정리
- 해당 글에서는 메시지 브로커가 아닌 데이터베이스를 사용하여 여러 패턴들을 구현했습니다.
  - 데이터베이스에 영구적으로 저장하다보면 시간이 지나며 데이터베이스에 부담이 될 수 있습니다.
- 또한 이벤트가 많이 쌓일수 있는 서비스에서는 데이터베이스 병목도 생길 수 있다는 점을 염두해야합니다.

## 중복 실행 문제
- 분산 환경에서는 중복하여 이벤트를 실행하는 경우도 생길 수 있습니다.
  - 이를 엄밀하게 관리하기 위해서는 분산락 혹은 멱등성 보장을 보장할 방법을 고려해야합니다.

# 배운 점

1. **애플리케이션 레벨 재시도의 한계**: 무중단 배포나 분산 환경에서는 재시도는 휘발될 수 있습니다. 단순 재시도로는 가용성을 보장하기 어렵습니다.
2. **영구 저장소의 필요성**: DB나 메시지 큐처럼 분산 서버가 공유할 수 있수 있고 영속성을 제공할 만한 방법을 사용합니다.
3. **규모에 따른 선택**: 현재는 이벤트 생성 규모를 고려하여 DB 기반 Transactional Outbox 방식을 채택했지만, 처리량이 많아지면 메시지 큐 도입을 고려해야 합니다.

# 결론

3겹 방어(Transactional Outbox + Failover + DLQ)를 구축했지만, Slack API마저 실패하면 알람을 받지 못합니다. 이는 다음의 딜레마로 귀결됩니다.
> 실패를 막기 위한 방어선도 실패할 수 있다.
> **해당 기능의 안정성을 위해 얼마나 자원을 투자할 수 있는가?**

아무리 방어선을 많이 두어도,가용성 100%는 불가능에 가깝습니다.
대신, 우리 서비스의 SLA 기준을 세우고 이를 충족하도록 설계하는 것은 가능합니다.
![AWS CloudFront SLA](https://velog.velcdn.com/images/praisebak/post/7652c147-5dfe-4cf6-8757-dce6da6b7f7e/image.png)

예로, AWS CloudFront는 월별 가용성에 따라 요금을 환불하는 SLA를 제공합니다. 

해당 고찰로 무작정 기술로 해결할 수 있다고 생각하기보다
현실적인 목표를 설정하고, 이를 달성하기 위한 적절한 전략을 생각하는것이 개발자가 가야할 방향이 아닐까 생각할 수 있었습니다.
독자분들도 우리 서비스의 가용성 목표는 어느정도인지 다시 한번 생각해보면 어떨까요? 🤓   


---

## 참고 자료

- [AWS Outbox Pattern](https://aws.amazon.com/blogs/database/implementing-the-outbox-pattern-with-amazon-dynamodb/)
- [Martin Fowler - Circuit Breaker](https://martinfowler.com/bliki/CircuitBreaker.html)
- [AWS SLA](https://aws.amazon.com/cloudfront/sla/)
