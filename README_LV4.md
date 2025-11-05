# MDC를 활용한 효율적인 로깅 관리

## 대상 독자
- 프로젝트에 로깅 기능을 처음 도입하려는 개발자
- 멀티 스레드 환경에서 요청의 로그 추적이 어려워 디버깅에 어려움을 겪는 개발자
- MDC를 사용하고 있으나 ThreadLocal 기반의 동작 원리와 주의사항을 명확히 이해하지 못한 개발자


## 들어가며: 로그 추적 문제

현대의 웹 애플리케이션은 수많은 요청을 동시에 처리합니다.  
하나의 서버에서도 수십, 수백 개의 스레드가 동시에 다른 요청을 처리하고 있습니다.  
이러한 환경에서 특정 요청의 처리 과정을 추적하거나 에러를 디버깅하는 것은 매우 어려운 일입니다.  

예를 들어 다음과 같은 로그를 살펴보겠습니다.

    2025-10-13 14:23:45 INFO  주문 처리 시작
    2025-10-13 14:23:45 INFO  결제 검증 시작
    2025-10-13 14:23:45 INFO  주문 처리 시작
    2025-10-13 14:23:45 ERROR 결제 실패: 잔액 부족
    2025-10-13 14:23:45 INFO  재고 확인 완료
    2025-10-13 14:23:46 INFO  주문 완료

위 로그만 보고는 어떤 요청에서 결제가 실패했는지, 어떤 주문이 정상적으로 완료되었는지 알 수 없습니다.  
여러 요청의 로그가 시간순으로 뒤섞여 있기 때문입니다.  
이런 상황에서 특정 주문의 처리 흐름을 파악하는 것은 거의 불가능에 가깝습니다.

이러한 문제를 해결하기 위해 각 요청에 고유한 ID를 부여할 수 있고,  
그 과정에서 활용할 수 있는 것이 바로 MDC(Mapped Diagnostic Context)입니다.

## MDC란 무엇인가?

MDC는 로깅 프레임워크(예: Logback, Log4j)에서 제공하는 기능으로, 스레드별로 컨텍스트 정보를 관리합니다.  
MDC를 활용하면 각 스레드마다 제공된 저장 공간에 로그에 포함할 정보를 저장할 수 있습니다.   
그리고 이 정보를 자동으로 로그에 포함시킬 수 있습니다.

가장 대표적인 활용 사례는 요청 추적 ID(Trace ID 또는 Request ID) 관리입니다. 

각 요청마다 고유한 ID를 부여하고 MDC에 저장하면,  
해당 요청을 처리하는 과정에서 발생하는 모든 로그에 이 ID가 자동으로 포함됩니다.

    2025-10-13 14:23:45 [a1b2c3] INFO  주문 처리 시작
    2025-10-13 14:23:45 [d4e5f6] INFO  결제 검증 시작
    2025-10-13 14:23:45 [g7h8i9] INFO  주문 처리 시작
    2025-10-13 14:23:45 [d4e5f6] ERROR 결제 실패: 잔액 부족
    2025-10-13 14:23:45 [a1b2c3] INFO  재고 확인 완료
    2025-10-13 14:23:46 [a1b2c3] INFO  주문 완료

이제 각 로그가 어떤 요청에 속하는지 명확하게 구분할 수 있습니다.  
로그 분석 도구에서 d4e5f6으로 검색하면 결제가 실패한 요청에서 발생한 로그를 모아 보고 그 흐름을 한눈에 파악할 수 있습니다.

## MDC를 사용하지 않을 때의 문제점

