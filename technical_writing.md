# Java의 매개변수 호출 방식(Pass by Value vs. Pass by Reference)

Java로 함수를 수행하다보면 예기치 못한 결과에 당황할 때가 있다.

아래 Java 예제를 보며 `main()` 함수를 실행했을 때 결과를 예측해보자.
```java
class Person {
    
    private String name;
    
    public Person(String name) {
        this.name = name;
    }
    
    public String getName() {
        return name;
    }
    
    public void setName(String name) {
        this.name = name;
    }
}


public class Main {
    
    public static void main(String[] args) {
        int x = 3;
        int[] array = {1, 2, 3};
        Person person1 = new Person("나");
        Person person2 = new Person("우리");
        
        foo(x, array, person1, person2);

        System.out.println("x: " + x);
        System.out.println("array[0]: " + array[0]);
        System.out.println("person1.name: " + person1.getName());
        System.out.println("person2.name: " + person2.getName());
    }
    
    private static void foo(int x, int[] array, Person person1, Person person2) {
        x++;
        array[0]++;
        person1 = new Person("바뀐 나");
        person2.setName("바뀐 우리");
    }
}
```
실행 결과는 흥미를 위해 아래 예제 분석에서 다루겠다.

Java를 예시로 들었지만, 흥미로운 사실은 개발 언어마다 실행 결과가 다르다는 것이다.
이는 개발 언어마다 함수의 매개변수 전달 방식이 다르기 때문이다.

그 중 가장 일반적으로 알려진 값에 의한 전달(Pass by Value)과 참조에 의한 전달(Pass by Reference)을 토대로 매개변수 전달 방식을 알아보자.

> [!note] 
> 매개변수 전달 방식은 값에 의한 방식과 참조에 의한 전달만 존재하는 것이 아니다. </br>
> 본 글에서는 가장 많이 알려진 값에 의한 전달(Pass by Value)과 참조에 의한 전달(Pass by Reference)에 대해서만 논한다. </br>
> 이름에 대한 전달(Call by Name), 공유에 의한 값 전달(Pass by Value Sharing), 이동 의미론(Move Semantics) 등 다양한 전달 방식이 있으니 궁금하다면 찾아보길 권장한다. </br>

## 용어 정리

### 값에 의한 전달(Pass by Value, Call by Value)
값에 의한 전달은 매개변수로 데이터의 복사본을 전달하는 방식이다.
그래서 함수 내부의 모든 수정 사항은 함수 외부로 영향을 미치지 않는다.

아래 예시 코드로 값에 의한 전달을 알아보자.
예시 코드로 C++ 언어를 사용하였는데, 값에 의한 전달과 참조에 의한 전달 모두 표현할 수 있는 언어이기 때문이다.
대부분은 주석을 통해 남겨두었으니 코드를 이해하는데는 큰 어려움은 없을 것이다.
```cpp
// 입출력 스트림을 포함하는 표준 라이브러리 헤더로, 콘솔에 출력하기 위해 사용한다
#include <iostream>

// std 이름 공간을 사용하여 std::cout, std::endl 등을 간략하게 사용하도록 설정했다
using namespace std;

void process(int value) {
    // 문자열과 변수의 값을 줄바꿈과 함께 콘솔에 출력한다
    cout << "Value passed into function: " << value << endl;
    
    value = 100;
    
    cout << "Value before leaving function: " << value << endl;
}

int main() {
    int parameter = 10;
    cout << "Value before function call: " << parameter << endl;
    
    process(parameter);
    
    cout << "Value after function call: " << parameter << endl;
    return 0;
}
```
`main()` 함수를 실행하면 어떤 결과가 나올까?

```bash
Value before function call: 10                                                                                                                                              
Value passed into function: 10                                                                                                                                                    
Value before leaving function: 100                                                                                                                                                
Value after function call: 10
```
`paramter` 변수의 값이 `process()` 함수를 수행한 후 100으로 바뀔 것을 예상했지만 변하지 않았다.

왜 이런 결과가 도출되었는지 콜 스택(Call Stack)을 통해 알아보자.
콜 스택은 프로그램 실행 중에 함수 호출을 관리하는 데이터 구조이다.

### 값에 의한 전달과 콜 스택

![pass-by-value-call-stack.png](img/pass-by-value-call-stack.png)

먼저 `main()` 함수에서 호출한 변수 `parameter`가 콜 스택에 쌓인다. 
이후 `process()` 함수가 호출되고 반환 주소(Return Address)가 저장된다. 
반환 주소란 함수가 실행을 마치고 다시 돌아가야 할 위치를 저장하는 메모리 주소이다. 

다음으로 매개변수인 변수 `value`가 콜 스택에 쌓인다.

