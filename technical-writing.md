### 이 문서가 다루는 것

이 문서는 MySQL Primary-Replica 레플리케이션에서 발생하는 복제 지연 문제를 다룹니다. PACELC 정리의 평상시 지연시간 vs 일관성 관점에서 문제를 분석하고, 비동기 복제와 준동기 복제를 통해 일관성 수준을 조절하는 방법을 설명합니다.

### 누가 읽으면 좋은가

이 문서는 다음과 같은 분들에게 도움이 됩니다.

- 레플리케이션 도입을 고려하는 개발자

읽기 부하가 증가하여 레플리케이션 도입을 검토 중이라면, 어떤 트레이드오프가 있는지 미리 이해할 수 있습니다.

- 복제 지연 문제를 겪고 있는 백엔드 엔지니어

이미 레플리케이션을 사용 중이지만 데이터 불일치 문제로 고민한다면, 일관성 수준별 해결 방법을 찾을 수 있습니다.

- 시스템의 트레이드오프를 이해하고 싶은 누구나

분산 시스템에서 일관성과 성능 사이의 트레이드오프를 실제 사례로 이해하고 싶다면 유용합니다.

## 1. MySQL Replication

### 1. 해결하려는 문제

MySQL 서버 하나로 서비스를 운영하다 보면 피할 수 없는 한계에 부딪힙니다.

**읽기 부하 증가**

사용자가 늘어나면 데이터베이스에 대한 읽기 요청이 급증합니다. 일반적인 웹 서비스는 쓰기보다 읽기가 압도적으로 많습니다. 한 전자상거래 사이트를 예로 들면, 주문 생성(쓰기)은 하루 1만 건이지만 상품 조회(읽기)는 100만 건이 넘습니다. 읽기와 쓰기의 비율이 100:1인 상황입니다.

단일 MySQL 서버는 초당 처리할 수 있는 쿼리 수에 한계가 있습니다. CPU 코어를 늘리거나 메모리를 증설하는 수직 확장(Scale-up)으로는 비용 대비 성능 향상이 제한적입니다.

**단일 장애 지점 (SPOF)**

단일 서버가 다운되면 서비스 전체가 중단됩니다. 백업에서 데이터를 복원하는 시간 동안 서비스를 제공할 수 없습니다.

백업 작업도 문제입니다. mysqldump로 전체 데이터를 백업하는 동안 서버에 부하가 걸려 서비스 응답 시간이 길어집니다.

이러한 문제를 해결하기 위해 MySQL 레플리케이션을 도입합니다. 데이터를 여러 서버에 복제해 읽기 부하를 분산하고, 가용성을 확보합니다.

### 2. 레플리케이션으로 얻는 것

레플리케이션을 도입하면 세 가지 핵심 이점을 얻습니다.

**읽기 성능 향상**

Primary 서버 하나에 Replica 서버 3대를 추가하면 읽기 처리량을 이론적으로 4배까지 늘릴 수 있습니다. 상품 목록 조회, 검색, 통계 조회 같은 읽기 쿼리를 Replica로 분산시켜 Primary는 주문 생성, 재고 업데이트 같은 쓰기 작업에 집중하게 만듭니다.

**읽기 가용성 확보**

Replica가 여러 대 있으면 일부 서버에 장애가 발생해도 읽기 서비스는 계속 제공할 수 있습니다. Replica 4대 중 1대가 다운되어도 나머지 3대가 읽기 요청을 처리합니다. 사용자는 장애를 인지하지 못합니다.

Primary 서버에 장애가 발생해도 Replica 중 하나를 Primary로 승격시켜 쓰기 기능을 복구할 수 있습니다. 단일 서버 환경보다 훨씬 빠르게 서비스를 재개할 수 있습니다.

**백업 부담 분산**

백업 작업을 Replica에서 수행하면 Primary 서버에 영향을 주지 않습니다. 사용자가 몰리는 시간대에도 백업을 진행할 수 있습니다. Replica 하나를 백업 전용으로 운영하는 것도 가능합니다.

또한 Replica에서 분석 쿼리나 배치 작업을 실행할 수 있습니다. 시간이 오래 걸리는 데이터 집계 작업을 Replica에서 처리하면 Primary의 운영 쿼리에 영향을 주지 않습니다.

