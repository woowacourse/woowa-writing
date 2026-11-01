# 리플렉션은 정말 느릴까? 직접 만든 DI 컨테이너에서 찾아본 유연함의 비용

Android에서 <code>Serializable</code>과 <code>Parcelable</code>의 차이를 공부하다가 익숙한 설명 하나를 발견했다.

> Serializable은 리플렉션을 사용하기 때문에 Parcelable보다 느리다.

처음에는 별다른 의문 없이 받아들였다. 그런데 조금 더 생각해 보니 궁금한 점이 생겼다.

<b>리플렉션의 정확히 어떤 작업이 느리다는 걸까?</b>

클래스 정보를 찾아보는 과정일까? 생성자를 호출하는 과정일까? 아니면 객체의 필드에 접근하는 과정일까?

그리고 그 차이는 실제 애플리케이션에서도 문제가 될 만큼 클까?

마침 직접 구현한 DI(Dependency Injection) 컨테이너에서도 리플렉션을 사용하고 있었다. ViewModel마다 Factory를 작성하지 않아도 필요한 의존성을 자동으로 연결해 주는 구조였다.

구현할 때는 반복되는 코드가 줄어들어 편리하다고 생각했다. 하지만 컨테이너가 그 일을 어떻게 대신 수행하는지, 그리고 그 과정에서 어떤 비용을 치르는지는 깊이 살펴보지 않았다.

이번 기회에 직접 만든 DI 컨테이너의 객체 생성 과정을 따라가며 다음 질문에 답해 보려고 한다.

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

평범한 Kotlin 코드다.

하지만 이 한 줄에는 중요한 전제가 있다. <b>컴파일 시점에 어떤 생성자를 호출할지 이미 알고 있다는 것</b>이다.

컴파일러는 <code>User</code>의 생성자와 파라미터를 확인한다. <code>name</code>에 문자열을 전달했는지, <code>age</code>에 정수를 전달했는지 검사하고, 잘못된 인자를 전달하면 컴파일 과정에서 오류를 발견할 수 있다.

그런데 어떤 클래스를 생성할지 실행 중에 결정해야 한다면 어떨까?

예를 들어 전달받은 타입에 따라 서로 다른 객체를 만들어야 하는 범용 라이브러리를 개발한다고 생각해 보자.

모든 클래스에 대해 <code>User(...)</code>, <code>ProductRepository(...)</code>처럼 구체적인 생성 코드를 작성할 수는 있지만, 라이브러리가 처리할 타입이 늘어날 때마다 생성 로직도 수정해야 한다.

리플렉션은 이런 상황에서 다른 접근 방식을 제공한다.

### 1.1 런타임에 생성자를 찾아 호출하기

앞선 <code>User</code> 객체를 Kotlin Reflection으로 생성해 보자.

```txt
import kotlin.reflect.full.primaryConstructor

val constructor = User::class.primaryConstructor
    ?: error("User의 주 생성자를 찾을 수 없습니다.")

val user = constructor.call("조디악", 25)
```

결과적으로 만들어지는 객체는 동일하다. 하지만 객체를 생성하는 과정은 다르다.

먼저 <code>User::class</code>를 통해 <code>User</code> 클래스의 정보를 나타내는 <code>KClass</code>를 얻는다.

<code>KClass</code>는 Kotlin에서 클래스에 관한 런타임 정보를 다루는 인터페이스다. 이를 통해 생성자, 프로퍼티, 함수 등에 관한 정보를 확인할 수 있다.

그다음 <code>primaryConstructor</code>를 사용해 주 생성자를 가져온다.

```txt
val constructor = User::class.primaryConstructor
```

여기서 얻은 생성자는 <code>KFunction</code>으로 표현된다. <code>KFunction</code>은 호출 가능한 함수를 나타내며, 생성자도 호출 가능한 대상으로 다룰 수 있다.

마지막으로 <code>call()</code>에 인자를 전달해 생성자를 실행한다.

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

여기서 주목할 부분은 리플렉션이 새로운 종류의 객체를 만드는 기술이 아니라는 점이다.

<b>리플렉션은 객체를 생성하는 방법을 런타임에 결정할 수 있도록 해 준다.</b>

다만 그만큼 컴파일러가 직접 확인하던 일부 책임이 실행 시점으로 옮겨간다.

가령 <code>call()</code>에 전달한 인자의 개수나 타입이 생성자와 맞지 않으면 런타임에 예외가 발생한다.

그렇다면 의문이 생긴다.

이렇게 복잡하게 객체를 생성할 필요가 있을까?

<code>User</code> 하나만 생성한다면 굳이 리플렉션을 사용할 이유가 없다. 직접 생성하는 편이 간단하고 명확하다.

하지만 수십 개의 서로 다른 클래스를 공통 규칙으로 생성해야 한다면 이야기가 달라진다.

<b>리플렉션의 진짜 가치는 개별 객체를 쉽게 만드는 데 있는 것이 아니라, 서로 다른 객체를 만드는 과정을 일반화할 수 있다는 데 있다.</b>

이 특성이 직접 구현한 DI 컨테이너에서 활용됐다.

---

## 2. 직접 만든 DI 컨테이너는 리플렉션을 어떻게 사용했을까?

DI는 객체가 필요로 하는 의존성을 객체 내부에서 직접 생성하는 대신 외부에서 제공하는 방식이다.

예를 들어 <code>ProductsViewModel</code>이 <code>ProductRepository</code>를 사용한다면, ViewModel이 Repository의 생성과 관리까지 책임지지 않도록 역할을 분리할 수 있다.

그런데 의존성을 외부에서 제공하려면 결국 어딘가에서는 객체를 생성하고 연결해야 한다.

이 작업을 담당하도록 만든 것이 <code>DependencyContainer</code>다.