MDC 없이 요청 추적 ID를 로그에 포함시키려면 어떻게 해야 할까요?  
가장 직관적인 방법은 모든 로깅 코드에 요청 ID를 명시적으로 전달하는 것입니다.

    @RestController
    public class OrderController {
        
        @PostMapping("/orders")
        public OrderResponse createOrder(
                @RequestHeader("X-Request-Id") String requestId,
                @RequestBody OrderRequest request
        ) {
            
            log.info("[{}] 주문 처리 시작", requestId);
            
            // 서비스 레이어로 requestId 전달
            Order order = orderService.createOrder(requestId, request);
            
            log.info("[{}] 주문 처리 완료", requestId);
            return OrderResponse.from(order);
        }
    }
    
    @Service
    public class OrderService {
        
        public Order createOrder(String requestId, OrderRequest request) {
            log.info("[{}] 주문 검증 시작", requestId);
            validateOrder(requestId, request);
            
            log.info("[{}] 재고 확인", requestId);
            inventoryService.checkStock(requestId, request.getProductId());
            
            log.info("[{}] 결제 처리", requestId);
            paymentService.processPayment(requestId, request.getPaymentInfo());
            
            // 주문 생성 로직
            return order;
        }
    }
    
    @Service
    public class PaymentService {
        
        public void processPayment(String requestId, PaymentInfo info) {
            log.info("[{}] 결제 검증 시작", requestId);
            // 결제 처리 로직
            log.info("[{}] 결제 완료", requestId);
        }
    }

이러한 방식에는 여러 문제가 있습니다.

**1. 모든 메서드 시그니처에 requestId 추가**

컨트롤러부터 서비스, 리포지토리까지 모든 레이어의 메서드에 requestId 매개변수를 추가해야 합니다.  
실제로 비즈니스 로직에서 이 값을 사용하지 않더라도, 단지 로깅을 위해 계속 전달해야 합니다.

**2. 관심사 혼재**

비즈니스 로직을 담당하는 서비스 클래스가 HTTP 요청 정보에 의존하게 됩니다.  
서비스 레이어는 본래 HTTP와 무관하게 동작해야 하지만, 로깅을 위해 요청 ID를 알아야 하는 상황이 됩니다.

**3. 코드 중복 및 일관성 문제**

모든 로깅 코드마다 requestId를 명시적으로 전달해야 합니다.  
이는 실수로 누락하기 쉽기 때문에 일부 로그에서 요청 ID가 빠져 추적이 어려워질 수 있습니다.

이러한 문제들은 애플리케이션의 유지보수성을 저하시키고 디버깅을 어렵게 만들 수 있습니다.

## MDC를 활용한 개선

MDC를 사용하면 위에서 살펴본 문제들을 깔끔하게 해결할 수 있습니다.  
요청 처리 시작 시점에 한 번만 MDC에 정보를 저장하면, 이후 발생하는 모든 로그에 자동으로 해당 정보가 포함됩니다.

    @Component
    public class RequestLoggingFilter extends OncePerRequestFilter {
        
        @Override
        protected void doFilterInternal(
                HttpServletRequest request,
                HttpServletResponse response,
                FilterChain filterChain) throws ServletException, IOException {
            
            // 요청 ID 생성 또는 헤더에서 추출
            String requestId = request.getHeader("X-Request-Id");
            if (requestId == null) {
                requestId = UUID.randomUUID().toString();
            }
            
            // MDC에 요청 ID 저장
            MDC.put("requestId", requestId);
            MDC.put("userId", extractUserId(request));
            
            try {
                filterChain.doFilter(request, response);
            } finally {
                // 요청 처리 완료 후 MDC 정리
                MDC.clear();
            }
        }
    }

Logback 설정 파일에서 로그 패턴에 MDC 정보를 추가합니다.

    <configuration>
        ...
        <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
            <encoder>
                <pattern>%d{yyyy-MM-dd HH:mm:ss} [%X{requestId}] [%X{userId}] %-5level %logger{36} - %msg%n</pattern>
            </encoder>
        </appender>
        ...
    </configuration>

