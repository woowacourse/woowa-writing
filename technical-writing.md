# ViewModel 수명 범위 확장을 통한 상태 보관

---

## 1. 개요

ViewModel은 UI 상태를 **구성 변경** 이후에도 보관하도록 설계된 Jetpack 아키텍처 컴포넌트이다.

ViewModel은 기본적으로 **ViewModelStoreOwner**에 결합되어 있으며,
ViewModelStore는 프로세스 내에서 소유자 수명과 함께 유지·소멸된다.

| 구분 | ViewModel 소유자 | 수명 |
| --- | --- | --- |
| Fragment | Fragment | Fragment 소멸 시까지 |
| Activity | Activity | Activity 종료 시까지 |
| 부모 Fragment | 상위 Fragment | 부모 Fragment 소멸 시까지 |

**아래는 ViewModel의 생명주기 범위를 확장하여 상태를 보관하는 방법**을 실전 경험과 함께 설명한다.

---

## 2. 상태 보관이 중요한 이유

앱의 신뢰도와 사용자 경험(UX)은 상태 유지를 얼마나 잘 처리하느냐에 따라 크게 달라진다.

- 화면 회전, 프로세스 재시작, 화면 전환 등은 모두 Fragment 재생성을 유발한다.
- ViewModel 범위를 잘못 지정하면 사용자 입력 상태나 API 응답이 유실된다.

이유: ViewModel 범위 확장은 사용자의 행동 플로우를 끊지 않고 이어줄 수 있도록 한다.

---

## 3. ViewModel의 수명 범위 종류

+ AndroidX 1.8 이후부터는 `navGraphViewModels()`를 통해 Navigation 그래프 단위 스코프도 지원한다.
+ 이는 Fragment/Activity 스코프 중간 수준으로, 다중 Fragment 전환 시에도 일관된 상태 유지가 가능하다.

### 3.1 Fragment 범위 (기본 스코프)

```kotlin
private val viewModel by viewModels<MyViewModel>()
```

**특징**: Fragment 재생성과 함께 ViewModel도 소멸
**적합한 경우**
    - 단일 화면(UI 단위)의 임시 상태 관리
    - 독립적 입력 폼, 검색 필드 등
**제한점**
    - Fragment 전환 시 ViewModel이 새로 생성
    - Fragment 간 공유 불가능

**핵심**: UI 중심의 단일 화면 상태를 관리할 때 적합하다.

---

### 3.2 Activity 범위

```kotlin
private val viewModel by viewModels<MyViewModel>(
    ownerProducer = { requireActivity() }
)
```

- **특징**: Activity 종료 전까지 ViewModel 유지
- **장점**:
    - 여러 Fragment 간 상태 공유 가능
    - 탭 전환, BottomSheet, 다중 Fragment 간 일관성 유지
- **주의점**:
    - 불필요한 데이터가 Activity 전역에 남을 수 있음

**핵심**: Activity 범위는 화면 묶음 단위 상태 공유에 이상적이다.

---

### 3.3 상위 Fragment 범위

```kotlin
private val parentViewModel by viewModels<ParentViewModel>(
    ownerProducer = { requireParentFragment() }
)
```

- **특징**: 부모 Fragment가 살아있는 동안 유지
- **적합한 경우**:
    - 부모-자식 관계가 강하게 연결된 구조
    - 여러 하위 Fragment가 부모의 상태를 참조
- **주의점**:
    - 구조적 결합이 강해 Fragment 재사용성 저하

**핵심**: 부모-자식 간 의존이 불가피한 경우에만 선택한다.

---

## 4. ViewModel 스코프 선택 가이드라인

| 상황 | 권장 스코프 | 설명 |
| --- | --- | --- |
| 독립된 단일 화면 | Fragment | Fragment 재생성과 함께 소멸 |
| 탭 전환 간 상태 공유 | Activity | Activity 단위로 상태 일관성 유지 |
| 부모-자식 강결합 | 상위 Fragment | 부모의 상태를 자식이 공유 |
| BottomSheet, Dialog 등 일시 화면 | Activity | Activity 스코프로 올려 상태 유지 |

스코프를 넓히면 공유가 쉬워지지만, 메모리 비용과 결합도가 증가한다.

---

## 5. 실전 예제 — BottomSheet 상태 유지

### 5.1 문제 상황

상황: 요구상세 토론방 화면에는 **댓글 목록을 보여주는 BottomSheet**가 있다.

이 BottomSheet 안에는 사용자가 댓글을 입력할 수 있는 **입력 내용**도 포함되어 있다.

문제: `BottomSheetDialogFragment`는 기본적으로 **독립된 ViewModelStore**를 가진다.