현재 프로젝트에서는 Compose 화면에서 다음과 같이 ViewModel을 요청한다.

```txt
val viewModel: ProductsViewModel =
    viewModel(factory = DependencyContainer)
```

화면에서 <code>ProductRepository</code>나 <code>CartRepository</code>의 생성 코드를 직접 작성하지 않는다.

대신 <code>DependencyContainer</code>가 ViewModel을 만들고, 필요한 의존성을 찾아 연결한다.

여기에는 크게 세 단계가 있다.

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

Factory는 새로운 <code>DependencyScope</code>를 만들고, 해당 스코프를 사용해 ViewModel을 생성한다.

생성한 스코프는 <code>addCloseable()</code>을 통해 ViewModel의 생명주기와 연결한다.

즉, ViewModel에 필요한 의존성을 관리할 범위를 준비한 뒤 실제 객체 생성을 진행하는 것이다.

핵심은 다음 <code>create()</code> 함수에 있다.

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

가장 중요한 부분은 이 코드다.

```txt
modelClass.kotlin.primaryConstructor?.call()
```

<code>modelClass</code>는 생성할 ViewModel의 클래스 정보를 담은 Java <code>Class</code> 객체다.

여기서 <code>.kotlin</code>으로 Kotlin의 <code>KClass</code>를 얻고, <code>primaryConstructor</code>로 주 생성자를 조회한 뒤 <code>call()</code>로 실행한다.

앞서 <code>User</code> 객체를 만들었던 방법과 같다.

차이가 있다면 실제 ViewModel에서는 생성자에 인자를 전달하지 않는다는 것이다.

현재 구현은 ViewModel의 의존성을 생성자 파라미터가 아닌 필드로 주입하는 방식을 사용한다. 따라서 인자 없는 주 생성자를 호출해 ViewModel을 먼저 만든다.

이후 다음 함수를 실행한다.

```txt
injectFields(viewModel, scope)
```

여기서부터 의존성 주입이 시작된다.

### 2.2 두 번째 단계: @Inject가 붙은 필드 찾기

객체를 생성했다고 해서 필요한 의존성이 자동으로 연결되는 것은 아니다.

컨테이너는 어떤 필드에 의존성을 주입해야 하는지 알아야 한다.

현재 구현에서는 <code>@Inject</code> 애너테이션을 그 기준으로 사용한다.

<code>ProductsViewModel</code>은 <code>ProductRepository</code>와 <code>CartRepository</code>를 필드 의존성으로 사용한다. 그중 <code>CartRepository</code>에는 <code>@RoomCart</code> qualifier가 지정돼 있다.

qualifier는 동일한 타입에 여러 구현체가 연결될 수 있을 때 어떤 구현체를 사용할지 구분하기 위한 정보다.

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

여기에서는 Kotlin Reflection이 아니라 Java Reflection을 사용한다.

<code>instance::class.java</code>는 객체의 Java <code>Class</code> 정보를 가져온다.

그리고 <code>declaredFields</code>는 해당 클래스에 선언된 필드들을 반환한다.

컨테이너는 각 필드를 순회하면서 다음 조건을 검사한다.

```txt
field.isAnnotationPresent(Inject::class.java)
```

필드에 <code>@Inject</code> 애너테이션이 있다면 의존성 주입 대상으로 판단한다.

즉, 개발자가 매번 어떤 필드에 값을 대입해야 하는지 작성하지 않아도, 컨테이너가 애너테이션을 기준으로 주입 대상을 찾아낼 수 있다.

이것이 리플렉션을 사용해 얻은 첫 번째 편리함이다.

다만 <code>declaredFields</code>는 해당 클래스에 직접 선언된 필드를 조회한다. 상속받은 필드까지 모두 탐색하는 API는 아니다. 따라서 상속 구조까지 지원하려면 별도의 탐색 로직을 고려해야 한다.

### 2.3 세 번째 단계: 필요한 객체를 찾아 필드에 넣기

주입할 필드를 찾았다면 이제 그 필드에 들어갈 객체를 구해야 한다.

실제 구현의 핵심은 다음과 같다.

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

여기서는 두 종류의 작업이 일어난다.

먼저 <code>field.type.kotlin</code>을 통해 해당 필드가 요구하는 타입을 확인하고, <code>resolve()</code>로 그 타입에 맞는 의존성을 구한다.

그다음 필드에 값을 대입한다.

```txt
field.isAccessible = true
field.set(instance, dependency)
```

<code>isAccessible = true</code>는 Java Reflection을 통해 해당 필드의 접근 검사를 우회할 수 있도록 설정하는 부분이다.

이후 <code>field.set()</code>으로 객체의 필드에 의존성을 대입한다.

평소 Kotlin에서는 다음과 같이 작성할 작업이다.

```txt
viewModel.repository = repository
```

하지만 리플렉션을 사용하면 필드 이름을 직접 알지 못해도, 런타임에 찾은 <code>Field</code> 객체를 통해 값을 대입할 수 있다.

이 덕분에 컨테이너는 ViewModel마다 별도의 대입 코드를 작성하지 않고도 공통 주입 로직을 적용할 수 있다.

물론 여기에도 대가가 있다.

컴파일러가 확인하는 일반적인 프로퍼티 대입과 달리, 동적으로 찾은 필드에 대입하는 값이 올바른지 런타임에 검증해야 한다.

또한 필드의 접근 제어를 우회하는 코드이므로, 객체의 내부 구현에 대한 의존성도 생긴다.

지금까지 살펴본 내용을 요약하면 다음과 같다.

<b>Kotlin Reflection은 생성자를 찾고 호출하는 데, Java Reflection은 필드와 애너테이션을 탐색하고 값을 대입하는 데 사용했다.</b>

