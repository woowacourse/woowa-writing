# 0. 들어가며 

스프링에서 예외 처리는 단순한 오류 제어를 넘어, 프레임워크 전반의 설계 철학을 보여주는 중요한 부분이다. 그럼에도 우리는 종종 @ExceptionHandler나 ProblemDetail 같은 기능만 사용한 채, 그 배경에 담긴 설계 의도를 깊이 생각하지 않은 채 지나치곤 한다.

이 글은 “스프링은 왜 이런 방식으로 예외를 다루는가?”라는 질문에서 출발한다. 자바의 Checked Exception이 가진 철학과 한계를 짚고, 스프링이 그 문제를 어떻게 해석하고 확장했는지 살펴본다. 그리고 더 나아가, 예외를 정의하는 기준에 대해서도 함께 다룬다.

# 1. 예외는 왜 존재하는가

소프트웨어는 실패를 전제로 움직인다. 입력은 사양에서 벗어나고, 파일은 없거나 잠겨 있으며, 네트워크는 중간에 끊기고, 데이터베이스는 때때로 응답하지 않는다. 실패는 예외적인 사건이 아니라, 실행 과정에서 자연스럽게 일어나는 현상이다.

이런 실패를 다루는 방식은 언어마다 다르다. 어떤 언어는 오류 코드를 반환해 호출자에게 실패를 알리고, 호출자는 그 값을 검사해 적절히 대응해야 한다. 하지만 이 방식은 실패를 무시하기 쉽고, 호출자가 검사를 빼먹는 순간 문제는 조용히 지나간다. 이 한계를 보완하기 위해 등장한 방식이 예외(exception)다. 예외는 실패를 감추지 않고 프로그램의 제어 흐름 안으로 끌어들여 다루기 위한 구조적 장치다. 단순히 프로그램을 멈추는 신호가 아니라, 실패를 감지하고 처리할 기회를 명시적으로 제공한다.

그러나 모든 실패가 같은 방식으로 다뤄져야 하는 것은 아니다. 예외는 넓은 의미에서 실패를 표현하는 언어적 구조지만, 그중 일부는 개발자가 예상하고 처리해야 하는 실패를 나타낸다.  예를 들어, 파일이 존재하지 않거나 네트워크가 끊기는 상황은 언제든 발생할 수 있으므로, 이러한 실패는 코드 차원에서 미리 대비해야 한다. 반면 프로그래밍 오류나 버그처럼 개발자의 통제 범위 안에 있는 문제는 예외로 다루기보다는 코드 수정으로 해결해야 한다.

즉, 예외는 모든 실패를 동일하게 처리하기 위한 장치가 아니라, 실패의 성격에 따라 다르게 대응할 수 있도록 구분된 구조다. 자바의 예외 체계는 이 개념을 중심으로 구성되어 있다.

# 2. 자바의 예외 체계

자바는 예외를 Error와 Exception 두 축으로 나누고, 그중 Exception을 다시 Checked와 Unchecked로 구분한다. 이 구분은 단순한 문법적 차이가 아니라, 예상 가능한 실패(predictable failure) 를 코드 수준에서 드러내기 위한 구조다.

| 구분                  | 상위 클래스                                       | 처리 강제 여부 | 대표 예시                                          | 설명                              |
| ------------------- | -------------------------------------------- | -------- | ---------------------------------------------- | ------------------------------- |
| Error               | java.lang.Error                              | 불가능      | OutOfMemoryError, StackOverflowError           | 애플리케이션이 제어할 수 없는 시스템 수준의 오류     |
| Checked Exception   | java.lang.Exception (단, RuntimeException 제외) | 필수       | IOException, SQLException                      | 프로그램이 예상할 수 있는 실패를 명시적으로 표현 |
| Unchecked Exception | java.lang.RuntimeException                   | 선택       | NullPointerException, IllegalArgumentException | 프로그래밍 오류나 잘못된 호출을 표현        |

