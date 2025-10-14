# 의존성 주입(DI), 아직도 헷갈리시나요? (부제: 모찌는 어떻게 밤을 샐 수 있을까?)

안녕하세요! 우아한테크코스 안드로이드 7기 모찌입니다.
DI(Dependency Injection)라는 개념을 처음 배웠을 때, "이걸 왜 써야 하지?" 라는 의문이 먼저 들었습니다. '테스트가 쉬워진다', '확장성이 좋아진다'는 장점들이 머리로는 이해되지만, 가슴으로는 와닿지 않았죠.

이 글을 통해 의존성 주입(DI), 제어의 역전(IoC), 그리고 의존관계 역전 원칙(DIP)이 무엇인지, 그리고 왜 필요한지를 저의 경험에 빗대어 이야기해보려 합니다.

## 의존성이란 무엇일까요?

"의존성"이라는 단어를 들으면 무엇이 떠오르시나요?

저는 프로젝트 발표 전날처럼 밤을 꼬박 새워야 할 때, 꼭 **핫식스**가 있어야만 버틸 수 있습니다. 즉, 밤샘하는 저는 **핫식스**에 의존하고 있는 셈이죠.

이 관계를 코드로 표현해 볼까요.

```kotlin
// Mochi가 HotSix를 직접 생성하여 의존하는 코드
class Mochi {
    private val hotSix = HotSix() // HotSix를 직접 생성!

    fun stayUpAllNight() {
        hotSix.boostEnergy()
        println("모찌는 밤을 새웠다! 🔥")
    }
}

class HotSix {
    fun boostEnergy() {
        println("힘을 내요 슈퍼 파워~!!")
    }
}
```

이 코드는 한 문장으로 "모찌는 밤을 새우기 위해 핫식스를 **직접 만들어서** 마신다"라고 요약할 수 있습니다. `Mochi` 클래스는 `HotSix`라는 **구체적인 클래스**에 직접 의존하고 있습니다.

즉, `Mochi`는 `HotSix`라는 구체적인 클래스에 **직접 의존**하고 있어요. 핫식스가 없으면, 모찌는 더 이상 힘을 내서 밤을 셀 수 없습니다. 

<img width="520" height="283" alt="image" src="https://github.com/user-attachments/assets/7f468904-6030-48e4-9943-fa91d5537dbe" />

**그런데 말입니다.**

이 구조에는 큰 문제점이 있어요.

- 저희 집 앞 편의점에 더 이상 핫식스를 팔지 않는다면?
- 새로 나온 다른 에너지 드링크인 얼박사가 마시고 싶다면?

위에 2가지 상황이 생기면, 모찌는 밤을 세우기 어려워요. 왜냐구요?

코드를 봤을 때, 모찌는 핫식스를 마셔야지만, 힘을 내서 밤을 세울 수 있기 때문이죠. 위와 같은 상황이 닥치면 `Mochi`는 밤을 새우기 곤란해집니다. 코드를 직접 수정해서 `private val drink = Eolbaksa()` 와 같이 바꾸기 전까지는요. 이처럼 `Mochi`와 `HotSix`는 너무 단단하게 묶여있어(강한 결합, Tight Coupling), 변화에 유연하게 대처하기 어렵습니다.

그렇다면, 모찌가 특정 에너지 드링크가 아닌 **어떤 종류의 에너지 드링크라도** 유연하게 마시며 밤을 샐 수 있게 만들려면 어떻게 해야 할까요?

이 문제를 해결하는 과정에서, 우리는 **의존성 주입(DI)**, **제어의 역전(IoC)**, 그리고 **의존관계 역전 원칙(DIP)**이라는 중요한 개념들을 자연스럽게 만나게 될 것입니다. 함께 그 여정을 떠나볼까요?

# 제어의 역전(IoC: Inversion of Control)  (부제: "내가 만들게"에서 "만들어 줘"로)

앞서 우리는 `Mochi`가 `HotSix`를 직접 만들어 사용하는, 아주 단단하게 묶인 관계를 보았습니다.

<img width="540" height="535" alt="image 1" src="https://github.com/user-attachments/assets/e1eb02fe-bb24-4c7a-8bd8-b7093d67b0ec" />


