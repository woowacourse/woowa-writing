# 1. 들어가며

소프트웨어는 실패를 전제로 움직인다. 입력은 사양에서 벗어나고, 파일은 없거나 잠겨 있으며, 네트워크는 중간에 끊기고, 데이터베이스는 때때로 응답하지 않는다. 예외(exception)는 이런 실패를 제어 흐름 안으로 끌어들여 다룰 수 있게 하는 언어적 장치다. 즉, 예외는 단순한 오류가 아니다. 예외는 정상 흐름에서 벗어난 제어 흐름이며, 프로그램을 멈추기 위한 비상정지 장치가 아니라 실패를 다룰 기회를 제공하는 구조다.

자바는 이 기회를 언어 차원에서 확장했다. 자바의 철학은 명확했다. “실패를 인지하고, 반드시 처리하라.” 이 강제된 주의의 결과물이 바로 Checked Exception이다. 자바가 제시한 질문은 단순하다. 이 실패를 알고 있는가? 알고 있다면 어떻게 처리할 것인가? 이 질문을 컴파일 타임까지 끌어올린 것이 자바의 안전성 철학이었다.

이 설계는 작은 프로그램에서는 분명 효과가 있었다. 메서드 시그니처에 드러난 throws 목록은 호출자에게 “여기서 이런 위험이 생긴다”를 명확히 보여줬고, IDE는 경로를 따라가며 어떤 예외가 발생할 수 있는지 알려주었다. 그러나 프로그램의 규모가 커질수록 문제는 발생했다. 하위 계층의 예외가 상위 계층까지 연쇄적으로 전파되면서 예외 선언은 호출 체인을 따라 증식했고, 의미를 잃은 예외 정보가 코드에 쌓여 갔다. 예외가 발생했을 때 무엇을 해야 하는지 명확하지 않은 채, try-catch 블록만 늘어나는 현상이 반복되었다.

결국 “실패를 드러내라”는 철학은 점차 “일단 잡고 넘겨라”는 관행으로 흐르게 되었다. 이 글은 바로 그 지점에서 출발한다. 자바가 왜 Checked Exception을 만들었고, 시간이 지나며 어떤 한계가 드러났는지, 그리고 스프링은 그 철학을 어떻게 다른 방식으로 해석했는지 살펴본다.

# 2. 자바의 예외 체계

자바는 예외를 Error와 Exception 두 축으로 나누고, 그중 Exception을 다시 Checked와 Unchecked로 구분한다. 이 구분은 단순히 문법상의 분류가 아니라, 복구 가능성(recoverability) 이라는 기준에 따라 설계된 구조다.

| 구분 | 상위 클래스 | 처리 강제 여부 | 대표 예시 | 의도 |
| --- | --- | --- | --- | --- |
| Error | java.lang.Error | 불가능 | OutOfMemoryError, StackOverflowError | JVM 수준 오류 |
| Checked Exception | Exception (단, RuntimeException 제외) | 필수 | IOException, SQLException | 복구 가능한 예외 |
| Unchecked Exception | RuntimeException | 선택 | NullPointerException, IllegalArgumentException | 프로그래밍 오류 |

이 구조의 의도는 명확했다. Error는 애플리케이션이 다룰 수 없는 영역이다. 메모리가 부족하거나 스택이 넘치는 상황은 프로그램이 제어할 수 없다. 따라서 그냥 종료하는 것이 맞다. 반면 Exception은 애플리케이션 내부나 외부 자원에서 발생하는 예상 가능한 실패를 표현한다. 그중에서도 Checked Exception은 반드시 처리(catch)하거나 던져야(throws) 한다.

예를 들어 다음 코드는 컴파일조차 되지 않는다.

```java
public void readFile(String path) {
    FileReader reader = new FileReader(path); // 컴파일 에러!
}
```

FileReader는 FileNotFoundException을 던질 수 있기 때문이다. 자바는 이 예외를 “복구 가능한 실패”로 보고, 개발자에게 “반드시 다루라”고 요구한다. 즉, 언어 차원에서 안전성을 강제한 실험이었다.

## 2.1 안전성을 위한 설계 의도