Error는 애플리케이션이 제어할 수 없는 시스템 수준의 오류다. 이에 비해 Exception은 실행 중 발생할 수 있는 실패를 표현하며, 그중 Checked Exception은 예상 가능한 실패를 코드 수준에서 명시적으로 드러내도록 설계되었다. 이 예외는 반드시 처리(catch)하거나 선언(throws)해야 하며, 그렇지 않으면 컴파일이 허용되지 않는다. 반면 Unchecked Exception은 주로 프로그래밍 오류나 잘못된 호출을 나타내며, 처리 여부는 개발자의 선택에 맡겨진다.

이 구조는 이전 언어의 오류 처리 방식을 개선하기 위해 설계되었다. C언어에서도 파일 접근 실패나 네트워크 오류처럼 예상 가능한 실패를 다루기 위한 장치가 존재했다. 함수가 실패하면 -1이나 null 같은 특수한 리턴값으로 그 사실을 알렸고, 호출자가 이를 직접 확인해야 했다. 하지만 이 검사를 누락하면 실패는 조용히 무시되었고, 프로그램은 오류를 인식하지 못한 채 계속 실행되었다.

자바는 이러한 문제를 해결하기 위해 예외를 타입 시스템에 포함시켰다. 예외가 메서드 시그니처의 일부로 명시되면서, 호출자는 해당 메서드가 발생시킬 수 있는 실패를 반드시 인식하고 처리해야 했다.

예를 들어 다음 코드는 컴파일조차 되지 않는다.

```java
public void readFile(String path) {
    FileReader reader = new FileReader(path); // 컴파일 에러!
}
```

FileReader는 FileNotFoundException을 던질 수 있기 때문이다.  자바는 이런 실패를 예상 가능한 상황으로 보고, 호출자에게 “이 실패를 인식하고 다루라”고 요구한다.

## 2.1 Checked Exception 확산 문제

하지만 프로그램 규모가 커질수록 이 구조는 한계를 드러냈다. 하위 계층에서 Checked Exception을 던지면, 상위 계층은 선택의 여지 없이 throws를 선언해야 했다. 예외를 잡아 처리하지 않으면, 호출자도 똑같이 던져야 한다. 결과적으로 예외 선언이 호출 체인을 따라 증식했다.

```java
public void process() throws FileNotFoundException {
    readFile("data.txt"); // 내부에서 FileNotFoundException 발생
}
```

한 계층의 작은 변화가 상위 계층 전체의 시그니처 수정으로 번지기 시작했다. Checked Exception이 호출 체인을 따라 강제로 전파되면서 계층 간 결합도는 높아지고 변경 비용은 기하급수적으로 커졌다.

그 결과 많은 개발자는 Checked Exception을 의미 없는 형식적 의무로 다루게 되었다. 실제로는 예외를 처리하지 않고, 그저 다시 던지거나 포장하는 코드가 늘어나기 시작했다.

```java
public void process() {
    try {
        readFile("data.txt");
    } catch (FileNotFoundException e) {
        throw new RuntimeException(e); // Checked Exception을 포장해 다시 던짐
    }
}
```

이 코드는 컴파일러의 요구를 충족할 뿐, 실패를 의미 있게 다루지는 않는다. 결국 Checked Exception은 “예상 가능한 실패를 명시한다”는 본래 의도와 달리 “형식적 선언”으로 전락했다.

# 3. Checked Exception의 의도

자바가 Checked Exception을 설계할 때 중심에 둔 원칙은 다음 세 가지다.

1. 예상 가능한 실패를 명시적으로 드러내라. - 예외를 메서드 시그니처에 선언해 호출자가 대비할 수 있게 한다.  
2. 복구 가능한 실패는 직접 처리하라. - 강제된 try-catch로 시스템 안정성을 높인다.  
3. 비정상 상황을 제어 흐름 안에서 관리하라. - 오류를 숨기지 않고 프로그램 구조 안에서 다룬다.  

