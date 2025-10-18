# 1. 들어가며: 예외는 왜 존재하는가

소프트웨어는 실패를 전제로 움직인다. 입력은 사양에서 벗어나고, 파일은 없거나 잠겨 있으며, 네트워크는 중간에 끊기고, 데이터베이스는 때때로 응답하지 않는다. 예외(exception)는 이런 실패를 제어 흐름 안으로 끌어들여 다룰 수 있게 하는 언어적 장치다.

예외는 단순한 오류가 아니다. 예외는 **정상 흐름에서 벗어난 제어 흐름**이며, 프로그램을 멈추기 위한 비상정지 장치가 아니라 **실패를 다룰 기회**를 제공하는 구조다.

자바는 이 기회를 언어 차원에서 확장했다. 자바의 철학은 명확했다. “실패를 인지하고, 반드시 처리하라.” 이 강제된 주의의 결과물이 바로 Checked Exception이다. 자바가 제시한 질문은 단순하다. 이 실패를 알고 있는가? 알고 있다면 어떻게 처리할 것인가? 이 질문을 컴파일 타임까지 끌어올린 것이 자바의 안전성 철학이었다.

이 설계는 작은 프로그램에서는 분명 효과가 있었다. 메서드 시그니처에 드러난 `throws` 목록은 호출자에게 “여기서 이런 위험이 생긴다”를 명확히 보여줬고, IDE는 경로를 따라가며 어떤 예외가 발생할 수 있는지 알려주었다. 그러나 프로그램의 규모가 커질수록 문제는 드러났다. 하위 계층의 예외가 상위 계층까지 연쇄적으로 전파되면서 예외 선언은 호출 체인을 따라 증식했다. 예외를 복구할 수 없는 상황에서도 `try-catch` 블록을 강제로 추가해야 했고, 의미 없는 포장이 늘어났다.

“복구 가능한 실패를 명시하라”는 철학은, 어느 순간 “잡아서 다시 던져라”는 관습으로 바뀌었다. 이 글은 바로 그 지점에서 출발한다. 자바가 왜 Checked Exception을 만들었고, 왜 시간이 지나며 그것이 한계에 부딪혔는지, 그리고 스프링이 어떤 방식으로 그 철학을 재해석했는지를 따라가 본다.

# 2. 자바의 예외 체계: 신뢰성을 위한 실험

자바는 예외를 **Error**와 **Exception** 두 축으로 나누고, 그중 Exception을 다시 **Checked**와 **Unchecked**로 구분한다. 이 구분은 단순히 문법상의 분류가 아니라, **복구 가능성(recoverability)** 이라는 기준에 따라 설계된 구조다.

| 구분 | 상위 클래스 | 처리 강제 여부 | 대표 예시 | 의도 |
| --- | --- | --- | --- | --- |
| Error | `java.lang.Error` | 불가능 | `OutOfMemoryError`, `StackOverflowError` | JVM 수준 오류 |
| Checked Exception | `Exception` (단, `RuntimeException` 제외) | 필수 | `IOException`, `SQLException` | 복구 가능한 예외 |
| Unchecked Exception | `RuntimeException` | 선택 | `NullPointerException`, `IllegalArgumentException` | 프로그래밍 오류 |

이 구조의 의도는 명확했다. Error는 애플리케이션이 다룰 수 없는 영역이다. 메모리가 부족하거나 스택이 넘치는 상황은 프로그램이 제어할 수 없다. 따라서 그냥 종료하는 것이 맞다. 반면 Exception은 애플리케이션 내부나 외부 자원에서 발생하는 **예상 가능한 실패**를 표현한다. 그중에서도 Checked Exception은 반드시 처리(catch)하거나 던져야(throws) 한다.

예를 들어 다음 코드는 컴파일조차 되지 않는다.

```java
public void readFile(String path) {
    FileReader reader = new FileReader(path); // 컴파일 에러!
}
```

`FileReader`는 `FileNotFoundException`을 던질 수 있기 때문이다. 자바는 이 예외를 “복구 가능한 실패”로 보고, 개발자에게 “반드시 다루라”고 요구한다. 즉, 언어 차원에서 안전성을 강제한 실험이었다.

## 2.1 안전성을 위한 설계 의도

C++ 시대의 오류 처리는 대부분 리턴값에 의존했다. 함수가 실패하면 특수한 리턴값(예: `-1` 또는 `null`)을 돌려줬고, 호출자가 그것을 일일이 확인해야 했다. 하지만 확인을 빼먹는 일은 너무 흔했다. 오류는 조용히 지나가고, 문제는 나중에야 드러났다.

자바는 이 실수를 반복하지 않으려 했다. 그래서 예외를 타입 시스템에 포함시켰다. 예외가 함수 시그니처의 일부가 되어, 호출자는 반드시 그 존재를 인식해야 했다.

