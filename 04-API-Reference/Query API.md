---
title: 쿼리 API
tags:
  - api
  - query
  - search
  - python-api
created: 2025-01-12
type: api-reference
links:
  - [[Indexing API]]
  - [[Local Search]]
  - [[Global Search]]
  - [[DRIFT Search]]
  - [[Query Module]]
---

# 쿼리 API

**쿼리 API**는 GraphRAG 지식 그래프 인덱스에서 정보를 검색하고 가져오기 위한 프로그래매틱 액세스를 제공합니다.

## 📋 개요

```python
from graphrag.api import local_search, global_search, drift_search, basic_search

# 로컬 검색
result = await local_search("X란 무엇인가요?", config)

# 전역 검색
result = await global_search("주제를 요약해주세요", config)

# DRIFT 검색
result = await drift_search("X와 Y의 관계는 무엇인가요?", config)

# 기본 검색
result = await basic_search("X에 대한 문서를 찾아주세요", config)
```

## 🔧 함수 레퍼런스

### local_search()

특정 엔티티에 초점을 맞춘 로컬 검색을 수행합니다.

```python
async def local_search(
    query: str,
    config: GraphRagConfig | str,
    context_builder: LocalContextBuilder | None = None,
    conversation_history: list[dict] | None = None,
    callbacks: QueryCallbacks | None = None,
) -> str:
    """
    로컬 검색을 수행합니다.

    Args:
        query: 검색 쿼리
        config: 설정 객체 또는 경로
        context_builder: 선택적 사용자 정의 컨텍스트 빌더
        conversation_history: 선택적 채팅 기록
        callbacks: 모니터링을 위한 선택적 콜백

    Returns:
        str: 생성된 응답
    """
```

**참조**: [[Local Search]]

### global_search()

커뮤니티 수준 컨텍스트를 사용하여 전역 검색을 수행합니다.

```python
async def global_search(
    query: str,
    config: GraphRagConfig | str,
    community_level: int = 2,
    response_type: str = "JSON Page",
    conversation_history: list[dict] | None = None,
    callbacks: QueryCallbacks | None = None,
) -> str:
    """
    전역 검색을 수행합니다.

    Args:
        query: 검색 쿼리
        config: 설정 객체 또는 경로
        community_level: 계층 레벨 (0-3)
        response_type: 응답 형식
        conversation_history: 선택적 채팅 기록
        callbacks: 모니터링을 위한 선택적 콜백

    Returns:
        str: 생성된 응답
    """
```

**참조**: [[Global Search]]

### drift_search()

복잡한 다중 홉 질문에 대한 DRIFT 검색을 수행합니다.

```python
async def drift_search(
    query: str,
    config: GraphRagConfig | str,
    conversation_history: list[dict] | None = None,
    callbacks: QueryCallbacks | None = None,
) -> str:
    """
    DRIFT 검색을 수행합니다.

    Args:
        query: 검색 쿼리
        config: 설정 객체 또는 경로
        conversation_history: 선택적 채팅 기록
        callbacks: 모니터링을 위한 선택적 콜백

    Returns:
        str: 생성된 응답
    """
```

**참조**: [[DRIFT Search]]

### basic_search()

그래프 구조 없이 기본 RAG 검색을 수행합니다.

```python
async def basic_search(
    query: str,
    config: GraphRagConfig | str,
    conversation_history: list[dict] | None = None,
    callbacks: QueryCallbacks | None = None,
) -> str:
    """
    기본 검색을 수행합니다.

    Args:
        query: 검색 쿼리
        config: 설정 객체 또는 경로
        conversation_history: 선택적 채팅 기록
        callbacks: 모니터링을 위한 선택적 콜백

    Returns:
        str: 생성된 응답
    """
```

## 📝 사용 예제

### 기본 로컬 검색

```python
from graphrag.api import local_search

result = await local_search(
    "AI 연구에 참여하는 사람은 누구인가요?",
    "./settings.yaml"
)
print(result)
```

### 레벨 선택과 함께 전역 검색

