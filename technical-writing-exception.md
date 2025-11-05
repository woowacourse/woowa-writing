
# Part 1 : Exception의 개념과 종류

## 1. 서론  

이 글은 우아한테크코스의 테크니컬 라이팅을 목적으로, 프로그래밍을 어느 정도 경험했지만 **예외(Exception)** 에 대한 이해가 부족하거나 궁금한 사람들을 위해 작성되었다.  
관련 코드는 **Java** 로 작성되었으며, 자바와 스프링 영역에서의 예외의 개념부터 시작하여 다양한 예외 처리 전략 까지 단계적으로 살펴본다.  

대부분의 사람들은 프로그래밍을 하다 보면 다음과 같은 문제 상황을 한 번쯤 경험하게 된다.

- 코드에 오타가 났다던가  
- 함수에 잘못된 값을 전달했다던가  
- 무한 루프를 만들고 실행했다던가  

이러한 문제들은 프로그램의 정상적인 실행을 방해한다.  
그렇다면 **프로그래머들은 이런 상황을 어떻게 해결할까?**  
이 질문에 대한 답이 바로 **예외 처리(Exception Handling)** 이다.  

---

## 2. 예외와 에러의 차이  

예외(Exception)를 이해하기 위해서는 먼저 **에러(Error)** 와의 차이를 명확히 구분해야 한다. 결론부터 말하자면 차이점은 아래와 같다.
> **에러(Error)** : 시스템의 고장으로 인한 복구 **불가능한** 문제
**예외(Exception)** : 프로그램 실행 중 발생하는 복구 **가능한** 문제

> 추가적으로 오류와 에러를 혼동하시는 분들이 있는데 프로그래밍 세계에서 일반적으로 **‘에러’와 ‘오류’는 같은 의미**로 사용된다.  
즉, 일상적으로 “오류가 났다”라고 말하는 것은 대부분 **시스템 수준의 에러(Error)** 를 의미한다.


### 2-1. 자바의 예외 계층 구조


**자바**에서는 에러와 예외를 Throwable 클래스를 최상위로 두고 시스템 수준의 에러는 Error,
프로그램 실행 중 발생하는 문제는 Exception으로 나누어 관리하고있다.

```
Throwable
├── Error
└── Exception
     ├── RuntimeException (Unchecked)
     └── Checked Exception
```


다음으로는 에러와 예외에 대한 개념을 알아보자.


### 2-2. 에러(Error)란 무엇인가  

**오류(Error)** 는 **프로그램이 제어할 수 없는 심각한 문제 상황**을 뜻한다.  
주로 **시스템 자원 부족**, **가상머신(VM)의 비정상 동작**, **하드웨어 장애** 등의 원인으로 발생하며, 다음과 같이 세 가지 유형으로 나눌 수 있다.  



#### 🔹 컴파일 에러 (Compile-time Error)  

코드의 문법이나 구조가 언어의 규칙(Java, Python 등)에 맞지 않아 **프로그램이 실행조차 되지 않는 오류**이다.  
이러한 에러는 **컴파일러나 IDE에서 자동으로 탐지**되며, 메시지를 통해 비교적 쉽게 해결할 수 있다.  

**예시:**  
- 문법 에러(Syntax Error)  
- 타입 불일치(Type Error)  

---

#### 🔹 런타임 에러 (Runtime Error)  

프로그램 실행 도중 발생하는 예외적 상황으로, **프로그램이 중단되거나 비정상 동작**하게 된다.  
이번 글의 주제인 **예외(Exception)** 는 바로 이 **런타임 시점에서 발생**한다.  
이러한 에러는 **로그 분석**이나 **스택 트레이스(Stack Trace)** 를 통해 원인을 파악할 수 있으며, **예외 처리 전략**으로 대응이 가능하다.  

**예시:**  
- 0으로 나누기  
- Null 참조  
- 파일 없음  

---

#### 🔹 논리 에러 (Logical Error)  

문법적으로나 실행적으로는 문제가 없지만 **프로그램의 결과가 의도와 다르게 나오는 오류**이다.  
예를 들어, **잔고가 0원인데 결제가 되는 상황**을 생각할 수 있다.  
이런 오류는 **테스트와 디버깅**을 통해서만 탐지할 수 있다.  

