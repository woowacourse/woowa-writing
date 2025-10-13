# 좋은 API 설계하기: 적절한 추상화 레벨 찾기

**예상 독자:** 백엔드 개발 경험이 있으면서 API 설계 원칙을 고민해본 주니어 개발자, 팀 프로젝트에서 API를 설계해야 하는 개발자

---

## 시작: 팀프로젝트를 하던 중 의문점이 생기다

개발을 하다 보면 누구나 "좋은 코드"에 대해 고민합니다. 변수 이름은 어떻게 지을까, 함수는 어디까지 분리할까, 클래스의 책임은 무엇일까. 이런 질문들 덕분에 우리는 클린 아키텍처를 배우고, 디자인 패턴을 공부합니다.

하지만 한 가지 놓치기 쉬운 부분이 있습니다. 바로 **API 설계**입니다.

API는 우리가 짜는 코드만큼 중요한데, 마치 당연히 해야 하는 일처럼 취급되곤 합니다. 그래서 많은 개발자는 API를 설계할 때 명확한 기준 없이 직관에만 의존하게 됩니다.

### 찝찝함의 시작

팀 프로젝트를 진행하던 중 프론트엔드 개발자로부터 이런 요청을 받았습니다.

> "여러 번 요청 보내면 네트워크만 여러 번 타니까, 그냥 다 한 번에 보내줘"

당시에는 이 요청이 합리적으로 들렸습니다. 실제로 네트워크 왕복 횟수를 줄이는 것은 성능 최적화니까요. 그래서 API를 그대로 수정했습니다.

하지만 요청을 들었을 때 마음 한구석에 찝찝함이 있었는데, 정확히 무엇이 불편한지 설명할 수 없었습니다. 그 이유는 간단했습니다. 저 자신도 "좋은 API"에 대한 명확한 기준을 가지지 못하고 있었던 것입니다.

### 그 후의 탐색

그 이후로 여러 사례를 살펴보며 이 찝찝함의 정체를 파악하기 시작했습니다. 그렇게 다양한 사례를 공부하며 점점 명확해진 것이 있습니다.

결론은 이것입니다: **좋은 API는 적절한 추상화 레벨로 설계된 API이다.**

이 말이 다소 추상적으로 들릴 수 있으니, 구체적인 사례와 함께 풀어서 설명하겠습니다.

---

## API의 본질: 인터페이스로 보기

API(Application Programming Interface)는 **시스템 간의 정보를 교환할 수 있도록 공유되는 경계면**입니다. 여기서 "인터페이스"라는 단어에 집중해야 합니다.

우리가 만드는 서비스들은 여러 개의 API들이 레고처럼 조립되어서 하나의 서비스의 기능들을 만들고 있습니다. 즉, API라는 경계면을 어떻게 설정했느냐에 따라 이 조립 과정이 얼마나 쉽고 유연한지가 결정됩니다.

**핵심 질문:** API를 어느 정도로 "추상화"해야 클라이언트는 쉽게 사용하고, 서버는 유지보수하기 쉽게 만들 수 있을까요?

---

## 사례 1: GitHub의 Pull Request API - 리소스별 추상화

GitHub의 PR 기능을 만드는 상황을 상상해봅시다. 사용자가 PR을 만들 때 일반적인 워크플로우는 다음과 같습니다:

1. 제목을 적고
2. 작업한 브랜치와 병합 대상 브랜치를 설정하고
3. 리뷰어를 배정하고
4. 라벨을 지정하고
5. (필요에 따라) 다른 설정들을 조정

### ❌ 나쁜 설계: 워크플로우를 그대로 API로

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

클라이언트 입장에서는 굉장히 사용하기 쉬운 API입니다. 요청 한 번만 보내면 끝나니까요.

**하지만 문제가 발생합니다:**

새로운 요구사항이 나타날 때마다 API가 깨집니다. 예를 들어:

- **Draft PR이 추가되는 경우:** 필드를 하나 더 추가해야 합니다.
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

- **자동 머지 기능을 추가하고 싶은 경우:** 또 다른 필드를 추가하거나, 새로운 API를 만들어야 합니다.

이렇게 필드가 계속 추가되다 보면:

1. **기존 기능도 문제가 생깁니다** - 새 요구사항을 개발하면서 기존의 PR 기능이 깨질 가능성이 높아집니다.
2. **백엔드에 중복 코드가 쌓입니다** - 요구사항마다 새로운 PR 생성 API를 만든다면, 비슷한 로직이 여러 곳에서 반복됩니다.
3. **클라이언트는 혼란스러워집니다** - 어떤 API를 써야 할지, 어떤 필드가 필요한지 알기 어려워집니다.

### ✅ 좋은 설계: 조합 가능한 작은 단위로 분리

다른 접근을 생각해봅시다. PR을 하나의 리소스로 보되, **클라이언트에서 내부 구현은 알 수 없도록 적절히 추상화**합니다.

PR을 생성할 때 변하지 않는 요소가 무엇일까요?
- 제목
- 작업 브랜치 (head)
- 병합 대상 브랜치 (base)

이 세 가지는 PR의 핵심이고, 아마 앞으로도 변하지 않을 것 같습니다. 그럼 PR 생성 API는 이렇게 만들 수 있습니다:

```json
POST /repos/{owner}/{repo}/pulls
{
  "title": "[2단계 - 리팩터링] 미미(홍향미) 미션 제출합니다.",
  "head": "step2",
  "base": "main"
}
```

이제 리뷰어, 라벨 같은 것들은 별도의 리소스로 처리합니다.

**리뷰어 할당은 리뷰어란 리소스를 생성하는 것:**
```json
POST /repos/{owner}/{repo}/pulls/{pull_number}/requested_reviewers
{
  "reviewers": ["user1", "user2"]
}
```

**라벨 추가도 마찬가지로 별도의 리소스 생성:**
```json
POST /repos/{owner}/{repo}/issues/{issue_number}/labels
{
  "labels": ["enhancement", "urgent"]
}
```

**PR 머지는 PR의 상태를 변경하는 것:**
```json
PUT /repos/{owner}/{repo}/pulls/{pull_number}/merge
{
  "merge_method": "squash"
}
```

### 이 설계가 왜 더 나을까?

**1. 새로운 요구사항이 기존 API를 깨지 않습니다**

Draft PR 기능이 추가된다면? PR 생성 API에 `draft` 파라미터만 추가되고, 리뷰어 할당 API는 그대로입니다.

```json
POST /repos/{owner}/{repo}/pulls
{
  "title": "...",
  "head": "...",
  "base": "...",
  "draft": true  // 간단하게 추가됨
}
```

**2. 새로운 기능도 기존 API 패턴을 따릅니다**

자동 머지 기능이 필요하다면? 기존 `/merge` URL과 별도로 `/auto_merge`라는 새로운 URL을 가진 API를 만들면 됩니다.

```json
PUT /repos/{owner}/{repo}/pulls/{pull_number}/auto_merge
{
  "merge_method": "squash"
}
```

기존 머지 로직은 그대로 유지되고, 신규 기능은 별도 리소스로 분리됩니다.

**3. 워크플로우가 바뀌어도 대처 가능합니다**

만약 사용자의 워크플로우가 이렇게 바뀐다면?
- PR 생성 → 나중에 리뷰어 배정 → 그 다음 라벨 지정

기존 API들로 완벽하게 대처할 수 있습니다. 각 단계가 독립적인 리소스 조작이기 때문입니다.

---

## 사례 2: Stripe의 결제 API 진화 - 추상화의 재설계

Stripe는 시간이 지나면서 요구사항이 급격히 늘어나는 상황에서 어떻게 API를 재설계했는지 보여주는 최고의 사례입니다.

### Phase 1 (2010년대 초): 카드 결제만 필요

처음에는 결제 수단이 카드 하나뿐이었습니다. Stripe의 초기 API는 이렇게 단순했습니다:

```json
curl https://api.stripe.com/v1/charges \
  -u sk_test_xxx: \
  -d amount=2000 \
  -d currency=usd \
  -d source=token \
  -d description="Charge for order 1234"
```

프로세스는 간단했습니다:
1. 클라이언트에서 카드 정보를 Stripe 서버로 보냅니다
2. Stripe는 카드 정보를 암호화한 Token을 생성해 반환합니다
3. 자신의 서버는 이 Token과 결제 정보로 Stripe API에 결제를 요청합니다
4. 결제가 즉시 처리되고 Charge 객체가 반환됩니다