```java
public void copyFile(String src, String dest) throws IOException {
    // ...
}
```

이제 IDE는 호출자에게 경고한다. “이 메서드는 IOException을 던질 수 있습니다. 처리하지 않으면 컴파일되지 않습니다.” 이 덕분에 작은 프로그램에서는 오류를 놓치기 어려웠다. 자바는 언어 차원에서 **‘안전한 프로그래밍’을 강제하는 언어**로 자리 잡았다.

## 2.2 강제의 그림자: 체인의 확산

하지만 프로그램이 커질수록 문제는 명확해졌다. 하위 계층에서 Checked Exception을 던지면, 상위 계층은 선택의 여지 없이 `throws`를 선언해야 했다. 그 예외를 잡아 처리하지 않으면, 호출자도 똑같이 던져야 한다. 결국 예외 선언은 **호출 체인을 따라 증식**했다.

```java
public void process() throws IOException {
    readFile(); // 내부에서 IOException 발생
}
```

한 계층의 변화가 수십 개의 인터페이스 시그니처 수정으로 번진다. 이는 **Open-Closed Principle(OCP)** 를 정면으로 위반한다. 하위 계층의 작은 수정이 상위 계층 전체를 흔드는 구조가 된 것이다. 이때부터 개발자들은 Checked Exception을 ‘형식적’으로 처리하기 시작했다. 예외를 복구하지 않고, 그저 다시 던지거나 포장했다.

```java
try {
    repository.save(user);
} catch (SQLException e) {
    throw new RuntimeException(e);
}
```

이 코드는 언어의 요구를 만족하지만, 의미적으로는 아무 복구도 하지 않는다. 결국 Checked Exception은 “안전성 보장” 대신 “형식적 의무”로 전락했다.

## 2.3 신뢰성 실험의 결과

자바의 Checked Exception 설계는 초기에 분명 매력적이었다. 작은 규모에서는 예외를 명시적으로 다루게 하여 오류를 조기에 드러냈고, IDE의 정적 분석과 결합하면 높은 안정성을 제공했다. 그러나 규모가 커질수록 “복구 가능한가?”라는 기준은 모호해졌다.

파일을 읽는 예외(`IOException`)는 복구할 수 있는가? 데이터베이스 연결 예외(`SQLException`)는 재시도로 복구 가능한가? 많은 경우, 답은 “아니오”였다. 이처럼 **복구 가능성에 대한 판단을 언어가 강제하기에는 세계가 너무 복잡**했다. 자바의 Checked Exception은 결국 개발자의 선택에 의존하게 되었고, 현대 자바에서는 점점 `RuntimeException` 중심의 코드가 늘어갔다.

# 3. 체크 예외의 철학적 의도와 한계

Checked Exception은 단순한 문법 장치가 아니라, **언어가 개발자에게 건넨 질문**이었다. “이 실패를 알고 있는가?” “그렇다면 어떻게 처리할 것인가?” 자바는 이 질문을 코드의 표면으로 끌어올렸다. 즉, **실패를 감추지 않고 드러내는 정직한 프로그래밍**을 요구한 것이다.

## 3.1 Checked Exception이 지키려 했던 세 가지 가치

자바가 Checked Exception을 설계할 때, 그 중심에는 세 가지 원칙이 있었다.

1. **예상 가능한 실패를 명시적으로 드러내라.** - 예외를 메서드 시그니처에 선언함으로써 호출자가 대비할 수 있게 한다.
2. **복구 가능한 실패는 직접 처리하라.** - 강제된 `try-catch`를 통해 시스템 안정성을 높인다.
3. **비정상 상황을 정상 제어 흐름 안으로 편입하라.** - 오류를 숨기지 않고 프로그램 구조 안에서 관리한다.
    

이 세 가지는 당시로선 매우 이상적인 목표였다. 자바는 “개발자에게 실수를 허용하지 않는 언어”를 지향했고, Checked Exception은 그 철학의 상징이었다. 조슈아 블로크는 *이펙티브 자바*에서 이렇게 정리했다.

> “복구 가능한 상황에는 Checked Exception을, 프로그래밍 오류에는 RuntimeException을 사용하라.”
> 

즉, Checked Exception은 **복구를 기대할 수 있는 실패**를 다루기 위한 것이었다.

## 3.2 그러나 ‘복구 가능성’은 모호하다

문제는 바로 그 핵심 개념, **복구 가능성(recoverability)** 이었다. 예를 들어 `SQLException`을 생각해보자. 이 예외는 Checked Exception이다. 하지만 데이터베이스 연결이 끊겼을 때, 개발자가 정말 복구할 수 있을까? DB 서버가 다운된 상황에서 애플리케이션이 스스로 연결을 복구하는 것은 거의 불가능하다.

