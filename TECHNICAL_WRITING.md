# 지금 당장 polling으로 채팅을 구현하는 법 💬

## 서론

커뮤니티 어플리케이션을 개발하다 보면 실시간 채팅 기능의 필요성을 자연스럽게 느끼게 됩니다. 저 역시 이번 총대마켓 프로젝트에서 빠르게 어플리케이션 내 채팅 기능을 구현해야 했습니다. 이 글에서는 Kotlin을
사용해 안드로이드 어플리케이션에서 채팅 기능을 간단하게 구현하는 방법을 설명드리고자 합니다. View부터 Polling 방식을 통한 채팅 구현까지 단계별로 세세하게 다룰 예정입니다.

특히, Kotlin의 coroutines를 활용한 적절한 job 할당 및 해제를 통해 Polling을 효율적으로 구현하는 방법을 중점적으로 소개하겠습니다. 프로젝트의 요구사항과 상황에 따라 실시간성을 확보하는 데
가장 적합한 방법을 선택하는 것이 중요합니다. 이번 글을 통해 여러분이 안드로이드 어플리케이션에 실시간 기능을 빠르고 쉽게 도입할 수 있기를 바랍니다. Polling 방식은 복잡하지 않으면서도 일정 수준의 실시간성을
충족시켜주는 강력한 도구입니다. 실시간으로 업데이트가 필요한 기능이라면, 굳이 채팅이 아니더라도 Polling 방식을 활용하여 프로젝트에서 유용하게 적용할 수 있을 것입니다.

## 왜 Polling으로 채팅을 구현하지?

실시간 채팅을 구현하는 방법에는 여러 가지가 있습니다. 그 중 대표적인 방법 3가지만 간단하게 설명드리자면 WebSocket, Server-Sent Events(SSE), Polling 방식을 꼽을 수 있습니다.
각각의 방식은 구현의 난이도, 서버 부담, 그리고 실시간성 측면에서 차이가 있습니다. 각 기술을 활용한다면 예상되는 장점과 단점에 대해 비교하며 제가 polling 방법을 채택한 이유를 알려드리겠습니다.

### WebSocket

WebSocket은 클라이언트와 서버 간에 **양방향 통신**을 가능하게 하는 프로토콜입니다.
연결이 이루어진 후에는 클라이언트와 서버가 서로 자유롭게 데이터를 주고받을 수 있습니다. HTTP(Hyper Text Transfer Protocol)와 달리 **별도의 요청을 보내지 않고도 데이터를 수신**할 수
있습니다. 왜냐하면, WebSocket은 클라이언트와 서버가 연결된다면 응답 이후 연결을 끊는 것이 아닌 연결을 그대로 유지하고 있습니다. 그러므로 대기 시간 없이 서버가 클라이언트에게 데이터를 푸시할 수 있기
때문입니다.

WebSocket은 실시간 통신에 매우 적합하며 채팅을 구현한다면 가장 많이 사용되는 기술입니다. 하지만 WebSocket을 설정하고 관리하려면 서버 측에서 추가적인 인프라 설정이 필요하며, 프로젝트의 요구사항과
시간 제약을 고려했을 때는 다소 복잡한 솔루션이 될 수 있습니다.

<img width="500" alt="image" src ="https://velog.velcdn.com/images/wjdcogus6/post/84e70367-a1ab-496b-9e47-40e1a4107753/image.png">

⬆️ WebSocket 통신 방법

### Server-Sent Events (SSE)

Server-Sent Events(SSE)는 클라이언트가 서버로부터 이벤트를 수신할 수 있는 **단방향 통신** (서버 -> 클라이언트) 방법입니다.
서버는 클라이언트에 지속적으로 데이터를 전송하여 실시간 업데이트가 가능합니다. SSE는 HTTP 프로토콜 위에서 동작하며 설정이 비교적 간단한 편입니다. 그러나 양방향 통신을 지원하지 않기 때문에, 클라이언트가
서버에 데이터를 보낼 때는 별도의 요청을 해야 합니다. 대규모 트래픽 환경에서는 WebSocket보다 성능이 떨어질 수 있다는 단점과 단방향 통신이라는 단점도 존재합니다.
채팅은 양방향 통신이 필요하기 때문에 단방향 통신 흐름인 뉴스 또는 피드 업데이트에 SSE를 더 효과적으로 사용할 수 있습니다.