### 3. 고려해야할 단점

레플리케이션을 도입해서 읽기 부하와 단일 장애점 문제를 해결했습니다. 이제 완전 해결된 걸까요?

모든 기술적 선택에는 트레이드오프가 있습니다. 레플리케이션도 예외가 아닙니다.

**관리 복잡도 증가**

서버가 1대에서 4대로 늘어나면 관리해야 할 대상이 4배가 됩니다. 각 서버의 상태를 모니터링해야 하고, 설정을 동기화해야 하며, 버전 업그레이드도 순차적으로 진행해야 합니다.

복제가 중단되거나 지연되는 상황을 감지하고 대응하는 운영 프로세스가 필요합니다. 장애 발생 시 어느 서버가 최신 데이터를 가지고 있는지 파악하는 절차도 마련해야 합니다.

**인프라 비용 증가**

서버 대수가 늘어나면 하드웨어 비용, 네트워크 비용, 전력 비용이 증가합니다. 클라우드 환경이라면 인스턴스 비용이 서버 대수에 비례해서 늘어납니다.

**복제 지연**

Primary에서 데이터를 쓴 직후 Replica에서 읽으면 최신 데이터가 보이지 않을 수 있습니다.

이 문제는 이후 섹션에서 상세히 다룹니다.

레플리케이션은 단순히 서버를 추가하는 것이 아니라 분산 시스템을 도입하는 것입니다. 분산 시스템에는 일관성과 가용성, 응답 지연 사이의 트레이드오프를 이해하고 선택해야 합니다.

## 2. Replication이 만드는 골치아픈 문제

### 2-1. 복제 지연

**"주문 완료했는데 내역이 안 보여요"**

사용자가 주문을 완료했지만 마이페이지에서 주문 내역이 보이지 않는다는 것입니다. 결제는 정상적으로 처리되었고, 데이터베이스를 직접 확인해보니 주문 데이터도 있었습니다. 그런데 왜 사용자 화면에는 보이지 않았을까요?

원인은 복제 지연이었습니다. 주문 생성은 Primary 서버에 기록되었지만, 주문 조회는 Replica 서버에서 읽고 있었습니다. Replica가 Primary의 데이터를 복제하는 동안 수 초의 지연이 발생했고, 사용자는 그 짧은 순간에 주문 내역을 조회한 것입니다.

## 3. 분산 시스템이 만드는 트레이드오프

왜 이런 문제가 발생하는 걸까요?

단일 MySQL 서버를 사용할 때는 이런 고민이 없었습니다. 데이터를 쓰면 바로 읽을 수 있었고, 일관성은 자동으로 보장되었습니다. 하지만 레플리케이션을 도입하는 순간, 분산 시스템을 만든 것입니다. 데이터가 여러 서버에 흩어져 있고, 네트워크를 통해 동기화됩니다.

분산 시스템에는 피할 수 없는 물리적 한계가 있습니다. 네트워크는 완벽하지 않고, 서버는 언제든 다운될 수 있습니다. 이러한 한계 때문에 선택을 강요받습니다.

### 3-1. PACELC 정리: 분산 시스템의 선택

분산 시스템을 설계할 때 자주 언급되는 것이 CAP 정리입니다. 일관성(Consistency), 가용성(Availability), 분할 내성(Partition tolerance) 중 두 가지만 선택할 수 있다는 이론입니다. 하지만 CAP 정리는 네트워크 분할(장애) 상황만 다룹니다.

PACELC 정리는 CAP를 확장한 개념입니다. 분산 시스템에서는 두 가지 상황에서 선택을 해야 합니다.

**P (Partition) - 장애 발생 시**

- A (Availability): 빠른 복구, 일부 데이터 손실 가능
- C (Consistency): 느린 복구, 데이터 보존

**E (Else) - 정상 상황에서**

- L (Latency): 빠른 응답, 복제 지연 허용
- C (Consistency): 느린 응답, 데이터 일관성 보장

MySQL 레플리케이션에서 마주하는 문제가 바로 이 PACELC의 상황입니다.

