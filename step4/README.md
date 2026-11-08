# 직접 만든 DI 컨테이너에서 리플렉션은 무슨 일을 대신할까?

Android에서 <code>Serializable</code>과 <code>Parcelable</code>의 차이를 공부하다가 익숙한 설명 하나를 발견했다.

> Serializable은 리플렉션을 사용하기 때문에 Parcelable보다 느리다.

처음에는 별다른 의문 없이 받아들였다. 그런데 조금 더 생각해 보니 궁금한 점이 생겼다. 리플렉션의 정확히 어떤 작업이 느리다는 걸까? 클래스 정보를 찾아보는 과정일까? 생성자를 호출하는 과정일까? 아니면 객체의 필드에 접근하는 과정일까? 그리고 그 차이는 실제 애플리케이션에서도 문제가 될 만큼 클까?

마침 직접 구현한 DI(Dependency Injection) 컨테이너에서도 리플렉션을 사용하고 있었다. ViewModel마다 Factory를 작성하지 않아도 필요한 의존성을 자동으로 연결해 주는 구조였다. 구현할 때는 반복되는 코드가 줄어들어 편리하다고 생각했다. 하지만 컨테이너가 그 일을 어떻게 대신 수행하는지, 그리고 그 과정에서 어떤 비용을 치르는지는 깊이 살펴보지 않았다.

ViewModel의 의존성을 Factory에서 연결해 본 적이 있고, 리플렉션이 그 코드를 어떻게 대신하는지 궁금한 Android 개발자를 대상으로 썼다. Kotlin의 기본 문법과 ViewModel을 알고 있다고 가정하고, 리플렉션 API는 코드와 함께 설명한다. 직접 만든 DI 컨테이너의 객체 생성 과정을 따라가며 다음 질문을 살펴보려고 한다.

- 리플렉션은 런타임에 어떤 정보를 읽고, 어떻게 객체를 생성할까?
- DI 컨테이너는 리플렉션을 사용해 의존성을 어떻게 연결할까?
- 직접 객체를 생성하는 방식과 비교해 무엇이 편리해지고, 어떤 대가가 생길까?
- 리플렉션의 성능을 제대로 비교하려면 무엇을 측정해야 할까?

성능 측정은 아직 수행하지 않았다. 따라서 이번 글에서는 실제 구현으로 확인할 수 있는 동작과 구조적 특성을 분석하고, 성능에 대해서는 확인된 사실과 검증이 필요한 가설을 구분하려 한다.

---

## 1. 리플렉션은 무엇을 가능하게 할까?

일반적으로 객체를 생성할 때는 어떤 클래스의 인스턴스를 만들지 코드에 직접 작성한다.

```txt
data class User(
    val name: String,
    val age: Int,
)

val user = User(
    name = "조디악",
    age = 25,
)
```

평범한 Kotlin 코드다. 하지만 이 한 줄에는 중요한 전제가 있다. 컴파일 시점에 어떤 생성자를 호출할지 이미 알고 있다는 것이다. 컴파일러는 <code>User</code>의 생성자와 파라미터를 확인한다. <code>name</code>에 문자열을 전달했는지, <code>age</code>에 정수를 전달했는지 검사하고, 잘못된 인자를 전달하면 컴파일 과정에서 오류를 발견할 수 있다. 그런데 어떤 클래스를 생성할지 실행 중에 결정해야 한다면 어떨까?

예를 들어 전달받은 타입에 따라 서로 다른 객체를 만들어야 하는 범용 라이브러리를 개발한다고 생각해 보자. 모든 클래스에 대해 <code>User(...)</code>, <code>ProductRepository(...)</code>처럼 구체적인 생성 코드를 작성할 수는 있지만, 라이브러리가 처리할 타입이 늘어날 때마다 생성 로직도 수정해야 한다. 리플렉션은 이런 상황에서 다른 접근 방식을 제공한다.

### 1.1 런타임에 생성자를 찾아 호출하기

앞선 <code>User</code> 객체를 Kotlin Reflection으로 생성해 보자.

```txt
import kotlin.reflect.full.primaryConstructor

val constructor = User::class.primaryConstructor
    ?: error("User의 주 생성자를 찾을 수 없습니다.")

val user = constructor.call("조디악", 25)
```

결과적으로 만들어지는 객체는 동일하다. 하지만 객체를 생성하는 과정은 다르다. 먼저 <code>User::class</code>를 통해 <code>User</code> 클래스의 정보를 나타내는 <code>KClass</code>를 얻는다. <code>KClass</code>는 Kotlin에서 클래스에 관한 런타임 정보를 다루는 인터페이스다. 이를 통해 생성자, 프로퍼티, 함수 등에 관한 정보를 확인할 수 있다.

그다음 <code>primaryConstructor</code>를 사용해 주 생성자를 가져온다.

```txt
val constructor = User::class.primaryConstructor
```

여기서 얻은 생성자는 <code>KFunction</code>으로 표현된다. <code>KFunction</code>은 호출 가능한 함수를 나타내며, 생성자도 호출 가능한 대상으로 다룰 수 있다. 마지막으로 <code>call()</code>에 인자를 전달해 생성자를 실행한다.

```txt
val user = constructor.call("조디악", 25)
```

이 과정을 정리하면 다음과 같다.

| 직접 생성 | 리플렉션을 이용한 생성 |
| --- | --- |
| <code>User(...)</code> 호출 | <code>KClass</code>에서 생성자 조회 |
| 컴파일러가 인자 검사 | 런타임에 인자 전달 |
| 정해진 생성 코드 실행 | <code>KFunction.call()</code> 실행 |
| 주요 타입 불일치를 컴파일 시점에 검출 | 동적 호출의 인자 오류가 런타임에 드러날 수 있음 |

여기서 주목할 부분은 리플렉션이 새로운 종류의 객체를 만드는 기술이 아니라는 점이다. 리플렉션은 객체를 생성하는 방법을 런타임에 결정할 수 있도록 해 준다. 다만 그만큼 컴파일러가 직접 확인하던 일부 책임이 실행 시점으로 옮겨간다. 가령 <code>call()</code>에 전달한 인자의 개수나 타입이 생성자와 맞지 않으면 런타임에 예외가 발생한다. 그렇다면 이렇게 복잡하게 객체를 생성할 필요가 있을까?