그럼에도 불구하고 자바는 이를 Checked Exception으로 분류했다. “복구 가능할 수도 있으니 직접 처리하라”고 강제한 것이다. 결국 대부분의 개발자는 다음과 같은 코드를 작성하게 된다.

```java
try {
    repository.save(user);
} catch (SQLException e) {
    throw new RuntimeException(e);
}
```

복구 로직은 없다. 단지 컴파일러의 요구를 만족시키기 위해 다시 던질 뿐이다. 이 시점에서 Checked Exception은 **강제된 형식**이 되고, 의미는 사라진다.

## 3.3 강제의 역설: 안전을 위한 제도가 생산성을 해친다

자바의 Checked Exception은 처음에는 “실패를 다루는 습관”을 길러주었다. 그러나 시간이 지나면서 그 제도는 **생산성을 갉아먹는 형식적 의무**로 바뀌었다. 하위 계층의 코드가 새로운 Checked Exception을 던지면 상위 계층의 모든 메서드 시그니처를 수정해야 했다. 인터페이스, 서비스, 컨트롤러를 거쳐 수십 개의 파일이 함께 변한다.

이는 **OCP(Open-Closed Principle)** 를 위반한다. “변경에는 닫혀 있고 확장에는 열려 있어야 한다”는 객체지향의 기본 원칙이, 예외 선언 하나로 무너진다. 작은 안전을 위해 큰 비용을 치르는 셈이었다.

이 상황을 로버트 C. 마틴은 다음과 같이 표현했다.

> “논쟁은 끝났다. Checked Exception은 실수였다. 복구할 수 없는 예외를 강제하는 순간, 시스템은 유연성을 잃는다.”
> 

이 말은 단순한 감정적 비판이 아니라, 언어 설계의 근본 문제를 짚는다. **“복구 불가능한 실패에 대한 강제”는 불필요하다.**

## 3.4 Checked Exception의 구조적 한계

이런 철학적 한계는 결국 코드 구조에서도 명확히 드러났다. Checked Exception이 낳은 구조적 문제는 다음과 같다.

1. **계층적 확산** – 한 계층의 Checked Exception이 상위 호출자 전체를 따라간다. 이는 설계의 경계를 흐리고, 모듈 간 결합도를 높인다.
2. **의미의 희석** – “복구할 수 있는 예외”와 “복구할 수 없는 예외”가 코드상 구분되지 않는다. 결국 `catch` 블록은 로그만 남기고 다시 던지는 역할로 전락한다.
3. **예외의 중복 포장** – 많은 코드가 Checked Exception을 받아 다시 RuntimeException으로 감싼다. 이렇게 중첩된 스택트레이스는 오히려 디버깅을 어렵게 만든다.
4. **API의 불편한 의무** – 공개 API에서 Checked Exception을 사용하면, 그 API를 호출하는 모든 개발자가 동일한 의무를 진다. 이는 API의 확장성을 떨어뜨린다.

결국 Checked Exception은 “의도는 좋았지만, 구조는 불편한 제도”가 되어버렸다.

## 3.5 그래도 남은 의의

그렇다고 Checked Exception이 완전히 잘못된 개념은 아니다. 이 철학 덕분에 자바는 다른 언어들보다 **실패를 명시적으로 표현하는 문화**를 만들었다. 예외를 무시하지 않고 프로그램의 흐름 안에서 다루려는 태도는 자바가 남긴 중요한 유산이다. 즉, Checked Exception은 ‘실패를 드러내는 습관’을 길러줬다. 그 자체가 오늘날의 예외 설계에 밑바탕이 된다.

# 4. 복구할 수 없는 예외들: 스프링이 마주한 현실

이제 문제의 무대는 언어가 아니라, 프레임워크로 옮겨온다. 스프링이 처음 등장했을 때, 자바 개발자들은 이미 Checked Exception의 한계를 체감하고 있었다. 복구할 수 없는 예외들—데이터베이스 연결 끊김, 트랜잭션 오류, 외부 API 타임아웃—이 코드 전반을 뒤덮고 있었다. 이 상황에서 스프링은 단순히 불편함을 줄이는 게 아니라, **예외의 철학 자체를 다시 정의해야 했다.**

## 4.1 현실의 문제: 잡을 수 없는 예외들

실제 코드에서 Checked Exception은 대부분 “잡아서 다시 던지는 코드”로 귀결됐다. 복구보다는 형식적인 처리였다.

```java
try {
    connection.prepareStatement("INSERT INTO users ...");
} catch (SQLException e) {
    logger.error("Database error", e);
    throw new RuntimeException(e);
}
```