이 문서에서는 정상 상황(E)에서 운영 중 마주하는 복제 지연 문제, 즉 **L vs C 트레이드오프**에 집중합니다.

### 3-2. MySQL 레플리케이션의 일관성 트레이드 오프

**응답 속도 vs 일관성**

Primary가 정상적으로 동작할 때, 우리는 빠른 응답과 데이터 일관성 사이에서 선택해야 합니다. 이것이 복제 지연 문제의 본질입니다.

비동기 복제는 빠르지만 일관성을 포기하고, 준동기 복제는 일관성을 높이지만 성능을 포기합니다. 어느 쪽을 선택할 것인가는 비즈니스 요구사항에 따라 다릅니다. 비즈니스 적으로 데이터 일관성이 중요하다면, 성능 저하를 감수할 수 있습니다.

## 4. 복제 지연과 일관성 트레이드오프

이제 복제 지연 문제가 PACELC의 ELC 트레이드오프임을 이해했습니다. 다음으로 이 문제를 구체적으로 분석하고, 일관성 수준을 조절하는 다양한 방법을 살펴보겠습니다.

### 4-1. 복제 지연이란

**비동기 복제의 동작 원리**

MySQL의 기본 복제 방식은 비동기입니다. Primary에서 트랜잭션을 커밋하면 즉시 클라이언트에게 성공 응답을 보냅니다. Replica에 데이터가 전송되었는지 확인하지 않습니다.

```java
1. 클라이언트 → Primary: INSERT INTO orders ...
2. Primary: 데이터 저장, binlog에 기록
3. Primary → 클라이언트: "성공" (즉시 응답)
4. Primary → Replica: binlog 전송 (바이너리 로그 덤프 스레드)
5. Replica: relay log에 저장 (I/O 스레드)
6. Replica: relay log 읽어서 SQL 실행 (SQL 스레드)

응답 시간: Primary의 쓰기 시간만 (1~5ms)
복제 지연: 네트워크 지연 + SQL 실행 시간 (수 밀리초 ~ 수 초)
```

**복제 지연 확인 방법**

Replica에서 `SHOW SLAVE STATUS` 명령으로 복제 지연을 확인할 수 있습니다.

```sql
SHOW SLAVE STATUS\G

Seconds_Behind_Master: 2
```

`Seconds_Behind_Master`는 Primary의 binlog 이벤트 타임스탬프와 Replica가 현재 실행 중인 이벤트의 타임스탬프 차이입니다. 0이면 동기화된 상태입니다.

**지연이 커지는 원인**

복제 지연은 여러 이유로 발생합니다.

- 네트워크가 느리거나 불안정함
- Replica에 부하가 많아 SQL 실행이 느림
- Primary에서 대량의 UPDATE/DELETE가 발생
- Replica가 단일 스레드로 실행 (병렬 복제 미설정)

실무에서는 평상시 50ms 이하로 유지되다가, 배치 작업이나 갑작스런 트래픽 증가 시 수 초까지 늘어나는 경우가 많습니다.

### 4-2. 운영 중 발생하는 문제들

복제 지연으로 인해 두 가지 일관성 문제가 발생합니다.

**Read-after-Write 불일치**

사용자가 데이터를 쓴 직후 읽었을 때 방금 쓴 데이터가 보이지 않는 현상입니다.

```sql
사용자: 주문 생성 (Primary에 기록, 3ms 소요)
시스템: "주문이 완료되었습니다" 응답
사용자: 마이페이지 클릭 (즉시, 0.5초 후)
시스템: Replica에서 조회 (아직 복제 안 됨, 50ms 지연)
사용자: 주문 내역이 없음! "버그인가?"
```

이 문제는 특히 쓰기 직후 읽기가 연속으로 일어나는 경우에 자주 발생합니다. 사용자는 "방금 주문했는데 왜 안 보이지?"라고 생각하고 같은 주문을 다시 시도할 수 있습니다.

**Monotonic Read 위반**

같은 데이터를 여러 번 조회할 때 시간이 거꾸로 가는 것처럼 보이는 현상입니다.