<code>User</code> 하나만 생성한다면 굳이 리플렉션을 사용할 이유가 없다. 직접 생성하는 편이 간단하고 명확하다. 하지만 수십 개의 서로 다른 클래스를 공통 규칙으로 생성해야 한다면 이야기가 달라진다. 리플렉션은 서로 다른 객체를 만드는 과정을 하나의 규칙으로 일반화할 수 있게 해 준다. 이 특성이 직접 구현한 DI 컨테이너에서 활용됐다.

---

## 2. 직접 만든 DI 컨테이너는 리플렉션을 어떻게 사용했을까?

DI는 객체가 필요로 하는 의존성을 객체 내부에서 직접 생성하는 대신 외부에서 제공하는 방식이다. 예를 들어 <code>ProductsViewModel</code>이 <code>ProductRepository</code>를 사용한다면, ViewModel이 Repository의 생성과 관리까지 책임지지 않도록 역할을 분리할 수 있다. 그런데 의존성을 외부에서 제공하려면 결국 어딘가에서는 객체를 생성하고 연결해야 한다.

이 작업을 담당하도록 만든 것이 <code>DependencyContainer</code>다. 현재 프로젝트에서는 Compose 화면에서 다음과 같이 ViewModel을 요청한다.

```txt
val viewModel: ProductsViewModel =
    viewModel(factory = DependencyContainer)
```

화면에서 <code>ProductRepository</code>나 <code>CartRepository</code>의 생성 코드를 직접 작성하지 않는다. 대신 <code>DependencyContainer</code>가 ViewModel을 만들고, 필요한 의존성을 찾아 연결한다. 여기에는 크게 세 단계가 있다.

1. ViewModel의 생성자를 찾아 인스턴스를 만든다.
2. <code>@Inject</code>가 붙은 필드를 탐색한다.
3. 필요한 의존성을 해결하고 필드에 주입한다.

각 단계를 실제 코드로 살펴보자.

### 2.1 첫 번째 단계: ViewModel 생성하기

먼저 Factory가 호출되는 부분이다.

```txt
override fun <T : ViewModel> create(modelClass: Class<T>): T {
    val scope = createScope()
    val viewModel = create(modelClass, scope)
    viewModel.addCloseable(scope)
    return viewModel
}
```

Factory는 새로운 <code>DependencyScope</code>를 만들고, 해당 스코프를 사용해 ViewModel을 생성한다. 생성한 스코프는 <code>addCloseable()</code>을 통해 ViewModel의 생명주기와 연결한다. 즉, ViewModel에 필요한 의존성을 관리할 범위를 준비한 뒤 실제 객체 생성을 진행하는 것이다.

다음은 실제 <code>create()</code> 구현이다. 예외 메시지와 반환문의 <code>!!</code>도 원본 그대로 옮겼으며, 개선할 점은 4.1절에서 살펴본다.

```txt
fun <T : ViewModel> create(
    modelClass: Class<T>,
    scope: DependencyScope,
): T {
    val viewModel = modelClass.kotlin.primaryConstructor?.call()
        ?: throw IllegalArgumentException()

    injectFields(viewModel, scope)
    return modelClass.cast(viewModel)!!
}
```

생성자를 조회하고 호출하는 부분을 따로 보면 다음과 같다.

```txt
modelClass.kotlin.primaryConstructor?.call()
```

<code>modelClass</code>는 생성할 ViewModel의 클래스 정보를 담은 Java <code>Class</code> 객체다. 여기서 <code>.kotlin</code>으로 Kotlin의 <code>KClass</code>를 얻고, <code>primaryConstructor</code>로 주 생성자를 조회한 뒤 <code>call()</code>로 실행한다. 앞서 <code>User</code> 객체를 만들었던 방법과 같다.

차이가 있다면 실제 ViewModel에서는 생성자에 인자를 전달하지 않는다는 것이다. 현재 구현은 ViewModel의 의존성을 생성자 파라미터가 아닌 필드로 주입하는 방식을 사용한다. 따라서 인자 없는 주 생성자를 호출해 ViewModel을 먼저 만든다. 이후 다음 함수를 실행한다.

```txt
injectFields(viewModel, scope)
```

여기서부터 의존성 주입이 시작된다.

### 2.2 두 번째 단계: @Inject가 붙은 필드 찾기

객체를 생성했다고 해서 필요한 의존성이 자동으로 연결되는 것은 아니다. 컨테이너는 어떤 필드에 의존성을 주입해야 하는지 알아야 한다. 현재 구현에서는 <code>@Inject</code> 애너테이션을 그 기준으로 사용한다.

<code>ProductsViewModel</code>은 <code>ProductRepository</code>와 <code>CartRepository</code>를 필드 의존성으로 사용한다. 그중 <code>CartRepository</code>에는 <code>@RoomCart</code> qualifier가 지정돼 있다. qualifier는 동일한 타입에 여러 구현체가 연결될 수 있을 때 어떤 구현체를 사용할지 구분하기 위한 정보다.

컨테이너는 이러한 필드를 어떻게 찾을까?

```txt
instance::class.java.declaredFields
    .filter { field ->
        field.isAnnotationPresent(Inject::class.java)
    }
    .forEach { field ->
        // 의존성 조회 및 주입
    }
```

여기에서는 Kotlin Reflection이 아니라 Java Reflection을 사용한다. <code>instance::class.java</code>는 객체의 Java <code>Class</code> 정보를 가져온다. 그리고 <code>declaredFields</code>는 해당 클래스에 선언된 필드들을 반환한다. 컨테이너는 각 필드를 순회하면서 다음 조건을 검사한다.

```txt
field.isAnnotationPresent(Inject::class.java)
```

필드에 <code>@Inject</code> 애너테이션이 있다면 의존성 주입 대상으로 판단한다. 즉, 개발자가 매번 어떤 필드에 값을 대입해야 하는지 작성하지 않아도, 컨테이너가 애너테이션을 기준으로 주입 대상을 찾아낼 수 있다. 이것이 리플렉션을 사용해 얻은 첫 번째 편리함이다.