이제 컨트롤러와 서비스는 요청에 대해 아무것도 모른 채 로그를 남기면 됩니다.

    @RestController
    public class OrderController {

        @PostMapping("/orders")
        public OrderResponse createOrder(@RequestBody OrderRequest request) {
            log.info("주문 처리 시작"); // requestId와 userId가 자동으로 포함됨
            
            Order order = orderService.createOrder(request);
            
            log.info("주문 처리 완료"); // requestId와 userId가 자동으로 포함됨
            return OrderResponse.from(order);
        }
    }
    
    @Service
    public class OrderService {
        
        public Order createOrder(OrderRequest request) {
            log.info("주문 검증 시작"); // requestId와 userId가 자동으로 포함됨
            validateOrder(request);
            
            log.info("재고 확인"); // requestId와 userId가 자동으로 포함됨
            inventoryService.checkStock(request.getProductId());
            
            log.info("결제 처리"); // requestId와 userId가 자동으로 포함됨
            paymentService.processPayment(request.getPaymentInfo());
            
            return order;
        }
    }
    
    @Service
    public class PaymentService {
        
        public void processPayment(PaymentInfo info) {
            log.info("결제 검증 시작"); // requestId와 userId가 자동으로 포함됨
            // 결제 처리 로직
            log.info("결제 완료"); // requestId와 userId가 자동으로 포함됨
        }
    }

이로부터 발생한 로그는 다음과 같습니다.

    2025-10-13 14:23:45 [a1b2c3d4] [user-leo] INFO  OrderController - 주문 처리 시작
    2025-10-13 14:23:45 [d4e5f6a7] [user-mingo] INFO  OrderService - 주문 검증 시작
    2025-10-13 14:23:45 [g7h8i9j0] [user-posty] INFO  OrderController - 주문 처리 시작
    2025-10-13 14:23:45 [d4e5f6a7] [user-mingo] ERROR PaymentService - 결제 실패: 잔액 부족
    2025-10-13 14:23:45 [a1b2c3d4] [user-leo] INFO  OrderService - 재고 확인 완료
    2025-10-13 14:23:46 [a1b2c3d4] [user-leo] INFO  OrderController - 주문 처리 완료

이처럼 MDC를 활용하면 다음과 같은 이점을 얻을 수 있습니다.

- 관심사의 분리: 비즈니스 로직은 로깅 컨텍스트를 몰라도 됩니다.
- 일관성 보장: 모든 로그에 자동으로 컨텍스트가 포함되므로 누락 가능성이 없습니다.
- 디버깅 용이성: 요청 ID와 같은 컨텍스트 정보를 저장하면 해당 요청의 전체 처리 과정을 추적할 수 있습니다.
- 유지보수성 향상: 새로운 컨텍스트 정보 추가 시 MDC 코드만 수정하면 됩니다.

예제에서는 서블릿 필터를 활용했지만 스프링 MVC를 사용하는 경우 스프링 인터셉터를 활용할 수도 있습니다.  
하지만 인터셉터는 요청 처리 흐름 상 필터보다 뒤에 위치하므로,  
다음과 같은 상황에서는 인터셉터까지 요청이 도달하지 못해 로그에 컨텍스트가 포함되지 않는 문제가 있습니다.
- 필터에서 인증 실패로 요청이 거부되는 경우
- 요청 파라미터 검증 실패로 인터셉터 이전에 예외가 발생하는 경우
- 핸들러 매핑 실패로 404 에러가 발생하는 경우

따라서 이러한 상황의 로그에도 컨텍스트를 포함하고 싶다면, 요청 처리 흐름의 가장 앞단인 서블릿 필터에서 MDC를 설정하는 것을 권장합니다.

## MDC의 동작 원리

MDC가 어떻게 스레드별로 정보를 관리하는지 내부 동작 원리를 이해하면, MDC를 더욱 효과적으로 사용할 수 있습니다.

### ThreadLocal 기반 저장소

MDC.put(key, value)를 호출하면 내부적으로 MDCAdapter 구현체의 ThreadLocal<Map<String, String>>에 데이터를 보관합니다.

