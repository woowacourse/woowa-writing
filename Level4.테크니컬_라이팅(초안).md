# 실시간 KBO 웹데이터는 왜 이렇게 '흐르는 값'일까 
## Playwright 크롤링과 Adaptive Polling으로 만든 야구보구의 실시간 구조

야구 경기 중 채팅창을 보면 "지금 몇 대 몇?" 같은 질문이 끊이지 않는다. 야구보구는 이 간단한 질문에 답하기 위해 KBO 웹페이지를 실시간으로 크롤링한다.

하지만 KBO는 공식적인 실시간 경기 데이터 API를 제공하지 않는다. 실제 경기 정보는 모두 웹페이지에서 동적으로 갱신되는 값으로만 노출된다.

야구보구는 이 웹페이지를 실시간으로 크롤링해 **GPS 직관 인증, SSE 기반 경기장 팬 비율 조회, 팬 승률 분석**을 제공하는 구조를 선택했다.

처음에는 단순했다. "그냥 웹에서 긁어오면 되겠지."

하지만 실제로 마주한 데이터는 **완성된 값이 아니라, 경기 도중 계속 바뀌는 '흐르는 상태'** 였다.

- 사용자는 **분 단위 실시간성**을 기대했고
- 경기 시간과 수, 상태는 **매일 달라졌으며**
- 실시간 데이터는 "한 번 저장하면 끝나는 스냅샷"이 아니었다.

이 글은 이 문제를 해결하기 위해 **MVP1(폴링 최적화) → MVP2(데이터 파이프라인) → MVP3(성능 개선)** 로 개선한 과정을 다룬다.

## 0. 시작: Playwright 선택과 실시간 요구사항

### **0-1. 크롤링 도구: 왜 Playwright인가**

BeautifulSoup으로 KBO 페이지를 긁으면 빈 화면만 나온다. 점수·이닝 같은 핵심 데이터가 모두 JavaScript로 동적 렌더링되기 때문이다.

따라서 KBO처럼 **JavaScript로 데이터를 동적 렌더링하는 사이트**는 반드시 브라우저 자동화 도구가 필요하다. BeautifulSoup 같은 **정적 HTML 파싱 라이브러리**는 JavaScript 실행이 불가능하기 때문에 Playwright와 Selenium 중 하나를 선택해야 했다.

**Playwright를 선택한 이유:**

- **Selenium 대비 20~30% 빠른 페이지 로딩**: WebSocket 기반 CDP로 브라우저와 직접 통신하여, Selenium의 HTTP 요청/응답 방식 대비 통신 오버헤드가 적다.
- **Headless 모드 안정성**: Headless 모드에서 더 빠른 페이지 로딩 성능을 보인다.

**트레이드오프:**

다만 Playwright는 브라우저를 계속 실행해야 해서 메모리와 CPU를 많이 쓴다. 이 문제는 진행 중인 경기만 크롤링하는 방식으로 해결했다.

### 0-2. 저장소 선택: MongoDB가 아니라 MySQL + JSON(Bronze)을 택한 이유

실시간 경기 데이터는 구조가 완전히 고정돼 있지 않다.

DOM 구조가 바뀌거나, 필드가 생겼다가 사라지는 일이 충분히 일어날 수 있다. 처음에는 **MongoDB 단일 컬렉션**이 가장 자연스러운 선택처럼 보였다.

**MongoDB 옵션의 장점**

- 스키마 유연성: 웹데이터 구조 변경에 자연스럽게 대응
- JSON 저장 최적화: 크롤링 결과를 변환 없이 저장 가능
- 단일 컬렉션 기반으로 "오늘 경기, 현재 상태" 같은 조회에 적합

하지만 현실적인 제약은 다음과 같았다.

- 팀 전체가 **RDB(MySQL)에 더 익숙**했고 우테코 프로젝트 기간 내에 MongoDB 학습 커브를 흡수하기 어렵다.
- 팬 인증, 승률 분석, 승리 요정 랭킹 등은 **관계형 조인/집계 쿼리**가 핵심이었다.
- 체크인/랭킹/경기 데이터 업데이트 간의 **트랜잭션 일관성(ACID)** 도 중요했다.