다만 <code>declaredFields</code>는 해당 클래스에 직접 선언된 필드를 조회한다. 상속받은 필드까지 모두 탐색하는 API는 아니다. 따라서 상속 구조까지 지원하려면 별도의 탐색 로직을 고려해야 한다.

### 2.3 세 번째 단계: 필요한 객체를 찾아 필드에 넣기

주입할 필드를 찾았다면 이제 그 필드에 들어갈 객체를 구해야 한다. 실제 구현의 핵심은 다음과 같다.

```txt
// qualifier를 판별하는 부분은 생략

val dependency = resolve(
    field.type.kotlin,
    qualifier,
    scope,
)

field.isAccessible = true
field.set(instance, dependency)
```

여기서는 두 종류의 작업이 일어난다. 먼저 <code>field.type.kotlin</code>을 통해 해당 필드가 요구하는 타입을 확인하고, <code>resolve()</code>로 그 타입에 맞는 의존성을 구한다. 그다음 필드에 값을 대입한다.

```txt
field.isAccessible = true
field.set(instance, dependency)
```

<code>isAccessible = true</code>는 Java Reflection을 통해 해당 필드의 접근 검사를 우회할 수 있도록 설정하는 부분이다. 이후 <code>field.set()</code>으로 객체의 필드에 의존성을 대입한다. 평소 Kotlin에서는 다음과 같이 작성할 작업이다.

```txt
viewModel.repository = repository
```

하지만 리플렉션을 사용하면 필드 이름을 직접 알지 못해도, 런타임에 찾은 <code>Field</code> 객체를 통해 값을 대입할 수 있다. 이 덕분에 컨테이너는 ViewModel마다 별도의 대입 코드를 작성하지 않고도 공통 주입 로직을 적용할 수 있다. 물론 여기에도 대가가 있다. 컴파일러가 확인하는 일반적인 프로퍼티 대입과 달리, 동적으로 찾은 필드에 대입하는 값이 올바른지 런타임에 검증해야 한다.

또한 필드의 접근 제어를 우회하는 코드이므로, 객체의 내부 구현에 대한 의존성도 생긴다. 지금까지 살펴본 내용을 요약하면 다음과 같다. Kotlin Reflection은 생성자를 찾고 호출하는 데, Java Reflection은 필드와 애너테이션을 탐색하고 값을 대입하는 데 사용했다. 그런데 아직 한 가지 중요한 질문이 남았다. <code>resolve()</code>는 필요한 의존성을 어떻게 만들어 낼까?

---

## 3. 의존성의 의존성은 어떻게 해결할까?

2장에서 주입한 두 필드를 실제 프로젝트의 의존성 그래프로 그리면 다음과 같다.

```txt
ProductsViewModel
    ├── @Inject ProductRepository
    │       └── 인자 없는 주 생성자로 생성 → ViewModel 스코프에 저장
    │
    └── @Inject @RoomCart CartRepository
            └── 앱 스코프에 등록된 RoomCartRepository 반환
                    └── CartProductDao (앱 시작 시 생성자에 전달)
```

<code>ProductRepository</code>는 생성자 의존성이 없는 구체 클래스이므로 컨테이너가 바로 생성할 수 있다. 반면 <code>CartRepository</code>는 인터페이스다. 앱의 <code>ShoppingApplication.onCreate()</code>에서 <code>RoomCartRepository(cartProductDao)</code>를 만들어 <code>CartRepository</code> 타입과 <code>RoomCart</code> qualifier에 연결해 등록한다.

따라서 이 화면에서 컨테이너가 <code>RoomCartRepository</code>와 DAO를 재귀적으로 만드는 것은 아니다. 이미 연결해 둔 인스턴스를 찾아 반환한다. 생성자가 다른 객체를 요구하는 미등록 타입이라면, 아래의 재귀 호출 경로로 그 파라미터를 해결한다.

### 3.1 생성자 파라미터를 따라가는 재귀적 탐색

실제 <code>resolve()</code>에서 객체를 생성하는 핵심 부분을 추리면 다음과 같다.

```txt
val constructor = type.primaryConstructor
    ?: throw IllegalArgumentException("No primary constructor for ${type.qualifiedName}")

val arguments = constructor.parameters.map { parameter ->
    val dependencyType = parameter.type.classifier as? KClass<*>
        ?: throw IllegalArgumentException("Unsupported parameter: ${parameter.name}")

    resolve(dependencyType, scope = scope)
}

return constructor.call(*arguments.toTypedArray()).also { instance ->
    injectFields(instance, scope)
    scope.put(DependencyKey(type), instance)
}
```

동작을 나누면 네 단계다.

첫째, 생성자를 찾는다.

```txt
val constructor = type.primaryConstructor
```

생성하려는 타입의 주 생성자를 가져온다.

둘째, 생성자가 요구하는 파라미터를 확인한다.

```txt
constructor.parameters.map { parameter ->
    val dependencyType = parameter.type.classifier as? KClass<*>
        ?: throw IllegalArgumentException("Unsupported parameter: ${parameter.name}")

    resolve(dependencyType, scope = scope)
}
```

<code>constructor.parameters</code>로 생성자의 파라미터 목록을 읽고, 각 파라미터의 타입을 확인한다. 그리고 다시 <code>resolve()</code>를 호출한다. 필요한 객체가 또 다른 객체를 필요로 한다면, 그 의존성부터 해결하는 것이다. 이런 방식으로 의존성 관계를 따라 내려가다가 더 이상 추가 의존성이 필요하지 않은 객체를 만나면 그 객체부터 생성할 수 있다.

셋째, 준비한 인자로 인스턴스를 만든다.

```txt
constructor.call(*arguments.toTypedArray())
```

재귀적인 호출을 통해 얻은 의존성들을 <code>arguments</code>에 모아 생성자에 전달한다.

넷째, 생성한 객체를 스코프에 저장한다.

```txt
scope.put(DependencyKey(type), instance)
```

객체를 생성한 뒤 현재 스코프에 저장한다. 이후 같은 스코프에서 같은 의존성을 요청하면 저장된 인스턴스를 재사용할 수 있다.

