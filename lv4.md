# 좋은 API 설계하기: 적절한 추상화 레벨 찾기

**예상 독자:** 백엔드 개발 경험이 있으면서 API 설계 원칙을 고민해본 주니어 개발자, 팀 프로젝트에서 API를 설계해야 하는 개발자

---

## 시작: 팀 프로젝트 중 마주친 질문

개발을 하다 보면 누구나 "좋은 코드"에 대해 고민합니다. 변수 이름은 어떻게 지을까, 함수는 어디까지 분리할까, 클래스의 책임은 무엇일까. 이런 질문들 덕분에 우리는 클린 아키텍처를 배우고, 디자인 패턴을 공부합니다.

하지만 한 가지 놓치기 쉬운 부분이 있습니다. 바로 **API 설계**입니다.

API는 우리가 작성하는 코드만큼 중요하지만, 마치 당연히 해야 하는 일처럼 취급되곤 합니다. 그래서 많은 개발자가 API를 설계할 때 명확한 기준 없이 직관에만 의존하게 됩니다.

### 찝찝함의 시작

팀 프로젝트를 진행하던 중 프론트엔드 개발자로부터 이런 요청을 받았습니다.

> "여러 번 요청 보내면 네트워크를 여러 번 타니까, 그냥 다 한 번에 보내줘."

당시에는 이 요청이 합리적으로 들렸습니다. 실제로 네트워크 왕복 횟수를 줄이는 것은 성능 최적화이기 때문입니다. 그래서 API를 그대로 수정했습니다.

하지만 요청을 들었을 때 마음 한구석에 찝찝함이 있었는데, 정확히 무엇이 불편한지 설명할 수 없었습니다. 그 이유는 간단했습니다. 저 자신도 "좋은 API"에 대한 명확한 기준을 가지지 못하고 있었던 것입니다.

### 그 후의 탐색

그 이후로 여러 사례를 살펴보며 이 찝찝함의 정체를 파악하기 시작했습니다. 다양한 사례를 공부하며 점점 명확해진 것이 있습니다.

결론은 이것입니다: **좋은 API는 적절한 추상화 레벨로 설계된 API입니다.**

이 말이 다소 추상적으로 들릴 수 있으니, 구체적인 사례와 함께 풀어서 설명하겠습니다.

---

## API의 본질: 인터페이스로 보기

API(Application Programming Interface)는 **시스템 간의 정보를 교환할 수 있도록 공유되는 경계면**입니다. 여기서 "인터페이스"라는 단어에 집중해야 합니다.

우리가 만드는 서비스는 여러 API가 레고처럼 조립되어 하나의 기능을 구성합니다. 즉, API라는 경계면을 어떻게 설정했는지에 따라 이 조립 과정이 얼마나 쉽고 유연한지가 결정됩니다.

**핵심 질문:** API를 어느 정도로 "추상화"해야 클라이언트는 쉽게 사용하고, 서버는 유지보수하기 쉽게 만들 수 있을까요?

---

## 사례 1: GitHub의 Pull Request API - 리소스별 추상화

GitHub의 PR 기능을 만드는 상황을 상상해봅시다. 사용자가 PR을 생성할 때 일반적인 워크플로우는 다음과 같습니다:

1. 제목을 작성한다
2. 작업한 브랜치와 병합 대상 브랜치를 설정한다
3. 리뷰어를 배정한다
4. 라벨을 지정한다
5. (필요에 따라) 다른 설정들을 조정한다

### ❌ 나쁜 설계: 워크플로우를 그대로 API로 만들기

이 워크플로우를 그대로 API에 반영하면 이런 스펙이 나올 수 있습니다:

```json
POST /pull-request
{
  "title": "[2단계 - 리팩터링] 미미(홍향미) 미션 제출합니다.",
  "head": "step2",
  "base": "main",
  "reviewers": ["mint", "norang"],
  "labels": ["step2"]
}
```