```sql
사용자: 리뷰 작성 (Primary에 기록)

첫 번째 조회:
로드밸런서 → Replica A (이미 복제됨, 지연 10ms)
결과: 내 리뷰가 보임 ✅

두 번째 조회 (새로고침):
로드밸런서 → Replica B (아직 복제 안 됨, 지연 200ms)
결과: 내 리뷰가 없음 ❌

사용자: "방금까지 있었는데 사라졌네? 삭제된 건가?"`
```

여러 Replica가 있고 로드밸런서가 요청을 분산할 때, 각 Replica의 복제 지연이 다르면 이런 문제가 발생합니다.

### 4-3. 일관성 수준 조절하기

일관성과 성능 사이에서 어떻게 균형을 맞출 수 있을까요? MySQL은 준동기 복제를 통해 일관성 수준을 조절할 수 있는 방법을 제공합니다.

**준동기 복제: ACK를 기다립니다**

준동기 복제에서는 Primary가 트랜잭션을 커밋한 후, 최소 N개의 Replica로부터 "데이터 받았어요"라는 확인(ACK)을 받을 때까지 기다립니다. 그리고 나서 클라이언트에게 성공 응답을 보냅니다.

```sql
1. 클라이언트 → Primary: INSERT INTO orders ...
2. Primary: 데이터 저장, binlog에 기록
3. Primary → Replica들: binlog 전송
4. Replica들: relay log에 저장
5. Replica들 → Primary: "받았어요" ACK
6. Primary: N개 이상의 ACK 확인
7. Primary → 클라이언트: "성공" (ACK 받은 후 응답)

응답 시간: Primary 쓰기 + 네트워크 왕복 (5~20ms)
복제 보장: 최소 N개 Replica는 데이터 보유 확실
```

**ACK 개수로 일관성 조절**

`rpl_semi_sync_master_wait_for_slave_count` 설정으로 몇 개의 Replica로부터 ACK를 받을지 결정합니다. Master 서버 1대, Replica 서버 3대의 경우로 예시를 들어보겠습니다.

**1개 Replica (최소 보장)**

`SET GLOBAL rpl_semi_sync_master_wait_for_slave_count = 1;`

- 응답 시간: 가장 빠른 1개 Replica의 네트워크 왕복 시간만 추가
- 일관성: 최소 1개는 데이터 보유

```sql
Replica 3대 구성:
- Replica A: 80ms 지연 (가장 빠름)
- Replica B: 150ms 지연
- Replica C: 300ms 지연

Primary는 80ms만 대기 (A의 ACK만 받으면 됨)
```

**과반수 Replica (균형잡힌 선택)**

- Replica 3대 중 2개

`SET GLOBAL rpl_semi_sync_master_wait_for_slave_count = 2;`

- 응답 시간: 두 번째로 빠른 Replica까지 대기
- 일관성: 과반수가 데이터 보유

```sql
Replica 3대 구성:
- Replica A: 80ms
- Replica B: 150ms (여기까지 대기)
- Replica C: 300ms

Primary는 150ms 대기 (A, B의 ACK 받음)
```

**모든 Replica (최대 보장)**

- Replica 3대 모두

`SET GLOBAL rpl_semi_sync_master_wait_for_slave_count = 3;`

- 응답 시간: 가장 느린 Replica까지 대기
- 일관성: 모든 Replica가 데이터 보유

```sql
`Replica 3대 구성:
- Replica A: 80ms ✅
- Replica B: 150ms ✅
- Replica C: 300ms ✅ (가장 느린 것까지 대기)

Primary는 300ms 대기 (모든 ACK 받음)
```

**주의사항**

준동기 복제에서 Replica가 "받았다"는 것은 relay log에 저장했다는 의미입니다. 아직 SQL을 실행해서 데이터를 적용하지는 않았을 수 있습니다. 따라서 여전히 약간의 복제 지연은 존재할 수 있습니다.

## 5. 모니터링과 운영

복제 지연에 대응하는 전략을 세웠다면, 이제 실제로 운영하면서 복제 상태를 모니터링하고 관리해야 합니다.

### 5-1. 복제 지연 모니터링

SHOW SLAVE STATUS로 확인

Replica에서 복제 상태를 확인하는 가장 기본적인 방법입니다.