예외를 메서드 시그니처에 선언하도록 강제한 것도 같은 이유다. 호출자는 메서드가 어떤 위험을 가질 수 있는지 명확히 인식해야 했고, 이에 대한 처리를 명시적으로 작성해야 했다. 즉, 예상 가능한 실패를 코드에 드러내어 호출자가 그 가능성을 인식하고 대응하도록 한 것이다. 이러한 구조를 통해 자바는 실패를 숨기지 않는 언어적 특성을 확보했다.

자바는 여기서 한 단계 더 나아가, 예상 가능한 실패를 단순히 알리는 데 그치지 않고 복구할 수 있는 기회를 제공하려 했다. Checked Exception은 “이 실패를 알고 있다면, 처리하라”는 강제와 함께 “필요하다면 복구하라”는 의도를 포함한다. 이를 통해 예외를 단순한 오류 신호가 아닌 제어 가능한 흐름의 일부로 다루게 되었다.

조슈아 블로크는 이펙티브 자바에서 이러한 철학을 다음과 같이 정리했다.

> “복구 가능한 상황에는 Checked Exception을, 프로그래밍 오류에는 RuntimeException을 사용하라.”

이 문장은 Checked Exception이 단순한 오류 통지가 아니라, 복구 가능한 실패를 표현하기 위한 언어적 장치임을 보여준다.

## 3.1 복구 가능성의 모호함

그러나 실제 개발 환경에서는 이 ‘복구 가능성’의 의미가 명확하지 않았다. 어떤 실패가 복구 가능한지 판단하기 어렵고, 복구가 실제로 무엇을 의미하는지도 불분명했다. 

이 지점에서 Checked Exception의 한계가 드러난다. SQLException은 Checked Exception이지만, 데이터베이스 연결이 끊겼을 때 애플리케이션이 이를 스스로 복구할 수 있을까? DB 서버가 다운된 상황에서 애플리케이션이 직접 연결을 복원하는 것은 현실적으로 어렵다. 그럼에도 자바는 이런 예외를 Checked Exception으로 분류했다. “복구할 수도 있으니 직접 처리하라”는 강제였다.

예상 가능한 결과인 우발적인 상황에서 복구를 시도하려는 의도는 분명했지만, 복구가 실제로 무엇을 수반하는지에 대한 기준은 불명확했다. 연결을 재시도하는 것이 복구인지, 단순히 로그를 남기고 종료하는 것도 복구로 볼 수 있는지 명확하지 않았다. 

결국 대부분의 개발자는 다음과 같은 코드를 작성하게 되었다.

```java
try {
    repository.save(user);
} catch (SQLException e) {
    throw new RuntimeException(e);
}
```

이 코드는 문법적으로는 올바르지만, 실제로는 아무런 복구를 수행하지 않는다. 데이터베이스 연결이 회복되지도, 트랜잭션이 복원되지도 않는다. 로그를 남기고 예외를 다시 던질 뿐이다. Checked Exception의 “복구 가능성”이라는 전제는 현실에서 거의 성립하지 않았다.

Checked Exception을 반대하는 의견의 핵심도 여기에 있다. 대부분의 예외는 복구할 수 없으며, 개발자는 오류가 발생한 코드나 서브시스템을 직접 소유하지도, 그 내부 구현을 수정할 책임도 없다. 따라서 “복구 가능한 실패”라는 전제 자체가 애플리케이션의 통제 범위를 넘어서는 경우가 많다.

## 3.2 강제의 역설

이러한 한계는 Checked Exception이 본래 의도와 다르게 작동하게 만들었다. 초기에는 “실패를 다루는 습관”을 형성하는 데 기여했지만, 시간이 지나면서 생산성을 떨어뜨리는 형식적 의무로 변했다. 하위 계층에서 발생한 예외가 상위 계층 전체로 확산되며, 모든 메서드가 예외를 선언하거나 포장해야 하는 상황이 반복되었다. 작은 변경에도 시그니처 수정이 연쇄적으로 발생했고, 예외는 불필요한 보일러플레이트로 전락했다.

