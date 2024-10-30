# 꼭 필요한 추상화를 하기

## Intro

- 사람은 정보를 말로 전달한 후 72시간 뒤에 약 10%를 기억한다. 하지만 그림을 곁들이면 65%를 기억한다.

- 객체 지향 프로그래밍을 접할 때 SOLID 원칙을 처음으로 배우게 된다.
- 글보다는 그림에 집중 -> 목적과 의도보다는 각 클래스간의 구조에 집중하게 된다.



### 추상화란?

> Abstraction is one of the key concepts of object-oriented programming (OOP) languages. Its main goal is to handle complexity by hiding unnecessary details from the user. That enables the user to implement more complex logic on top of the provided abstraction without understanding or even thinking about all the hidden complexity.
[Abstraction in Programming: A Beginner’s Guide]

- (해당 코드를 사용하는) 사람에게 불필요한 정보를 제거하여 복잡한 정보를 알지 않더라도 사용할 수 있도록 한다.
- 메서드의 호출에 따라 해당 객체가 어떻게 행동하는지만을 판단한다. 내부가 어떤 식으로 구성되어 있는지 알지 않아도 사용할 수 있다.

```java

```

- ex. Collection Library (List)
  - List 구조가 어떤 구조이고, 메서드에 따라 어떻게 행동하는지만 알면 된다.
  - 실제 구현체가 어떤 구조로 되어있는지 알지 않아도 사용자는 해당 행동에 따라 어떻게 될지 예측하여 사용할 수 있다. 

### ISP란?

- ISP (Interface Segregation Principle)

> The Interface Segregation Principle (ISP) suggests that a class should not be forced to implement methods it doesn’t need. In other words, a class should have small, focused interfaces rather than large, monolithic ones.

- 인터페이스와 실 구현체의 분리
- 목적 : 

- ex. Collection Library (List)
  - 
  - 상황에 따라, 목적에 따라 더 어울리는 구현체를 사용할 수 있다. (ex. ArrayList, LinkedList, CopyOnWriteArrayList)
  - 인터페이스가 실 구현체의 상위 타입이 되기 때문에 

```java

```

- ISP의 목적에 대해 잘 알아보지 않고, 단순히 구조만을 적용하려 한다.
- 아쉽게도 ~해서 좋다 라고 알려주지, 언제 사용해야 하는지 알려주지 않는다. 그리고 객체지향 프로그래밍을 처음 배우는 사람은 이를 아무 의심 없이 받아들인다.

![image](./image/isp.png)

## 인터페이스를 잘못 적용한 예시

- 아무 목적 없이 "명세와 구현의 분리"한 예시
- 

### 무의미한 추상화

```java
public interface Car {
    
    void move();
}

public class CarImpl implements Car {
    
    private int position;

    @Override
    public void move() {
        position++;
    }
}
```

### 너무 넓은 범위 기능을 인터페이스로 적용한 경우

```java
public interface Validator {

    void validate(String input);
}
```

```java
public class NameValidator implements Validator {
    
    @Override
    public void validate(String input) {
        // 이름 검증 기능
    }
}

public class MovingCountValidator implements Validator {

    @Override
    public void validate(String input) {
        // 이동 횟수 검증 기능
    }
}
```
- 인터페이스의 목적

- 사용하는 메서드의 형태가 일치한다고 생각하여 섣부르게 인터페이스를 사용함
  - 서로 다른 구현체를 넣으면 원하는 데로 동작하지 않는다.

### 현재 존재하지 않는 요구사항을 대비한 추상화


```java

```

## 좋은 추상화란?

### 



### 목적 있는 추상화



## 인터페이스를 잘 적용한 예시


### 현재 상황에 알맞게 적용한 인터페이스 도입



### 용이한 테스트를 위한 인터페이스 도입

```java
import java.util.Random;

public class Car {

    private static final Random RANDOM = new Random();
    private static final int MAX_RANDOM_NUMBER = 9;
    private static final int MOVE_FORWARD_BOUND = 4;
    
    private int position = 0;
    ...

    public void move() {
        int randomNumber = Random.nextInt(MAX_RANDOM_NUMBER);
        if (numberSupplier.supply() >= MOVE_FORWARD_BOUND) {
            position++;
        }
    }
}
```

