# Paging3를 사용해 무한스크롤 구현하기

# 데이터 로딩 전략

---
안드로이드에서 데이터 로딩 전략은 사용자 경험을 향상시키고, 성능을 최적화하며, 배터리 및 네트워크 자원을 효율적으로 사용하기 위한 중요한 요소이다.   
대규모 리스트 데이터를 가져올 때 주로 고려되는 UX패턴은 다음과 같다.

## Pagination
<img src="https://velog.velcdn.com/images/yunsuk0328/post/0cacec49-1261-4662-98dd-bdc8d1a73f5b/image.png" width="200" height="300">

사용자가 '다음', '이전', 페이지 번호와 같은 링크를 사용하여 페이지 간에 이동할 수 있다.
이때 검색결과가 한 번에 한 페이지씩 표시된다.

**장점**
- 사용자에게 검색결과의 크기 및 현재 위치와 관련된 유용한 정보를 제공한다.


**단점**
- 사용자가 검색결과 간에 이동하는 데 사용하는 제어 방식이 다소 복잡하다.
- 콘텐츠가 하나의 연속 목록 형태가 아니라 여러 페이지에 걸쳐 나뉜다.
- 항목을 더 보려면 새로 페이지를 로드해야 한다.

## Load More
<img src="https://velog.velcdn.com/images/yunsuk0328/post/6df5f46e-00b2-4e29-b3ac-ecdc15745165/image.png" width="200" height="300">

사용자가 더보기 버튼을 클릭하여 검색결과 집합을 펼칠 수 있다.


**장점**
- 단일 페이지에 모든 콘텐츠 포함한다.
- 사용자에게 검색결과의 총 크기를 알릴 수 있다.(버튼 위나 근처에 있음)


**단점**
- 모든 검색결과가 단일 웹페이지에 포함되므로 검색결과가 아주 많은 경우 처리할 수 없다.

## 무한스크롤
<img src="https://velog.velcdn.com/images/yunsuk0328/post/ce32ba35-532e-4a88-97bd-546a7c9a4345/image.png" width="200" height="300">


사용자가 페이지의 끝까지 스크롤하면 더 많은 콘텐츠가 로드된다.

**장점**
- 단일 페이지에 모든 콘텐츠 포함한다.
- 직관적: 사용자가 콘텐츠를 더 보려면 계속 스크롤하면 된다.


**단점**
- 검색결과의 크기가 명확하지 않아 '스크롤하는 데 피로감'이 발생할 수 있다.
  검색결과가 아주 많은 경우 처리할 수 없다.



# 무한스크롤 구현

---

## 직접 구현

리스트의 하단에 도달하면 자동으로 추가 데이터를 불러와 표시하는 스크롤 리스너를 추가한다.
(이 글에서는 직접 구현을 다루지 않을 것이므로 간략하게만 적어보았다..)


## 라이브러리를 사용한 구현

구글에서 제공하는 페이징 라이브러리인 `Paging3` 라이브러리를 사용하는 방법이 있다. 해당 라이브러리는 대규모 리스트 데이터를 효율적으로 로드하고 표시할 수 있도록 도와준다.


# Paging3

---

Android Jectpack의 구성요소로써 공식문서에서는 아래와 같이 설명하고 있다.

> Paging 라이브러리를 사용하면 로컬 저장소에서나 네트워크를 통해 대규모 데이터 세트의 데이터 페이지를 로드하고 표시할 수 있습니다. 이 방식을 사용하면 앱에서 네트워크 대역폭과 시스템 리소스를 모두 더 효율적으로 사용할 수 있습니다. Paging 라이브러리의 구성요소는 권장 Android 앱 아키텍처에 맞게 설계되었으며 다른 Jetpack 구성요소와 원활하게 통합되고 최고 수준으로 Kotlin을 지원합니다.

Paging라이브러리를 활용했을 때의 **장점**은 다음과 같다.

- Paging된 데이터의 메모리 내 캐싱.
  - 앱이 Paging 데이터로 작업하는 동안 시스템 리소스를 효율적으로 사용할 수 있다.
- 요청 중복 삭제 기능이 기본 제공되므로 앱에서 네트워크 대역폭과 시스템 리소스를 효율적으로 사용할 수 있다.
- 사용자가 로드된 데이터의 끝까지 스크롤할 때 구성 가능한 `RecyclerView` 어댑터가 자동으로 데이터를 요청한다.
- Kotlin 코루틴 및 플로우 뿐만 아니라 `LiveData` 및 RxJava를 최고 수준으로 지원한다.
- 새로고침 및 재시도 기능을 포함하여 오류 처리를 기본으로 지원한다.


