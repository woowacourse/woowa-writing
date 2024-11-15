# ObservableField와 LiveData의 이해(+ Data Binding와 결합하기)

> **글 소개**  
> `ObservableField`와 `LiveData`의 개념과 차이점, `DataBinding`과 함께 활용하는 방법을 알아보는 글입니다.

## 목차
### 0️⃣ 도입부
### 1️⃣ 데이터 바인딩의 기본 개념
- 데이터 바인딩이란?
- 데이터 바인딩의 중요성

### 2️⃣ ObservableField와 LiveData 개념
- **ObservableField 개념**
  - ObservableField란 무엇인가?
  - ObservableField의 장점과 단점
  - ObservableField의 사용 예제 

- **LiveData란?**
  - LiveData란 무엇인가?
  - LiveData의 장점과 단점
  - LiveData의 사용 예제

### 3️⃣ ObservableField와 LiveData 비교
- 두 데이터 홀더의 개념적 차이점
- ObservableField vs LiveData 선택 기준
  - 성능 및 메모리 관리 측면에서의 비교

### 4️⃣ ObservableField와 LiveData를 활용한 데이터 바인딩
- 단방향 데이터 바인딩에서의 ObservableField와 LiveData
- 양방향 데이터 바인딩에서의 ObservableField와 LiveData

<br>

___

<br>

## 0️⃣ 도입부

안드로이드에서 데이터를 관찰하고 UI에 업데이트할 때 흔히 `ObservableField`와 `LiveData`를 사용합니다. 두 객체의 각 특성과 용도에 따라 어느 것을 사용할 지 선택할 수 있습니다.  
`ObservableField`나 `LiveData`를 데이터 바인딩과 함께 사용하면 XML 레이아웃과 데이터를 연결해 UI와 데이터 동기화를 자동화하고, 코드의 간결성과 유지보수성을 높일 수 있습니다.

이 글에서는 `Data Binding`과 함께 자주 사용되는 데이터 홀더인 `ObservableField`와 `LiveData`의 특징 및 사용 예제를 살펴보고, 두 기술의 선택 기준에 대해 알아보려고 합니다.

## 1️⃣ 데이터 바인딩의 기본 개념
### 데이터 바인딩이란?

**데이터 바인딩**이란 레이아웃의 UI 구성 요소를 프로그래밍 방식이 아닌 **선언적 형식**을 사용하여 앱의 데이터 소스에 바인딩할 수 있는 라이브러리입니다.
데이터 바인딩은 코드의 복잡성을 줄이고, 유지보수성을 높이며, **자동으로 뷰에 데이터를 넘겨주어 UI와 데이터 간의 연결을 쉽게 해주는 기술**입니다.

> **데이터 바인딩의 주요 기능**

- **레이아웃 변수 및 표현식(expression)**  
레이아웃 파일에서 데이터 객체를 직접 참조할 수 있습니다. 
이를 통해 레이아웃에서 데이터의 속성을 직접 사용하거나, 데이터를 사용하는 표현식을 작성할 수 있습니다.


- **레이아웃 관찰**  
LiveData나 Observable과 같은 관찰 가능한 데이터 패턴을 지원합니다.


- **이벤트 처리**  
레이아웃에서 이벤트 핸들러를 직접 참조하거나 호출할 수 있습니다. 이를 통해 레이아웃에서 직접 클릭 이벤트를 처리할 수 있습니다.

따라서, 데이터 바인딩은 UI와 데이터를 간단하고 효율적으로 연결할 수 있는 도구입니다.

### 데이터 바인딩의 중요성
데이터 바인딩은 레이아웃을 사용할 때 몇가지 장점을 제공합니다. 

<br>

> **데이터 바인딩의 장점**
- **데이터 바인딩 클래스 자동 생성 및 타입 안정성**
    ```xml
    <layout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto">
        <data>
            <variable
            name="viewmodel"
            type="com.myapp.data.ViewModel" />
        </data>
        
        <ConstraintLayout... />
    </layout>
    ```
  
    위 코드와 같이 `<layout>` 태그로 레이아웃 파일 내의 xml 코드 전체를 둘러싸면 데이터 바인딩을 사용할 수 있습니다.
    `<layout>` 태그는 이 레이아웃에 데이터 바인딩을 한다는 것을 나타냅니다. xml 파일에 `<layout>` 태그가 있다면 데이터 바인딩 라이브러리가 바인딩 클래스를 자동으로 생성해줍니다.  
    <br>
    이러한 자동 생성 기능 덕분에 개발자는 코드에서 잘못된 ID를 참조하거나 잘못된 타입으로 캐스팅할 위험을 줄일 수 있습니다. 자동 생성된 데이터 바인딩 클래스는 레이아웃에서 뷰의 ID와 타입을 기반으로 만들어져, 컴파일 시점에 오류를 발견할 수 있습니다.
    이로 인해 타입 검사가 컴파일 시 이루어지며, 런타임 오류가 감소하고 코드의 안정성이 향상됩니다.

    <br>