콜 스택에 저장된 두 변수의 값은 10으로 같지만 스택의 서로 다른 메모리 주소로 관리된다.
위의 그림에서 보듯이 `main()` 함수의 `parameter` 변수는 `0x1234` 주소 값을 갖는 것에 비해 `process()` 함수의 `value` 변수는 `0x1270` 주소 값을 갖는 것을 알 수 있다.

![pass-by-value-call-stack-change-value.png](img/pass-by-value-call-stack-change-value.png)

`process()` 함수 내 value 변수의 값을 변경하면 `0x1270` 메모리에 할당된 값만 바뀌게 된다.

![pass-by-value-call-stack-reclaimed.png](img/pass-by-value-call-stack-reclaimed.png)

이후 `process()` 함수를 모두 수행한 후 반환되면 스택내  할당된 메모리가 해제된다. 
그렇기에 `process()` 함수의 결과가 `parameter`에 반영되지 않고 10을 유지했다.

이러한 매개변수 전달 방식을 값에 의한 전달이라 한다.

## 참조에 의한 전달(Pass by Reference, Call by Reference)

참조에 의한 전달은 주소 값을 전달하여 실제 값에 대한 참조(Alias)를 구성함으로써, 값을 수정하면 원본의 데이터가 수정되도록 하는 방식이다.

코드로 예시를 들어보자.
```cpp
// 입출력 스트림을 포함하는 표준 라이브러리 헤더로, 콘솔에 출력하기 위해 사용한다
#include <iostream>

// std 이름 공간을 사용하여 std::cout, std::endl 등을 간략하게 사용하도록 설정했다
using namespace std;

// c++은 참조값을 전달하기 위해 &를 사용한다.
void process(int& value) {
    cout << "Value passed into function: " << value << endl;
    
    value = 100;
    
    cout << "Value before leaving function: " << value << endl;
}

int main() {
    int parameter = 10;
    cout << "Value before function call: " << parameter << endl;
    
    process(parameter);
    
    cout << "Value after function call: " << parameter << endl;
    return 0;
}
```
`main()` 함수를 실행해보면 마지막 출력 문구가 값에 의한 전달과 다른 것을 알 수 있다.

```bash
Value before function call: 10                                                                                                                                              
Value passed into function: 10                                                                                                                                                    
Value before leaving function: 100                                                                                                                                                
Value after function call: 100
```

`process()` 함수의 참조를 넘겼기에 `process()` 함수 내부에서의 값의 변경이 함수 외부까지 영향을 끼친다.
이 과정을 콜 스택을 통해 다시 보자.

### 참조에 의한 전달과 콜 스택

![pass-by-reference-call-stack.png](img/pass-by-reference-call-stack.png)

먼저 `main()` 함수에서 선언된 `parameter` 변수가 스택에 쌓인다.
이후 `process()` 함수 호출되고 반환 주소(Return Address)가 적재되고 그 후 매개변수 정보를 스택에 저장한다.
이때 저장 되는 것은 10 이 아닌 변수 `parameter` 의 값이 저장된 메모리의 주소이다.

![pass-by-reference-change-value.png](img/pass-by-reference-change-value.png)

`process()` 함수에서 매개변수로 전달 받은 변수 `value`는 변수 `parameter`가 저장된 메모리 주소의 참조이다. 
그래서 `value`를 변경하면 `parameter`의 메모리 주소인 `0x1234` 내부의 값을 변경한다.

## Java의 매개변수 전달 방식

그렇다면 Java는 어떤 방식으로 매개변수를 전달할까?

> When the method or constructor is invoked (§15.12), the values of the actual argument expressions **_initialize newly created parameter variables_**, each of the declared type, before execution of the body of the method or constructor.
> </br> \- JLS(Java Language Specification) ch 8.4.1

JLS(Java Language Specification)에 의하면 Java는 인수가 복사되어 함수의 매개변수를 초기화하고, 그 후 함수의 본문이 실행된다고 나와있다.
여기서 인수(Argument)는 호출하는 쪽, 매개변수(Parameter)는 호출 되는 쪽을 의미한다.
즉, 공식 문서를 통해 Java가 값에 의한 호출을 하고 있음을 알 수 있다.

Java에는 기본적으로 원시 타입과 참조 타입이 존재한다. 
두 타입이 어떻게 값에 의한 전달(Pass by Value)를 사용하는지 이해하려면 Java의 메모리 할당 방식을 먼저 알아야 한다.

## Java 메모리 할당

![jvm-memory-structure.png](img/jvm-memory-structure.png)

Java의 메모리 할당은 Java의 가상머신인 JVM의 메모리 구조를 기반으로 이루어진다.
JVM의 메모리 구조는 크게 Stack 영역, Heap 영역, Method 영역으로 나눌 수 있다. 