그런데 아직 한 가지 중요한 질문이 남았다.

<code>resolve()</code>는 필요한 의존성을 어떻게 만들어 낼까?

---

## 3. 의존성의 의존성은 어떻게 해결할까?

ViewModel에 <code>ProductRepository</code>가 필요하다는 사실을 알았다고 하자.

이때 <code>ProductRepository</code>가 이미 만들어져 있다면 그대로 가져오면 된다.

하지만 아직 없다면 직접 생성해야 한다.

그리고 <code>ProductRepository</code> 역시 다른 객체를 필요로 할 수 있다.

예를 들어 다음과 같은 관계가 있다고 가정해 보자.

```txt
ProductsViewModel
    └── ProductRepository
            └── ProductDataSource
                    └── ProductApi
```

이 예시는 의존성 해결 과정을 설명하기 위한 가상의 객체 구조다.

<code>ProductsViewModel</code>을 만들려면 <code>ProductRepository</code>가 필요하고, <code>ProductRepository</code>를 만들려면 <code>ProductDataSource</code>가 필요하다.

그러면 컨테이너는 의존성을 한 단계씩 따라가면서 필요한 객체를 생성해야 한다.

현재 구현에서는 이 과정을 재귀적인 <code>resolve()</code> 호출로 처리한다.

### 3.1 생성자 파라미터를 따라가는 재귀적 탐색

실제 <code>resolve()</code>에서 객체를 생성하는 핵심 부분을 추리면 다음과 같다.

```txt
val constructor = type.primaryConstructor
    ?: throw IllegalArgumentException("주 생성자가 없습니다.")

val arguments = constructor.parameters.map { parameter ->
    val dependencyType = parameter.type.classifier as? KClass<*>
        ?: throw IllegalArgumentException("지원하지 않는 파라미터입니다.")

    resolve(dependencyType, scope = scope)
}

return constructor.call(*arguments.toTypedArray()).also { instance ->
    injectFields(instance, scope)
    scope.put(DependencyKey(type), instance)
}
```

처음에는 코드가 복잡해 보였지만, 동작을 나누면 비교적 단순하다.

<b>첫째, 생성자를 찾는다.</b>

```txt
val constructor = type.primaryConstructor
```

생성하려는 타입의 주 생성자를 가져온다.

<b>둘째, 생성자가 요구하는 파라미터를 확인한다.</b>

```txt
constructor.parameters.map { parameter ->
    val dependencyType = parameter.type.classifier as? KClass<*>
        ?: throw IllegalArgumentException()

    resolve(dependencyType, scope = scope)
}
```

<code>constructor.parameters</code>로 생성자의 파라미터 목록을 읽고, 각 파라미터의 타입을 확인한다.

그리고 다시 <code>resolve()</code>를 호출한다.

여기가 핵심이다.

<b>필요한 객체가 또 다른 객체를 필요로 한다면, 그 의존성부터 해결하는 것이다.</b>

이런 방식으로 의존성 관계를 따라 내려가다가 더 이상 추가 의존성이 필요하지 않은 객체를 만나면 그 객체부터 생성할 수 있다.

<b>셋째, 준비한 인자로 인스턴스를 만든다.</b>

```txt
constructor.call(*arguments.toTypedArray())
```

재귀적인 호출을 통해 얻은 의존성들을 <code>arguments</code>에 모아 생성자에 전달한다.

<b>넷째, 생성한 객체를 스코프에 저장한다.</b>

```txt
scope.put(DependencyKey(type), instance)
```

객체를 생성한 뒤 현재 스코프에 저장한다.

이후 같은 스코프에서 같은 의존성을 요청하면 저장된 인스턴스를 재사용할 수 있다.

### 3.2 리플렉션과 스코프는 서로 다른 문제를 해결한다

여기서 한 가지 구분할 필요가 있다.

리플렉션은 어떤 타입의 생성자를 호출할지 결정하는 데 사용된다.

반면 스코프는 생성한 객체를 어디에 보관하고, 어느 범위에서 재사용할지 결정한다.

실제 프로젝트에서도 이 둘을 함께 사용했다.

<code>ProductRepository</code>처럼 별도의 등록 없이 생성할 수 있는 타입은 컨테이너가 생성자를 확인하고 인스턴스를 만든 뒤 ViewModel 스코프에 저장한다.

반면 <code>RoomCartRepository</code>는 앱 시작 시 <code>CartRepository</code> 타입과 <code>RoomCart</code> qualifier에 연결해 앱 스코프에 등록한다.

이후 ViewModel에서 <code>CartRepository</code>를 요청하면, 현재 스코프와 부모 앱 스코프를 확인해 등록된 구현체를 가져온다.

전체 흐름을 정리하면 다음과 같다.

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

이 구조에서 얻은 이점은 분명했다.

ViewModel이나 Repository가 늘어나더라도 컨테이너의 공통 생성 로직을 재사용할 수 있다.

하지만 여기서 중요한 사실을 깨달았다.

<b>객체 생성 코드가 사라진 것이 아니라, 그 책임이 개발자가 작성한 개별 생성 코드에서 컨테이너의 런타임 로직으로 이동한 것이다.</b>

이 차이가 리플렉션의 장점이자 비용으로 이어진다.

---

## 4. 편리해진 만큼 무엇을 포기했을까?

처음 DI 컨테이너를 구현할 때 가장 만족스러웠던 부분은 반복 코드가 줄어든다는 점이었다.

ViewModel마다 의존성을 연결하는 Factory를 따로 작성하지 않아도 됐다.

객체의 생성자 구조가 달라지더라도 컨테이너가 파라미터를 읽고 의존성을 재귀적으로 해결할 수 있었다.

