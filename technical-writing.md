📘 Jetpack Compose Modifier — 선언적 UI의 핵심 요소


1. 서론
Jetpack Compose는 Android UI 개발의 패러다임을 명령형(Imperative) 에서 선언형(Declarative) 으로 전환한 프레임워크이다.
 개발자는 이제 더 이상 “UI를 직접 갱신하는 절차”를 작성하지 않는다. 대신 “UI는 상태(State)의 함수” 라는 원칙 아래, UI를 선언하고 상태 변화를 Compose가 자동으로 반영한다.
이 글은 Compose의 철학을 구현하는 핵심 도구인 Modifier 에 집중한다.
Modifier는 시각적 스타일, 배치, 상호작용을 하나의 선언적 체인으로 표현함으로써, “UI를 함수처럼 조합할 수 있게 만드는 Compose의 언어적 핵심” 역할을 한다

2. Compose의 철학과 Modifier의 등장 배경
2.1 선언적 UI의 원리
Compose의 가장 중요한 철학은 “UI는 상태의 함수”이다.
UI = f(State)
즉, UI는 현재 상태를 기반으로 그려지는 결과이며, 상태가 변경되면 Compose는 자동으로 해당 부분만 재구성(Recomposition)한다.
 이 방식은 전통적인 View 시스템의 명령형 UI 갱신(setText(), setPadding()) 과 달리, 불변성과 일관성을 보장한다.
[Code 1] 선언적 UI의 기본 예시:
val isSelected by remember { mutableStateOf(false) }

Button(
    onClick = { isSelected = !isSelected },
    colors = ButtonDefaults.buttonColors(
        containerColor = if (isSelected) Color.Red else Color.Gray
    )
) {
    Text("Click Me")
}

여기서 isSelected가 변경되면 Button의 색상은 자동으로 다시 그려진다.
이러한 선언적 구조 속에서, Compose는 “UI의 속성”과 “UI의 구조”를 분리해야 할 필요가 있었다. 그 해답이 바로 Modifier이다.

2.2 Modifier의 철학적 위치
기존 Android View는 View 객체 자체가 모든 속성을 포함했다. 예를 들어, 배경색과 여백을 설정하려면 다음과 같은 명령이 필요했다.
view.setPadding(16)
view.setBackgroundColor(Color.Gray)

이는 상태 변경을 명령형으로 수행하는 방식으로, 재사용성이 낮고 테스트가 어렵다.
Compose에서는 이 로직이 Modifier 체인으로 대체된다.
Modifier
    .padding(16.dp)
    .background(Color.Gray)

이 구조는 다음과 같은 Compose의 세 가지 핵심 철학을 반영한다:
불변성(Immutability) — Modifier는 항상 새 객체를 반환한다.


조합성(Composability) — 여러 속성을 순차적으로 연결할 수 있다.


단일 책임(Single Responsibility) — 각 Modifier는 특정 기능(레이아웃, 그리기, 입력)을 담당한다.



3. Modifier 구조와 체인의 내부 동작 원리
3.1 Modifier의 불변성과 결합 구조
Modifier는 불변 객체(Immutable Object) 로 설계되어 있다.
 각 연산(padding, background, clickable)은 새로운 Modifier 인스턴스를 반환한다.
[Code 2] Modifier의 불변성 예시:
val paddingModifier = Modifier.padding(8.dp)
val backgroundModifier = paddingModifier.background(Color.Gray)

paddingModifier는 변경되지 않으며, backgroundModifier는 별도의 객체로 생성된다.
 이로 인해 Compose는 Modifier를 안정적으로 캐싱하고, 불필요한 UI 갱신을 방지할 수 있다.

3.2 체인 결합(Chaining)과 then()
Modifier는 내부적으로 연결 리스트(linked list) 형태로 결합된다.
 각 Modifier는 이전 Modifier를 참조하며, Compose 엔진이 이 체인을 순차적으로 해석한다.
[Code 3] Modifier 체인 결합:
Modifier
    .padding(16.dp)
    .background(Color.Black)
    .clickable { /* ... */ }

[Figure 1] Modifier 체인 내부 구조
[Composable] 
   │
   ▼
[PaddingModifier] → [BackgroundModifier] → [ClickableModifier] → [End]

Compose는 이 체인을 측정(Measure) → 배치(Layout) → 그리기(Draw) → 입력 처리(Input) 단계에서 순서대로 해석한다.

3.3 Node API의 도입 (Compose 1.5+)
Compose 1.5 이후, Modifier는 Node API 기반으로 재설계되었다.
 이 구조는 각 Modifier가 “Node 단위”로 분리되어, 특정 단계(Measure, Draw 등)만 담당하도록 최적화되었다.
[Figure 2] Modifier Node의 생명주기
Create → Attach → Measure/Layout → Draw → Detach

Node 기반 설계는 Modifier 간 상호 간섭을 줄이고, 성능을 개선하는 데 기여한다.

4. Modifier 순서의 중요성
Modifier의 순서에 따라 UI 결과가 완전히 달라진다.
 그 이유는 Compose가 Modifier를 적용 순서대로 해석하기 때문이다.
[Code 4] 순서가 다른 두 Modifier 예시:
// 예시 1
Modifier.padding(16.dp).background(Color.Gray)