***Paging 라이브러리의 흐름은 아래와 같다.***  

<img src="https://velog.velcdn.com/images/yunsuk0328/post/d1573498-179b-4ff3-bd37-b6fca0d2f41d/image.png" width="600" height="200">



### Repository 레이어
`PagingSource` 객체는 데이터 소스와 이 소스에서 데이터를 검색하는 방법을 정의한다.
`PagingSource` 객체는 네트워크 소스 및 로컬 데이터베이스를 포함한 단일 소스에서 데이터를 로드할 수 있다.
사용할 수 있는 다른 페이징 라이브러리 구성요소는 `RemoteMediator`이다. `RemoteMediator` 객체는 로컬 데이터베이스 캐시가 있는 네트워크 데이터 소스와 같은 계층화된 데이터 소스의 페이징을 처리한다.


### ViewModel 레이어
`Pager` 구성요소는 PagingSource 객체 및 PagingConfig 구성 객체를 바탕으로 반응형 스트림에 노출되는 PagingData 인스턴스를 구성하기 위한 공개 API를 제공한다.
ViewModel 레이어를 UI에 연결하는 구성요소는 `PagingData`이다. `PagingData` 객체는 페이지로 나눈 데이터의 스냅샷을 보유하는 컨테이너이다. `PagingSource` 객체를 쿼리하여 결과를 저장한다.


### UI 레이어
UI 레이어의 기본 페이징 라이브러리 구성요소는 페이지로 나눈 데이터를 처리하는 RecyclerView 어댑터인 `PagingDataAdapter`이다.

<br>

***이제 코드를 통해 더 알아보도록 하자!***


## PagingSource 정의
```kotlin
class OfferingPagingSource(
    private val offeringsRepository: OfferingRepository,
) : PagingSource<Long, Offering>() {
    override suspend fun load(params: LoadParams<Long>): LoadResult<Long, Offering> {
        val lastOfferingId = params.key
        return runCatching {
            val offerings =
                offeringsRepository.fetchOfferings(
                    lastOfferingId = lastOfferingId,
                    pageSize = params.loadSize,
                ).getOrThrow()

            val prevKey = if (lastOfferingId == null) null else lastOfferingId + DEFAULT_PAGE_SIZE
            val nextKey =
                if (offerings.isEmpty() || offerings.size < DEFAULT_PAGE_SIZE) null else offerings.last().id

            LoadResult.Page(
                data = offerings,
                prevKey = prevKey,
                nextKey = nextKey,
            )
        }.onFailure { throwable ->
            LoadResult.Error<Long, Offering>(throwable)
        }.getOrThrow()
    }

    override fun getRefreshKey(state: PagingState<Long, Offering>): Long? {
        return state.anchorPosition?.let { anchorPosition ->
            state.closestPageToPosition(anchorPosition)?.prevKey?.minus(DEFAULT_PAGE_SIZE)
        }
    }

    companion object {
        private const val DEFAULT_PAGE_SIZE = 10
    }
}
```
> 데이터를 불러오는 방식은 No Offest방식으로 마지막 id를 기준으로 pageSize만큼 데이터를 가져오고 있다.

`PagingSource<Key, Value>`는 두 가지 유형 매개변수를 사용합니다. `Key`는 데이터를 로드하는 데 필요한 식별자, `Value`는 실제 데이터 유형을 나타냅니다. 현재 코드에서는 `Long` 타입의 ID를 식별자로, `Offering` 타입의 데이터를 사용하므로 `PagingSource<Long, Offering>()`로 정의되어 있다.
### load()
`load()` 메서드는 `PagingSource`의 핵심 메서드로, 데이터를 비동기적으로 로드하여 `RecyclerView`에 제공합니다. 성공 시 `LoadResult.Page` 객체를 반환하고, 실패 시 `LoadResult.Error`를 반환합니다.

- `params`: 페이징 요청 정보를 담고 있음.
    - `key`: 마지막 아이템 ID 또는 특정 키값
    - `loadSize`: 한 번에 로드할 데이터 수

<br> 

- `prevKey`: 이전 페이지를 요청할 때 사용할 키
    - 이전에 어떤 페이지에서 데이터를 불러왔고, 그 페이지로 다시 돌아가고 싶을 때 prevKey를 사용한다.
    - **첫 번째 페이지인 경우 null을 할당해주어야 한다.**

  <br> 