그래서 이렇게 결론을 냈다.

- **원본(raw)은 JSON 그대로 보존**한다.
- 하지만 정제·집계·서비스 제공은 **MySQL 기반**으로 처리한다.

### 0-3. 사용자가 기대한 실시간성

초기에는 “하루 한 번 경기 결과를 업데이트하면 되겠지”라고 생각했다.
하지만 실제 사용자 피드백은 달랐다.

> “채팅창에서 점수 바뀔 때마다 반영되면 좋겠어요.”
> “경기 결과를 최대한 빨리 보고 싶어요.”


<img width="1068" height="344" alt="image" src="https://github.com/user-attachments/assets/b8c5a9bc-46c7-4992-8fa8-0e8b2716d23d" />
<img width="1068" height="334" alt="image" src="https://github.com/user-attachments/assets/8fc122fe-d971-4f4a-86dc-a56962f01bea" />
<img width="1278" height="566" alt="image" src="https://github.com/user-attachments/assets/62ffc376-ca22-40b1-8397-87dce1d1772d" />

요약하면, 사용자가 원한 건:

> **“하루 1회 스냅샷”이 아니라, “분 단위로 흘러가는 경기 상황이 바로 반영되는 흐름”**

즉, 웹페이지가 갱신되는 순간을 **폭주 없이, 적절한 비용으로, 빠르게 감지해 전달하는 구조**가 필요했다.

# MVP 1. Adaptive Polling — "언제 가져올까?" 최적화

### 1-1. 문제 정의: 고정 주기 폴링의 한계

가장 단순한 접근은 **“1분마다 전 경기 크롤링”** 이다.

하지만 이 방식은 금방 한계에 부딪힌다.

1. **경기 스케줄이 불규칙하다**
    
    - 어떤 날은 경기가 오후 2시, 어떤 날은 5시, 6시에 시작한다.
        
    - 경기 수 역시 매일 다르다.
        
        → 고정된 “시작 시간/끝 시간 + N분 간격” 규칙으로는 맞추기 어렵다.
        
    
2. **경기 상태에 따라 데이터 변화량이 다르다**
    
    - **Scheduled**: 거의 변하지 않음 → 자주 긁을 이유가 없다.
        
    - **Live**: 점수/이닝/상황이 계속 바뀜 → 짧은 주기가 필요하다.
        
    - **Final**: 이미 확정된 값 → 다시 긁어도 값이 바뀌지 않는다.
        
    
3. **크롤링 비용이 곧 인프라 비용이다**
    
    - Playwright 인스턴스 유지 비용 (CPU/메모리)
        
    - KBO 웹 서버에 대한 부담
        
    - 우리 서버의 네트워크·DB 부하
    

따라서, “실시간성을 확보하면서도 비용을 줄이려면” 고정 주기가 아니라 **상태 기반 폴링(Adaptive Polling)** 이 필요했다.

### 1-2. 해법: 상태 기반 Adaptive Polling

핵심 질문은 하나였다.

> **“지금 이 경기를, 지금 이 시점에 정말 크롤링해야 하는가?”**

  일정을 00시에 불러오고, 각 경기의 **현재 상태(State)** 에 따라 크롤링 주기를 다르게 가져가는 구조로 바꿨다.
  
|**경기 상태**|**크롤링 정책**|
|---|---|
|Scheduled|크롤링 스킵|
|Live|1분 간격 크롤링|
|Final|크롤링 스킵|

**효과**

1. **반영 시간 단축**
    
    - Before: 하루 1회 업데이트 → 최대 7시간 지연
        
    - After: Live 경기 1분 주기 → **최대 1분 지연**
        
    
2. **크롤링 요청 88% 절감**
    
    - Before: 모든 경기 고정 주기 크롤링 (~1,440회/일)
	    - 24시간 × 60분 = 1,440회 / 하루
    - After: Live 경기만 선택적 크롤링 (~180회/일)
	    - 3시간 × 60분 = 180회 / 하루

    - 절감률: (1,440 - 180) / 1,440 = 87.5% ≈ 88%
        

