# 코틀린 코루틴 테스트

### 목차
1. 서론
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
