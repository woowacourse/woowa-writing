# 코틀린 코루틴 테스트

### 목차
1. 서론
2. 코루틴 테스트의 필요성
3. 코루틴 테스트의 두 가지 원칙
    - 가정을 버려라
    - 나무가 아닌 숲에 집중하라
    - 두 가지 원칙을 지킨 코루틴 테스트
4. 코틀린 코루틴 테스트 라이브러리
    - 코틀린 코루틴 테스트 버전
    - 코틀린 코루틴 훑어보기
    - runTest
    - TestDispatcher + TestCoroutineScheduler
        - StadardTestDispather
        - UnconfinedTestDispatcher 
6. 요약

<br>

## 1. 서론
아시다시피 코루틴은 비동기 프로그래밍을 도와주는 라이브러리이다. 

비동기적 특성 때문에 테스트에 어려움이 있다. 

이번 글에서는 코틀린 코루틴 테스트하는 방법을 소개한다.

<br>

## 2. 코루틴 테스트의 필요성

코루틴 테스트가 왜 필요할까요?

동시성 코드 특성상 개발 후반부 혹은 프로덕션 코드에서 버그가 발견되기 때문입니다.

이런 현상이 나타나는 이유는 동시성과 관련된 많은 버그가 실제로 발생할 가능성이 없다고 생각했거나 혹은 버그가 발생할 확률이 너무 낮아서 개발자나 코드 검토자가 염두에 두지 않았던 에지 케이스에서 발생하기 때문입니다.

이러한 파급효과를 예방하기 위해 코루틴 테스트가 필요합니다.

<br>

## 3. 코루틴 테스트의 두 가지 원칙

다음으로는 코루틴 테스트를 잘 작성하기 위한 두 가지 원칙입니다.

첫 번째는 "가정을 버려라."
- ex) 인프라 구조에 따른 가정을 하지 말기

두 번째는 "나무가 아닌 숲에 집중하라"라는 원칙입니다.
- 단위 테스트도 중요하지만, 기능 테스트에 집중

<br>

### 3.1. 가정을 버려라

