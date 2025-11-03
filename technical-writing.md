### 이 문서가 다루는 것

이 문서는 MySQL Primary-Replica 레플리케이션에서 발생하는 복제 지연 문제를 다룹니다.

PACELC 정리의 지연시간 vs 일관성 관점에서 문제를 분석하고, 비동기 복제와 준동기 복제를 통해 일관성 수준을 조절하는 방법을 설명합니다.

### 누가 읽으면 좋은가

이 문서는 다음과 같은 분들에게 도움이 됩니다.

- **레플리케이션 도입을 고려하는 개발자**: 읽기 부하가 증가하여 레플리케이션 도입을 검토 중이라면, 어떤 트레이드오프가 있는지 미리 이해할 수 있습니다.
- **복제 지연 문제를 겪고 있는 백엔드 엔지니어**: 이미 레플리케이션을 사용 중이지만 데이터 불일치 문제로 고민한다면, 일관성 수준별 해결 방법을 찾을 수 있습니다.
- **시스템의 트레이드오프를 이해하고 싶은 누구나**: 분산 시스템에서 일관성과 성능 사이의 트레이드오프를 실제 사례로 이해하고 싶다면 유용합니다.

## MySQL Replication

### 해결하려는 문제

MySQL 서버 하나로 서비스를 운영하다 보면 피할 수 없는 한계에 부딪힙니다.

**읽기 부하 증가**

사용자가 늘어나면 데이터베이스에 대한 읽기 요청이 급증합니다. 일반적인 웹 서비스는 쓰기보다 읽기가 압도적으로 많습니다. 단일 MySQL 서버는 초당 처리할 수 있는 쿼리 수에 한계가 있으며, 수직 확장(Scale-up)으로는 비용 대비 성능 향상이 제한적입니다.

**단일 장애 지점 (SPOF)**

단일 서버가 다운되면 서비스 전체가 중단됩니다.

이러한 문제를 해결하기 위해 MySQL 레플리케이션을 도입합니다. 데이터를 여러 서버에 복제해 읽기 부하를 분산하고, 가용성을 확보합니다.

### 레플리케이션으로 얻는 것

**읽기 성능 향상**

Primary 서버 하나에 Replica 서버 3대를 추가하면 읽기 처리량을 이론적으로 4배까지 늘릴 수 있습니다. 상품 목록 조회, 검색 같은 읽기 쿼리를 Replica로 분산시켜 Primary는 주문 생성, 재고 업데이트 같은 쓰기 작업에 집중하게 만듭니다.

**읽기 가용성 확보**

Replica가 여러 대 있으면 일부 서버에 장애가 발생해도 읽기 서비스는 계속 제공할 수 있습니다. Primary 서버에 장애가 발생해도 Replica 중 하나를 Primary로 승격시켜 쓰기 기능을 복구할 수 있습니다.

**백업 부담 분산**

백업 작업을 Replica에서 수행하면 Primary 서버에 영향을 주지 않습니다.

### 고려해야 할 단점

모든 기술적 선택에는 트레이드오프가 있습니다. 레플리케이션도 예외가 아닙니다.

**관리 복잡도 증가**

서버가 1대에서 4대로 늘어나면 관리해야 할 대상이 4배가 됩니다. 각 서버의 상태를 모니터링해야 하고, 설정을 동기화해야 하며, 버전 업그레이드도 순차적으로 진행해야 합니다.

**인프라 비용 증가**

서버 대수가 늘어나면 하드웨어 비용, 네트워크 비용이 증가합니다.

**복제 지연**

Primary에서 데이터를 쓴 직후 Replica에서 읽으면 최신 데이터가 보이지 않을 수 있습니다. 이것이 이 문서의 핵심 주제입니다.

## 복제 지연 문제

### 실제 사례

"주문 완료했는데 내역이 안 보여요"

사용자가 주문을 완료했지만 마이페이지에서 주문 내역이 보이지 않습니다. 결제는 정상 처리되었고 데이터베이스에 주문 데이터도 있는데, 왜 사용자 화면에는 보이지 않았을까요?

원인은 복제 지연이었습니다. 주문 생성은 Primary 서버에 기록되었지만, 주문 조회는 Replica 서버에서 읽고 있었습니다. Replica가 Primary의 데이터를 복제하는 동안 수 초의 지연이 발생했고, 사용자는 그 짧은 순간에 주문 내역을 조회한 것입니다.