```java
    @Test
    void moveTest_whenSuppliedNumberIsOverBound_moveForward(int overBoundNumber) {
        Car car = new Car();

        car.move();

        // move를 한 이후에 어떻게 될지 알 수 없음
    }
```

```java
public interface NumberSupplier {

    int supply();
}

public class RandomNumberSupplier implements NumberSupplier {

    private static final int MIN_RANDOM_NUMBER = 0;
    private static final int MAX_RANDOM_NUMBER = 9;

    @Override
    public int supply() {
        return Randoms.pickNumberInRange(MIN_RANDOM_NUMBER, MAX_RANDOM_NUMBER);
    }
}
```

```java
public class Car {

    private static final NumberSupplier DEFAULT_NUMBER_SUPPLIER = new RandomNumberSupplier();
    private static final int INITIAL_POSITION = 0;
    private static final int MOVE_FORWARD_BOUND = 4;

    private final NumberSupplier numberSupplier;
    private int position;

    public Car(NumberSupplier numberSupplier) {
        this.numberSupplier = numberSupplier;
        this.position = INITIAL_POSITION;
    }

    public Car() {
        this(DEFAULT_NUMBER_SUPPLIER);
    }

    public void move() {
        if (numberSupplier.supply() >= MOVE_FORWARD_BOUND) {
            position++;
        }
    }

    public int getPosition() {
        return position;
    }

    ...

}
```


```java
class CarTest {

    @ParameterizedTest
    @ValueSource(ints = {4, 9})
    void moveTest_whenSuppliedNumberIsOverBound_moveForward(int overBoundNumber) {
        NumberSupplier overBoundSupplier = new FixedNumberSupplier(overBoundNumber);
        Car car = new Car(overBoundSupplier);
        int expectedPosition = car.getPosition() + 1;

        car.move();

        assertThat(car.getPosition()).isEqualTo(expectedPosition);
    }

    @ParameterizedTest
    @ValueSource(ints = {0, 3})
    void moveTest_whenSuppliedNumberIsUnderBound_stop(int underBoundNumber) {
        NumberSupplier underBoundSupplier = new FixedNumberSupplier(underBoundNumber);
        Car car = new Car(underBoundSupplier);
        int expectedPosition = car.getPosition();

        car.move();

        assertThat(car.getPosition()).isEqualTo(expectedPosition);
    }
}

public class FixedNumberSupplier implements NumberSupplier {

    private final int number;
    
    public FixedNumberSupplier(int number) {
        this.number = number;
    }

    @Override
    public int supply() {
        return number;
    }
}
```

## 결론

- ISP
  - 미래의 문제를 대비하기 위해 적용하지 말자
  - 현재의 문제를 해결할 수 있는 하나의 수단으로 생각해보자
- Trade-Off
  - 파일의 개수가 많아지는 만큼 복잡도가 증가한다
  - 

## Outro

- "망치를 들면 모든 것이 못으로 보인다", "미래를 위한 대비?"
- 우리는 좋은 코드를 작성하기 위해 SOLID를 도입했지, SOLID를 도입한다고 무조건 좋은 코드가 되는 것은 아니다.
  - 배운 것을 도입할 때에는 나에게 주어진 상황에 적합한지 의심해보는 습관이 필요하다. 

### 참고 자료

- [Abstraction in Programming: A Beginner’s Guide](https://stackify.com/oop-concept-abstraction/) (검색 일자 : 24.10.30)
- [4. Interface Segregation Principle (ISP): SOLID Principle](https://medium.com/@ramdhas/4-interface-segregation-principle-isp-solid-principle-39e477bae2e3) (검색 일자 : 24.10.30)
- [SOLID: Interface Segregation Principle (ISP)](https://www.evertop.pl/en/understanding-solid-principles-interface-segregation/) (검색 일자 : 24.10.30)