<img width="500" alt="image" src ="https://velog.velcdn.com/images/wjdcogus6/post/00555c39-5f55-4107-a6f6-241ec1333add/image.png">

⬆️ Server-Sent Events(SSE) 통신 방법

### Polling

Polling은 클라이언트가 일정한 주기(간격)으로 서버에 요청을 보내 최신 데이터를 받아오는 방식입니다. 이 방식은 구현이 매우 간단하며, 서버에 특별한 설정이 필요하지 않습니다. 클라이언트는 주기적으로 서버에
데이터를 요청하고, 서버는 요청이 있을 때마다 새로운 데이터를 응답하여 사용자에게 실시간으로 통신하는 것처럼 보이도록합니다. 실시간성 측면에서는 WebSocket과 SSE에 비해 뒤처지지만, 빠르게 기능을 구현해야
하는 상황에서는 적합한 솔루션이 될 수 있습니다.

<img width="500" alt="image" src ="https://velog.velcdn.com/images/wjdcogus6/post/de4d9d3f-c8c2-4b26-bd99-154348c758d6/image.png">

⬆️ Polling 통신 방법

### 세가지 방식 비교

| **항목**           | **WebSocket**              | **Server-Sent Event (SSE)** | **Polling**                   |
|------------------|----------------------------|-----------------------------|-------------------------------|
| **브라우저 지원**      | 대부분 브라우저에서 지원 (IE 10부터 지원) | 대부분 모던 브라우저 지원              | 모든 브라우저에서 지원                  |
| **통신 방향**        | 양방향                        | 일방향 (서버에서 클라이언트로)           | 일방향 (클라이언트에서 서버로 주기적인 요청)     |
| **데이터 형태**       | Binary, UTF-8              | UTF-8                       | JSON, XML 등 다양한 형태            |
| **자동 재접속**       | No                         | Yes (3초마다 재시도)              | 요청 주기 설정 가능 (재요청 필요 시 재접속 가능) |
| **프로토콜**         | WebSocket                  | HTTP                        | HTTP                          |
| **배터리 소모량**      | 큼                          | 작음                          | 상대적으로 큼                       |
| **Firewall 친화적** | Nope                       | Yes                         | Yes                           |
| **서버 부하**        | 낮음 (지속적인 연결)               | 낮음 (지속적인 연결)                | 높음 (주기적인 요청에 따른 부하 발생)        |
| **복잡성**          | 중간 (설정 필요)                 | 낮음 (간단한 HTTP 기반)            | 낮음 (주기적인 HTTP 요청)             |
| **리얼타임**         | Yes                        | Yes                         | 제한적 (주기적 요청에 따라 다름)           |
| **실시간성**         | 높음                         | 중간                          | 낮음 (주기적 요청 간격에 의존)            |

### Polling 방식을 선택한 이유

제가 Polling 방식을 선택한 이유는 서버 설정의 복잡성을 최소화하고, 빠른 시간 내에 채팅 기능을 구현하는 것이 목표였기 때문입니다. WebSocket이나 SSE를 사용할 경우 서버와의 실시간 양방향 통신이
가능하지만, 서버 설정이 복잡해지고 유지보수가 어려워질 수 있습니다. 반면, Polling은 상대적으로 구현이 간단하고, 서버 부담을 예측하기 쉽습니다. 프로젝트의 요구사항이 서버의 간단한 통신 구조와 빠른 구현을
필요로 했기 때문에, 안정적이고 직관적인 Polling 방식을 채택하게 되었습니다.

하지만 Polling에는 명확한 단점이 존재합니다.

1. 요청 주기가 짧으면 오버헤드/트래픽으로 인해 서버에 부담이 갑니다.
2. 요청 주기가 길면 실시간성이 떨어집니다.

이 단점을 해결하기 위한 캐싱 전략과 job 해제애 대해서도 고민도 소개하겠습니다.

---

## Polling 구현하기

이번 섹션에서는 MVVM 패턴과 DataBinding을 사용해 실제 채팅 애플리케이션에 Polling 방식을 적용하는 방법을 예시 코드와 함께 설명드리겠습니다.

### Polling 구현 방법