```kotlin
// 기존 코드: Mochi가 직접 HotSix를 생성하고 제어한다.
class Mochi {
    private val drink = HotSix() // Mochi가 직접 HotSix 객체 생성의 '제어권'을 가짐
    
    fun stayUpAllNight() {
        drink.boostEnergy()
        println("모찌는 밤을 새웠다! 🔥")
    }
}

class HotSix {
    fun boostEnergy() {
        println("힘을 내요 슈퍼 파워~!!")
    }
}
```

그런데, 곰곰히 생각해보면, 모찌가 밤을 새우기 위해 반드시 직접 핫식스를 만들 필요는 없습니다. 우리가 편의점에서 음료를 사는 것처럼 핫식스를 구해도 되지 않을까요? 예로, 집앞 편의점에서 핫식스를 사는 거죠. 즉, 객체가 사용할 의존 객체를 스스로 만드는 것이 아니라, **외부로부터 만들어진 것을 전달받는 것**은 어떨까요? 이것이 바로 **제어의 역전(IoC: Inversion of Control)**이라는 개념의 핵심입니다.

객체 생성의 ‘제어권’이 객체 자신(`Mochi`)에서 외부(`main` 함수, 혹은 DI 컨테이너)로 역전되었다는 의미죠!

### **💡 코드로 보는 제어의 역전**

IoC 원칙을 적용하면 코드가 어떻게 바뀔까요? `Mochi`가 **생성자(constructor)**를 통해 외부에서 에너지 드링크를 받도록 수정해 보겠습니다.

```kotlin
// IoC 적용: Mochi는 더 이상 HotSix를 직접 만들지 않는다.
class Mochi(
private val drink: HotSix // 외부에서 '만들어진' drink 객체를 받음
) { 
    fun stayUpAllNight() {
        drink.boostEnergy()
        println("모찌는 밤을 새웠다! 🔥")
    }
}

fun main() {
    // '외부'에 해당하는 main 함수가 객체를 생성하고 관계를 맺어준다.
    val hotSix = HotSix()         // 1. 의존 객체를 만든다.
    val mochi = Mochi(hotSix)     // 2. Mochi에게 만들어진 객체를 '넣어준다'.

    mochi.stayUpAllNight()
}
```

이제 `Mochi`는 `HotSix`를 직접 만들지 않습니다. 그저 누군가 **"자, 여기 핫식스!"** 하고 건네주는 것을 받아서 마시기만 하면 됩니다. 객체 생성과 의존 관계 설정의 책임이 `Mochi`에서 `main` 함수로 완전히 넘어갔죠. 이것이 바로 **제어의 역전**입니다. 즉, 제어의 역전은 **"객체는 자신이 사용할 의존성을 직접 만들지 않고, 외부에서 관리해야 해!"** 라는 **원칙/개념 (Principle)**입니다.

### 의존성 주입(DI): 그래서, 어떻게 넣어줄 건데?

"외부에서 의존 객체를 전달한다"는 제어의 역전(IoC) 개념, 이제 조금 감이 오시나요?

**의존성 주입(DI: Dependency Injection)** 은 IoC라는 원칙을 구현하는 **구체적인 방법론(패턴)** 입니다. 즉, 의존성을 외부에서 '주입(Injection)'해서 제어의 역전을 달성하는 기술이죠.

그렇다면 이런 의문이 생길 수 있습니다.
"외부에서 객체를 전달하는 방법은 방금 본 생성자 방식만 있는 걸까?"

의존성을 주입하는 대표적인 방법은 3가지가 있습니다. 생성자 주입, 세터 주입, 필드 주입이 있습니다.

### 1. 생성자 주입 (Constructor Injection)

객체를 생성하는 시점에 생성자를 통해 의존성을 주입합니다. **가장 권장되는 방식**입니다.

- **특징**
    - **불변성**: 의존성을 `val`로 선언하여 변경 불가능하게 만들 수 있습니다.
    - **신뢰성**: 객체가 생성되는 시점에 필요한 모든 의존성이 주입됨을 보장합니다.

```kotlin
// Mochi 객체는 생성될 때 반드시 HotSix 객체가 필요하다.
class Mochi(private val drink: HotSix) {
    fun stayUpAllNight() {
        drink.boostEnergy()
        println("모찌는 밤을 새웠다! 🔥")
    }
}

fun main() {
    val hotSix = HotSix()
    val mochi = Mochi(hotSix) // 생성자를 통해 주입
    mochi.stayUpAllNight()
}
```

### 2. 세터 주입 (Setter Injection)

이 경우에는 프레임워크가 개입하지 않고, **외부 코드(main 함수 등)** 이 명시적으로 “넣어주는”  것