**예시:**  
- 잘못된 알고리즘  
- 조건식 오류  
- 계산 실수  

---

### 2-3. 예외(Exception)란 무엇인가  

사전적으로 **예외**는 어떤 일에서 제외되거나 특별한 경우를 의미하는 단어이다. 그러나 프로그래밍 세계에서의 예외(Exception)는 다른 의미를 가진다.

프로그래밍에서의 예외는 **프로그램 실행 중(런타임)** 발생하는 오류 상황을 말한다. **예외 처리는** 이러한 오류가 프로그램의 비정상 종료로 이어지지 않도록 오류를 감지하고, 이에 대한 적절한 대응 로직을 수행하는 과정이다.
즉, 예외는 “**복구 가능한 오류**”이며, 코드 레벨에서 처리할 수 있는 문제이다.

#### 예외 처리의 예시  

예를 들어, 파일을 열 때 해당 파일이 존재하지 않는다면 예외가 발생한다.  
이 상황에서 예외를 처리하지 않으면 프로그램이 즉시 종료된다.  
하지만 예외 처리를 통해 다음과 같이 대응할 수 있다.

```java
try {
    FileReader reader = new FileReader("data.txt");
} catch (FileNotFoundException e) {
    System.out.println("파일이 없습니다. 기본 설정으로 진행합니다.");
}
```

---

위의 설명으로 사전 설명은 끝났다고 생각한다. 이제 본격적으로 예외에 대해 알아보자.

## Exception 클래스 

자바 프로그래밍을 경험한 사람이라면 `Exception` 클래스를 한 번쯤은 마주쳤을 것이다.  
`Exception` 클래스가 **어떻게 구성되어 있는지**, 그리고 **어떤 역할을 하는지**를 자세히 살펴보자.

💡[**Exception.class docs**](https://docs.oracle.com/javase/8/docs/api/java/lang/Exception.html) 공식문서를 보는 것도 추천한다.

### 1. Exception 클래스의 정의

`Exception`은 `java.lang` 패키지에 포함된 클래스이며,`Throwable` 클래스를 상속 받고 있고, 자바의 모든 **체크 예외(Checked Exception)** 의 기본 클래스이다.

```java
package java.lang;

public class Exception extends Throwable {
    ...
}
```

### 2. Exception 클래스의 주요 생성자

Exception 클래스는 다양한 상황에서 예외를 생성할 수 있도록 여러 생성자 오버로딩을 제공한다.


| 생성자                                                                                                 | 설명                                        |
| --------------------------------------------------------------------------------------------------- | ----------------------------------------- |
| `Exception()`                                                                                       | 기본 생성자. 메시지와 원인 없음                        |
| `Exception(String message)`                                                                         | 예외 메시지 지정                                 |
| `Exception(String message, Throwable cause)`                                                        | 메시지와 원인 예외를 함께 지정                         |
| `Exception(Throwable cause)`                                                                        | 원인 예외만 지정 (메시지는 `cause.toString()`으로 설정됨) |
| `Exception(String message, Throwable cause, boolean enableSuppression, boolean writableStackTrace)` | 예외 억제 및 스택 트레이스 기록 여부 제어 (Java 7+)        |


### 3. Checked 예외 ,Unchecked 예외

위에서 **Checked 예외**를 언급했다. 모두가 예상했겠지만 **Unchekced 예외** 또한 존재하고, 두 예외의 차이점을 알아보자.

#### 3-1. checked 예외
- 컴파일러가 명시적 처리(try-catch 또는 throws 선언) 를 강제하는 예외이다.

- Exception의 하위 클래스 중 RuntimeException을 상속하지 않은 것들이 이에 해당한다.
- 예시) IOException, SQLException, FileNotFoundException

```java
public void readFile() throws IOException {
    FileReader reader = new FileReader("data.txt"); // FileReader 생성 시 IOException 발생 가능
    // 반드시 예외 선언 필요: checked 예외는 메서드에서 직접 처리하거나 선언해야 함
}
```
위 코드에서 FileReader 생성 시, 생성자의 매개변수로 지정된 "data.txt" 파일이 존재하지 않거나 경로가 잘못되었다면
`IOException`(checked 예외)이 발생한다. 이러한 경우를 대비하기 위해 checked 예외는 메서드 내부에서 반드시 처리되어야 한다.  
처리 방법은 두 가지가 있다:

1. 메서드 내부에서 try-catch 블록으로 직접 처리
2. 메서드 선언부에 `throws IOException`을 선언하여 상위 호출자에게 전달

이 선언이나 처리가 이루어지지 않으면 **컴파일 오류**가 발생한다.  
즉, Checked 예외는 코드 어디선가 반드시 해결해야 하며, 처리하지 않고 넘어갈 수 없다.

#### 3-1. Unchecked 예외
- RuntimeException을 상속하는 예외이다.

- 명시적 예외 처리 없이 발생 가능하다 (컴파일러가 강제하지 않음)

- 주로 프로그래밍 오류로 발생한다.

- 예시) NullPointerException, IllegalArgumentException, ArrayIndexOutOfBoundsException

```java
public void divide(int a, int b) {
    // b가 0이면 ArithmeticException 발생 (Unchecked 예외)
    int result = a / b;
    System.out.println("결과: " + result);
}
```

위 코드에서 정수를 0으로 나누면 `ArithmeticException`이 발생한다.  
하지만 예외가 발생해도 애플리케이션이 자동으로 종료되지는 않으며, 컴파일러도 이를 체크하지 않기 때문에 **try-catch 선언이 필수는 아니다**.  

즉, Unchecked 예외는 발생 가능성이 있어도 **처리해야 한다는 강제성은 없다**.  
다만 필요하다면 try-catch로 처리할 수 있으며, 코드 설계 시 예외가 발생하지 않도록 **사전에 오류를 방지하는 것이 권장**된다.


#### Checked Exception과 Unchecked Exception 비교표

| 구분 | Checked Exception | Unchecked Exception |
|------|-------------------|---------------------|
| **상속 관계** | Exception 상속 (RuntimeException 제외) | RuntimeException 상속 |
| **컴파일러 체크** | 강제 (try-catch 또는 throws 필수) | 선택 (명시하지 않아도 컴파일 가능) |
| **발생 원인** | 외부 환경 문제 (파일, 네트워크, DB) | 프로그래밍 오류 |
| **대표 예시** | IOException, SQLException | NullPointerException, IllegalArgumentException |
| **처리 권장** | 반드시 처리 | 사전 검증으로 예방 |

#### ❓왜 예외를 두가지 상황으로 나누었을까?
그에 대한 해답은 자바의 아버지라 불리는 James Gosling 과의 대화에서 알 수 있다.