ThreadLocal<T>는 자바에서 제공하는 클래스로, T 타입의 변수를 스레드별로 독립적으로 저장하고 관리합니다.  
각 스레드는 자신만의 ThreadLocal 복사본을 가지므로, 다른 스레드의 데이터에 영향을 주거나 받지 않습니다.

    // Thread-1에서
    MDC.put("requestId", "req-001");
    MDC.get("requestId"); // "req-001" 반환
    
    // Thread-2에서 (동시에)
    MDC.put("requestId", "req-002");
    MDC.get("requestId"); // "req-002" 반환
    
    // Thread-1에서
    MDC.get("requestId"); // "req-001" 반환 (Thread-2의 영향을 받지 않음)

이러한 ThreadLocal 기반 구조 덕분에 MDC는 멀티스레드 환경에서 스레드 안정성(thread-safety)을 보장합니다.

### 로깅 시 MDC 정보 포함 과정

로그를 출력할 때는 다음과 같은 과정을 거칩니다.

    1. log.info 호출
    2. LoggingEvent 생성 후 현재 스레드의 MDC 정보를 복사하여 포함
    3. Encoder, Layout이 로그 조립
    4. MDC 패턴을 만나면 MDCConverter가 MDC 정보를 읽어 로그에 추가
    5. Appender가 완성된 로그 출력

로그 패턴에서 MDC 값을 참조하는 방법은 다음과 같습니다.

    <!-- 특정 키의 값 출력 -->
    %X{requestId}
    %mdc{requestId}
    
    <!-- 모든 MDC 내용 출력 -->
    %X
    %mdc

## MDC 사용 시 주의사항
### 1. 스레드 풀 환경에서의 메모리 누수

대부분의 웹 애플리케이션은 성능 향상을 위해 스레드 풀을 사용합니다.  
요청이 들어오면 풀에서 가용 스레드를 가져와 요청을 처리합니다.  
요청 처리가 완료되면 스레드는 다시 풀로 반환되어 다음 요청을 기다립니다.

**문제 상황**

ThreadLocal은 스레드가 종료될 때까지 데이터를 보관합니다.  
스레드 풀 환경에서는 스레드가 종료되지 않고 재사용되기 때문에, 이전 요청의 MDC 정보가 ThreadLocal에 그대로 남게 됩니다.

    // 첫 번째 요청 (Thread-1 사용)
    MDC.put("requestId", "req-001");
    MDC.put("userId", "user-leo");
    log.info("첫 번째 요청 처리");
    // MDC.clear()를 깜빡함!
    
    // Thread-1이 풀로 반환됨
    
    // 두 번째 요청 (같은 Thread-1 재사용)
    log.info("두 번째 요청 처리");
    // [req-001] [user-leo] 두 번째 요청 처리 (실제로는 다른 사용자의 요청인데 user-leo로 로깅됨)

이는 다음과 같은 심각한 문제를 일으킬 수 있습니다.

- 잘못된 로그 추적: 한 사용자의 요청이 다른 사용자의 것으로 로깅됩니다.
- 보안 문제: 민감한 정보가 다른 사용자의 로그에 노출될 수 있습니다.

**해결 방법 1: 서블릿 필터에서 정리**

가장 간단하고 확실한 방법은 서블릿 필터의 finally 블록에서 MDC를 정리하는 것입니다.

    @Component
    public class MdcLoggingFilter extends OncePerRequestFilter {
        
        @Override
        protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
                                        FilterChain filterChain) throws ServletException, IOException {
            try {
                // MDC 설정
                String requestId = UUID.randomUUID().toString();
                MDC.put("requestId", requestId);
                MDC.put("userId", extractUserId(request));
                MDC.put("uri", request.getRequestURI());
                MDC.put("method", request.getMethod());
                
                log.info("요청 시작");
                
                filterChain.doFilter(request, response);
                
                log.info("요청 완료");
            } finally {
                // 반드시 정리해야 함!
                MDC.clear();
            }
        }
    }