### 3.2 리플렉션과 스코프는 서로 다른 문제를 해결한다

여기서 한 가지 구분할 필요가 있다. 리플렉션은 어떤 타입의 생성자를 호출할지 결정하는 데 사용된다. 반면 스코프는 생성한 객체를 어디에 보관하고, 어느 범위에서 재사용할지 결정한다. 실제 프로젝트에서도 이 둘을 함께 사용했다. <code>ProductRepository</code>처럼 별도의 등록 없이 생성할 수 있는 타입은 컨테이너가 생성자를 확인하고 인스턴스를 만든 뒤 ViewModel 스코프에 저장한다.

반면 <code>RoomCartRepository</code>는 앱 시작 시 <code>CartRepository</code> 타입과 <code>RoomCart</code> qualifier에 연결해 앱 스코프에 등록한다. 이후 ViewModel에서 <code>CartRepository</code>를 요청하면, 현재 스코프와 부모 앱 스코프를 확인해 등록된 구현체를 가져온다. 전체 흐름을 정리하면 다음과 같다. 그림은 정상 생성 경로를 요약했다. 실제 <code>resolve()</code>는 같은 타입이 다른 qualifier로 등록되어 있으면 생성자 탐색 전에 예외를 던지며, 이 검사는 그림에서 생략했다.

```txt
Compose 화면
    │
    ▼
ProductsViewModel 요청
    │
    ▼
DependencyContainer
    │
    ├── ViewModel 스코프 생성
    │
    └── ViewModel 생성
            │
            ▼
        @Inject 필드 탐색
            │
            ▼
        resolve(타입, qualifier, scope)
            │
            ▼
        등록된 인스턴스가 있는가?
            │
            ├── YES → 기존 인스턴스 반환
            │
            └── NO
                 │
                 ▼
              생성자 탐색
                 │
                 ▼
              파라미터 의존성 해결
                 │
                 ▼
              새 인스턴스 생성
                 │
                 ▼
              필드 주입 및 스코프 저장
```

이 구조에서 얻은 이점은 분명했다. ViewModel이나 Repository가 늘어나더라도 컨테이너의 공통 생성 로직을 재사용할 수 있다. <b>객체 생성과 연결의 책임이 개별 생성 코드에서 컨테이너의 런타임 로직으로 이동했다.</b> 이 차이가 리플렉션의 장점이자 비용으로 이어진다.

---

## 4. 편리해진 만큼 무엇을 포기했을까?

처음 DI 컨테이너를 구현할 때 가장 만족스러웠던 부분은 반복 코드가 줄어든다는 점이었다. ViewModel마다 의존성을 연결하는 Factory를 따로 작성하지 않아도 됐다. 객체의 생성자 구조가 달라지더라도 컨테이너가 파라미터를 읽고 의존성을 재귀적으로 해결할 수 있었다. 하지만 객체를 만드는 코드를 직접 작성하지 않는다는 것은, 그 과정을 컴파일러가 모두 검증해 줄 수도 없다는 의미다.

### 4.1 컴파일 시점에 발견하던 오류가 런타임으로 이동한다

직접 객체를 생성한다면 다음과 같이 작성할 수 있다.

```txt
val repository = ProductRepository()
```

생성자의 파라미터가 변경되거나 잘못된 타입의 인자를 전달하면 해당 코드에서 컴파일 오류가 발생할 수 있다. 반면 리플렉션 기반 컨테이너는 런타임에 생성자를 찾는다. 실제 구현에는 다음과 같은 코드가 존재한다.

```txt
val constructor = type.primaryConstructor
    ?: throw IllegalArgumentException("No primary constructor for ${type.qualifiedName}")
```

이 경로에서 주 생성자가 없는 타입을 요청하면 <code>No primary constructor for ...</code> 뒤에 요청한 타입 이름이 붙는다. 이는 소스 코드에서 확인한 메시지 형식이며, 앱에서 오류를 재현해 수집한 로그는 아니다.

반면 2.1절의 ViewModel 생성 경로는 메시지 없는 <code>IllegalArgumentException()</code>을 던진다. 실패한 타입을 알기 쉽도록 개선한다면 다음처럼 생성자 조회와 호출을 나눌 수 있다. 아래 코드는 개선 제안이며 현재 구현에는 적용하지 않았다.

```txt
val constructor = modelClass.kotlin.primaryConstructor
    ?: throw IllegalArgumentException(
        "No primary constructor for ${modelClass.name}",
    )
val viewModel = constructor.call()
```

이 메시지는 주 생성자가 없는 경우만 설명한다. 인자가 필요한 생성자에 빈 인자 목록으로 <code>call()</code>을 실행하면 호출 단계에서 실패하므로, 해당 오류는 별도로 구분해야 한다. Kotlin의 <code>KCallable.call</code> 문서도 인자의 수나 타입이 맞지 않으면 예외를 던진다고 명시한다.