- 닫혔다가 다시 열리면 새로 생성되고,
- ViewModel의 범위가 BottomSheet 내부로 한정되어 있으면 입력 중이던 내용이 사라진다.

따라서 닫힐 경우 작성 했던 내용도 사라진다.

```kotlin
private val viewModel by viewModels<CommentViewModel>()
```

요약:

- BottomSheet 닫힘
- ViewModel 소멸
- 입력 데이터 유실

---

### 5.2 해결 전략 — Activity 범위로 확장

**Activity를 ViewModelStoreOwner로 지정**한다.

```kotlin
class CommentsBottomSheet : BottomSheetDialogFragment() {

    private val viewModel by viewModels<CommentsViewModel>(
        ownerProducer = { requireActivity() },
        factoryProducer = {
            val repositoryModule = (requireActivity().application as App).container.repositoryModule
            CommentsViewModelFactory(
                repositoryModule.commentRepository,
                repositoryModule.tokenRepository,
            )
        },
        extrasProducer = {
            MutableCreationExtras(requireActivity().defaultViewModelCreationExtras).apply {
                this[DEFAULT_ARGS_KEY] = buildCommentArgs()
            }
        },
    )
}
```

✅ **효과**

- BottomSheet 닫혀도 ViewModel 유지
- Activity 재생성 시에도 `SavedStateHandle`을 통해 자동 복원

---

## 6. 실전 예제 CreationExtras로 초기 인자 전달

### 6.1 문제 상황

`BottomSheetDialogFragment`에서 댓글 입력 상태를 관리하는 `CommentsViewModel`은
**Activity 스코프**로 유지되어야 했다.

이유는, BottomSheet가 닫혔다가 다시 열릴 때 **입력 중이던 댓글 내용이 사라지지 않도록** 하기 위함이다.

그래서 `ownerProducer = { requireActivity() }`로 ViewModel의 생명주기를 Activity에 맞췄다.

하지만 이 변경으로 인해 새로운 문제가 발생했다.

> discussionId 값이 SavedStateHandle로 전달되지 않아 ViewModel이 초기화될 때 인자를 읽지 못하는 문제.

---

### 6.2 문제 원인

`viewModels()`는 내부적으로 `CreationExtras`를 자동 생성한다.

이때 `extrasProducer`를 지정하지 않으면, 기본적으로 **owner의 `defaultViewModelCreationExtras`** 를 사용한다.

- `owner = Fragment`인 경우 → `Fragment.arguments`가 `DEFAULT_ARGS_KEY`에 자동 포함됨
- `owner = Activity`인 경우 → **Activity의 extras만 포함**, `Fragment.arguments`는 전달되지 않음

즉, ViewModel을 Activity 범위로 확장한 순간, Fragment에 전달하는 `discussionId`가 포함되지 않게 된다.

결과적으로 `SavedStateHandle`이 비어 있는 상태로 생성되어 다음 코드가 실패한다.

```kotlin
val discussionId: Long =
    requireNotNull(savedStateHandle[Long](KEY_DISCUSSION_ID))
```

---

### 6.3 해결 방법: extrasProducer에서 수동 전달

Fragment의 `arguments`를 `DEFAULT_ARGS_KEY`에 직접 넣어 `SavedStateHandle`이 초기 인자를 복원할 수 있도록 한다.
단, `requireActivity()` 스코프 사용 시 ViewModel은 Activity 전역에서 공유되므로,
`discussionId` 등 식별값이 다른 Fragment에서 중복되지 않도록 주의해야 한다.

```kotlin
private val viewModel by viewModels<CommentsViewModel>(
    ownerProducer = { requireActivity() },
    factoryProducer = {
        val repositoryModule = (requireActivity().application as App).container.repositoryModule
        CommentsViewModelFactory(
            repositoryModule.commentRepository,
            repositoryModule.tokenRepository,
        )
    },
    extrasProducer = {
        MutableCreationExtras(requireActivity().defaultViewModelCreationExtras).apply {
            this[DEFAULT_ARGS_KEY] = buildCommentArgs()
        }
    },
)
```

---

### 6.4 buildCommentArgs 구현

```kotlin
private fun Fragment.buildCommentArgs(): Bundle {
    val id = requireArguments().getLong(CommentsViewModel.KEY_DISCUSSION_ID, -1L)
    check(id != -1L) { "Missing ${CommentsViewModel.KEY_DISCUSSION_ID}" }
    return bundleOf(CommentsViewModel.KEY_DISCUSSION_ID to id)
}
```

- Fragment의 `arguments`에서 `discussionId`를 가져와 검증
- `DEFAULT_ARGS_KEY`로 넣어주면, `SavedStateHandle` 초기화 시 자동으로 복원됨

---

### 6.5 ViewModel에서의 처리