클라이언트 입장에서는 사용하기 쉬운 API처럼 보입니다. 요청 한 번만 보내면 끝나니까요.

**하지만 문제가 발생합니다:**

새로운 요구사항이 나타날 때마다 API 스펙이 변경됩니다. 예를 들어:

- **Draft PR 기능이 추가되는 경우:** 필드를 하나 더 추가해야 합니다.
  ```json
  {
    "title": "...",
    "head": "...",
    "base": "...",
    "reviewers": [...],
    "labels": [...],
    "draft": true  // 새로운 필드
  }
  ```

- **자동 병합 기능을 추가하는 경우:** 또 다른 필드를 추가하거나, 새로운 API를 만들어야 합니다.

이렇게 필드가 계속 추가되면:

1. **기존 기능도 영향을 받습니다** - 새 요구사항을 개발하면서 기존 PR 생성 기능이 깨질 가능성이 높아집니다.
2. **백엔드에 중복 코드가 쌓입니다** - 요구사항마다 새로운 PR 생성 API를 만든다면, 비슷한 로직이 여러 곳에서 반복됩니다.
3. **클라이언트가 혼란스러워합니다** - 어떤 API를 사용해야 할지, 어떤 필드가 필요한지 파악하기 어려워집니다.

### ✅ 좋은 설계: 조합 가능한 작은 단위로 분리하기

다른 접근을 생각해봅시다. PR을 하나의 **리소스(Resource)**로 보되, **클라이언트가 내부 구현을 알 수 없도록 적절히 추상화**합니다.

> **리소스(Resource)란?** REST API 설계에서 중요한 개념으로, 서버가 관리하는 모든 것을 의미합니다. 예를 들어 사용자(User), 게시글(Post), Pull Request 등이 모두 리소스입니다.

PR을 생성할 때 변하지 않는 핵심 요소가 무엇일까요?
- 제목
- 작업 브랜치 (head)
- 병합 대상 브랜치 (base)

이 세 가지는 PR의 본질이며, 앞으로도 변하지 않을 가능성이 높습니다. 그렇다면 PR 생성 API는 이렇게 만들 수 있습니다:

```json
POST /repos/{owner}/{repo}/pulls
{
  "title": "[2단계 - 리팩터링] 미미(홍향미) 미션 제출합니다.",
  "head": "step2",
  "base": "main"
}
```

이제 리뷰어, 라벨과 같은 부가 기능은 별도의 리소스로 처리합니다.

**리뷰어 할당은 별도 리소스 생성:**
```json
POST /repos/{owner}/{repo}/pulls/{pull_number}/requested_reviewers
{
  "reviewers": ["user1", "user2"]
}
```

**라벨 추가도 별도 리소스 생성:**
```json
POST /repos/{owner}/{repo}/issues/{issue_number}/labels
{
  "labels": ["enhancement", "urgent"]
}
```

**PR 병합은 PR의 상태 변경:**
```json
PUT /repos/{owner}/{repo}/pulls/{pull_number}/merge
{
  "merge_method": "squash"
}
```

### 이 설계가 왜 더 나을까?

**1. 새로운 요구사항이 기존 API를 깨지 않습니다**

Draft PR 기능이 추가된다면? PR 생성 API에 `draft` 파라미터만 추가되고, 리뷰어 할당 API는 그대로 유지됩니다.

```json
POST /repos/{owner}/{repo}/pulls
{
  "title": "...",
  "head": "...",
  "base": "...",
  "draft": true  // 간단하게 추가
}
```

**2. 새로운 기능도 기존 API 패턴을 따릅니다**

자동 병합 기능이 필요하다면? 기존 `/merge` 엔드포인트와 유사하게 `/auto_merge` 엔드포인트를 만들면 됩니다.

```json
PUT /repos/{owner}/{repo}/pulls/{pull_number}/auto_merge
{
  "merge_method": "squash"
}
```

