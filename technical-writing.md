# 외부 API 최종적 일관성 보장 포기하기

## 예상 독자
- 외부 API 의존도가 높은 서비스를 개발하는 백엔드 개발자
- 시스템 안정성과 장애 대응에 관심 있는 개발자
- 트랜잭션 아웃박스 패턴과 DLQ 적용을 고민하는 개발자

## 개요
TIL(Today I Learned) 플랫폼에 AI 기반 태그 추천 기능을 도입하면서 외부 API 장애로 인한 데이터 정합성 불일치가 발생했습니다. 이 글에서는 Transactional Outbox Pattern, Failover, DLQ를 단계적으로 도입해 시스템 가용성을 높인 과정을 소개합니다.

---

## 문제 상황

사용자가 TIL을 작성하면 OpenAI나 Claude 같은 AI 서비스가 적절한 태그를 자동으로 추천하는 기능을 운영했습니다. 그러나 다음의 문제가 발생했습니다.

- **AI 서비스 장애 시 태그가 생성되지 않음**
- **태그가 없어 유저 작성 TIL의 태그 기반 검색이 불가능**

조사 결과, 간헐적 실패의 원인은 두 가지였습니다.

1. OpenAI 서버 다운
2. API 할당량 초과

![OpenAI 서버 상태](https://velog.velcdn.com/images/praisebak/post/7c8f375d-2dbc-4d9d-8b51-b5ff01cc5cb8/image.png)

특정 외부 API는 예상보다 자주 장애가 발생했고, 이로 인해 서비스를 이용하던 사용자는 검색 기능을 사용할 수 없었습니다.
기능 **최종적 일관성**을 보장할 수 있는 방법이 필요합니다.

---

## 해결 방향

### 핵심 요구사항
- **TIL 저장은 AI 서비스 장애와 무관하게 성공해야 합니다**
- **태그는 결과적으로 생성되어야 합니다**

이를 위해 세 가지 방어선을 단계적으로 설계했습니다.

### 1단계: Transactional Outbox Pattern
외부 API 호출이 실패하면 그 요청을 DB에 저장해두고, 백그라운드 스케줄러가 주기적으로 재시도하는 패턴입니다. 서버가 재시작되어도 DB에 저장된 요청은 사라지지 않으므로, 결과적으로 태그가 생성됨을 보장할 수 있습니다.

### 2단계: Failover
하나의 AI 서비스(OpenAI)가 실패하면 즉시 다른 AI 서비스(Claude)로 전환하는 방식입니다. 주 서비스에 문제가 생겨도 대체 서비스로 기능을 계속 제공할 수 있어 가용성이 높아집니다.

### 3단계: DLQ (Dead Letter Queue)
Outbox Pattern과 Failover를 모두 거쳤는데도 실패한 메시지를 별도 저장소에 격리하는 방식입니다. 이 경우 자동 복구가 불가능하므로, 개발자에게 Slack 알람을 보내 수동 조치를 유도합니다. DLQ에는 에러 메시지와 스택 트레이스가 함께 저장되어 빠른 원인 분석이 가능합니다.

---

## 1단계: 트랜잭션 아웃박스 패턴(Transactional Outbox Pattern) 도입

### 왜 트랜잭션 아웃박스 패턴이 필요한가?
- TIL 생성에 성공한뒤 외부 API에 의한 태그 생성이 실패하는 경우, 데이터 정합성이 불일치합니다.
- 태그 생성에 대해 **최종적 일관성**을 보장하기 위해 트랜잭션 아웃박스 패턴을 도입합니다.

### 트랜잭션 아웃박스 패턴이란?
![](https://velog.velcdn.com/images/praisebak/post/0abd72eb-fda4-49bd-bde7-c813cf1f34a2/image.png)

출처 : https://microservices.io/

microservices.io에 따르면 다음과 같은 것을 트랜잭션 아웃박스라고 합니다.

> The solution is for the service that sends the message to first store the message in the database as part of the transaction that updates the business entities. A separate process then sends the messages to the message broker.

번역 : 메시지를 전송하는 서비스가 비즈니스 엔티티를 업데이트하는 트랜잭션의 일부로서, 먼저 메시지를 데이터베이스에 저장하는 것이다.
그 후, 별도의 프로세스가 데이터베이스에 저장된 메시지를 메시지 브로커로 전송한다.

간단하게는, 메시지 전송(외부 api 호출)시에 메시지 그 자체를 데이터베이스에 저장하고, 이후에 전송한다고 볼 수 있습니다.

### 메시지 브로커를 써야하나요?
- 해당 글에서는 메시지 브로커를 **스케줄링 + 주기적 polling** 방식으로 간소화시켜 같은 효과를 내보겠습니다.

### 재시도로는 최종적 일관성이 해소되지 않나요?
재시도 방식은 서버가 재시작되는 순간 실패한 요청이 완전히 휘발되는 문제가 있었습니다.   
따라서 최종적 일관성을 보장하려면 재시도 정보가 영구 저장되어야 합니다. 이것이 **트랜잭션 아웃박스 패턴**을 도입한 이유입니다.


### TagCreationOutboxEvent 엔티티
이제 본격적으로 트랜잭션 아웃박스 패턴을 구현해보겠습니다.

트랜잭션 아웃박스 패턴을 위해 `재시도에 필요한 정보`와, `요청 상태`가 필요합니다.
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

    @Column(name = "retry_count", nullable = false)
    @Builder.Default
    private Integer retryCount = 0;

    @Column(name = "scheduled_at", nullable = false)
    private LocalDateTime scheduledAt;

    public void incrementRetryCount() {
        this.retryCount++;
    }

    public boolean canRetry() {
        return retryCount < 2 && status == OutboxEventStatus.FAILED;
    }
}
```

`TagCreationOutboxEvent` 엔티티의 tilContent 필드는 재시도에 필요합니다.
`status` 필드로 재요청이 실패했는지를 판단합니다.

상세 필드 설명:
- **status**: 이벤트 처리 상태 추적 (PENDING, PROCESSING, COMPLETED, FAILED)
- **retryCount**: 재시도 횟수 제한 (최대 2회)
- **scheduledAt**: 다음 재시도 스케줄링을 위한 이전 스케줄링 시도 측정


### 스케줄러 구현

```java
@Scheduled(fixedDelay = 30000) // 30초마다
@Transactional
public void processPendingEvents() {
    List<TagCreationOutboxEvent> pendingEvents = 
        outboxRepository.findPendingEvents(LocalDateTime.now());

    for (TagCreationOutboxEvent event : pendingEvents) {
        try {
            processEvent(event.getId());
        } catch (Exception e) {
            log.error("Failed to process pending event {}", event.getId(), e);
        }
    }
}

```

30초마다 대기 중인 이벤트를 처리합니다.

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
이제 외부 api가 실패하는 경우에도 안전하게 영구적으로 이벤트를 저장함으로써
**최종적 일관성**을 보장할 수 있습니다.

---

## 2단계: Failover 시스템 구축
지금까지 구현으로는 장기간 API 서버 자체가 다운된 경우 대응할 수 없습니다.   
이런 **단일장애지점**을 없애기 위해 여러 AI 서비스를 순차적으로 시도하는 Failover 기반의 외부 API 시스템을 구축해봅시다.

### Failover란?
![](https://velog.velcdn.com/images/praisebak/post/029ea49b-0b8b-46c3-afb1-a59883377405/image.png)
- failover는 실패하면, 다른 대체 경로로 요청을 하는 방식을 말합니다.
- 해당 글에서는 open ai 요청이 실패하면, claude ai를 요청하도록 failover하는 것을 설계 목표로 합니다.

사진 출처 : https://velog.io/@zxcvbnm5288/%ED%8E%98%EC%9D%BC%EC%98%A4%EB%B2%84Failover%EC%99%80-%ED%8E%98%EC%9D%BC%EB%B0%B1Failback

### AIClient 인터페이스 설계
먼저 공통적으로 ai 클라이언트를 아우를 수 있는 인터페이스를 구현합시다.

```java
public interface AIClient {
    String callAI(List<Map<String, Object>> messages, 
                  Map<String, Object> functionDefinition);
    String getClientName();
    boolean isAvailable();
}
```

### FailoverAIServiceManager 구현
이제 위 인터페이스를 차례대로 호출하고 실패하면 failover 하도록 구현합시다.

```java
public String callAIWithSimpleFallback(
    List<Map<String, Object>> messages,
    Map<String, Object> functionDefinition
) {
    for (AIClient client : aiClients) {
        try {
            log.info("Attempting to call {} API", client.getClientName());
            String result = client.callAI(messages, functionDefinition);
            log.info("Successfully called {} API", client.getClientName());
            return result;

        } catch (Exception e) {
            log.warn("Failed to call {} API: {}", 
                     client.getClientName(), e.getMessage());
            if (isLastClient(client)) {
                throw new RuntimeException("All AI services failed", e);
            }
            log.info("Trying next AI service");
        }
    }

    throw new RuntimeException("No available AI services");
}
```

테스트를 위해 openai client가 항상 예외를 반환하도록하여 테스트해보았습니다

### 테스트 실행 로그

```
2025-07-15 10:30:15 INFO  FailoverAIServiceManager - Attempting to call openai API
2025-07-15 10:30:16 WARN  FailoverAIServiceManager - Failed to call openai API: Connection timeout
2025-07-15 10:30:16 INFO  FailoverAIServiceManager - Trying next AI service
2025-07-15 10:30:16 INFO  FailoverAIServiceManager - Attempting to call Claude API
2025-07-15 10:30:17 INFO  FailoverAIServiceManager - Successfully called Claude API
```


## 결과
### 지금까지의 전체 구조
전체 구조는 다음과 같습니다.
![전체 플로우](https://velog.velcdn.com/images/praisebak/post/ca07ceb1-a06f-4f5e-9c58-5cf6ae87fcdb/image.png)

---

## 3단계: DLQ로 최종 방어선 구축
지금까지 트랜잭션 아웃박스 패턴과 failover로 실패하는 경우 재시도하는 방식으로 **최종적 일관성**을 보장했습니다.
하지만 개발자는 트랜잭션 아웃박스 등으로 최종적 일관성을 보장한다는 것만 알지, 왜 실패했는지를 알지 못하는 상태입니다.
외부 API에 대한 상태 관리가 중요한 경우 이는 문제가 생길 수 있습니다.
해당 글에서는 **DLQ 패턴(Dead Letter Queue)**을 통해 실패한 이벤트를 잘 관리하는 방법에 대해서 알아보겠습니다.

### DLQ란?

DLQ(Dead Letter Queue)는 처리하지 못한 메시지를 별도로 저장하는 특수 큐입니다. 정상 처리되지 않은 메시지를 격리하여 재처리나 분석을 가능하게 합니다.

### Transactional Outbox Pattern과 DLQ의 차이

| 항목 | Transactional Outbox | DLQ |
|---|---|---|
| 목적 | 메시지 처리 일관성 보장 | 실패 메시지 격리 및 분석 |
| 메시지 저장 시점 | 정상 흐름 | 예외 발생 시 |
| 재처리 | 자동 재시도 | 수동 개입 필요 |
| 알람 | X | O |

즉 DLQ는 실패한 이벤트를 잘 관리하기 위한 패턴입니다.
최종적 일관성을 보장해주는 트랜잭션 아웃박스 패턴과는 결이 다르다고 볼 수 있습니다.

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

알람 메시지에는 로그를 확인하지 않고도 디버깅할 수 있도록 stackTrace, 에러 메시지, 이벤트 정보를 포함했습니다.

## 주의) 오래된 이벤트들 정리

저희는 메시지 브로커가 아닌 데이터베이스를 사용하여 여러 패턴들을 구현했습니다.
데이터베이스에 영구적으로 저장하다보니 시간이 지나며 여러 이벤트들을 쌓일 수 있습니다.
한 달 이상 된 이벤트는 자동으로 삭제해줍시다.

다음의 코드는 DLQ를 대상으로 오래된 이벤트를 삭제합니다.
```java
@Scheduled(cron = "0 0 3 * * *") // 매일 새벽 3시
@Transactional
public void cleanupOldDLQEvents() {
    LocalDateTime oneMonthAgo = LocalDateTime.now().minusMonths(1);
    int deletedCount = dlqEventRepository.deleteOldEvents(oneMonthAgo);
    log.info("Cleaned up {} old DLQ events", deletedCount);
}
```

## 결론
지금까지, 실패한 api를 **트랜잭션 아웃박스 패턴**과, **failover**로 최종적 일관성을 보장했습니다.   
DLQ를 통해 실패하는 이벤트들도 개발자가 인식할 수 있게 합니다.

### 배운 점

1. **애플리케이션 레벨 재시도의 한계**: 무중단 배포나 스케일아웃 환경에서는 메모리 기반 재시도로 가용성을 보장하기 어렵습니다.
2. **영구 저장소의 필요성**: DB나 메시지 큐처럼 서버가 공유할 수 있는 자원을 사용해야 합니다.
3. **규모에 따른 선택**: 현재 서비스는 우아한테크코스 크루들에게만 제공되어 DB 기반 Transactional Outbox Pattern을 채택했지만, 처리량이 많아지면 메시지 큐 도입을 고려해야 합니다.

### 한계와 SLA

3겹 방어(Transactional Outbox + Failover + DLQ)를 구축했지만, Slack API마저 실패하면 알람을 받지 못합니다. 이는 다음의 딜레마로 귀결됩니다.

> 실패를 막기 위한 방어선은 실패할 수 있다.

우리는 해결되지 않는 딜레마 대신에 다음의 질문을 던져야합니다.

> **해당 기능의 안정성을 위해 얼마나 자원을 투자할 수 있는가?**

아무리 방어선을 많이 두어도,가용성 100%는 불가능에 가깝습니다.
대신 우리 서비스의 SLA 기준을 세우고 이를 충족하도록 설계하는 것이 이상적입니다.

![AWS CloudFront SLA](https://velog.velcdn.com/images/praisebak/post/7652c147-5dfe-4cf6-8757-dce6da6b7f7e/image.png)

예로, AWS CloudFront는 월별 가용성에 따라 요금을 환불하는 SLA를 제공합니다. 이처럼 현실적인 가용성 목표를 설정하고, 이를 달성하기 위한 적절한 방어 전략을 구축하는 것이 중요합니다.

---

## 참고 자료

- [AWS Outbox Pattern](https://aws.amazon.com/blogs/database/implementing-the-outbox-pattern-with-amazon-dynamodb/)
- [Martin Fowler - Circuit Breaker](https://martinfowler.com/bliki/CircuitBreaker.html)
- [AWS SLA](https://aws.amazon.com/cloudfront/sla/)