이 중 우리가 주목할 메모리는 Stack과 Heap이다. 
Stack은 변수, 매개변수, 반환 값등 임시로 사용하는 정보들이 저장되며, Heap은 객체의 데이터와 참조 자료형이 저장된다.

아래 코드로 바탕으로 메모리에 어떻게 저장되는 알아보자.
```java
public class MemoryEx {
    
    public void ex(int x, boolean isDaon, String name, String[] array) {
        x = 3;
        isDaon = true;
        name = "daon";
        array[0] = "daon";
    }
    
    public static void main(String[] args) {
        int x = 3;
        boolean isDaon = true;
        String name = "Me";
        String[] array = {"I", "YOU"};
        ex(x, isDaon, name, array);
    }
}
```
각각의 변수에 대해 아래와 같이 메모리에 적재된다.

![java-ex-stack-heap-memory.png](img/java-ex-stack-heap-memory.png)

Java 는 2개의 타입이 존재한다.
하나는 int, long, short 등 원시 타입(Primitive Type)이고, 다른 하나는 이외 나머지를 뜻하는 참조 타입(Reference Type)이다. 
배열과 같은 객체들이 참조 타입에 속한다.

> [!tip]
> `String`은 객체로서 Heap에 저장되지만 Java에서 불변(immutable) 객체로 설계되어 있어서 특별한 동작 방식을 갖는다. 
> 자세한 개념이 궁금하다면 String Constant Pool에 대해 찾아보길 권장한다.

원시 타입은 위에서 설명한 것과 동일하게 값 복사 방식으로 동작한다.
문제는 참조 타입이다. 이에 대해 자세히 알아보자.

### 참조 타입 - 배열 array

배열 array를 살펴보면 Stack 영역에 Heap 영역의 메모리 주소를 저장하는 것을 볼 수 있다. 이를 보고 array가 `Pass by Reference`라고 헷갈릴 수 있다.
앞서 `Pass by Reference`는 실제 값에 대한 참조를 구성하여 값을 수정하면 원본의 데이터가 수정되는 방식이라고 정의하였다.
Java의 객체를 전달하는 방식은 주소값을 전달하지만, 이는 그저 `array`에 대한 복사본일 뿐이다.
다시 말해, 객체의 주소값으로 객체의 필드 값에 접근하여 값을 변경하는 것일 뿐, 실제 객체 자체에 변화를 주는 것이 아니다.
<br/>
<br/>
추가로 필드 변경 이외의 참조 타입으로 가능한 연산은 아래와 같다.

> JLS(Java Language Specification)의 4.3.1 절 Object
> - Field 접근
> - Method Invocation
> - Cast Operator
> - String의 `+` 연산자와 호출되면 `toString()` 함수를 호출하여 문자열로 변환하여 연결한다.
> - instanceof 연산자
> - == 또는 != 또는 ? :

## 서론의 예제 분석

처음에 다뤘던 Java 코드를 다시 보자.

```java
class Person {
    
    private String name;
    
    public Person(String name) {
        this.name = name;
    }
    
    public String getName() {
        return name;
    }
    
    public void setName(String name) {
        this.name = name;
    }
}


public class Main {
    
    public static void main(String[] args) {
        int x = 3;
        int[] array = {1, 2, 3};
        Person person1 = new Person("나");
        Person person2 = new Person("우리");
        
        foo(x, array, person1, person2);

        System.out.println("x: " + x);
        System.out.println("array[0]: " + array[0]);
        System.out.println("person1.name: " + person1.getName());
        System.out.println("person2.name: " + person2.getName());
    }
    
    private static void foo(int x, int[] array, Person person1, Person person2) {
        x++;
        array[0]++;
        person1 = new Person("바뀐 나");
        person2.setName("바뀐 우리");
    }
}
```

`main()` 함수의 실행 결과는 아래와 같다.

```bash
x: 3
array[0]: 2
person1.name: 나
person2.name: 바뀐 우리
```

이를 바탕으로 어떤 일이 일어났는지 메모리 공간을 통해 알아보자.
Stack 영역의 메모리 주소에서 검정 글씨는 `main()` 함수에서 일어난 변경을, 빨간 글씨는 `foo()` 함수에서 일어난 변경을 나타낸다.

### 0. 변수 생성
```java
public static void main(String[] args) {
    int x = 3;
    int[] array = {1, 2, 3};
    Person person1 = new Person("나");
    Person person2 = new Person("우리");
}
```
![img_1.png](img/java-memory-ex2-object.png)