기존 병합 로직은 그대로 유지되고, 신규 기능은 별도 리소스로 분리됩니다.

**3. 워크플로우가 바뀌어도 대처 가능합니다**

만약 사용자의 워크플로우가 이렇게 바뀐다면?
- PR 생성 → 나중에 리뷰어 배정 → 그다음 라벨 지정

기존 API들로 완벽하게 대처할 수 있습니다. 각 단계가 독립적인 리소스 조작이기 때문입니다.

**핵심 원칙:** 변하지 않는 핵심 개념은 하나의 리소스로, 변할 수 있는 부가 기능은 별도 리소스로 분리합니다.

---

## 사례 2: Stripe의 결제 API 진화 - 추상화의 재설계

Stripe는 시간이 지나면서 요구사항이 급격히 증가하는 상황에서 API를 어떻게 재설계했는지 보여주는 훌륭한 사례입니다.

### Phase 1 (2010년대 초): 카드 결제만 필요한 시기

처음에는 결제 수단이 카드 하나뿐이었습니다. Stripe의 초기 API는 매우 단순했습니다:

```bash
curl https://api.stripe.com/v1/charges \
  -u sk_test_xxx: \
  -d amount=2000 \
  -d currency=usd \
  -d source=tok_visa \
  -d description="Charge for order 1234"
```

프로세스는 간단했습니다:
1. 클라이언트에서 카드 정보를 Stripe.js로 토큰화
2. Stripe는 카드 정보를 암호화한 Token을 생성해 반환
3. 서버는 이 Token으로 결제 요청
4. 결제가 즉시 처리되고 Charge 객체 반환

이 시점의 추상화 레벨은 **매우 적절했습니다**. 카드 정보를 '토큰'이라는 리소스로 분리하고, 결제는 결제 관련 정보만 사용하도록 깔끔하게 설계되었기 때문입니다.

### Phase 2 (2015년): 비트코인 결제 추가 - 문제 발생

2015년, Stripe는 비트코인 결제를 지원해야 했습니다. 하지만 **비트코인은 카드와 완전히 다른 특성**을 가지고 있었습니다:

**카드 결제:**
- 결제 요청 → 즉시 처리 → 완료

**비트코인 결제:**
- 결제 요청 → 사용자가 특정 주소로 송금 → 블록체인 네트워크 확인 → 완료
- 사용자가 직접 해야 하는 단계가 추가됨
- 블록체인 확인을 무한정 기다릴 수 없음

이 문제를 해결하기 위해 Stripe는 `BitcoinReceiver`라는 새로운 리소스를 만들었습니다:

```bash
POST /v1/bitcoin/receivers
{
  "amount": 2000,
  "currency": "usd",
  "email": "customer@example.com"
}
```

응답으로 비트코인 주소를 포함한 Receiver 객체가 생성됩니다:

```json
{
  "id": "btcrcv_xxx",
  "object": "bitcoin_receiver",
  "bitcoin_address": "1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa",
  "amount": 2000,
  "currency": "usd",
  "filled": false
}
```

사용자가 이 주소로 비트코인을 송금하면, Stripe 서버는 블록체인을 모니터링하다가 Receiver의 상태를 `filled: true`로 변경합니다.

이때 서버는 주기적으로 Stripe API를 호출하여 상태를 확인해야 합니다(**폴링, Polling**):

```bash
# 서버가 주기적으로 확인
GET /v1/bitcoin/receivers/btcrcv_xxx
```

상태가 `filled: true`로 변경되면, 이제 기존의 Charge API를 호출합니다:

```bash
POST /v1/charges
{
  "amount": 2000,
  "currency": "usd",
  "source": "btcrcv_xxx"
}
```

**하지만 이 방식에도 문제가 있었습니다:**