3. **리소스 효율**
    
    - 불필요한 Playwright 실행 감소
        
    - 서버 CPU·메모리, KBO 서버 부하 모두 감소
        
구현 관점에서는:

- AdaptivePoller — 상태 기반 폴링 루프
    
- GameScheduleManager — 00시에 경기 일정 로딩
    
- BackoffStrategy — 실패 시 재시도 정책 등으로 나누어 정리했지만, 더 본질적인 문제는 남았다. 

> “언제 가져올까”는 해결했지만,
 **“무엇을 어떻게 저장하고, 서비스에 어떻게 전달할까”는 여전히 취약**했다.

## 1-3. Adaptive Polling의 구조적 한계

> 크롤링 → 바로 가공 → DB 저장 → 서비스에서 바로 사용

이 방식에는 두 가지 치명적인 문제가 있었다.

1. **원본 데이터가 남지 않는다**
    
    - KBO 응답을 가공해 바로 “서비스용 스키마”로 저장했다.
        
    - 값이 이상할 때, 그게
        
        - 크롤링 문제인지
            
        - 정제 로직 문제인지
            
        - KBO 페이지 문제인지
            
     를 구분할 방법이 없었다.
    
2. **서비스 로직이 크롤러에 직접 종속된다**
    
    - DOM 필드명이 바뀌면 크롤러 → SSE → 팬 인증 → 통계 로직까지 연쇄적으로 수정해야 한다.
            
    - 새 기능 추가도 항상 **크롤러부터 다시 수정해야** 했다.
        
    - 크롤링 실패 시 전체 기능이 동시에 장애가 났다.

"언제 가져올까"는 해결했지만, "무엇을 어떻게 저장할까"가 다음 단계의 핵심 문제가 되었다.

---
# MVP 2. Data Pipeline — “무엇을 어떻게 저장·관리할까?” 해결

## 2-1. 설계 원칙 4가지

Adaptive Polling의 한계를 겪으며, 데이터 구조에 대해 네 가지 원칙을 다시 세웠다.

1. **원본 보존 (Bronze Layer)**
    
    - 크롤링한 JSON을 **가공 없이 100% 그대로** 저장한다.
        
    - 나중에 어떤 문제가 생겨도 “그 시점의 실제 응답”을 재현할 수 있어야 한다.
        
    
2. **정제 분리 (Silver Layer)**
    
    - Bronze를 읽어 **표준화된 엔티티**로 변환한다.
        
    - 타입 변환, ID 매핑, ENUM 처리, null 보정 등 정제 작업을 이 레이어에서만 수행한다.
        
    
3. **서비스 최적화 (Gold Layer)**
    
    - API/SSE/랭킹처럼 **서비스별로 최적화된 모델**은 Gold에서 제공한다.
        
    - 집계가 무겁거나, 조회 패턴이 다른 기능은 별도 Gold 테이블로 분리한다.
        
    
4. **변경 최소화 (Hash 기반 변경 감지)**
    
    - Bronze 내용이 실제로 바뀐 경우에만 Silver/Gold를 갱신한다.
        
    - SHA-256 해시를 이용해, 동일 payload는 재처리하지 않는다.

---

## 2-2. 전체 아키텍처

<img width="797" height="1534" alt="image" src="https://github.com/user-attachments/assets/dfdb8662-b5db-44b3-8955-f07185546aac" />



> **Bronze는 하나, Silver/Gold는 여러 개일 수 있다.** 모든 Silver/Gold는 원칙적으로 Bronze로부터 재생성 가능하다.
---

### 🥉 Bronze Layer — 원본 그대로 저장

**역할**

- KBO 크롤링 응답을 **그대로(JSON)** 저장한다.
    
- ETL 상태, 해시값 등 최소한의 메타데이터만 붙인다.

**핵심 개념**

