# API 성능 개선 과정 — 덕지덕지 붙은 외부 API에서 자체 알고리즘으로

> 이동 시간 기반 모임 추천 서비스 [모잇지(Moitz)](https://moitz.kr)에서 API 응답 시간을 30초 이상에서, 약 5초 이하로 줄이기까지의 개선 여정


## 서비스 배경과 문제 인식

모잇지는 **이동 시간이 공평한 모임 장소를 정하자**는 아이디어에서 출발했습니다.
사용자들이 각자 출발지를 입력하면, 각 출발지로부터 이동 시간이 비슷하고 모임 카테고리(예: 카페, 식당 등)의 장소가 존재하는 지역을 추천해주는 서비스입니다.

내부적으로 위치(Location)는 만날 동네(성수역, 행리단길 등), 장소(Place)는 만날 곳(특정 가게)로 구분합니다. 추천은 이 두 개념을 중심으로 이루어집니다.

### 초기 MVP(Minimum Viable Product) 구조

개발 초기, 적은 노력으로 사용자의 선호에 대한 피드백을 빠르게 받기 위해 MVP 개발을 결정했습니다. MVP는 Minimum Viable Product의 약자로 가장 핵심적인 가치를 검증할 수 있을 만큼만 기능을 구현한 제품을 의미합니다. 저희는 LLM을 적극 기용해서 많은 로직을 대체하고 응답을 얻기 위한 MVP 개발 계획을 세웠습니다. 

- LLM API(Gemini Client): 지역 후보지 선정, 검색 장소 데이터 기반 지역 추천
- Odsay API: 출발지로부터 추천 지역까지의 이동 경로 및 시간 계산
- Kakao Place API: 추천 지역 내 장소 검색

이 세 가지를 조합하여 로직을 설계하였습니다. 먼저 사용자로부터 입력받은 각 출발지 이름을 통해 역 데이터를 검색합니다. 검색한 출발지 데이터를 사전에 정의한 프롬프트와 함께 Gemini Client에 장소 후보지 선정을 요청합니다. 이후 Odsay API를 통해 각 출발지로부터 각 후보지까지의 이동 경로 및 시간 계산 정보를, Kakao Place API를 통해 후보지의 장소를 검색합니다. 최종적으로 Gemini Client에 해당 정보를 함수 호출로 재전달하여 적절한 장소를 추천 받습니다.

헤딩 과정을 통해 꽤나 그럴싸한 데이터를 얻을 수 있었습니다. 여기서 문제는 이 과정이 너무 느리다는 것이었습니다.


## 문제 정의 — 느림의 정체

테스트에서 API 응답 시간은 **평균 30~50초** 정도였습니다. 단일 요청의 경우에도 이 속도였으니 동시 요청 수가 늘어나면 체감 지연은 기하급수적으로 증가했습니다.

LLM API 등 여러 외부 API를 동시에 사용하고 있었기 때문에 응답이 오래 걸린다는 것은 알고 있었지만, 어느 항목에서 응답 속도가 느린지 정확히 파악하기 위해 각 로직 단계별 소요 시간을 정량적으로 측정하기로 했습니다.

Spring Framework에서 제공하는 StopWatch 클래스를 활용해 요청 처리 과정의 각 구간을 세분화하여 실행 시간을 기록했습니다. 이 방식은 외부 모니터링 도구를 붙이지 않아도 서버 내부에서 빠르게 병목 구간을 식별할 수 있어 러닝 커브가 낮기 때문에 현재 상황에 적합하다고 판단하였습니다.

![초기 응답 시간 측정 결과](/imgs/image1.png)

이동 경로 및 시간 조회 및 장소 추천에서 소요 시간이 긴 것을 확인할 수 있었고, 이에 따라 저희가 정의한 문제는 다음과 같습니다.

1. 단계 간 종속성이 높음
각 단계가 이전 단계의 결과를 필요로 하기 때문에 현재 구조로는 병렬화나 캐싱 적용이 쉽지 않았습니다. 예를 들어 후보지를 LLM이 제시해야만 ODsay API를 호출할 수 있는, 프로세스의 논리적 순서 자체가 병목을 포함한 구조였습니다.

2. 외부 API 비율이 높음
초기 로직 대부분이 내부 연산이 아닌 외부 API 호출이었습니다. 외부 API는 네트워크 지연이나 제한(초당 요청 수 등)에 영향을 받기 때문에 단순한 코드 최적화로는 개선이 불가능했습니다.

다만 해당 문제들은 MVP 단계의 구조적 한계로 인해 발생한 문제였습니다. 기능 검증을 우선으로 조합한 구조였기 때문에 최적화된 상태가 아니었습니다. 내부적으로 계산 가능한 부분은 외부 API를 걷어내는 것이 가장 좋겠지만, 마감 일정에 따른 현재 진행 상황을 고려했을 때 남은 일정 안에 계산 로직을 구현할 수 있을지 장담할 수 없었습니다. 또한 남은 기능 마저 개발하며 이후 개선하는 방향을 생각할 수 있겠지만 서비스를 지속적으로 운영하며 개선하는 팀 내의 목표를 생각했을 때, 사용자가 MVP를 사용하며 체감할 속도 저하를 두고만 볼 순 없었습니다.

따라서 팀은 **기존 로직을 유지한 채 성능을 개선할 수 있는 방법**을 탐색하기로 했습니다.


## 첫 번째 접근 — WebClient로 비동기 전환

처음 손댄 부분은 수십 번 이상 요청하는 외부 API 호출입니다. 기존에는 API 호출 클라이언트로 미션에서 사용해본 적이 있어 러닝커브가 낮은 RestClient를 사용하였습니다.

```java
@Override
public List<Route> findRoutes(final List<StartEndPair> placePairs) {
    return placePairs.stream()
            .map(pair -> {
                final Place startPlace = pair.start();
                final Place endPlace = pair.end();
                return new Route(
                        convertPaths(odsayClient.getRoute(startPlace.getPoint(), endPlace.getPoint()))
                );
                })
            .toList();
}
```

이 때 각 API를 동기적으로 호출하여 하나의 요청이 완료될 때까지 스레드가 점유된 상태로 대기하고 있었고, 외부 API 호출이 10회 이상 필요한 상황에서는 10개의 요청이 직렬로 처리되어 응답 시간이 단순히 10배 이상 증가했습니다. 동일한 API를 각기 다른 값으로 요청하는 구조였지만, 개별 작업 간 서로 영향을 미치지 않았기 때문에 구조적인 개선이 필요했습니다. 

### 변경 방향

이 문제를 해결하기 위해 서비스 레이어에서 WebClient와 Flux를 활용해 외부 API 호출을 비동기 병렬로 처리하도록 구조를 변경했습니다.

```java
@Override
public Flux<Route> findRoutes(final List<StartEndPair> placePairs) {
    return Flux.fromIterable(placePairs)
            .flatMap(pair ->
                    odsaiMultiClient.getRoute(pair.start().getPoint(), pair.end().getPoint())
                            .map(this::convertPaths)
                            .map(Route::new)
            );
}
```

### 선택 이유

WebClient는 리액티브 스트림(Reactor) 기반으로 설계되어 있기 때문에 `Flux.merge()`나 `flatMap()`을 사용해 자연스럽게 병렬 실행을 구현할 수 있습니다. 10개의 API를 병렬로 호출하더라도 스레드풀 크기가 크게 증가하지 않으며, 동기 호출 방식과 달리 요청 수에 비례한 스레드 증가가 필요하지 않습니다.

### 결과

![1차 개선 후 응답 시간 측정 결과](/imgs/image2.png)

Scale-Up 없이 동일한 하드웨어 리소스로도 더 많은 요청을 처리할 수 있게 되었습니다. 외부 API 호출 부분의 응답 시간은 약 10초에서 6초로 **40% 정도 단축**되었습니다. 비효율적인 작업 처리가 줄어들고, 큰 I/O 병목이 제거되면서 제대로 응답이 처리되는 느낌이 들기 시작했습니다.


## 두 번째 접근 — LLM API 지역 후보지 추천 및 

하지만 LLM API를 사용하는 로직이 여전히 존재했기 때문에 전체 응답 시간은 약 20초 수준에 머물렀습니다. 일반적으로 LLM을 사용하는 서비스에서는 스트리밍 응답(SSE, WebSocket 등)을 활용해 응답을 순차적으로 내려주며 체감 속도를 개선하는 방식을 자주 사용합니다.

그러나 현재 구조는 최종 추천 결과가 완성되어야만 사용자에게 의미 있는 정보를 제공할 수 있었습니다. 중간 결과를 스트리밍으로 노출하는 것은 오히려 혼란을 줄 수 있다고 판단했고, 스트리밍 방식은 현재 서비스 흐름에 적합하지 않다고 결론 내렸습니다.

### 변경 방향

먼저 장소 검색 데이터를 기반으로 지역 추천을 하던 로직에서 장소 데이터 없이도 지역을 추천하도록 변경하여 각 단계 사이의 종속성을 줄여보고자 하였습니다. 기존에는 장소 추천 로직 내부적으로 장소를 검색하고, 일정 수치 이상이라면 해당 후보지를 추천 지역으로 선정하도록 구성되어 있었지만, 후보지에 대한 장소 검색 결과 수가 일정 기준치 미만이라면 해당 지역을 후보지에서 제거하도록 추가 로직을 구성하여 해결하고자 하였습니다. 이후 `@Async`와 `CompletableFuture`를 통해 소요 시간이 긴 LLM API와 이동 경로 조회 로직을 각각 별도의 스레드풀에서 병렬로 처리했습니다.

먼저 이전에 작업한 `findRoutes` 함수를 비동기로 처리할 수 있도록 CompletableFuture로 래핑하고 `@Async` 키워드를 추가합니다.

```java
@Async("asyncTaskExecutor")
@Override
public CompletableFuture<List<Route>> findRoutesAsync(final List<StartEndPair> placePairs) {
    return findRoutes(placePairs)
            .collectList()
            .toFuture();
}
```

그리고 LLM API 처리 로직 간 종속성을 없앤 후, 다음과 같이 별도의 스레드에서 병렬로 실행할 수 있도록 구성하였습니다.

```java
@Async("asyncTaskExecutor")
@Override
public CompletableFuture<Map<Place, List<RecommendedPlace>>> recommendPlacesAsync(
        final List<Place> targets,
        final String requirement
) {
    return recommendPlaces(targets, requirement).toFuture();
}

private Mono<Map<Place, List<RecommendedPlace>>> recommendPlaces(final List<Place> targets, final String requirement) {
    Map<Place, List<KakaoApiResponse>> searchedAllPlaces = searchPlacesWithRequirement(targets, requirement);

    return Flux.fromIterable(searchedAllPlaces.entrySet())
            .flatMap(entry -> processPlaceFilteringAsync(entry.getKey(), entry.getValue(), requirement))
            .collectMap(Map.Entry::getKey, Map.Entry::getValue);
}
```

이후 외부 API 호출을 비동기 처리하던 로직과 LLM API 호출 로직을 동시에 실행했습니다.

### 결과

![비동기 병렬 로직으로 변경 후 실행 결과](/imgs/image3.png)

순차 호출 시 약 20초 소요되던 로직이 병렬 처리 시 평균 7초 내외로 **약 60% 단축**되었습니다. 응답 속도가 확실히 개선됨을 확인할 수 있었습니다.

### 새로운 문제

하지만 예상치 못한 문제가 발생했습니다. ODsay API의 공식 문서에 명시되지 않은 초당 요청 제한 수 정책이 존재했기 때문에 동시에 여러 사용자가 요청을 보낼 경우 정상적으로 서비스를 제공할 수 없었습니다.


## 세 번째 접근 — ODsay API 대체 알고리즘 구현

ODsay API의 초당 요청 제한은 구조 개선과 관련 없는 근본적인 한계였습니다. 문서에 명시된 제한이 없어 예측도 어렵고, 동시 요청이 많을 때는 429 에러가 불규칙적으로 발생했습니다.

딜레이를 걸거나 회당 요청 수를 제한하는 식으로 임시 대응했지만 말 그대로 임시 방편일 뿐이었고, 사용자 경험을 저하시키는 요인이었기 때문에 장기적인 관점에서 결국 외부 API에 의존하지 않는 방향으로 나아가야 한다는 결론에 도달했습니다. 따라서 Odsay API의 경로 탐색을 대체할 내부 로직을 직접 구현하기로 결정하였습니다.

### 변경 방향

먼저 [공공데이터 포털](http://www.data.go.kr/)에서 서울 지하철 약 600개 역에 대한 노선 데이터 수집합니다.

![저장된 데이터](/imgs/image5.png)

이후 각 역 간 연결 관계를 그래프로 모델링하여 사진과 같은 형태로 저장하고, 다익스트라(Dijkstra) 기반 탐색으로 최소 이동 시간 및 환승 조건으로 최적 경로를 선택합니다.  

```java
PriorityQueue<Node> pq = new PriorityQueue<>(Comparator.comparingInt(n -> n.time));

for (Edge edge : getEdges(currentStation)) {
    SubwayStation neighbor = edge.getDestination();
    if (visited.contains(neighbor)) continue;

    int newTime = times.get(currentStation) + edge.getTimeInSeconds();

    if (!edgeLines.get(currentStation).equals(edge.getSubwayLine())) {
        newTime += TRANSFER_TIME;
    }

    if (newTime < times.get(neighbor)) {
        times.put(neighbor, newTime);
        prev.put(neighbor, currentStation);
        pq.add(new Node(neighbor, newTime));
    }
}
```

해당 코드는 내부적으로 경로 계산하는 로직 중 일부입니다. 이 알고리즘은 단순하지만, 데이터 규모가 작아 충분히 빠르게 동작했습니다. 외부 API 호출이 필요 없기 때문에 약 5초 소요되던 ODsay 호출 구간이 **0.1초 이내**로 처리되었습니다.

![이동 경로 계산 로직 추가 후 실행 결과](/imgs/image4.png)

또한 해당 로직을 도입한 후 추가적인 이점이 생겼습니다. 외부 서버 영향으로 인한 실패율이 사라지게 되었고, 외부 API 호출로 인한 비용 문제가 해소되어 결과적으로 서비스 독립성을 확보할 수 있었습니다.


## 결과 요약

| 단계 | 주요 개선 포인트 | 적용 기술 / 방법 | 평균 응답 시간 | 주요 효과 |
|------|------------------|------------------|----------------|------------|
| 초기 | 외부 API 다중 호출 (LLM, ODsay, Kakao 순차) | RestClient (동기) | 약 30초 | 병목 다수, UX 저하 |
| 1차 | 외부 API 병렬화 | WebClient + Flux.merge() | 약 20초 | I/O 효율 개선, 90% 단축 |
| 2차 | LLM API + 외부 API 병렬 처리 | @Async + CompletableFuture | 약 8초 | 병목 완화, 병렬 실행 |
| 3차 | ODsay 대체 알고리즘 | 공공데이터 + Dijkstra 알고리즘 기반 경로 탐색 | 약 5초 | 외부 의존성 제거 |


## 7. 회고

이번 프로젝트를 통해 얻은 큰 교훈은 단계별 성능 개선을 통해 사용자 경험을 향상시키는 다양한 방법에는 정답이 없다는 점입니다. Flux, WebClient, CompletableFuture 등 다양한 비동기 처리 기술을 실제 서비스에 적용하며 기술 선택은 프로젝트 상황과 팀 역량에 맞춰야 한다는 점도 상기시킬 수 있었습니다. 비동기 및 병렬 처리로 응답 시간을 줄일 수 있었지만 근본적인 병목(외부 제한, 구조적 문제)은 다른 접근이 필요했습니다. 여러 방법을 시도했고, 시도한 방법 외에도 여러 방법이 있으며, 프로젝트 상황, 일정, 데이터 특성에 따라 적절한 방법을 선택하고 조합하는 것이 중요함을 느꼈습니다. 이번 경험을 바탕으로 향후 서비스 독립성과 확장성을 위한 개선을 원활하게 진행할 수 있을 것이라 기대되며, 앞으로도 사용자 편의성을 고려한 다양한 개선 작업을 진행하려 합니다.

> 🧭 [모잇지](https://moitz.kr)의 개선 여정이 비슷한 문제를 겪는 팀들에게 작지만 실질적인 참고가 되길 바랍니다.  
