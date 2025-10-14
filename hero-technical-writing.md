# 장애 대응이 가능한 모니터링 대시보드 구축하기

### **1. 서론**

대규모 트래픽 환경에서 장애는 필연적이다. 이러한 상황에서의 핵심은 **얼마나 빠르게 원인을 탐지하고 복구하느냐**이다.

본 보고서는 Spring Boot 애플리케이션에 **Spring Actuator** 와 **AWS CloudWatch** 를 연계하여 실시간 대시보드를 구축한 과정을 다룬다.

목표는 단순한 수집이 아닌 **장애 대응 수준의 관찰성과 자동 알림 체계**를 확보하는 것이다.

---

### **2. 모니터링의 필요성**

#### **2.1 서버 신뢰성 확보**

모니터링은 **서버 신뢰성의 핵심**이다. 시스템이 정상 동작하는 것처럼 보이더라도, 내부에서는 다음과 같은 병목이 발생할 수 있다.

- **GC(가비지 컬렉션, Garbage Collection) 일시 정지**: 메모리 회수 중 애플리케이션이 멈춰 요청 처리가 중단된다.

- **스레드 풀(Thread Pool) 포화**: 실행 가능한 스레드가 모두 바쁘면 새 요청이 대기열에 쌓인다.

- **DB 커넥션 풀 포화**: 연결 반환이 지연되면 전체 API 응답 속도가 느려진다.

이 지표들은 외부에서 보이지 않지만 사용자 경험을 크게 저하시킨다.

신뢰성은 정상 상태를 유지하는 능력이 아니라, **비정상 상태를 빠르게 감지하고 대응하는 능력**에서 결정된다.

---

#### **2.2 장애 대응 속도 향상**

대시보드는 장애 상황을 시각적으로 빠르게 파악하도록 돕는다. 로그 분석만으로는 원인 추적에 오랜 시간을 투자해야 하지만, 실시간 지표를 통해 아래와 같은 변화를 즉시 인지할 수 있다.

| **관찰 항목**                | **시각적 변화**    | **의미**             |
|--------------------------|---------------|--------------------|
| GC 정지 시간 0.8초 이상 지속      | 그래프 급상승       | 메모리 회수 지연          |
| Busy Thread 비율 95% 초과    | Thread 그래프 포화 | 요청 과부하 또는 DB 대기    |
| GC Time%와 API 응답시간 동시 상승 | 시점 일치         | CPU 경쟁 또는 객체 생성 과다 |

지표 간 상관관계를 즉시 확인하면 병목 지점을 빠르게 파악할 수 있다. 더 나아가, 이상 징후 감지 시 CloudWatch 경보를 발송하도록 설정해 **평균 복구 시간(MTTR)**을 단축할 수 있다.

---

#### **2.3 지표 중심의 문제 해결 문화**

모니터링은 추측이 아닌 **데이터 기반 판단**을 가능하게 한다. 정량적 지표를 사용해 개발자 간 해석의 차이를 줄이고, 문제의 근거를 명확히 제시할 수 있다.

| **추측에 의한 표현** | **지표 기반 표현**                      |
|---------------|-----------------------------------|
| “느려졌다.”       | “GC Pause Time이 평균 대비 300% 증가했다.” |
| “CPU가 높다.”    | “Runnable Thread 비율이 80% 이상이다.”   |
| “캐시가 느리다.”    | “Redis 적중률이 85%에서 60%로 하락했다.”     |

이런 방식은 팀 내 에서의 언어 및 용어 통일을 통해 **객관성, 속도 및 일관성**을 높인다.

---

### **3. 시스템 구성 개요**

| **구성 요소**            | **역할**           | **비고**                 |
|----------------------|------------------|------------------------|
| Spring Actuator      | 애플리케이션 내부 메트릭 수집 | Micrometer 기반 Exporter |
| AWS CloudWatch Agent | 메트릭 전달 및 대시보드 구성 | EC2 인스턴스 단위            |
| CloudWatch Dashboard | 지표 시각화 및 알림 트리거  | GC, Thread, API 모듈별 패널 |
| JMeter               | 부하 테스트 및 시나리오 검증 | TPS(초당 트랜잭션 수) 기반      |

---