**주기적인 요청**
클라이언트는 일정 간격으로 서버에 요청을 보내 새 메시지가 있는지 확인합니다. 이를 통해 지속적으로 서버의 새로운 데이터를 가져올 수 있습니다. viewModelScope 내에서 launch를 사용해 주기적으로
데이터를 가져오는 코루틴을 생성합니다.

**서버 응답 처리**
서버에서 새로운 메시지를 응답하면, 이를 클라이언트 UI에 표시해야 합니다. 서버로부터 데이터를 성공적으로 받았다면, LiveData를 사용하여 UI를 업데이트할 수 있습니다.

### 코루틴 Job 관리 방법

**Job 설정하기**

``` kotlin
private var pollJob: Job? = null

private fun startPolling() {
    pollJob?.cancel()  // 기존 Job이 있으면 취소
    pollJob = viewModelScope.launch {
        while (this.isActive) {
            loadComments()  // 서버에서 채팅 데이터를 불러오는 함수
            delay(1000)
        }
    }
}
```

startPolling() 메서드는 ViewModel이 생성될 때 init 블록에서 호출되도록 설정하였습니다.

this.isActive는 코루틴의 활성 상태를 확인하는 플래그입니다. while 루프와 같은 반복 작업을 사용할 때, 이 플래그를 사용해 코루틴이 여전히 활성 상태인지 확인할 수 있습니다. 코루틴이 취소되더라도
즉시 작업이 중단되지 않기 때문에, isActive 플래그를 통해 취소 상태를 감지하고, 작업을 안전하게 중단할 수 있습니다.
더 자세한 내용은 "코틀린 코루틴의 정석" 4장(코루틴 빌더와 Job)을 참고하시면 도움이 됩니다.

**Job 해제하기**
주기적으로 서버에 요청을 보내는 Polling은 리소스를 많이 소비할 수 있는 작업이므로, 필요하지 않을 때는 꼭 Job을 해제해야 합니다. 예를 들어, 사용자가 화면을 떠나거나 Polling이 더 이상 필요하지
않을 때는 Job을 취소해 리소스를 낭비하지 않도록 해야 합니다. 이를 위해 ViewModel의 onCleared() 메서드를 오버라이드하여 ViewModel이 종료될 때 Polling도 함께 종료되도록 설정할 수
있습니다.

``` kotlin
override fun onCleared() {  // ViewModel 생명주기 종료 시 호출
    super.onCleared()
    stopPolling()  // Polling 종료
}

private fun stopPolling() {
    pollJob?.cancel()  // Job 취소
}
```

## 서버 과부하를 막기 위한 캐싱 전략

서버에 불필요한 과부하가 발생하지 않도록 하기 위해서는 캐싱 전략이 매우 중요합니다. 캐싱은 동일한 데이터를 여러 번 요청하지 않도록 함으로써 서버의 부담을 줄이는 방법 중 하나입니다. 특히, Polling과 같은
주기적인 서버 요청을 사용하는 경우, 새로운 데이터가 없는 상황에서도 반복적으로 데이터를 요청하는 것은 서버에 부담을 줄 수 있습니다. 이러한 문제를 해결하기 위해서는 캐싱 전략을 도입하고, 서버에서의 응답 처리
방식을 개선할 수 있습니다.

### 캐싱 전략

클라이언트 입장에서 제가 채택한 캐싱 전략은 마지막 댓글 ID를 로컬 Room 데이터베이스에 저장하고, 서버 응답에 따라 UI 업데이트를 결정하는 방식입니다. 새로운 댓글이 없을 경우 UI를 갱신할 필요가 없으므로,
서버와의 불필요한 통신을 줄여 서버 과부하를 줄일 수 있습니다.

**마지막 댓글 ID 저장**
Polling 요청을 통해 서버에서 댓글 데이터를 받아올 때, 각 댓글의 고유 ID가 존재합니다. 이 중 마지막으로 받아온 댓글의 ID를 Room DB에 저장해둡니다. 이후 서버에 요청을 보낼 때, 이 ID를
기준으로 http status code를 통해 새로운 댓글이 있는지 확인합니다.

