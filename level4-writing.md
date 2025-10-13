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
}public void execute() throws IOException {
    process();
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

## 3.6 한계 이후의 전환점

문제는 결국 철학의 불일치였다. 자바는 “복구 가능한 실패”를 강조했지만, 현실의 대부분 예외는 복구할 수 없었다. 네트워크, 데이터베이스, 파일 I/O — 모두 외부 세계의 문제였다.

이 모순을 정면으로 마주한 것이 바로 **스프링 프레임워크**였다. 스프링은 이렇게 질문했다.

> “복구할 수 없는 예외라면, 왜 개발자에게 catch를 강제하는가?”
> 

이 질문은 단순한 불편함의 해소가 아니라, **철학의 재해석**이었다. 스프링은 Checked Exception의 본래 의도를 버리지 않고, 그 정신을 더 현실적인 방식으로 계승했다. 즉, “복구 가능한 예외만 명시적으로 다루고, 복구 불가능한 예외는 런타임으로 단순화한다”는 선택이었다.

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

## 4.2 예외 의미의 붕괴

현대 애플리케이션에서 예외는 두 종류뿐이었다.

1. **복구 불가능한 시스템 예외** — 데이터베이스 장애, 네트워크 단절, 메모리 부족 등
2. **의도적으로 처리해야 하는 비즈니스 예외** — 규칙 위반, 도메인 검증 실패 등

하지만 Checked Exception은 이 두 세계를 하나로 묶어버렸다. 결과적으로 개발자는 복구 불가능한 오류에도 `try-catch`를 쓰고, 의미 없는 로그를 남기고, 다시 RuntimeException으로 던졌다. 이런 코드가 많아질수록, 예외의 본래 목적—*“실패의 의미를 드러내는 것”*—은 점점 희미해졌다. 강제된 처리 구조가 예외의 의미를 오히려 지워버린 셈이다.

## 4.3 언어에서 프레임워크로: 스프링의 문제의식

이 문제를 처음 정면으로 다룬 것이 바로 **스프링 프레임워크**였다. 스프링은 단순한 기술 스택이 아니라, 자바 언어의 불편한 설계를 보완하려는 시도에서 출발했다. Rod Johnson은 다음과 같이 말했다.

> “스프링은 EJB의 불필요한 복잡성을 제거하려 했다. Checked Exception은 그 복잡성의 대표적인 예였다.”
> 

즉, 스프링의 목표는 “Checked Exception을 없애는 것”이 아니라 “예외를 의미 있게 재구성하는 것”이었다. 스프링은 이렇게 질문했다. “복구 불가능한 예외를 왜 개발자가 직접 처리해야 하는가?” “복구가 불가능하다면, 왜 언어가 강제로 catch를 요구하는가?” 그 답은 분명했다. 스프링은 **복구할 수 없는 예외를 모두 런타임으로 전환**하고, **복구 가능한 예외만 선택적으로 다루는 구조**를 택했다.

## 4.4 Checked Exception을 포기하는 이유

스프링이 내린 결론은 명료했다.

1. 대부분의 인프라 예외는 복구 불가능하다.
2. 복구 불가능한 예외에 대해 catch를 강제하는 것은 불필요하다.
3. 따라서 이러한 예외는 모두 RuntimeException으로 전환해야 한다.

스프링은 이 결론을 `DataAccessException`을 통해 실현했다. 이 예외 계층은 JDBC, JPA, Hibernate 등 다양한 기술에서 발생하는 오류를 공통된 런타임 예외 구조로 감싼다. 그리고 더 이상 Checked Exception을 노출하지 않는다.

예를 들어 JDBC의 `SQLException`은 Checked Exception이다. 하지만 스프링은 내부에서 이 예외를 다음과 같이 변환한다.

```java
try {
    ps.executeUpdate();
} catch (SQLException e) {
    throw getExceptionTranslator().translate("update", sql, e);
}
```

`SQLErrorCodeSQLExceptionTranslator`가 벤더별 SQL 오류 코드를 분석하고, `DuplicateKeyException`, `DataIntegrityViolationException` 등으로 바꾼다. 결과적으로 개발자는 DB 종류에 관계없이 **의미가 일관된 런타임 예외**를 다룰 수 있다. 이것이 바로 “예외 전환(Exception Translation)”의 출발점이다.

## 4.5 의미 없는 복구 대신, 의미 있는 전환

스프링의 철학은 이렇게 요약된다.

> 복구 불가능한 예외는 런타임으로 단순화하고, 복구 가능한 예외만 의미 있게 전환하라.
> 

이는 단순히 `throws`를 없애자는 이야기가 아니다. 오히려 **“복구”라는 단어의 의미를 재정의한 선언**이다. 스프링은 예외를 잡아서 “복구”하지 않는다. 대신 “해석”한다. 즉, 단순한 오류를 *의미 있는 예외로 번역*해 개발자가 상황을 이해할 수 있게 한다.

이 변화는 예외 처리의 중심축을 바꿔놓았다. 언어가 예외를 “강제”하던 시대에서, 프레임워크가 예외를 “해석”하는 시대로 옮겨간 것이다.

# 5. 복구 가능성으로 재정의된 예외 체계

스프링은 자바의 Checked Exception 철학을 부정하지 않았다. 다만 그것을 **현실에 맞게 재해석**했다. 핵심 질문은 단 하나였다.

> “이 예외는 복구 가능한가?”
> 

자바는 이 질문에 대한 판단을 **컴파일러**에 맡겼지만, 스프링은 그 책임을 **프레임워크 구조**로 옮겼다. 즉, Checked Exception의 형식적 강제는 사라졌지만 그 철학 — “복구 가능한 실패만 명시하라” — 는 여전히 살아 있다. 다만 이제는 코드의 표면이 아니라, **예외의 계층 구조**로 표현될 뿐이다.

## 5.1 Checked Exception을 대체한 새로운 기준

스프링은 더 이상 예외를 Checked / Unchecked로 나누지 않는다. 대신 **“복구 가능성(recoverability)”** 을 기준으로 예외를 세밀하게 구분한다. 이 구조는 `org.springframework.dao` 패키지의 예외 계층에 잘 드러난다. 모든 데이터 접근 예외의 루트 클래스는 `DataAccessException`이며, 그 아래는 복구 가능성의 정도에 따라 세 방향으로 갈라진다.

| 구분 | 루트 클래스 | 복구 조건 | 의미 |
| --- | --- | --- | --- |
| **TransientDataAccessException** | 즉시 재시도하면 성공 가능 | 애플리케이션 개입 없이 재시도만으로 복구 | 네트워크 지연, 락 경합 |
| **RecoverableDataAccessException** | 복구 절차 수행 후 재시도 가능 | 커넥션 재획득, 트랜잭션 재시작 등 수동 복구 필요 | 연결 재설정, 세션 재생성 |
| **NonTransientDataAccessException** | 원인 수정 없이는 실패 | 데이터나 코드 교정 없이는 재시도로 불가 | 무결성 위반, 중복 키, 제약 조건 오류 |

이 세 가지는 단순한 기술적 분류가 아니다. 자바가 “복구 가능한 실패”를 컴파일 타임에 강제했다면, 스프링은 그것을 **런타임 구조 안에서 표현한 것**이다. 즉, “잡을 수 있느냐”가 아니라 “복구할 수 있느냐”가 예외 설계의 기준이 된다.

## 5.2 SQLException: 복구 불가능한 Checked Exception의 대표 사례

자바에서 가장 유명한 Checked Exception은 `SQLException`이다. JDBC API는 이를 “복구 가능한 실패”로 보고 모든 데이터 접근 코드에 `throws SQLException`을 강제했다. 하지만 현실에서 대부분의 SQL 예외는 복구 불가능했다. 네트워크 단절, 커넥션 타임아웃, 트랜잭션 충돌 같은 오류는 애플리케이션이 스스로 해결할 수 있는 성질이 아니었다. 그럼에도 컴파일러는 “catch하거나 던져라”를 요구했다. 개발자는 어쩔 수 없이 이런 코드를 작성했다.

```java
try {
    repository.save(user);
} catch (SQLException e) {
    throw new RuntimeException(e);
}
```

복구는 없고, 단지 형식만 남았다. 스프링은 바로 이 모순에서 출발했다. **복구 불가능한 예외를 강제로 다루게 하지 않겠다.** 대신 “의미를 해석하고 전달할 수 있도록” 구조를 바꾸기로 했다. 그 첫 결과물이 바로 `DataAccessException` 계층이었다.

## 5.3 복구 가능성의 계층화

스프링의 예외 계층은 단순히 편리함을 위한 구조가 아니다. 이는 Checked Exception의 철학을 더 현실적으로 구현한 구조적 실험이었다.

1. **복구 가능한 실패는 구체적으로 표현한다.** 락 경합, Deadlock, 일시적 연결 오류는 `TransientDataAccessException`으로 분류되어 재시도 정책(`@Retryable`, CircuitBreaker 등)과 자연스럽게 연계된다.
2. **복구 불가능한 실패는 단순히 전파한다.** 데이터 무결성 위반, 중복 키, 제약 조건 오류는 `NonTransientDataAccessException`으로 처리되어 원인 수정 없이는 다시 시도하지 않음을 명확히 한다.
3. **일부 복구 가능한 상황은 별도로 분리한다.** 세션 손실이나 커넥션 만료처럼 복구 절차가 필요한 상황은 `RecoverableDataAccessException`으로 구분되어 “닫고 다시 시도”라는 회복 절차를 유도한다.

이렇게 스프링은 “예외를 강제로 잡게 하는 언어” 대신 “예외를 해석하게 하는 프레임워크”로 진화했다.

## 5.4 트랜잭션에서의 복구 가능성 적용

트랜잭션 경계는 스프링이 Checked Exception의 철학을 가장 현실적으로 재해석한 영역이다. 스프링은 트랜잭션 실패를 다루며 다음 두 가지 원칙을 세웠다. 첫째, 인프라 수준의 오류는 복구 불가능하다. 둘째, 복구 가능성에 따라 롤백 여부를 결정한다.

스프링 공식 문서는 먼저 트랜잭션 인프라 예외의 성격을 이렇게 정의한다.

> “Again in keeping with Spring’s philosophy, the TransactionException that can be thrown by any of the PlatformTransactionManager interface’s methods is unchecked (that is, it extends java.lang.RuntimeException). Transaction infrastructure failures are almost invariably fatal.”
> 
> 
> — *Spring Framework Reference, §10.3 Understanding the Spring Framework transaction abstraction*
> 

즉, 트랜잭션 인프라 오류는 복구 불가능(fatal)한 상황으로 간주된다. 따라서 `PlatformTransactionManager`가 던지는 모든 예외는 `TransactionException`의 하위 타입이며, **모두 `RuntimeException`** 으로 정의되어 있다. 이는 “복구 불가능한 실패는 강제하지 않는다”는 스프링의 예외 철학을 트랜잭션 계층에서도 일관되게 유지한 결과다.

다음으로, 스프링의 트랜잭션 롤백 정책은 복구 가능성(recoverability)을 기준으로 동작한다.

> “Any RuntimeException triggers rollback, and any checked Exception does not.”
> 
> 
> — *Spring Framework Reference, §10.5.3 Rolling back a declarative transaction*
> 

즉, **RuntimeException 또는 Error** 가 발생하면 트랜잭션은 **롤백(rollback)** 되고, **Checked Exception** 이 발생하면 트랜잭션은 **커밋(commit)** 을 유지한다. 이 규칙은 단순한 편의가 아니라, Checked Exception의 본질적 전제 — “복구 가능한 실패는 직접 처리하라” — 를 트랜잭션 단위의 정책으로 재구성한 것이다. Checked Exception은 복구 가능하다는 가정 아래에서 롤백 대신 커밋의 기회를 남기며, RuntimeException은 복구 불가능하므로 즉시 롤백된다.

필요하다면 예외의 복구 가능성을 직접 정의할 수도 있다. 예를 들어 Checked Exception이라도 도메인 규칙상 복구 불가능하다면 `rollbackFor` 속성을 통해 명시적으로 롤백 대상으로 지정할 수 있다.

```java
@Transactional(rollbackFor = IOException.class)
public void process() throws IOException {
    // Checked Exception도 롤백 처리
}
```

결국 스프링의 트랜잭션 모델은 Checked Exception의 철학을 “언어의 강제” 대신 **“프레임워크의 정책”으로 계승**한 구조다. “이 실패는 복구 가능한가?” 바로 이 질문이 트랜잭션의 커밋과 롤백을 결정하는 기준이 된다.

---

## 5.5 Checked Exception 철학의 계승

결국 스프링은 Checked Exception의 이상을 버린 것이 아니라, 그 철학을 **형식에서 의미로 옮겼다.** 즉, 자바가 언어로 강제했던 것을 스프링은 프레임워크의 구조와 규칙으로 자연스럽게 흘려보냈다. Checked Exception의 불편한 강제는 사라졌지만, 그 정신 — *복구 가능한 실패를 명시하라* — 는 여전하다.

# 6. 예외 전환(Exception Translation): 의미를 잃지 않게 던지기

복구 가능성의 계층을 세운 스프링은 여기서 한 걸음 더 나아갔다. 

> “복구할 수 없는 실패라면, 그래도 그 의미는 남길 수 있지 않을까?”
> 

스프링의 예외 전환(Exception Translation)은 바로 이 질문에서 시작된다. 스프링은 Checked Exception을 단순히 없앤 것이 아니라, **복구 불가능한 예외를 의미 있게 번역하는 구조**로 바꾸었다.

## 6.1 Checked Exception 이후의 과제

스프링은 복구 불가능한 예외를 런타임으로 단순화했지만, 단순히 “던지고 끝내는” 방식을 택하지 않았다. 오히려 그 실패가 *무엇이, 어디서, 왜* 발생했는지를 전달할 수 있도록 **의미의 보존**을 새로운 과제로 삼았다. 자바의 Checked Exception이 “복구 가능한 실패를 명시하라”였다면, 스프링의 예외 전환은 “복구 불가능한 실패라도 맥락을 잃지 않게 하라”였다. 이때 스프링은 “전환(translation)”이라는 용어를 쓴다. 이는 단순한 wrapping이 아니라, “낯선 세계의 예외를 의미 있는 언어로 번역하는 행위”를 뜻한다.

## 6.2 예외 전환의 의도: 기술 종속성을 끊다

스프링의 공식 문서는 이렇게 말한다.

> “Spring provides a convenient translation from technology specific exceptions like SQLException to its own exception hierarchy with the DataAccessException as the root exception. These exceptions wrap the original exception so there is never any risk that you would lose any information as to what might have gone wrong.” — Spring Framework Reference, §10.2 Consistent Exception Hierarchy
> 

즉, 스프링은 데이터 접근 기술마다 달랐던 Checked Exception 체계를 모두 걷어내고 **DataAccessException**을 루트로 한 **일관된 런타임 예외 계층**을 도입했다. 이 계층은 단순히 `SQLException`을 감싸는 게 아니라, 그 원인을 “어떤 종류의 실패인지”로 해석해 의미를 부여한다. 예외는 감춰지지 않고, 오히려 더 읽기 쉬운 언어로 번역된다. 예를 들어 JDBC, Hibernate, JDO 등 각각의 기술은 고유한 Checked Exception을 던진다. 기존에는 개발자가 이를 직접 처리해야 했다. 그러나 스프링은 각 기술의 예외를 **하나의 추상 계층으로 통합**한다.

- JDBC → `SQLException`
- Hibernate → `HibernateException`
- JDO → `JDOException`

이 세 가지는 모두 `DataAccessException` 계층으로 번역되어, 개발자는 기술에 상관없이 **동일한 방식으로 실패를 이해하고 대응**할 수 있다.

## 6.3 일관된 예외 계층: 실패의 언어를 통일하다

공식 문서에 따르면, 스프링은 다음과 같은 일관된 모델을 제공한다.

> “This allows you to handle most persistence exceptions, which are non-recoverable, only in the appropriate layers, without annoying boilerplate catches/throws, and exception declarations.” — Spring Framework Reference, §10.2
> 

대부분의 데이터 접근 예외는 복구 불가능(non-recoverable)하다. 따라서 스프링은 그 예외들을 Checked로 강제하지 않는다. 대신 **의미를 유지한 채로 런타임 예외로 변환**한다.

```java
try {
    ps.executeUpdate();
} catch (SQLException e) {
    throw getExceptionTranslator().translate("update", sql, e);
}
```

`SQLErrorCodeSQLExceptionTranslator`는 데이터베이스별 SQL 오류 코드를 해석해 다음과 같은 스프링 표준 예외로 변환한다.

| SQL 오류 코드 | 변환된 예외 | 설명 |
| --- | --- | --- |
| ORA-00001 / 1062 | `DuplicateKeyException` | 중복 키 위반 |
| 23503 | `DataIntegrityViolationException` | 참조 무결성 위반 |
| 08S01 | `DataAccessResourceFailureException` | 연결 실패 |

결과적으로 개발자는 “무엇이 실패했는가”만 이해하면 된다. 기술마다 다른 예외 타입을 구분할 필요가 없다. 이 일관된 예외 계층 덕분에 스프링의 DAO 계층은 JDBC, Hibernate, JDO를 모두 같은 프로그래밍 모델로 다룰 수 있게 되었다.

## 6.4 예외 전환의 핵심: 의미를 보존하는 감싸기

스프링은 Checked Exception을 없앴지만 **의미 없는 단순 포장(wrapper)** 을 만든 게 아니다. 오히려 “의미를 보존한 감싸기”를 했다. 모든 `DataAccessException`은 내부에 원래 예외를 포함하고 있으며, 필요하다면 `getRootCause()`로 정확한 근본 원인을 확인할 수 있다. 이 덕분에 스택트레이스의 정보 손실이 없고, 기술별 세부 원인도 그대로 추적할 수 있다. 즉, 예외 전환은 **catch를 없애되 의미를 잃지 않는 구조**다. 기술 예외를 “추상화된 도메인 언어”로 번역해, 복구는 불가능하더라도 **맥락(context)** 은 남기도록 설계되어 있다.

## 6.5 프레임워크로서의 전환: 복구 대신 해석

스프링은 더 이상 “예외를 잡아서 처리”하지 않는다. 대신 “이해할 수 있는 형태로 번역”해 전달한다. 자바의 Checked Exception은 **catch를 강제**했다. 스프링의 Exception Translation은 **이해를 유도**한다. 이 차이는 단순한 문법의 변화가 아니라, 예외의 존재 이유에 대한 철학적 전환이다. 예외는 프로그램을 멈추기 위한 장치가 아니라 **의미를 전달하는 매개체**다. 스프링은 이를 언어 수준에서가 아니라 **프레임워크 수준의 일관된 정책**으로 구현했다. 결국 스프링은 자바가 Checked Exception으로 시도했던 실험을 “의미 중심의 런타임 구조”로 재탄생시켰다. 복구할 수 없는 실패라도, 그 이유와 맥락은 끝까지 남겨야 한다. 이것이 바로 스프링이 말하는 “예외 전환”의 철학이다.

# 7. 웹에서도 자바답게: 예외 전환의 재해석

스프링의 예외 철학은 단지 내부 코드에 머물지 않는다. 그 철학은 컨트롤러, 응답, 그리고 HTTP의 세계로 확장된다. 왜냐하면 웹의 실패 또한 프로그램의 실패이기 때문이다. 웹이라는 환경은 예외를 처리하는 또 하나의 층을 만든다. 여기서는 예외가 곧 **응답(Response)** 으로 변환되어야 한다. 즉, 예외는 더 이상 단순한 객체가 아니라 클라이언트에게 전달되는 **의미 있는 메시지**가 된다.

## 7.1 웹 환경에서 예외는 다른 이름의 응답이다

스프링 MVC에서 컨트롤러는 메서드의 결과를 HTTP 응답으로 직렬화한다. 그런데 예외가 발생하면 정상적인 반환 흐름이 깨진다. 이때 프레임워크는 예외를 “에러 응답”이라는 새로운 표현으로 바꿔야 한다. 즉, **예외 전환(Exception Translation)** 은 이제 **HTTP 응답 변환(Response Translation)** 으로 확장된다.

| 계층 | 예외 전환 대상 | 변환 결과 |
| --- | --- | --- |
| 데이터 계층 | SQLException → DataAccessException | 런타임 예외 |
| 웹 계층 | Exception → HTTP Response | 상태 코드 + 메시지 |

이 표는 단순하지만, 철학적으로는 중요한 전환을 보여준다. 스프링은 여전히 같은 원칙을 따른다. “복구 불가능한 예외는 숨기고, 복구 가능한 실패는 의미 있게 드러낸다.”

## 7.2 “HTTP라서 다르다”는 착각

웹 개발자들은 흔히 이렇게 생각한다. “웹은 HTTP 규약이 있으니까 예외 처리는 그냥 상태 코드(400, 500)로 나누면 돼.” 하지만 그것은 **표현만 바뀐 동일한 문제**다. 예외를 400과 500으로 나누는 건 표면적인 분류일 뿐이다. 더 중요한 건 “왜 실패했는가, 복구 가능성은 있는가”다.

예를 들어 이런 두 상황을 비교해보자.

| 상황 | 복구 가능성 | HTTP 상태 |
| --- | --- | --- |
| 사용자가 존재하지 않음 | 가능 (입력 수정으로 해결) | 404 Not Found |
| 데이터베이스 연결 끊김 | 불가능 (서버 내부 문제) | 500 Internal Server Error |

두 예외는 모두 실패이지만, 하나는 사용자의 조치로 해결될 수 있고, 다른 하나는 시스템이 복구해야 한다. 스프링은 바로 이 “복구 가능성”을 기준으로 예외를 분리한다. 웹의 예외 처리 또한 Checked vs Unchecked의 개념을 그대로 따른다. 단지 결과물이 “컴파일 오류” 대신 “HTTP 응답”일 뿐이다.

## 7.3 스프링 MVC의 예외 전환 구조

스프링 MVC는 `DispatcherServlet`을 중심으로 예외를 처리한다. 컨트롤러에서 예외가 던져지면 다음 흐름을 따른다.

1. `DispatcherServlet`이 예외를 감지한다.
2. `HandlerExceptionResolver` 목록을 순서대로 탐색한다.
3. 해당 예외를 처리할 수 있는 Resolver가 응답을 생성한다.
4. 생성된 `ModelAndView`나 `ResponseEntity`가 HTTP로 변환된다.

스프링의 기본 Resolver는 다음과 같다.

| 클래스 | 역할 |
| --- | --- |
| `ExceptionHandlerExceptionResolver` | `@ExceptionHandler` 기반 처리 |
| `ResponseStatusExceptionResolver` | 예외 클래스에 `@ResponseStatus`가 붙은 경우 처리 |
| `DefaultHandlerExceptionResolver` | 스프링 내부 표준 예외 처리 (`HttpRequestMethodNotSupportedException` 등) |

이 체계 덕분에 스프링은 예외를 “자동으로” 응답 형태로 번역한다. 개발자는 예외의 의미만 정의하면 된다.

## 7.4 예외 전환의 실전 형태

스프링에서 예외 전환은 다음처럼 표현된다.

```java
@ResponseStatus(HttpStatus.NOT_FOUND)
public class EventNotFoundException extends RuntimeException {
    public EventNotFoundException(Long eventId) {
        super("이벤트를 찾을 수 없습니다: " + eventId);
    }
}
```

컨트롤러에서 이 예외가 발생하면, 스프링은 자동으로 404 응답을 반환한다.

```
HTTP/1.1 404 Not Found
Content-Type: application/json

{
  "error": "이벤트를 찾을 수 없습니다: 1"
}
```

이는 Checked Exception을 강제하는 대신 “의미 있는 예외를 명시적으로 선언하는 선택”을 가능하게 만든다. 자바 언어에서 `throws`를 없애면서도, HTTP 세계에서는 여전히 “의미를 강제”하는 셈이다.

## 7.5 예외 전환의 확장: ProblemDetail과 RFC 9457

스프링 6 이후, 스프링은 IETF 표준(RFC 9457)에 따라 `ProblemDetail` 기반의 예외 응답 형식을 지원하기 시작했다. 이는 예외의 의미를 단순한 메시지 문자열이 아니라, **구조화된 데이터로 전달**하려는 시도다.

```json
{
  "type": "https://example.com/errors/event-not-found",
  "title": "Event not found",
  "status": 404,
  "detail": "이벤트 ID 1을 찾을 수 없습니다.",
  "instance": "/api/events/1"
}
```

이 포맷은 단순히 “에러가 났다”가 아니라, “무엇이, 왜, 어디서 실패했는가”를 명확히 보여준다. 즉, 자바의 Checked Exception이 하던 “복구 가능한 실패의 명시”를 이제 HTTP 수준에서 실현하고 있는 셈이다.

# 8. 예외 정의의 기준: 복구 가능성과 의미의 경계

스프링의 예외 설계는 단순히 “Checked를 버렸다”가 아니라, “복구 가능성과 의미의 경계를 명확히 그었다”는 선언이었다. 이 경계가 바로 예외 정의의 기준이 된다. 예외는 단순한 실패 신호가 아니다. 예외는 시스템이 스스로에게 던지는 질문이다. 

> “이 실패는 복구할 수 있는가?”, “그렇다면, 그 의미를 어떻게 전달할 것인가?”
> 

이 두 질문이 예외 설계의 출발점이자 끝이다.

## 8.1 예외의 첫 번째 기준: 복구 가능성

복구 가능한 실패란 “시스템이 혹은 클라이언트가 대응할 수 있는 실패”를 의미한다. 예를 들어 입력 검증 오류, 비즈니스 규칙 위반, 중복 요청 등은 복구 가능하다. 클라이언트가 입력을 수정하거나 요청을 재시도함으로써 해결할 수 있다. 이런 경우에는 반드시 **의미 있는 예외를 명시적으로 정의**해야 한다.

```java
public class DuplicateInvitationException extends BusinessException {
    public DuplicateInvitationException() {
        super("이미 초대된 구성원입니다.");
    }
}
```

이 예외는 단순한 오류가 아니라 “무엇을 잘못했는가”를 설명하는 도메인 메시지다. 스프링은 이런 예외를 `@ResponseStatus`나 `ProblemDetail`로 전환하여 클라이언트에게 명확히 전달한다. 복구 가능한 예외는 **사용자와 시스템의 대화**를 담당한다.

## 8.2 예외의 두 번째 기준: 의미의 필요성

복구 불가능한 실패라도, 의미를 남겨야 하는 경우가 있다. 데이터베이스 장애, 외부 API 실패, 메일 전송 오류 같은 예외가 그렇다. 이들은 복구할 수는 없지만, 원인을 이해해야 한다. 따라서 스프링은 이런 예외를 런타임으로 단순화하되, **의미를 보존하는 전환(Exception Translation)** 을 적용한다.

```java
try {
    emailSender.send(to, subject, body);
} catch (MailException e) {
    throw new EmailDeliveryException("이메일 전송 실패", e);
}
```

이 코드는 복구를 시도하지 않는다. 하지만 실패의 맥락(“어떤 이메일이, 왜 전송되지 않았는가”)은 남긴다. 즉, **복구할 수 없는 예외라도 의미를 잃지 않게 던진다.**

## 8.3 복구 가능성과 의미의 조합

예외를 정의할 때는 “복구 가능성”과 “의미 필요성”을 동시에 고려해야 한다.

| 복구 가능성 | 의미 필요성 | 예외 정의 방식 |
| --- | --- | --- |
| 높음 | 높음 | 커스텀 예외 + 명시적 전환 |
| 낮음 | 높음 | 커스텀 예외 + 명시적 전환 |
| 낮음 | 낮음 | 런타임 예외 그대로 전파 |
| 높음 | 낮음 | 거의 존재하지 않음 |

즉, 예외는 “복구 가능한가?”와 “의미를 남겨야 하는가?”라는 두 축 위에서 정의된다. 이 기준만 명확히 잡아도 불필요한 예외 정의나 무의미한 포장은 줄어든다.

## 8.4 예외를 정의할 때 스스로에게 물어야 할 질문

1. **이 실패는 복구 가능한가?**
    - 가능하다면 Checked 대신 명시적 커스텀 예외를 정의하라.
    - 불가능하다면 런타임 예외로 단순화하라.
2. **이 실패는 의미를 남겨야 하는가?**
    - 남겨야 한다면 원인을 감싸되 맥락을 유지하라.
    - 의미가 없다면 그대로 전파하라.
3. **이 예외는 어디까지 도달해야 하는가?**
    - 도메인 로직의 일부인가, 아니면 외부 시스템의 문제인가?
    - 경계를 넘는다면 반드시 “전환”하라.

이 세 가지 질문이 예외 정의의 최소 기준이다. 이를 일관되게 적용하면 코드 전체가 예외를 통해 “무엇이 실패했는가”를 자연스럽게 말하게 된다. 결국 예외란 실패의 신호가 아니라 **의미의 언어**이며, 스프링은 Checked Exception의 철학을 현실적인 기준과 구조로 재해석한 것이다.

우리도 이 원칙을 따르기만 하면 된다. 복구 가능한 예외는 명확히 드러내고, 복구 불가능한 예외는 의미 있게 전환하라. 그것이 “스프링답게, 그리고 자바답게” 실패를 다루는 방법이다.