하지만 객체를 만드는 코드를 직접 작성하지 않는다는 것은, 그 과정을 컴파일러가 모두 검증해 줄 수도 없다는 의미다.

### 4.1 컴파일 시점에 발견하던 오류가 런타임으로 이동한다

직접 객체를 생성한다면 다음과 같이 작성할 수 있다.

```txt
val repository = ProductRepository(/* 필요한 의존성 */)
```

생성자의 파라미터가 변경되거나 잘못된 타입의 인자를 전달하면 해당 코드에서 컴파일 오류가 발생할 수 있다.

반면 리플렉션 기반 컨테이너는 런타임에 생성자를 찾는다.

실제 구현에는 다음과 같은 코드가 존재한다.

```txt
val constructor = type.primaryConstructor
    ?: throw IllegalArgumentException("주 생성자가 없습니다.")
```

해당 타입에 주 생성자가 없으면 예외가 발생한다.

또한 생성자 파라미터를 해결하지 못하거나, 필요한 구현체가 등록되지 않았거나, 동적 호출에 전달한 인자가 올바르지 않더라도 실행 과정에서 실패할 수 있다.

즉, 의존성 관계의 일부 오류는 해당 객체의 생성 경로가 실제로 실행돼야 발견된다.

작은 프로젝트에서는 큰 문제가 아닐 수도 있다.

하지만 객체의 수와 의존성 관계가 복잡해질수록, 이런 오류를 언제 발견할 수 있는지는 중요한 차이가 된다.

### 4.2 의존성 관계를 코드만 보고 파악하기 어려워진다

현재 ViewModel은 필드 주입을 사용한다.

이 방식은 생성자 파라미터 없이 ViewModel을 만들 수 있다는 장점이 있다.

하지만 생성자만 보고 어떤 의존성이 필요한지 알기는 어렵다.

개발자는 클래스 내부에서 <code>@Inject</code>가 붙은 필드를 찾아야 한다.

또한 <code>lateinit</code>으로 선언된 필드가 필요한 시점까지 주입되지 않는다면 초기화되지 않은 상태로 접근하는 오류가 발생할 수 있다.

생성자 주입을 사용한다면 필수 의존성을 생성자에 명시할 수 있다.

따라서 새로운 DI 구조를 설계한다면 생성자 주입을 우선 고려할 이유가 있다.

여기서 필드 주입 자체가 항상 잘못됐다는 결론을 내리려는 것은 아니다.

현재 구현에서는 공통 Factory와 필드 주입을 통해 원하는 동작을 구현했다.

다만 <b>반복 코드를 줄이는 편리함과 의존성을 명시적으로 드러내는 장점 사이에 트레이드오프가 존재한다</b>는 것을 확인할 수 있었다.

---

## 5. 리플렉션은 정확히 어디에서 비용이 발생할까?

이제 처음 궁금했던 성능 문제로 돌아가 보자.

리플렉션은 느리다고 알려져 있지만, 단순히 <code>call()</code>을 사용한다는 이유만으로 실제 앱에서 어느 정도의 성능 문제가 생기는지 알 수는 없다.

비용을 이해하려면 먼저 컨테이너에서 어떤 작업이 실행되는지 구분해야 한다.

### 5.1 메타데이터 탐색 비용

첫 번째는 클래스와 생성자 정보를 조회하는 과정이다.

```txt
val constructor = type.primaryConstructor
val parameters = constructor?.parameters
```

직접 생성 코드를 작성하는 대신, 런타임에 클래스의 생성자와 파라미터 정보를 확인한다.

실제 소요 시간은 Reflection 구현, 메타데이터 캐싱 여부, 실행 환경에 따라 달라질 수 있다.

따라서 <code>primaryConstructor</code>를 호출할 때마다 클래스 정보를 처음부터 전부 분석한다고 가정해서는 안 된다.

하지만 적어도 직접 생성자 호출과는 다른 정보 조회 경로를 사용한다는 사실은 분명하다.

### 5.2 동적 호출 비용

두 번째는 런타임에 찾아낸 생성자를 호출하는 과정이다.

```txt
constructor.call(*arguments.toTypedArray())
```

직접 호출은 컴파일 시점에 호출 대상이 정해진다.

반면 이 코드는 <code>KFunction</code>을 통해 대상 생성자를 동적으로 호출한다.

호출 과정에서는 전달된 인자를 처리하고 실제 생성자를 실행해야 한다.

이런 추가 경로가 직접 호출과 비교해 어느 정도 차이를 만드는지는 측정이 필요하다.

### 5.3 필드 탐색과 주입 비용

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

따라서 실제 컨테이너의 작업은 단순한 생성자 호출 하나로 끝나지 않는다.

생성자 탐색, 파라미터 확인, 의존성 해결, 필드 탐색, 애너테이션 검사, 필드 대입까지 여러 단계가 이어진다.

리플렉션 기반 컨테이너의 성능을 분석하려면 이러한 단계들을 구분해서 볼 필요가 있다.

### 5.4 하지만 모든 화면 갱신에서 이 비용을 지불하는 것은 아니다

여기서 가장 중요하게 생각한 부분이 있다.

<b>추가 비용이 존재하는 것과 그 비용이 실제 앱 성능에 큰 영향을 미치는 것은 다른 문제다.</b>

현재 컨테이너는 생성한 객체를 스코프에 저장하고 재사용한다.

이미 등록된 의존성을 요청하면 생성자를 다시 호출하지 않고 기존 인스턴스를 반환할 수 있다.

또한 Compose에서 <code>viewModel()</code>을 사용한다고 해서 매번 새로운 ViewModel을 생성하는 것도 아니다.

기존 ViewModel이 유지되는 동안에는 해당 인스턴스를 재사용하므로, 일반적인 recomposition이 발생할 때마다 ViewModel 생성과 필드 주입을 반복하는 구조는 아니다.