**서버 응답 처리**
서버는 클라이언트가 마지막으로 받은 댓글 ID를 전달받고, 새로운 댓글이 있을 경우에만 데이터를 응답합니다. 만약 새로운 댓글이 없으면 서버는 204 No Content와 같은 상태 코드를 반환합니다. 이 상태
코드를 클라이언트가 확인하여, 새로운 댓글이 있을 때만 UI를 업데이트하고, 그렇지 않으면 업데이트를 하지 않습니다.

`CommentDetailRepositoryImpl` 구현 예시

``` kotlin
class CommentDetailRepositoryImpl @Inject constructor(
    private val commentRemoteDataSource: CommentRemoteDataSource,
    private val commentLocalDataSource: CommentLocalDataSource
) : CommentDetailRepository {

    // 서버에서 댓글을 가져오고 캐싱하는 로직
    override suspend fun fetchCommentsAndCache(offeringId: Long): Result<List<Comment>, DataError.Network> {
        // 1. 로컬 데이터베이스에서 마지막으로 저장된 댓글 ID를 가져옴
        val lastCommentId = commentLocalDataSource.getLatestCommentId(offeringId)

        // 2. 서버에서 댓글 목록을 요청
        return commentRemoteDataSource.fetchComments(offeringId).map { response ->
            if (response.statusCode == 204) {
                // 3. 서버에서 새로운 데이터가 없을 경우 (204 No Content)
                return Result.Success(emptyList())
            }

            val newComments = response.commentsResponse.map { it.toDomain() }

            if (newComments.isNotEmpty()) {
                // 4. 서버에서 받아온 댓글 중 마지막 댓글 ID를 가져옴
                val fetchedLastCommentId = newComments.last().id

                // 5. 로컬 데이터베이스의 마지막 댓글 ID와 비교하여, 새로운 댓글이 있는지 확인
                if (fetchedLastCommentId != lastCommentId) {
                    // 새로운 댓글이 있을 경우에만 로컬 데이터베이스에 저장 및 반환
                    commentLocalDataSource.insertComments(newComments.map { it.toEntity() })
                    Result.Success(newComments)
                } else {
                    // 새로운 댓글이 없는 경우 빈 리스트 반환
                    Result.Success(emptyList())
                }
            } else {
                Result.Success(emptyList())
            }
        }
    }
}

```

`CommentDetailViewModel` 구현 예시

``` kotlin
class CommentDetailViewModel @AssistedInject constructor(
    private val commentDetailRepository: CommentDetailRepository
) : ViewModel() {

    private val _comments: MutableLiveData<List<Comment>> = MutableLiveData()
    val comments: LiveData<List<Comment>> get() = _comments

    // 새로운 댓글을 확인하고, UI를 업데이트하는 함수
    fun loadComments(offeringId: Long) {
        viewModelScope.launch {
            when (val result = commentDetailRepository.fetchCommentsAndCache(offeringId)) {
                is Result.Success -> {
                    val newComments = result.data

                    // 1. Repository에서 새로운 댓글이 있을 때만 전달된 데이터를 UI 업데이트
                    if (newComments.isNotEmpty()) {
                        _comments.value = newComments
                    }
                }
                is Result.Error ->
                        when (result.error) {
                            DataError.Network.UNAUTHORIZED -> {기타 예시}

                            else -> {
                                pollJob?.cancel()
                                errorEvent.value = result.error.name
                            }
                        }
            }
        }
    }
}

```

## 채팅 뷰 구현

RecyclerView와 Sealed Class를 사용해서 채팅 UI 구현했습니다.
RecyclerView를 사용해 메시지를 리스트 형태로 표시합니다. 각 채팅 메시지를 다른 사용자와 본인의 메시지로 구분하기 위해 두 가지 뷰 타입을 사용하고 있으며, 이 뷰 타입은 Sealed Class로
정의했습니다.

### RecyclerView 설정 (xml)

``` xml
<androidx.recyclerview.widget.RecyclerView
    android:id="@+id/rv_comments"
    android:layout_width="match_parent"
    android:layout_height="0dp"
    android:layout_marginTop="@dimen/size_36"
    android:layout_marginBottom="@dimen/size_5"
    app:layout_constraintBottom_toTopOf="@+id/et_comment"
    app:layout_constraintEnd_toEndOf="parent"
    app:layout_constraintStart_toStartOf="parent"
    app:layout_constraintTop_toBottomOf="@id/tv_update_status"
    tools:itemCount="5"
    tools:listitem="@layout/item_other_comment" />

```