이 시점의 추상화 레벨은 **매우 적절했습니다**. 카드 정보를 '토큰'이란 리소스로 분리하고, 결제는 결제와 관련된 정보만 사용하도록 깔끔하게 설계되었기 때문입니다.

### Phase 2 (2015년): 비트코인 결제 추가 - 문제 발생

2015년, Stripe는 비트코인 결제를 지원해야 했습니다. 하지만 **비트코인은 카드와 완전히 다른 특성이 있습니다:**

카드 결제:
- 결제 요청 → 즉시 처리 → 완료

비트코인 결제:
- 결제 요청 → 고객이 특정 주소로 송금 → 블록체인 네트워크에서 충분히 확인되어야 → 완료
- 고객이 해줘야 하는 과정이 하나 더 생깁니다
- 블록체인 네트워크의 확인을 무한정으로 기다릴 수 없습니다

이 문제를 해결하기 위해 Stripe는 BitcoinReceiver라는 새로운 리소스를 만들었습니다:

```json
POST /receivers
{
  "type": "bitcoin",
  "amount": 2000,
  "currency": "usd"
}
```

응답으로 특정 비트코인 주소를 담은 Receiver 객체를 생성해 반환했습니다:

```json
{
  "id": "receiver_xxx",
  "type": "bitcoin",
  "address": "1A1z7agoat...",
  "amount": 2000,
  "currency": "usd",
  "filled": false
}
```

사용자가 이 주소로 송금하면, Stripe 서버에서는 Receiver의 상태를 `filled = true`로 변경합니다. 그동안 서버는 계속해서 Stripe 서버에 "filled가 됐나?"라고 물어봅니다.

감지되면, `source = BitcoinReceiver.id`를 사용해서 기존의 Charge API를 호출하면 됩니다.

```json
POST /v1/charges
{
  "amount": 2000,
  "currency": "usd",
  "source": "receiver_xxx"
}
```

**하지만 이 방식도 문제가 있었습니다:**

1. **클라이언트는 결제 수단마다 다른 API를 알아야 합니다** - 카드면 토큰을 미리 생성하고, 비트코인이면 Receiver를 생성해야 합니다.
2. **서버는 결제 수단의 특성을 일일이 관리해야 합니다** - Receiver라면 상태를 폴링해야 하고, 카드라면 바로 처리하고...
3. **간편 결제가 계속 나타나면서 복잡도가 증가합니다** - 간편 결제들도 비트코인처럼 사전에 송금 정보를 받거나, 리다이렉트가 필요했습니다.

결제 수단이 다양해질수록, 각 수단마다 특성을 개발자가 잘 알아야 했고, 이는 **API를 사용하는 개발자 경험을 심각하게 해쳤습니다.**

### Phase 3 (2017년 이후): 완전한 재설계 - PaymentIntent

결제 수단이 계속 다양해지자, Stripe는 근본적인 재설계를 결심합니다.

**목표:** 어떤 결제 수단이든 상관없이, **동일한 API 플로우로 처리**하기

Stripe가 발견한 공통 플로우를 쉽게 풀면 이렇습니다:

```
나 얘네 "결제할 의도"가 있어 
  → OK, 접수. 근데 결제 뭘로 할건데?

나 이런 "결제 수단" 쓸거임 
  → OK, 접수. 이거 니 "지갑 id"임. 결제 확실히 할거면 얘 들고 와

나 "결제 결심함" 해줘 
  → OK, 완!
```

이 플로우를 API로 구현했습니다:
- **결제할 의도** → `PaymentIntent`
- **결제 수단** → `PaymentMethod`
- **결제 확정** → `PaymentIntent/confirm`

#### Step 1: PaymentIntent 생성 - 결제 의도 선언

서버에서 결제 요청을 받으면, Stripe 서버로 PaymentIntent 생성을 요청합니다:

```json
POST /v1/payment_intents
{
  "amount": 2000,
  "currency": "usd",
  "description": "iphone 17 pro max"
}
```

Stripe는 이렇게 응답합니다:

```json
{
  "id": "pi_1Nz9a123...",
  "object": "payment_intent",
  "amount": 2000,
  "currency": "usd",
  "status": "requires_payment_method",
  "client_secret": "pi_1Nz9a123_secret_..."
}
```

`status`가 `requires_payment_method`라는 것은 "결제 수단을 알려줄 때까지 기다리고 있어"라는 뜻입니다.

#### Step 2: PaymentMethod 생성 - 결제 수단 등록

이제 뭘로 결제할지를 등록합니다. 여기서의 핵심은 **결제 수단이 무엇이든 같은 API를 사용한다**는 것입니다.

**카드 결제:**
```json
POST /v1/payment_methods
{
  "type": "card",
  "card": {
    "number": "4242424242424242",
    "exp_month": 12,
    "exp_year": 2030,
    "cvc": "123"
  }
}
```

**간편 결제 (MimiPay):**
```json
POST /v1/payment_methods
{
  "type": "mimipay"
}
```

응답도 동일한 구조입니다:

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
  "id": "pm_mimipay_123",
  "object": "payment_method",
  "type": "mimipay"
}
```

#### Step 3: PaymentIntent/confirm - 결제 확정

이제 "진짜 결제할거야"라고 확정합니다. 결제 수단이 뭐든 같은 API입니다:

```json
POST /v1/payment_intents/pi_1Nz9a123/confirm
{
  "payment_method": "pm_card_123"
}
```

또는:

```json
POST /v1/payment_intents/pi_1Nz9a123/confirm
{
  "payment_method": "pm_mimipay_123"
}
```

### 응답의 차이: 상태에 따른 다음 액션 지시

여기서 중요한 것은 **응답입니다.**

**즉시 완료되는 결제 (카드):**
```json
{
  "id": "pi_1Nz9a123",
  "status": "succeeded"
}
```

**다음 액션이 필요한 결제 (간편 결제):**
```json
{
  "id": "pi_1Nz9a123",
  "status": "requires_action",
  "next_action": {
    "type": "redirect_to_url",
    "redirect_to_url": {
      "url": "https://checkout.mimipay.com/..."
    }
  }
}
```

클라이언트는 이 응답을 보고:
- `status`가 `succeeded`면 바로 결제 완료 로직을 수행
- `status`가 `requires_action`이면 사용자를 `next_action`에 명시된 URL로 리다이렉트

**서버도 더 똑똑해집니다:**

결제가 완료되는 동안 서버는 계속 Stripe에 "완료됐나?"라고 물어보지 않습니다. 대신 Stripe에서 `payment_intent.succeeded` Webhook을 보내줄 때까지 기다립니다:

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

이 Webhook은 **모든 결제 수단에서 동일합니다.** 카드든 비트코인이든 간편 결제든 상관없이, 결제가 완료되면 같은 형태의 Webhook이 옵니다.

### 이 재설계가 얼마나 강력한가?

**1. 새로운 결제 수단이 추가되어도 기존 코드가 변하지 않습니다**

비트코인을 추가한다면?
```json
POST /v1/payment_methods
{
  "type": "bitcoin"
}
```

클라이언트는 기존 플로우를 그대로 따릅니다. 서버도 여전히 같은 Webhook을 기다립니다.

**2. 클라이언트 경험이 일관됩니다**

개발자는 결제 수단의 복잡한 특성을 알 필요가 없습니다. 간단한 플로우를 따르면 됩니다:
- PaymentIntent 생성
- PaymentMethod 생성
- confirm 호출
- next_action 확인

**3. 서버 유지보수가 간단합니다**

각 결제 수단마다 특성을 관리할 필요가 없습니다. PaymentIntent와 PaymentMethod라는 추상화된 리소스가 모든 복잡성을 감춥니다.

---

## 좋은 API 설계의 원칙

지금까지 본 두 가지 사례에서 도출할 수 있는 원칙들을 정리하면:

### 원칙 1: 추상화는 이해관계자의 편의성을 기준으로

API를 설계할 때는 이해관계자들을 생각해야 합니다:

**클라이언트 개발자:**
- 일관된, 예측 가능한 플로우를 원합니다
- 각 기능의 복잡한 특성을 일일이 알고 싶지 않습니다

**서버 개발자:**
- 유지보수하기 쉬운 코드 구조를 원합니다
- 도메인별로 책임이 명확하게 나뉘길 원합니다

**미래의 요구사항:**
- 새로운 기능이 추가될 때 기존 API가 깨지지 않아야 합니다
- 확장 가능한 구조로 설계되어야 합니다

### 원칙 2: 변하지 않는 것과 변할 것을 구분

GitHub 사례에서:
- **변하지 않는 것:** PR의 제목, head 브랜치, base 브랜치
- **변할 수 있는 것:** 리뷰어, 라벨, 머지 방식, 자동 머지 등

Stripe 사례에서:
- **변하지 않는 것:** 결제 금액, 화폐 단위
- **변할 수 있는 것:** 결제 수단의 종류, 각 수단의 특성

**변할 수 있는 것들을 별도 리소스로 분리**하면, 새로운 요구사항이 기존 API를 깨지 않습니다.

### 원칙 3: 공통 플로우를 찾으면 단순해집니다

Stripe의 PaymentIntent 설계는 이것을 잘 보여줍니다.

처음에는 각 결제 수단마다 다른 전처리 과정이 필요했습니다. 하지만 요구사항을 면밀히 분석하면:
- 모든 결제는 "의도 선언" → "수단 선택" → "확정"의 3단계를 거칩니다

이 **공통 플로우를 API에 반영**하면 간단해집니다.

### 원칙 4: 추상화는 한 번에 정해지지 않습니다

Stripe도 처음부터 PaymentIntent를 설계한 것이 아닙니다:
- 2010년대 초: 카드만 지원 (단순한 설계)
- 2015년: 비트코인 추가 (BitcoinReceiver 추가)
- 2017년 이후: 완전히 새로운 추상화 (PaymentIntent)

중요한 것은 **요구사항 변화에 적응할 수 있는 구조로 설계**하는 것입니다. 초기 설계가 모든 미래를 예측할 수 없다는 것을 인정하고, 그래도 깨지지 않는 API를 만드는 것이 목표입니다.

### 원칙 5: 과도한 추상화도 피하기

너무 잘게 쪼개면 오버 엔지니어링이 됩니다. 

좋은 질문들:
- 이 추상화가 정말 필요한가? 아니면 미래의 가정일 뿐인가?
- 이 리소스는 정말 독립적인 관심사인가?
- 클라이언트가 이 플로우를 이해하고 따를 수 있는가?

---

## 결론

API는 단순한 엔드포인트가 아닙니다. **팀 전체의 생산성과 코드의 유지보수성을 좌우하는 중요한 설계 결정**입니다.

**잘 설계된 API의 특징:**
- 클라이언트가 일관된 플로우로 사용할 수 있다
- 새로운 요구사항이 기존 API를 깨지 않는다
- 도메인별로 책임이 명확하게 분리되어 있다
- 개발자가 내부 복잡성을 신경 쓰지 않아도 된다

**체크리스트:**
- [ ] 변하지 않는 것과 변할 것을 구분했는가?
- [ ] 각 리소스의 책임이 명확한가?
- [ ] 새로운 요구사항이 기존 API에 미치는 영향을 생각해봤는가?
- [ ] 클라이언트가 따를 수 있는 일관된 플로우가 있는가?
- [ ] 팀 모두가 이 설계의 의도를 이해하는가?

**마지막이 가장 중요합니다.** API를 설계할 때 이해관계자들과 **적절한 추상화 레벨에 대해 함께 논의하는 과정 자체**가 좋은 API를 만드는 시작입니다.

우리가 클린 아키텍처와 디자인 패턴을 배워 좋은 코드를 작성하듯이, 오늘 살펴본 GitHub과 Stripe의 사례들이 여러분의 API 설계에 조금이나마 도움이 되기를 바랍니다.

---

## 참고자료

- [GitHub REST API Documentation](https://docs.github.com/en/rest)
- [GitHub API - Pulls (Pull Requests)](https://docs.github.com/en/rest/pulls)
- [Stripe Payment Intents API](https://stripe.com/docs/payments/payment-intents)
- [Stripe API Evolution - From Charges to Payment Intents](https://stripe.com/blog)
- [REST API Design Best Practices](https://restfulapi.net/)