C++ 시대의 오류 처리는 대부분 리턴값에 의존했다. 함수가 실패하면 특수한 리턴값(예: -1 또는 null)을 돌려줬고, 호출자가 그것을 일일이 확인해야 했다. 하지만 확인을 빼먹는 일은 너무 흔했다. 오류는 조용히 지나가고, 문제는 나중에야 드러났다.

자바는 이 실수를 반복하지 않으려 했다. 그래서 예외를 타입 시스템에 포함시켰다. 예외가 함수 시그니처의 일부가 되어, 호출자는 반드시 그 존재를 인식해야 했다.

```java
public void copyFile(String src, String dest) throws IOException {
    // ...
}
```

이제 IDE는 호출자에게 경고한다. “이 메서드는 IOException을 던질 수 있습니다. 처리하지 않으면 컴파일되지 않습니다.” 이 덕분에 작은 프로그램에서는 오류를 놓치기 어려웠다. 자바는 언어 차원에서 ‘안전한 프로그래밍’을 강제하는 언어로 자리 잡았다.

## 2.2 Checked Exception 확산 문제

하지만 프로그램이 커질수록 문제는 명확해졌다. 하위 계층에서 Checked Exception을 던지면, 상위 계층은 선택의 여지 없이 throws를 선언해야 했다. 그 예외를 잡아 처리하지 않으면, 호출자도 똑같이 던져야 한다. 결국 예외 선언은 호출 체인을 따라 증식했다.

```java
public void process() throws IOException {
    readFile(); // 내부에서 IOException 발생
}
```

한 계층의 작은 변화가 상위 계층 전체의 시그니처 수정으로 번지기 시작했다. Checked Exception이 호출 체인을 따라 강제로 전파되면서 계층 간 결합도는 높아지고 변경 비용은 기하급수적으로 커졌다.

그 결과 많은 개발자는 Checked Exception을 의미 없는 형식적 의무로 다루게 되었다. 실제로는 예외를 처리하지 않고, 그저 다시 던지거나 포장하는 코드가 늘어나기 시작했다.

```java
try {
    repository.save(user);
} catch (SQLException e) {
    throw new RuntimeException(e);
}
```

이 코드는 컴파일러의 요구를 충족할 뿐, 실패를 의미 있게 다루지는 않는다. 결국 Checked Exception은 “안전성 보장”이 아닌 “형식적 의무”로 전락했다.

# 3. Checked Exception의 의도와 한계

Checked Exception은 단순한 문법 요소가 아니라, 언어가 개발자에게 던진 질문이었다. “이 실패를 알고 있는가?” “그렇다면 어떻게 처리할 것인가?” 자바는 이 질문을 코드 수준에서 강제함으로써, 실패를 숨기지 않고 드러내는 방식을 택했다.

## 3.1 지키려 했던 가치

자바가 Checked Exception을 설계할 때 중심에 둔 원칙은 다음 세 가지다.

1. 예상 가능한 실패를 명시적으로 드러내라. - 예외를 메서드 시그니처에 선언해 호출자가 대비할 수 있게 한다.  
2. 복구 가능한 실패는 직접 처리하라. - 강제된 try-catch로 시스템 안정성을 높인다.  
3. 비정상 상황을 제어 흐름 안에서 관리하라. - 오류를 숨기지 않고 프로그램 구조 안에서 다룬다.  

자바는 “실패를 명시적으로 다루는 언어”를 목표로 삼았고, Checked Exception은 그 철학의 구현이었다. 이는 “예측 가능한 실패는 코드로 처리하라”는 선언에 가깝다. 

이 철학을 명확히 정리한 사람이 조슈아 블로크다. 그는 *이펙티브 자바*에서 다음과 같이 설명한다.

> “복구 가능한 상황에는 Checked Exception을, 프로그래밍 오류에는 RuntimeException을 사용하라.”

즉, Checked Exception은 복구 가능한 실패를 표현하기 위한 장치로 설계되었다.

## 3.2 복구 가능성의 모호함

하지만 문제는 바로 그 핵심 개념, ‘복구 가능성(recoverability)’에 있었다. 예를 들어 SQLException은 Checked Exception이지만, 데이터베이스 연결이 끊겼을 때 애플리케이션이 이를 스스로 복구할 수 있을까? DB 서버가 다운된 상황에서 애플리케이션이 직접 연결을 복원하는 것은 현실적으로 어렵다. 그럼에도 자바는 이런 예외를 Checked Exception으로 분류했다. “복구할 수도 있으니 직접 처리하라”는 강제였다. 결국 대부분의 개발자는 다음과 같은 코드를 작성하게 된다.