객체가 생성된 이후, 세터(setter) 메서드를 통해 의존성을 주입합니다.

- **특징**
    - **선택적 의존성**: 반드시 필요하지는 않은, 선택적인 의존성을 주입할 때 유용합니다.
    - **불완전한 상태**: 의존성이 주입되기 전까지 객체가 불완전한 상태일 수 있습니다.
    - **가변성**: 외부에서 언제든지 의존성을 변경할 수 있습니다.

```kotlin
class Mochi {
    private var drink: HotSix? = null // 일단 자리를 비워두고, 나중에 주입받겠다!

    fun setDrink(drink: HotSix) {
        this.drink = drink
    }

    fun stayUpAllNight() {
         drink?.boostEnergy() ?: println("음료가 없어요... 모찌는 잠들었다 😴")
    }
}

fun main() {
    val hotSix = HotSix()
    val mochi = Mochi()
    mochi.setDrink(hotSix) // 세터 메서드를 통해 주입
    mochi.stayUpAllNight()
}
```

### 3. 필드 주입 (Field Injection)

클래스의 멤버 변수(필드)에 어노테이션을 붙여 직접 의존성을 주입합니다. **DI 프레임워크(Hilt, Dagger, Spring 등)의 도움이 반드시 필요합니다.**

- **특징**
    - **간결한 코드**: 코드가 매우 간결해집니다.
    - **프레임워크 의존**: DI 프레임워크 없이는 동작하지 않습니다.
    - **숨은 의존성**: 주입 시점이 코드에 명시적으로 드러나지 않아 흐름 파악이 어려울 수 있습니다.

```kotlin
// 예시: Hilt나 Dagger 같은 프레임워크 사용 시
// 이 코드는 DI 프레임워크가 있어야만 동작합니다.
class Mochi {
    @Inject // "프레임워크야, 여기에 HotSix 타입의 객체를 주입해 줘!"
    lateinit var drink: HotSix

	  // 개발자는 Mochi()로 객체를 생성하지만,
    // 프레임워크가 내부적으로 drink 필드에 HotSix 객체를 넣어줍니다.
    fun stayUpAllNight() {
        drink.boostEnergy()
        println("모찌는 밤을 새웠다! 🔥")
    }
}

```

- **IoC (제어의 역전)**: "객체는 자신이 사용할 의존성을 직접 만들지 않고, 외부에서 관리해야 해!" 라는 **원칙/개념 (Principle)**
- **DI (의존성 주입)**: 그 원칙을 실현하기 위해 "생성자나 세터 등을 통해 외부에서 의존성을 넣어주자!" 라는 **구현 패턴 (Pattern)**

이제 우리는 제어의 역전을 통해 `Mochi`와 `HotSix`의 단단한 연결을 조금 느슨하게 만들었습니다. 하지만 아직 문제가 완전히 해결되지는 않았습니다. `Mochi`는 여전히 `HotSix`라는 **구체적인 클래스**에 의존하고 있으니까요. 만약 **얼박사** `Eolbaksa`를 마시려면 코드를 또 바꿔야 합니다.

이 문제를 해결하기 위해, 우리는 마지막 열쇠인 **의존관계 역전 원칙(DIP)** 을 살펴봐야 합니다.

<img width="640" height="436" alt="image 2" src="https://github.com/user-attachments/assets/83a082b7-5eb4-48b7-b59c-836ba05cab6b" />


(모찌가 좋아하는 얼박사 자랑 타임)

## 마지막 열쇠, DIP: "구현 말고 약속에 의존하기”

### **DIP(Dependency Inversion Principle): “구체가 아니라 ‘역할(추상화)’에 기대라”**

지금까지 우리는 IoC/DI로 **“모찌가 음료를 직접 만들지 않게”** 했습니다. 하지만 Mochi는 여전히 **구체 클래스 HotSix** 에 묶여 있습니다.

> "핫식스 말고 얼박사를 마시고 싶으면 `Mochi(private val drink: Eolbaksa)`처럼 Mochi 클래스 자체를 수정해야 하잖아?"
> 

이것이 바로 **구체 클래스에 의존**하기 때문에 발생하는 문제입니다. 이 문제를 해결하는 열쇠가 바로 **의존관계 역전 원칙(DIP, Dependency Inversion Principle)** 입니다.

**DIP의 핵심 원칙**