이 코드는 문법적으로는 올바르지만, 의미적으로는 아무것도 하지 않는다. 데이터베이스가 복구되지도, 트랜잭션이 회복되지도 않는다. 그저 로그를 남기고 다시 던질 뿐이다. 결국 Checked Exception이 약속한 “복구 가능한 실패의 명시”는 현실 속에서 무너졌다. 예외를 강제로 선언하는 일은 **불필요한 반복과 중복 포장**으로 이어졌다. 더 큰 문제는 **예외의 의미가 사라진다는 것**이었다.

## 4.2 언어에서 프레임워크로: 스프링의 문제의식

이 문제를 처음 정면으로 다룬 것이 바로 **스프링 프레임워크**였다. 스프링은 단순한 기술 스택이 아니라, 자바 언어의 불편한 설계를 보완하려는 시도에서 출발했다. 즉, 스프링의 목표는 “Checked Exception을 없애는 것”이 아니라 “예외를 의미 있게 재구성하는 것”이었다. 스프링은 이렇게 질문했다. 
> “복구할 수 없는 예외라면, 왜 개발자에게 catch를 강제하는가?”
>

그 답은 분명했다. 스프링은 **복구할 수 없는 예외를 모두 런타임으로 전환**하고, **복구 가능한 예외만 선택적으로 다루는 구조**를 택했다.

# 5. 복구 가능성으로 재정의된 예외 체계

스프링은 자바의 Checked Exception 철학을 부정하지 않았다. 오히려 그것을 **현실에 맞게 재해석**했다. Rod Johnson이 지적했듯, 핵심은 단순히 “Checked Exception을 없앨 것인가”가 아니라 “그 예외가 진짜 복구 가능한가를 어떻게 표현할 것인가” 에 있었다.  자바는 이 질문의 답을 컴파일러에게 맡겼다. 복구 가능성의 판단을 언어가 강제하도록 설계한 것이다. 반면 스프링은 그 책임을 프레임워크의 구조로 옮겼다.

## 5.1 SQLException: 복구 불가능한 Checked Exception의 대표 사례

자바에서 가장 유명한 Checked Exception은 `SQLException`이다. JDBC API는 이를 “복구 가능한 실패”로 보고 모든 데이터 접근 코드에 `throws SQLException`을 강제했다. 하지만 현실에서 대부분의 SQL 예외는 복구 불가능했다. 네트워크 단절, 커넥션 타임아웃, 트랜잭션 충돌 같은 오류는 애플리케이션이 스스로 해결할 수 있는 성질이 아니었다. 그럼에도 컴파일러는 “catch하거나 던져라”를 요구했다. 개발자는 어쩔 수 없이 이런 코드를 작성했다.

```java
try {
    repository.save(user);
} catch (SQLException e) {
    throw new RuntimeException(e);
}
```

복구는 없고, 단지 형식만 남았다. 스프링은 바로 이 모순에서 출발했다. **복구 불가능한 예외를 강제로 다루게 하지 않겠다.** 대신 “의미를 해석하고 전달할 수 있도록” 구조를 바꾸기로 했다. 그 첫 결과물이 바로 `DataAccessException` 계층이었다.

## 5.2 복구 가능성의 계층화

스프링은 더 이상 예외를 Checked / Unchecked로 나누지 않는다. 대신 **“복구 가능성(recoverability)”** 을 기준으로 예외를 세밀하게 구분한다. 이 구조는 `org.springframework.dao` 패키지의 예외 계층에 잘 드러난다. 모든 데이터 접근 예외의 루트 클래스는 `DataAccessException`이며, 그 아래는 복구 가능성의 정도에 따라 세 방향으로 갈라진다.

| 구분 | 루트 클래스 | 복구 조건 | 의미 |
| --- | --- | --- | --- |
| **TransientDataAccessException** | 즉시 재시도하면 성공 가능 | 애플리케이션 개입 없이 재시도만으로 복구 | 네트워크 지연, 락 경합 |
| **RecoverableDataAccessException** | 복구 절차 수행 후 재시도 가능 | 커넥션 재획득, 트랜잭션 재시작 등 수동 복구 필요 | 연결 재설정, 세션 재생성 |
| **NonTransientDataAccessException** | 원인 수정 없이는 실패 | 데이터나 코드 교정 없이는 재시도로 불가 | 무결성 위반, 중복 키, 제약 조건 오류 |

이 계층은 단순한 분류표가 아니라, Checked Exception의 철학을 프레임워크의 구조로 옮긴 결과물이다. 자바가 컴파일러를 통해 “복구 가능한 실패를 명시하라”고 강제했다면, 스프링은 예외 계층을 통해 “복구 가능성의 정도를 구조적으로 표현하라”고 제시한다.

즉, 이제는 예외를 "잡을 수 있는가?"가 아니라, "복구할 수 있는가?"가 기준이 된다. 이를 통해 스프링은 Checked Exception의 철학을 ‘형식의 강제’에서 ‘의미의 표현’으로 바꿔냈다.

