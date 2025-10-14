# 검색 품질 향상시키기

> **더 나은 검색 경험을 구축한 기록**

## 대상 독자

검색 결과의 품질을 향상시키고 싶은 사람
검색 시 원하는 결과가 상위에 노출되지 않아 고민인 서버 개발자

## 서비스 배경

모아온은 프로젝트라는 맥락 안에서의 기술적 인사이트를 제공하는 서비스입니다. 따라서 핵심 페르소나는 기술적 인사이트를 얻고싶어 하는 뷰어였습니다. 뷰어가 원하는 정보를 찾기 위해서는 검색, 필터, 정렬 등의 기능이 핵심이었습니다.

또한 사용성 테스트에서도 절반에 가까운 사용자는 원하는 정보를 얻기위해 **검색**을 가장 먼저 수행했습니다.

결국 검색 기능은 모아온의 핵심 기능이며, 서비스의 가치에 막대한 영향을 미치는 중요한 기능입니다.

### 기존 검색 시스템: MySQL Full-Text Search

개발 초기에는 단순히 MySQL의 Like 기능을 활용했습니다. 입력한 검색어와 완전히 일치하는 것만 검색되는 것이 옳다고 생각했습니다.

하지만 사용성 테스트에서, 그리고 실제로 사용해 보면서 한계에 직면했습니다.
Like의 “완전 일치”로 인해 검색에 실패하는 경우가 많았습니다. “스프링 CORS”라고 검색했을 때, 스프링 환경에서 CORS를 다루는 글은 검색되지 않았습니다. 완전 일치가 아니기 때문입니다.

그래서 MySQL FullText Search 기능을 활용했습니다. 2-gram parser를 이용해 단어를 2글자씩 잘라 색인했습니다. 덕분에 Like보다 유연한 검색이 가능했습니다. 또 매칭되는 토큰의 수에 비례해 점수화하는 기능 덕분에 정확도가 있었습니다.

### Full-Text Search의 문제

불만족스러운 검색 결과를 마주치는 상황이 많았습니다.

1.관련 없는 문서가 많이 검색됩니다.

![aws-mysql.png](aws-mysql.png)

![aws-mysql2.png](aws-mysql2.png)

“aws”에 대한 검색 결과입니다.

2-gram 파서를 사용중이므로 “aws”를 “aw”, “ws”로 자르게 됩니다. 그리고 토큰의 매칭 빈도만으로 점수를 매깁니다. 첫 번째 게시글의 경우 아래 사진과 같이 “ws”가 많이 매칭되어서 최고 점수를 받았음을 알 수 있습니다.

![aws-mysql3.png](aws-mysql3.png)

2-gram을 사용하지 않고 유의미한 토큰을 추출하기 위해서는 형태소 분석이 필요했습니다.
“로드밸런서”와 같이 복합명사들을 쪼개지 않도록 사전도 필요했습니다.

2.세밀한 스코어링이 불가능합니다.

Full-Text Search는 검색어의 빈도만으로 점수를 매깁니다.

![https-mysql.png](https-mysql.png)

![https-mysql2.png](https-mysql2.png)

“HTTPS 적용”에 대한 검색 결과입니다.
첫 번째를 비롯한 다른 주제의 글들이 상위에 있는 이유는 “HTTPS”라는 단어 자체가 많이 등장했기 때문입니다.

1위 문서인 소셜 로그인 설계 아티클의 경우 API의 URL에 https://가 많아서 1등을 했습니다. 즉 단어의 빈도만으로는 아티클의 관련도가 높다고 볼 수 없습니다.

3.같거나 유사한 의미의 다른 단어로 검색 시 검색되지 않습니다.

많은 사람들이 “리팩터링”과 “리팩토링”을 구분 없이 사용합니다.
둘은 같은 의미임에도 불구하고 검색 결과는 달랐습니다.

![refactoring1.png](refactoring1.png)

![refactoring2.png](refactoring2.png)

“리팩터링”에 대한 결과(위)와 “리팩토링”에 대한 결과(아래)입니다.
전혀 다른 결과가 검색됨을 알 수 있습니다.

## 검색 품질을 결정짓는 세 가지 축

검색 품질은 단순히 “검색 결과가 나온다”로 판단할 수 없습니다.
사용자는 검색창에 단어를 입력하는 것이 아니라, **의도**를 전달합니다.
따라서 검색 품질은 아래 세 가지 요소에 의해 결정됩니다.

1. **형태소 분석** – 사용자의 단어를 기계가 이해 가능한 언어로 해석  
2. **사전(Synonym / Domain Dictionary)** – 서로 다른 단어를 같은 개념으로 연결  
3. **스코어링(Scoring)** – 정말 중요한 문서를 위쪽에 노출

### 형태소 분석

사용자의 검색 의도는 **단어, 품사, 의미에 존재합니다.**
예를들어 “리팩터링을 진행합니다.”라는 문장에서 “리팩터링”을 추출해야합니다.