결국 많은 코드는 다음과 같은 형태로 귀결되었다.

```java
try {
    repository.save(user);
} catch (SQLException e) {
    throw new RuntimeException(e);
}
```

복구 로직은 없고, 컴파일러의 요구를 만족시키기 위한 재포장만 남는다. 이 시점에서 Checked Exception은 강제된 형식이 되고, 본래의 의미는 흐려진다.

## 3.3 강제의 역설

Checked Exception은 초기에 “실패를 다루는 습관”을 만들어줬지만, 시간이 지나면서 생산성을 떨어뜨리는 형식적 의무로 변했다. 하위 계층에서 발생한 예외가 상위 계층 전체로 확산되며, 모든 메서드가 예외를 선언하거나 포장해야 하는 상황이 반복되었다. 작은 변경에도 코드가 크게 흔들렸고, 예외는 불필요한 보일러플레이트로 변했다.

이 상황을 로버트 C. 마틴은 다음과 같이 표현했다.

> “논쟁은 끝났다. Checked Exception은 실수였다. 복구할 수 없는 예외를 강제하는 순간, 시스템은 유연성을 잃는다.”

이 말은 단순한 비판이 아니라, 복구 불가능한 실패를 강제하는 설계 자체가 불필요하다는 지적이었다.

# 4. 스프링이 마주한 현실

Checked Exception의 한계는 언어 차원을 넘어 프레임워크로 이어졌다. 스프링이 등장할 당시, 자바 개발자들은 이미 Checked Exception의 불편함을 체감하고 있었다. 복구할 수 없는 예외들(데이터베이스 연결 끊김, 트랜잭션 오류, 외부 API 타임아웃)이 코드 전반을 뒤덮고 있었다. 이 상황에서 스프링은 단순히 불편함을 줄이는 것을 넘어, 예외의 철학 자체를 다시 정의해야 했다.

## 4.1 잡을 수 없는 예외

현실에서 Checked Exception은 대부분 “잡아서 다시 던지는 코드”로 귀결됐다. 복구보다는 형식적인 처리였다.

```java
try {
    connection.prepareStatement("INSERT INTO users ...");
} catch (SQLException e) {
    logger.error("Database error", e);
    throw new RuntimeException(e);
}
```

이 코드는 문법적으로는 올바르지만, 실제로는 아무런 복구를 수행하지 않는다. 데이터베이스 연결이 회복되지도, 트랜잭션이 복원되지도 않는다. 로그를 남기고 예외를 다시 던질 뿐이다. 결국 Checked Exception이 의도했던 “복구 가능한 실패의 명시”는 현실에서 작동하지 않았다. 개발자들은 예외를 잡아 재포장하는 반복적인 코드에 익숙해졌고, 예외의 의미는 점차 사라졌다.

## 4.2 스프링의 문제의식

스프링은 이러한 상황을 언어의 한계가 아닌 설계의 문제로 인식했다. Checked Exception의 불편함은 단순한 문법의 문제가 아니라, 복구 가능성에 대한 판단을 언어가 강제하고 있다는 점에서 비롯되었다. 스프링은 이 문제를 해결하기 위해 “Checked Exception을 없애는 것”이 아니라 “예외를 의미 있게 재구성하는 것”을 목표로 삼았다. 

스프링은 다음과 같은 질문에서 출발했다.

> “복구할 수 없는 예외라면, 왜 개발자에게 catch를 강제하는가?”
>

이 질문에 대한 답은 명확했다. 스프링은 복구할 수 없는 예외를 모두 런타임으로 전환하고, 복구 가능한 예외만 선택적으로 다루는 구조를 택했다.

# 5. 복구 가능성 기준의 재정립

스프링은 자바의 Checked Exception 철학을 부정하지 않았다. 오히려 그것을 현실에 맞게 재해석했다. Rod Johnson이 지적했듯, 핵심은 “Checked Exception을 없앨 것인가”가 아니라 “그 예외가 실제로 복구 가능한가를 어떻게 표현할 것인가”에 있었다. 자바는 이 질문의 답을 컴파일러에게 맡겼다. 복구 가능성의 판단을 언어가 강제한 것이다. 반면 스프링은 그 책임을 프레임워크의 구조로 옮겼다.