- 자연키(Natural Key)로 경기 식별:
    
    date + stadium + home_team + away_team + start_time
    
- payload: KBO 응답 JSON 전체
    
- content_hash: SHA-256(payload)
    
- etl_processed_at: 이 Bronze가 Silver/Gold로 변환된 적 있는지 표시

**해시 계산 예시**
```java
private String calculateHash(final String content) {
    try {
        MessageDigest digest = MessageDigest.getInstance("SHA-256");
        byte[] hash = digest.digest(content.getBytes(StandardCharsets.UTF_8));
        return HexFormat.of().formatHex(hash);
    } catch (NoSuchAlgorithmException e) {
        throw new IllegalStateException("SHA-256 algorithm not available", e);
    }
}
```

#### **Hash 기반 변경 감지**

Adaptive Poller는 1분마다 크롤링한다.
하지만 실제로 경기 데이터가 매번 변하는 건 아니다.
```
예시:
10:32:00 크롤링 → { "score": "3-2", "inning": 4 }
10:33:00 크롤링 → { "score": "3-2", "inning": 4 }  (동일!)
10:34:00 크롤링 → { "score": "4-2", "inning": 4 }  (변경!)
```

동일한 데이터를 매번 Silver/Gold로 변환하면:
- DB 쓰기 부하 증가
- ETL 비용 낭비
- 불필요한 이벤트 발생

따라서 payload 전체를 해시로 변환하고, SHA-256 해시를 비교하였다.

**업서트 시 해시 비교 흐름**
```java
@Transactional
public boolean upsertByNaturalKey(..., String payload) {
    String contentHash = calculateHash(payload);

    return repository.findByNaturalKey(...)
        .map(existing -> updateIfHashChanged(existing, payload, contentHash))
        .orElseGet(() -> createNew(..., payload, contentHash));
}
```

> Bronze는 “기록의 출발점”이자, **디버깅·재처리·새 기능 도입의 안전망** 역할을 한다.

---

### 🥈 Silver Layer — “표준화된 경기 데이터”

Silver는 Bronze의 raw JSON을 읽어, 서비스 전반에서 공통으로 사용할 **정제된 도메인 모델**을 만든다.

