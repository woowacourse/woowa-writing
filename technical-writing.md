# 📘 Jetpack Compose Modifier — 선언적 UI의 핵심 요소

## 서론
Jetpack Compose는 Android UI 개발의 패러다임을 **명령형(Imperative)** 에서 **선언형(Declarative)** 으로 전환한 프레임워크이다.  
개발자는 이제 더 이상 “UI를 직접 갱신하는 절차”를 작성하지 않는다.  
대신 **“UI = f(State)”**, 즉 UI는 상태(State)의 함수이며, 상태가 변화하면 Compose가 자동으로 UI를 재구성한다.  

이 글은 Compose의 철학을 구현하는 핵심 도구 **Modifier**에 집중한다.  
Modifier는 시각적 스타일, 배치, 상호작용을 하나의 선언적 체인으로 표현함으로써  
**“UI를 함수처럼 조합할 수 있게 만드는 언어적 핵심”** 역할을 한다.


---

<br>

## 1. Compose의 철학과 Modifier의 등장 배경

### 1.1 선언적 UI의 원리
Compose의 가장 중요한 철학은 **UI는 상태의 함수**라는 개념이다.

> UI = f(State)

UI는 현재 상태를 기반으로 그려지고, 상태가 변하면 Compose가 자동으로 해당 부분만 재구성(Recomposition)한다.  
이 방식은 전통적인 View 시스템의 명령형 UI 갱신(`setText()`, `setPadding()` 등)과 달리, 불변성과 일관성을 보장한다.

```kotlin
val isSelected by remember { mutableStateOf(false) }

Button(
    onClick = { isSelected = !isSelected },
    colors = ButtonDefaults.buttonColors(
        containerColor = if (isSelected) Color.Red else Color.Gray
    )
) {
    Text("Click Me")
}
```

`isSelected`가 변경되면 Compose가 자동으로 버튼 색상을 갱신한다.  
이 과정에서 UI의 **구조(Structure)** 와 **속성(Style)** 을 분리할 필요가 생겼고,  
그 해답이 바로 **Modifier**이다.

---

<br>

### 1.2 Modifier의 철학적 위치
기존 Android View에서는 View 객체 자체가 모든 속성을 포함했다.


```kotlin
view.setPadding(16)
view.setBackgroundColor(Color.Gray)
```

이는 명령형으로 UI를 갱신하는 방식으로, 재사용성과 예측 가능성이 떨어졌다.
Compose에서는 이 로직을 Modifier 체인으로 대체한다.

```kotlin
Modifier
    .padding(16.dp)
    .background(Color.Gray)
```

이 구조는 Compose의 세 가지 핵심 철학을 반영한다.

- **불변성 (Immutability)** : Modifier는 항상 새 객체를 반환한다.

- **조합성 (Composability)** : 여러 속성을 순차적으로 연결할 수 있다.

- **단일 책임 (Single Responsibility)** : 각 Modifier는 하나의 기능(레이아웃, 그리기, 입력)을 담당한다.

<br>

## 2. Modifier 구조 및 체인 동작 원리
### 2.1 불변성과 결합 구조
Modifier는 불변(immutable) 객체로 설계되어 있다.
각 연산(padding, background, clickable)은 새로운 Modifier 인스턴스를 반환한다.

```kotlin
val paddingModifier = Modifier.padding(8.dp)
val backgroundModifier = paddingModifier.background(Color.Gray)
```

paddingModifier는 변경되지 않으며, backgroundModifier는 별도의 객체로 생성된다.
이 불변성은 Compose가 Modifier를 안정적으로 캐싱하고 불필요한 Recomposition을 방지하게 한다.


<br>

### 2.2 체인 결합(Chaining) 및 then()
Modifier는 내부적으로 연결 리스트(Linked List)처럼 결합된다.
각 Modifier가 이전 Modifier를 참조하며, Compose 런타임이 이를 **Measure → Layout → Draw → Input** 단계에서 순차적으로 해석한다.

```kotlin
Modifier
    .padding(16.dp)
    .background(Color.Black)
    .clickable { /* … */ }
```

<br>

### 2.3 Node API (Compose 1.5 이후)
Compose 1.5+ 버전부터 Modifier는 **Node API** 기반으로 재설계되었다.
각 Modifier가 Node 단위로 분리되어 측정, 레이아웃, 그리기 등의 특정 단계만 담당한다.

Node API의 장점은 다음과 같다.

- Modifier 간 상호 간섭 감소
- 성능 및 메모리 최적화
- Modifier 간 데이터 공유를 위한 ModifierLocal 지원

```kotlin
Modifier
    .then(CustomLayoutModifier())
```
Node 기반 Modifier는 Compose Multiplatform에서도 공통적으로 활용되어, 플랫폼 간 호환성을 확보한다.
<br>(출처: Create custom modifiers – developer.android.com)
<br>
<br>

## 3. Modifier 순서의 중요성
Modifier의 적용 순서에 따라 UI 결과가 완전히 달라진다.
Compose는 Modifier를 적용 순서대로 해석한다.

```kotlin
// 예시 1
Modifier
    .padding(16.dp)
    .background(Color.Gray)

// 예시 2
Modifier
    .background(Color.Gray)
    .padding(16.dp)
```

1️⃣ padding → background	배경 밖으로 16dp 여백 생김
2️⃣ background → padding	배경 안쪽에 16dp 여백 생김
<br>

레이아웃 관련 Modifier는 앞쪽에, 시각적 Modifier는 뒤쪽에 배치하는 것이 권장된다.
이 규칙은 Compose 공식 가이드라인에 명시되어 있다.
<br>
<br>

