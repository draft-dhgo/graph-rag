---
title: 로컬 검색
tags:
  - query
  - search
  - local
  - entity-centric
created: 2025-01-12
type: documentation
links:
  - [[Query Module]]
  - [[Global Search]]
  - [[Entity]]
  - [[Relationship]]
---

# 로컬 검색

**로컬 검색**은 지식 그래프에서 특정 엔티티와 그 인접 이웃에 초점을 맞춘 쿼리 메서드입니다. 특정 엔티티, 그들의 관계, 구체적인 세부 정보에 대한 질문에 가장 적합합니다.

## 🎯 사용 사례

로컬 검색은 다음에 탁월합니다:
- "X와 Y의 관계는 무엇인가?"
- "Y에 관여하는 핵심 인물은 누구인가?"
- "X에 대한 구체적인 세부 정보는 무엇인가?"
- "엔티티 X와 그 연결에 대해 알려달라"

## 🏗️ 작동 방식

```
쿼리 -> 엔티티 추출 -> 컨텍스트 빌딩 ->
관계 수집 -> 텍스트 유닛 검색 ->
LLM 생성 -> 응답
```

### 단계 1: 엔티티 추출

쿼리를 분석하여 관련 엔티티를 찾습니다:

1. **의미 유사도**: 쿼리를 임베딩하고 엔티티 임베딩과 비교
2. **오버샘플링**: 필요보다 더 많은 엔티티를 검색 (오버샘플링 계수)
3. **필터링**: 유사도 점수를 기반으로 상위 엔티티 선택

### 단계 2: 컨텍스트 빌딩

**로컬 컨텍스트 빌더**는 여러 소스에서 컨텍스트를 빌드합니다:

| 구성 요소 | 토큰 할당 | 목적 |
|-----------|-----------------|---------|
| 텍스트 유닛 | 50% | 증거가 포함된 소스 텍스트 |
| 커뮤니티 리포트 | 25% | 커뮤니티 레벨 컨텍스트 |
| 엔티티 및 관계 | 25% | 직접 그래프 연결 |

### 단계 3: 관계 우선순위 지정

관계는 우선순위 순서로 수집됩니다:

1. **네트워크 내**: 관련 엔티티 간의 양방향 연결
2. **네트워크 외**: 관련 엔티티에서의 단방향 연결
3. **단일**: 동일한 텍스트 유닛에 언급된 엔티티

### 단계 4: LLM 생성

최종 응답은 다음을 사용하여 생성됩니다:
- 원본 쿼리
- 빌드된 컨텍스트
- 선택적 대화 기록
- LLM을 위한 구조화된 프롬프트

## ⚙️ 구성

### YAML 구성

```yaml
local_search:
  # 컨텍스트 비율
  text_unit_prop: 0.5      # 텍스트 유닛에 50%
  community_prop: 0.25     # 커뮤니티 리포트에 25%

  # 엔티티 선택
  top_k_entities: 10      # 검색할 엔티티 수
  top_k_relationships: 10  # 엔티티당 관계 수

  # 컨텍스트 제한
  max_context_tokens: 12000  # 최대 컨텍스트 크기

  # 대화
  conversation_history_max_turns: 10

  # 검색 매개변수
  oversample: 2.0  # 엔티티 선택을 위한 오버샘플링 계수
```

### 프로그래밍 방식 사용

```python
from graphrag.api import local_search

result = await local_search(
    query="AI 연구에 누가 관여하고 있는가?",
    config="./settings.yaml",
    conversation_history=None  # 선택적: 이전 메시지 목록
)
```

## 🆚 다른 메서드와의 비교

| 측면 | 로컬 검색 | 글로벌 검색 | DRIFT 검색 |
|--------|-------------|---------------|--------------|
| **범위** | 2-3 홉 이웃 | 전체 그래프 | 반복적 확장 |
| **최적 용도** | 구체적인 질문 | 광범위한 주제 | 복잡한 multi-hop |
| **컨텍스트** | 엔티티 중심 | 커뮤니티 중심 | 동적 |
| **속도** | 빠름 | 중간 | 느림 |
| **토큰 사용량** | 중간 | 높음 | 높음 |

## 📊 컨텍스트 구조

### 예시 컨텍스트

```json
{
  "entities": [
    {
      "title": "Microsoft Research",
      "type": "organization",
      "description": "Microsoft의 연구 부서...",
      "rank": 1
    }
  ],
  "relationships": [
    {
      "source": "Microsoft Research",
      "target": "AI Research",
      "description": "연구 수행",
      "weight": 1.0
    }
  ],
  "community_reports": [
    {
      "community": "comm-001",
      "summary": "이 커뮤니티는 AI 연구에 집중합니다..."
    }
  ],
  "text_units": [
    {
      "text": "Microsoft Research는 다음과 같은 작업을 해왔습니다...",
      "tokens": 256
    }
  ]
}
```

## 🔧 구현 세부 정보

### 위치
`/graphrag/query/llm/local_search/local_search.py`

### 주요 함수

```python
async def local_search(
    query: str,
    context_builder: LocalContextBuilder,
    chat_model: LanguageModel,
    token_encoder: Tokenizer,
) -> SearchResult:
    # 1. 쿼리에서 엔티티 추출
    entities = await extract_entities(query)

    # 2. 컨텍스트 빌드
    context = await context_builder.build_context(
        query=query,
        entities=entities,
    )

    # 3. 응답 생성
    response = await chat_model.execute(
        prompt=format_prompt(query, context)
    )

    return SearchResult(response)
```

## 💡 최적 결과를 위한 팁

1. **구체적이기**: 가능하면 엔티티 이름 사용
   - "Microsoft와 OpenAI의 관계는 무엇인가?"
   - "회사에 대해 알려달라" (X)

2. **관계 타겟팅**: 연결에 대해 질문
   - "Microsoft를 설립한 사람은 누구인가?" (O)
   - "Microsoft Research와 협력하는 기관은 어디인가?" (O)

3. **대화 기록 사용**: 이전 쿼리 기반 구축
   ```python
   history = [
       {"role": "user", "content": "Microsoft의 AI 연구를 이끄는 사람은 누구인가?"},
       {"role": "assistant", "content": "John Smith가 AI 연구 부서를 이끌고 있습니다."},
       {"role": "user", "content": "그의 배경은 무엇인가?"}
   ]
   ```

## 🐛 문제 해결

### 관련 엔티티를 찾지 못함
**문제**: 쿼리가 엔티티와 일치하지 않음

**해결 방법**:
- 더广泛的 용어 시도
- 인덱싱 중 엔티티가 추출되었는지 확인
- 임베딩 모델 작동 여부 확인

### 컨텍스트가 너무 큼
**문제**: 토큰 제한에 대한 경고

**해결 방법**:
- `max_context_tokens` 감소
- `top_k_entities` 또는 `top_k_relationships` 감소
- 비율 설정 조정

## 🔗 관련 주제

- [[Global Search]] - 커뮤니티 레벨 검색
- [[DRIFT Search]] - 반복적 정제
- [[Entity]] - 지식 그래프 엔티티
- [[Relationship]] - 엔티티 관계
- [[Query Module]] - 검색 메서드 개요

---
*참고: [[Query Module]], [[Global Search]], [[Context Building Deep Dive]]*