- **UI 업데이트와 관련된 중복 코드 감소**  
  <br>
  데이터 바인딩을 활용하면 `findViewById()`를 이용해 뷰를 찾고 데이터를 설정하는 등의 반복적인 UI 업데이트 작업을 줄일 수 있습니다. 
  데이터 바인딩 클래스는 자동으로 생성되므로, 각 뷰를 `findViewById()`로 찾을 필요가 없으며, XML에서 정의한 데이터를 코드에 수동으로 연결하지 않아도 됩니다.
  이로써 코드가 간결해지고 유지 보수가 용이해집니다.
    ```kotlin
        findViewById<TextView>(R.id.name).apply {
            text = viewModel.userName
        }
    ```

   <br>

-  **UI와 데이터 간의 양방향 바인딩 (Two-Way Binding)**  
   <br>
   데이터 변경을 자동으로 UI에 반영하고, 사용자가 UI를 통해 입력한 데이터의 변경사항도 자동으로 업데이트하고 싶을 때도 데이터 바인딩을 활용할 수 있습니다. 
   데이터 바인딩을 이용하면 데이터와 UI 요소 간의 양방향 데이터 바인딩을 손쉽게 구현할 수 있습니다.

   양방향 데이터 바인딩을 EditText에서 사용하는 예를 통해 살펴보겠습니다.
   ```xml
    <EditText
        android:id="@+id/editText"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="@={viewModel.userName}" />
   ```
    `@={viewModel.userName}`와 같이 `@={}` 표기법을 사용하면, ViewModel의 데이터를 EditText로 전달하고 사용자의 입력도 동시에 ViewModel에서 받을 수 있습니다. 
    
<br>

## 2️⃣ ObservableField와 LiveData 개념
### ObservableField 개념

`ObservableField`를 살펴보기 전 먼저 `Observable`에 대해 알아보겠습니다.
> **Observable이란?**

`Observable`은 UI에 연결된 데이터의 변경 사항을 알릴 수 있는 방법을 제공합니다. 아래 이미지와 같이 `Observable`은 `interface`입니다. 

  ![img_1.png](img_1.png)  
`Observable`을 구현한 클래스들은 `LiveData`처럼 변경 사항을 알릴 수 있는 관찰 가능한 클래스가 됩니다.
`Observable` 객체는 클래스 내에서 관찰된 프로퍼티가 변경될 때마다 `Observable.OnPropertyChangedCallback`에 변경 사항을 전달해야 합니다.

<br>

> **ObservableField란 무엇인가?**  

`ObservableField`란 객체를 관찰할 수 있도록 만들어주는 래퍼로, 객체 뿐만 아니라 필드의 변경 사항도 관찰할 수 있습니다.
변경 사항이 관찰될 때 UI가 자동으로 업데이트되어, UI와 데이터 모델 간의 상호작용을 쉽게 만듭니다.

`ObservableField.class`  

![img_2.png](img_2.png)

위 이미지에서 `ObservableField`가 `androidx.databinding` 패키지에 포함되어 있는 것을 확인할 수 있습니다.
<br>

> **ObservableField의 장점과 단점**

**ObservableField의 장점**
- **직관적이고 쉬움**  
  `ObservableField`는 사용법이 매우 직관적이어서, LiveData와 달리 쉽게 이해하고 사용할 수 있습니다. 특히 데이터 바인딩과 함께 사용할 때 유용하며, 추가적인 라이브러리나 설정 없이 바로 데이터를 바인딩할 수 있습니다.

**ObservableField의 단점**
- **생명주기 인식 X**  
  `ObservableField`는 생명주기(LifecycleOwner)를 인식하지 못합니다. 즉, Activity나 Fragment가 비활성화 상태에 있어도 값이 변경될 수 있으며, 이로 인해 메모리 누수 등의 문제가 발생할 수 있습니다.


- **비동기 작업에 적합하지 않음**  
  `ObservableField`는 비동기 작업을 지원하지 않기 때문에 네트워크 작업이나 데이터베이스 쿼리와 같은 비동기 처리를 위해서는 별도의 방법이 필요합니다. 또한, 비동기 작업이 완료된 후 결과를 UI에 반영하려면 추가적인 코드 작성이 필요합니다.