이 상황을 Robert C. Martin은 다음과 같이 표현했다.

> “논쟁은 끝났다. Checked Exception은 실수였다. 복구할 수 없는 예외를 강제하는 순간, 시스템은 유연성을 잃는다.”

이 말은 단순한 비판이 아니라, 복구 불가능한 실패까지 강제로 처리하게 만드는 설계 자체가 불필요하다는 지적이다. 예외의 존재가 개발자의 선택을 넓히지 못하고 오히려 제약으로 작용했다는 의미다.

자바의 방향성도 점차 이를 인정하는 쪽으로 변했다. 자바 8의 함수형 인터페이스는 Checked Exception을 선언하지 않는다. 스트림, 람다, CompletableFuture 등 현대 자바의 API는 Checked Exception을 언어적 강제에서 제외했다.

# 4. 스프링이 마주한 현실

자바가 Checked Exception의 한계를 언어 차원에서 완화했을 때,  스프링은 그 문제를 프레임워크 수준에서 직접 마주해야 했다.  데이터베이스 연결 끊김, 트랜잭션 충돌, 외부 API 타임아웃과 같은 예외들은 대부분 복구가 불가능했지만, 여전히 코드 전반에는 Checked Exception이 남아 있었다.

개발자는 이 예외들을 실제로 복구할 수도, 완전히 무시할 수도 없었다. 언어는 여전히 “catch하거나 던져라”를 요구했기 때문이다. 복구할 방법이 없으니, 결국 형식적인 try-catch로 감싸거나 런타임 예외로 다시 던지는 방식이 관행이 되었다.

## 4.1 SQLException의 모순

이러한 한계는 데이터 접근 계층에서 특히 두드러졌다. 외부 자원에 의존하는 데이터베이스 연동 과정에서 발생하는 실패는 대부분 복구 불가능했고, Checked Exception의 강제는 그 현실을 반영하지 못했다.

대표적인 사례가 바로 JDBC의 SQLException 이다. JDBC API는 이를 “복구 가능한 실패”로 간주해 모든 데이터 접근 코드에 throws SQLException을 강제했다. 그러나 현실에서 대부분의 SQL 예외는 네트워크 단절, 커넥션 타임아웃, 트랜잭션 충돌처럼 애플리케이션이 스스로 해결할 수 없는 문제였다.

그럼에도 컴파일러는 “catch하거나 던져라”를 요구했고, 개발자는 다음과 같은 코드를 작성할 수밖에 없었다.

```java
try {
    repository.save(user);
} catch (SQLException e) {
    throw new RuntimeException(e);
}
```
복구는 없고 형식만 남았다. 개발자는 오류를 처리하지도, 무시하지도 못한 채 “잡는 척” 하는 코드만 반복하게 되었다.

스프링은 이 문제를 단순한 문법적 불편함이 아닌 구조적 문제로 인식했다. 복구 가능성에 대한 판단이 언어에 과도하게 위임된 결과였다. 따라서 Checked Exception을 단순히 제거하기보다, “언제 복구할 수 있고, 언제 복구할 수 없는가”를 프레임워크 수준에서 명확히 구분하는 구조가 필요했다.

Rod Johnson은 이 문제를 다음과 같이 정리했다.

> “논점은 Checked Exception을 없앨 것인가가 아니라, 그 예외가 실제로 복구 가능한가를 어떻게 표현할 것인가이다.”

스프링은 이 관점을 바탕으로, 데이터 접근 예외를 모두 런타임 예외로 통일하되, 그 내부에서 복구 가능성의 정도를 계층적으로 구분하는 방식을 택했다.

## 4.2 복구 가능성 계층화

이 방식은 org.springframework.dao 패키지의 DataAccessException 계층으로 구체화되었다.
DataAccessException은 RuntimeException을 상속하는 루트 클래스이며,
그 하위 클래스들은 복구 가능성(recoverability) 의 수준에 따라 다음 세 가지로 구분된다.