## 5.1 SQLException의 모순

자바에서 가장 대표적인 Checked Exception은 SQLException이다. JDBC API는 이를 “복구 가능한 실패”로 간주해 모든 데이터 접근 코드에 throws SQLException을 강제했다. 그러나 현실에서 대부분의 SQL 예외는 복구 불가능했다. 네트워크 단절, 커넥션 타임아웃, 트랜잭션 충돌 같은 오류는 애플리케이션이 스스로 해결할 수 없는 문제였다. 그럼에도 컴파일러는 “catch하거나 던져라”를 요구했고, 개발자는 다음과 같은 코드를 작성할 수밖에 없었다.

```java
try {
    repository.save(user);
} catch (SQLException e) {
    throw new RuntimeException(e);
}
```
복구는 없고 형식만 남았다. 스프링은 이러한 문제를 해결하기 위해 복구 불가능한 예외를 런타임 예외로 전환하고, 개발자가 처리해야 하는 경우만 명시적으로 다루도록 했다. 이 방식을 구현한 것이 DataAccessException 계층이다.

## 5.2 복구 가능성 계층화

스프링은 예외를 Checked와 Unchecked로 구분하지 않는다. 대신 복구 가능성(recoverability) 을 기준으로 예외를 분류한다. 이 구조는 org.springframework.dao 패키지에 정의되어 있다. 루트 클래스는 DataAccessException이며, 하위 클래스는 복구 가능성의 정도에 따라 세 가지로 구분된다.

| 구분 | 루트 클래스 | 복구 조건 | 의미 |
| --- | --- | --- | --- |
| TransientDataAccessException | 즉시 재시도하면 성공 가능 | 애플리케이션 개입 없이 재시도만으로 복구 | 네트워크 지연, 락 경합 |
| RecoverableDataAccessException | 복구 절차 수행 후 재시도 가능 | 커넥션 재획득, 트랜잭션 재시작 등 수동 복구 필요 | 연결 재설정, 세션 재생성 |
| NonTransientDataAccessException | 원인 수정 없이는 실패 | 데이터나 코드 교정 없이는 재시도로 불가 | 무결성 위반, 중복 키, 제약 조건 오류 |

이 계층은 Checked Exception의 분류 기준을 언어 수준이 아닌 프레임워크 수준으로 옮긴 것이다. 자바가 컴파일러를 통해 복구 가능성을 강제했다면, 스프링은 예외 계층 구조로 이를 표현한다. 기준은 “잡을 수 있는가”가 아니라 “복구할 수 있는가”이다.

스프링은 Checked Exception의 문제를 단순히 제거하지 않고, 복구 가능성에 따라 구조적으로 구분했다. 언어가 하던 강제를 프레임워크가 대신 수행하도록 바꾼 것이다. 결과적으로 예외는 형식이 아닌 의미 중심으로 다루어졌고, 불필요한 코드 의존성을 줄이면서도 명확한 책임 구분이 가능해졌다.

# 6. 예외 전환(Exception Translation)

스프링은 Checked Exception의 강제적 처리를 제거했지만, 예외를 단순히 던지고 끝내지 않았다. 예외를 공통된 형태로 변환(translate)해, 복구가 불가능하더라도 실패의 의미를 유지하도록 했다. 스프링에서 말하는 예외 전환(Exception Translation)은 이 과정을 의미한다.

## 6.1 기술 종속성 제거

스프링 공식 문서는 다음과 같이 설명한다.

> “Spring provides a convenient translation from technology specific exceptions like SQLException to its own exception hierarchy with the DataAccessException as the root exception. These exceptions wrap the original exception so there is never any risk that you would lose any information as to what might have gone wrong.” 
> 
> — Spring Framework Reference, §10.2 Consistent Exception Hierarchy

스프링은 데이터 접근 기술마다 달랐던 예외(SQLException, HibernateException, PersistenceException, JDOException)를 모두 DataAccessException 계층으로 변환한다. 이 과정에서 예외는 단순히 감싸지는 것이 아니라, 실패의 의미를 일관된 형태로 표현된다.

