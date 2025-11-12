# **사용자를 기다리게 하지 않는 기술, 낙관적 UI 구현기**

## **들어가며: 사용자는 기다려주지 않는다**

현대의 디지털 환경에서 사용자는 거의 즉각적인 피드백을 기대합니다. 연구에 따르면 100밀리초를 넘는 지연은 사용자의 작업 흐름을 방해하며 앱이 느리고 신뢰할 수 없다는 인상을 준다. 로딩 스피너나 비활성화된 버튼은 시스템이 바쁘다는 신호지만, 동시에 사용자에게 "지금 당신은 아무것도 할 수 없습니다"라는 메시지를 전달하며 사용자를 수동적인 관찰자로 만든다.

이 문제의 핵심은 실제 지연 시간(latency)이 아니라, 지연에 대한 **사용자의 인식**이다. 바로 이 '인지 성능(perceived performance)'을 극대화하기 위해 고안된 전략이 **낙관적 사용자 인터페이스**(Optimistic UI)이다.

이 글은 네트워크 지연 문제로 고민하는 프론트엔드/모바일 개발자, 그리고 단순히 기능 구현을 넘어 뛰어난 사용자 경험을 제공하고 싶은 신입 개발자를 대상으로 한다. 낙관적 UI의 기본 개념부터 실제 앱('hEARit')에 적용한 코드와 데이터, 그리고 그 과정에서 마주한 고민과 배운 점까지 상세히 공유하고자 한다.

<p align="center">
    <img width="420" alt="ChatGPT Image" src="https://github.com/user-attachments/assets/b727b66c-0b22-4e6f-91ea-acb46b937c62" />
</p>
<p align="center"><sub>Image generated with <b>ChatGPT (GPT-5, 2025)</b></sub></p>

## **1. 낙관적 UI란 무엇인가?**

### **1.1. 왜 필요한가: 기다림의 심리학**

디지털 제품과의 상호작용에서 ‘기다림’은 단순히 시간이 흐르는 것이 아니라, 불확실성과 통제력 상실이라는 부정적인 감정을 유발하는 심리적 경험이다. 애플리케이션의 실제 성능이 아무리 뛰어나도, 사용자가 자신의 행동 결과를 즉시 눈으로 확인하지 못하면 그 경험은 느리고 답답하게 느껴질 뿐이다.

따라서 현대 UX 설계의 핵심 과제는 실제 성능을 개선하는 것만큼이나, 사용자가 느끼는 빠르기 즉 **인지 성능**을 향상시키는 데 있다.

### **1.2. 정의와 기본 개념: 긍정적 가정의 힘**

낙관적 UI는 사용자의 작업이 성공할 것이라고 ‘낙관적’으로 가정하고, 서버의 확인 응답을 기다리지 않고 즉시 UI를 업데이트하는 디자인 패턴이다. 로딩 상태를 제거하고 즉각적인 반응을 제공하여 지연 시간에 대한 인식을 근본적으로 없애는 것을 목표로 한다.

작동 방식은 간단하다:

1. 즉시 업데이트: 사용자가 ‘좋아요’ 버튼을 누르면, 앱은 서버 응답을 기다리지 않고 즉시 버튼 색상을 바꾸거나 숫자를 1 올린다.
2. 비동기 서버 통신: UI가 업데이트되는 동안, 백그라운드에서 서버로 실제 요청을 보낸다.
3. 상태 조정:
- 성공 시: 요청이 성공하면 아무것도 할 필요가 없다. UI는 이미 올바른 상태이다.
- 실패 시: 드물게 요청이 실패하면, UI를 이전 상태로 되돌리는 **롤백**(rollback)을 수행하고 사용자에게 실패 사실을 알린다.

이러한 접근 방식은 앱을 훨씬 빠르고 반응적으로 느끼게 만들어 사용자 만족도를 높인다.

### **1.3. 전통적 방식과의 비교**

전통적인 UI 업데이트 방식은 데이터의 정확성과 정합성을 최우선으로 하여, 서버의 명확한 성공 응답을 받은 후에 UI를 변경한다.

| 기능 | 전통적 방식                                           | 낙관적 UI (Optimistic UI) |
| --- |--------------------------------------------------| --- |
| 사용자 피드백 | 지연됨 (서버 응답 대기)                                   | 즉각적 (성공 가정) |
| 주요 목표 | 데이터 무결성 및 정합성                                    | 인지 성능 및 반응성 |
| 작업 흐름 | 1. 행동 → 2. 로딩 표시 → 3. 서버 요청 → 4. 응답 → 5. UI 업데이트 | 1. 행동 → 2. UI 업데이트 → 3. 서버 요청 → 4. 상태 조정 (필요시) |
| 구현 복잡도 | 낮음                                               | 높은 (롤백을 위한 상태 관리 필요) |
| 이상적 사용 사례 | 금융 거래, 중요한 양식 제출                                 | ‘좋아요’, 댓글, 장바구니 추가, 채팅 메시지 |