- `nextKey`: 다음 페이지를 요청할 때 사용할 키
    - 현재 페이지가 로드되었을 때, 이 페이지의 마지막 데이터를 기준으로 다음 페이지를 불러오기 위한 키이다.
    - **마지막 페이지인 경우 null을 할당해주어야 한다.**


### getRefreshKey()

`getRefreshKey()` 메서드는 `PagingSource에서` 새로고침(refresh) 시 어떤 키를 기준으로 데이터를 다시 로드할지를 결정하는 메서드이다.
사용자가 스크롤을 멈춘 위치에서 데이터 새로고침을 할 때, 해당 위치와 가까운 페이지의 키를 기준으로 새로 데이터를 가져온다.

코드를 설명해보자면 다음과 같다.

- `state.anchorPosition`
    - `anchorPosition`은 사용자가 현재 보고 있는 스크롤 위치의 인덱스이다.
    - null이 아니라면 스크롤 위치가 유효하다는 의미다.

<br>

- `closestPageToPosition(anchorPosition)`
    - 현재 스크롤 위치와 가장 가까운 페이지 정보를 반환한다.

<br>

- `prevKey`를 기준으로 새로고침
    - 찾은 페이지의 `prevKey`(이전 페이지 키값)를 기준으로 새로고침할 키값을 계산한다.
    - `prevKey?.minus(DEFAULT_PAGE_SIZE)`로 새로고침 기준이 되는 키값을 설정한다.
    - `prevKey`에서 페이지 크기만큼 빼서 이전 데이터의 첫 번째 항목부터 다시 로드하게끔 만든다.


## PagingData 스트림 설정

```kotlin
//viewModel
private val _offerings = MutableLiveData<PagingData<Offering>>()
val offerings: LiveData<PagingData<Offering>> get() = _offerings

fun fetchOfferings() {
	viewModelScope.launch {
		Pager(
			config = PagingConfig(pageSize = PAGE_SIZE),
			pagingSourceFactory = {
				OfferingPagingSource(
					offeringRepository,
				)
			},
		).flow.cachedIn(viewModelScope).collectLatest { pagingData ->
			_offerings.value = pagingData
		}
	}
}
```
- `config`: PagingConfig를 통해 페이징 처리에 대한 설정을 정의
    - `pageSize`: 한 페이지에서 로드할 데이터의 개수

- `cachedIn()`: 페이징 데이터의 상태를 제공된 `CoroutineScope`을 사용해 로드된 데이터를 캐싱하여, 이미 로드된 데이터를 재사용할 수 있도록 한다.

## RecyclerView 어댑터 정의

```kotlin
class OfferingAdapter : PagingDataAdapter<Offering, OfferingViewHolder>(productComparator) {

    override fun onCreateViewHolder(
        parent: ViewGroup,
        viewType: Int,
    ): OfferingViewHolder {
        val binding =
            ItemOfferingBinding.inflate(LayoutInflater.from(parent.context), parent, false)
        return OfferingViewHolder(binding)
    }

    override fun onBindViewHolder(
        holder: OfferingViewHolder,
        position: Int,
    ) {
        getItem(position)?.let { offering ->
            holder.bind(offering)
        }
    }

    companion object {
        private val productComparator =
            object : DiffUtil.ItemCallback<Offering>() {
                override fun areItemsTheSame(
                    oldItem: Offering,
                    newItem: Offering,
                ): Boolean {
                    return oldItem.id == newItem.id
                }

                override fun areContentsTheSame(
                    oldItem: Offering,
                    newItem: Offering,
                ): Boolean {
                    return oldItem == newItem
                }
            }
    }
}
```

Paging 라이브러리에서 제공하는 `PagingDataAdapter`를 확장하여 Adapter를 만들어 줍니다.

또한 Adpater는 `DiffUtil.ItemCallback`을 지정해주어야 한다.


## 검색과 필터기능 추가하기
<img src="https://velog.velcdn.com/images/yunsuk0328/post/d2c3ecc4-4552-49c2-a937-296f5814818e/image.png" width="350" height="400">

홈화면에서 위와 같이 게시글 검색과 필터링 기능을 추가해 주기위해 API가 변경되었다.

```kotlin
// Before

    @GET("/offerings")
    suspend fun getOfferings2(
        @Query("last-id") lastOfferingId: Long?,
        @Query("page-size") pageSize: Int?,
    ): Response<OfferingsResponse>
    
// After (검색과 필터가 추가됨)

    @GET("/offerings")
    suspend fun getOfferings(
        @Query("filter") filter: String?,
        @Query("search") search: String?,
        @Query("last-id") lastOfferingId: Long?,
        @Query("page-size") pageSize: Int?,
    ): Response<OfferingsResponse>
```