1. **클라이언트가 결제 수단마다 다른 플로우를 알아야 합니다** - 카드는 토큰을 미리 생성하고, 비트코인은 Receiver를 생성한 뒤 상태를 폴링해야 합니다.
2. **서버가 결제 수단의 특성을 모두 관리해야 합니다** - Receiver는 상태를 폴링해야 하고, 카드는 바로 처리하는 등 각 수단별 로직이 필요합니다.
3. **간편 결제가 추가되면서 복잡도가 증가합니다** - 각 간편 결제도 비트코인처럼 사용자 인증이나 리다이렉트가 필요했습니다.

결제 수단이 다양해질수록, 각 수단의 특성을 개발자가 모두 이해해야 했고, 이는 **API를 사용하는 개발자 경험을 심각하게 해쳤습니다.**

### Phase 3 (2017년 이후): 완전한 재설계 - PaymentIntent

결제 수단이 계속 다양해지자, Stripe는 근본적인 재설계를 결심합니다.

**목표:** 어떤 결제 수단이든, **동일한 API 플로우로 처리**하기

Stripe가 발견한 모든 결제의 공통 플로우:

```
1. 결제 의도 선언: "나 이 금액을 결제할 거야"
2. 결제 수단 선택: "이 방법으로 결제할 거야"
3. 결제 확정: "진짜 결제할게"
4. 추가 작업 수행: (필요시) "이 단계를 추가로 해줘"
```

이 플로우를 API로 구현했습니다:
- **결제 의도** → `PaymentIntent`
- **결제 수단** → `PaymentMethod`
- **결제 확정** → `PaymentIntent.confirm()`
- **추가 작업** → `next_action`

#### Step 1: PaymentIntent 생성 - 결제 의도 선언

서버에서 결제 요청을 받으면, Stripe에 PaymentIntent 생성을 요청합니다:

```bash
POST /v1/payment_intents
{
  "amount": 2000,
  "currency": "usd",
  "description": "iPhone 17 Pro Max"
}
```

Stripe 응답:

```json
{
  "id": "pi_1Nz9a123",
  "object": "payment_intent",
  "amount": 2000,
  "currency": "usd",
  "status": "requires_payment_method",
  "client_secret": "pi_1Nz9a123_secret_abc123"
}
```

`status: "requires_payment_method"`는 "결제 수단을 알려줄 때까지 대기 중"이라는 의미입니다.

#### Step 2: PaymentMethod 생성 - 결제 수단 등록

이제 무엇으로 결제할지를 등록합니다. **핵심은 결제 수단이 무엇이든 같은 API를 사용한다는 것입니다.**

보안을 위해 이 단계는 클라이언트에서 Stripe.js를 통해 직접 수행됩니다:

**카드 결제:**
```javascript
// 클라이언트 코드
const paymentMethod = await stripe.createPaymentMethod({
  type: 'card',
  card: cardElement,  // Stripe.js의 카드 입력 컴포넌트
});
```

**간편 결제 (예: Alipay):**
```javascript
const paymentMethod = await stripe.createPaymentMethod({
  type: 'alipay',
});
```

응답 구조도 동일합니다:

**카드 결제 응답:**
```json
{
  "id": "pm_card_123",
  "object": "payment_method",
  "type": "card",
  "card": { "brand": "visa", "last4": "4242" }
}
```

**간편 결제 응답:**
```json
{
  "id": "pm_alipay_456",
  "object": "payment_method",
  "type": "alipay"
}
```

#### Step 3: PaymentIntent.confirm() - 결제 확정

이제 서버 또는 클라이언트에서 "진짜 결제할 거야"라고 확정합니다. **결제 수단이 무엇이든 같은 API입니다:**

```bash
POST /v1/payment_intents/pi_1Nz9a123/confirm
{
  "payment_method": "pm_card_123"
}
```

또는:

```bash
POST /v1/payment_intents/pi_1Nz9a123/confirm
{
  "payment_method": "pm_alipay_456"
}
```