## 5.3 Checked Exception 철학의 계승

결국 스프링은 Checked Exception의 이상을 버린 것이 아니라, 그 철학을 **형식에서 의미로 옮겼다.** 즉, 자바가 언어로 강제했던 것을 스프링은 프레임워크의 구조와 규칙으로 자연스럽게 흘려보냈다. Checked Exception의 불편한 강제는 사라졌지만, 그 정신 — *복구 가능한 실패를 명시하라* — 는 여전하다.

# 6. 예외 전환(Exception Translation): 의미를 잃지 않게 던지기

스프링은 Checked Exception의 강제적 처리를 없앴지만, 예외를 단순히 “던지고 끝내는” 구조로 두지 않았다. 오히려 예외를 **이해할 수 있는 형태로 번역(translate)** 하여, 복구가 불가능하더라도 실패의 의미를 잃지 않게 했다. 스프링이 말하는 **예외 전환(Exception Translation)** 은 바로 이 ‘의미를 보존하는 번역’을 뜻한다.

## 6.1 기술 종속성을 끊고 의미를 통일하다

스프링의 공식 문서는 이렇게 설명한다.

> “Spring provides a convenient translation from technology specific exceptions like SQLException to its own exception hierarchy with the DataAccessException as the root exception. These exceptions wrap the original exception so there is never any risk that you would lose any information as to what might have gone wrong.” 
> 
> 
> — Spring Framework Reference, §10.2 Consistent Exception Hierarchy

즉, 스프링은 데이터 접근 기술마다 달랐던 예외(`SQLException`, `HibernateException`, `PersistenceException`, `JDOException`)를 모두 `DataAccessException` 계층으로 번역한다. 이때 예외는 단순히 감싸지는 것이 아니라 **‘실패의 의미’를 공통된 언어로 표현**한다.

| 원래 예외 | 변환된 예외 | 의미 |
|------------|--------------|------|
| `SQLException` | `DuplicateKeyException` | 중복 키 위반 |
| `SQLException` | `DataIntegrityViolationException` | 무결성 제약 위반 |
| `HibernateException` | `HibernateJdbcException` | Hibernate 내부 JDBC 오류 |
| `PersistenceException` | `JpaObjectRetrievalFailureException` | 엔티티 조회 실패 |

이 계층은 특정 기술에 종속되지 않고, 애플리케이션이 이해할 수 있는 **일관된 실패 모델**을 제공한다.

## 6.2 예외를 해석하고 번역한다

스프링의 예외 전환은 내부적으로 **Translator** 또는 **AOP 프록시**를 통해 수행된다. 각 기술 스택마다 예외를 전환하는 방식이 다르다.

| 기술 스택 | 전용 Translator |
|------------|-----------------------------|
| **JDBC** | `SQLErrorCodeSQLExceptionTranslator` |
| **Hibernate** | `HibernateExceptionTranslator`, `HibernateJpaDialect` |
| **JPA (표준)** | `PersistenceExceptionTranslationPostProcessor` |
| **JDO** | `JdoExceptionTranslator` |

예를 들어, Spring JDBC는 다음과 같이 작동한다.

```java
try {
    ps.executeUpdate();
} catch (SQLException e) {
    throw getExceptionTranslator().translate("update", sql, e);
}
```

`SQLExceptionTranslator`는 데이터베이스의 SQL 상태 코드나 벤더별 에러 코드를 해석해 적절한 `DataAccessException` 하위 클래스로 변환한다. 반면 JPA나 Hibernate의 경우 `@Repository`가 붙은 클래스에 AOP 프록시를 적용하여 `PersistenceException`이나 `HibernateException`을 감지하고 `DataAccessException`으로 변환한다. 이 역할을 수행하는 것이 `PersistenceExceptionTranslationPostProcessor`이다.

이러한 구조 덕분에 개발자는 각 기술의 세부적인 예외를 몰라도, 언제나 같은 방식으로 예외를 해석할 수 있다. **실패의 형태는 다르더라도, 의미는 통일된다.**

## 6.3 예외의 언어를 일원화하다

스프링의 예외 전환은 단순한 편의 기능이 아니다. 자바의 Checked Exception이 “실패를 반드시 처리하라”고 강제했다면, 스프링의 Exception Translation은 “실패를 일관된 언어로 이해하라”고 제안한다.

> “This allows you to handle most persistence exceptions, which are non-recoverable, only in the appropriate layers, without annoying boilerplate catches/throws.” 
> 
> 
> — Spring Framework Reference, §10.2

즉, 스프링은 예외를 잡아 처리하는 대신 **의미를 해석해 전달**한다. 이제 개발자는 “어떤 예외를 잡을까?”가 아니라 “이 실패는 무엇을 의미하는가?”를 묻게 된다.