따라서 `PagingSource`와 `PagingData` 스트림도 수정해주었다.

```kotlin
// PagingSource

class OfferingPagingSource(
    private val offeringsRepository: OfferingRepository,
    private val search: String?,
    private val filter: String?,
) : PagingSource<Long, Offering>() {
    override suspend fun load(params: LoadParams<Long>): LoadResult<Long, Offering> {
        val lastOfferingId = params.key
        return runCatching {
            val offerings =
                offeringsRepository.fetchOfferings(
                    filter = filter,
                    search = search,
                    lastOfferingId = lastOfferingId,
                    pageSize = params.loadSize,
                ).getOrThrow()
                
	// 중략
}

//ViewModel

private fun fetchOfferings() {
    viewModelScope.launch {
        Pager(
            config = PagingConfig(
                pageSize = PAGE_SIZE,
            ),
            pagingSourceFactory = {
                OfferingPagingSource(
                    offeringRepository,
                    search.value,
                    _selectedFilter.value,
                )
            },
        ).flow.cachedIn(viewModelScope).collectLatest { pagingData ->
            _offerings.value = pagingData
        }
    }
}

```

단순하게 위와 같이 코드를 추가해주고 실행했을 때 다음과 같은 **문제**가 발생했다.

- 라이브러리 자체적으로 item을 캐싱하기에 검색이나 필터링을 통해 아이템을 불러올 때 이미 존재하는 item은 그대로 있고 새로운 아이템이 기존 아이템들 사이로 끼워지는 방식으로 보여 UX적으로 좋지 않다.

- 검색이나 필터링으로 새롭게 아이템을 불러올 때 이미 캐싱 된 item들과 새롭게 추가된 item들이 섞여서 스크롤 시 `IndexOutOfBoundsException` 오류가 발생한다.


### 해결 방안

검색이나 필터링을 통해 새롭게 item을 불러올 때, 빈 값을 넣어주고 이후에 API를 호출하는 방식으로 해결해 주었다. 해당 방법으로 Paging을 초기화시켜줌으로써`IndexOutOfBoundsException`을 방지하였다.

```kotlin
viewModel.filterOfferingsEvent.observe(viewLifecycleOwner) {
    offeringAdapter.submitData(viewLifecycleOwner.lifecycle, PagingData.empty())
}
```

그리고 목록이 불러와 지기 전에 빈 화면과 동시에 `ProgressBar`를 사용자에게 보여주도록 하여서 UX적으로 개선할 수 있었다.

```kotlin
viewLifecycleOwner.lifecycleScope.launch {
	viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
		offeringAdapter.loadStateFlow.collect { loadState ->
			binding.pbLoading.isVisible = loadState.refresh is LoadState.Loading
		}
	}
}
```

## Paging을 사용할 때 상태변경

**상태변경 처리 전**  

<img src="https://velog.velcdn.com/images/yunsuk0328/post/b1fb1494-72bf-4d8b-96e0-5a1e997616f0/image.gif" width="250">

남은 인원이 2명인 공동구매에 참여하고 홈 화면으로 돌아왔을 때 그대로 2명이 남았다고 뜬다.


**상태변경 처리 후**  

<img src="https://velog.velcdn.com/images/yunsuk0328/post/7d65eb73-cd3d-4eab-bf61-99f54d3988a0/image.gif" width="250">
남은 인원이 2명인 공동구매에 참여하고 홈 화면으로 돌아왔을 때, 1명이 남았다고 상태가 바뀐 것을 볼 수 있다.


위와 같이 공동구매에 참여한 후, 홈으로 돌아왔을 때 사용자가 직접 새로고침하지 않아도 상태가 자동으로 반영되게 하고 싶었다.

가장 간단한 방법은 참여 시 전체 게시물 목록을 다시 불러와 업데이트하는 것이지만, 이 방식은 매번 전체 게시글을 로드해야 해 오버헤드가 커지고, 게시물의 최상단으로 이동하게 되어 UX 측면에서도 좋지 않다.

따라서, 상태가 변화한 게시물의 상태만 업데이트해주어야 하고 아래와 같은 흐름으로 구현을 해주었다.

#### 구현 흐름
게시물 상세 화면: DetailFragment
홈 화면: HomeFragment