```kotlin
class CommentsViewModel(
    private val commentRepository: CommentRepository,
    private val tokenRepository: TokenRepository,
    savedStateHandle: SavedStateHandle,
) : ViewModel() {

    val discussionId: Long =
        requireNotNull(savedStateHandle[Long](KEY_DISCUSSION_ID)) {
            "Missing $KEY_DISCUSSION_ID"
        }

    private val _commentInput = MutableStateFlow("")
    val commentInput: StateFlow<String> = _commentInput
}
```

---

## 정리

| 항목 | Fragment 스코프 | Activity 스코프 |
| --- | --- | --- |
| arguments 자동 전달 | ✅ 자동 포함 | ❌ 포함 안 됨 |
| `SavedStateHandle` 초기화 | 자동 | 직접 extrasProducer에서 세팅 필요 |
| ViewModel 생명주기 | BottomSheet 단위 | Activity 단위 |
| 입력 상태 유지 | 닫으면 소멸 | 닫혀도 유지 |

---

## 7. Fragment.viewModels 내부 구조

## 내부 구조 요약

| 매개변수 | 핵심 역할 | 기본값 | 커스터마이징 시점 | 실수 포인트 |
| --- | --- | --- | --- | --- |
| `ownerProducer` | **스코프 결정** → `owner.viewModelStore` | `{ this }`(Fragment) | Activity·부모 Fragment 등 **수명 확장**이 필요할 때 | 스코프 올리면 `Fragment.arguments`가 자동 전달되지 않음 |
| `extrasProducer` | **초기 인자 전달** → `CreationExtras` → `SavedStateHandle` | `owner.defaultViewModelCreationExtras` | Activity 스코프에서 **Fragment.arguments를 수동 주입**할 때 | `DEFAULT_ARGS_KEY`에 `Bundle` 미설정 시 값 유실 |
| `factoryProducer` | **생성 전략** → 의존성 주입·커스텀 생성자 | `owner.defaultViewModelProviderFactory` | 리포지토리 등 **의존성 주입** 필요할 때 | `SavedStateHandle` 파라미터 누락, Context 보관 누수 |

---

## 예시 적용 포인트

```kotlin
private val viewModel by viewModels<CommentsViewModel>(
    ownerProducer = { requireActivity() }, // 스코프: Activity로 확장
    factoryProducer = {
        val repo = (requireActivity().application as App).container.repositoryModule
        CommentsViewModelFactory(repo.commentRepository, repo.tokenRepository) // 의존성 주입
    },
    extrasProducer = {
        MutableCreationExtras(requireActivity().defaultViewModelCreationExtras).apply {
            this[DEFAULT_ARGS_KEY] = buildCommentArgs() // Fragment.arguments를 SavedStateHandle로 연결
        }
    },
)
```

---

## 8. ViewModel 생명주기 흐름
ViewModel은 Activity의 ViewModelStore에 저장되며,
동일 Activity 내의 다른 Fragment에서도 동일 인스턴스를 획득한다.
단, 프로세스가 완전히 종료되면 SavedStateHandle을 통해 일부 상태만 복원된다 (in-memory 상태는 소실).

```
Activity onCreate()
 └── ViewModelStore 생성
     └── ViewModelProvider 초기화
         └── CommentsViewModel 생성
BottomSheet 열림
 └── requireActivity().viewModelStore 에서 ViewModel 재사용
BottomSheet 닫힘
 └── ViewModel 유지
Activity 종료
 └── ViewModelStore 폐기 → onCleared() 호출
```

---

## 9. 스코프별 비교 요약

| 스코프 | 생명주기 | 상태 유지 | 공유 범위 | 주요 사용처 |
| --- | --- | --- | --- | --- |
| Fragment | Fragment 소멸 시 종료 | ❌ | 단일 화면 | 화면 전용 UI 상태 |
| Activity | Activity 소멸 시 종료 | ✅ | 여러 Fragment | 탭 전환, BottomSheet |
| 상위 Fragment | 부모 소멸 시 종료 | ✅ | 자식 Fragment | 부모-자식 상태 공유 |

---

## 10. 결론

ViewModel 스코프는 **데이터의 공유 단위**이자 **메모리 생명주기 단위**다.
BottomSheet, Dialog 등은 Activity 범위로 확장 시 UX 개선에 효과적이지만,
**Activity 재사용이 불필요할 때는 Fragment 스코프 유지가 더 안전**하다.

| 상황 | 권장 스코프 |
| --- | --- |
| 단일 화면 | Fragment |
| 탭 전환, BottomSheet | Activity |
| 부모-자식 강결합 | 상위 Fragment |

> ViewModel 수명 범위를 명확히 정의하면, 사용자 경험을 향상 시킬 수 있다.