finally 블록은 예외가 발생하더라도 항상 실행되므로, 어떤 상황에서도 MDC가 정리됨을 보장합니다.

스프링 인터셉터를 활용하는 경우 afterCompletion 메서드에서 MDC를 명시적으로 비워줍니다.

**해결 방법 2: ThreadPoolExecutor 커스터마이징**

별도의 스레드 풀을 사용하는 경우, ThreadPoolExecutor를 상속받아 작업 완료 후 자동으로 MDC를 정리할 수 있습니다.

    public class MdcThreadPoolExecutor extends ThreadPoolExecutor {
        
        public MdcThreadPoolExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime, 
                                     TimeUnit unit, BlockingQueue<Runnable> workQueue) {
            super(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue);
        }
        
        @Override
        protected void afterExecute(Runnable r, Throwable t) {
            MDC.clear(); // 작업 완료 후 MDC 정리
        }
    }
    
    @Configuration
    public class ExecutorConfig {

        @Bean
        public Executor taskExecutor() {
            return new MdcThreadPoolExecutor(10, 20, 60L, TimeUnit.SECONDS, new LinkedBlockingQueue<>(100));
        }
    }

### 2. 비동기 작업과 MDC 전파

최근의 애플리케이션은 성능 향상을 위해 비동기 처리를 적극 활용합니다.  
하지만 MDC는 ThreadLocal 기반이기 때문에, 비동기 작업을 처리하는 다른 스레드에서는 메인 스레드의 MDC 정보에 접근할 수 없습니다.

**문제 상황**

    @Service
    public class OrderService {
        
        @Async
        public CompletableFuture<Order> processOrderAsync(OrderRequest request) {
            // 메인 스레드에서 MDC 설정
            MDC.put("requestId", "req-001");
            MDC.put("userId", "user-leo");
            
            log.info("주문 처리 시작"); // [req-001] [user-leo] 주문 처리 시작
            
            return CompletableFuture.supplyAsync(() -> {
                // 다른 스레드에서 실행됨
                log.info("비동기 작업 시작"); // requestId와 userId가 없음!
                
                // 재고 확인
                checkInventory(request.getProductId());
                
                // 결제 처리
                processPayment(request);
                
                log.info("비동기 작업 완료"); // requestId와 userId가 없음!
                
                return createOrder(request);
            });
        }
        
        private void checkInventory(String productId) {
            log.info("재고 확인: {}", productId); // requestId와 userId가 없음!
        }
        
        private void processPayment(OrderRequest request) {
            log.info("결제 처리"); // requestId와 userId가 없음!
        }
    }

실행 결과:

    2025-10-13 14:23:45 [req-001] [user-leo] INFO  주문 처리 시작
    2025-10-13 14:23:45 [] [] INFO  비동기 작업 시작
    2025-10-13 14:23:45 [] [] INFO  재고 확인: PROD-123
    2025-10-13 14:23:45 [] [] INFO  결제 처리
    2025-10-13 14:23:46 [] [] INFO  비동기 작업 완료

비동기 작업의 로그에는 requestId와 userId가 전혀 포함되지 않아, 어떤 요청의 처리 과정인지 알 수 없습니다.  
이는 분산 환경에서 로그 추적을 거의 불가능하게 만듭니다.

**해결 방법 1: MDC 컨텍스트 수동 전파**