![image](https://github.com/user-attachments/assets/c530837f-d926-4a0c-a30c-6f2614e38fa9)

가정한 속도 비교
Cache > DB > API

크루의 id를 사용해 해당 크루에 정보 세트를 검색 및 구성하고 반환해야 하는 간단한 상황을 가정해 보겠습니다.
- 나이는 cache로부터
- 이름은 db로부터
- 포지션은 API로부터 옵니다.

가정에서 보이는 바와 같이 cache가 가장 빠르고 db 그리고 API 순입니다. 당연하게도 개발자들은 이런 순서대로 데이터를 받을 거라 가정합니다.

<br>

**그러나** 다음과 같이 인프라가 바뀌어 DB가 API보다 느려지게 된다면 작동을 멈추거나 예상치 못한 일이 발생할 수 있습니다.

![image](https://github.com/user-attachments/assets/ca3e012b-55cd-4cbe-836b-66cab784a31e)

예상하지 못한 속도
Cache > API > DB

<br>

### 3.2. 나무가 아닌 숲에 집중하라

![image](https://github.com/user-attachments/assets/02f0c7de-8cde-46cb-8f24-dc38e2ae6cb2)

다음으로 "나무가 아닌 숲에 집중하라" 원칙에 대해 알아보겠습니다.

코루틴 테스트 할 때 "캐시로부터 정보 조회 성공" 같은 나무에 집중하는 것도 중요합니다. 하지만 숲에도 집중해야 합니다.
- DB가 API 보다 오래 걸려 누락된 정보가 있을 때 무슨 일이 발생하는지
- 하나에서만 정보를 가져온다면 무슨 일이 생기는지
- 애플리케이션은 검색한 것을 반환하거나 오류를 던져야 하는지 혹은 둘 다 수행해야 하는지?

즉, 하나 이상의 동시성 작업이 예상에서 벗어난 경우 준비된 작업을 하는 것을 보장하도록 작성된 애플리케이션에 초점을 맞추는 것이 중요합니다.

이러한 유형의 테스트를 기능 테스트라고 합니다.

기능 테스트를 하지 않은 동시성 애플리케이션은 취약해지며, 코드 변경에 따라 레이스 컨디션, 원자성 위반, 데드락 등의 혼란을 유발할 수 있습니다.

<br>

### 3.3. 두 가지 원칙을 지킨 코루틴 테스트

![image](https://github.com/user-attachments/assets/45de2d89-54e7-4677-a2d8-71f176992e98)
가정한 속도: DB > API

보시는 바와 같이 FastDbDataSource는 대부분의 개발자가 예상하듯이 DB가 API보다 빠르게 delay가 설정 되어 있습니다.

<br>

예상하지 못한 속도: API > DB
![image](https://github.com/user-attachments/assets/32f80bb2-501d-457a-9fdf-a3efc4cccdf1)

SlowDbDataSource는 대부분의 개발자가 예상에 빗나가게 db가 API보다 오래 걸리게 설정되어 있습니다.

<br>

![image](https://github.com/user-attachments/assets/c5fc5a50-cb39-4c5f-82f5-f478139d99a3)
![image](https://github.com/user-attachments/assets/482b420a-d03d-4415-b949-04435d277603)

두가지 원칙을 지켜서 작성한 테스트 코드를 살펴봅니다.

해피케이스에서는 CrewManager가 FastDbDataSource를 가지고 있습니다.
언해피케이스에서는 CrewManager가 SlowDbDataSource를 가지고 있습니다.

DB가 외부 시스템보다 더 오래 걸리는 것은 불가능하다는 말에 현혹돼 이점을 고려하지 않았다면 적어도 테스트를 작성할 때만큼은 고려해야 합니다.

테스트를 실행하면 언해피케이스 테스트가 실패하는 것을 볼 수 있습니다. 이를 통해 문제가 있다는 것을 인지하고 코드를 수정할 수 있습니다.

<br>

![image](https://github.com/user-attachments/assets/bc5a768b-3caa-4603-94cf-f9b40c22cbd7)

문제가 됐던 코드를 보면 postion이 가장 오래걸리는 api를 통해서 올것이라고 가정했기 때문에 postion에만 await가 걸려있습니다.

하지만 api가 더 빨리오면 오류가 발생합니다.

<br>

![image](https://github.com/user-attachments/assets/08911228-5e97-4b3f-b3a4-457724cd87c3)

실패한 테스트를 보고 수정한 코드입니다.
가정을 없애고 name, age, poistion 각각에 await가 있는 것을 볼 수 있습니다.
이제 테스트가 통과합니다.

<br>

![image](https://github.com/user-attachments/assets/98e88ff5-2be6-475a-8d39-d4617a0778d5)

그런데 아직 남아있는 문제점이 있습니다. 테스트를 돌리면 delay만큼 오래 걸린다는 점입니다.

이러한 테스트 코드가 늘어나면 테스트 돌리는 시간이 기하급수적으로 증가할 것입니다.

이러한 문제를 코틀린 코루틴 테스트 라이브러리를 사용하면 해결할 수 있습니다.

<br>

코틀린 코루틴 테스트 라이브러리 적용 시
![image](https://github.com/user-attachments/assets/69b5ef55-c2c8-4d30-a031-16dbf35343f5)

<br>

## 4. 코틀린 코루틴 테스트 라이브러리

### 4.1. 코틀린 코루틴 테스트 버전

![image](https://github.com/user-attachments/assets/4ec82a74-6178-459a-8bda-1a17dd753bcb)

코틀린 코루틴 테스트 라이브러리의 가장 최신 버전인 1.9.0 기준으로 설명하겠습니다.

<br>

### 4.2. 코틀린 코루틴 훑어보기

![image](https://github.com/user-attachments/assets/c930c1f4-780d-48fe-8eb2-2f0e8c23c11b)

테스트 코드를 보시면 runTest 블록 안에 코드들이 있는 것을 볼 수 있습니다. 

결론부터 말하자면 runTest는 테스트 환경 가상 시간 위에서 돌아가기 때문에 delay를 스킵할 수 있습니다.

<br>

### 4.3. runTest

![image](https://github.com/user-attachments/assets/4d8f0342-32cd-4dd6-a233-142a7c01e152)

runTest 구조입니다. runTest는 코루틴 테스트 목적으로 만들어진 특별한 코루틴 빌더입니다.
테스트 코드를 실행하고 자동으로 딜레이 스킵합니다. 또한 처리되지 않은 예외 핸들링을 할 수 있고 TimeOut도 설정할 수 있습니다.

runTest는 TestScope을 가지고 있고 TestScope은 TestDispatcher를 가지고 있습니다.
그리고 TestDispatcher는 TestCoroutineScheduler에 의존하고 있습니다.

### 4.4 TestDispatcher + TestCoroutineScheduler

![image](https://github.com/user-attachments/assets/71063a9c-c03f-407e-9d61-10ec47e26e37)

TestDispatcher의 구현체로는 StadradTestDispather, UnconfinedTestDispather가 있습니다.

모든 TestDispatcher는 하나의 TestCoroutineScheduler를 사용합니다.
왜냐하면 테스트 환경에서 시간의 흐름을 일관되게 제어하기 위해서입니다.

TestDispatcher의 딜레이는 TestCoroutineScheulder에 의해 컨트롤 됩니다.
실제 시간만큼 기다리는 DefaultDelay 대신 디스패처가 가진 scheduleResumeAfterDelay 함수를 호출합니다.

<br>

#### 4.4.1 StadardTestDispather

TestDispatcher의 구현체인 StadradTestDispather에 대해서 살펴보겠습니다.

runTest는 디폴트로 StandardTestDispatcher 사용합니다.
TestCoroutineScheduler에 연결되어 있다는 것을 제외하면 특별한 동작이 없는 단순한 디스패처입니다.

![image](https://github.com/user-attachments/assets/d2fb6e83-47c7-43e2-a84e-e49eb6cc4b16)

당연하게도 위 테스트 코드는 실패하게 됩니다. 

![image](https://github.com/user-attachments/assets/f7495f94-ca5c-44dd-b60d-441c51bd18d2)

자식 코루틴들이 실행되기 전에 assertEquals 코드가 실행되기 때문입니다.

테스트를 성공하게 하려면 가상 시간을 조작해야 합니다.

세 가지 메서드를 사용하여 가상 시간을 조작할 수 있습니다.

![image](https://github.com/user-attachments/assets/3c896038-085a-4ab9-a4cf-f145b49839b0)

- runcurrent()
    - runcurrent() 메서드는 현재 큐에 있는 모든 코루틴을 즉시 실행합니다.
 
runcurrent() 메서드는 현재 큐에 있는 모든 코루틴을 즉시 실행합니다.

![image](https://github.com/user-attachments/assets/f52c2a5e-eaec-42dd-89fc-3d20f6582291)

<br>

![image](https://github.com/user-attachments/assets/02537a45-9c6b-417e-bb9e-c809bfe7ea11)

- advanceTimeBy(delayTimeMillis: Long)
    - advanceTimeBy는 지정된 시간만큼 가상 시간을 진행합니다. 이 시간 동안 지연된 코루틴이 실행됩니다.

![image](https://github.com/user-attachments/assets/749a8676-cd9a-458c-9ab2-c6cbc495e7d5)

- advanceUntilIdle() 
    - 대기열에 있는 작업이 더 이상 없을 때까지 모든 대기열 작업을 실행합니다.

![image](https://github.com/user-attachments/assets/57d0d56b-e01b-48fe-b3d9-bf672c7618aa)

<br>

#### 4.4.2 UnconfinedTestDispatcher

![image](https://github.com/user-attachments/assets/b4690c23-f06d-4bb7-a64f-b41ce88e77f3)

UnconfinedTestDispatcher에 대해 살펴보겠습니다.

UnconfinedTestDispatcher는 코루틴을 즉시 실행하기 때문에, 테스트 실행 속도가 빠릅니다.

하지만 실제 동시성을 따르지 않기 때문에, 복잡한 테스트에서는 예상치 못한 결과가 발생할 수 있습니다.

따라서 꼭 필요한 경우가 아닌 이상 UnconfinedTestDispatcher보다는 StandardTetsDispatcher 사용하는 것이 좋습니다.

<br>

## 요약
코틀린 코루틴 테스트하는 방법에 대해 알아봤습니다.

runTest를 사용하면 편리하게 코틀린 코루틴 테스트를 사용할 수 있지만 잘 알고 어떻게 테스트할지 고민하고 사용하는 것도 중요합니다.

<br>

### 참고
- 코틀린 코루틴의 정석 책
- 코틀린 코루틴 책
- 코루틴 동시성 프로그래밍 책
- https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-test/
- <a href="https://www.flaticon.com/kr/free-icons/" title="나무 아이콘">나무 아이콘 제작자: Freepik - Flaticon</a>
- <a href="https://www.flaticon.com/kr/free-icons/" title="삼림지 아이콘">삼림지 아이콘 제작자: surang - Flaticon</a>
=======
2. 코루틴 테스트의 3원칙
3. 테스트 환경 설정
4. 테스트 작성하는 법
  4.1 runTest
  4.2 advanceTimeBy, advanceUntilIdle
5. 결론 


## 1. 서론
아시다시피 코루틴은 비동기 프로그래밍을 도와주는 라이브러리이다. 비동기적 특성 때문에 테스트에 어려움이 있다. 
이번 글에서는 코틀린 코루틴 테스트하는 방법을 소개한다.

## 2. 코루틴 테스트의 3원칙
코루틴 테스트에 대해 알아보기에 앞서 코투틴 테스트의 3원칙이 무엇인지 알아본다.

코류틴 테스트의 3원칙
1. 예측 가능성: 테스트는 항상 동일한 결과가 나와야한다.
2. 독립성: 각 테스트는 독립적으로 실행할 수 있어야 한다.
3. 속도: 테스트는 빠르게 실행되어야 한다.

## 3. 테스트 환경 설정
코틀린 코루틴 테스트를 위해서는 kotlinx-coroutines-test 라이브러리를 추가해야 한다.
build.gradle 파일에 다음과 같이 추가한다.

```kotlin
dependencies {
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:$coroutines_version")
    ...
}
```

kotlinx-coroutines-test 라이브러리는 Coroutine 테스트를 위한 다양한 도구를 제공한다.
- runTest: 간편하게 Coroutine 테스트를 실행하는 함수
- advanceTimeBy: 지정된 시간만큼 가상의 시간을 진행
- advanceUntilIdle: 모든 pending Coroutine이 실행될 때까지 가상의 시간을 진행

## 4. 테스트 작성하는 법

### 4.1 runTest

Coroutine은 특정 Scope 내에서 실행된다.
테스트 환경에서는 Dispatchers.Main 대신 StandardTestDispatcher를 사용하여 Coroutine 실행을 제어해야 한다. 
이를 통해 테스트 실행 속도를 높이고 예측 가능성을 높일 수 있다.

```kotlin
class MyViewModelTest {
    private val testDispatcher = StandardTestDispatcher()
    
    @BeforeEach
    fun setUp() {
        Dispatchers.setMain(testDispatcher)
    }
    
    @Test
    fun `MyViewModel 테스트`() = runTest { 
        // 테스트 작성
    }
    
    @AfterEach
    fun tearDown() {
        Dispatchers.resetMain()
    }
}
```

runTest는 Coroutine 테스트를 위한 함수이다. 
TestCoroutineScope를 제공하며, 테스트 종료 시 자동으로 모든 Coroutine을 취소해준다. 
runTest 내부에서는 pauseDispatcher와 advanceUntilIdle 등의 함수를 사용하여 Coroutine 실행을 제어할 수 있다.

### 4.2 advanceTimeBy, advanceUntilIdle

advanceTimeBy는 지정된 시간만큼 가상 시간을 진행시키고, advanceUntilIdle은 모든 pending Coroutine이 실행될 때까지 가상 시간을 진행시킨다.

```kotlin
class MyViewModelTest {

    private val testDispatcher = StandardTestDispatcher()

    @BeforeEach
    fun setUp() {
        Dispatchers.setMain(testDispatcher)
    }

    @Test
    fun `MyViewModel 테스트`() = runTest {
        val deferred = async {
            delay(100000)
            "Delayed Result"
        }

        advanceTimeBy(100000)

        assertEquals("Delayed Result", deferred.await())
    }

    @AfterEach
    fun tearDown() {
        Dispatchers.resetMain()
    }
}
```

위 테스트에서 delay가 100초가 걸려있음에도 아래와 같이 빠르게 33ms만에 실행완료 되는 것을 볼 수 있다.
<img width="983" alt="image" src="https://github.com/user-attachments/assets/263a2682-a0b1-4c05-ba98-9056722bb463">

```kotlin
class MyViewModelTest {

    private val testDispatcher = StandardTestDispatcher()

    @BeforeEach
    fun setUp() {
        Dispatchers.setMain(testDispatcher)
    }

    @Test
    fun `MyViewModel 테스트`() = runTest {
        val deferred = async {
            delay(100000)
            "Delayed Result"
        }
        val deferred2 = async {
            delay(100000)
            "Delayed Result2"
        }

        advanceUntilIdle()

        println(deferred.await())

        assertEquals("Delayed Result", deferred.await())
        assertEquals("Delayed Result2", deferred2.await())
    }

    @AfterEach
    fun tearDown() {
        Dispatchers.resetMain()
    }
}
```

advanceUntilIdle() 메서드를 사용하면 advanceTimeBy와 같이 일일히 시간을 설정하지 않아도 편하게 가상 시간을 진행시킬 수 있다.

그런데 사실 위 두 메서드를 사용하지 않아도 아래 테스트를 실행하면 빠르게 종료된다.
그 이유는 runTest가 대신 그러한 일들을 해주기 때문이다.

```kotlin
class MyViewModelTest {

    private val testDispatcher = StandardTestDispatcher()

    @BeforeEach
    fun setUp() {
        Dispatchers.setMain(testDispatcher)
    }

    @Test
    fun `MyViewModel 테스트`() = runTest {
        val deferred = async {
            delay(100000)
            "Delayed Result"
        }
        val deferred2 = async {
            delay(100000)
            "Delayed Result2"
        }
        
        println(deferred.await())

        assertEquals("Delayed Result", deferred.await())
        assertEquals("Delayed Result2", deferred2.await())
    }

    @AfterEach
    fun tearDown() {
        Dispatchers.resetMain()
    }
}
```

### 결론
코틀린 코루틴 테스트하는 방법에 대해 알아봤다. 
runTest를 사용하면 편리하게 코틀린 코루틴 테스트를 사용할 수 있지만 잘 알고 사용하는게 중요하다.

참고
- 코틀린 코루틴의 정석 책
- 코틀린 코루틴 책
- 코루틴 동시성 프로그래밍 책