우선 `main()` 함수에서 생성된 데이터들을 메모리에 나타내었다.
각각의 화살표는 참조하고 있는 메모리 주소를 나타낸다.

이제 `foo()` 함수 내부에서의 동작을 알아보자.

### 1. 원시 타입 x
```java
private void foo(int x) {
    x++;
}
```
![java-memory-ex-2-primitive.png](img/java-memory-ex-2-primitive.png)
변수 x는 값이 복사되어 매개변수로 오기 때문에 값이 변경된 이후 함수가 반환되면 `main()` 까지 x의 변경이 유지되지 않는다.

### 2. 참조 타입 배열 array

```java
private void foo(int[] array) {
    array[0]++;
}
```
![java-memory-ex-2-array.png](img/java-memory-ex-2-array.png)
앞서 참조 타입은 생성된 객체 내에 접근하여 값을 변경하는 것이 가능하다고 설명하였다.
참조 타입은 스택 안에 Heap 영역에서 할당된 메모리 주소를 저장하고 있기 때문에 실제 저장된 값의 변경이 일어난다.
<br/>
<br/>
위 그림에서 보면, `0x1000`이란 동일한 메모리 주소가 가리키는 공간에 변경을 가하여 Heap 영역 내 할당된 메모리 공간 내부 값에 직접적인 변경이 일어났다. 
따라서 매개변수로 받은 배열 내부 값이 변경되면 함수가 반환된 이후에도 변경이 유지되게 된다.

### 3. 참조 타입 Person1

```java
private void foo(Person person1) {
    person1 = new Person("바뀐 나");
}
```
![java-memory-ex-2-person-1.png](img/java-memory-ex-2-person-1.png)
person1 변수의 경우 앞서 설명한 array 와는 다르게 동작한다.
person1은 새로운 Person 객체를 생성하여 재할당하고 있다.
<br/>
<br/>
이때 위 그림을 보면, Heap 영역에 새롭게 할당된 `new Person("바뀐 나")`의 메모리 주소가 저장되었다.
Stack 영역을 보면 `main()` 함수의 person1과 `foo()` 의 `new Person("바뀐 나")`에 저장된 메모리 주소가 서로 다른 것을 알 수 있다.
이후 `foo()` 함수에서 person1를 수정해도 그 변경이 함수 외부까지 전파되지 않는 것이다.
<br/>
<br/>
따라서 객체가 재할당되는 경우 함수가 반환되면 그 변경이 유지되지 않는다.

### 4. 참조 타입 Person2

```java
private void foo(Person person2) {
    person2.setName("바뀐 우리");
}
```
![java-memory-ex-2-person-2.png](img/java-memory-ex-2-person-2.png)
이번 예시는 3번의 예시와 달리 `setter`를 활용하여 객체의 필드 값의 변경을 시도한다.
앞서 설명한 array의 동작 방식과 동일하게 Heap 영역에 할당된 메모리 주소로 직접적인 변경을 가한다.
그래서 `foo()` 함수가 반환된 이후에도 변경이 유지된다.

## 결론

예시 코드와 함께 Java의 매개변수 전달 방식에 대해 사용하는지 알아보았다.
<br/>
추후 함수의 변경이 의도한 대로 반영되지 않는다면 그 언어의 매개변수 전달 방식도 한번 살펴보길 바라며 글을 마무리하겠다.

---
### 참고 자료
[Java Language Specification 1.0](https://titanium.cs.berkeley.edu/doc/java-langspec-1.0/)
- Written by: James Gosling, Bill Joy, Guy Steele
- Created on: Aug. 1996
- Referenced on: Sep. 28, 2024
  
[The Java Language Specification, Java SE 23 Edition](https://docs.oracle.com/javase/specs/)
- Written by: Oracle
- Created on: Sep. 2024
- Referenced on: Sep. 28, 2024

[Passing by Value vs. Passing by Reference in Java](https://dzone.com/articles/pass-by-value-vs-reference-in-java)
- Written by: Justin Albano
- Created on: Oct. 20, 2017
- Referenced on: Sep. 28, 2024

[Pass-By-Value as a Parameter Passing Mechanism in Java](https://www.baeldung.com/java-pass-by-value-or-pass-by-reference)
- Written by: Baeldung
- Last Updated on Jan. 8, 2024
- Referenced on Sep. 28, 2024

[[Java] 메모리 관리 및 Pass By Value의 동작 방식 (2/3)](https://mangkyu.tistory.com/106)
<br/>[[Java] Pass By Value와 Pass By Reference의 차이 및 이해 (3/3)](https://mangkyu.tistory.com/106)
- Written by: 망나니개발자 (aka. Mangkyu)
- Created on Jan. 18, 2021
- Referenced on Sep. 28, 2024
