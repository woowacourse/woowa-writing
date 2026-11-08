# 선택 속성을 필수로 바꾸려다 함수의 책임을 다시 생각했다

TypeScript로 함수를 작성할 때 옵션 객체의 속성에 `?`를 붙일지는 대개 호출 편의로 결정하곤 했다. 당장 모든 호출부에서 넘기지 않는 값이면 선택 속성으로 두고, 함수 안에서 기본값을 처리하면 쓰기 편하기 때문이다.

연어버스 프런트엔드에 Sentry 기반 에러 트래킹을 도입하던 [PR #92](https://github.com/woowacourse-teams/2026-salmonbus/pull/92)에서도 처음에는 비슷한 시각으로 출발했다. 하지만 동료 레스와 코드 리뷰를 주고받으며 “호출부에서 명시적으로 적게 할까, 생략하게 둘까”라는 질문은 곧 **“같은 값이 어떤 함수에서는 필수이고, 다른 함수에서는 선택이어도 될까?”**라는 질문으로 바뀌었다.

이 글은 요청 옵션 타입의 선택 속성을 두고 고민하다가, 값을 누락했을 때 실제로 깨지는 것이 무엇인지 찾아내고 계층마다 다른 함수의 책임을 타입에 반영하기까지의 기록이다.

## 세 계층으로 이어지는 API 요청 구조

연어버스의 프런트엔드 요청 흐름은 세 단계로 이어진다.

```text
화면 또는 폴링 훅 (RouteSelectPage, VerdictBoardPage, useLiveVehicles)
  → 기능별 API 함수 (fetchRoutes, fetchBoard, fetchLiveVehicles)
    → 공통 HTTP 클라이언트 (requestJson)
```

공통 클라이언트의 `requestJson`은 모든 API 요청이 거치는 함수다. 당시 에러 트래킹 PR에서 논의 대상이 된 `RequestOptions`는 다음 두 가지 선택 속성을 갖고 있었다.

```ts
interface RequestOptions {
  signal?: AbortSignal;
  errorContext?: RequestErrorContext;
}
```

`signal`은 사용자가 화면을 떠나거나 새로운 폴링 요청이 이전 요청을 대체했을 때 진행 중인 요청을 취소하기 위해 전달한다. `errorContext`는 Sentry에서 같은 원인의 오류를 한 이슈로 묶을 수 있도록 정규화된 `endpoint`(예: `/api/v1/routes/:routeId/board`)와 태그(`route_id`)를 전달하는 속성이다.

`requestJson`은 이 옵션을 받아 HTTP 요청을 전송하고, 외부 취소 신호(`signal`)가 있으면 연결하며, 10초 내부 타임아웃을 적용한다. 그리고 응답과 실패를 `ApiResult` 타입으로 분류한 뒤 실패 시 등록된 콜백으로 보고한다. 외부에서 `signal`을 넘기지 않아도 내부 10초 타임아웃과 응답 분류, 실패 보고는 그대로 동작한다.

## 출발점: 선택 속성 대신 `null`을 명시하게 하면 어떨까?

같은 PR의 다른 리뷰에서 실패 보고 콜백에 전달하는 상세 정보 타입(`ApiFailureDetails`)을 먼저 수정했다. 원래는 여러 선택 속성이 흩어져 있었는데, 함께 전달되는 필드를 객체로 묶고 정보가 없을 때는 `null`로 표현하도록 바꿨다.

그 작업을 마치고 나니 `RequestOptions`에도 같은 방식을 적용하면 어떨지 궁금해졌다. [첫 코멘트](https://github.com/woowacourse-teams/2026-salmonbus/pull/92#discussion_r4230466807)에서 레스에게 다음과 같은 대안을 질문했다.

```ts
// 검토했던 대안 (최종 구현이 아님)
interface RequestOptions {
  signal: AbortSignal | null;
  errorContext: RequestErrorContext | null;
}
```

출발점은 “선택 속성은 나쁘다”가 아니라 호출부의 명시성과 편의 사이의 고민이었다.

선택 속성으로 두면 호출자가 의도적으로 생략한 것인지 실수로 빠뜨린 것인지 구분할 수 없다. 반대로 필수 속성에 `null`을 허용하면 호출자가 “이 옵션은 사용하지 않겠다”는 선택을 코드에 드러내게 된다. 다만 당시 대부분의 호출부는 `{ signal }`만 전달하고 있었고, `errorContext`는 일부 API 함수가 내부에서 채워주고 있었기 때문에 모든 호출부마다 `null`을 적어야 하는 부담이 생겼다. 그 명시성이 호출부의 부담을 감수할 만큼 이득인지 레스의 의견이 궁금했다.

## 값을 정하는 주체와 화면 보호 장치

레스와 의견을 나누며 가장 먼저 확인한 것은 **“그 값을 누가 정하는가”**였다. 어떤 값을 직접 결정하는 계층이라면 그 값을 알아야 하지만, 값을 결정하지 않는 계층이라면 그 존재를 몰라도 된다.

### `errorContext`는 화면 호출자가 정하는 값이 아니다

`errorContext`는 화면 컴포넌트가 아니라 기능별 API 함수가 내부에서 결정하는 값이었다.

```ts
export function fetchBoard(
  routeId: string,
  options?: RequestOptions,
): Promise<ApiResult<Board>> {
  const endpoint = ENDPOINTS.board;

  return requestJson<Board>(routeUrlFrom(endpoint, routeId), {
    ...options,
    errorContext: { endpoint, tags: { route_id: routeId } },
  });
}
```

실제 요청 URL에는 노선 ID가 포함되므로, 오류를 같은 원인끼리 묶으려면 정규화된 `endpoint`와 `route_id` 태그를 함께 전달해야 한다. 이 정보는 엔드포인트 구조를 아는 API 함수가 직접 만든다. 화면 호출자가 `errorContext`를 전달하더라도 `fetchBoard` 내부에서 덮어쓰고 있었고, 노선 목록을 불러오는 `fetchRoutes`에는 아예 필요하지 않은 값이었다.

따라서 화면 호출부에서까지 `errorContext`를 필수로 요구하면 화면이 직접 정하지 않는 오류 수집 규칙을 알아야 하고, 실제로는 쓰이지 않는 값을 매번 전달해야 하며, 공통 옵션의 변화가 화면 호출부까지 그대로 전파된다. 게다가 `RequestOptions`의 선택 속성은 `requestJson` 내부에서 `options.errorContext ?? null`로 기본값을 채워 에러 수집기에 넘기고 있었기 때문에, 읽는 쪽의 일관성을 위해 호출부에 `null`을 강제할 이유도 없었다.

### 처음에는 `signal`도 선택으로 두어도 괜찮아 보였다

그렇다면 화면이 직접 정하는 값인 `signal`은 어떨까?

프로젝트에는 이미 요청의 최신 여부를 확인하는 장치가 있었다. 새 요청이 이전 요청을 대체하거나 화면이 정리될 때 요청을 취소할 뿐 아니라, 늦게 도착한 응답이 최신 요청의 결과인지(`isLatest()`)도 검사한다. 따라서 누군가 실수로 `signal`을 빠뜨려 요청이 취소되지 않더라도, 뒤늦게 도착한 오래된 응답은 버려져 화면에 잘못 반영되지 않는다.

레스는 처음에 이 점을 근거로 화면에서 `signal`을 빠뜨려도 화면에 틀린 내용이 그려지지 않으니 `signal` 역시 선택 속성으로 유지해도 된다고 보았다.

## 판단의 전환: 응답을 버리는 것과 요청을 취소하는 것은 다르다

레스의 답변을 읽고 정리하던 중 한 가지 문제를 발견했다. **응답을 버리는 것과 요청을 취소하는 것은 다르다**는 점이었다.

화면에서 `signal`을 빠뜨리면 사용자가 화면을 떠난 뒤에도 요청은 취소되지 않은 채 남을 수 있다. 그 요청이 완료되지 못하고 10초 내부 타임아웃에 도달하면, 최신 요청인지 검사해 응답을 버리는 단계까지 가기도 전에 `requestJson`이 Sentry로 타임아웃 오류를 보고한다.

물론 사용자가 보지 않은 오류라고 해서 모두 가짜 오류라고 일반화할 수는 없다. 타임아웃 자체는 실제로 발생한 실패다. 진짜 문제는 **화면 수명에 맞춰 취소하려던 설계 의도 및 관측 기준과 어긋나는 요청이 살아남아 실패로 수집된다는 것**이었다.

화면 상태가 잘못 바뀌지 않더라도, 화면을 떠나며 취소됐어야 할 요청이 10초 뒤 에러로 기록되면 오류 수집 결과를 신뢰하기 어렵다. 이번 사례에서는 화면의 정확성뿐 아니라 오류 수집 결과도 필수 여부를 판단하는 근거가 되어야 했다. 이 반론을 계기로 레스도 판단을 수정했고, `signal`은 `errorContext`와 분리해 필수로 강제하기로 의견이 모였다.

## `signal`을 어느 경계에서 필수로 만들 것인가

화면에서 API 함수를 호출할 때 `signal` 누락을 타입으로 막기로 하자, 다음 쟁점이 떠올랐다. **`signal`을 공통 `requestJson`부터 필수로 만들 것인가, 아니면 기능별 API 함수에서만 필수로 만들 것인가?**

두 대안 모두 나름의 근거가 있었다([관련 논의](https://github.com/woowacourse-teams/2026-salmonbus/pull/92#discussion_r4244916130)).

공통 `requestJson`부터 `signal`을 필수로 받으면(대안 A), 모든 요청이 통과하는 한곳에서 누락을 막을 수 있다. 나중에 새로운 API 함수가 추가되더라도 공통 클라이언트의 타입이 제약을 걸어주므로 안전하다. 또한 `requestJson`은 실패를 보고할 때 `options.signal?.reason`을 읽어 취소 사유를 전달하기 때문에 `signal`과 아예 무관한 함수도 아니었다.

반면 기능별 API 함수에서만 `signal`을 필수로 받고 `requestJson`에서는 선택으로 유지하는 쪽(대안 B)은 각 계층의 책임에 집중한다.

고민 끝에 우리는 **대안 B**를 선택했다. `requestJson`이 왜 존재하는 함수인지에서 다시 출발해 보니 판단이 명확해졌다.

첫째, 화면 수명에 맞춰 요청을 취소해야 한다는 요구는 화면과 기능별 API 함수의 경계에 있다. 어떤 요청이 특정 화면의 수명에 묶여 있는지는 화면과 API 호출단이 아는 맥락이지, 공통 HTTP 클라이언트가 알아야 할 맥락이 아니다.

둘째, `requestJson`은 외부 `signal`이 없어도 자기 역할을 끝까지 수행할 수 있다. 요청을 전송하고, 내부 10초 타임아웃을 적용하고, 응답과 실패를 `ApiResult`로 분류해 보고하는 일은 외부 취소 신호 없이도 온전히 성립한다. 외부에서 취소한 적 없는 요청이 타임아웃으로 실패했을 때 이를 실패로 보고하는 것 역시 사실에 맞다.

셋째, 함수가 어떤 값을 받아 사용한다는 사실과 그 값 없이는 자기 역할을 수행할 수 없다는 주장은 다르다. 브라우저의 `fetch`가 `signal`을 받아 요청을 취소할 수 있지만 `signal`을 선택 옵션으로 두는 것과 같은 이치다.

물론 이 선택에도 비용은 있다. 같은 `signal`의 필수 여부가 계층마다 달라지므로 코드를 읽을 때 두 계약을 각각 확인해야 하고, 새 API 함수를 만들 때 화면용 옵션 계약을 따르지 않으면 누락을 놓칠 가능성이 남는다. 한곳에서 모든 누락을 막는 이점을 포기하는 대신 각 함수의 책임을 과장하지 않는 쪽을 택했기 때문에, 만약 향후 새 API 함수에서 `signal` 누락이 반복되거나 화면에서 `requestJson`을 직접 부르는 일이 생긴다면 그때 제약을 `requestJson`으로 옮기기로 했다.

## 최종 구현: `Required<Pick<RequestOptions, "signal">>`

[최종 반영 내용](https://github.com/woowacourse-teams/2026-salmonbus/pull/92#discussion_r4245823294)([커밋 `4a8c3fa`](https://github.com/woowacourse-teams/2026-salmonbus/commit/4a8c3fa557d848119077f85993d7ea7b666e4329))에서는 API 호출단에서 사용할 `FetchRequestOption` 타입을 별도로 정의했다.

```ts
type FetchRequestOption = Required<Pick<RequestOptions, "signal">>;

export function fetchRoutes(
  options: FetchRequestOption,
): Promise<ApiResult<RouteListResponse>> {
  return requestJson<RouteListResponse>("/api/v1/routes", options);
}
```

이제 `fetchRoutes()`처럼 옵션 없이 호출하면 컴파일 타임에 타입 오류가 발생한다. 화면 호출부에는 취소가 필요 없는 요청이 없었으므로 처음 검토했던 `AbortSignal | null` 대신 `AbortSignal`만 허용했고, `requestJson`의 `signal`은 선택 속성으로 남겨두었다. `errorContext`는 필요한 API 함수(`fetchBoard`, `fetchLiveVehicles`)가 내부에서 만들어 전달한다.

타입을 선언할 때 `RequestOptions` 전체에 `Required`를 적용하지 않고 `Pick`과 `Required`를 조합한 것도 계층 간 경계를 지키기 위해서였다. `Required<RequestOptions>`를 쓰면 화면이 정하지 않는 `errorContext`까지 필수가 되고, 반대로 `RequestOptions`를 확장해 `signal`만 필수로 바꾸면 API 함수가 내부에서 덮어쓰는 `errorContext`를 여전히 화면 호출부에서 넘길 수 있게 된다. 필요한 `signal`만 `Pick`으로 가져와 필수로 만들면 화면 호출자의 계약을 정확히 좁힐 수 있고, 이후 `RequestOptions`에 다른 공통 옵션이 추가되어도 API 함수의 호출 계약이 자동으로 넓어지는 것을 피할 수 있다.

실제로 이후 프로젝트에 인증 관련 API가 추가되면서 `RequestOptions`에는 `method`, `body`, `headers` 같은 속성이 늘어났고, 화면 수명과 무관하게 `requestJson`을 호출하는 코드도 생겼다. 당시 `requestJson`의 `signal`을 선택으로 유지하고 화면용 API 함수의 옵션을 `Pick`으로 좁혀둔 덕분에, 공통 클라이언트가 확장될 때 기존 API 함수들의 호출 계약은 영향을 받지 않았다.

## 마치며

처음에 던졌던 질문으로 돌아가 보자. **같은 값이 어떤 함수에서는 필수이고, 다른 함수에서는 선택이어도 될까?**

선택 속성을 필수로 바꿀지 고민하기 전에는 `?`를 호출부를 편하게 해주는 문법 장치로만 생각했다. 하지만 실제 리뷰를 거치며 확인한 것은, 어떤 속성의 필수 여부는 값 자체에 고정된 성질이 아니라 **그 값을 받는 함수가 맡은 책임과 경계에 따라 결정된다**는 점이었다.

`signal`은 화면과 API 함수의 경계에서는 빠뜨리는 순간 의도하지 않은 타임아웃 오류를 Sentry에 남기므로 필수여야 했다. 반면 공통 클라이언트인 `requestJson`의 경계에서는 외부 취소 신호가 없어도 요청 전송과 타임아웃, 결과 분류라는 자기 역할이 온전히 성립하므로 선택이어야 했다.

결국 옵션 타입의 물음표 하나를 지우는 일은 호출부의 편의를 조정하는 작업이 아니라, “누가 이 값을 결정하는가”, “빠뜨렸을 때 무엇이 깨지는가”, 그리고 “이 값이 없어도 함수의 역할이 성립하는가”를 코드의 경계마다 다시 묻는 과정이었다.