`tools:listitem`을 사용해서 미리보기로 RecyclerView를 확인해볼 수 있습니다. 그외 xml에서의 특별한 처리가 필요하지 않습니다.

### Activity 설정

**하지만!** 채팅의 경우 채팅 메시지가 적을 때 아래부터 스크롤을 시작한다는 특이점이 있습니다. 채팅 메시지를 아래에 고정하는 기능은 RecyclerView의 스크롤 관리를 통해 구현하였습니다. 그리고 채팅
메시지 목록이 업데이트될 때 자동으로 가장 최근 메시지로 스크롤되도록 처리하고 있습니다.

아래는 `Activity` 에서 해주었던 처리입니다.

``` kotlin
binding.rvComments.apply {
    adapter = commentAdapter
    layoutManager = LinearLayoutManager(this@CommentDetailActivity).apply {
        stackFromEnd = true
    }
}
```

LinearLayoutManager의 stackFromEnd = true 설정: 이 설정을 통해 RecyclerView는 채팅 메시지가 아래에서부터 쌓이도록 동작합니다. 즉, 최신 메시지가 리스트의 끝에 추가되며,
스크롤을 리스트 끝으로 자동으로 고정할 수 있습니다. 사용자가 채팅을 할 때, 새로운 메시지가 들어올 때마다 가장 아래에 있는 최신 메시지를 보여주기 위함입니다.

``` kotlin
viewModel.comments.observe(this) { comments ->
    commentAdapter.submitList(comments) {
        binding.rvComments.doOnPreDraw {
            binding.rvComments.scrollToPosition(comments.size - 1)
        }
    }
}
```

여기서 doOnPreDraw 메서드는 RecyclerView의 아이템이 렌더링되기 전에 수행되는 작업을 정의할 수 있는 메서드입니다.
채팅 목록이 갱신될 때, 최신 메시지로 스크롤하기 위해 `scrollToPosition(comments.size - 1)` 를 호출하여 가장 마지막 메시지로 스크롤을 이동시킵니다.

### 채팅 item layout

그리고 또 힘든 과정 하나가 더 남아 있습니다..
일정한 공백을 두고 채팅 내용이 너무 길다면 자동으로 정렬을 맞춰주어야겠죠?

``` xml
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools">

    <data>

        <variable
            name="comment"
            type="~~~.domain.model.Comment" />
    </data>

    <androidx.constraintlayout.widget.ConstraintLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:padding="@dimen/size_6">

        <androidx.constraintlayout.widget.Guideline
            android:id="@+id/guideline"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:orientation="vertical"
            app:layout_constraintGuide_begin="@dimen/size_150" />

        <TextView
            android:id="@+id/tv_comment"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_marginEnd="@dimen/margin_20"
            android:background="@drawable/bg_comment_main_color"
            android:ellipsize="end"
            android:fontFamily="@font/suit_medium"
            android:gravity="left"
            android:maxLines="10"
            android:paddingStart="@dimen/size_10"
            android:paddingTop="@dimen/size_7"
            android:paddingEnd="@dimen/size_10"
            android:paddingBottom="@dimen/size_7"
            android:text="@{comment.content}"
            android:textColor="@color/white"
            android:textSize="@dimen/size_14"
            app:layout_constrainedWidth="true"
            app:layout_constraintBottom_toBottomOf="parent"
            app:layout_constraintEnd_toEndOf="parent"
            app:layout_constraintHorizontal_bias="1.0"
            app:layout_constraintStart_toEndOf="@id/guideline"
            app:layout_constraintTop_toTopOf="parent"
            tools:text="내용\n내용이야 채팅 내용이야 내용이야 채팅 내용이야 내용이야 채팅 내용이" />

        <TextView
            android:id="@+id/tv_time"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_marginEnd="@dimen/margin_10"
            android:fontFamily="@font/suit_medium"
            android:textSize="@dimen/size_10"
            app:formattedCommentTime="@{comment.commentCreatedAt.time}"
            app:layout_constraintBottom_toBottomOf="@id/tv_comment"
            app:layout_constraintEnd_toStartOf="@id/tv_comment"
            tools:text="오전 10:30" />

    </androidx.constraintlayout.widget.ConstraintLayout>
</layout>


```