### 응답의 차이: 상태 기반 다음 액션

여기서 중요한 것은 **응답입니다.**

**즉시 완료되는 결제 (카드):**
```json
{
  "id": "pi_1Nz9a123",
  "status": "succeeded",
  "payment_method": "pm_card_123"
}
```

**추가 액션이 필요한 결제 (간편 결제):**
```json
{
  "id": "pi_1Nz9a123",
  "status": "requires_action",
  "next_action": {
    "type": "redirect_to_url",
    "redirect_to_url": {
      "url": "https://checkout.alipay.com/..."
    }
  }
}
```

클라이언트는 이 응답을 보고:
- `status: "succeeded"`면 바로 결제 완료 처리
- `status: "requires_action"`이면 사용자를 `next_action.redirect_to_url.url`로 리다이렉트

**서버는 더 이상 폴링하지 않습니다:**

결제가 완료될 때까지 서버가 반복적으로 상태를 확인하는 대신, Stripe가 **Webhook**을 통해 결과를 알려줍니다:

```json
{
  "type": "payment_intent.succeeded",
  "data": {
    "object": {
      "id": "pi_1Nz9a123",
      "status": "succeeded"
    }
  }
}
```

> **Webhook이란?** 서버가 특정 이벤트 발생 시 미리 등록된 URL로 자동으로 알림을 보내는 방식입니다. 클라이언트가 반복적으로 상태를 확인(폴링)할 필요가 없어집니다.

이 Webhook은 **모든 결제 수단에서 동일합니다.** 카드든 비트코인이든 간편 결제든, 결제가 완료되면 같은 형태의 Webhook이 전송됩니다.

### 이 재설계가 얼마나 강력한가?

**1. 새로운 결제 수단이 추가되어도 기존 코드가 변하지 않습니다**

비트코인을 추가한다면?
```javascript
const paymentMethod = await stripe.createPaymentMethod({
  type: 'bitcoin',
});
```

클라이언트는 기존 플로우를 그대로 따릅니다. 서버도 여전히 같은 Webhook을 기다립니다.

**2. 클라이언트 경험이 일관됩니다**

개발자는 결제 수단의 복잡한 특성을 알 필요가 없습니다. 단순한 플로우만 따르면 됩니다:
- PaymentIntent 생성
- PaymentMethod 생성
- confirm 호출
- status와 next_action 확인

**3. 서버 유지보수가 간단합니다**

각 결제 수단의 특성을 관리할 필요가 없습니다. PaymentIntent와 PaymentMethod라는 추상화된 리소스가 모든 복잡성을 캡슐화합니다.

**핵심 원칙:** 서로 다른 구현 세부사항을 가진 여러 기능이 있을 때, 공통 플로우를 찾아서 추상화하면 일관된 API를 만들 수 있습니다.

---

## 좋은 API 설계의 원칙

지금까지 살펴본 두 사례에서 도출할 수 있는 원칙들을 정리하겠습니다:

### 원칙 1: 추상화는 이해관계자의 편의성을 기준으로

API를 설계할 때는 이해관계자들을 고려해야 합니다:

**클라이언트 개발자:**
- 일관되고 예측 가능한 플로우를 원합니다
- 각 기능의 복잡한 특성을 일일이 알고 싶지 않습니다
- 최소한의 요청으로 작업을 완료하고 싶습니다

**서버 개발자:**
- 유지보수하기 쉬운 코드 구조를 원합니다
- 도메인별로 책임이 명확하게 나뉘길 원합니다
- 새로운 기능 추가 시 기존 코드를 건드리지 않고 싶습니다

**미래의 요구사항:**
- 새로운 기능이 추가될 때 기존 API가 깨지지 않아야 합니다
- 확장 가능한 구조로 설계되어야 합니다

### 원칙 2: 변하지 않는 것과 변할 것을 구분