1. 참여 시 DetailFragment에서 해당 게시물의 id를 HomeFragment로 넘겨준다.
2. id를 ViewModel로 넘기고 ViewModel에서 업데이트된 게시물들의 정보를 liveData로 저장한다.
3. HomeFragment에서 해당 정보를 observe하고 있고, 변화가 있을 시 업데이트된 게시물들의 정보를 PagingDataAdapter로 보낸다.
4. PagingDataAdapter snapshot으로 현재 로드된 데이터들을 가져와 업데이트된 게시물 정보와 비교하여 상태가 바뀐것이 있다면 해당 게시물의 position을 찾아 notifyItemChanged(position)을 통해 update 해준다.

한 단계씩 코드를 통해 자세히 살펴보도록 하자.

### 1단계

```kotlin
// DetailViewModel
private val _updatedPostId: MutableLiveData<Long> = MutableLiveData()
val updatedPostId: LiveData<Long> get() = _updatedPostId

						(중략)

_updatedPostId.value = postId
````
DetailViewModel에서 위와 같은 프로퍼티를 가지고 있고 공동구매 참여시 해당 게시물의 id값을 할당해준다.

```kotlin
// DetailFragment
viewModel.updatedPostId.observe(viewLifecycleOwner) {
	setFragmentResult(DETAIL_BUNDLE_KEY, bundleOf(UPDATED_POST_ID_KEY to it))
}     
```        
DetailFragment에서 위 liveData를 ovserve하고 있고 `setFragmentResult`를 통해 id를 전달한다.

### 2단계

```kotlin
// HomeFragment
setFragmentResultListener(DetailFragment.DETAIL_BUNDLE_KEY) { _, bundle ->
	viewModel.fetchUpdatedPost(bundle.getLong(DetailFragment.UPDATED_POST_ID_KEY))
}

// HomeViewModel
private val _updatedPost: MutableSingleLiveData<MutableList<Post>> = MutableSingleLiveData(mutableListOf())
val updatedPost: SingleLiveData<MutableList<Post>> get() = _updatedPost

. . .

fun fetchUpdatedPost(postId: Long) {
    viewModelScope.launch {
        when (val result = postRepository.fetchPost(postId)) {

					(중략)

            is Result.Success -> {
                val updatedPosts = _updatedPost.getValue() ?: mutableListOf()
                updatedPosts.add(result.data)
                _updatedPost.setValue(updatedPosts)
            }
        }
    }
}
```
위와 같은 로직으로 id를 ViewModel로 넘기고 ViewModel에서 업데이트된 게시물들의 정보를 liveData로 저장한다.

바로 Adpater로 넘기지 않은 이유는 ViewModel에 캐싱해둠으로써 configuration change에 대응하기 위함이다.

### 3, 4단계
```kotlin
// HomeFragment
viewModel.updatedPost.observe(viewLifecycleOwner) {
	postAdapter.addUpdatedItem(it.toList())
}

// PagingDataAdapter
class PostAdapter() : PagingDataAdapter<Post, PostiewHolder>(postComparator) {

    private var updatedPosts: List<Post> = emptyList()
    
						(중략)

    fun addUpdatedItem(updatedPosts: List<Post>) {
        this.updatedPosts = updatedPosts
        updatedPosts.forEach { post ->
            val position = findPositionByPostID(post)
            if (position != -1) {
                notifyItemChanged(position)
            }
        }
    }


    private fun findPositionByPostID(post: Post) =
        snapshot().items.indexOfFirst { it.id == post.id }
}

```

앞서 설명한 바와 같이 상태가 바뀐 게시물을 찾아서 `notifyItemChanged`해줌으로써 상태를 변경시켜준다.




# 무조건 라이브러리를 사용하는 것이 좋을까?

라이브러리를 사용할 때의 **장점**은 역시 구현할 때의 리소스가 줄어든다는 것이다.
물론 성능적인 면도 좋아지겠지만 나는 체감할 수는 없었다.

**단점**은 라이브러리의 특성상 추상화가 되어있다 보니 내부 동작이 정확히 어떻게 되어있는지 몰라서 우리 서비스에 맞게 커스텀하기가 어렵다는 것이다.

새로운 라이브러리를 적용해보는 경험을 통해 라이브러리를 사용하는 것이 만능은 아니라는 것을 느낄 수 있었다.

## 참고자료

[Pagination, incremental page loading, and their impact on Google Search](https://developers.google.com/search/docs/specialty/ecommerce/pagination-and-incremental-page-loading)

[페이징 라이브러리 개요](https://developer.android.com/topic/libraries/architecture/paging/v3-overview?hl=ko)