따라서 리플렉션을 사용하는 위치와 실행 빈도를 함께 살펴봐야 한다.

한 가지 더 구분할 점이 있다.

현재 컨테이너가 수행하는 <b>인스턴스 캐싱과 리플렉션 메타데이터 캐싱은 서로 다르다.</b>

인스턴스 캐싱은 이미 생성한 객체를 재사용하는 것이다.

메타데이터 캐싱은 조회한 생성자나 필드 정보를 다시 활용하는 것이다.

현재 구현의 스코프 캐싱만으로 모든 리플렉션 탐색 결과가 별도로 캐싱된다고 말할 수는 없다.

이는 추후 성능 실험에서 구분해 확인해 볼 부분이다.

---

## 6. 그렇다면 성능은 어떻게 비교해야 할까?

현재까지의 분석으로 확인한 것은 리플렉션 기반 컨테이너가 직접 생성보다 많은 런타임 작업을 수행한다는 점이다.

하지만 아직 실행 시간이나 메모리 할당량을 측정하지 않았다.

따라서 현재 단계에서 '몇 배 느리다'거나 '실제 앱에서 체감할 수 있을 정도로 느리다'고 결론 내리는 것은 적절하지 않다.

성능을 확인하려면 비교 대상을 명확하게 분리해야 한다.

### 6.1 첫 번째 실험: 객체 생성 자체를 비교하기

가장 간단한 <code>User</code> 객체부터 비교해 볼 수 있다.

```txt
// A. 직접 생성
val user = User("조디악", 25)
```

```txt
// B. 미리 조회한 생성자 재사용
val constructor = User::class.primaryConstructor!!

val user = constructor.call("조디악", 25)
```

```txt
// C. 생성자 조회부터 실행
val constructor = User::class.primaryConstructor!!

val user = constructor.call("조디악", 25)
```

B와 C는 코드 자체는 같지만 <b>무엇을 측정 구간에 포함하는지가 다르다.</b>

B에서는 생성자 조회를 측정 전에 끝내고 <code>call()</code>만 반복한다.

C에서는 매번 생성자를 조회하는 API 호출부터 <code>call()</code>까지 측정한다.

이렇게 나누면 직접 생성과 동적 호출의 차이뿐 아니라, 생성자 조회를 측정에 포함했을 때의 차이도 살펴볼 수 있다.

다만 C가 항상 새로운 메타데이터 분석을 강제하는 것은 아니다. 내부적으로 캐싱된 정보가 재사용될 가능성도 있다.

### 6.2 두 번째 실험: 실제 DI 컨테이너를 비교하기

단순 객체 생성에서 차이가 나타나더라도 실제 DI 컨테이너에서 같은 결과가 나온다고 단정할 수는 없다.

컨테이너에는 의존성 탐색과 캐싱, 필드 주입 등이 포함되기 때문이다.

따라서 동일한 의존성 그래프를 대상으로 다음을 비교해 보고 싶다.

| 비교 대상 | 측정하려는 작업 |
| --- | --- |
| 직접 생성 Factory | 생성자 호출과 의존성 직접 연결 |
| 리플렉션 DI 컨테이너 | 생성자 탐색, 재귀적 의존성 해결, 필드 주입 |
| 이미 생성된 의존성 조회 | 스코프에서 기존 객체 반환 |

특히 처음 인스턴스를 만드는 상황과 이미 생성한 인스턴스를 재사용하는 상황은 분리해야 한다.

필요하다면 리플렉션 메타데이터를 별도로 캐싱한 구현도 추가해 비교할 수 있을 것이다.

### 6.3 측정 환경도 결과의 일부다

아주 짧은 객체 생성 코드를 단순히 반복 실행해 평균을 계산하는 것만으로는 신뢰할 만한 결과를 얻기 어렵다.

JVM 또는 Android 런타임의 최적화, 워밍업 상태, 가비지 컬렉션, 메모리 할당 등이 측정에 영향을 줄 수 있기 때문이다.

또한 측정하려던 객체 생성이 컴파일러나 런타임에 의해 최적화되어 결과가 왜곡되지 않도록 해야 한다.

Android 환경에서는 작은 코드 구간을 반복 측정할 수 있는 Jetpack Benchmark의 Microbenchmark 도구를 검토할 수 있다.

측정 시에는 다음 조건을 기록할 필요가 있다.

- 동일한 데이터와 의존성 구조
- 동일한 기기 및 실행 환경
- 동일한 빌드 조건
- 워밍업과 실제 측정 구간
- 반복 측정 결과와 분산
- 실행 시간과 필요시 메모리 할당량

그리고 실제 앱에 미치는 영향을 판단하려면 단순 함수 실행 시간뿐 아니라 ViewModel 생성이 사용자 경험에서 차지하는 비중도 확인해야 한다.

수치 차이가 존재하더라도 실행 빈도가 낮고 전체 작업 시간에서 차지하는 비중이 작다면 앱 수준의 영향은 제한적일 수 있다.

반대로 생성이 빈번하거나 초기화 과정에서 많은 객체를 만들어야 한다면 추가 작업이 더 중요해질 수 있다.

현재는 어느 쪽인지 단정할 측정 결과가 없다.

---

## 7. 처음 궁금했던 Serializable과 Parcelable은 어떻게 다를까?

여기까지 살펴보니 처음 접했던 설명을 다시 생각하게 됐다.

> Serializable은 리플렉션을 사용하기 때문에 느리다.

이 문장은 리플렉션이 비용을 발생시킬 수 있다는 점을 알려주지만, 두 직렬화 방식의 성능 차이를 설명하기에는 지나치게 단순하다.