// 예시 2
Modifier.background(Color.Gray).padding(16.dp)

[Figure 3] Modifier 순서 비교
예시
결과 설명
(1) padding → background
배경 밖으로 16dp 여백이 생김
(2) background → padding
배경 안쪽에 16dp 여백이 생김

이는 Compose의 측정(Layout) → 그리기(Draw) 순서와 직결된다.
 따라서 레이아웃 관련 Modifier는 앞쪽에, 시각적 Modifier는 뒤쪽에 배치하는 것이 바람직하다.

5. Modifier와 성능 (Recomposition)
5.1 Modifier의 재구성 영향
Compose는 상태가 변경되면 관련 UI를 재구성한다.
 이때 Modifier가 새로운 객체로 생성되면, Compose는 해당 Composable을 “변경된 것”으로 인식한다.
[Code 5] 잘못된 Modifier 생성 예시:
// ❌ 매 호출마다 다른 Modifier 생성
Text(
    text = "Hello",
    modifier = Modifier.padding(Random.nextInt(4).dp)
)

Modifier가 매번 새 객체로 생성되어, 불필요한 Recomposition이 일어난다.

5.2 성능 최적화 전략
Modifier 재사용: remember를 사용해 Modifier 객체를 캐싱한다.


Stable 객체 유지: 불필요한 .composed 호출을 피한다.


Layout Inspector 활용: Recomposition 횟수를 시각화하여 병목 지점 파악.



6. Modifier를 통한 UI 일관성 확보
6.1 Modifier로 디자인 시스템 구현
디자인 시스템은 색상, 간격, 모서리 반경 같은 토큰(Token)의 일관성을 요구한다.
 Modifier는 이러한 토큰을 코드 레벨에서 구현하는 가장 효과적인 수단이다.
[Code 6] Modifier 기반 디자인 토큰 정의:
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

[Table 1] 디자인 토큰 → Modifier 매핑 예시
Token
Modifier 표현
Spacing 16
Modifier.padding(16.dp)
Radius 12
Modifier.clip(RoundedCornerShape(12.dp))
Surface
Modifier.background(AppColors.surface)


7. Modifier 확장과 커스텀 Modifier 제작
7.1 Modifier.composed 활용
Modifier.composed는 커스텀 Modifier를 정의할 때 사용된다.
 Compose의 Recomposition 스코프에 접근할 수 있어, 상태나 애니메이션을 제어하는 데 유용하다.
[Code 7] shakeOnClick 예시:
fun Modifier.shakeOnClick() = composed {
    val offset = remember { Animatable(0f) }
    this.clickable {
        offset.animateTo(10f)
        offset.animateTo(0f)
    }.graphicsLayer {
        translationX = offset.value
    }
}


7.2 상태를 가진 Modifier 설계 시 주의점
Modifier.composed 내의 상태는 해당 Composable의 수명에 종속된다.


장기 상태는 ViewModel 또는 State hoisting으로 분리하는 것이 권장된다.



8. Modifier와 협업
8.1 함수 시그니처 컨벤션
Compose 가이드라인은 다음 순서를 권장한다:
Required → Modifier → Optional → Trailing Lambda
이 규칙은 코드 리뷰와 협업 시 일관성을 높인다.
@Composable
fun AppButton(
    onClick: () -> Unit,
    modifier: Modifier = Modifier,
    enabled: Boolean = true,
    content: @Composable () -> Unit,
)


8.2 협업 시 주요 리뷰 포인트
Modifier 순서가 UI 결과에 영향을 미치는지 검토


중복 Modifier (padding().padding()) 제거


Modifier 내 상태 로직 혼입 여부 확인



9. 잘못된 사용 패턴과 안티패턴
[Table 2] Modifier 사용 시 자주 발생하는 문제
문제
원인
해결 방법
.padding().padding()
중복 측정
하나로 합치기
.fillMaxSize().width(100.dp)
상충 지시
하나만 사용
Modifier.composed 남용
과도한 Recomposition
Stateless 우선
Modifier 내 remember
상태 범위 혼동
State hoisting


10. 한계와 개선 방향
Modifier의 불변성은 성능적 이점이 있지만, 상태 기반 인터랙션에는 제약을 준다.
 이를 해결하기 위해 Compose 팀은 Node API와 ModifierLocal을 도입하여 Modifier 간 데이터 전달과 캐시를 개선했다.
향후 Compose Multiplatform에서도 동일한 Node 구조가 채택되어, 플랫폼 간 Modifier 호환성이 강화될 것으로 전망된다.

11. 결론
Modifier는 Jetpack Compose의 철학 — 불변성, 선언성, 조합성 — 을 구현하는 실질적인 도구이다.
 Modifier를 올바르게 설계하고 사용하는 것은 단순한 UI 꾸밈을 넘어, 유지보수성과 협업 효율성을 높이는 핵심 역량이다.
 결국 Modifier는 단순한 속성 집합이 아니라, Compose의 언어 그 자체라 할 수 있다.

참고 문헌
Android Developers Docs – Modifiers in Jetpack Compose


Google I/O 2023 – Deep Dive into Modifiers and Nodes


AndroidX Source – androidx.compose.ui.Modifier


Compose Samples Repository – github.com/android/compose-samples


Android Developers Blog – Performance Tips for Compose (2023)