## 4. Modifier와 성능 (Recomposition)
### 4.1 Modifier 객체 생성과 재구성
Compose는 상태가 변경될 때 관련 Composable을 재구성한다.
이 때 Modifier가 매번 새 객체로 생성되면, 불필요한 Recomposition이 발생할 수 있다.

```kotlin
// ❌ 매 호출마다 다른 Modifier 생성
Text(
    text = "Hello",
    modifier = Modifier.padding(Random.nextInt(4).dp)
)
```

### 4.2 성능 최적화 전략
- Modifier **재사용** : remember { Modifier.padding(8.dp) } 로 객체 캐싱

- Stable 객체 유지 : 불필요한 .composed {} 호출 지양

- Layout Inspector 활용 : Recomposition 횟수 시각화 및 병목 지점 분석

Compose 팀은 Performance Tips for Compose (2023) 에서 Modifier 체인 재사용을 명시적으로 권장한다.

<br>

## 5. Modifier를 활용한 UI 일관성 확보
### 5.1 디자인 시스템 구현
디자인 시스템은 색상, 간격, 모서리 반경 등의 토큰 일관성을 요구한다.
Modifier는 이러한 토큰을 코드 레벨에서 표현하는 효율적인 수단이다.

```kotlin
object AppModifiers {
    val Card = Modifier
        .clip(RoundedCornerShape(12.dp))
        .background(Color.White)
        .padding(16.dp)

    val Button = Modifier
        .height(56.dp)
        .fillMaxWidth()
        .clip(RoundedCornerShape(8.dp))
}
```

Design Token	- Modifier 표현
<br>
Spacing 16dp	- Modifier.padding(16.dp)
<br>
Radius 12dp	- Modifier.clip(RoundedCornerShape(12.dp))
<br>
Surface	- Modifier.background(AppColors.surface)

<br>

## 6. Modifier 확장 및 커스텀 Modifier 제작
### 6.1 Modifier.composed 사용 및 한계
과거에는 Modifier.composed 가 커스텀 Modifier 구현에 널리 쓰였으나,
현재는 Node API 도입으로 권장되지 않는다.

```kotlin
fun Modifier.shakeOnClick() = composed {
    val offset = remember { Animatable(0f) }
    this
        .clickable {
            offset.animateTo(10f)
            offset.animateTo(0f)
        }
        .graphicsLayer { translationX = offset.value }
}
```

⚠️ Modifier.composed 는 Recomposition 스코프에 직접 접근하므로,
성능 문제를 일으킬 수 있다.
가능하면 Modifier.Node 또는 draw/measure 확장 함수를 사용하는 것이 좋다.

### 6.2 상태를 가진 Modifier 설계 시 주의점
- Modifier.composed 내의 상태는 해당 Composable 수명에 종속된다.
- 장기 상태(long-lived state)는 ViewModel 또는 State Hoisting으로 분리한다.
- Stateless Modifier를 기본값으로 두고, 상태 관리는 UI 외부로 위임한다.

<br>

## 7. 협업 및 코드 컨벤션
### 7.1 함수 시그니처 컨벤션
Compose API 가이드라인은 다음 순서를 권장한다.

**Required → Modifier → Optional → Trailing Lambda**

```kotlin
@Composable
fun AppButton(
    onClick: () -> Unit,
    modifier: Modifier = Modifier,
    enabled: Boolean = true,
    content: @Composable () -> Unit,
)
```
출처: Compose API Guidelines

<br>
<br>

### 7.2 협업 시 리뷰 포인트
- Modifier 순서가 UI 결과에 영향을 주는지 검토
- 중복 Modifier (padding().padding()) 제거
- Modifier 내에 상태 로직이 섞이지 않았는지 확인

<br>

## 8. 잘못된 사용 패턴 및 안티패턴
문제 - 	원인	 - 해결 <br>
.padding().padding() 중복	중복 측정 발생	하나로 합치기 <br>
.fillMaxSize().width(100.dp)	상충 지시	하나만 사용 <br>
Modifier.composed 남용	불필요한 Recomposition	Stateless 우선 <br>
remember 상태 혼입	State hoisting 미사용	상태를 외부로 분리 <br>

<br>
<br>

## 9. 한계 및 향후 방향
Modifier의 불변성은 성능적 이점을 주지만, 상태 기반 인터랙션에는 제약이 있다.
이를 보완하기 위해 Compose 팀은 Node API와 ModifierLocal을 도입했다.

- ModifierLocal: Modifier 간 데이터 공유 및 상위 전달 가능
- Node API: Modifier를 컴파일러 레벨에서 최적화
- Compose Multiplatform 호환 강화

향후 Compose 릴리스에서는 이 Node 기반 아키텍처가 Compose Desktop / Web에서도 통일될 예정이다.


<br>
<br>

## 결론
Modifier는 Jetpack Compose의 철학 — **불변성, 선언성, 조합성** — 을 구현하는 핵심 도구이다.
Modifier를 적절히 설계하고 활용하는 것은 단순한 UI 꾸밈을 넘어,
**성능, 유지보수성, 협업 효율성**을 모두 높이는 핵심 역량이다.

결국 Modifier는 “속성 집합”이 아니라, Compose 언어의 문법적 단위이다.


참고 문헌
Modifiers in Jetpack Compose | Android Developers

Create custom modifiers | Android Developers

Compose API Guidelines | android.googlesource.com

Performance Tips for Compose (2023)

Android Developers Blog – Deep Dive into Modifiers and Nodes (Google I/O 2023)