```sql
SHOW SLAVE STATUS\G

*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: primary.example.com
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
        Seconds_Behind_Master: 2
         Retrieved_Gtid_Set: 3e11fa47-71ca-11e1-9e33-c80aa9429562:1-100
          Executed_Gtid_Set: 3e11fa47-71ca-11e1-9e33-c80aa9429562:1-98

```

**주요 지표**

```
Slave_IO_Running: Yes/No
→ Primary에서 binlog를 받아오는 스레드 상태
→ No면 네트워크 문제 또는 인증 실패

Slave_SQL_Running: Yes/No
→ relay log를 실행하는 스레드 상태
→ No면 SQL 실행 오류 (중복 키, 권한 등)

Seconds_Behind_Master: 숫자
→ Primary보다 몇 초 뒤처져 있는가
→ NULL이면 복제가 멈춘 상태
→ 0이면 완벽히 동기화

Last_IO_Error, Last_SQL_Error
→ 복제 중단 시 에러 메시지
```

`Seconds_Behind_Master`는 Primary의 binlog 이벤트 타임스탬프와 Replica가 현재 실행 중인 이벤트의 타임스탬프 차이입니다. 이 값이 0이면 Replica가 Primary와 완전히 동기화된 상태입니다.

**자동화된 모니터링**

실무에서는 수동으로 매번 확인할 수 없으므로 자동화된 모니터링 시스템을 구축해야 합니다.

Prometheus + MySQL Exporter를 사용한 예시:

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'mysql'
    static_configs:
      - targets: ['replica1:9104', 'replica2:9104', 'replica3:9104']

# Alert 규칙
groups:
  - name: mysql_replication
    rules:
      - alert: ReplicationLag
        expr: mysql_slave_status_seconds_behind_master > 10
        for: 1m
        annotations:
          summary: "Replica {{ $labels.instance }} lag > 10 seconds"
```

**실시간 대시보드**

Grafana로 복제 상태를 시각화합니다.

- Seconds_Behind_Master 시계열 그래프
- 시간대별 복제 지연 추이
- Replica별 비교 차트
- 임계값 표시선
- 복제 중단 알림 패널

대시보드를 보면 "평상시 지연은 100ms인데 오후 3시에 5초까지 올라갔네?" 같은 패턴을 발견할 수 있습니다.

### 5-2. 대응 방안

복제 지연을 언제 경고할지 임계값을 설정하고 대응합니다.

```
복제 지연 5초 이상 발생 시:

1. 원인 파악
   - Replica 서버 부하 확인 (CPU, Disk I/O)
   - Primary의 대량 쓰기 작업 확인
   - 네트워크 지연 확인
   - SHOW PROCESSLIST로 긴 쿼리 확인

2. 즉시 조치
   - 읽기 부하가 많으면 일시적으로 Primary로 라우팅
   - 긴 SELECT 쿼리 종료 (복제 방해하는 경우)
   - 불필요한 인덱스 제거 검토

3. 장기 대응
   - 병렬 복제 설정 확인
   - Replica 하드웨어 성능 향상 검토
   - Primary의 배치 작업 시간 조정
```

**복제 중단 대응**

```
Slave_SQL_Running: No 인 경우:

1. SHOW SLAVE STATUS에서 Last_SQL_Error 확인

2. 에러 유형별 대응
   - 중복 키: 수동으로 건너뛰기 or 데이터 정정
   - 권한 문제: GRANT 재설정
   - 테이블 없음: 스키마 동기화

3. 복제 재시작
   START SLAVE;

4. 확인
   Slave_SQL_Running이 Yes인지 확인
```

### 5-3. **복제 지연을 줄이는 방법**

**1. 병렬 복제 설정**

MySQL 5.7부터는 여러 스레드로 relay log를 병렬 실행할 수 있습니다.

```sql
-- Replica에서 설정
SET GLOBAL slave_parallel_workers = 4;
SET GLOBAL slave_parallel_type = 'LOGICAL_CLOCK';