### **4. 주요 대시보드 설계**

#### **4.1 JVM GC 모니터링 대시보드**

- **목적:** GC로 인한 응답 정지 시간 감시

- **지표:** Major/Minor GC 횟수, 평균 GC 정지 시간

- **활용:** GC 정지 시간이 0.5초 이상 지속되면 경고 알림 발송

**이 지표의 필요성**

GC는 자동 메모리 회수 과정에서 모든 스레드를 멈춘다. 정지 시간은 곧 서비스 응답 중단 시간이다.

따라서 GC 빈도와 정지 시간을 추적하면 **비정상적인 메모리 사용 패턴을 조기 탐지**할 수 있다. Minor GC 증가 시 객체 생성 과다를, Full GC 반복 시 JVM 옵션 관련 설정 오류를 재고해봐야
한다.

![jvm-gc-monitoring.png](images/jvm-gc-monitoring.png)

---

#### **4.2 GC - API 모니터링 대시보드**

- **목적:** 주요 엔드포인트의 성능 변동과 GC 동작의 상관관계 분석

- **구성:**

    - `/auth/kakao`, `/intake/history/cup` 등 핵심 API 호출 횟수

    - 요청당 평균 응답 시간

    - GC Time% 및 Heap 사용률

**이 지표의 필요성**

API는 사용자가 직접적으로 느끼는 응답 속도와 직결된다. 그러나 응답 지연은 비즈니스 로직보다 **GC나 Thread Pool**과 같은 내부 리소스 병목에서 시작되는 경우가 많다.

이를 구분하기 위해 CloudWatch에서 **API 호출량, 응답 시간, GC Time%를 오버레이** 형태로 표시한다.

예를 들어,

- `/auth/kakao` 호출이 급증하며 GC Time%가 함께 상승하면 해당 API 내부에서의 객체 생성량이 과도하다는 의미다.

- GC Time%는 일정한데 `/intake/history/cup` 응답만 지연된다면, GC 를 제외한 로직이나 DB 쿼리의 문제일 가능성이 높다.

이 분석을 통해 **특정 API로 인한 병목인지, 시스템 전체의 자원 문제인지**를 명확히 구분할 수 있다. 잘못된 추정 확률을 낮추고, 원인별로 최적화 우선순위를 정할 수 있다.

![gc-api-monitoring.png](images/gc-api-monitoring.png)

---

#### **4.3 스레드 모니터링 대시보드**

- **목적:** Tomcat Thread Pool의 활용률과 JVM Thread 상태 추적

- **지표:** Busy / Current / Max Thread 수, Thread 상태(Runnable, Waiting, Timed-Waiting)

**이 지표의 필요성**

스레드는 서버의 **동시 처리 용량**을 결정한다. Thread Pool이 포화되면 새 요청은 대기하거나, DB 대기와 GC 정지로 시스템이 정체된다.

- Busy Thread가 Max에 도달하면 처리 용량 한계
- Waiting Thread가 급증하면 외부 API나 DB 호출 지연
- Runnable Thread가 과도하면 CPU 경쟁 가능성

이 지표는 GC·API 지표와 함께 분석할 때 병목의 전체 경로를 파악할 수 있다.

![thread-monitoring.png](images/thread-monitoring.png)

---

### **5. 장애 시나리오 실험 및 가설 검증**

- **도구:** Apache JMeter

- **시나리오:** 10분간 1000명의 가상 사용자(Virtual User) 부하 유지

- **목표:** GC 발생률, API 응답 시간, Thread 상태 간 상관관계 분석

추후 실험을 통해 추가할 예정

---

### **6. 개선 및 확장 방안**

1. **GC 튜닝**


2. **자동 알림 체계 구축**

추후 실험을 통해 추가할 예정

---

### **7. 결론**

- 추후 실험을 통해 추가할 예정

---

### **참고문헌 및 출처**

- [Spring Boot Actuator Documentation](https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html)

- [AWS CloudWatch Metrics and Dashboards](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/working_with_metrics.html)

- [Apache JMeter User Manual](https://jmeter.apache.org/usermanual/index.html)

- [Java Performance Tuning (O’Reilly)](https://www.oreilly.com/library/view/java-performance/9781449363512/)