**Bronze 원본 (payload JSON):**
```json
{
  "date": "2024-10-15",
  "home_team": "LG",
  "away_team": "SK",
  "stadium": "잠실",
  "home_score": "3",
  "away_score": "2",
  "state": "LIVE"
}
````

Silver 변환

- "LG" → teams.team_id = 5
- "잠실" → stadiums.stadium_id = 1
- "3" → 3 (INT)
- "LIVE" → ENUM('LIVE')

핵심은 타입을 정규화하고 ID/관계(teams, stadiums)를 연결하고 상태를 ENUM으로 수렴시키는 것이다.

Silver는 **“서비스 공통의 기초 데이터”** 이기 때문에, 대부분의 단순 조회 API는 Silver를 직접 보고 동작한다.

### 🥇 Gold Layer — “무거운 집계/랭킹을 위한 별도 모델”

Gold는 모든 기능에 필요한 것이 아니다. 다음 조건을 만족할 때만 Gold로 뽑는다.

- 집계가 무겁다.
    
- 조회가 잦다.
    
- API 응답 구조가 Silver와 꽤 다르다.


**예시: 승리요정 랭킹**

- 연도별/사용자별 승리 기여도를 집계
    
- 체크인/경기 결과를 조합해야 함
    
- 매번 실시간 계산하기에는 비용이 크다

그래서 victory_fairy_rankings 같은 별도 Gold 테이블을 두고,

- ETL 시에 미리 집계
    
- API는 이 Gold를 빠르게 조회

하는 방식으로 설계했다.


반대로, 팬 비율처럼 단순 집계, TTL 캐시 + Silver 조회로 충분한 기능은 굳이 Gold로 분리하지 않고, **Silver + 캐시**로만 처리했다.

### Hash 기반 ETL 흐름

>**ETL = Extract(추출) + Transform(변환) + Load(적재)**
>데이터를 **소스에서 가져와 → 가공해서 → 목적지에 저장**하는 과정

ETL은 크게 두 가지 방식으로 동작한다.
1. **정기 ETL (1분마다)**
2. **이벤트 기반 즉시 ETL (경기 종료)**

<img width="1280" height="681" alt="image" src="https://github.com/user-attachments/assets/68e4397c-8256-4476-9698-65251f52ffcc" />


Bronze의 content_hash를 기준으로:

- 내용이 바뀌지 않았다면 → ETL 스킵
- 바뀌었다면 → Silver/Gold 갱신

이렇게 해서 **변경된 데이터에만 비용을 쓰는 구조**를 만들었다.

## MVP2가 해결한 것

| **문제**             | **Before (MVP1)**  | **After (MVP2)**             |
| ------------------ | ------------------ | ---------------------------- |
| 원본 미보존             | 디버깅/재처리 불가능        | Bronze로 모든 시점 재현 가능          |
| 서비스 로직이 크롤러에 직접 종속 | DOM 변경 시 전체 서비스 영향 | Gold만 조정하거나 ETL만 수정해도 대응 가능  |
| 새 기능 추가            | 크롤러부터 수정 필요        | Bronze 기준 재정제·재집계로 빠르게 실험 가능 |
| 정제 오류              | 복구 불가              | Bronze 기준 전체 재처리 가능          |
> **“앞으로 무슨 기능이 추가될지 모르니까, 원본을 남겨두고 레이어를 나누자”**

# MVP 3. 성능 최적화 — "어떻게 더 빠르고, 더 안전하게 보낼까?"

MVP1·2로

- **언제 가져올지(Adaptive Polling)**
- **무엇을 어떻게 저장·정제할지(Bronze/Silver/Gold)** 가 정리되었다.

이제 남은 질문은 아래와 같았다. 체크인이 몰릴 때 **DB 집계 쿼리**를 어떻게 버틸까? SSE 구독자가 많을 때 **fan-out 폭발**을 어떻게 제어할까?

## 3-1. 초깃값: 직선적인 구조

처음 구현은 매우 직관적이었다.

```java
@Async
@TransactionalEventListener
public void onCheckInCreated(CheckInCreatedEvent event) {
    // 매 이벤트마다 팬 비율 집계 쿼리 실행
    List<GameWithFanRateParam> payload =
        checkInService.buildCheckInEventData(LocalDate.now());

    // 현재 SSE 구독자 전체에 전송
    for (var emitter : sseEmitterRegistry.all()) {
        emitter.send(event().name("check-in-created").data(payload));
    }
}
```

시나리오를 단순화하면:

- 체크인 100건
- SSE 구독자 100명

→ **집계 쿼리 100회 + SSE 전송 10,000회**

문제는 **팬 비율 조회 집계 쿼리 100회**였다.

이 쿼리는 그 자체로 비용이 컸고, 체크인 수에 비례해 선형으로 비싸졌다.

---

## 3-2. 해법: 스케줄러 + 캐시 + 이벤트 핸들러 분리
핵심 인사이트는 단순했다.

> **전송은 이벤트마다 해야 한다.**
> 하지만 **집계는 매번 할 필요가 없다.**

그래서 구조를 이렇게 나눴다.

<img width="863" height="1561" alt="image" src="https://github.com/user-attachments/assets/37a01bee-7ff0-4935-87fc-4b5bec6dcfe4" />


**역할 분리**

1. **스케줄러**
    
    - 5초마다 “오늘 경기 팬 비율”을 계산하고 캐시를 갱신한다.
        
    - 요청이 없더라도, 스스로 신선도를 유지한다.
        
    
2. **이벤트 핸들러**
    
    - 체크인 이벤트가 들어오면 캐시에서 읽어온다.
        
    - 캐시 Miss일 때만 집계 쿼리를 실행한다.
        
    - 결과를 SSE 구독자에게 fan-out 한다.
        
    
3. **TTL은 신선도용이 아니라 “정리용”**
    
    - 캐시 TTL은 15초로 설정했다.
        
    - 신선도는 스케줄러 주기(5초)가 책임진다.
        
    - TTL은 “오랫동안 사용되지 않는 키를 정리”하는 용도에 가깝다.

---

### 3-3. FanRateCache — Caffeine 로컬 캐시

**선택: Caffeine**

- 현재 단일 서버 기준으로는
    
    - 네트워크 왕복 없는 로컬 캐시가 가장 빠르고
        
    - Redis 인프라를 추가로 운영할 이유도 없었다.

```java
@Component
public class FanRateCache {