왜냐하면 <code>Serializable</code>과 <code>Parcelable</code>은 리플렉션 사용 여부만 다른 기술이 아니기 때문이다.

### 7.1 Serializable은 객체 그래프를 직렬화한다

Java의 기본 직렬화 메커니즘은 <code>ObjectOutputStream</code>과 <code>ObjectInputStream</code>을 사용한다.

객체를 직렬화할 때 클래스에 대한 설명 정보와 직렬화 대상 필드, 참조된 객체 등을 정해진 스트림 형식으로 기록한다.

역직렬화할 때는 그 정보를 해석해 객체 그래프를 복원한다.

이 과정에서 런타임의 클래스·필드 정보를 다루는 작업이 필요하며, 기본 직렬화 메커니즘의 비용은 이러한 정보 처리뿐 아니라 객체 그래프를 탐색하고 스트림을 읽고 쓰는 과정에서도 발생한다.

따라서 Serializable의 성능을 단순히 리플렉션 호출 비용 하나로 환원하기는 어렵다.

### 7.2 Parcelable은 Android의 Parcel 형식을 사용한다

<code>Parcelable</code>은 Android의 <code>Parcel</code> 형식으로 객체의 데이터를 기록하고 복원하기 위한 인터페이스다.

Activity 간 데이터 전달이나 Binder IPC 등 Android의 데이터 전달 경로에서 사용된다.

개발자가 <code>Parcelable</code> 구현을 직접 작성할 수도 있지만, Kotlin에서는 <code>@Parcelize</code>를 활용할 수 있다.

```txt
@Parcelize
data class User(
    val name: String,
    val age: Int,
) : Parcelable
```

<code>kotlin-parcelize</code> 플러그인은 컴파일 과정에서 필요한 <code>Parcelable</code> 구현을 생성한다.

덕분에 직렬화 로직을 매번 직접 작성하지 않아도 된다.

다만 <code>Parcelable</code>을 사용했다고 해서 모든 상황에서 리플렉션이 완전히 배제된다거나, 어떤 데이터든 항상 더 빠르다고 단정할 수는 없다. 실제로 처리하는 타입과 직렬화 경로에 따라 동작 방식은 달라질 수 있다.

### 7.3 결국 비교하려는 비용이 다르다

이제 두 가지 질문을 구분할 수 있다.

<b>첫 번째 질문: 리플렉션을 통한 객체 생성은 직접 생성보다 얼마나 느릴까?</b>

이것은 앞서 살펴본 생성자 탐색과 <code>call()</code>의 비용을 비교해야 한다.

<b>두 번째 질문: Serializable과 Parcelable 중 실제 Android 데이터 전달에서 어떤 방식이 더 효율적일까?</b>

이것은 같은 데이터를 두 방식으로 직렬화하고 역직렬화하면서 전체 처리 시간, 데이터 크기, 메모리 할당량 등을 비교해야 한다.

첫 번째 실험에서 리플렉션의 추가 비용이 확인되더라도, 그 수치만으로 두 번째 질문에 대한 답을 얻을 수는 없다.

서로 다른 처리 과정이 포함되어 있기 때문이다.

결국 처음 들었던 설명은 더 정확하게 표현할 수 있다.

<b>Serializable과 Parcelable은 서로 다른 직렬화 메커니즘을 사용하며, 리플렉션을 포함한 런타임 정보 처리 비용은 그 차이를 구성하는 여러 요소 중 하나다.</b>

실제 성능을 판단하려면 사용하려는 객체와 데이터 전달 경로를 기준으로 측정해야 한다.

---

## 8. 그렇다면 리플렉션 대신 코드 생성을 사용하면 어떨까?

지금까지 살펴본 리플렉션의 특징은 런타임에 필요한 정보를 읽고 생성 과정을 결정한다는 것이다.

그런데 앱에서 사용할 타입과 의존성 관계가 이미 정해져 있다면, 매번 런타임에 그 관계를 해석할 필요가 있을까?

여기에서 코드 생성(Code Generation)이라는 대안이 등장한다.

대표적으로 Kotlin의 KSP와 Android에서 사용하는 Hilt를 살펴볼 수 있다.

### 8.1 KSP는 무엇을 하는 도구일까?

KSP(Kotlin Symbol Processing)는 Kotlin 소스 코드의 선언 정보를 분석하고, 이를 바탕으로 코드를 생성할 수 있도록 지원하는 프레임워크다.

클래스나 생성자, 프로퍼티, 애너테이션 등의 정적인 선언 정보를 분석할 수 있다.

이를 활용하면 개발자가 반복해서 작성하던 코드를 빌드 과정에서 생성할 수 있다.

예를 들어 어떤 클래스의 의존성 선언을 분석해 해당 객체를 생성하는 코드를 만들어 낼 수 있다.

중요한 것은 KSP 자체가 DI 컨테이너는 아니라는 점이다.

KSP는 코드 생성을 위한 기반을 제공하고, 실제로 어떤 코드를 생성하고 어떤 검증을 수행할지는 이를 이용하는 프로세서가 결정한다.

### 8.2 Hilt는 이 문제를 어떻게 해결할까?

Hilt는 Dagger를 기반으로 Android에서 의존성 주입을 편리하게 사용할 수 있도록 지원한다.

Dagger/Hilt의 핵심적인 특징 중 하나는 빌드 시점에 의존성 관계를 분석하고 필요한 코드를 생성한다는 것이다.

이 과정에서 지원되는 의존성 관계의 오류를 빌드 시점에 발견할 수 있다.

반면 직접 만든 리플렉션 컨테이너는 타입을 요청받은 뒤 런타임에 생성자와 의존성을 탐색한다.

결과적으로 두 방식은 비슷한 목적을 달성하지만 일부 책임을 수행하는 시점이 다르다.

