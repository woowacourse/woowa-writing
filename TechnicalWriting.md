# 🚀 React Query로 구현하는 효율적인 서버 상태 관리

## 목차

- [🚀 React Query로 구현하는 효율적인 서버 상태 관리](#-react-query로-구현하는-효율적인-서버-상태-관리)
  - [목차](#목차)
  - [1. 서문: 데이터 패칭의 혼돈 속에서](#1-서문-데이터-패칭의-혼돈-속에서)
  - [2. Situation(상황) : 동기화되지 않는 데이터의 혼란](#2-situation상황--동기화되지-않는-데이터의-혼란)
    - [발생한 문제](#발생한-문제)
    - [2-1. 기존 방식의 한계](#2-1-기존-방식의-한계)
  - [3. Task(과제) : 문제 해결 목표 정의](#3-task과제--문제-해결-목표-정의)
    - [3-1. 대안 검토 및 선택 이유 (참고)](#3-1-대안-검토-및-선택-이유-참고)
  - [4. Action(행동1) : React Query의 도입과 코드 전환](#4-action행동1--react-query의-도입과-코드-전환)
    - [4-1. 핵심 기능 ① — 데이터 동기화: (Invalidate Query)](#4-1-핵심-기능---데이터-동기화-invalidate-query)
    - [4-2. 핵심 기능 ② — 캐싱(Caching)과 staleness 관리](#4-2-핵심-기능---캐싱caching과-staleness-관리)
  - [5. Action(행동2) : 무한 스크롤에서의 적용: useInfiniteQuery](#5-action행동2--무한-스크롤에서의-적용-useinfinitequery)
    - [5-1. 페이지별 캐싱 전략 수립](#5-1-페이지별-캐싱-전략-수립)
  - [6. Result(결과) : 성능 최적화 및 결과](#6-result결과--성능-최적화-및-결과)
    - [✅ (1) 낙관적 업데이트(Optimistic Update)](#-1-낙관적-업데이트optimistic-update)
    - [✅ (2) 실시간 데이터 신뢰성 확보](#-2-실시간-데이터-신뢰성-확보)
    - [6-1. React Query 도입 전후 비교](#6-1-react-query-도입-전후-비교)
  - [7. 정리 및 회고](#7-정리-및-회고)
    - [7-1. React Query 설계 철학](#7-1-react-query-설계-철학)
    - [7-2. React Query를 통한 선언적 상태 관리](#7-2-react-query를-통한-선언적-상태-관리)
    - [7-3. 도입 과정에서의 학습 포인트](#7-3-도입-과정에서의-학습-포인트)
    - [7-4. 결론 및 회고](#7-4-결론-및-회고)
  - [8. 요약](#8-요약)

## 1. 서문: 데이터 패칭의 혼돈 속에서

---

프론트엔드 개발을 하다 보면 필연적으로 **데이터 패칭(data fetching)** 과 **상태 관리(state management)** 에 대한 고민을 하게 된다.<br/>
특히 서비스 규모가 커지고 다뤄야 할 데이터가 많아질수록, 단순히 fetch 나 axios를 이용한 API 호출만으로는 한계가 드러난다.

우리 팀 역시 초기에 직접 만든 `apiClient`를 사용하며 큰 문제 없이 진행했으나, 프로젝트가 확장되면서 **데이터 동기화 문제, 무한 스크롤의 비효율, 화면 깜빡임(flickering)** 등의 현상을 겪었다.

이 문제를 해결하기 위해 **React Query**를 도입하게 되었다.
<br/>

## 2. Situation(상황) : 동기화되지 않는 데이터의 혼란

---

프로젝트는 관리자 대시보드 서비스를 중심으로 구성되어 있었다.<br/>
이 대시보드에는 사용자들이 남긴 피드백 목록이 무한 스크롤 형태로 표시되고, 상단에는 피드백 통계(총 개수, 완료 개수 등)가 실시간으로 보여졌다.

문제는 관리자가 피드백 목록에서 특정 항목을 **삭제**하거나 **완료** 처리했을 때 발생했다.

![situation](/.github/assets/TechnicalWriting/situation.png)

<br/>

### 발생한 문제

1. **데이터 불일치**

   - 목록에서 피드백을 삭제해도 통계 패널의 숫자가 갱신되지 않았다.
   - 데이터를 갱신하려면 페이지를 새로고침해야 했고, 이는 UX 결함으로 이어졌다.

2. **무한 스크롤 동기화 문제**

   - 중간 피드백이 삭제되면 전체 데이터를 다시 불러와야 했고, 화면 깜빡임과 스크롤 위치 손실이 발생했다.

결국 핵심 과제는 **데이터 일관성과 자연스러운 UI 경험의 동시 달성**이었다.

<br/>

### 2-1. 기존 방식의 한계

React Query를 도입하기 전, 아래와 같이 `useEffect`와 `useState`로 데이터를 관리했다.

```jsx
// 조직 통계 데이터를 직접 패칭해 관리하는 기존 방식
export default function useUserOrganizationsStatistics() {
  const [statistics, setStatistics] = useState({
    reflectionRate: "0",
    confirmedCount: "0",
    waitingCount: "0",
    totalCount: "0",
  });

  useEffect(() => {
    // 컴포넌트 마운트 시 API 호출
    const getData = async () => {
      const response = await getOrganizationStatistics({ organizationId: 1 });
      // 응답 데이터를 state에 저장
      setStatistics(response.data);
    };
    getData();
  }, []);

  return { statistics };
}
```

이 방식은 간단하지만 다음과 같은 구조적 한계를 갖는다.

- **데이터 갱신 제어 불가** : API 응답 이후 언제, 어떤 조건으로 데이터를 다시 패칭해야 할지 명확하지 않다.
- **전역 공유 어려움** : 같은 데이터를 여러 컴포넌트에서 재사용하려면 별도의 상태 관리 로직이 필요하다.
- **불필요한 API 호출** : 중복 패칭이 잦아 성능이 저하된다.
- **복잡한 상태 관리** : 로딩, 에러, 성공 상태를 모두 수동으로 처리해야 한다.

이러한 문제는 프로젝트가 확장될수록 코드 유지보수성을 떨어뜨렸다.

<br/>

## 3. Task(과제) : 문제 해결 목표 정의

---

문제상황을 정리하고, React Query를 도입하기 전 우리가 해결해야 하는 문제를 먼저 간단히 정의해봤다.

- 피드백 ‘완료’ 또는 ‘삭제’ 시에 **대시보드 패널에도 값이 즉각적으로 반영**되어야 한다.
- 피드백 ‘삭제’시에 제거된 피드백을 제외한 **상태가 변경되지 않은 피드백들을 다시 호출하지 않아야 한다.**

<br/>

### 3-1. 대안 검토 및 선택 이유 (참고)

React Query를 도입하기 전, 우리는 다양한 데이터 패칭 및 상태 관리 라이브러리를 검토해봤다. <br/>
우리가 중점적으로 봤던 부분은 **서버 상태**를 안정적으로 관리할 수 있는지, 데이터 동기화 및 캐싱(무한 스크롤)문제를 해결할 수 있는지를 살펴봤다.

React Query 외에 고민해봤던 대안으로는 대표적으로 `RTK Query`와 `SWR`, `Apollo Client` 등이 있다.

| 라이브러리                                         | 주요 특징                                                                                    | 한계점                                                                                                                             |
| -------------------------------------------------- | -------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| **Redux Toolkit Query (RTK Query)**                | Redux 공식 툴킷 기반의 서버 상태 관리 라이브러리. Redux Store와 완벽하게 통합됨.             | Redux를 이미 사용 중인 환경에는 적합하지만, **Redux 미사용 프로젝트에서는 초기 설정이 과도하게 복잡함**.                           |
| **SWR (by Vercel)**                                | React Hooks 기반의 간결한 데이터 패칭 라이브러리. `useSWR` 훅 하나로 캐싱·revalidation 가능. | 기본 기능은 강력하지만, **복잡한 캐싱 정책(staleTime, cacheTime), 무한 스크롤, 낙관적 업데이트 등 고급 기능 구현 시 제약**이 많음. |
| **Apollo Client (GraphQL)**                        | GraphQL API에 최적화된 클라이언트로, 정교한 캐싱과 쿼리 병합 기능 제공.                      | REST API 기반 프로젝트에서는 **오버엔지니어링**이 되며, GraphQL 서버 구축 비용이 발생함.                                           |
| **TanStack Query (React Query v5 이후 통합 이름)** | React Query의 최신 통합 버전. 다양한 프레임워크 지원.                                        | 당시 프로젝트 시점(React Query v4)에서는 도입 시 **호환성 테스트가 필요했음**. 안정성을 위해 React Query(v4) 선택.                 |

<br/>

**React Query 선택 근거**

1. REST 기반 프로젝트에 최적화된 서버 상태 관리

우리 프로젝트는 GraphQL이 아닌 RESTful API 기반이었기 때문에, <br/>
`Redux Toolkit Query`나 `Apollo Client`는 구조적으로 과했거나 설정이 복잡했다.<br/>
React Query는 별도 아키텍처 변경 없이 **자동 캐싱·refetch·동기화**를 지원해 REST 환경에 바로 적용할 수 있었다.

2. 세밀한 캐싱 제어와 데이터 신선도 관리

React Query는 `staleTime`, `cacheTime`, `refetchOnWindowFocus` 등을 통해
**데이터의 신선도(freshness)** 를 정밀하게 제어할 수 있었다.
SWR보다 유연한 이 구조 덕분에, 대시보드는 즉시 갱신하고 설정 페이지는 장기 캐싱하는 등 페이지별 전략을 세밀히 조정할 수 있었다.

3. 무한 스크롤에서의 자연스러운 로딩 경험

`useInfiniteQuery`는 커서 기반 페이징을 손쉽게 구현할 수 있어,
데이터 추가·삭제 시에도 깜빡임 없이 부드러운 갱신이 가능했다.
`SWR`이나 `Redux Toolkit Query`에서는 무한 스크롤을 직접 관리해야 해 페이지 캐싱·커서 계산·refetch 제어를 모두 수동으로 구현해야 했다.

<br/>

## 4. Action(행동1) : React Query의 도입과 코드 전환

---

React Query는 선언적 데이터 패칭과 자동 동기화를 제공한다. <br/>
`useQuery` 하나로 로딩, 에러, 성공 상태를 모두 관리할 수 있어 코드가 간결해졌다.

### 4-1. 핵심 기능 ① — 데이터 동기화: (Invalidate Query)

React Query는 invalidateQueries 기능을 통해 <br/>
특정 `queryKey`를 사용하는 모든 컴포넌트를 **자동으로 최신 상태로 갱신**할 수 있다.

즉, 사용자가 데이터를 수정하거나 삭제하는 등의 변경이 발생하면 <br/>
해당 쿼리에 연결된 컴포넌트들이 **자동으로 refetch** 되어, 화면에 즉시 최신 데이터가 반영된다.

> **💡 Note:** 자세한 설명 보기
>
> 이를 구현할 때는 `useMutation`과 `queryClient.invalidateQueries`를 함께 사용한다.<br/> `useMutation`은 데이터 수정(쓰기) 작업을 담당하고,<br/> 그 작업이 성공하면 `invalidateQueries`를 호출하여 관련된 쿼리들을 **무효화(invalidate)한다.**
>
> 여기서 “무효화”란, <br/>
> React Query가 해당 쿼리의 캐시 데이터를 “더 이상 신뢰할 수 없다”고 표시하는 것을 의미한다. <br/>
> 즉, 캐시는 남아 있지만 “이 데이터는 오래되었으니 새로 가져와야 한다"고 판단하고 데이터를 다시 refetch 하게 되는 것이다.
>
> 그 결과, React Query는 무효화된 쿼리를 **자동으로 다시 요청(refetch)** 하여 최신 데이터를 가져오고, 이를 사용하는 모든 컴포넌트를 즉시 업데이트한다.

<br/>

```jsx
// 피드백 상태 변경 후 관련 쿼리 자동 갱신
const queryClient = useQueryClient();

const confirmMutation = useMutation({
  // 피드백 상태 변경 API 요청
  mutationFn: ({ feedbackId, comment }) =>
    patchFeedbackStatus({ feedbackId, comment }),
  // 요청 성공 시: 관련 데이터 쿼리 무효화 → 자동 refetch
  onSuccess: () => {
    // 조직 통계 패널 데이터 갱신
    queryClient.invalidateQueries({
      queryKey: QUERY_KEYS.organizationStatistics(organizationId),
    });
    // 피드백 목록(무한 스크롤) 데이터 갱신
    queryClient.invalidateQueries({ queryKey: QUERY_KEYS.infiniteFeedbacks });
  },
});
```

이 방식으로 다음 두 가지가 가능해졌다!

- 통계 패널 자동 갱신 — 새로고침 없이 최신 피드백 수 반영
- 무한 스크롤 목록 실시간 반영 — 수정·삭제 시 자연스러운 UI 갱신

<br/>

### 4-2. 핵심 기능 ② — 캐싱(Caching)과 staleness 관리

React Query의 장점은 **데이터를 언제 다시 가져올지 정밀하게 제어**할 수 있다는 것이다. <br/>
React Query는 내부적으로 API 응답을 메모리에 **캐싱**하지만, 캐시가 있다고 해서 항상 네트워크 요청을 생략하는 것은 아니다.<br/>
**데이터의 신선도(freshness)** 를 판단해, 필요할 때만 자동으로 다시 가져오도록 설계되어 있다.

이 동작을 제어하는 핵심 요소가 바로 `staleTime`과 `cacheTime` 두 가지 옵션이다.

- **staleTime** : 데이터를 “fresh”로 간주하는 시간, 이 시간 내에는 refetch하지 않는다.
- **cacheTime** : 캐시를 메모리에 유지하는 시간, 이 시간이 지나면 가비지 컬렉션(GC)으로 제거된다.

> **💡 가비지 컬렉션(Garbage Collection)** <br/>
> 더 이상 사용되지 않는 데이터를 자동으로 메모리에서 정리해주는 동작을 말한다.<br/>
> 즉, React Query는 cacheTime이 지나면 오래된 캐시 데이터를 스스로 정리한다.

기본값은 다음과 같다.

```jsx
staleTime = 0; // 항상 오래된 데이터로 간주 → 매번 refetch
cacheTime = 5 * 60 * 1000; // 5분 동안 캐시 유지
```

즉, React Query는 기본적으로 “**항상 최신 데이터를 보장**”하는 방향으로 설계되어 있다. <br/>
`staleTime`이 0이면, 데이터는 즉시 **“stale(오래된)”** 상태로 간주되어 매번 refetch한다.<br/>
`cacheTime`이 만료되면 캐시가 메모리에서 제거되어 다시 요청해야 한다.

![stale](/.github/assets/TechnicalWriting/stale.png)

결과적으로, 이러한 기본 설정은 데이터의 최신성은 극대화하지만 <br/>
반대로 **불필요한 네트워크 호출이 증가**할 수 있다는 단점도 있다.
따라서 서비스 특성에 맞게 `staleTime`과 `cacheTime`을 조정하는 것이 중요하다.

> **React Query 캐시 상태 흐름도** <br/>
> fresh: 최신 데이터로 간주되어 바로 사용 가능 <br/>
> stale: 캐시에 있지만, 일정 시간이 지나 재검증이 필요한 상태 <br/>
> inactive: 사용 중인 컴포넌트가 없어도 일정 시간 캐시에 남아 있는 상태 <br/>
> garbage collected: cacheTime이 지나 캐시에서 완전히 제거된 상태

<br/>

## 5. Action(행동2) : 무한 스크롤에서의 적용: useInfiniteQuery

---

무한 스크롤에서는 useInfiniteQuery 훅을 사용했다.

```jsx
// 커서 기반 무한 스크롤 데이터 패칭
const query = useInfiniteQuery({
  queryKey: ["infinity", key, url, size], // 쿼리 식별 키
  enabled: enabled && Boolean(url),
  retry: 3, // 요청 실패 시 재시도 횟수
  queryFn: ({ pageParam }) => fetchCursorPage({ url, size, pageParam }), // 커서 기반 페이지 데이터 fetch 함수
  getNextPageParam: (lastPage) =>
    lastPage?.hasNext ? lastPage.nextCursorId : undefined, // 다음 페이지 커서 계산
});
```

여기서 `staleTime`과 `cacheTime`을 명시하지 않으면 기본 설정(staleTime=0, cacheTime=5분)이 적용되어 **페이지 접속 시마다 새로 API 요청**이 일어난다. <br/>

이 문제를 해결하기 위해 우리는 페이지의 특성과 사용 패턴에 맞춘 캐싱 전략을 수립했다.

### 5-1. 페이지별 캐싱 전략 수립

| 페이지             | staleTime | cacheTime | 이유                               |
| ------------------ | --------- | --------- | ---------------------------------- |
| 피드백 대시보드    | 0초       | 5분       | 실시간성 중요. 최신 통계 반영 필요 |
| 관리자 방 목록     | 5분       | 15분      | 변동 적음. 캐시 유지 유리          |
| 관리자 방 대시보드 | 0초       | 5분       | 최신 데이터 필요. 캐시 fallback    |
| 관리자 설정 페이지 | 1시간     | 24시간    | 정적 데이터. 캐시 재활용 적합      |
| QR 코드            | 24시간    | 24시간    | 정적 리소스. 재호출 불필요         |
| 관리자 정보        | 1분       | 5분       | 로그인 계정별 갱신 필요            |

이 정책을 통해 “**필요할 때만 refetch**” 하고, “**불필요한 API 호출**” 을 줄일 수 있었다.

<br/>

## 6. Result(결과) : 성능 최적화 및 결과

---

### ✅ (1) 낙관적 업데이트(Optimistic Update)

> React Query의 또 다른 강점은 낙관적 업데이트다. <br/>
> 서버 응답을 기다리지 않고 UI를 먼저 변경한 뒤, 실패 시 롤백한다.

- 오프라인 상태: 일시적으로 값이 증가했다가, 요청 실패 시 원래 상태로 복귀
  ![optimistic-error](/.github/assets/TechnicalWriting/optimistic-error.gif) <br/>
- 느린 네트워크(3G): 서버 응답이 느려도 사용자에게는 즉시 반영된 것처럼 보임
  ![optimistic-3G](.github/assets/TechnicalWriting/optimistic-3G.gif)

<br/>

### ✅ (2) 실시간 데이터 신뢰성 확보

- admin dashboard: staleTime=0 → 항상 최신 데이터 패칭
  ![dashborad-not-cache](/.github/assets/TechnicalWriting/dashborad-not-cache.png)<br>

- QR code 페이지: staleTime=24h → 동일 요청 시 캐시 재활용
  ![qr-code-cache](/.github/assets/TechnicalWriting/qr-code-cache.png)

이를 통해 페이지 특성에 따라 성능 최적화와 데이터 신뢰성의 균형을 맞출 수 있었다.

<br/>

### 6-1. React Query 도입 전후 비교

| 항목          | 도입 전 (apiClient)             | 도입 후 (React Query)            |
| ------------- | ------------------------------- | -------------------------------- |
| 데이터 캐싱   | ❌ 직접 구현 필요               | ✅ 자동 캐싱 지원                |
| 데이터 동기화 | ❌ 수동 업데이트 필요           | ✅ invalidateQueries로 자동 반영 |
| 코드 복잡도   | 높음 (useEffect, useState 다수) | 낮음 (선언적 구조)               |
| API 호출 횟수 | 많음                            | 최적화 가능                      |
| UX            | 새로고침 필요                   | 실시간 반영                      |

React Query는 코드 품질뿐 아니라 팀 생산성에도 직접적인 영향을 미쳤다.

> 특히 QR 코드 페이지의 경우, staleTime을 24시간으로 설정해 동일 요청 시 캐시를 재활용함으로써 <br/>
> API 호출 횟수를 약 90% 이상 줄여 네트워크 트래픽을 크게 절감할 수 있었다.

<br/>

## 7. 정리 및 회고

---

### 7-1. React Query 설계 철학

React Query는 “데이터의 최신성과 사용자 경험 간의 균형을 자동으로 관리”하는 것을 목표로 한다.
`staleTime`, `cacheTime`, `refetchOnWindowFocus`, `refetchOnReconnect` 등의 옵션을 통해 정확한 시점 제어가 가능하다.

예를 들어, 대시보드처럼 실시간성이 필요한 페이지에서는 staleTime=0으로 설정하고,
정적 설정 페이지에서는 staleTime=1시간으로 두어 서버 부하를 줄였다.

<br/>

### 7-2. React Query를 통한 선언적 상태 관리

React Query를 도입하면서 얻은 가장 큰 이점은 선언적 프로그래밍 패러다임이었다.
`useQuery`, `useMutation`, `useInfiniteQuery`를 이용하면 다음과 같은 구조로 사고할 수 있다.

- “언제 데이터를 가져올까?” → `enabled` 옵션
- “데이터가 바뀌면 어떻게 동기화할까?” → `invalidateQueries`
- “네트워크 실패 시 어떻게 복구할까?” → `retry` / `onError`\
- “UI는 언제 업데이트할까?” → `onSuccess`, `onSettled`

이 덕분에 코드의 유지보수성과 테스트 용이성이 크게 향상되었다.

<br/>

### 7-3. 도입 과정에서의 학습 포인트

React Query 도입 과정에서 우리는 ‘캐싱은 설정의 문제가 아니라 설계의 문제’라는 교훈을 얻었다.
캐싱의 효율은 `queryKey` 설계, `staleTime·cacheTime` 설정, `invalidateQueries`의 활용 방식에 달려 있다.

### 7-4. 결론 및 회고

이전에는 React Query를 단순히 데이터 패칭을 더 편리하게 할 수 있도록 도와주는 도구로서만 봤다면, 이번 경험을 통해 React Query는 데이터의 일관성을 유지하며 관리할 수 있도록 돕는 강력한 도구임을 실감했다.

우리 팀은 이를 통해 다음을 달성할 수 있었다.

- 실시간 데이터 동기화 문제 해결
- 불필요한 API 호출 감소
- 코드 구조 간결화
- 사용자 경험(UX) 향상

무엇보다 React Query는 데이터의 “신뢰도”와 “성능” 사이의 균형을 조정하는 유연한 도구라는 점을 깨달았다.

<Br/>

## 8. 요약

---

- React Query는 서버 상태 관리 라이브러리이다.
- staleTime, cacheTime을 통해 데이터 최신성과 성능 간의 균형을 조정할 수 있다.
- invalidateQueries()는 변경된 데이터를 자동으로 동기화하여 개발 효율성을 높인다.
- 페이지 특성에 맞는 캐싱 전략 설계가 성능 최적화의 핵심이다.
- React Query의 철학은 “데이터 일관성과 사용자 경험의 조화”에 있다.