1. 상위 모듈은 하위 모듈에 의존해서는 안 된다. 둘 모두 **추상화**에 의존해야 한다.
2. 추상화는 세부 사항에 의존해서는 안 된다. 세부 사항이 추상화에 의존해야 한다.

너무 어렵게 들리나요? 쉽게 말해 **"특정 부품(구현)에 의존하지 말고, 부품을 꽂을 수 있는 규격(인터페이스)에 의존하라"** 는 뜻입니다.

### 1. '규격(인터페이스)' 정의하기

먼저 '에너지 드링크'라는 공통의 '규격'을 정의합시다. 이 규격의 이름은 `EnergyDrink`이고, `boostEnergy()`라는 기능을 반드시 제공해야 한다는 '약속'입니다.

```kotlin
// '에너지 드링크'라면 반드시 이 기능을 제공해야 한다는 약속(추상화)
interface EnergyDrink {
    fun boostEnergy()
}
```

이제 모든 에너지 드링크 '부품'들은 이 규격에 맞춰 만들어집니다.

```kotlin
// HotSix는 EnergyDrink 규격을 따르는 구체적인 구현체
class HotSix : EnergyDrink {
    override fun boostEnergy() = println("🔥 핫식스 파워업!")
}

// Eolbaksa도 EnergyDrink 규격을 따르는 구체적인 구현체
class Eolbaksa : EnergyDrink {
    override fun boostEnergy() = println("🧊 얼박사 각성 완료!")
}
```

### 2. '규격'에 의존하도록 코드 수정하기

이제 `Mochi`는 `HotSix`나 `Eolbaksa` 같은 특정 '부품'이 아닌, `EnergyDrink`라는 '규격'에만 의존하게 됩니다.

```kotlin
// ⛳️ 포인트: 특정 클래스(HotSix)가 아닌 약속(EnergyDrink)에 의존한다.
class Mochi(private val drink: EnergyDrink) {
    fun stayUpAllNight() {
        drink.boostEnergy()
        println("모찌는 밤을 새웠다! 🔥")
    }
}
```

이제 외부에서 어떤 부품을 갈아 끼우든 `Mochi` 코드는 전혀 영향을 받지 않습니다.

```kotlin
fun main() {
    // Mochi 코드는 그대로인데, 외부에서 주입하는 부품만 교체!
    val mochiWithHotSix = Mochi(HotSix())
    val mochiWithEolbaksa = Mochi(Eolbaksa())

    mochiWithHotSix.stayUpAllNight()
    // 출력: 🔥 핫식스 파워업!
    //      모찌는 밤을 새웠다! 🔥

    println("----- 음료 교체! -----")

    mochiWithEolbaksa.stayUpAllNight()
    // 출력: 🧊 얼박사 각성 완료!
    //      모찌는 밤을 새웠다! 🔥
}
```

이것이 바로 **DI**와 **DIP**가 함께 만들어내는 강력한 시너지, **"기존 코드 수정 없이 기능을 확장하고 교체하는 힘"** 입니다.

### 3. 왜 이렇게 하는 게 중요한가요? (DIP가 가져다주는 강력한 효과)

- **유연한 코드**: 새로운 에너지 드링크(`RedBull`)가 출시되어도 `Mochi` 코드를 건드릴 필요 없이 새로운 클래스를 추가하기만 하면 됩니다. **(OCP: 개방-폐쇄 원칙)**
- **느슨한 결합**: `Mochi`는 `HotSix`가 어떻게 에너지를 내는지 전혀 알 필요가 없습니다. 그저 `EnergyDrink` 규격에 맞는 기능(`boostEnergy`)을 호출할 뿐입니다.
- **쉬운 테스트**: 실제 `HotSix` 객체 없이도, 테스트용 가짜 객체(`FakeDrink`)를 만들어 `Mochi`의 행동을 완벽하게 검증할 수 있습니다.

```kotlin
// 테스트가 정말 쉬워집니다!
class FakeDrink : EnergyDrink {
    var isBoosted = false
        private set
    override fun boostEnergy() {
        isBoosted = true
    }
}

// @Test // JUnit 테스트 코드 예시
fun `모찌는_음료가_에너지를_주입하면_밤샘에_성공해야_한다`() {
    // given
    val fakeDrink = FakeDrink()
    val mochi = Mochi(fakeDrink)

    // when
    mochi.stayUpAllNight()

    // then
    assert(fakeDrink.isBoosted) // boostEnergy()가 호출되었는지 검증
}
```