<br>

> **ObservableField의 사용 예제**  

**1. ObservableField 선언**  
관찰할 값을 `ObservableField`로 감싸줍니다.

`ViewModel.kt`
```kotlin
class ObservableFieldViewModel : ViewModel() {
    val title = ObservableField<String>()
    ...
}
```

**2. 값 업데이트**  

![img_5.png](img_5.png)  
`get()`과 `set()`을 이용해 값을 읽고 업데이트할 수 있습니다.

`ViewModel.kt`
```kotlin
class ObservableFieldViewModel : ViewModel() {
    val title = ObservableField<String>()
    
    fun getTitle() {
        title.get()
    }

    fun setTitle(newTitle: String) {
        title.set(newTitle)
    }
}
```

**3. 변경사항 관찰(observe)**  
`addOnPropertyChangedCallback`을 활용해 `Observable`의 변경 사항을 감지할 수 있습니다.

`Activity.kt`
```kotlin
class MainActivity : AppCompatActivity() {
    private val viewModel: ObservableFieldViewModel by viewModels()
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        setContentView(binding.root)
        observeTitle()
        ...
    }
    
    private fun observeTitle() {
        viewModel.title.addOnPropertyChangedCallback(object : Observable.OnPropertyChangedCallback() {
            override fun onPropertyChanged(sender: Observable?, propertyId: Int) {
                // 변화가 감지 되었을 때 수행할 작업
            }
        })
    }
}
```

<br>

### LiveData란?
> **LiveData란 무엇인가?**  

`LiveData`는 앱의 데이터를 관찰 가능한 형태로 제공하는 `데이터 홀더 클래스`입니다.
UI 컴포넌트와 ViewModel, 모델 간의 상호 작용을 쉽게 해주며 데이터의 변경을 자동으로 감지하여 UI에 즉시 업데이트할 수 있도록 합니다.
![img_3.png](img_3.png)

<br>

> **LiveData의 장점과 단점**  

**LiveData의 장점**
- **UI가 데이터 상태와 일치하는지 보장**  
 LiveData는 관찰자 패턴을 따릅니다. 기본 데이터가 변경될 때 `Observer` 객체에 알려 UI를 업데이트할 수 있습니다. 즉, 앱 데이터가 변경될 때마다 관찰자가 UI를 업데이트하므로 개발자가 업데이트할 필요가 없습니다.


- **메모리 누수 방지**  
`Observer`는 `Lifecycle` 객체에 결합되어 있으며 연결된 수명 주기가 끝나면 자동으로 삭제되기 때문에 메모리 누수를 방지할 수 있습니다.


- **Activity가 중단된 동안 크래시 발생 X**  
  Activity가 백 스택에 있을 때를 비롯해 `Observer`의 수명 주기가 비활성 상태라면, `Observer`는 어떤 `LiveData` 이벤트도 받지 않기 때문에 Activity가 중단된 동안 크래시 발생할 위험이 없습니다.


- **생명 주기를 자동으로 관리**
`LiveData`는 Observer를 통해서 생명주기를 자동으로 관리해주기 때문에 UI 컴포넌트는 그저 관련 데이터를 관찰하기만 하면 됩니다.


- **적절한 구성 변경**  
화면 회전 시 새로 생성된 Activity나 Fragment에서도 LiveData를 관찰하여 즉시 최신 데이터를 다시 받을 수 있습니다. 이는 LiveData가 수명 주기와 결합되어 있어, 관찰자(Observer)가 재등록될 때 최신 데이터를 자동으로 제공하기 때문입니다.

 **LiveData의 단점**
- **비동기 데이터 스트림 제한**  
`비동기 데이터 스트림`이란, 시간의 흐름에 따라 비동기적으로 발생하는 데이터를 연속적으로 전달하는 흐름을 말합니다. `LiveData`는 Data Layer에서 비동기 데이터 스트림을 처리하도록 설계된 도구가 아닙니다.  
`LiveData`를 변형하거나 `MediatorLiveData`를 활용해 Data Layer에서 `LiveData`를 사용할 수는 있으나 이는 메인 스레드에서 호출되는 비용이 많이 드는 호출이며, UI에서 심각한 끊김 현상을 유발할 수 있습니다.

<br>

> **LiveData의 사용 예제**  

**1. LiveData 선언**  
`MutableLiveData`로 선언하여 name의 변경사항을 트래킹할 수 있게 합니다.

`ViewModel.kt`
```kotlin
class LiveDataViewModel : ViewModel() {
    private val _name = MutableLiveData<String>()
    val name: LiveData<String> get() = _name
    
    ...
}
```
일반적으로 `MutableLiveData`을 `public`하게 노출하기보다, 관찰 가능하지만 변경이 불가능한 `LiveData` 타입의 값을 노출하곤 합니다. `Backing Property`의 형태를 볼 수 있습니다.