| 방식 | 객체 생성 경로 결정 | 장점 | 고려할 점 |
| --- | --- | --- | --- |
| 직접 생성 | 개발자가 작성한 코드 | 명시적이고 단순하며 컴파일러 검사를 활용할 수 있음 | 생성·연결 코드가 반복될 수 있음 |
| 리플렉션 기반 DI | 런타임의 타입 정보 탐색 | 다양한 타입에 공통 생성 로직을 적용할 수 있음 | 동적 탐색 비용과 런타임 오류 가능성 |
| 코드 생성 기반 DI | 빌드 시 선언을 분석해 생성 코드 준비 | 반복 코드를 줄이고 많은 의존성 오류를 빌드 시점에 검출 | 프로세서 설정, 생성 코드 및 빌드 과정 관리 |

코드 생성이 객체 생성 비용 자체를 없애 주는 것은 아니다.

프로그램이 실행될 때 실제 객체는 여전히 만들어야 한다.

차이는 객체를 <b>어떻게 만들어야 하는지 판단하는 작업의 일부를 빌드 시점으로 옮길 수 있다는 것</b>이다.

그리고 이 비교에서 한 가지 더 알 수 있다.

모든 상황에서 코드 생성이 정답인 것은 아니다.

실행 중에야 결정되는 타입을 처리해야 하거나 외부에서 동적으로 불러온 클래스를 다뤄야 한다면, 런타임에 구조를 확인하는 리플렉션이 여전히 유용할 수 있다.

반대로 의존성 관계가 정적이고 컴파일 시점에 충분히 파악할 수 있다면 코드 생성이 더 적합한 선택지가 될 수 있다.

---

## 9. 실제 프로젝트에 적용할 때 확인해야 할 것들

직접 DI 컨테이너를 구현하면서 리플렉션으로 많은 작업을 일반화할 수 있다는 점을 확인했다.

하지만 일반화된 로직이 모든 타입을 안전하게 처리할 수 있다는 의미는 아니다.

현재 구현의 한계도 함께 살펴볼 필요가 있다.

### 9.1 모든 생성자 구조를 지원하지는 않는다

현재 컨테이너는 다음 코드를 사용한다.

```txt
val constructor = type.primaryConstructor
    ?: throw IllegalArgumentException("주 생성자가 없습니다.")
```

따라서 주 생성자가 없는 타입은 이 경로에서 생성하지 못한다.

또한 생성자 파라미터를 전부 <code>resolve()</code>한 뒤 <code>call()</code>에 전달한다.

Kotlin은 기본 인자가 있는 함수를 호출할 때 일부 인자를 생략할 수 있지만, 현재 구현은 그러한 생략을 별도로 처리하지 않는다.

Kotlin Reflection에는 선택적 파라미터를 다룰 수 있는 <code>callBy()</code> API도 있지만, 현재 컨테이너가 사용하는 방식과는 다르다.

### 9.2 제네릭과 qualifier 처리에도 범위가 있다

현재 구현은 다음과 같이 파라미터 타입을 얻는다.

```txt
val dependencyType =
    parameter.type.classifier as? KClass<*>
        ?: throw IllegalArgumentException()
```

하지만 <code>List<Product></code>처럼 제네릭 타입의 인자까지 구별해야 하는 경우에는 외부 클래스가 <code>List</code>라는 사실만으로 충분하지 않다.

<code>Product</code>라는 타입 인자 정보도 함께 고려해야 한다.

또한 현재 구현에서는 필드 주입 시 qualifier를 확인하지만, 생성자 파라미터의 qualifier까지 해석하는 구조는 아니다.

따라서 생성자 파라미터에 동일한 인터페이스의 여러 구현체가 필요한 경우를 처리하려면 추가적인 설계가 필요하다.

재귀적 의존성 해결 과정에서 순환 참조를 어떻게 감지하고 실패를 처리할지도 컨테이너를 확장할 때 확인해야 할 문제다.

### 9.3 Android의 R8 최적화도 고려해야 한다

Android 앱을 배포할 때는 R8에 의한 코드 최적화가 적용될 수 있다.

R8은 사용되지 않는 코드를 제거하거나 클래스와 멤버의 이름을 변경하는 등의 최적화를 수행한다.

문제는 리플렉션을 통해 간접적으로 접근하는 코드다.

일반적인 직접 호출과 달리, 동적으로 탐색하는 클래스나 필드, 애너테이션의 사용 관계를 R8이 항상 충분히 파악할 수 있는 것은 아니다.

그 결과 최적화된 빌드에서 필요한 정보가 변경되거나 제거되면 런타임 문제가 발생할 수 있다.

이것이 현재 프로젝트에서 실제로 발생했다는 의미는 아니다.

다만 리플렉션을 사용하는 DI 컨테이너라면 디버그 빌드에서 정상 동작한다는 사실만으로 충분하지 않다.

최적화된 배포 빌드에서도 생성자 탐색과 필드 주입이 정상적으로 동작하는지 확인해야 한다.

필요하다면 사용하는 클래스와 멤버, 애너테이션에 대해 적절한 R8 keep 규칙을 작성해야 한다. 이때 불필요하게 많은 코드를 최적화 대상에서 제외하지 않도록 규칙의 범위를 좁히는 것도 중요하다.

추가로 Kotlin의 전체 Reflection 기능은 <code>kotlin-reflect</code> 런타임 라이브러리를 통해 제공된다.

따라서 리플렉션을 선택할 때는 실행 비용뿐 아니라 앱에 포함되는 라이브러리가 최종 바이너리 크기에 미치는 영향도 살펴볼 수 있다.

현재 프로젝트에서는 이 부분을 별도로 측정하지 않았다.

---

## 10. 그래서 언제 리플렉션을 선택해야 할까?