| 구분 | 루트 클래스 | 복구 조건 | 의미 |
| --- | --- | --- | --- |
| TransientDataAccessException | 즉시 재시도하면 성공 가능 | 애플리케이션 개입 없이 재시도만으로 복구 | 네트워크 지연, 락 경합 |
| RecoverableDataAccessException | 복구 절차 수행 후 재시도 가능 | 커넥션 재획득, 트랜잭션 재시작 등 수동 복구 필요 | 연결 재설정, 세션 재생성 |
| NonTransientDataAccessException | 원인 수정 없이는 실패 | 데이터나 코드 교정 없이는 재시도로 불가 | 무결성 위반, 중복 키, 제약 조건 오류 |

이 계층은 Checked Exception의 분류 기준을 언어 수준이 아닌 프레임워크 수준으로 옮긴 것이다. 자바가 컴파일러를 통해 복구 가능성을 강제했다면, 스프링은 예외 계층 구조로 이를 표현한다.

스프링은 Checked Exception의 문제를 단순히 제거하지 않고, 복구 가능성에 따라 구조적으로 구분했다. 언어가 하던 강제를 프레임워크가 대신 수행하도록 바꾼 것이다. 결과적으로 예외는 형식이 아닌 의미 중심으로 다루어졌고, 불필요한 코드 의존성을 줄이면서도 명확한 책임 구분이 가능해졌다.

# 5. 예외 전환(Exception Translation)

스프링은 Checked Exception의 강제적 처리를 제거했지만, 예외를 단순히 던지고 끝내지 않았다. 대신 발생한 예외를 공통된 형태로 변환(translate) 하여, 기술 세부사항과 무관하게 동일한 언어로 다룰 수 있도록 했다. 스프링에서 말하는 예외 전환(Exception Translation) 은 바로 이 “실패의 언어를 일원화하는 과정”을 의미한다.

마찬가지로, 데이터 접근 계층에서 이 철학이 가장 뚜렷하게 드러났다. 데이터베이스 연동 과정은 다양한 기술 스택과 예외 체계를 다루는 대표적인 복잡 영역이었고, 스프링은 이 영역에서 “기술별 예외를 하나의 언어로 통합한다”는 목표를 실현하기 시작했다.

## 5.1 기술 종속성 제거

데이터 접근 기술마다 예외 체계는 제각각이었다. SQLException, HibernateException, PersistenceException, JDOException 등은 각기 다른 규칙과 계층을 따랐고, 결과적으로 애플리케이션은 기술별 예외에 직접 의존해야 했다.

이로 인해 예외 처리 로직은 기술에 묶였고, 데이터 접근 기술을 교체할 때마다 예외 처리 코드까지 함께 수정해야 했다. 스프링은 이러한 기술 종속성을 제거하기 위해, 모든 데이터 접근 예외를 하나의 구조로 통합하는 방식을 제시했다.

스프링 공식 문서는 이를 다음과 같이 설명한다.

> “Spring provides a convenient translation from technology specific exceptions like SQLException to its own exception hierarchy with the DataAccessException as the root exception. These exceptions wrap the original exception so there is never any risk that you would lose any information as to what might have gone wrong.” 
> 
> — Spring Framework Reference, §10.2 Consistent Exception Hierarchy

즉, 스프링은 데이터 접근 기술에서 발생한 다양한 예외를 DataAccessException 계층으로 변환하여, 복잡하고 불규칙하던 예외 체계를 일관된 모델로 정리했다.

| 원래 예외 | 변환된 예외 | 의미 |
|------------|--------------|------|
| SQLException | DuplicateKeyException | 중복 키 위반 |
| SQLException | DataIntegrityViolationException | 무결성 제약 위반 |
| HibernateException | HibernateJdbcException | Hibernate 내부 JDBC 오류 |
| PersistenceException | JpaObjectRetrievalFailureException | 엔티티 조회 실패 |

이 계층은 특정 기술의 세부 구현을 감추면서도, 실패의 의미를 유지하는 일관된 표현 방식을 제공했다. 이것이 바로 스프링의 예외 전환이 단순한 “감싸기(wrapper)”가 아닌 이유다.