-- 효과: SQL 실행을 병렬로 처리하여 지연 감소
-- 주의: CPU 코어 수에 맞게 조정
```

**2. 네트워크 최적화**

네트워크 지연도 복제 지연의 한 요소입니다.

```
Primary와 Replica 간 네트워크 지연 최소화:
- 같은 데이터센터 또는 가용 영역(AZ) 배치
- 전용 네트워크 사용 (public 인터넷 회피)
- 네트워크 대역폭 확보 (1Gbps 이상)
```

**3. Primary의 쓰기 최적화**

대량 작업은 Replica에 큰 부담을 줍니다.

```sql
-- 나쁜 예: 100만 건 한 번에 업데이트
UPDATE products SET status = 'inactive' WHERE created_at < '2020-01-01';
→ Replica가 따라잡는 데 수 분 소요

-- 좋은 예: 1000건씩 나눠서 실행
UPDATE products SET status = 'inactive'
WHERE created_at < '2020-01-01' LIMIT 1000;
-- 1초 대기
UPDATE products SET status = 'inactive'
WHERE created_at < '2020-01-01' LIMIT 1000;
-- 반복...

→ Replica가 중간중간 따라잡을 시간 확보
```

복제는 "한 번 설정하면 끝"이 아닙니다. 트래픽 변화, 데이터 증가, 새로운 기능 추가에 따라 지속적으로 모니터링하고 조정해야 합니다.

## 6. 결론

이 문서가 MySQL 레플리케이션을 도입하거나 운영하는 데 도움이 되기를 바랍니다.

### 6-1. 핵심 요약

MySQL 레플리케이션은 단순히 서버를 추가하는 것이 아니라 분산 시스템을 도입하는 것입니다. 분산 시스템에는 PACELC 정리가 적용되며, 평상시(Else) 빠른 응답(Latency)과 데이터 일관성(Consistency) 사이에서 트레이드오프해야 합니다.

**복제 지연은 피할 수 없습니다**

비동기 복제는 빠르지만 복제 지연이 발생합니다. Primary에 쓴 데이터가 Replica에 도착하기까지 수 밀리초에서 수 초의 시간이 걸립니다. 이로 인해 사용자가 방금 쓴 데이터를 읽지 못하는 Read-after-Write 불일치나, 새로고침할 때마다 데이터가 달라 보이는 Monotonic Read 위반이 발생합니다.

**일관성 수준은 조절할 수 있습니다**

준동기 복제를 사용하고 ACK를 받을 Replica 개수를 조절하면 일관성 수준을 높일 수 있습니다. ACK 개수를 늘릴수록 일관성은 높아지지만 응답 시간이 느려집니다. 이것이 바로 일관성과의 트레이드오프입니다.

또한 기능별로 다른 전략을 사용할 수 있습니다. 주문 조회는 Primary에서 읽어 강한 일관성을 보장하고, 상품 목록은 Replica에서 읽어 성능을 확보하는 식입니다.

**비즈니스 요구사항을 먼저 정의하자**

일관성 수준은 기술적 결정이 아니라 비즈니스 결정입니다. "이 기능에서 1초의 데이터 지연은 허용되는가?", "사용자가 방금 쓴 데이터를 즉시 봐야 하는가?" 같은 질문에 답해야 합니다.

**모니터링이 핵심입니다**

복제 지연은 시스템 부하, 네트워크 상태, 대량 작업 등에 따라 실시간으로 변합니다. Seconds_Behind_Master를 지속적으로 모니터링하고, 임계값을 넘으면 즉시 대응해야 합니다. Prometheus와 Grafana 같은 도구로 시각화하고, 경고 시스템을 구축하세요.

### 6-2. 더 알아보기

MySQL Group Replication

이 문서에서 다룬 전통적인 Primary-Replica 레플리케이션으로 해결하기 어려운 요구사항이 있다면 MySQL Group Replication을 검토해보세요. 정족수 기반 합의 알고리즘으로 강한 일관성을 자동으로 보장하고, Multi-Primary도 지원합니다. 다만 설정과 운영이 복잡하고 네트워크 비용이 크다는 점을 고려해야 합니다.

참고 자료

MySQL 공식 문서: Replication
『Designing Data-Intensive Applications』 by Martin Kleppmann
『Database Internals』 by Alex Petrov
MySQL High Availability (O'Reilly)

이 문서는 실무 경험을 바탕으로 작성되었으며, MySQL 8.0 기준으로 설명합니다. 버전에 따라 일부 기능이나 동작이 다를 수 있습니다.