GitHub 사례:
- **변하지 않는 것:** PR의 제목, head 브랜치, base 브랜치
- **변할 수 있는 것:** 리뷰어, 라벨, 병합 방식, 자동 병합 등

Stripe 사례:
- **변하지 않는 것:** 결제 금액, 화폐 단위, 결제 의도
- **변할 수 있는 것:** 결제 수단의 종류, 각 수단의 특성

**변할 수 있는 것들을 별도 리소스로 분리**하면, 새로운 요구사항이 기존 API를 깨지 않습니다.

### 원칙 3: 공통 플로우를 찾으면 단순해집니다

Stripe의 PaymentIntent 설계가 이를 잘 보여줍니다.

처음에는 각 결제 수단마다 다른 전처리 과정이 필요했습니다. 하지만 요구사항을 면밀히 분석하면:
- 모든 결제는 "의도 선언" → "수단 선택" → "확정"의 3단계를 거칩니다

이 **공통 플로우를 API에 반영**하면 클라이언트와 서버 모두 간단해집니다.

### 원칙 4: 추상화는 한 번에 정해지지 않습니다

Stripe도 처음부터 PaymentIntent를 설계한 것이 아닙니다:
- 2010년대 초: 카드만 지원 (단순한 설계)
- 2015년: 비트코인 추가 (BitcoinReceiver 추가)
- 2017년 이후: 완전히 새로운 추상화 (PaymentIntent)

중요한 것은 **요구사항 변화에 적응할 수 있는 구조로 설계**하는 것입니다. 초기 설계가 모든 미래를 예측할 수 없다는 것을 인정하고, 변화에 유연한 API를 만드는 것이 목표입니다.

### 원칙 5: 과도한 추상화도 피하기

너무 잘게 쪼개면 오버 엔지니어링이 됩니다.

좋은 질문들:
- 이 추상화가 정말 필요한가? 아니면 미래의 가정일 뿐인가?
- 이 리소스는 정말 독립적인 관심사인가?
- 클라이언트가 이 플로우를 이해하고 따를 수 있는가?
- 이 분리가 유지보수 비용을 정말로 줄여주는가?

---

## 결론

API는 단순한 엔드포인트가 아닙니다. **팀 전체의 생산성과 코드의 유지보수성을 결정하는 중요한 설계 결정**입니다.

**잘 설계된 API의 특징:**
- 클라이언트가 일관된 플로우로 사용할 수 있습니다
- 새로운 요구사항이 기존 API를 깨지 않습니다
- 도메인별로 책임이 명확하게 분리되어 있습니다
- 개발자가 내부 복잡성을 신경 쓰지 않아도 됩니다

**설계 시 체크리스트:**
- [ ] 변하지 않는 것과 변할 것을 구분했는가?
- [ ] 각 리소스의 책임이 명확한가?
- [ ] 새로운 요구사항이 기존 API에 미치는 영향을 고려했는가?
- [ ] 클라이언트가 따를 수 있는 일관된 플로우가 있는가?
- [ ] 팀 모두가 이 설계의 의도를 이해하는가?

**마지막이 가장 중요합니다.** API를 설계할 때 이해관계자들과 **적절한 추상화 레벨에 대해 함께 논의하는 과정 자체**가 좋은 API를 만드는 시작입니다.

우리가 클린 아키텍처와 디자인 패턴을 배워 좋은 코드를 작성하듯이, 오늘 살펴본 GitHub과 Stripe의 사례들이 여러분의 API 설계에 도움이 되기를 바랍니다.

---

## 참고자료

- [GitHub REST API Documentation](https://docs.github.com/en/rest)
- [GitHub API - Pulls (Pull Requests)](https://docs.github.com/en/rest/pulls)
- [Stripe Payment Intents API](https://stripe.com/docs/payments/payment-intents)
- [Stripe API Evolution](https://stripe.com/blog/payment-api-design)
- [REST API Design Best Practices](https://restfulapi.net/)