## 5.2 예외 해석과 전환

스프링은 예외 전환을 수동으로 강제하지 않았다. 각 기술 스택에 맞는 Translator 또는 AOP 프록시를 제공하여, 개발자가 사용하는 데이터 접근 기술에 따라 자동으로 예외가 변환되도록 했다.

| 기술 스택 | Translator |
|------------|-------------|
| JDBC | SQLErrorCodeSQLExceptionTranslator |
| Hibernate | HibernateExceptionTranslator, HibernateJpaDialect |
| JPA | PersistenceExceptionTranslationPostProcessor |
| JDO | JdoExceptionTranslator |

이 구조 덕분에 개발자는 데이터 접근 기술을 교체하더라도 동일한 예외 구조를 사용할 수 있었다. 스프링은 기술별 예외를 제거함으로써, “실패의 근원은 달라도 해석은 동일하게 유지되는” 환경을 구현했다.

스프링 공식 문서는 이 점을 다음과 같이 설명한다.

> “This allows you to handle most persistence exceptions, which are non-recoverable, only in the appropriate layers, without annoying boilerplate catches/throws.” 
> 
> — Spring Framework Reference, §10.2

즉, 스프링의 예외 전환은 단순히 기술 의존성을 숨기기 위한 추상화가 아니다. 복구 불가능한 예외를 모든 코드에서 강제로 처리하지 않고, 의미가 필요한 계층에서만 해석할 수 있도록 구조화한 설계 방식이다. 

# 6. 웹 환경으로의 확장

스프링의 예외 설계는 데이터 접근 계층에서 끝나지 않는다. 데이터베이스에서 발생한 SQLException을 DataAccessException으로 전환해 의미를 정리했던 것처럼, 웹 계층에서는 내부 예외를 클라이언트가 이해할 수 있는 HTTP 응답으로 전환한다. 

이것도 결국 “실패를 그냥 던지지 말고 해석해서 전달하자”는 같은 철학 위에 있다. 스프링 MVC는 이 과정을 프레임워크 수준에서 기본 제공한다.

## 6.1 웹 응답으로의 전환

스프링 MVC에서 컨트롤러는 메서드의 반환값을 HTTP 응답으로 변환한다. 하지만 예외가 발생하면 정상적인 반환 흐름이 끊기므로, 프레임워크는 예외를 잡아(catch) 적절한 HTTP 상태 코드와 본문으로 전환(translate)해야 한다.

이 동작은 DispatcherServlet 내부의 다음 구조에서 시작된다.

```java
try {
    handlerAdapter.handle(request, response, handler);
} catch (Exception ex) {
    processDispatchResult(request, response, handler, mv, ex);
}
```

DispatcherServlet은 컨트롤러에서 던져진 예외를 잡은 뒤, processDispatchResult() 메서드를 통해 등록된 ExceptionResolver 들에게 위임한다.
그리고 ExceptionResolver들이 예외의 종류와 메타데이터를 해석해 알맞은 HTTP 응답으로 변환한다.

예를 들어, 컨트롤러에서 다음과 같은 예외가 발생했다고 하자.

```java
throw new ResponseStatusException(HttpStatus.BAD_REQUEST, "잘못된 요청입니다.");
```

스프링은 이 예외를 DispatcherServlet에서 잡아, ResponseStatusExceptionResolver를 통해 다음과 같은 응답으로 전환한다.

```json
HTTP/1.1 400 Bad Request
Content-Type: application/problem+json
{
  "type": "about:blank",
  "title": "Bad Request",
  "status": 400,
  "detail": "잘못된 요청입니다.",
  "instance": "/api"
}
```

이처럼 스프링은 예외를 잡아 HTTP 응답으로 전환하는 전체 흐름을 프레임워크 수준에서 담당한다. 개발자는 “무엇이 실패했는가”만 정의하면 되고, “그 실패를 어떻게 응답으로 표현할 것인가”는 스프링이 대신 처리한다. 