스프링의 예외 전환은 실패의 의미를 **이해할 수 있게 만드는 것**에 집중한다. 예외는 더 이상 단순한 오류 신호가 아니라, 시스템이 스스로 실패를 설명하는 언어다. 예외 전환은 이 언어를 정제하여 기술적 혼란을 제거하고, 개발자가 **맥락 있는 실패를 해석할 수 있는 토대**를 마련한다.

# 7. 웹에서도 자바답게: 예외 전환의 확장

스프링의 예외 철학은 데이터 접근 계층에서 멈추지 않는다. 그 철학은 컨트롤러, 응답, 그리고 HTTP의 세계로까지 이어진다. 웹의 실패 역시 프로그램의 실패이기 때문이다. 스프링은 데이터베이스의 `SQLException`을 `DataAccessException`으로 번역하듯, 컨트롤러의 예외도 **HTTP 응답**이라는 의미 있는 언어로 번역한다.

이것이 바로 **예외 전환(Exception Translation)** 의 웹 확장판이다. 예외는 더 이상 단순한 객체가 아니라, 클라이언트에게 전달되는 **의미 있는 메시지**가 된다.

## 7.1 웹 환경에서 예외는 다른 이름의 응답이다

스프링 MVC에서 컨트롤러는 메서드의 결과를 HTTP 응답으로 직렬화한다. 그런데 예외가 발생하면 정상적인 반환 흐름이 깨진다. 이때 프레임워크는 예외를 “에러 응답”이라는 또 다른 표현으로 바꿔야 한다. 즉, **예외 전환(Exception Translation)** 은 이제 **HTTP 응답 변환(Response Translation)** 으로 확장된다.

| 계층 | 예외 전환 대상 | 변환 결과 |
| --- | --- | --- |
| 데이터 계층 | `SQLException` → `DataAccessException` | 런타임 예외 |
| 웹 계층 | `Exception` → `HTTP Response` | 상태 코드 + 메시지 |

이 구조는 단순하지만, 중요한 전환을 보여준다. 데이터 접근 계층에서 기술별 예외를 추상화한 것처럼, 웹 계층에서는 예외를 **표준화된 응답 구조로 추상화**한다. 이때 스프링은 동일한 원칙을 유지한다 — **의미는 통일하고, 표현은 일관되게 만든다.**

## 7.2 예외 전환의 연속선: SQLException에서 HTTP까지

스프링의 예외 전환은 사실상 하나의 연속적인 흐름으로 작동한다. 데이터 접근 계층에서 발생한 `SQLException`은 `DataAccessException`으로 번역되고, 서비스 계층을 거쳐 컨트롤러에 도달하면 결국 **HTTP 응답이라는 최종 형태**로 전환된다.

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

아래와 같은 HTTP 응답으로 변환된다.

```
HTTP/1.1 409 Conflict
Content-Type: application/json
{
  "error": "이미 등록된 사용자입니다."
}
```

데이터베이스의 `SQLException`이 `DataIntegrityViolationException`으로, 그리고 그것이 다시 `DuplicateUserException` → 409 응답으로 번역되는 이 흐름은 **예외 전환의 일관된 구조적 표현**이다. 계층마다 다른 책임을 가지지만 철학은 하나다 — “각 계층은 자신이 이해할 수 있는 언어로 예외를 번역한다.”

## 7.3 스프링 MVC의 예외 전환 구조

스프링 MVC는 `DispatcherServlet`을 중심으로 예외 전환을 수행한다. 컨트롤러에서 예외가 던져지면 다음 순서로 동작한다.

1. `DispatcherServlet`이 예외를 감지한다.
2. 등록된 `HandlerExceptionResolver` 목록을 순서대로 탐색한다.
3. 해당 예외를 처리할 수 있는 Resolver가 적절한 응답 객체를 생성한다.
4. 이 응답이 `ResponseEntity` 혹은 JSON 형태로 직렬화되어 클라이언트에게 전달된다.

스프링은 이미 여러 기본 Resolver를 제공한다.

| 클래스 | 역할 |
| --- | --- |
| `ExceptionHandlerExceptionResolver` | `@ExceptionHandler` 기반 처리 |
| `ResponseStatusExceptionResolver` | 예외 클래스의 `@ResponseStatus` 처리 |
| `DefaultHandlerExceptionResolver` | 스프링 내부 표준 예외 처리 (`HttpRequestMethodNotSupportedException` 등) |

이 체계 덕분에 개발자는 예외를 직접 응답으로 바꾸는 로직을 작성할 필요가 없다. **예외의 의미만 정의하면**, 프레임워크가 나머지를 대신한다.

## 7.4 명시적 예외 선언: 자바의 throws를 HTTP로 옮기다

스프링은 자바가 언어 차원에서 하던 일을, 웹 계층에서는 **명시적 예외 클래스로 표현**하게 만든다.

