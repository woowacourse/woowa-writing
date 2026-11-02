# Jev 소개: 코드가 바로 쓰는 판단 모델

LLM에게 분류, 라우팅, 점수 매기기를 맡겨 왔다면 Jev로 같은 판단을 더 빠르고 안정적으로 처리할 수 있어요. Jev는 문장을 생성하지 않고, 미리 정해 둔 선택지 중 하나와 그 확률을 반환하는 의사결정 모델이에요. 그래서 JSON 파싱이나 형식 검증 없이 응답을 바로 `if` 문에 넣을 수 있어요.

이 문서를 읽으면 Jev가 LLM과 어떻게 다른지 이해하고, 지금 LLM에 맡긴 판단 중 어떤 것을 Jev로 옮길 수 있는지 판단할 수 있어요.

::: info 이 문서의 기준

2026년 11월 기준 정보예요. Jev는 얼리 액세스 단계라서 가격, 모델 버전, API가 바뀔 수 있어요. 최신 정보는 [Jev 공식 문서](https://docs.typesafe.ai/introduction)를 확인하세요.

:::

## LLM으로 판단할 때 생기는 문제

고객 문의를 담당 팀에 배정하는 기능을 LLM으로 만든다고 해볼게요. 보통 다음처럼 프롬프트로 JSON 응답을 요청하고, 받은 문자열을 파싱해서 검증해요.

```python
import json
from openai import OpenAI

client = OpenAI()
ticket = "My running shoes arrived in the wrong size. Can I swap them for a size 10?"

completion = client.chat.completions.create(
    model="gpt-4o-mini",  # 사용 중인 모델로 바꾸세요.
    response_format={"type": "json_object"},
    messages=[{
        "role": "user",
        "content": (
            'Which team should handle this ticket? Answer in JSON: '
            '{"department": "returns" | "shipping" | "billing", "confidence": 0.0-1.0}\n\n'
            f"Ticket: {ticket}"
        ),
    }],
)

result = json.loads(completion.choices[0].message.content)
department = result.get("department")

if department not in {"returns", "shipping", "billing"}:
    ...  # 선택지에 없는 값이 오면 다시 요청하거나 기본값으로 처리해야 해요.
```

이 방식에는 네 가지 문제가 있어요.

- **응답을 파싱하고 검증해야 해요.** LLM은 문자열을 생성하기 때문에 선택지에 없는 값이나 깨진 JSON이 올 수 있어요. 이런 응답을 처리하는 코드가 판단 로직보다 길어지기도 해요.
- **LLM이 알려주는 신뢰도를 믿기 어려워요.** 프롬프트로 신뢰도를 요청해도 LLM은 실제보다 높은 값을 답하는 경향이 있고, 같은 입력에도 매번 다른 값을 답해요. 틀릴 수 있는 판단을 걸러내는 기준으로 쓰기 어려워요.
- **응답이 느려요.** 토큰을 하나씩 생성하기 때문에 판단 한 번에 몇 초가 걸릴 수 있어요. 요청 경로 중간에 넣으면 전체 응답 시간이 길어져요.
- **비용이 커요.** 판단 하나에 짧은 답만 필요해도 생성 모델 요금을 그대로 내야 해요.

Jev는 이 문제를 해결하려고 만든 모델이에요.

## Jev란

Jev는 TypeSafe AI가 2026년 9월에 공개한 모델로, TypeSafe AI는 이런 모델을 **System One 모델**이라고 불러요. System One 모델은 소프트웨어가 바로 쓸 수 있는 판단을 빠르게 내리도록 만든 모델이에요. 이름은 빠르고 직관적인 사고를 뜻하는 심리학 용어 '시스템 1'에서 따왔어요.

Jev에는 두 가지를 보내요.

- **상태(state)**: 판단할 대상이에요. 문자열이나 JSON 객체를 보낼 수 있어요. 예를 들면 고객 문의 내용이나 주문 정보예요.
- **질문(question)**: 상태에 대해 알고 싶은 것과 답의 선택지예요. 한 요청에 여러 개를 보낼 수 있어요.

Jev는 질문마다 선택지별 확률을 계산해서 반환해요. 응답은 항상 개발자가 정한 선택지 안에서만 나오기 때문에 형식 오류가 생기지 않아요.

```
 상태: 고객 문의 내용          질문: 담당 팀, 긴급 여부
            │                         │
            └────────────┬────────────┘
                         ▼  요청 1번
               ┌───────────────────┐
               │        Jev        │
               │ 질문별로 병렬 판단 │
               └───────────────────┘
                         ▼  응답 1번
          선택지 + 확률 + 신뢰도(질문 유형에 따라 다름)
                         ▼
               코드에서 분기, 정렬, 라우팅
```

### 질문 유형

질문은 답의 형태에 따라 세 가지 유형으로 만들어요.

| 유형 | 언제 쓰나요 | 반환값 | 예시 |
|---|---|---|---|
| Choice | 순서 없는 선택지 중 하나를 고를 때 | `choice`, `probabilities`, `confidence` | 문의를 담당할 팀 |
| Score | 순서가 있는 단계 중 위치를 정할 때 | `score`, `probabilities`, `confidence` | 고객의 불만 정도 |
| Noul | 참인지 거짓인지 판단할 때 | `noul`(참일 확률, 0~1) | 긴급한 문의인지 여부 |

Score의 `score`는 단계 사이의 값이 될 수도 있어요. 예를 들어 0단계와 1단계 사이의 상태라면 0.6처럼 반환돼요.

### 질문은 서로 독립적으로 처리돼요

한 요청에 담긴 질문은 모두 병렬로 처리되고, 한 질문의 답이 다른 질문에 영향을 주지 않아요. 그래서 질문을 추가해도 응답 시간이 거의 늘지 않아요.

반대로 한 질문이 다른 질문의 답에 따라 달라져야 한다면 요청을 두 번으로 나눠야 해요. 첫 번째 응답을 받은 뒤, 그 답을 두 번째 요청의 상태에 넣어서 보내세요.

## LLM과 Jev 비교

| 항목 | LLM | Jev |
|---|---|---|
| 출력 | 생성한 문자열 | 정해 둔 선택지와 확률 |
| 파싱과 검증 | 필요해요 | 필요 없어요 |
| 선택지 밖의 값 | 나올 수 있어요 | 나오지 않아요 |
| 신뢰도 | 프롬프트로 요청해야 하고, 실제보다 높게 답하는 경향이 있어요 | 모든 Choice와 Score 응답에 포함돼요 |
| 응답 시간 | 수 초에서 수십 초 | 대부분 100ms 안팎 |
| 입력 토큰 가격 | 100만 토큰당 $0.20~$10 | 100만 토큰당 $0.042 |
| 잘하는 일 | 글쓰기, 요약, 코드 생성, 대화 | 분류, 라우팅, 점수 매기기, 참/거짓 판단 |

응답 시간과 가격은 TypeSafe AI가 발표한 수치예요. 회사가 직접 만든 워크플로로 측정한 결과라서 실제 서비스에서는 차이가 줄어들 수 있어요. 도입하기 전에 내 데이터로 직접 측정해 보세요.

::: warning Jev도 틀릴 수 있어요

Jev는 선택지에 없는 값을 만들어 내지 않지만, 틀린 선택지를 고를 수는 있어요. 그래서 다음 섹션의 신뢰도를 기준으로 자동 처리할 범위를 정해야 해요.

:::

## 예제: 고객 문의를 담당 팀에 배정하기

앞의 LLM 예제와 같은 기능을 Jev로 만들어 볼게요. Python SDK(`typesafe-sdk`)를 사용하고, API 키는 `TYPESAFE_API_KEY` 환경 변수에서 읽어요. 설치 방법은 [Jev 퀵스타트](https://docs.typesafe.ai/introduction/quickstart)를 참고하세요.

```python
from typesafe_sdk import Choice, Noul, TypeSafeClient

ticket = "My running shoes arrived in the wrong size. Can I swap them for a size 10?"

with TypeSafeClient(model="jev-latest") as client:
    response = client.system_one(
        state=ticket,
        questions={
            # 담당 팀: 세 팀 중 하나를 골라요.
            "department": Choice(
                instructions="Which team should handle this?",
                criteria={
                    "returns": "Exchanges, wrong or damaged items",
                    "shipping": "Delivery status, delays, lost packages",
                    "billing": "Charges, invoices, payment problems",
                },
            ),
            # 긴급 여부: 참일 확률을 받아요.
            "is_urgent": Noul(
                instructions="The message conveys urgency or time-sensitivity",
            ),
        },
    )

    department = response.answers["department"]
    print(department.choice)      # returns
    print(department.confidence)  # 1.0
    print(response.answers["is_urgent"].noul)
```

`criteria`의 키는 선택지 이름이고, 값은 선택지를 구분하는 설명이에요. Jev는 선택지 이름과 설명을 모두 보고 판단해요. 반면 `department`, `is_urgent` 같은 질문 ID는 코드에서 답을 찾는 데만 쓰이고 Jev에는 전달되지 않아요. 그래서 질문 내용은 `instructions`에 빠짐없이 써야 해요.

Choice 질문의 응답은 다음과 같아요.

```json
{
  "type": "choice",
  "choice": "returns",
  "confidence": 1.0,
  "probabilities": { "shipping": 0.0, "returns": 1.0, "billing": 0.0 }
}
```

LLM 예제와 달리 `json.loads()`도, 선택지 검증 코드도 없어요. `choice`는 항상 `returns`, `shipping`, `billing` 중 하나예요.

## 신뢰도로 자동 처리할 범위 정하기

신뢰도(`confidence`)는 확률이 한 선택지에 얼마나 몰려 있는지를 0~1 사이 값으로 나타내요. 확률이 한 선택지에 모두 몰려 있으면 1이고, 모든 선택지에 똑같이 나뉘어 있으면 0이에요. Choice와 Score 응답에만 포함되고, Noul은 `noul` 값 자체가 확률이라서 0.5에 가까울수록 불확실하다는 뜻이에요.

신뢰도를 기준으로 판단을 세 가지로 나눠서 처리하세요.

- **신뢰도가 높을 때**: 사람의 확인 없이 자동으로 처리해요.
- **신뢰도가 중간일 때**: 사용자에게 확인을 받거나 검토 대상으로 표시해요.
- **신뢰도가 낮을 때**: 자동으로 처리하지 않고 담당자에게 넘겨요.

기준값은 작업이 잘못됐을 때 생기는 피해에 맞춰 정해요. 다음은 음성 뱅킹 서비스에서 사용자의 의도를 판단한 뒤, 작업마다 다른 기준을 적용하는 예시예요. `intent`는 `check_balance`(잔액 조회), `approve_transfer`(송금 승인) 같은 선택지 중 사용자의 의도를 고르는 Choice 질문이에요. 질문을 정의하는 코드는 앞의 예제와 같은 방식이라서 생략했어요.

```python
action = response.answers["intent"]

if action.confidence < 0.6:
    # 어떤 작업이든 신뢰도가 0.6보다 낮으면 상담원에게 넘겨요.
    route_to_support_agent(account_id)

elif action.choice == "check_balance":
    # 잔액 조회는 잘못돼도 피해가 작아서 0.6 이상이면 처리해요.
    show_balance(account_id)

elif action.choice == "approve_transfer":
    if action.confidence > 0.85:
        # 송금 승인은 신뢰도가 0.85보다 높을 때만 자동으로 처리해요.
        approve_transfer(account_id)
    else:
        # 그 사이라면 사용자에게 한 번 더 확인해요.
        ask_user_to_confirm(account_id)
```

예시의 0.6과 0.85는 출발점일 뿐이에요. 보수적인 값으로 시작하고, 내 데이터로 결과를 확인하면서 조정하세요.

## 질문을 작게 나누기

Jev는 해당 분야를 잘 아는 사람이 몇 초 안에 내릴 수 있는 판단에 맞춰져 있어요. "이 메시지가 긴급한가요?"는 좋은 질문이지만, "이 메시지를 분석해서 가장 좋은 대응 방법을 정해 주세요"는 Jev에 맞지 않아요.

여러 요소를 따져야 하는 판단이라면 요소마다 질문을 따로 만들고, 결과는 코드에서 합치세요. 예를 들어 스타트업 사업 계획서를 평가한다면 "이 계획서를 평가해 주세요" 대신 시장 규모, 기술 실현 가능성, 차별성을 각각 Score 질문으로 만들어요. 그리고 세 점수에 가중치를 곱해서 최종 점수를 계산해요. 평가 기준이 바뀌면 프롬프트를 다시 쓰는 대신 코드의 가중치만 바꾸면 돼요.

## Jev에 맞는 작업과 맞지 않는 작업

지금 LLM에 맡긴 작업 중 응답이 정해진 선택지 중 하나라면 Jev로 옮길 수 있어요.

**Jev에 맞는 작업**

- 문의, 요청, 문서를 정해진 카테고리로 분류하기
- 요청을 처리할 담당자나 기능으로 라우팅하기
- 긴급도, 품질, 위험도처럼 단계를 정의할 수 있는 기준으로 점수 매기기
- 문서나 메시지가 특정 조건을 만족하는지 확인하기
- LLM 앱의 입력과 출력을 검사하는 가드레일 만들기

**Jev에 맞지 않는 작업**

- 글쓰기, 요약, 번역, 코드 생성처럼 문장을 만들어야 하는 작업
- 여러 단계를 거쳐 추론해야 하는 판단. 작은 질문으로 나눌 수 있다면 Jev를 쓸 수 있어요.
- 코딩 에이전트나 챗봇의 모델 바꾸기. Jev는 대화하거나 도구를 호출하지 않아요.

## 다음 단계

- [퀵스타트](https://docs.typesafe.ai/introduction/quickstart): API 키를 발급받고 첫 요청을 보내요.
- [질문 유형](https://docs.typesafe.ai/primitives): Choice, Score, Noul을 정의하는 방법과 고르는 기준을 알아봐요.
- [신뢰도](https://docs.typesafe.ai/confidence): 신뢰도 계산 방식과 활용법을 자세히 알아봐요.
- [패턴](https://docs.typesafe.ai/patterns): 신뢰도 기반 라우팅, 복합 점수처럼 Jev로 시스템을 만드는 방법을 알아봐요.
- [플레이그라운드](https://console.typesafe.ai/playground): 코드를 작성하기 전에 브라우저에서 Jev를 직접 써 봐요.