### 분산 시스템의 트레이드오프: PACELC

단일 MySQL 서버를 사용할 때는 데이터를 쓰면 바로 읽을 수 있었고, 일관성은 자동으로 보장되었습니다. 하지만 레플리케이션을 도입하는 순간 분산 시스템을 만든 것입니다. 분산 시스템에는 피할 수 없는 물리적 한계가 있습니다.

PACELC 정리는 분산 시스템에서 두 가지 상황에서 선택을 해야 한다고 설명합니다.

- **P (Partition) - 장애 발생 시**: A (빠른 복구, 일부 데이터 손실 가능) vs C (느린 복구, 데이터 보존)
- **E (Else) - 정상 상황에서**: L (빠른 응답, 복제 지연 허용) vs C (느린 응답, 데이터 일관성 보장)

이 문서에서는 정상 상황(E)에서 운영 중 마주하는 복제 지연 문제, 즉 **L vs C 트레이드오프**에 집중합니다.

## 복제 지연 분석과 해결

### 비동기 복제의 동작 원리

MySQL의 기본 복제 방식은 비동기입니다. Primary에서 트랜잭션을 커밋하면 즉시 클라이언트에게 성공 응답을 보냅니다. Replica에 데이터가 전송되었는지 확인하지 않습니다.

1. 클라이언트 → Primary: INSERT INTO ...
2. Primary: 데이터 저장, binlog에 기록
3. Primary → 클라이언트: "성공" (즉시 응답)
4. Primary → Replica: binlog 전송
5. Replica: relay log에 저장 후 SQL 실행

응답 시간: Primary의 쓰기 시간만 (1~5ms)
복제 지연: 네트워크 지연 + SQL 실행 시간 (수 밀리초 ~ 수 초)

### 복제 지연 확인

Replica에서 `SHOW SLAVE STATUS` 명령으로 복제 지연을 확인할 수 있습니다.

```sql
SHOW SLAVE STATUS\G

Slave_IO_Running: Yes     -- binlog 수신 스레드 상태
Slave_SQL_Running: Yes    -- SQL 실행 스레드 상태
Seconds_Behind_Master: 2  -- 복제 지연 시간 (초)
```

`Seconds_Behind_Master`는 Primary의 binlog 이벤트 타임스탬프와 Replica가 현재 실행 중인 이벤트의 타임스탬프 차이입니다. 0이면 동기화된 상태입니다.

지연이 커지는 주요 원인:
- 네트워크가 느리거나 불안정함
- Replica에 부하가 많아 SQL 실행이 느림
- Primary에서 대량의 UPDATE/DELETE가 발생
- Replica가 단일 스레드로 실행 (병렬 복제 미설정)

### 발생하는 일관성 문제

**Read-after-Write 불일치**

사용자가 데이터를 쓴 직후 읽었을 때 방금 쓴 데이터가 보이지 않는 현상입니다. 사용자는 "방금 주문했는데 왜 안 보이지?"라고 생각하고 같은 주문을 다시 시도할 수 있습니다.

**Monotonic Read 위반**

같은 데이터를 여러 번 조회할 때 시간이 거꾸로 가는 것처럼 보이는 현상입니다. 여러 Replica가 있고 로드밸런서가 요청을 분산할 때, 각 Replica의 복제 지연이 다르면 이런 문제가 발생합니다.

### 준동기 복제로 일관성 조절

준동기 복제에서는 Primary가 트랜잭션을 커밋한 후, 최소 N개의 Replica로부터 "데이터 받았어요"라는 확인(ACK)을 받을 때까지 기다립니다. 그리고 나서 클라이언트에게 성공 응답을 보냅니다.

1. 클라이언트 → Primary: INSERT INTO orders ...
2. Primary: 데이터 저장, binlog에 기록
3. Primary → Replica들: binlog 전송
4. Replica들: relay log에 저장
5. Replica들 → Primary: "받았어요" ACK
6. Primary: N개 이상의 ACK 확인
7. Primary → 클라이언트: "성공" (ACK 받은 후 응답)

응답 시간: Primary 쓰기 + 네트워크 왕복 (5~20ms)
복제 보장: 최소 N개 Replica는 데이터 보유 확실