### 4. DI 프레임워크와 함께 사용하기 (Hilt 예시)

**DIP**를 사용하면 **DI** 프레임워크에게 **"이 규격(`EnergyDrink`)을 요청하면, 저 부품(`HotSix`)을 주세요"** 라고 알려줘야 합니다. 이 역할을 하는 것이 바로 **모듈(Module)** 입니다.

```kotlin
// Hilt에게 의존성 주입 방법을 알려주는 설명서(Module)
@Module
@InstallIn(SingletonComponent::class)
abstract class DrinkModule {
    // "누군가 EnergyDrink(규격)을 달라고 하면, HotSix(구현체)를 줘"
    @Binds
    abstract fun bindEnergyDrink(impl: HotSix): EnergyDrink
}
```

이제 Hilt를 사용하는 곳에서는 `EnergyDrink`를 요청하기만 하면 됩니다.

```kotlin
// Hilt가 이 코드를 보고 알아서 new Mochi(new HotSix())를 해준다.
@AndroidEntryPoint
class MyActivity : AppCompatActivity() {
    // Mochi를 주입해줘! (Mochi는 EnergyDrink가 필요하네?
    // DrinkModule을 보니 HotSix를 주면 되는구나!)
    @Inject lateinit var mochi: Mochi
}

// Mochi도 Hilt에게 생성 방법을 알려준다.
class Mochi @Inject constructor(
    private val drink: EnergyDrink // 구체 클래스가 아닌 '규격'에 의존!
) {
    // ...
}
```

나중에 기본 음료를 `Eolbaksa`로 바꾸고 싶다면, `DrinkModule`의 `impl: HotSix` 부분만 `impl: Eolbaksa`로 바꾸면 앱 전체의 동작이 변경됩니다. `Mochi`는 단 한 줄도 수정할 필요가 없죠.

### 5. DIP 적용 시 흔히 하는 실수 (안티패턴)

1. **인터페이스를 사용했지만, 의존은 구체 클래스에 할 때**
    
    ```
    // ❌ DIP 위반: EnergyDrink 인터페이스가 있음에도 불구하고,
    // Mochi는 여전히 HotSix라는 특정 구현에 묶여 있다.
    class Mochi(private val drink: HotSix) { /* ... */ }
    
    // ✅ 올바른 사용법: 추상화(인터페이스)에 의존해야 한다.
    class Mochi(private val drink: EnergyDrink) { /* ... */ }
    
    ```
    
2. **너무 많은 책임을 가진 거대한 인터페이스**`EnergyDrink` 인터페이스에 `boostEnergy()`, `getFlavor()`, `calculateCalories()` 등 관련 없는 기능까지 모두 넣는 것은 좋지 않습니다. 역할에 맞게 인터페이스를 작게 분리하는 것이 좋습니다. **(ISP: 인터페이스 분리 원칙)**

### 6. 내 코드를 DIP로 리팩토링하기: 체크리스트

1. **역할(추상화) 정의**: 내 코드에서 변하기 쉬운 부분의 '역할'이 무엇인지 생각하고 인터페이스로 정의합니다. (예: `EnergyDrink`)
2. **구현체 분리**: 기존 코드를 이 인터페이스의 '구현체'로 만듭니다. (예: `class HotSix : EnergyDrink`)
3. **소비자 수정**: 구현체를 사용하던 코드가 인터페이스에 의존하도록 수정합니다.
4. **의존성 연결**: 외부(main 함수, Hilt 모듈 등)에서 어떤 구현체를 주입할지 결정하여 연결해줍니다.

### 최종 요약

- **의존성**: 어떤 객체가 다른 객체를 필요로 하는 관계.
- **IoC (제어의 역전)**: 필요한 객체를 내가 만들지 않고, 외부에서 만들어서 넣어주는 **개념**.
- **DI (의존성 주입)**: IoC를 실제로 구현하는 **방법** (생성자, 세터, 필드 주입).
- **DIP (의존관계 역전 원칙)**: DI를 할 때, 구체적인 구현이 아닌 **추상화(인터페이스)**에 의존하게 만들어, 코드를 유연하고 테스트하기 쉽게 만드는 **설계 원칙**.

> "모찌는 이제 어떤 에너지 드링크인지 전혀 신경 쓰지 않고도, 밤을 샐 수 있게 되었습니다."
>