**2. 값 업데이트**  

![img_4.png](img_4.png)

`MutableLiveData` 타입을 활용하면 값을 업데이트할 수 있습니다. `MutableLiveData`는 `setValue(T)`와 `postValue(T)` 두 가지 메서드를 제공합니다.  
`setValue(T)`는 메인 스레드에서 호출해야 합니다. 백그라운드 스레드에서 값을 설정해야 하는 경우 `postValue(T)`를 사용해야 합니다. `postValue(T)` 는 값을 바로 변경하지 않고 메인 스레드에게 작업을 할당합니다. 

`ViewModel.kt`
```kotlin
class LiveDataViewModel : ViewModel() {
    private val _name = MutableLiveData<String>()
    val name: LiveData<String> get() = _name

    fun updateName(name: String) {
        _name.value = name
    }
}
```

**3. 변경사항 관찰(observe)**

`Activity.kt`
```kotlin
class MainActivity : AppCompatActivity() {
    private val viewModel: LiveDataViewModel by viewModels()
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        setContentView(binding.root)
        observeName()
        ...
    }
    
    private fun observeName() {
        viewModel.name.observe(this) {
            // 변화가 감지 되었을 때 수행할 작업
        }
    }
}
```

<br>

## 3️⃣ ObservableField와 LiveData 비교
### 두 데이터 홀더의 개념적 차이점


|    특성     |   	ObservableField	   |                    LiveData                     
|:---------:|:---------------------:|:-----------------------------------------------:|
| 생명 주기 인식  |     생명주기를 인식하지 않음     |     생명주기를 인식하며, UI가 활성 상태일 때만 관찰자에게 데이터 전달      |
|  바인딩 방식   | 데이터 바인딩 라이브러리와 함께 사용됨 |            데이터 바인딩을 지원하며 생명 주기까지 관리             |
| 비동기 작업 처리 | 비동기 작업을 위한 특별한 기능 없음  | ViewModel과 Coroutines 또는 RxJava와 결합하면 비동기 처리 가능 |
|   구독 관리   | 수동으로 값을 업데이트하거나 관찰 필요 |            자동 구독 및 해제 관리 (메모리 누수 방지)            |
|   상태 관리   |       단일 값만 관리        |              복잡한 상태 관리 및 데이터 변환 가능              
| 변환 기능 | 값 변환 기능이 없음	|        Transformations를 통해 값 변환 및 조합 가능         |