```java
@ResponseStatus(HttpStatus.NOT_FOUND)
public class EventNotFoundException extends RuntimeException {
    public EventNotFoundException(Long eventId) {
        super("이벤트를 찾을 수 없습니다: " + eventId);
    }
}
```

이 코드는 `throws EventNotFoundException` 대신, HTTP 404 상태 코드로 “의미를 드러내는 선언”이다. 자바의 `throws`가 컴파일러에게 예외의 존재를 알렸다면, 스프링의 `@ResponseStatus`는 클라이언트에게 그 의미를 전달한다. 이것이 Checked Exception의 철학 — *복구 가능한 실패는 드러내라* — 가 웹 환경에서 자연스럽게 확장된 형태다.

## 7.5 ProblemDetail: 예외 전환의 완성

스프링 6부터는 IETF 표준 RFC 9457에 따라 `ProblemDetail` 응답 구조를 지원한다. 이는 단순한 에러 메시지가 아니라, 예외 전환이 **최종적으로 도달한 구조적 표현의 형태**다.

```json
{
  "type": "https://example.com/errors/event-not-found",
  "title": "Event not found",
  "status": 404,
  "detail": "이벤트 ID 1을 찾을 수 없습니다.",
  "instance": "/api/events/1"
}
```

데이터 계층에서 기술 예외를 통일된 의미로 번역하듯, 웹 계층에서도 실패는 구조화된 언어로 번역된다. `ProblemDetail`은 예외의 의미를 잃지 않으면서, 클라이언트가 이해할 수 있는 일관된 형태로 전달한다.

# 8. 예외를 정의할 때의 기준: 복구 가능성과 계층의 일관성

스프링은 이제 대부분의 실패를 자동으로 처리하고, 예외를 일관된 구조로 번역한다.   그러나 **스프링이 모든 예외를 대신 정의해주는 것은 아니다.** 프레임워크는 실패를 전달할 수는 있지만, 그 실패가 ‘예외로서 존재할 가치가 있는가’를 판단하는 일은 여전히 개발자의 몫이다.  

즉, 스프링이 “예외를 어떻게 다루는가”를 이해했다면,  이제 우리는 “예외를 언제, 왜, 어떤 기준으로 정의할 것인가”를 고민해야 한다. 그 지점을 출발점으로, 스프링의 예외 설계를 다시 바라보자.

## 8.1 복구 가능한 실패는 드러내라

복구 가능한 실패는 **예외를 정의해야 하는 이유**가 된다. 사용자의 입력이 잘못되었거나, 요청한 리소스가 없거나, 동일한 데이터를 중복으로 요청하는 경우처럼 **클라이언트가 행동을 수정함으로써 회복할 수 있는 상황**은 명시적으로 예외를 만들어야 한다.

```java
public class DuplicateMemberException extends BusinessException {
    public DuplicateMemberException() {
        super("이미 등록된 구성원입니다.");
    }
}
```

이런 예외는 단순한 오류가 아니라 “어떻게 복구할 수 있는가”를 드러내는 신호다.

즉, 클라이언트나 상위 계층이 실패의 의미를 이해하고 재시도나 입력 수정으로 대응할 수 있는 경우에는 반드시 **명시적인 예외를 선언**해야 한다. 복구 가능한 예외는 스프링에서 두 가지 수준으로 표현된다.

- **언어 수준**에서는 `BusinessException`, `InvalidStateException` 같은 도메인 예외로서, 코드 내에서 의미를 직접 표현한다.
- **표현(웹) 수준**에서는 `@ResponseStatus`나 `ProblemDetail`을 통해 그 의미를 클라이언트에게 전달한다.

자바가 `throws`를 통해 호출자에게 예외의 존재를 알렸다면, 스프링은 동일한 철학을 프레임워크 전반에 확장해, 예외의 의미를 **도메인과 HTTP 응답 양쪽에서 명시적으로 드러내도록** 설계했다.

결국 복구 가능한 실패는 “잡아야 할 예외”가 아니라 “드러내야 할 의미”다. 이것이 Checked Exception이 지키려던 철학의 현대적 형태이며, 스프링이 그것을 프레임워크 구조로 이어받은 이유다.

## 8.2 복구 불가능한 실패는 어디까지 전달해야 하는가

복구할 수 없는 실패라면 그 자체를 처리할 필요는 없다. 그러나 그렇다고 해서 **의미 없이 던져도 되는 것은 아니다.** 예외를 정의할 때는 “이 실패가 어디까지 올라가야 해석되는가?”를 함께 고려해야 한다. 그리고 더 중요한 질문이 있다 — **정말 상위 계층이나 클라이언트가 그 실패를 이해할 필요가 있는가?**