가장 직접적인 방법은 비동기 작업 실행 전에 메인 스레드의 MDC 정보를 복사하여 작업 스레드로 전달하는 것입니다.

    @Service
    public class OrderService {
        
        @Async
        public CompletableFuture<Order> processOrderAsync(OrderRequest request) {
            // 현재 스레드의 MDC 컨텍스트 복사
            Map<String, String> mdcContext = MDC.getCopyOfContextMap();
            
            log.info("주문 처리 시작");
            
            return CompletableFuture.supplyAsync(() -> {
                // 작업 스레드에 MDC 컨텍스트 복원
                if (mdcContext != null) {
                    MDC.setContextMap(mdcContext);
                }
                
                try {
                    log.info("비동기 작업 시작"); // requestId와 userId 포함됨!
                    
                    checkInventory(request.getProductId());
                    processPayment(request);
                    
                    log.info("비동기 작업 완료");
                    
                    return createOrder(request);
                } finally {
                    // 작업 완료 후 MDC 정리
                    MDC.clear();
                }
            });
        }
    }

이제 비동기 작업의 로그에도 올바른 컨텍스트가 포함됩니다.

    2025-10-13 14:23:45 [req-001] [user-alice] INFO  주문 처리 시작
    2025-10-13 14:23:45 [req-001] [user-alice] INFO  비동기 작업 시작
    2025-10-13 14:23:45 [req-001] [user-alice] INFO  재고 확인: PROD-123
    2025-10-13 14:23:45 [req-001] [user-alice] INFO  결제 처리
    2025-10-13 14:23:46 [req-001] [user-alice] INFO  비동기 작업 완료

**해결 방법 2: TaskDecorator를 활용한 자동 전파**

모든 비동기 작업마다 MDC를 수동으로 복사하는 것은 번거롭고 실수하기 쉽습니다.  
Spring의 TaskDecorator를 사용하면 모든 비동기 작업에 자동으로 MDC를 전파할 수 있습니다.

    public class MdcTaskDecorator implements TaskDecorator {
        
        @Override
        public Runnable decorate(Runnable runnable) {
            // 현재 스레드(메인 스레드)의 MDC 컨텍스트 복사
            Map<String, String> mdcContext = MDC.getCopyOfContextMap();
            
            return () -> {
                try {
                    // 작업 스레드에 MDC 컨텍스트 복원
                    if (mdcContext != null) {
                        MDC.setContextMap(mdcContext);
                    }
                    
                    // 실제 작업 실행
                    runnable.run();
                } finally {
                    // 작업 완료 후 MDC 정리
                    MDC.clear();
                }
            };
        }
    }
    
    @Configuration
    @EnableAsync
    public class AsyncConfig implements AsyncConfigurer {
        
        @Override
        public Executor getAsyncExecutor() {
            ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
            executor.setCorePoolSize(10);
            executor.setMaxPoolSize(20);
            executor.setQueueCapacity(100);
            executor.setThreadNamePrefix("async-");
            
            // MDC 전파를 위한 TaskDecorator 설정
            executor.setTaskDecorator(new MdcTaskDecorator());
            
            executor.initialize();
            return executor;
        }
    }

이제 @Async 애너테이션을 사용하는 모든 비동기 메서드에서 자동으로 MDC가 전파됩니다.

## 정리

MDC는 효과적인 로깅과 디버깅을 위한 필수 도구입니다.

**MDC의 핵심 특징**

- MDC는 로그 정보를 스레드별로 관리하고, 설정된 정보를 로그에 자동으로 포함합니다.
- ThreadLocal을 기반으로 동작하여 스레드 안정성을 보장합니다.
- 로깅 코드와 컨텍스트 정보 관리를 분리하여 코드의 응집도를 높이고 유지보수성을 향상시킵니다.

**반드시 지켜야 할 사항**

- 작업이 끝나면 MDC.clear()를 호출하여 MDC 정보를 필수로 정리합니다.
- 비동기 작업을 사용할 때는 MDC 정보를 명시적으로 전파해야 합니다.

이러한 주의사항을 지키고 MDC를 올바르게 활용한다면, 효과적으로 로그를 추적하고 문제를 신속하게 해결할 수 있습니다.  
MDC는 단순한 로깅 도구가 아니라, 애플리케이션의 상태를 파악하고 문제를 진단하는 데 필수적인 관찰성(Observability) 도구입니다.