```python
from graphrag.api import global_search

# 레벨 0: 광범위한 개요
result = await global_search(
    "주요 주제는 무엇인가요?",
    "./settings.yaml",
    community_level=0
)

# 레벨 2: 중간 세부 수준 (기본값)
result = await global_search(
    "핵심 주제를 요약해주세요",
    "./settings.yaml",
    community_level=2
)

# 레벨 3: 상세 세부 정보
result = await global_search(
    "상세한 하위 주제는 무엇인가요?",
    "./settings.yaml",
    community_level=3
)
```

### 대화 기록과 함께 검색

```python
from graphrag.api import local_search

history = [
    {"role": "user", "content": "마이크로소프트의 AI 연구를 이끄는 사람은 누구인가요?"},
    {"role": "assistant", "content": "John Smith가 AI 연구 부문을 이끌고 있습니다."},
    {"role": "user", "content": "그의 배경은 무엇인가요?"}
]

result = await local_search(
    "그의 배경은 무엇인가요?",
    "./settings.yaml",
    conversation_history=history
)
```

### 콜백과 함께 검색

```python
from graphrag.api import global_search
from graphrag.callbacks import QueryCallbacks

class MyQueryCallbacks(QueryCallbacks):
    def on_search_start(self, query: str, method: str):
        print(f"Searching: {query} ({method})")

    def on_search_complete(self, result: str):
        print(f"Result: {len(result)} characters")

    def on_llm_call(self, prompt: str, response: str):
        print(f"LLM call: {len(prompt)} → {len(response)} tokens")

result = await global_search(
    "주요 주제는 무엇인가요?",
    "./settings.yaml",
    callbacks=MyQueryCallbacks()
)
```

### 사용자 정의 컨텍스트 빌더

```python
from graphrag.api import local_search
from graphrag.query.llm.local_search import LocalContextBuilder

# 사용자 정의 컨텍스트 빌더 사용
context_builder = LocalContextBuilder(
    token_encoder=tokenizer,
    text_unit_prop=0.6,  # 60% 텍스트 단위
    community_prop=0.2,  # 20% 커뮤니티
)

result = await local_search(
    "X란 무엇인가요?",
    "./settings.yaml",
    context_builder=context_builder
)
```

## 📊 응답 형식

API는 문자열 응답을 반환합니다. 구조화된 출력의 경우:

```python
result = await local_search("Query", config)

# 결과는 LLM이 생성한 응답입니다
print(result)

# 구조화된 응답의 경우 response_type 파라미터 사용
result = await global_search(
    "Query",
    config,
    response_type="JSON Page"  # 또는 "JSON", "Markdown"
)
```

## ⚙️ 쿼리 설정

### 로컬 검색 설정

```yaml
local_search:
  text_unit_prop: 0.5
  community_prop: 0.25
  top_k_entities: 10
  top_k_relationships: 10
  max_context_tokens: 12000
  conversation_history_max_turns: 10
```

### 전역 검색 설정

```yaml
global_search:
  max_tokens: 12000
  data_max_tokens: 12000
  map_max_tokens: 1000
  reduce_max_tokens: 2000
  concurrency: 32
  community_level: 2
```

## 🐛 오류 처리

```python
from graphrag.api import local_search
from graphrag.errors import GraphRAGError, QueryError

try:
    result = await local_search("Query", "./settings.yaml")
except QueryError as e:
    print(f"Query error: {e}")
except GraphRAGError as e:
    print(f"General error: {e}")
```

## 🔍 검색 방법 선택

| 질문 유형 | 추천 방법 |
|---------------|-------------------|
| "X는 누구/무엇인가요?" | `local_search` |
| "X와 Y의 관계는 무엇인가요?" | `local_search` |
| "주요 주제는 무엇인가요?" | `global_search` |
| "데이터셋을 요약해주세요" | `global_search` |
| "X는 코퍼스를 통해 Y와 어떻게 관련되나요?" | `drift_search` |
| "X에 대한 문서를 찾아주세요" | `basic_search` |

## 🔗 관련 주제

- [[Indexing API]] - 인덱스 구축
- [[Local Search]] - 로컬 검색 상세 정보
- [[Global Search]] - 전역 검색 상세 정보
- [[DRIFT Search]] - DRIFT 검색 상세 정보
- [[Query Module]] - 쿼리 아키텍처

---
*참고: [[Indexing API]], [[Local Search]], [[Global Search]], [[Query Module]]*