| 원래 예외 | 변환된 예외 | 의미 |
|------------|--------------|------|
| SQLException | DuplicateKeyException | 중복 키 위반 |
| SQLException | DataIntegrityViolationException | 무결성 제약 위반 |
| HibernateException | HibernateJdbcException | Hibernate 내부 JDBC 오류 |
| PersistenceException | JpaObjectRetrievalFailureException | 엔티티 조회 실패 |

이 계층은 특정 기술에 종속되지 않으며, 애플리케이션이 이해할 수 있는 공통된 예외 모델을 제공한다.

## 6.2 예외 해석과 번역

스프링의 예외 전환은 Translator나 AOP 프록시를 통해 수행된다. 기술 스택별로 전용 Translator가 다르다.

| 기술 스택 | Translator |
|------------|-------------|
| JDBC | SQLErrorCodeSQLExceptionTranslator |
| Hibernate | HibernateExceptionTranslator, HibernateJpaDialect |
| JPA | PersistenceExceptionTranslationPostProcessor |
| JDO | JdoExceptionTranslator |

이 구조 덕분에 개발자는 기술별 예외 세부 사항을 몰라도 동일한 방식으로 예외를 처리할 수 있다.

## 6.3 예외 언어의 일원화

스프링의 예외 전환은 단순한 편의 기능이 아니라 예외 처리 체계의 일원화를 위한 메커니즘이다. 자바의 Checked Exception이 “실패를 반드시 처리하라”고 강제했다면, 스프링의 Exception Translation은 “실패를 일관된 형태로 이해하라”고 제시한다.

> “This allows you to handle most persistence exceptions, which are non-recoverable, only in the appropriate layers, without annoying boilerplate catches/throws.” 
> 
> — Spring Framework Reference, §10.2

스프링은 예외를 직접 처리하지 않고, 의미가 유지된 형태로 전달한다. 개발자는 기술별 예외 대신 공통된 구조를 기반으로 실패를 해석할 수 있다. 이를 통해 예외 처리가 단순화되고, 계층 간 의존성이 줄어든다.

# 7. 웹 환경으로의 확장

스프링의 예외 설계는 데이터 접근 계층에 한정되지 않는다. 컨트롤러와 HTTP 응답으로 이어지는 웹 환경에서도 동일한 원칙이 적용된다. 웹의 실패 역시 프로그램의 실패이며, 스프링은 데이터베이스의 SQLException을 DataAccessException으로 변환하듯 컨트롤러의 예외도 HTTP 응답으로 전환한다. 이 과정을 스프링은 웹 환경의 예외 전환(Exception Translation)으로 확장했다.

## 7.1 웹 응답으로의 전환

스프링 MVC에서 컨트롤러는 메서드의 결과를 HTTP 응답으로 변환한다. 예외가 발생하면 정상적인 반환 흐름이 끊기므로, 프레임워크는 예외를 HTTP 응답 형태로 변환해야 한다. 즉, 예외 전환은 데이터 계층의 예외 번역을 HTTP 응답 변환(Response Translation)으로 확장한 구조다.

| 계층 | 예외 전환 대상 | 변환 결과 |
| --- | --- | --- |
| 데이터 계층 | SQLException → DataAccessException | 런타임 예외 |
| 웹 계층 | Exception → HTTP Response | 상태 코드 + 메시지 |

데이터 계층에서 기술별 예외를 추상화하듯, 웹 계층에서는 예외를 표준화된 응답 구조로 추상화한다. 핵심은 일관된 의미 표현이다.

## 7.2 전환의 연속성

스프링의 예외 전환은 계층 간 일관된 흐름으로 동작한다. 데이터베이스의 SQLException은 DataAccessException으로 변환되고, 서비스 계층을 거쳐 컨트롤러에 도달하면 최종적으로 HTTP 응답으로 전환된다.

```java
try {
    repository.save(user);
} catch (DataIntegrityViolationException e) {
    throw new DuplicateUserException();
}
```

이 예외는 웹 계층으로 전파되며, 

```java
@ResponseStatus(HttpStatus.CONFLICT)
public class DuplicateUserException extends RuntimeException {
    public DuplicateUserException() {
        super("이미 등록된 사용자입니다.");
    }
}
```