    private final Cache<LocalDate, List<GameWithFanRateParam>> cache;

    public FanRateCache() {
        this.cache = Caffeine.newBuilder()
                .expireAfterWrite(15, TimeUnit.SECONDS)
                .build();
    }

    public void put(LocalDate date, List<GameWithFanRateParam> data) {
        cache.put(date, data);
    }

    public List<GameWithFanRateParam> getOrCompute(
            LocalDate date,
            Function<LocalDate, List<GameWithFanRateParam>> loader
    ) {
        return cache.get(date, loader);
    }
}
```

- TTL 15초: 메모리 정리용
    
- 신선도: 아래 스케줄러가 5초마다 갱신

### 3-4. FanRateCacheScheduler — 5초마다 강제 갱신
```java
@Slf4j
@Component
@RequiredArgsConstructor
public class FanRateCacheScheduler {

    private final CheckInService checkInService;
    private final FanRateCache fanRateCache;
    private final GameService gameService;

    @Scheduled(fixedDelayString = "PT5S")
    public void updateFanRateCache() {
        if (!gameService.isLiveToday()) {
            return; // Live 경기 없으면 skip
        }

        LocalDate today = LocalDate.now();
        List<GameWithFanRateParam> data =
                checkInService.buildCheckInEventData(today);
        fanRateCache.put(today, data);

        log.debug("Fan rate cache updated: {} games", data.size());
    }
}
```

- 5초마다 DB 집계 쿼리를 한 번 실행
- 같은 값을 여러 이벤트가 공유해 쓰도록 한다.

---
### 3-5. SseEventHandler — 캐시 기반 fan-out
```java
@Slf4j
@Component
@RequiredArgsConstructor
public class SseEventHandler {

    private final FanRateCache fanRateCache;
    private final CheckInService checkInService;
    private final SseEmitterRegistry sseEmitterRegistry;