`ObservableField`와 `LiveData`는 데이터 관리를 위해 사용되지만, 각각의 특징이 다릅니다.   
`ObservableField`는 생명 주기를 인식하지 않고, 데이터 바인딩을 위해 주로 사용됩니다. 반면, `LiveData`는 생명 주기를 인식해 UI가 활성 상태일 때만 데이터 업데이트를 제공하여 메모리 누수를 방지합니다.  
`LiveData`는 [Transformations](https://developer.android.com/reference/androidx/lifecycle/Transformations)를 통해 값 변환이 가능하고, `ViewModel`과의 결합으로 비동기 작업 처리에 유리한 반면, `ObservableField`는 단일 값 관리와 수동 관찰이 필요하며, 비동기 처리를 위한 추가 기능이 없습니다.

<br>

## 4️⃣ ObservableField와 LiveData를 활용한 데이터 바인딩
우선 데이터 바인딩을 사용하려면 `build.gradle.kts(::app)` 파일에 `dataBinding` 사용 옵션을 설정해야 합니다.
```kotlin
android {
    ...
    buildFeatures {
        dataBinding = true
    }
}
```

`dataBinding` 사용 옵션을 설정 했다면 xml 파일에서 `layout` 태그를 설정해야 합니다. `option + enter(mac 기준)`를 활용하면 IDE가 자동으로 태그를 생성해 줍니다.

```xml
<layout xmlns:android="http://schemas.android.com/apk/res/android" 
    xmlns:app="http://schemas.android.com/apk/res-auto">
    
    <data>
        
        <variable
            name="viewmodel"
            type="com.myapp.data.ViewModel" />
    </data>
    ...
</layout>
```

<br>

### 단방향 데이터 바인딩에서의 ObservableField와 LiveData
**ObservableField를 이용해 DataBinding에서 값 변경사항 관찰하기**  

`ObservableField` 사용 예제에서 보았듯이 `DataBinding`을 사용하지 않는 경우, `Observable`의 변경 사항을 감지하기 위해 `addOnPropertyChangedCallback`을 활용해야 했습니다.
`DataBinding`을 사용하면 더 이상 `Activity`나 `Fragment`에서 수동으로 관찰할 필요가 없습니다.

`EditText`의 `text` 속성에 `ObservableFieldViewModel`이 관찰하고 있는 title 변수를 연결하면, 값이 변경될 때 UI에 자동으로 반영됩니다.

`activity_main.xml`
```xml
<layout ...

    <data>
    
        <variable
            name="viewmodel"
            type="com.example.ui.ObservableFieldViewModel" />
    </data>
    
    <EditText
        ...
        android:text="@{viewmodel.title}" />
```

`MainActivity.kt`
```kotlin
class MainActivity : AppCompatActivity() {
    private val binding: ActivityMainBinding by lazy { DataBindingUtil.setContentView(this, R.layout.activity_main) }
    private val viewModel: ObservableFieldViewModel by viewModels()
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        setContentView(binding.root)
        setupBinding()
        ...
    }
    
    private fun setupBinding() {
        binding.lifecycleOwner = this
        binding.viewmodel = viewModel
    }
}
```

**LiveData를 이용해 DataBinding에서 값 변경 사항 관찰하기**

`LiveData`를 이용해 `DataBinding`에서 변경 사항을 관찰하는 방식도 `ObservableField`와 동일합니다.

`activity_main.xml`
```xml
<layout ...

    <data>
    
        <variable
            name="viewmodel"
            type="com.example.ui.LiveDataViewModel" />
    </data>
    
    <EditText
        ...
        android:text="@{viewmodel.name}" />
```

`MainActivity.kt`
```kotlin
class MainActivity : AppCompatActivity() {
    private val binding: ActivityMainBinding by lazy { DataBindingUtil.setContentView(this, R.layout.activity_main) }
    private val viewModel: ExampleViewModel by viewModels()
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        setContentView(binding.root)
        setupBinding()
        ...
    }
    
    private fun setupBinding() {
        binding.lifecycleOwner = this
        binding.viewmodel = viewModel
    }
}
```

<br>

### 양방향 데이터 바인딩에서의 ObservableField와 LiveData
`ObservableField`와 `LiveData`를 활용한 양방향 데이터 바인딩은 간단하게 구현할 수 있습니다. 기본적으로, `@={}` 구문을 사용해 바인딩을 설정하면 양방향 데이터 바인딩을 쉽게 적용할 수 있습니다.

`activity_main.xml`
```xml
<layout ...

    <data>
    
        <variable
            name="viewmodel"
            type="com.example.ui.ExampleViewModel" />
    </data>
    
    <EditText
        ...
        android:text="@={viewmodel.value}" />
```
<br>

### ObservableField vs LiveData 선택 기준
**`LiveData`와 `ObservableField` 중 어떤 걸 사용해야 할까요?**
> **성능 및 메모리 관리 측면에서의 비교**
>
`LiveData`와 `ObservableField`는 모두 Android에서 UI와 데이터를 동기화하는 데 유용한 도구이지만, 두 가지에는 중요한 차이가 있습니다. 
`LiveData`는 Android의 `Lifecycle-Aware` 컴포넌트로, 데이터 변경을 UI에 반영하는 과정에서 Activity나 Fragment의 수명주기를 인식합니다. 
반면, `ObservableField`는 수명주기를 알지 못해 데이터와 UI를 자동으로 관리하지 못하므로, 관찰자가 계속 데이터를 관찰하고 있기 때문에 리소스가 낭비됩니다.

정답은 없지만, UI 수명주기를 인식하며 자동 관리가 필요한 경우에는 LiveData가 적합하고, 수명주기와 무관한 간단한 데이터 바인딩에는 ObservableField가 효율적입니다. 각 상황에 맞는 도구를 선택하여 사용하는 것이 중요합니다.

## 참고자료
- [안드로이드 공식 문서_DataBinding](https://developer.android.com/topic/libraries/data-binding)
- [안드로이드 공식 문서_Observable](https://developer.android.com/reference/android/databinding/Observable)
- [안드로이드 공식 문서_Work with observable data objects](https://developer.android.com/topic/libraries/data-binding/observability)
- [안드로이드 공식 문서_LiveData](https://developer.android.com/topic/libraries/architecture/livedata)
- [안드로이드 공식 문서_MutableLiveData](https://developer.android.com/reference/androidx/lifecycle/MutableLiveData)
- 실무에 바로 적용하는 안드로이드 프로그래밍
- 우아한테크코스 모바인 안드로이드 레벨2 - 2024 강의 자료