다음와 같은 HTTP 응답으로 변환된다.

```
HTTP/1.1 409 Conflict
Content-Type: application/json
{
  "error": "이미 등록된 사용자입니다."
}
```

이 흐름은 예외가 각 계층의 책임에 따라 적절한 의미로 번역되는 과정을 보여준다. 데이터 계층, 서비스 계층, 웹 계층은 각각의 맥락에서 예외를 자신이 이해할 수 있는 형태로 변환한다.

## 7.3 스프링 MVC의 예외 전환

스프링 MVC는 DispatcherServlet을 중심으로 예외 전환을 수행한다. 컨트롤러에서 예외가 발생하면 다음 순서로 처리된다.

1. DispatcherServlet이 예외를 감지한다.
2. 등록된 HandlerExceptionResolver 목록을 순서대로 탐색한다.
3. 해당 예외를 처리할 수 있는 Resolver가 적절한 응답 객체를 생성한다.
4. 이 응답이 ResponseEntity 혹은 JSON 형태로 직렬화되어 클라이언트에게 전달된다.

스프링은 기본 Resolver를 제공한다.

| 클래스 | 역할 |
| --- | --- |
| ExceptionHandlerExceptionResolver | @ExceptionHandler 기반 처리 |
| ResponseStatusExceptionResolver | 예외 클래스의 @ResponseStatus 처리 |
| DefaultHandlerExceptionResolver | 스프링 내부 표준 예외 처리 (HttpRequestMethodNotSupportedException 등) |

이 체계를 통해 개발자는 예외를 직접 응답으로 변환할 필요 없이, 예외의 의미만 정의하면 된다.

## 7.4 HTTP로 드러나는 의미

스프링은 자바의 throws 선언이 컴파일러에 예외 존재를 알렸던 것처럼, 웹 계층에서는 @ResponseStatus를 통해 클라이언트에 예외의 의미를 전달한다.

```java
@ResponseStatus(HttpStatus.NOT_FOUND)
public class EventNotFoundException extends RuntimeException {
    public EventNotFoundException(Long eventId) {
        super("이벤트를 찾을 수 없습니다: " + eventId);
    }
}
```

이 코드는 예외를 명시적으로 던지는 대신, HTTP 404 응답으로 의미를 표현한다. 언어 수준의 예외 선언이 프레임워크 수준의 의미 전달로 확장된 형태다.

## 7.5 ProblemDetail 도입

스프링 6부터는 IETF RFC 9457 표준에 따라 ProblemDetail 응답 구조를 지원한다. 이는 예외 전환이 최종적으로 표현되는 구조적 형태다.

```json
{
  "type": "https://example.com/errors/event-not-found",
  "title": "Event not found",
  "status": 404,
  "detail": "이벤트를 찾을 수 없습니다: 1",
  "instance": "/api/events/1"
}
```

데이터 계층에서 기술별 예외를 공통 구조로 번역하듯, 웹 계층에서도 실패가 구조화된 언어로 표현된다. ProblemDetail은 예외의 의미를 유지한 채, 클라이언트가 이해할 수 있는 표준화된 응답을 제공한다.

# 8. 예외 정의 기준

스프링은 대부분의 실패를 자동으로 처리하고 일관된 구조로 번역하지만, 모든 예외를 대신 정의해주지는 않는다. 프레임워크는 실패를 전달할 수 있으나, 그 실패가 예외로서 존재할 필요가 있는지는 개발자가 판단해야 한다. 즉, “예외를 어떻게 다룰 것인가” 이후에는 “언제, 왜, 어떤 기준으로 정의할 것인가”를 고민해야 한다.

## 8.1 복구 가능한 실패

복구 가능한 실패는 예외를 정의해야 하는 근거가 된다. 사용자의 입력 오류, 존재하지 않는 리소스, 중복 요청처럼 클라이언트가 행동을 수정해 회복할 수 있는 경우에는 명시적인 예외를 정의해야 한다.

```java
public class DuplicateMemberException extends BusinessException {
    public DuplicateMemberException() {
        super("이미 등록된 구성원입니다.");
    }
}
```

이러한 예외는 단순한 오류가 아니라 “복구 방법이 존재한다”는 사실을 드러낸다.