복구 불가능한 실패는 주로 시스템 외부 요인에서 발생한다. 데이터베이스 연결 끊김, 외부 API 타임아웃, 네트워크 장애 등은 애플리케이션이 직접 복구할 수 없는 영역이다. 이런 실패는 ‘잡아서 처리’할 필요는 없지만, 시스템 전체의 흐름에서 *의미가 필요한 경우에만* 번역해야 한다. 다시 말해, **예외는 의미가 소비될 곳이 있을 때만 정의되어야 한다.** 로그로 충분하다면 새로운 예외를 만들 이유가 없다.

예를 들어 다음 코드를 보자.

```java
try {
    repository.save(entity);
} catch (SQLException e) {
    throw new DataAccessException("데이터베이스 접근 실패", e);
}
```

이 코드는 복구를 시도하지 않지만, 단순히 예외를 던지는 대신 실패의 맥락을 상위 계층이 이해할 수 있는 형태로 바꿔 전달한다. `SQLException`이라는 기술 종속적인 예외를 `DataAccessException`으로 번역함으로써, 상위 계층은 데이터베이스 종류와 관계없이 “데이터 접근 실패”라는 의미로 인식할 수 있다. 그러나 이 예외가 컨트롤러까지 올라가 클라이언트에게 그대로 노출된다면 오히려 불필요한 정보가 된다. 이런 경우에는 내부 로그로 남기고 일반적인 500 응답으로 대체하는 편이 낫다.

스프링은 바로 이 균형을 잡는다. 예외를 “무조건 전파하거나 감추는 것”이 아니라, **진짜로 이해되어야 할 예외만 의미 있게 번역하도록** 설계했다. 이를 통해 불필요한 예외 정의를 줄이고, 계층 간 의미의 노이즈를 최소화한다.

| 계층 | 예외의 역할 | 전달 대상 | 정의 조건 |
| --- | --- | --- | --- |
| 데이터 계층 | 기술적 실패의 표현 | 프레임워크 내부 | 상위 계층이 기술 원인을 알아야 하는 경우 |
| 서비스 계층 | 업무적 의미로 번역 | 애플리케이션 내부 | 복구나 로직 분기 필요 시 |
| 표현 계층 | 사용자에게 전달될 언어로 변환 | 클라이언트 | 사용자가 이해하고 행동을 수정해야 하는 경우 |

즉, **복구할 수 없는 실패라도 상위 계층이 의미를 소비하지 않는다면, 예외를 번역하거나 정의할 필요가 없다.** 의미가 존재하지 않는 예외는 그저 노이즈일 뿐이다. 예외는 전달하기 위해 던지는 것이 아니라, **이해될 필요가 있을 때만 존재해야 한다.**

## 8.3 예외를 정의하기 전, 스스로에게 던질 두 가지 질문

예외를 정의한다는 것은 단순히 새로운 클래스를 추가하는 일이 아니다. 그것은 **시스템이 실패를 어떻게 해석하고, 어디까지 전달할 것인가를 결정하는 설계 행위**다. 불필요한 예외는 코드의 소음을 늘리고, 의미 없는 포장은 오히려 디버깅을 방해한다. 따라서 예외를 만들기 전에는 반드시 아래 두 가지 질문으로 스스로를 점검해야 한다.

1. **이 실패는 복구할 수 있는가?**
    - 복구할 수 있다면, 반드시 명시적으로 드러내야 한다. 사용자의 입력 수정, 재시도, 비즈니스 규칙의 보정 등으로 회복 가능한 실패라면 그것은 숨기면 안 된다.
    - 예를 들어 `InvalidRequestException`, `DuplicateMemberException` 같은 예외는 클라이언트가 행동을 바꿔 해결할 수 있음을 나타낸다. 이런 예외는 단순한 오류가 아니라 **복구 경로를 암시하는 신호**다.
2. **그렇지 않다면, 어느 계층에서 어떻게 이해되어야 하는가?**
    - 복구할 수 없는 실패라면 이제는 “누가 이 실패를 이해해야 하는가”를 판단해야 한다. 상위 계층이 그 의미를 해석해야 한다면, `SQLException`을 `DataAccessException`으로, `IOException`을 `StorageFailureException`으로 번역하라.
    - 그러나 어느 계층에서도 이 의미를 소비하지 않는다면, 굳이 예외를 정의할 필요가 없다. 단순히 로그로 남기고 500 응답으로 대체하면 충분하다. **이해되지 않을 예외는 존재할 이유가 없다.**

결국 이 두 질문이 예외 정의의 기준이다. 복구 가능한 실패는 드러내고, 복구 불가능한 실패는 의미가 필요한 곳에서만 번역하라. 나머지는 조용히 기록하고 흘려보내면 된다. 예외는 모든 실패를 포착하기 위한 장치가 아니라, **의미를 전달하기 위한 언어**이기 때문이다.