반환문의 <code>modelClass.cast(viewModel)!!</code>는 원본을 유지했다. Java [Class.cast()](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/Class.html#cast(java.lang.Object))는 입력이 <code>null</code>이면 <code>null</code>을 반환하므로 항상 non-null을 보장하는 API는 아니다. 다만 이 경로의 <code>viewModel</code>은 앞선 Elvis 연산자에서 null이 배제되어 있다. <code>!!</code> 정리는 이 조건을 근거로 별도 구현 변경에서 검토할 수 있다.

또한 생성자 파라미터를 해결하지 못하거나, 필요한 구현체가 등록되지 않았거나, 동적 호출에 전달한 인자가 올바르지 않더라도 실행 과정에서 실패할 수 있다. 즉, 의존성 관계의 일부 오류는 해당 객체의 생성 경로가 실제로 실행돼야 발견된다. 작은 프로젝트에서는 큰 문제가 아닐 수도 있다. 하지만 객체의 수와 의존성 관계가 복잡해질수록, 이런 오류를 언제 발견할 수 있는지는 중요한 차이가 된다.

### 4.2 의존성 관계를 코드만 보고 파악하기 어려워진다

현재 ViewModel은 필드 주입을 사용한다. 이 방식은 생성자 파라미터 없이 ViewModel을 만들 수 있다는 장점이 있다. 하지만 생성자만 보고 어떤 의존성이 필요한지 알기는 어렵다. 개발자는 클래스 내부에서 <code>@Inject</code>가 붙은 필드를 찾아야 한다. 또한 <code>lateinit</code>으로 선언된 필드가 필요한 시점까지 주입되지 않는다면 초기화되지 않은 상태로 접근하는 오류가 발생할 수 있다.

생성자 주입을 사용한다면 필수 의존성을 생성자에 명시할 수 있다. 따라서 새로운 DI 구조를 설계한다면 생성자 주입을 우선 고려할 이유가 있다. 여기서 필드 주입 자체가 항상 잘못됐다는 결론을 내리려는 것은 아니다. 현재 구현에서는 공통 Factory와 필드 주입을 통해 원하는 동작을 구현했다. 다만 반복 코드를 줄이는 편리함과 의존성을 명시적으로 드러내는 장점 사이에 트레이드오프가 존재한다는 것을 확인할 수 있었다.

---

## 5. 리플렉션은 정확히 어디에서 비용이 발생할까?

이제 처음 궁금했던 성능 문제로 돌아가 보자. 리플렉션은 느리다고 알려져 있지만, 단순히 <code>call()</code>을 사용한다는 이유만으로 실제 앱에서 어느 정도의 성능 문제가 생기는지 알 수는 없다. 비용을 이해하려면 먼저 컨테이너에서 어떤 작업이 실행되는지 구분해야 한다.

### 5.1 Kotlin Reflection: 생성자와 파라미터 조회

첫 번째는 클래스와 생성자 정보를 조회하는 과정이다.

```txt
val constructor = type.primaryConstructor
val parameters = constructor?.parameters
```

직접 생성 코드를 작성하는 대신, 런타임에 클래스의 생성자와 파라미터 정보를 확인한다. 실제 소요 시간은 Reflection 구현, 메타데이터 캐싱 여부, 실행 환경에 따라 달라질 수 있다. 따라서 <code>primaryConstructor</code>를 호출할 때마다 클래스 정보를 처음부터 전부 분석한다고 가정해서는 안 된다. 하지만 적어도 직접 생성자 호출과는 다른 정보 조회 경로를 사용한다는 사실은 분명하다.

### 5.2 Kotlin Reflection: 생성자 동적 호출

두 번째는 런타임에 찾아낸 생성자를 호출하는 과정이다.

```txt
constructor.call(*arguments.toTypedArray())
```

직접 호출은 컴파일 시점에 호출 대상이 정해진다. 반면 이 코드는 <code>KFunction</code>을 통해 대상 생성자를 동적으로 호출한다. 호출 과정에서는 전달된 인자를 처리하고 실제 생성자를 실행해야 한다. 이런 추가 경로가 직접 호출과 비교해 어느 정도 차이를 만드는지는 측정이 필요하다.

### 5.3 Java Reflection: 필드 탐색과 주입

세 번째는 필드 주입 과정에서 발생한다.

```txt
instance::class.java.declaredFields
```

이 코드로 필드 정보를 가져온 뒤, 애너테이션을 검사한다.

```txt
field.isAnnotationPresent(Inject::class.java)
```

주입 대상이라면 필요한 의존성을 구하고 필드에 값을 대입한다.

```txt
field.isAccessible = true
field.set(instance, dependency)
```

따라서 실제 컨테이너의 작업은 단순한 생성자 호출 하나로 끝나지 않는다. 생성자 탐색, 파라미터 확인, 의존성 해결, 필드 탐색, 애너테이션 검사, 필드 대입까지 여러 단계가 이어진다. Kotlin Reflection의 생성자 조회와 호출, Java Reflection의 필드 탐색과 대입은 사용하는 API와 작업이 다르다. 어느 쪽이 더 큰 비용인지 현재 분석만으로 판단할 수 없으므로, 6장에서도 두 경로를 나눠 측정할 계획이다.

### 5.4 하지만 모든 화면 갱신에서 이 비용을 지불하는 것은 아니다

<b>추가 비용이 존재하는 것과 그 비용이 실제 앱 성능에 큰 영향을 미치는 것은 다른 문제다.</b> 현재 컨테이너는 생성한 객체를 스코프에 저장하고 재사용한다. 이미 등록된 의존성을 요청하면 생성자를 다시 호출하지 않고 기존 인스턴스를 반환할 수 있다. 또한 Compose에서 <code>viewModel()</code>을 사용한다고 해서 매번 새로운 ViewModel을 생성하는 것도 아니다.

기존 ViewModel이 유지되는 동안에는 해당 인스턴스를 재사용하므로, 일반적인 recomposition이 발생할 때마다 ViewModel 생성과 필드 주입을 반복하는 구조는 아니다. 따라서 리플렉션을 사용하는 위치와 실행 빈도를 함께 살펴봐야 한다. 한 가지 더 구분할 점이 있다. 현재 컨테이너가 수행하는 인스턴스 캐싱과 리플렉션 메타데이터 캐싱은 서로 다르다. 인스턴스 캐싱은 이미 생성한 객체를 재사용하는 것이다.

메타데이터 캐싱은 조회한 생성자나 필드 정보를 다시 활용하는 것이다. 현재 구현의 스코프 캐싱만으로 모든 리플렉션 탐색 결과가 별도로 캐싱된다고 말할 수는 없다. 이는 추후 성능 실험에서 구분해 확인해 볼 부분이다.

---

## 6. 그렇다면 성능은 어떻게 비교해야 할까?

현재까지의 분석으로 확인한 것은 리플렉션 기반 컨테이너가 직접 생성보다 많은 런타임 작업을 수행한다는 점이다. 하지만 아직 실행 시간이나 메모리 할당량을 측정하지 않았다. 따라서 현재 단계에서 '몇 배 느리다'거나 '실제 앱에서 체감할 수 있을 정도로 느리다'고 결론 내리는 것은 적절하지 않다. 성능을 확인하려면 비교 대상을 명확하게 분리해야 한다.

### 6.1 첫 번째 실험: 객체 생성 자체를 비교하기

1장의 <code>User</code>를 대상으로 Kotlin Reflection의 생성자 조회와 호출을 분리한다. 아래는 Jetpack Microbenchmark에서 비교할 측정 구간을 보여 주는 설계 예시이며, 아직 실행한 벤치마크는 아니다. <code>BenchmarkRule</code>과 테스트 실행 환경을 준비했다고 가정한다. 테스트 클래스 내부의 코드이며 import와 러너 설정은 생략했다.

```txt
@get:Rule
val benchmarkRule = BenchmarkRule()

private var result: User? = null

@Test
fun directCreation() {
    // A. 직접 생성
    benchmarkRule.measureRepeated {
        result = User("조디악", 25)
    }
    assertEquals(User("조디악", 25), result)
}

@Test
fun cachedConstructorCall() {
    // B. 조회는 측정 전에 끝내고 호출부터 측정
    val constructor = User::class.primaryConstructor!!
    benchmarkRule.measureRepeated {
        result = constructor.call("조디악", 25)
    }
    assertEquals(User("조디악", 25), result)
}

@Test
fun lookupAndCall() {
    // C. 생성자 조회 API 호출까지 측정에 포함
    benchmarkRule.measureRepeated {
        result = User::class.primaryConstructor!!.call("조디악", 25)
    }
    assertEquals(User("조디악", 25), result)
}
```

<code>measureRepeated</code> 블록이 측정 경계를 드러낸다. 세 경우 모두 결과를 같은 필드에 저장하고 측정 후 검증한다. 실제 실행 전에는 생성 코드가 최적화로 제거되지 않는지도 확인해야 한다. B는 생성자를 미리 조회하므로 프로세스 최초 조회 비용을 포함하지 않는다. C도 블록을 반복 실행하므로 매번 새로운 메타데이터 분석이 일어난다고 볼 수 없다. 이 비교로 알 수 있는 것은 반복 실행 조건에서 조회 API를 포함했을 때의 차이다.

처음 한 번의 실행과 반복 실행은 별도 축으로 다뤄야 한다. [Android 공식 문서](https://developer.android.com/topic/performance/benchmarking/microbenchmark-overview)에 따르면 Microbenchmark는 워밍업을 수행하고 코드를 반복 실행하므로, 첫 호출과 반복 호출의 동작이 다른 작업은 측정에 주의해야 한다. 위 예시의 결과를 프로세스 최초 호출의 비용으로 해석해서는 안 된다.

ViewModel처럼 생성 빈도가 낮은 객체는 앱 실행 후 해당 경로에 처음 진입하는 상황도 확인할 필요가 있다. 이때는 새 프로세스에서 대상 경로에 진입하기 전 관련 리플렉션 API를 호출하지 않았는지 확인하고, 클래스 로딩과 앱 초기화 등 함께 포함된 작업을 기록해야 한다. 어느 조건이 앱에 더 중요한지는 실제 실행 빈도와 측정 결과로 판단한다. <code>kotlin-reflect</code>의 내부 캐시가 항상 어떤 비용을 없앤다는 주장은 이 글에서는 하지 않는다.

### 6.2 두 번째 실험: 실제 DI 컨테이너를 비교하기

단순 객체 생성에서 차이가 나타나더라도 실제 DI 컨테이너에서 같은 결과가 나온다고 단정할 수는 없다. 컨테이너에는 의존성 탐색과 캐싱, 필드 주입 등이 포함되기 때문이다. 따라서 동일한 의존성 그래프를 대상으로 다음을 비교해 보고 싶다.

| 비교 대상 | 측정하려는 작업 |
| --- | --- |
| 직접 생성 Factory | 생성자 호출과 의존성 직접 연결 |
| 리플렉션 DI 컨테이너 | 생성자 탐색, 재귀적 의존성 해결, 필드 주입 |
| 이미 생성된 의존성 조회 | 스코프에서 기존 객체 반환 |

컨테이너 전체 비교에 앞서 Java Reflection 경로도 별도로 구분한다. 동일한 주입 대상과 미리 준비한 의존성을 사용해, 조회해 둔 <code>Field</code>의 <code>set()</code> 호출과 <code>declaredFields</code>, 애너테이션 검사, 접근 설정, 대입을 모두 포함한 경로를 비교할 수 있다. 이때 의존성 생성은 측정 구간 밖에 둬서 필드 처리와 생성자 처리 비용이 섞이지 않도록 한다.

컨테이너 비교에서는 매번 새 ViewModel 스코프에서 <code>ProductRepository</code>를 생성하는 경우와 같은 스코프에서 기존 인스턴스를 찾는 경우를 나눈다. <code>@RoomCart CartRepository</code>는 앱 스코프에 이미 등록되어 있다는 조건도 맞춰야 한다. 새 스코프를 만들더라도 프로세스의 클래스 정보까지 초기 상태로 돌아가는 것은 아니다. 인스턴스의 최초 생성과 프로세스에서 리플렉션을 처음 사용하는 상황을 구별해야 한다.

필요하다면 리플렉션 메타데이터를 별도로 캐싱한 구현도 추가해 비교할 수 있을 것이다.

### 6.3 측정 환경도 결과의 일부다

아주 짧은 객체 생성 코드를 단순히 반복 실행해 평균을 계산하는 것만으로는 신뢰할 만한 결과를 얻기 어렵다. JVM 또는 Android 런타임의 최적화, 워밍업 상태, 가비지 컬렉션, 메모리 할당 등이 측정에 영향을 줄 수 있기 때문이다. 또한 측정하려던 객체 생성이 컴파일러나 런타임에 의해 최적화되어 결과가 왜곡되지 않도록 해야 한다.

Android 환경에서는 작은 코드 구간을 반복 측정할 수 있는 Jetpack Benchmark의 Microbenchmark 도구를 검토할 수 있다. 측정 시에는 다음 조건을 기록할 필요가 있다.

- 동일한 데이터와 의존성 구조
- 동일한 기기 및 실행 환경
- 동일한 빌드 조건
- 워밍업과 실제 측정 구간
- 반복 측정 결과와 분산
- 실행 시간과 필요시 메모리 할당량

그리고 실제 앱에 미치는 영향을 판단하려면 단순 함수 실행 시간뿐 아니라 ViewModel 생성이 사용자 경험에서 차지하는 비중도 확인해야 한다. 수치 차이가 존재하더라도 실행 빈도가 낮고 전체 작업 시간에서 차지하는 비중이 작다면 앱 수준의 영향은 제한적일 수 있다. 반대로 생성이 빈번하거나 초기화 과정에서 많은 객체를 만들어야 한다면 추가 작업이 더 중요해질 수 있다. 현재는 어느 쪽인지 단정할 측정 결과가 없다.

---

## 7. 처음 궁금했던 Serializable과 Parcelable은 어떻게 다를까?

여기까지 살펴보니 처음 접했던 설명을 다시 생각하게 됐다.

> Serializable은 리플렉션을 사용하기 때문에 느리다.

이 문장은 리플렉션이 비용을 발생시킬 수 있다는 점을 알려주지만, 두 직렬화 방식의 성능 차이를 설명하기에는 지나치게 단순하다. 왜냐하면 <code>Serializable</code>과 <code>Parcelable</code>은 리플렉션 사용 여부만 다른 기술이 아니기 때문이다.

### 7.1 Serializable은 객체 그래프를 직렬화한다

Java의 기본 직렬화 메커니즘은 <code>ObjectOutputStream</code>과 <code>ObjectInputStream</code>을 사용한다. 객체를 직렬화할 때 클래스에 대한 설명 정보와 직렬화 대상 필드, 참조된 객체 등을 정해진 스트림 형식으로 기록한다. 역직렬화할 때는 그 정보를 해석해 객체 그래프를 복원한다.

이 과정에서 런타임의 클래스와 필드 정보를 다루는 작업이 필요하며, 기본 직렬화 메커니즘의 비용은 이러한 정보 처리뿐 아니라 객체 그래프를 탐색하고 스트림을 읽고 쓰는 과정에서도 발생한다. 따라서 Serializable의 성능을 단순히 리플렉션 호출 비용 하나로 환원하기는 어렵다.

### 7.2 Parcelable은 Android의 Parcel 형식을 사용한다

<code>Parcelable</code>은 Android의 <code>Parcel</code> 형식으로 객체의 데이터를 기록하고 복원하기 위한 인터페이스다. Activity 간 데이터 전달이나 Binder IPC 등 Android의 데이터 전달 경로에서 사용된다. 개발자가 <code>Parcelable</code> 구현을 직접 작성할 수도 있지만, Kotlin에서는 <code>@Parcelize</code>를 활용할 수 있다.

```txt
@Parcelize
data class User(
    val name: String,
    val age: Int,
) : Parcelable
```

<code>kotlin-parcelize</code> 플러그인은 컴파일 과정에서 필요한 <code>Parcelable</code> 구현을 생성한다. 덕분에 직렬화 로직을 매번 직접 작성하지 않아도 된다. 다만 <code>Parcelable</code>을 사용했다고 해서 모든 상황에서 리플렉션이 완전히 배제된다거나, 어떤 데이터든 항상 더 빠르다고 단정할 수는 없다. 실제로 처리하는 타입과 직렬화 경로에 따라 동작 방식은 달라질 수 있다.

### 7.3 결국 비교하려는 비용이 다르다

두 실험은 질문과 측정 대상이 다르다.

| 질문 | 측정 대상 |
| --- | --- |
| 리플렉션을 통한 객체 생성은 직접 생성과 얼마나 다를까? | 생성자 조회와 <code>call()</code>을 포함한 생성 경로 |
| Serializable과 Parcelable은 Android 데이터 전달에서 얼마나 다를까? | 같은 데이터의 직렬화와 역직렬화 전체 시간, 데이터 크기, 메모리 할당량 |

첫 번째 실험에서 리플렉션의 추가 비용이 확인되더라도, 그 수치만으로 두 번째 질문에 대한 답을 얻을 수는 없다. 서로 다른 처리 과정이 포함되어 있기 때문이다. 결국 처음 들었던 설명은 더 정확하게 표현할 수 있다. Serializable과 Parcelable은 서로 다른 직렬화 메커니즘을 사용하며, 리플렉션을 포함한 런타임 정보 처리 비용은 그 차이를 구성하는 여러 요소 중 하나다.

실제 성능을 판단하려면 사용하려는 객체와 데이터 전달 경로를 기준으로 측정해야 한다.

---

## 8. 코드 생성은 책임을 언제 수행할까?

리플렉션 컨테이너는 런타임에 타입을 읽고 생성 경로를 결정한다. 반면 Dagger/Hilt는 빌드 시 의존성 관계를 분석하고 생성 코드를 준비한다. KSP는 선언 분석과 코드 생성을 지원하는 도구이며, 그 자체가 DI 컨테이너는 아니다.

| 방식 | 생성 경로를 결정하는 곳 | 고려할 점 |
| --- | --- | --- |
| 직접 생성 | 개발자가 작성한 호출 코드 | 명시적이지만 생성과 연결 코드가 반복될 수 있음 |
| 리플렉션 기반 DI | 런타임의 타입 정보 탐색 | 공통 생성 규칙을 적용할 수 있지만 동적 탐색과 오류 처리가 필요함 |
| 코드 생성 기반 DI | 빌드 시 분석과 생성 코드 | 의존성 오류를 빌드 시 검출할 수 있지만 프로세서와 빌드 설정을 관리해야 함 |

코드 생성도 런타임의 객체 생성 자체를 없애지는 않는다. 객체를 어떻게 조립할지 판단하는 작업의 일부를 빌드 시점으로 옮긴다. 각 프로세서의 구현과 Hilt 도입 방법은 이 글의 컨테이너 분석 범위를 넘어가므로 여기서는 시점의 차이만 비교한다.

---

## 9. 실제 프로젝트에 적용할 때 확인해야 할 것들

직접 DI 컨테이너를 구현하면서 리플렉션으로 많은 작업을 일반화할 수 있다는 점을 확인했다. 하지만 일반화된 로직이 모든 타입을 안전하게 처리할 수 있다는 의미는 아니다. 현재 구현의 한계도 함께 살펴볼 필요가 있다.

### 9.1 모든 생성자 구조를 지원하지는 않는다

현재 컨테이너는 다음 코드를 사용한다.

```txt
val constructor = type.primaryConstructor
    ?: throw IllegalArgumentException("No primary constructor for ${type.qualifiedName}")
```

따라서 주 생성자가 없는 타입은 이 경로에서 생성하지 못한다. 또한 생성자 파라미터를 전부 <code>resolve()</code>한 뒤 <code>call()</code>에 전달한다. Kotlin은 기본 인자가 있는 함수를 호출할 때 일부 인자를 생략할 수 있지만, 현재 구현은 그러한 생략을 별도로 처리하지 않는다.

Kotlin Reflection에는 선택적 파라미터를 다룰 수 있는 <code>callBy()</code> API도 있지만, 현재 컨테이너가 사용하는 방식과는 다르다.

### 9.2 제네릭과 qualifier 처리에도 범위가 있다

현재 구현은 다음과 같이 파라미터 타입을 얻는다.

```txt
val dependencyType =
    parameter.type.classifier as? KClass<*>
        ?: throw IllegalArgumentException("Unsupported parameter: ${parameter.name}")
```

하지만 <code>List<Product></code>처럼 제네릭 타입의 인자까지 구별해야 하는 경우에는 외부 클래스가 <code>List</code>라는 사실만으로 충분하지 않다. <code>Product</code>라는 타입 인자 정보도 함께 고려해야 한다. 또한 현재 구현에서는 필드 주입 시 qualifier를 확인하지만, 생성자 파라미터의 qualifier까지 해석하는 구조는 아니다.

따라서 생성자 파라미터에 동일한 인터페이스의 여러 구현체가 필요한 경우를 처리하려면 추가적인 설계가 필요하다. 재귀적 의존성 해결 과정에서 순환 참조를 어떻게 감지하고 실패를 처리할지도 컨테이너를 확장할 때 확인해야 할 문제다.

### 9.3 배포 빌드에서 확인할 범위

R8은 리플렉션으로 간접 접근하는 코드의 사용 관계를 모두 파악하지 못할 수 있다. 따라서 최적화된 배포 빌드에서도 생성자 조회와 필드 주입을 확인하고, 필요한 클래스와 멤버에 한정한 keep 규칙을 검토해야 한다. 전체 Kotlin Reflection 기능을 제공하는 <code>kotlin-reflect</code> 라이브러리의 최종 앱 크기 영향도 별도 확인 대상이다.

현재 프로젝트에서 R8 관련 장애를 재현하거나 크기 변화를 측정한 것은 아니다. 구체적인 keep 규칙과 바이너리 크기 분석은 별도 검증 과제로 남긴다.

---

## 10. 그래서 언제 리플렉션을 선택해야 할까?

호출할 타입과 생성 로직이 단순하다면 직접 생성으로도 충분하다. 의존성 관계가 정적이면서 반복적인 연결 코드를 줄이고 싶다면 코드 생성 기반 DI를 검토할 수 있다. 실행 중에 결정되는 타입의 구조를 읽어야 하는 범용 라이브러리라면 리플렉션이 유용할 수 있다.

직접 만든 컨테이너에서 리플렉션을 사용한 목적은 ViewModel마다 반복되는 Factory를 줄이고 생성과 주입 규칙을 공통화하는 것이었다. 이 선택을 평가하려면 줄어든 코드뿐 아니라 컨테이너가 맡게 된 런타임 탐색, 타입 검사, 예외 처리도 함께 봐야 한다.

---

## 마무리: 코드를 줄인 만큼, 누가 그 일을 대신하고 있을까?

처음에는 리플렉션이 느리다는 설명을 이해하고 싶었다. 실제 코드를 따라가며 확인한 것은 Kotlin Reflection이 생성자를 조회하고 호출하며, Java Reflection이 주입할 필드를 찾고 값을 대입한다는 점이다. 스코프는 이와 별개로 생성한 인스턴스의 재사용 범위를 관리한다.

<b>반복되는 생성과 연결 코드가 줄어든 만큼, 컨테이너가 런타임에 그 일을 대신 수행했다.</b> 코드 생성 방식은 그 판단의 일부를 빌드 과정으로 옮긴다. 어느 방식이 적합한지는 필요한 유연함과 오류를 발견할 시점, 실행 빈도를 기준으로 선택해야 한다.

추가 작업 경로가 있다는 사실만으로 현재 앱이 얼마나 느려지는지 알 수는 없다. 다음 검증에서는 Kotlin 생성자 조회와 호출, Java 필드 처리, 실제 컨테이너의 생성과 재사용을 구분하고 프로세스 최초 실행과 반복 실행의 조건도 기록하려 한다. 이 글은 그 측정 전에, 누가 어떤 작업을 대신하는지 확인한 기록이다.

---

## 참고 자료

- [Kotlin 공식 문서 - Reflection](https://kotlinlang.org/docs/reflection.html)
- [Kotlin API - KClass.primaryConstructor](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.reflect.full/primary-constructor.html)
- [Kotlin API - KCallable.call / callBy](https://kotlinlang.org/api/core/kotlin-stdlib/kotlin.reflect/-k-callable/)
- [Kotlin 공식 문서 - Kotlin Symbol Processing](https://kotlinlang.org/docs/ksp-overview.html)
- [Dagger 공식 문서 - Hilt Quick Start](https://dagger.dev/hilt/quick-start)
- [Android Developers - Parcelable Implementation Generator](https://developer.android.com/kotlin/parcelize)
- [Oracle - Java Object Serialization Specification](https://docs.oracle.com/en/java/javase/17/docs/specs/serialization/serial-arch.html)
- [Android Developers - Microbenchmark](https://developer.android.com/topic/performance/benchmarking/microbenchmark-overview)
- [Android Developers - Write a Microbenchmark](https://developer.android.com/topic/performance/benchmarking/microbenchmark-write)
- [Kotlin API - KCallable.call](https://kotlinlang.org/api/core/kotlin-stdlib/kotlin.reflect/-k-callable/call.html)
- [Oracle - Class.cast](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/Class.html#cast(java.lang.Object))
- [Android Developers - R8 Keep Rules](https://developer.android.com/topic/performance/app-optimization/keep-rules-overview)

<p />