💡[_**제임스 고슬링과의 인터뷰**_](https://www.artima.com/articles/failure-and-exceptions?utm_source=chatgpt.com)

James Gosling이 체크 예외 설계 의도를 설명한 인터뷰가 있으며, 그 중 일부가 다음과 같다:

> “In Java you can ignore exceptions, but you have to willfully do it. You can’t accidentally say, ‘I don’t care.’ You have to explicitly say, ‘I don’t care.’”
— Artima, “Failure and Exceptions — A Conversation with James Gosling, Part II” 
artima.com


이 발언은 “예외 처리를 무시하려면 **명시적으로** 그렇게 해야 한다”는 의미다.  
즉, 자바는 **컴파일러 수준에서 예외 처리를 강제**함으로써, 개발자가 단순히 “귀찮아서” 예외를 무시하는 일을 방지한다.   
이것이 바로 Gosling이 추구한 **안정성과 명시성 중심의 설계 철학**이다.

또한, 같은 인터뷰에서 Gosling은 C 언어에서 오류 코드를 무시하는 관습을 지적하면서 Java가 예외 처리를 강제한 이유를 설명한다. 

결국 자바는 이러한 철학을 기반으로,예외를 두 가지로 구분했다.
**컴파일러가 강제하는 “체크 예외(Checked Exception)”** 와  
**개발자 선택에 맡기는 “언체크 예외(Unchecked Exception)”** 로 말이다.



# Part 2 : throws 와 try-catch
이번에는 `throws`와 `try-catch` 구문을 중심으로 각각의 역할과 사용 시점을 설명할 것이다.

## 1. Throws
Java에서 메서드가 호출 중에 예외를 발생시킬 가능성이 있는 경우 그 예외를 메서드 선언부에 명시하는 것이 바로 `throws` 키워드다.

특히 Checked 예외는 컴파일러가 예외 처리를 **강제**하기 때문에 메서드 내부에서 직접 처리하지 않고 상위 호출자에게 전달하려면 반드시 throws를 사용해야 한다.

```java
public void readFile(String path) throws IOException {
    FileReader reader = new FileReader(path);
    // 파일 읽기 로직
}
```
위 코드에서 readFile 메서드는 `IOException`을 처리하지 않고 호출자에게 전달한다.
따라서 호출하는 쪽에서 `try-catch` 로 예외를 처리하거나 다시 `throws`로 던져야 한다.

> 💡개발자는 throws로 던져진 예외를 어느 `계층`에서 처리할지 명확히 결정해야 한다.
예를 들어 Controller–Service–Repository 구조에서는, Repository에서 발생한 예외를 어디까지 전파할지가 중요한 설계 포인트다.
이에 대한 구체적인 논의는 다음 글에서 다루겠다.


---

## 2. try-catch
예외가 발생하면 기본적으로 try-catch를 사용하여 예외를 처리할 수 있다. 
```java
try{
	//예외가 발생할 가능성이 있는 로직
}catch(Excetpion e){
	//예외가 발생시 실행할 로직
}
```
기본적인 형태는 위와 같이 `try` 구간과 `catch` 구간으로 나뉘어져 있다. 

### 2-1. try
- 예외가 발생할 가능성이 있는 로직이 위치한다.

### 2-2. catch
- 예외가 발생하면 catch 블록이 실행된다.  
- catch()의 파라미터에는 처리할 **예외 클래스를 지정**한다.  
- 예를 들어 `catch(IOException e)`는 IO 관련 예외만 처리한다.  

#### 주의점
- `catch(Exception e)`는 모든 예외를 포괄하기 때문에 예외별 맞춤 처리 불가능하다. 
- 일반적으로 **구체적 예외 → 상위 예외 순서**로 catch를 작성하는 것이 좋다.

#### 예제
```java
try {
    FileReader reader = new FileReader("data.txt");
} catch (FileNotFoundException e) {
    System.out.println("파일이 존재하지 않습니다: " + e.getMessage());
} catch (IOException e) {
    System.out.println("입출력 오류 발생: " + e.getMessage());
} catch(Exception e){
    System.out.println("Exception 예외까지 도달: " +  e.getMessage());
}
```
위의 코드를 보자. 
1. 'data.txt' 파일이 없으면 → FileNotFoundException 발생
2. 파일 읽는 도중 다른 IO 문제가 생기면 → IOException 발생
3. 위 두 경우에 해당하지 않는 기타 문제 발생 시 → Exception 발생

---
## 3. multi-catch

만약 하나의 catch 문에 `복수`의 예외를 잡고 싶다면 아래와 같이 `multi-catch`로 구현할 수 있다.(Java 7 이상에서 사용 가능)

- 처리 로직이 동일한 경우만 사용 가능

- multi-catch 내부에서 e 변수는 final 취급 → 재할당 불가
```java
try {
    // 파일 읽기 또는 DB 처리
} catch (IOException | SQLException e) {
    e.printStackTrace(); // 두 예외를 동일하게 처리
}
```

---

## 4. finally
만약 try-catch로 감싼 로직이 정상 동작하든, 예외가 발생하든 반드시 실행시키고 싶은 로직이 있다면 **`finally`** 를 이용하면 된다.
finally가 중요하게 쓰이는 대표적인 경우는 **파일, DB, 소켓** 등과 같은 자원 관리이다.
예외 발생 시 자원을 닫지 않으면 **메모리 누수 / 리소스 누수**가 발생할 가능성이 있기 때문이다.

```java
public void readFileWithoutFinally() {
    FileReader reader = null;
    try {
        reader = new FileReader("data.txt");
        int data = reader.read();
        // 파일 읽기 로직 수행
    } catch (IOException e) {
        e.printStackTrace();
    }
    // reader.close()가 예외 발생 시 호출되지 않아 자원 누수 가능
}
```
위 코드는 예외가 발생하면 `reader.close()`가 호출되지 않아 자원 누수 위험이 존재한다. 이를 보완한 코드를 살펴보자. 
예외 발생 여부와 관계없이 finally 블록 내부 코드는 **항상** 실행되기 때문에 자원 해제를 **강제**하여 메모리/리소스 누수를 방지한다.
<br>
<**finally로 보완한 안전한 코드**>
```java
public void readFileWithFinally() {
    FileReader reader = null;
    try {
        reader = new FileReader("data.txt");
        int data = reader.read();
        // 파일 읽기 로직 수행
    } catch (IOException e) {
        e.printStackTrace();
    } finally {
        try {
            if (reader != null) {
                reader.close(); // 예외 여부와 관계없이 항상 호출
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

그렇다면 개발자는 **파일, DB, 소켓**등을 사용할 때 매번 직접 finally를 붙여야 할까? 물론 메모리 / 리소스 누수를 방지하기 위해서 반드시 작성해야할 것이다. 
하지만 문제점이 있다. 그것은 개발자가 `finally`를 실수로 안붙이는 **휴먼에러**의 가능성이 존재한다.
이를 보완해줄 방법이 존재한다.

---
## 5. try-with-resources
Java 7 이상에서는 파일, DB, 소켓 등 자원을 안전하게 자동으로 해제하기 위해 **`try-with-resources`** 구문을 사용할 수 있다.
이 구문을 사용하면 finally 블록을 직접 작성하지 않아도 자원을 자동으로 닫을 수 있어 코드가 간결해지고 예외 안전성이 높아진다.

### 특징

1. **자동 자원 관리**
	- try 괄호 안에서 선언한 자원(AutoCloseable 구현 클래스)은 try 블록 종료 시 자동으로 close() 호출

2. **예외 발생 여부와 상관없이 실행**

	 - Checked/Unchecked 예외 모두 처리 가능

3. **코드가 간결해짐**

	- finally 블록에서 자원을 닫는 반복적인 코드를 제거


<br>

### 기본 문법
```java
try (자원 선언) {
    // 자원을 사용하는 로직
} catch (예외 e) {
    // 예외 처리
}
```

`try(자원 선언)` 의 괄호 안에는 FileReader, BufferedReader, Connection, PreparedStatement 과 같은 **AutoCloseable** 인터페이스를 구현한 객체를 선언할 수 있다.

또한 여러 자원도 `;`로 구분하여 동시에 선언 가능하다.
<br>
### 예제: JDBC 사용

아래 코드는 필자가 우테코 미션에서 구현한 **JdbcTemplate** 예제이다.
`try-with-resources`를 사용하여 Connection, PreparedStatement, ResultSet 객체를 자동으로 `close`하고 있다.

```java
    public <T>List<T> query(String sql, RowMapper<T> rowMapper) {
        try (Connection conn = dataSource.getConnection();
             PreparedStatement pstmt = conn.prepareStatement(sql);
             ResultSet rs = pstmt.executeQuery()) {

            log.debug("query : {}", sql);

            List<T> results = new ArrayList<>();
            while (rs.next()) {
                results.add(rowMapper.mapRow(rs));
            }
            return results;
        } catch (SQLException e) {
            log.error(e.getMessage(), e);
            throw new DataAccessException(e);
        }
    }
```

---

## Part 2 마무리

예외 처리는 코드의 품질을 결정짓는 중요한 요소다.
throws로 예외를 넘길지, try-catch로 직접 처리할지, 혹은 try-with-resources로 안전성을 확보할지는 모두 의도적인 설계 선택이다.

결국 예외 처리의 목표는 “**모든 예외를 잡는 것**”이 아니라, **올바른 위치**에서 책임 있게 다루는 것이다.

# Part 3 : 커스텀 예외

이번 글에서는 한 단계 더 나아가 **커스텀 예외(Custom Exception)를 가 무엇인지, 올바른 커스텀 예외 생성 방법에 대하여 설명한다다.**

---
## 커스텀 예외(Custom Exception)
예외 처리를 학습하다 보면 한 가지 의문이 생긴다.
> “표준 예외(IllegalArgumentException, IOException 등)만으로 충분할까?”
“우리 서비스의 도메인 로직을 반영한 의미 있는 예외는 어떻게 만들어야 할까?”

이 질문의 답을 찾는 과정에서 **커스텀 예외(Custom Exception)** 개념이 등장한다. 
<br>
### 커스텀 예외가 필요한 이유

표준 예외는 대부분 일반적인 오류 상황을 표현한다.
예를 들어 IllegalArgumentException은 잘못된 인자를 의미하지만 “사용자 포인트가 부족하다”와 같은 도메인 고유의 상황을 담기에는 한계가 있다.


즉, 비즈니스 맥락(Context)을 명확히 전달하기 어렵다.

**예시**

| 구분       | 표준 예외                      | 커스텀 예외                       |
| -------- | -------------------------- | ---------------------------- |
| 예외명      | `IllegalArgumentException` | `InsufficientPointException` |
| 의미       | 잘못된 인자                     | 포인트 부족으로 인한 결제 실패            |
| 맥락 반영 여부 | ❌ 일반적                      | ✅ 도메인에 특화됨                   |


이처럼 커스텀 예외를 통해 “어떤 문제인지”뿐만 아니라
“왜 발생했는지”를 코드 레벨에서 명확히 표현할 수 있다.
<br>

### 커스텀 예외 설계 기본 구조
커스텀 예외는 Throwable와 상속관계에 있는 클래스를 상속받아 사용한다. 일반적으로 RuntimeException 을 상속받아 정의한다.
이는 명시적인 try-catch 없이도 전파된다.

```java
pblic class ExternalApiException extends RuntimeException {

    public ExternalApiException(String message) {
        super(message);
    }
    
}
```

<br>

---

### 커스텀 예외 Best Practice

필요한 예외를 직접 정의하는 것은 좋은 접근이다.
하지만 만든 예외의 이름이 **"NoData", "GoodException", "BadSituation"** 과 같다면 어떨까?

이런 이름의 예외가 애플리케이션 실행 중 발생한다면 문제가 ‘무엇 때문인지’ 파악하기가 매우 어려울 것이다. 
예외 메시지를 보고도 원인을 유추할 수 없다면  그 예외는 사실상 **의미 없는 신호**에 불과하다.

단독으로 개발하는 상황이라면 그나마 감내할 수 있겠지만 프로젝트가 팀 단위로 진행되는 협업 환경이라면 이야기가 달라진다.
코드 리뷰나 디버깅 과정에서
“이 예외가 정확히 어떤 상황을 의미하는가?”라는 질문이 쏟아질 것이고 팀원들 사이에서 좋지 않은 인상을 줄 가능성이 높다.

즉, 예외 이름은 예외의 본질을 드러내야 한다.
“무엇이 잘못되었는가?”를 명확히 표현하지 못하는 예외는 오히려 문제 해결을 더디게 만든다.

이러한 상황을 방지하고 좋은 예외를 만드는 4가지 Best Practice를 살펴보자.

#### 1. Always Provide a Benefit 
> 항상 “이 예외를 통해 얻을 수 있는 정보”를 제공하자.

커스텀 예외는 단순히 오류를 알리는 용도가 아니다.
개발자에게 문제의 원인을 명확히 설명하고, 해결 방향을 제시해야 한다.

즉, “이 예외를 던짐으로써 팀원이 얻을 수 있는 **이점(benefit)**”이 있어야 한다.

아무런 이점도 제공할 수 없다면 `UnsupportedOperationException`  이나 `IllegalArgumentException` 과 같은 표준 예외 중 하나를 사용하는 것이 좋다 . 모든 Java 개발자는 이미 이러한 예외를 알고 있을 것이다. 이렇게 하면 코드와 API를 더 **쉽게** 이해할 수 있다.  

**예시**
```java
// ❌ 나쁜 예시
throw new NoDataException("데이터 없음");

// ✅ 좋은 예시
throw new UserNotFoundException("ID가 42인 사용자를 찾을 수 없습니다.");
```

좋은 예외는 에러 로그를 읽는 순간 무엇이, 왜, 어디서 발생했는지 즉시 이해할 수 있어야한다. 이는 **디버깅 속도**를 높이고  **장애**에 대응하는 시간을 단축시킨다.

<br>

#### 2. Follow the Naming Convention 
> 예외 클래스는 반드시 Exception으로 끝나야 한다.

명명 규칙을 통일하면 프로젝트 전체에서 예외를 **식별**하기 쉬워진다.
또한 IDE나 검색 도구를 사용할 때, `Exception` 접미사만으로 관련 클래스를 빠르게 찾을 수 있다.

실제로 Java에서 기본적으로 구현한 예외들을 살펴보면 모두 `Exception`으로 끝나는걸 확인할 수 있다. 

| ❌ 잘못된 이름       | ✅ 올바른 이름                     |
| -------------- | ---------------------------- |
| `UserNotFound` | `UserNotFoundException`      |
| `ApiError`     | `ExternalApiException`       |
| `PointLack`    | `InsufficientPointException` |

<br>

#### 3. Provide Javadoc Comments for Your Exception Class 
> 예외의 목적과 사용 의도를 명확히 문서화하자.

커스텀 예외 클래스는 **재사용**될 가능성이 높다.
따라서 예외가 언제 발생하는지, 어떤 상황에서 던져지는지를 Javadoc 주석으로 명확히 기술하는 것이 중요하다.
이를 통해 팀원이나 후속 개발자가 예외의 용도를 이해하기 쉽다.

실제로 모잇지 서비스를 구현하면서 커스텀 예외 문서화의 중요성을 크게 느꼈다.
프론트파트 개발시 간헐적 API 에러가 발생했는데 Swagger를 활용하여 다양한 커스텀 예외를 문서화하여 프론트엔드와 협업할 때 예외 발생 상황과 의미를 명확히 전달할 수 있었고 

그 결과 장애 대응을 빠르게 할 수 있었다.

```java
/**
 * 외부 API 호출 중 예기치 않은 오류가 발생했을 때 던져진다.
 * 주로 네트워크 지연, 응답 파싱 실패, 인증 오류 등의 상황에서 사용된다.
 */
public class ExternalApiException extends RuntimeException {
    public ExternalApiException(String message) {
        super(message);
    }
}
   
```

<br>

#### 4. Provider Constructor That Sets the Cause
> 원인 예외(cause)를 함께 전달하는 생성자를 제공하자.

커스텀 예외가 감싸는 `내부 예외`(예: IOException, SQLException)를 명시적으로 연결하면 전체 예외 흐름을 추적하기가 훨씬 쉬워진다.


```java
public class ExternalApiException extends RuntimeException {

    public ExternalApiException(String message) {
        super(message);
    }

    public ExternalApiException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

이 패턴을 사용하면 예외 스택 트레이스에 원인(cause) 이 함께 포함되어 로그 분석 및 장애 추적이 용이해진다.

#### 커스텀 예외 체크리스트 


| 항목 | 체크 |
|-----|------|
| 예외 이름이 Exception으로 끝나는가? | ☐ |
| 예외 이름이 발생 원인을 명확히 표현하는가? | ☐ |
| Javadoc 주석으로 예외의 목적을 설명했는가? | ☐ |
| 원인 예외를 전달하는 생성자를 제공하는가? | ☐ |
| 표준 예외로 충분한지 검토했는가? | ☐ |
| 비즈니스 맥락을 반영하고 있는가? | ☐ |

이 표는 커스텀 예외를 설계할 때 확인해야 할 체크리스트입니다. 각 항목을 검토하면서 좋은 커스텀 예외를 만들 수 있다.

---

## Part 3 마무리
커스텀 예외가 무엇인지, 어떻게 사용하는지 알아보았다. 
가장 중요한 것은 커스텀예외를 통해 **무엇을** 전달할 것인지 명확히 해야한다. 

예외 처리는 비용이 크다. 그렇기에 예외를 남발하기 보다는 꼭 필요한 부분에 구현하고 문제 상황에 따른 적절한 에외를 발생하는게 중요하다.