그러나 이 구조는 단순히 기술적인 편의 기능에 그치지 않는다. 스프링이 데이터 접근 계층에서 SQLException을 DataAccessException으로 전환했던 것처럼, 웹 계층에서는 내부 예외를 “의미 있는 응답”으로 전환함으로써 서버 내부의 실패를 외부 세계와 소통 가능한 언어로 변환하는 철학적 일관성을 보여준다.

## 6.2 서비스 계층에서의 책임
스프링은 예외를 잡아 HTTP 응답으로 바꾸는 전체 파이프라인을 이미 갖추고 있다. 덕분에 컨트롤러는 직접 try-catch하지 않아도 되고, 응답 JSON을 직접 만들지 않아도 된다. 프레임워크가 “이 실패를 클라이언트에게 이렇게 설명하자”까지 처리해 준다.

그런데 스프링은 한 가지는 절대 대신하지 않는다. 그건 “무엇이 실패했는지”를 정의하는 일이다.

예를 들어 DB에서 Unique 제약 조건이 깨져 DataIntegrityViolationException이 발생했다고 하자. 이건 기술적으로는 “데이터 무결성 위반”이지만, 우리 시스템에서는 상황에 따라 전혀 다른 의미를 가진다.

1. 이미 가입된 이메일이라서 회원을 더 만들 수 없는 건가?
2. 이미 같은 시간대에 예약이 잡혀 있어서 중복 예약이 불가능한 건가?
3. 취소할 수 없는 상태의 리소스를 취소하려 시도한 건가?

이건 기술의 문제가 아니라 비즈니스 규칙의 문제다. 따라서 서비스 계층에서는 이런 기술적 예외를 그대로 바깥으로 올리지 않고, “이 상황이 우리 시스템에서 어떤 의미를 가지는가”를 명시적으로 표현하는 예외로 전환해야 한다.

# 7. 예외 정의 기준

스프링은 실패를 해석하고 응답으로 전환하는 구조를 제공하지만, 그 실패가 ‘무엇을 의미하는가’를 결정하는 일은 여전히 개발자의 몫이다.

이 지점을 가볍게 넘기면 자바의 Checked Exception이 남긴 문제를 반복하게 된다. 예외의 존재 이유를 고민하지 않은 채 “일단 예외 클래스를 하나 만들고 던진다”는 접근은 곧 불필요한 예외 확산으로 이어진다. 결국 예외는 “실패를 표현하는 구조”가 아니라 “형식적 의무”로 전락한다.

스프링의 예외 전환 체계가 “복구 가능성과 불가능성의 구분”을 통해 예외의 의미를 명확히 했듯, 우리도 예외를 정의할 때 같은 기준을 적용해야 한다.

## 7.1 복구 가능한 실패

복구 가능한 실패는 예외를 정의해야 하는 대표적인 경우다. 이 실패는 단순히 오류가 아니라, 정상 흐름으로 되돌릴 여지가 남아 있는 실패를 의미한다.

스프링은 이러한 복구 가능성을 두 단계로 구분한다. TransientDataAccessException은 즉시 재시도만으로 복구 가능한 경우, RecoverableDataAccessException은 일정한 절차를 거친 뒤 재시도 가능한 경우를 뜻한다. 이 기준은 데이터 접근 계층뿐 아니라 일반적인 애플리케이션 예외 정의에도 그대로 적용할 수 있다.

예를 들어, 외부 API 타임아웃이나 일시적인 네트워크 지연, DB 락 경합 등은 일정 시간 후 재시도만으로 복구 가능한 Transient Failure에 해당한다. 이 경우 예외는 단순히 실패를 알리는 것이 아니라,
재시도를 유도하는 신호로서 정의된다.

```java
public class RetryableRemoteCallException extends RuntimeException {
    public RetryableRemoteCallException(Throwable cause) {
        super("외부 API 호출이 일시적으로 실패했습니다. 재시도가 가능합니다.", cause);
    }
}
```