    @Async
    @TransactionalEventListener
    public void onCheckInCreated(CheckInCreatedEvent event) {
        List<GameWithFanRateParam> payload = fanRateCache.getOrCompute(
                LocalDate.now(),
                checkInService::buildCheckInEventData // 캐시 Miss 시에만 실행
        );

        for (var emitter : sseEmitterRegistry.all()) {
            try {
                emitter.send(SseEmitter.event()
                        .name("check-in-created")
                        .data(payload));
            } catch (IOException e) {
                sseEmitterRegistry.removeWithError(emitter, e);
            }
        }
    }
}
```

- 체크인이 발생할 때마다 SSE 전송은 유지한다.
- 하지만 대부분의 경우, **집계는 이미 스케줄러가 계산해 둔 값을 재사용**한다.

### **3-6. 부하 테스트 결과: Q/E = 1.0 → 0.05**
#### Before
<img width="1280" height="60" alt="image" src="https://github.com/user-attachments/assets/f3239b4d-6a04-4209-862c-b5940a7d170d" />

#### After
<img width="1280" height="53" alt="image" src="https://github.com/user-attachments/assets/f6829e1f-6099-4122-b3ed-0adb755de775" />

**테스트 조건**

- 체크인 100건 (10 rps × 10초)
    
- SSE 구독자 100명
    
- 측정 시간: 총 25초 (체크인 10초 + 관찰 15초)
    
- 스케줄러: 5초 주기

| **지표**      | **Before** | **After**    | **개선율**    |
| ----------- | ---------- | ------------ | ---------- |
| 체크인 이벤트 수   | 100        | 100          | -          |
| 집계 쿼리 수     | 100        | **5**        | **95% 감소** |
| Q/E         | 1.000      | **0.050**    | **95% 감소** |
| SSE 전송 횟수   | 10,000     | 10,000       | 동일         |
| p95 latency | 130.14ms   | **105.02ms** | 약 19% 개선   |

**Q/E 의미:**

- Before: 체크인 1건당 집계 쿼리 1회
- After: 체크인 20건당 집계 쿼리 1회 (5초 주기)
- **DB 부하가 체크인 수가 아니라 시간에만 의존하게 변경**

**트레이드오프**

- 완벽한 실시간성 대신, 팬 비율이 **최대 5초 늦게 반영될 수 있다.**
  
  대신
    
    - DB 부하 95% 감소
        
    - 체크인이 몰려도 안정적
        
    - 구독자 수 증가와 DB 부하가 분리된다.

---

### **3-7. 확장: 서버가 2대 이상일 때의 고민 (요약)**

현재는 단일 서버 기준으로 Caffeine 로컬 캐시가 가장 합리적이다.

만약 서버가 2대 이상으로 확장된다면:

- **문제**: 서버 A 캐시와 서버 B 캐시가 다를 수 있다.
    
- **후보**
    
    - Redis로 캐시 중앙화
        
    - Redis Pub/Sub + 각 서버 로컬 캐시 무효화
        
    - Load Balancer의 Sticky Session으로 어느 정도 완화
        
    
지금은 “단일 서버 + `Caffeine`”으로 충분하고, 서버가 여러대라면 후보들을 고민해보아야한다.
 직관 인증 데이터는 사용자 기록(통계, 랭킹, 배지)에 즉시 반영되어야 하므로 조회 성능보다 일관성이 더 중요하다. 
> 경기 정보 데이터는 조회 성능을 위해 Pub/Sub + 로컬 캐시를 고려할 수 있을 것 같다.

## 3-4. 최종 성과 정리

실시간 KBO 데이터는 **“완성된 결과”가 아니라 “경기 도중 계속 바뀌는 상태”** 다.
야구보구는 이를 다루기 위해 세 단계로 구조를 만들었다.

|**단계**|**핵심 질문**|**주요 전략**|
|---|---|---|
|MVP1|**언제 가져올까?**|상태 기반 Adaptive Polling|
|MVP2|**무엇을 어떻게 저장할까?**|Bronze/Silver/Gold 파이프라인|
|MVP3|**어떻게 빠르게 보낼까?**|TTL 캐시 + 스케줄러 + SSE 분리|

## 최종 성과 정리

### **안정성**

- KBO DOM·필드 구조가 바뀌어도, **Bronze 원본을 기준으로 ETL만 수정해 복구** 가능.
    
- 서비스 로직을 크롤러에서 분리해 기능 추가·변경 시 영향 범위를 줄였다.
### **성능**

- **크롤링 효율**: 상태 기반 폴링으로 API 호출 88% 감소
    
- **DB 부하**: 팬 비율 집계 쿼리 95% 감소
    - Q/E: 1.0 → 0.05
        
    
- **응답 속도**: p95 latency 약 130ms → 105ms
    
- **반영 시간**: 하루 1회(최대 7시간 지연) → 1분 이내 반영

### **확장성**

- Bronze 기반으로 승리 요정 랭킹, 팬 통계, ML 피처 등 새 기능을 쉽게 추가
    
- 필요한 기능만 Gold로 분리해 무거운 집계를 안정적으로 제공
        
- 캐시·스케줄러·SSE 레이어를 나누어 트래픽 증가에도 단계적으로 대응할 수 있는 여지를 확보

---

## To be continued...

**실시간 웹데이터를 안정적으로 다루기 위해 최소한 갖춰야 할 뼈대**를 만들어보았다. 더 해보고 싶은 것은, 파이프라인에서 장애가 났을 때 복구하는 부분과 홈런·도루 등 특정 상황을 자동으로 탐지하는 것을 구현해보고 싶다.