형태소 분석은 검색엔진이 단어의 **의미 단위**를 인식하게 만듭니다.  
이 순간부터 “검색 실패”가 “검색 부족”으로 바뀌기 시작합니다.

### 사용자 사전

형태소 분석이 복합 명사를 분리하기도 합니다.
예를들어 “로그아웃”은 하나의 의미를 가지지만, “로그”와 “아웃”으로 분리되면 그 의미가 사라집니다.

따라서 형태소 분석 과정에서 “로그아웃”을 분리하지 않고 하나의 단어로 인식할 필요가 있습니다. 사전에 “로그아웃”을 정의함으로써 분리되지 않는 하나의 단어로 취급할 수 있습니다.

### 동의어 사전

사용자는 같은 의미의 단어를 제각기 사용합니다.

| 사용자 입력 | 실제 문서 표현 |
|-------------|----------------|
| 리팩토링 | 리팩터링 |
| docker | 도커 |
| CI/CD | 배포 자동화 |

형태소 분석만으로는 이것을 연결하지 못합니다.  
여기서 필요한 것이 **동의어 사전(Synonyms Dictionary)**입니다.

동의어 사전은 다음 두 가지를 해결합니다.

- **동일 개념어 통합** → “리팩토링” = “리팩터링”
- **도메인 용어 인식** → “스프링 CORS” = “Spring CORS 설정”

### 스코어링

앞의 예시로 볼 수 있듯이 단어가 많이 나온 문서가 사용자가 찾는 문서는 아닙니다.

간단한 예시로 아래와 같이 점수를 평가할 수 있습니다.
- 검색어가 제목에 있는 경우 높은 점수
- 검색어가 여러 개인 경우 문서에서 검색어들이 가까이 있을 수록 높은 점수
- “설계”, “구현”과 같은 상대적으로 비결정적인 흔한 단어는 낮은 점수

인기도나 최신성을 반영할 수도 있지만 정확도가 최우선 과제라 생략합니다.

## 적용해보기

구현은 ElasticSearch를 이용했습니다.

```json
{
  "analysis": {
    "analyzer": {
      "article_common_analyzer": {
        "filter": [
          "nori_pos_with_stoptags",
          "nori_readingform",
          "lowercase",
          "article_common_synonym"
        ],
        "tokenizer": "nori_tokenizer_with_dict",
        "type": "custom"
      }
    },
    "filter": {
      "article_common_synonym": {
        "synonyms_path": "synonyms.txt",
        "type": "synonym_graph"
      },
      "nori_pos_with_stoptags": {
        "stoptags": [
          "EC",
          "EF",
          "EP",
          "ETM",
          "ETN",
          "IC",
          "JC",
          "JKB",
          "JKC",
          "JKG",
          "JKO",
          "JKQ",
          "JKS",
          "JKV",
          "JX",
          "MAG",
          "MAJ",
          "MM",
          "SP",
          "SSC",
          "SSO",
          "SC",
          "SE",
          "XPN",
          "XSA",
          "XSN",
          "XSV",
          "UNA",
          "NA",
          "VSV",
          "VX"
        ],
        "type": "nori_part_of_speech"
      }
    },
    "tokenizer": {
      "nori_tokenizer_with_dict": {
        "decompound_mode": "mixed",
        "discard_punctuation": "false",
        "type": "nori_tokenizer",
        "user_dictionary": "dictionary.txt"
      }
    }
  }
}

```

### 형태소 분석