반면, 잘못된 입력, 중복 요청, 정책 위반처럼 시스템 내부의 재시도만으로 해결되지 않지만, 입력 수정이나 외부 조건 충족을 통해 복구 가능한 실패는 Recoverable Failure로 본다. 이 경우 예외는 단순히 “실패했다”가 아니라 “어떤 조건이 충족되면 성공할 수 있다”는 정보를 포함해야 한다.

```java
public class DuplicateMemberException extends RuntimeException {
    public DuplicateMemberException() {
        super("이미 등록된 구성원입니다. 다른 이메일을 사용해 다시 시도하세요.");
    }
}
```

이 예외는 단순한 종료 신호가 아니라, “사용자가 행동을 바꾸면 복구할 수 있다”는 메시지를 전달한다. 복구 가능한 실패의 핵심은 회복의 가능성을 코드로 드러내는 것이다. 이를 통해 상위 계층이나 클라이언트가 실패에 적절히 대응할 수 있다.

## 7.2 복구 불가능한 실패

복구 불가능한 실패는 시스템이 통제할 수 없는 영역에서 발생한다. 외부 API의 장애, 네트워크 단절, 데이터베이스 연결 끊김처럼 코드가 개입해도 결과를 바꿀 수 없는 상황이 여기에 해당한다. 이런 실패는 복구(recovery) 의 대상이 아니라 인지(recognition) 의 대상이다. 

즉, “어떻게 다시 시도할 것인가”보다 “이 실패를 어디까지 전달할 것인가”가 핵심이다.

스프링이 NonTransientDataAccessException을 통해
“이 실패는 재시도로 해결되지 않는다”는 의미를 구분했듯, 우리도 애플리케이션 설계에서 같은 기준을 적용해야 한다.

예를 들어 외부 결제 게이트웨이(PG) 서버가 장애로 응답하지 않는 상황을 생각해보자.

```java
public void processPayment(PaymentRequest request) {
    try {
        tossPaymentsClient.charge(request);
    } catch (HttpServerErrorException e) {
        // 복구 불가능: 외부 결제 API 서버 장애
        throw new PaymentGatewayUnavailableException("결제 게이트웨이 응답 불가", e);
    }
}
```

이 예외는 사용자가 수정할 수도, 시스템이 자동으로 복구할 수도 없다.

그렇다면 여기서 중요한 것은 ‘복구’가 아니라 ‘의미의 전달’이다. 결제 실패가 비즈니스적으로 어떤 영향을 주는지 판단하고, 그 책임이 상위 계층(예: 스케줄러, 서킷 브레이커, 또는 클라이언트)에 있다면 그곳에서 의미를 해석할 수 있도록 예외를 정의해 전달하면 된다.

그렇지 않다면 단순히 로그로 남기거나, PaymentGatewayUnavailableException, ExternalSystemException 같은 공통 예외로 감싸서 던지는 것으로 충분하다.

## 7.3 예외 정의를 위한 점검 기준

예외 정의는 단순히 클래스를 하나 추가하는 일이 아니다. 그렇게 하다 보면 자바가 그랬듯, 실패를 해석하기보다 예외를 양산하는 구조에 빠지게 된다.

따라서 예외를 정의하기 전에는 반드시 다음 두 가지를 점검해야 한다.

1. 이 실패는 복구할 수 있는가? - 즉시 재시도만으로 성공할 수 있는 실패거나, 일정한 복구 절차를 거쳐야 성공할 수 있는 실패라면
그 실패는 단순한 오류가 아니라 복구 가능한 실패로서 예외로 정의되어야 한다.
2. 복구할 수 없다면, 이 실패를 누가 이해해야 하는가? - 상위 계층이 의미를 해석해야 한다면
그곳에서 의미를 해석할 수 있도록 예외를 정의해 전달하면 된다. 그렇지 않다면 단순히 로그로 남기거나, 공통 예외로 감싸서 던지는 것으로 충분하다.