**ACK 개수로 일관성 조절**

`rpl_semi_sync_master_wait_for_slave_count` 설정으로 몇 개의 Replica로부터 ACK를 받을지 결정합니다.

**1개 Replica (최소 보장)**

```sql
SET GLOBAL rpl_semi_sync_master_wait_for_slave_count = 1;
```
- 응답 시간: 가장 빠른 1개 Replica의 네트워크 왕복 시간만 추가
- 일관성: 최소 1개는 데이터 보유

**과반수 Replica (권장)**

```sql
-- Replica 3대 중 2개
SET GLOBAL rpl_semi_sync_master_wait_for_slave_count = 2;
```
- 응답 시간: 두 번째로 빠른 Replica까지 대기
- 일관성: 과반수가 데이터 보유

**모든 Replica (최강 일관성)**

```sql
SET GLOBAL rpl_semi_sync_master_wait_for_slave_count = 3;
```
- 응답 시간: 가장 느린 Replica까지 대기
- 일관성: 모든 Replica가 데이터 보유

**주의사항**: 준동기 복제에서 Replica가 "받았다"는 것은 relay log에 저장했다는 의미입니다. 아직 SQL을 실행해서 데이터를 적용하지는 않았을 수 있으므로, 약간의 복제 지연은 여전히 존재할 수 있습니다.

## 모니터링과 운영

### 자동화된 모니터링

실무에서는 자동화된 모니터링 시스템이 필수입니다.

**Prometheus + MySQL Exporter 예시**

```yaml
# Alert 규칙
groups:
  - name: mysql_replication
    rules:
      - alert: ReplicationLag
        expr: mysql_slave_status_seconds_behind_master > 10
        for: 1m
```

**모니터링 지표**

- Seconds_Behind_Master 시계열 그래프
- 시간대별 복제 지연 추이
- Replica별 비교 차트

### 복제 지연 대응

**복제 지연 5초 이상 발생 시**

1. 원인 파악: Replica 서버 부하, Primary의 대량 쓰기 작업, 네트워크 지연 확인
2. 즉시 조치: 읽기 부하가 많으면 일시적으로 Primary로 라우팅
3. 장기 대응: 병렬 복제 설정, Replica 하드웨어 성능 향상, Primary의 배치 작업 시간 조정

### 복제 지연을 줄이는 방법

**병렬 복제 설정**

MySQL 5.7부터는 여러 스레드로 relay log를 병렬 실행할 수 있습니다.

```sql
SET GLOBAL slave_parallel_workers = 4;
SET GLOBAL slave_parallel_type = 'LOGICAL_CLOCK';
```

**네트워크 최적화**

- 같은 데이터센터 또는 가용 영역(AZ) 배치
- 전용 네트워크 사용
- 네트워크 대역폭 확보 (1Gbps 이상)

**Primary의 쓰기 최적화**

대량 작업은 Replica에 큰 부담을 줍니다.

```sql
-- 나쁜 예: 100만 건 한 번에 업데이트
UPDATE products SET status = 'inactive' WHERE created_at < '2020-01-01';

-- 좋은 예: 1000건씩 나눠서 실행
UPDATE products SET status = 'inactive'
WHERE created_at < '2020-01-01' LIMIT 1000;
-- 1초 대기 후 반복
```

## 결론

MySQL 레플리케이션은 단순히 서버를 추가하는 것이 아니라 분산 시스템을 도입하는 것입니다. 분산 시스템에는 PACELC 정리가 적용되며, 빠른 응답(Latency)과 데이터 일관성(Consistency) 사이에서 트레이드오프해야 합니다.

비동기 복제는 빠르지만 일관성을 포기하고, 준동기 복제는 일관성을 높이지만 성능을 포기합니다. 어느 쪽을 선택할 것인가는 비즈니스 요구사항에 따라 다릅니다.

### 더 알아보기

**MySQL Group Replication**

전통적인 Primary-Replica 레플리케이션으로 해결하기 어려운 요구사항이 있다면 MySQL Group Replication을 검토해보세요. 정족수 기반 합의 알고리즘으로 강한 일관성을 자동으로 보장하고, Multi-Primary도 지원합니다.

**참고 자료**

- MySQL 공식 문서: Replication

이 문서는 MySQL 8.0 기준으로 설명합니다.