[nori 형태소 분석기](https://esbook.kimjmin.net/06-text-analysis/6.7-stemming/6.7.2-nori)를 커스텀하여 사용했습니다.
어미, 조사, 감탄사 등 핵심과 무관한 단어를 색인하지 않도록 했습니다.

예를들어 "Flyway는 오픈소스 마이그레이션 툴이다" 라는 문장은 다음과 같이 분석됩니다.
```json
{
  "tokens": [
    {
      "token": "flyway",
      "start_offset": 0,
      "end_offset": 6,
      "type": "word",
      "position": 0
    },
    {
      "token": "오픈",
      "start_offset": 8,
      "end_offset": 10,
      "type": "word",
      "position": 3
    },
    {
      "token": "소스",
      "start_offset": 10,
      "end_offset": 12,
      "type": "word",
      "position": 4
    },
    {
      "token": "마이",
      "start_offset": 13,
      "end_offset": 15,
      "type": "word",
      "position": 6
    },
    {
      "token": "그레이",
      "start_offset": 15,
      "end_offset": 18,
      "type": "word",
      "position": 7
    },
    {
      "token": "션",
      "start_offset": 18,
      "end_offset": 19,
      "type": "word",
      "position": 8
    },
    {
      "token": "툴",
      "start_offset": 20,
      "end_offset": 21,
      "type": "word",
      "position": 10
    }
  ]
}
```

### 사용자 사전

위 예시에서는 "오픈소스"를 "오픈", "소스"로, "마이그레이션"을 "마이", "그레이", "션"으로 쪼개고 있습니다.

"마이그레이션"은 그 자체로 의미를 가지는 하나의 토큰이 되어야합니다. "마이", "그레이", "션"은 아무 의미를 갖지 않기 때문입니다.

이렇게 형태소 분석 시 단어를 쪼개지 않고 유지할 수 있도록 미리 사전을 통해 정의할 수 있습니다.

```text
// dictionary
오픈소스
마이그레이션
...
```

사전 정의 이후 "Flyway는 오픈소스 마이그레이션 툴이다" 라는 문장은 다음과 같이 분석됩니다.
```json
{
  "tokens": [
    {
      "token": "flyway",
      "start_offset": 0,
      "end_offset": 6,
      "type": "word",
      "position": 0
    },
    {
      "token": "오픈소스",
      "start_offset": 8,
      "end_offset": 12,
      "type": "word",
      "position": 3
    },
    {
      "token": "마이그레이션",
      "start_offset": 13,
      "end_offset": 19,
      "type": "word",
      "position": 4
    },
    {
      "token": "툴",
      "start_offset": 20,
      "end_offset": 21,
      "type": "word",
      "position": 6
    }
  ]
}

```

### 동의어 사전

언어에는 수많은 동의어들이 존재할 수 있습니다. 모아온의 경우 **개발**이라는 도메인 용어에서는 다음과 같은 예시가 있습니다.
- 데이터베이스, DB
- 로드밸런서, 로드밸런싱, 로드밸런스

또한 같은 단어를 한글로 검색할 수도 있고 영어로 검색할 수도 있습니다. 
"마이그레이션" 대신 "migration"으로 검색해도 "마이그레이션" 글이 포함되어야 합니다.

```text
// synonyms
마이그레이션, migration
로드밸런서, 로드밸런싱, 로드밸런스
...
```

동의어 사전 정의 이후 "Flyway는 오픈소스 마이그레이션 툴이다" 라는 문장은 다음과 같이 분석됩니다.

```json
{
  "tokens": [
    {
      "token": "flyway",
      "start_offset": 0,
      "end_offset": 6,
      "type": "word",
      "position": 0
    },
    {
      "token": "오픈소스",
      "start_offset": 8,
      "end_offset": 12,
      "type": "word",
      "position": 3
    },
    {
      "token": "migration",
      "start_offset": 13,
      "end_offset": 19,
      "type": "SYNONYM",
      "position": 4
    },
    {
      "token": "마이그레이션",
      "start_offset": 13,
      "end_offset": 19,
      "type": "word",
      "position": 4
    },
    {
      "token": "툴",
      "start_offset": 20,
      "end_offset": 21,
      "type": "word",
      "position": 6
    }
  ]
}

```

### 스코어링

검색 의도에 맞는 문서가 상위에 있을 수록 검색 엔진의 성능이 좋다고 할 수 있습니다.
그래서 단어의 빈도가 아닌 문서의 가치를 점수로 매겨보려 시도했습니다.

문서의 가치를 점수화하는 것은 상당히 어려웠습니다. "가치"라는 것이 너무 추상적이기 때문입니다.
혼자 또는 팀 단위로 문서의 관련도를 매긴다 해도, 집단의 크기가 너무 적었습니다. 즉 직감에 의존하는 점수화가 되어버렸습니다.

따라서 당장은 알고리즘을 수정하지 않았습니다.
다만 검색어가 제목에 있으면 더 높은 점수를 부여하고 있습니다. 제목은 글이 다루는 핵심 내용에 대한 소개이기 때문에 핵심을 포함할 확률이 높다고 생각했습니다.

## 마무리 : 앞으로의 과제

처음에는 검색을 단순한 “기능”으로 생각했습니다.   
그러나 사용자가 기대하는 것은 도구가 아니라 **의도 파악**이었습니다.

> “내가 찾고 싶은 것을, 제대로 찾아줄 수 있는가?”  
> “이 서비스는 나를 이해하는가?”

- 좋은 검색은 **사용자의 만족**으로 완성된다.
- 검색은 문자열 비교가 아니라 **의미 해석**이다.

이 글에서는 검색 품질이라는 추상적인 목표를 위해 기본적인 기반을 마련했습니다.

객관적인 품질 개선을 위해 앞으로는 이러한 작업을 해나가려 합니다.

- 클릭률, 체류 시간, 스크롤 횟수 등 사용자 행동 데이터 수집
- 통계를 바탕으로 품질 지표 측정 및 개선
- 검색어 자동 완성 및 오탈자 교정 등으로 UX 개선

## 참고

- [MySQL Full-Text Search](https://dev.mysql.com/doc/refman/8.4/en/fulltext-search.html)
- [Elasticsearch](https://www.elastic.co/kr/elasticsearch)