두 방식 중 어느 것이 절대적으로 우월하다고는 할 수 없으며, 적용할 기능의 맥락에 따라 신중하게 선택해야 한다. 예를 들어, 데이터 정확성이 무엇보다 중요한 금융 거래에는 전통적 방식이, 빠른 반응이 중요한 소셜 미디어 기능에는 낙관적 UI가 적합하다.

## **2. hEARit 북마크 기능에 낙관적 UI 적용하기**

이론을 넘어, 직접 개발한 [‘hEARit’](https://github.com/woowacourse-teams/2025-hEARit) 앱의 북마크 기능에 낙관적 UI를 적용한 실제 사례를 코드를 통해 살펴보자.

### **2.1. 기존 흐름: 서버 응답 후 UI 업데이트**

기존의 북마크 기능은 전형적인 방식으로 구현되었다.

1. 사용자가 북마크 버튼을 누른다.
2. 로딩 상태를 `true`로 변경하여 UI를 비활성화한다.
3. 서버에 북마크 추가/삭제 API를 요청한다.
4. 서버로부터 성공 응답을 받는다.
5. 로딩 상태를 `false`로 바꾸고 북마크 UI를 최종 상태로 업데이트한다.

아래는 hEARit 앱의 기존 북마크 코드이다.

```kotlin
// PlayerDetailViewModel.kt

private suspend fun pessimisticToggleBookmark() {
    // 1. 로딩 상태를 true로 설정하고 UI에 알림
    _uiState.value = _uiState.value?.copy(isLoading = true)

    if (_bookmarkId.value != null) { // 북마크 삭제 로직
        val id = _bookmarkId.value!!
        bookmarkRepository
            .deleteBookmark(id)
            .onSuccess {
                // 2. 서버 응답 성공 후, UI 상태 업데이트
                _bookmarkId.value = null
                _uiState.value = _uiState.value?.copy(isBookmarked = false, isLoading = false)
            }.onFailure { t ->
                // 3. 실패 시 에러 처리
                _uiState.value =
                    _uiState.value?.copy(
                        isLoading = false,
                        errorResId = R.string.all_toast_delete_bookmark_fail,
                    )
            }
    } else { // 북마크 추가 로직
        bookmarkRepository
            .addBookmark(hearitId)
            .onSuccess { id ->
                // 2. 서버 응답 성공 후, UI 상태 업데이트
                _bookmarkId.value = id
                _uiState.value = _uiState.value?.copy(isBookmarked = true, isLoading = false)
            }.onFailure { t ->
                // 3. 실패 시 에러 처리
                // ... (생략)
            }
    }
}
```

이 방식의 가장 큰 문제는 **네트워크 지연**에 매우 취약하다는 것이다.

### **2.2. 개선된 흐름: UI 선반영 후 서버 요청**

사용자 경험을 개선하기 위해, 위 로직을 낙관적 방식으로 전환했다.

1. 사용자가 북마크 버튼을 누른다.
2. **서버를 기다리지 않고**, 즉시 북마크 UI 상태를 **예상되는 결과**로 변경한다.
3. 백그라운드에서 서버에 API를 요청한다.
4. 요청이 성공하면 아무것도 하지 않는다.
5. 요청이 **실패**하면, UI 상태를 이전으로 **롤백**하고 사용자에게 알린다.

개선된 북마크 코드는 다음과 같다.

```kotlin
// PlayerDetailViewModel.kt

private suspend fun optimisticToggleBookmark() {
    val isBookmarked = _bookmarkId.value != null
    val originalBookmarkId = _bookmarkId.value // 롤백을 위해 이전 상태 저장

    // 1. UI 상태를 즉시 업데이트 (낙관적)
    _uiState.value = _uiState.value?.copy(isBookmarked = !isBookmarked, isLoading = true)

    if (isBookmarked) { // 북마크 삭제 로직
        val id = originalBookmarkId ?: return
        bookmarkRepository
            .deleteBookmark(id)
            .onSuccess {
                // 2. 성공 시: 로딩 상태만 해제
                _bookmarkId.value = null
                _uiState.value = _uiState.value?.copy(isLoading = false)
            }.onFailure { t ->
                // 3. 실패 시: UI 상태를 이전으로 롤백!
                Timber.w(t)
                _uiState.value =
                    _uiState.value?.copy(
                        isBookmarked = true, // 이전 상태(true)로 되돌림
                        isLoading = false,
                        errorResId = R.string.all_toast_delete_bookmark_fail,
                    )
            }
    } else { // 북마크 추가 로직
        bookmarkRepository
            .addBookmark(hearitId)
            .onSuccess { id ->
                // 2. 성공 시: 실제 ID를 저장하고 로딩 상태 해제
                _bookmarkId.value = id
                _uiState.value = _uiState.value?.copy(isLoading = false)
            }.onFailure { t ->
                // 3. 실패 시: UI 상태를 이전으로 롤백!
                Timber.w(t)
                _uiState.value =
                    _uiState.value?.copy(
                        isBookmarked = false, // 이전 상태(false)로 되돌림
                        isLoading = false,
                        errorResId = R.string.all_toast_add_bookmark_fail,
                    )
                // ... (로그인 예외 처리 생략)
            }
    }
}

```

**2.3. UI와 ViewModel 연결**

ViewModel의 상태 변화는 Activity에서 관찰하여 실제 UI 컴포넌트에 반영된다. `uiState`라는 단일 상태 객체를 사용해 UI 관련 모든 상태(북마크 여부, 로딩 상태, 에러)를 한 번에 관리함으로써 코드를 더 명확하고 일관성 있게 만들었다.

```kotlin
// PlayerDetailActivity.kt

private fun observeViewModel() {
    // ViewModel의 uiState가 변경될 때마다 호출됨
    viewModel.uiState.observe(this) { state ->
        // 1. isBookmarked 상태에 따라 북마크 아이콘 변경
        binding.baseController.setBookmarkSelected(state.isBookmarked)
        // 2. isLoading 상태에 따라 로딩 인디케이터/버튼 비활성화 처리
        binding.baseController.setBookmarkLoading(state.isLoading)
        // 3. errorResId가 있으면 토스트 메시지 표시
        state.errorResId?.let { Toast.makeText(this, getString(it), Toast.LENGTH_SHORT).show() }
    }
    // ...
}
```

이렇게 `ViewModel`은 UI의 상태를 관리하고, `Activity`는 그 상태를 화면에 그리는 역할에만 집중하는 단방향 데이터 흐름 아키텍처를 구현했다.

## **3. 적용하면서 마주한 고민들**

### **3.1. 서버 응답 지연이 UX에 미치는 영향**

낙관적 UI를 도입하며, 기존 방식과의 성능을 비교했을 때 놀라운 결과를 확인할 수 있었다. 네트워크 상태가 좋지 않을 경우, 사용자가 버튼을 누른 후 네트워크 응답까지 최대 10초가 걸렸다.

아래는 실제 성능 측정 로그이다.

```markdown
// Perf[48]: 북마크 추가 실패 케이스 (네트워크 지연) 
Perf[48] START name=BookmarkToggle attrs={policy=PESSIMISTIC, initial_isBookmarked=false}
Perf[48] MARK ui_emitted at=30441953 (+1ms) // 로딩 UI 표시까지 1ms 
Perf[48] MARK network_done at=30451974 (+10022ms) // 네트워크 응답까지 10,022ms 
Perf[48] END name=BookmarkToggle outcome=failure total=10023ms

// Perf[49]: 북마크 추가 성공 케이스 (네트워크 지연) 
Perf[49] START name=BookmarkToggle attrs={policy=PESSIMISTIC, initial_isBookmarked=false} 
Perf[49] MARK ui_emitted at=30456352 (+3ms) // 로딩 UI 표시까지 3ms 
Perf[49] MARK network_done at=30461616 (+5267ms) // 네트워크 응답까지 5,267ms 
Perf[49] END name=BookmarkToggle outcome=success total=5267ms
```
<img width="874" height="175" alt="스크린샷 2025-10-14 오후 2 52 34" src="https://github.com/user-attachments/assets/bd171280-d12b-48e5-bb4e-8aa87dca18de" />


로그를 보면,  `ui_emitted`(로딩 UI 표시) 시점과 `network_done`(서버 응답 완료) 시점 사이에 각각 10,022ms와 5,267ms의 지연이 발생했다. 이 시간 동안 사용자는 비활성화된 버튼과 로딩 스피너를 보며 아무것도 하지 못하고 기다려야 한다. 이는 사용자에게 답답함을 안겨주고 심각하게는 앱 이탈로 이어질 수 있다.

낙관적 UI는 바로 이 `network_done` **시간을 기다리지 않는다**. UI를 즉시 업데이트하여 사용자에게는 작업이 1ms만에 완료된 것처럼 느끼게 하고, 긴 네트워크 시간은 백그라운드에서 처리한다.

### **3.2. 데이터 정합성 vs UX의 균형**

낙관적 UI는 더 나은 사용자 경험을 위해 즉각적이고 강력한 일관성을 희생하는 근본적인 트레이드오프를 가진다. 클라이언트의 상태와 서버의 상태가 일시적으로 달라지는 것을 감수하고, 결국에는 데이터가 같아질 것이라는 최종적 일관성(Eventual Consistency) 모델을 수용하는 것이다.

이 선택은 항상 신중해야 한다. ‘좋아요’나 ‘북마크’처럼 실패 시 파급 효과가 적은 기능에는 낙관적 UI가 이상적이지만, 결제나 송금처럼 100% 데이터 정확성이 요구되는 중요한 트랜잭션에는 전통적 방식을 사용해 안정성을 확보해야 한다.

### 3.3. 실패 처리는 단순한 롤백 그 이상

낙관적 UI가 실패했을 때 사용자가 느끼는 좌절감은 일반적인 오류보다 훨씬 크다. 왜냐하면 시스템이 “요청이 완료되었습니다”라는 암묵적인 약속을 했다가 파기한, 즉 사용자에게 ‘거짓말’을 한 셈이 되기 때문이다.

따라서 실패 처리는 단순히 데이터를 이전으로 되돌리는 기술적 문제를 넘어, 깨진 신뢰를 회복하기 위한 UX 설계 과제로 다루어져야 한다.

실패 처리 시 고려사항:
1. **재시도 전략**: 일시적 네트워크 오류는 자동 재시도
2. **대기열 관리**: 오프라인 시 작업을 큐에 저장 후 재연결 시 처리
3. **사용자 피드백**: 토스트가 아닌 Snackbar로 "실행 취소" 옵션 제공

<p align="center">
    <img width="260" alt="Screenshot_20251014_145510" src="https://github.com/user-attachments/assets/0f27c3a3-fb59-4b11-883a-c63a0c1f5704" />
</p>


## **4. 낙관적 UI의 효과와 배운 점**

### **4.1. 장점: 놀라운 체감 속도 개선**

가장 큰 장점은 단연 인지 성능의 극적인 향상이다. 사용자는 자신의 행동에 앱이 즉각적으로 반응한다고 느끼게 되어, 더 만족스럽고 몰입감 있는 경험을 하게 된다.

### 4.2. 단점: 복잡성과 숨겨진 비용

반면, 구현 복잡도가 높아지는 것은 명백한 단점이다.

- 정교한 상태 관리: 데이터의 상태가 더 이상 ‘성공/실패’의 이진 상태가 아니라, ‘동기화됨(synced)’, ‘처리 중(pending)’, ‘오류(error)’ 등 더 많은 상태를 가질 수 있다.
- 롤백 로직: 실패 시 UI를 이전 상태로 되돌리는 롤백 로직을 견고하게 구현해야 한다.

### **4.3. 신입 개발자로서의 배움과 앞으로의 방향**

이번 프로젝트를 통해 “기능이 동작하는 것”과 사용자에게 좋은 경험을 주는 것” 사이의 간극을 깊이 이해하게 되었다. 사용자의 입장에서 ‘느리다’고 느끼는 순간을 포착하고, 이를 기술적인 해법으로 풀어내는 과정은 매우 값진 경험이었다.

## **마치며**

낙관적 UI는 즉각적인 데이터 정합성을 희생하여 대폭 향상된 반응성을 얻는 강력한 UX 패턴이다. 성공적인 구현은 단순히 코드 몇 줄을 바꾸는 것이 아닌, 견고한 오류 처리, 신뢰할 수 있는 API, 그리고 기능의 중요도와 실패 파급 효과를 고려하는 전략적 판단이 필요한 아키텍처 결정이다.

더 빠르고 유연한 사용자 경험에 대한 요구가 계속되는 한, 낙관적 패턴은 더 이상 특별한 '기법'이 아니라 모든 개발자가 알아야 할 기본 패러다임으로 자리 잡을 것이다.

### **여러분의 앱에 적용하기**

그렇다면 지금 개발 중인 앱에는 어떻게 적용할 수 있을까? 낙관적 UI를 도입하기 전에 다음 질문들을 스스로에게 던져보자:

- ✅ 이 기능의 성공률은 얼마나 되는가? (95% 이상이면 좋은 후보)
- ✅ 실패 시 사용자에게 미치는 영향은? (낮을수록 적합)
- ✅ 롤백이 가능한 작업인가? (가능해야 함)
- ✅ 사용자가 즉각적인 피드백을 기대하는가? (YES면 적합)

네 가지 질문에 모두 긍정적으로 답할 수 있다면, 낙관적 UI를 적용할 좋은 후보다. 다음 순서로 단계적으로 도입해보는 것을 추천한다:

**1단계**: '좋아요', '북마크' 같은 간단한 토글 기능  
**2단계**: 댓글, 채팅 메시지 등 생성 작업  
**3단계**: 편집, 삭제 등 복잡한 작업

이 글이 여러분의 앱을 한 단계 더 나은 사용자 경험으로 이끄는 데 작은 도움이 되었으면 좋겠다.