이번 분석을 통해 느낀 것은 리플렉션을 사용해야 하는지 판단할 때 단순히 빠르거나 느리다는 기준만으로는 부족하다는 점이다.

상황에 따라 적합한 선택은 달라질 수 있다.

<b>호출할 타입이 이미 정해져 있고 생성 로직이 단순하다면 직접 생성이 명확하다.</b>

컴파일러가 타입과 인자를 검사할 수 있고, 코드를 읽는 개발자도 어떤 객체가 생성되는지 쉽게 파악할 수 있다.

<b>의존성 관계가 정적이면서 반복적인 생성 코드를 줄이고 싶다면 코드 생성 기반 DI를 검토할 수 있다.</b>

Hilt처럼 빌드 과정에서 의존성 그래프를 분석하고 생성 코드를 준비하는 방식은 런타임에 해결해야 할 문제의 일부를 빌드 시점으로 옮길 수 있다.

<b>반면 타입이 실제로 런타임에 결정된다면 리플렉션이 적합할 수 있다.</b>

동적으로 선택되는 구현체나 다양한 클래스의 구조를 읽어야 하는 범용 프레임워크가 그 예다.

이런 경우에도 탐색한 메타데이터를 재사용할 수 있는지, 실패 상황을 명확히 처리할 수 있는지, 성능에 민감한 경로에서 실행되는지 등을 함께 검토할 필요가 있다.

직접 만든 DI 컨테이너에서도 리플렉션을 사용한 목적은 모든 객체를 더 빠르게 생성하기 위해서가 아니었다.

ViewModel마다 반복되는 Factory 코드를 줄이고, 타입 정보를 기반으로 객체 생성과 의존성 연결 과정을 일반화하기 위해서였다.

그 목적에는 부합하는 구현이었다.

하지만 그 편리함과 함께 런타임 타입 검사, 생성자 탐색, 필드 주입, 예외 처리 등의 책임을 컨테이너가 맡게 됐다.

이것이 이번 구현을 통해 가장 분명하게 확인한 트레이드오프다.

---

## 마무리: 코드를 줄인 만큼, 누가 그 일을 대신하고 있을까?

처음에는 리플렉션이 느리다는 설명을 이해하고 싶었다.

그런데 실제 코드를 따라가다 보니 성능뿐 아니라 객체 생성 방식 자체에 대한 고민으로 이어졌다.

직접 생성자를 호출할 때는 어떤 객체를 어떻게 만들어야 하는지가 코드에 명시되어 있었다.

리플렉션을 사용하자 그 정보의 일부를 런타임에 읽고 판단할 수 있게 됐다.

덕분에 서로 다른 타입의 객체 생성 과정을 공통화할 수 있었고, DI 컨테이너를 통해 반복적인 연결 코드를 줄일 수 있었다.

하지만 편리해졌다고 해서 해야 할 일이 사라진 것은 아니었다.

<b>개별 생성 코드에서 하던 일을 컨테이너가 대신 수행하게 됐고, 그 과정에서 런타임 작업과 오류 처리 책임이 추가됐다.</b>

코드 생성 방식은 또 다른 선택을 보여 줬다.

객체를 조립하기 위한 정보를 컴파일 시점에 충분히 분석할 수 있다면, 일부 작업을 빌드 과정에서 처리하는 방법도 있었다.

결국 세 방식의 차이는 단순히 객체를 생성할 수 있느냐가 아니다.

<b>객체 생성에 필요한 정보를 언제 결정하고, 그 과정의 책임과 비용을 어디에 둘 것인가의 차이였다.</b>

여기까지 분석했다고 해서 리플렉션의 성능 문제에 대한 답을 모두 얻은 것은 아니다.

직접 호출보다 추가적인 런타임 경로가 존재한다는 것은 확인했지만, 현재 DI 컨테이너에서 그 비용이 얼마나 발생하는지는 아직 측정하지 않았다.

다음에는 직접 생성, 캐시된 생성자의 <code>call()</code> 호출, 생성자 조회를 포함한 호출을 동일한 조건에서 비교해 보고 싶다.

이후 실제 DI 컨테이너의 초기 객체 생성과 캐시된 의존성 조회까지 측정하면, 리플렉션의 추가 비용이 앱에서 어느 정도 의미를 갖는지 더 구체적으로 판단할 수 있을 것이다.

이번에 얻은 결론은 리플렉션을 사용하지 말아야 한다는 것이 아니다.

<b>리플렉션이 제공하는 유연함이 어디에서 오는지 이해하고, 그 유연함을 위해 감수할 비용이 무엇인지 확인한 뒤 선택해야 한다는 것이다.</b>

---

## 참고 자료

- [Kotlin 공식 문서 - Reflection](https://kotlinlang.org/docs/reflection.html)
- [Kotlin API - KClass.primaryConstructor](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.reflect.full/primary-constructor.html)
- [Kotlin API - KCallable.call / callBy](https://kotlinlang.org/api/core/kotlin-stdlib/kotlin.reflect/-k-callable/)
- [Kotlin 공식 문서 - Kotlin Symbol Processing](https://kotlinlang.org/docs/ksp-overview.html)
- [Dagger 공식 문서 - Hilt Quick Start](https://dagger.dev/hilt/quick-start)
- [Android Developers - Parcelable Implementation Generator](https://developer.android.com/kotlin/parcelize)
- [Oracle - Java Object Serialization Specification](https://docs.oracle.com/en/java/javase/17/docs/specs/serialization/serial-arch.html)
- [Android Developers - Benchmark](https://developer.android.com/jetpack/androidx/releases/benchmark)
- [Android Developers - R8 Keep Rules](https://developer.android.com/topic/performance/app-optimization/keep-rules-overview)

<p />