복구 가능한 실패는 두 가지 수준에서 표현된다.

- 코드 수준: BusinessException, InvalidStateException 같은 도메인 예외로 의미를 명시한다.
- 표현(웹) 수준: @ResponseStatus나 ProblemDetail을 통해 클라이언트에 의미를 전달한다.

자바의 throws가 호출자에게 예외의 존재를 알렸다면, 스프링은 도메인 계층과 HTTP 응답 모두에서 예외의 의미를 명시하도록 설계했다. 복구 가능한 실패는 “잡아야 할 예외”가 아니라 “드러내야 할 의미”다.

## 8.2 복구 불가능한 실패

복구할 수 없는 실패는 처리 대상이 아니다. 다만 상위 계층이나 클라이언트가 그 실패를 해석해야 하는지 여부에 따라 예외 정의 여부가 달라진다. 데이터베이스 연결 단절, 외부 API 타임아웃, 네트워크 장애 등 시스템 외부 요인으로 발생한 실패는 보통 복구 불가능하다. 이 경우 예외는 단순히 로그로 남기거나 내부적으로 변환해 전달하면 충분하다.

```java
try {
    repository.save(entity);
} catch (SQLException e) {
    throw new DataAccessException("데이터베이스 접근 실패", e);
}
```

이 코드는 복구를 시도하지 않지만, 예외를 상위 계층이 이해할 수 있는 형태로 변환한다. SQLException을 DataAccessException으로 번역함으로써 데이터베이스 종류와 무관하게 “데이터 접근 실패”라는 의미를 전달한다. 그러나 이 예외를 클라이언트에 그대로 노출하는 것은 불필요하다. 이 경우에는 내부 로그로 남기고 일반적인 500 응답으로 대체하는 편이 적절하다.

스프링은 예외를 무조건 전파하거나 감추지 않는다. 의미가 필요한 예외만 선택적으로 번역하도록 설계했다. 이를 통해 불필요한 예외 정의를 줄이고, 계층 간 의미의 혼선을 최소화한다.

| 계층     | 예외의 역할           | 전달 대상     | 정의 조건                    |
| ------ | ---------------- | --------- | ------------------------ |
| 데이터 계층 | 기술적 실패의 표현       | 프레임워크 내부  | 상위 계층이 기술 원인을 알아야 하는 경우  |
| 서비스 계층 | 업무적 의미로 변환       | 애플리케이션 내부 | 복구나 로직 분기 필요 시           |
| 표현 계층  | 사용자에게 전달될 언어로 변환 | 클라이언트     | 사용자가 이해하고 행동을 수정해야 하는 경우 |

복구 불가능한 실패라도 상위 계층이 의미를 해석하지 않는다면, 예외를 정의할 필요가 없다. 의미가 없는 예외는 단순한 노이즈에 불과하다.

## 8.3 예외 정의를 위한 점검 기준

예외 정의는 단순히 새로운 클래스를 추가하는 일이 아니라, 시스템이 실패를 어떻게 해석하고 어디까지 전달할지를 결정하는 설계 행위다. 불필요한 예외는 코드의 복잡도를 높이고 디버깅을 어렵게 만든다. 예외를 정의하기 전에 다음 두 가지를 점검해야 한다.

1. 이 실패는 복구할 수 있는가? - 복구가 가능하다면 반드시 명시적으로 표현해야 한다. 입력 수정, 재시도, 비즈니스 규칙 조정 등으로 해결 가능한 실패라면 숨기지 않는다. 예를 들어 InvalidRequestException, DuplicateMemberException은 사용자가 수정 가능한 오류임을 나타낸다.
2. 복구할 수 없다면, 누가 이 실패를 이해해야 하는가? - 상위 계층이 의미를 해석해야 한다면 SQLException을 DataAccessException으로, IOException을 StorageFailureException으로 번역한다. 반대로 어느 계층에서도 의미가 소비되지 않는다면 로그로 남기고 500 응답으로 대체한다.

결국 복구 가능한 실패는 드러내고, 복구 불가능한 실패는 의미가 필요한 곳에서만 번역한다. 예외는 모든 실패를 포착하기 위한 장치가 아니라, 전달할 가치가 있는 실패를 표현하기 위한 수단이다.