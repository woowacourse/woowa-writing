# 모잇지 추천 기능의 성능 개선 과정

> 이동 시간 기반 모임 추천 서비스 [모잇지(Moitz)](https://moitz.kr)에서 추천 API 응답 시간을 20초 이상에서, 약 5초 이하로 줄이기까지의 개선 여정


## 서비스 배경과 문제 인식

모잇지는 **이동 시간이 공평한 모임 장소를 정하자**는 아이디어에서 출발했습니다.
사용자들이 각자 출발지를 입력하면, 각 출발지로부터 이동 시간이 비슷하고 모임 카테고리(예: 카페, 식당 등)의 장소가 존재하는 지역을 추천해주는 서비스입니다.

내부적으로 위치(Location)는 만날 동네(성수역, 행리단길 등), 장소(Place)는 만날 곳(특정 가게 등)으로 구분합니다. 추천은 이 두 개념을 중심으로 이루어집니다.

### 초기 MVP(Minimum Viable Product) 구조

개발 초기, 적은 노력으로 사용자의 선호에 대한 피드백을 빠르게 받기 위해 MVP 개발을 결정했습니다. MVP는 Minimum Viable Product의 약자로 가장 핵심적인 가치를 검증할 수 있을 만큼만 기능을 구현한 제품을 의미합니다. 저희는 외부 API를 적극 기용해서 많은 로직을 대체하고 응답을 얻기 위한 MVP 개발 계획을 세웠습니다.

- LLM(Large Language Model) - Gemini API : 지역 후보지 선정, 검색 장소 데이터 기반 지역 추천
- Odsay API: 출발지로부터 추천 지역까지의 이동 경로 및 시간 계산
- Kakao Place API: 추천 지역 내 장소 검색

이 세 가지 API를 조합하여 로직을 설계하였습니다. 먼저 사용자로부터 입력받은 각 출발지 이름을 통해 역 데이터를 검색합니다. 검색한 출발지 데이터를 사전에 정의한 프롬프트와 함께 Gemini Client에 장소 후보지 선정을 요청합니다. 이후 Odsay API를 통해 각 출발지로부터 각 후보지까지의 이동 경로 및 시간 계산 정보를, Kakao Place API를 통해 후보지의 장소를 검색합니다. 최종적으로 Gemini에 해당 정보를 함수 호출로 재전달하여 적절한 장소를 추천 받습니다.

해당 과정을 통해 필요한 수준의 추천 결과를 얻을 수 있었습니다. 여기서 문제는 이 과정이 너무 느리다는 것이었습니다.


## 문제 정의 — 느림의 정체

테스트에서 API 응답 시간은 **평균 30~50초** 정도였습니다. 단일 요청의 경우에도 이 속도였으니 동시 요청 수가 늘어나면 체감 지연은 기하급수적으로 증가했습니다.

LLM API 등 여러 외부 API를 동시에 사용하고 있었기 때문에 응답이 오래 걸린다는 것은 알고 있었지만, 어느 항목에서 응답 속도가 느린지 정확히 파악하기 위해 각 로직 단계별 소요 시간을 정량적으로 측정하기로 했습니다.

Spring Framework에서 제공하는 StopWatch 클래스를 활용해 요청 처리 과정의 각 구간을 세분화하여 실행 시간을 기록했습니다. 이 방식은 외부 모니터링 도구를 붙이지 않아도 서버 내부에서 빠르게 병목 구간을 식별할 수 있어 러닝 커브가 낮기 때문에 현재 상황에 적합하다고 판단하였습니다.

<img width="644" height="480" alt="image" src="https://github.com/user-attachments/assets/de1f182e-2e6b-4692-93f5-43b3f5370ff9" />


이동 경로 조회 및 장소 추천에서 소요 시간이 평균 5초 이상 소요되는 것을 관측할 수 있었고, 이는 사용자 이탈이 충분히 발생할 수 있을 정도의 수치였습니다.

초기 로직 대부분이 내부 연산이 아닌 외부 API 호출이었기 때문에 발생한 문제였습니다. 외부 API는 네트워크 지연이나 제한(초당 요청 수 등)에 영향을 받기 때문에 단순한 코드 최적화로는 개선이 불가능했습니다. 다수의 외부 API 호출로 인해 대부분의 연산이 네트워크 I/O로 인해 발생했고, 이로 인해 응답 시간이 선형적으로 증가했습니다.

다만 해당 문제들은 MVP 단계의 구조적 한계로 인해 발생한 문제였습니다. 기능 검증을 우선으로 조합한 구조였기 때문에 최적화된 상태가 아니었습니다. 내부적으로 계산 가능한 부분은 외부 API를 걷어내는 것이 가장 좋겠지만, 마감 일정에 따른 현재 진행 상황을 고려했을 때 남은 일정 내 구현 가능성을 보장하기 어려웠습니다. 또한 남은 기능 마저 개발하며 이후 개선하는 방향을 생각할 수 있겠지만 서비스를 지속적으로 운영하며 개선하는 팀 내의 목표를 생각했을 때, 사용자가 MVP를 사용하며 체감할 속도 저하를 두고만 볼 순 없었습니다.

따라서 팀은 초기 단계에서는 **기존 로직을 유지한 채 성능을 개선할 수 있는 방법**을 먼저 탐색하기로 했습니다.


### 초기 접근 방식

초기에는 로직 개선이 아닌 다른 해결 방안을 탐색했습니다. 가장 먼저 떠오른 것은 '굳이 모든 데이터가 완성되어야만 응답을 전송할 필요가 있을까?'라는 의문이었습니다.

LLM을 사용하는 다른 서비스에서는 스트리밍 응답(SSE, WebSocket 등)을 활용해 응답을 순차적으로 내려주며 체감 속도를 개선하는 방식을 자주 사용되곤 합니다.

그러나 구상한 서비스는 최종 추천 결과가 완성되어야만 사용자에게 의미 있는 정보를 제공할 수 있었습니다. 중간 결과를 스트리밍으로 노출하는 것은 오히려 혼란을 줄 수 있다고 판단했고, 스트리밍 방식은 현재 서비스 흐름에 적합하지 않다고 결론 내렸습니다.

따라서 가장 먼저 각 단계 간 종속성을 제거하고, 독립적으로 실행되는 로직을 동시에 실행할 수 있는 구조로 개선할 계획을 세웠습니다.



### 1차 개선

장소 추천 로직의 소요 시간이 긴 이유는 장소 추천의 품질을 향상시키기 위해 도입한 LLM 함수 호출 기능 때문이었습니다. LLM 도움 없이 장소를 추천받을 수 있도록 필요한 데이터를 검색할 수 있도록 최적의 검색 키워드를 찾아 장소 추천에 사용되던 LLM 함수 호출 기능을 제거하였습니다. 이 과정만으로 수 초의 대기 시간을 줄였습니다.


### 2차 개선

`@Async`와 `CompletableFuture`를 통해 이동 경로 조회 로직과 장소 추천 로직을 각각 별도의 스레드풀에서 병렬로 처리했습니다.

먼저 이전에 작업한 이동 경로 조회 로직을 비동기로 처리할 수 있도록 CompletableFuture로 래핑하고 `@Async` 키워드를 추가합니다.

```java
@Async("asyncTaskExecutor")
@Override
public CompletableFuture<List<Route>> findRoutesAsync(final List<StartEndPair> placePairs) {
    return findRoutes(placePairs)
            .collectList()
            .toFuture();
}
```

그리고 다음과 같이 장소 추천 로직을 별도의 스레드에서 병렬로 실행할 수 있도록 구성하였습니다.

```java
@Async("asyncTaskExecutor")
@Override
public CompletableFuture<Map<Place, List<RecommendedPlace>>> recommendPlacesAsync(
        final List<Place> targets,
        final String requirement
) {
    return recommendPlaces(targets, requirement).toFuture();
}
```

해당 두 로직을 장소 후보지 선정 이후 동시에 실행하였습니다.

### 결과

<img width="600" height="300" alt="image" src="https://github.com/user-attachments/assets/61ed2b3c-36c0-4b79-81b1-4a72e2396436" />

순차 호출 시 약 20초 소요되던 로직이 병렬 처리 시 평균 7초 내외로 **약 60% 단축**되었습니다. 응답 속도가 확실히 개선됨을 확인할 수 있었습니다.


## 근본적인 해결

하지만 예상치 못한 문제가 발생했습니다. ODsay API의 공식 문서에 명시되지 않은 초당 요청 제한 수 정책이 존재했기 때문에 동시에 여러 사용자가 요청을 보낼 경우 정상적으로 서비스를 제공할 수 없었습니다.


### 해결 방향

ODsay API의 초당 요청 제한은 구조 개선과 관련 없는 근본적인 한계였습니다. 문서에 명시된 제한이 없어 예측도 어렵고, 동시 요청이 많을 때는 429 에러가 불규칙적으로 발생했습니다.

딜레이를 걸거나 회당 요청 수를 제한하는 식으로 임시 대응했지만 말 그대로 임시 방편일 뿐이었고, 사용자 경험을 저하시키는 요인이었기 때문에 장기적인 관점에서 결국 외부 API에 의존하지 않는 방향으로 나아가야 한다는 결론에 도달했습니다. 따라서 Odsay API의 경로 탐색을 대체할 자체적인 로직을 직접 구현하기로 결정하였습니다.

### 구현 방법


먼저 [공공데이터 포털](http://www.data.go.kr/)에서 서울 지하철 약 600개 역에 대한 노선 데이터 수집하여 다음과 같은 형태로 저장합니다.

<img width="600" height="350" alt="image" src="https://github.com/user-attachments/assets/961f1bd0-7703-4294-80b5-3ff3e9291b2e" />


이후 각 역 간 연결 관계를 그래프로 모델링하여 사진과 같은 형태로 저장하고, 다익스트라(Dijkstra) 기반 탐색으로 최소 이동 시간 및 환승 조건으로 최적 경로를 선택합니다.  

지하철 노선은 역 사이의 이동 시간이 간선 가중치가 되는 전형적인 가중 그래프이기 때문에, 최단 이동 시간을 구하기 위해서는 시작 지점에서 모든 노드까지의 최소 비용을 효율적으로 계산하는 다익스트라(Dijkstra) 알고리즘을 사용하는 것이 적합하다고 판단하였습니다.

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

<img width="600" height="350" alt="image" src="https://github.com/user-attachments/assets/bb6ab839-8794-4516-a55e-fe903043f9fb" />


또한 해당 로직을 도입한 후 외부 서버 영향으로 인한 실패율이 사라지게 되었고, 외부 API 호출로 인한 비용 문제가 해소되어 결과적으로 서비스 독립성을 확보할 수 있었습니다.


## 요구사항 확장에 따른 성능 재악화

프로젝트를 진행하며 사용자 피드백에 의해 기존 '떠들고 놀기 좋은'과 같이 형용사던 모임 목적이 '식당', '카페'와 같이 명사로 변경되었습니다. 이에 따라 모임 목적을 다중으로 선택할 수 있도록 정책이 변경되며 장소 검색 API 호출이 5회에서 최악의 경우 75회까지 발생하게 되었습니다. 평균 5초였던 응답 시간이 요청 건수에 비례하여 증가하게 되었고, 이로 인한 사용자 이탈이 예상되었습니다.

### 해결 방향

동기식 호출 구조였기 때문에 네트워크 I/O 대기로 인한 지연이 누적되었고, 이를 병렬로 처리하기로 결정하였습니다.

이전에 해결한 것처럼 `@Async`와 `CompletableFuture`의 조합을 이용한 단순 비동기 병렬 처리를 고려했지만, `@Async`는 프록시 패턴을 사용하기 때문에 같은 클래스 내부의 메서드에는 적용이 안 된다는 점, 가용 가능한 인프라 비용이 적어 낮은 스펙의 인스턴스를 사용하고 있어 이미 CPU, 메모리 사용률이 높았다는 점 등을 고려하여 다른 해결 방법을 탐색하였습니다.

RestClient는 요청당 스레드를 점유하는 블로킹 구조라 동시 요청 수가 증가할수록 스레드 생성·컨텍스트 스위칭 비용이 커지지만, WebClient는 Netty 기반의 이벤트 루프에서 비동기 방식으로 요청을 처리하여 동일한 자원으로 더 많은 API 호출을 병렬로 처리할 수 있습니다.

이에 WebClient를 사용하여 API 병렬 호출하는 방식으로 구조를 개선하였습니다.

### 구현 방법

기존 RestClient로 구성된 KakaoMapClient를 통해 외부 API를 순차적으로 호출하는 코드입니다.

```java
private List<KakaoApiResponse> searchKeywordsForCondition(
            final Place place,
            final RecommendCondition condition
    ) {
        List<KakaoApiResponse> allResponses = new ArrayList<>();
        for (String keyword : condition.getKeywords()) {
            KakaoApiResponse response = kakaoMapClient.searchPlacesBy(
                new SearchPlacesLimitQuantityRequest(
                    keyword,
                    place.getName(),
                    place.getPoint().getX(),
                    place.getPoint().getY(),
                    800,
                    3
                )
            );
            allResponses.add(response);
        }
        return allResponses;
    }
```

다음과 같이 WebClient 기반의 KakaoMapAsyncClient를 구성하고 외부 API를 `flatMap()`을 사용하여 병렬로 호출하는 구조로 변경하였습니다.

```java
private Mono<Entry<RecommendCondition, List<KakaoApiResponse>>> searchKeywordsForConditionAsync(
            final RecommendCondition condition,
            final Place place
    ) {
        return Flux.fromIterable(condition.getKeywords())
                .flatMap(keyword -> kakaoMapAsyncClient.searchPlacesByAsync(
                        new SearchPlacesLimitQuantityRequest(
                                keyword,
                                place.getName(),
                                place.getPoint().getX(),
                                place.getPoint().getY(),
                                800,
                                3
                        )
                )
                .collectList()
                .map(responses -> Map.entry(condition, responses));
    }
```

### 결과

다음과 같이 2군데의 출발지, 4가지의 모임 조건을 선택하는 데이터를 요청하여 응답 시간을 비교해보겠습니다.

```json
{
  "startingPlaceNames": [
    "수원역",
    "인천역"
  ],
  "requirements" : ["CAFE", "RESTAURANT", "BAR", "ENTERTAINMENT"]
}
```

개선 전, 실행 결과 9.5초가 소요되었습니다.  
<img width="600" height="450" alt="image" src="https://github.com/user-attachments/assets/f55a914c-65c2-477f-b439-418aa14110fa" />

개선 후, 9.5초에서 5.7초로 기존 요구사항 변경 전과 비교하였을 때 동일한 응답 시간이 소요되는 것을 확인할 수 있었습니다.  
<img width="600" height="450" alt="image" src="https://github.com/user-attachments/assets/56c95e52-2cd1-44b7-a179-50c777c35191" />


## 마무리

문제가 발생하면 원인을 분석하고, 빠르게 해결 방안을 파악하며 이를 실행하는 능력의 중요성을 절감할 수 있었습니다. 또한 사용자 경험을 향상시키는 방법에는 정답이 없으며 필요 프로젝트 상황, 일정, 데이터 특성에 따라 적절한 방법을 선택하고 조합하는 것이 필요함을 느꼈습니다. 이번 개선 경험은 향후 서비스 확장이나 유사한 문제 상황에서도 중요한 기반이 될 것입니다.