Guideline을 사용하여 본인의 메시지를 오른쪽에 배치하기 위해 설정되었습니다. app:layout_constraintGuide_begin 속성으로 왼쪽에서 150dp 떨어진 위치에 Guideline을
배치했습니다.
app:layout_constrainedWidth="true"로 설정을 꼭 해주어야 의도한 뷰로 동작합니다.
그리고 다양한 옵션을 통해 구현을 하였습니다.

아래는 이런식으로 구현한 채팅 뷰 item 입니다.

<img width="300" alt="image" src ="https://velog.velcdn.com/images/wjdcogus6/post/2f1e5041-e85a-4f06-a760-f6c7386841e7/image.png">

<img width="300" alt="image" src ="https://velog.velcdn.com/images/wjdcogus6/post/54fb87ca-d14e-4f54-a216-9d95eb3e908b/image.png">

### ViewType에 따른 레이아웃 정하기

Sealed Class는 컴파일 시점에서 모든 하위 클래스를 알고 있기 때문에, 타입 안정성을 보장할 수 있고 when을 사용하여 Sealed Class의 모든 경우에 대한 처리가 이루어지며, 새로운
ViewType이 추가되면 when 표현식에서 누락된 타입에 대해 컴파일 오류가 발생해 바로 수정할 수 있으므로 sealed class를 사용하여 ViewType을 지정해줍니다.
저는 추가 기능 요구 사항으로 채팅의 날짜에 대한 ViewType을 추가해야했기 때문에, 현재는 UiModel을 만들어서 관리해주고 있습니다!

``` kotlin
sealed class CommentViewType {
    data class MyMessage(val comment: Comment) : CommentViewType()
    data class OtherMessage(val comment: Comment) : CommentViewType()
}
```

CommentAdapter 구현

``` kotlin
class CommentAdapter : ListAdapter<CommentViewType, RecyclerView.ViewHolder>(DIFF_CALLBACK) {
    override fun getItemViewType(position: Int): Int {
        return when (getItem(position)) {
            is CommentViewType.MyComment -> VIEW_TYPE_MY_COMMENT
            is CommentViewType.OtherComment -> VIEW_TYPE_OTHER_COMMENT
        }
    }

    override fun onCreateViewHolder(
        parent: ViewGroup,
        viewType: Int,
    ): RecyclerView.ViewHolder {
        return when (viewType) {
            VIEW_TYPE_MY_COMMENT -> {
                val binding =
                    ItemMyCommentBinding.inflate(LayoutInflater.from(parent.context), parent, false)
                MyCommentViewHolder(binding)
            }

            VIEW_TYPE_OTHER_COMMENT -> {
                val binding =
                    ItemOtherCommentBinding.inflate(
                        LayoutInflater.from(parent.context),
                        parent,
                        false,
                    )
                OtherCommentViewHolder(binding)
            }

            else -> throw IllegalArgumentException("CommentAdapter: viewType error")
        }
    }
```

## 실제 적용

polling을 적용하여 실제로 구현한 화면입니다.

<img width="300" alt="gif" src ="https://github.com/user-attachments/assets/c4530d77-00d2-4da0-bca3-b38a5cc14954">

이번 프로젝트에서 Polling 방식을 이용해 채팅 기능을 구현하면서, 실시간성을 유지하면서도 서버 성능을 최적화하는 균형의 중요성을 배울 수 있었습니다. Polling 방식이 단순해 보이지만, 서버 과부하를 줄이기
위한 캐싱 전략과 job 해제에 대한 깊이 있는 고민을 할 수 있는 기회가 되었습니다.

## 참고

* 참고 서적 : 코틀린 코루틴의 정석
* 참고
  블로그: [웹소켓 과 SSE(Server-Sent-Event) 차이점 알아보고 사용해보기](https://surviveasdev.tistory.com/entry/%EC%9B%B9%EC%86%8C%EC%BC%93-%EA%B3%BC-SSEServer-Sent-Event-%EC%B0%A8%EC%9D%B4%EC%A0%90-%EC%95%8C%EC%95%84%EB%B3%B4%EA%B3%A0-%EC%82%AC%EC%9A%A9%ED%95%B4%EB%B3%B4%EA%B8%B0)

