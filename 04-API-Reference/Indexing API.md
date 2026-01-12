---
title: 인덱싱 API
tags:
  - api
  - indexing
  - python-api
created: 2025-01-12
type: api-reference
links:
  - [[Query API]]
  - [[Indexing API]]
  - [[Indexing API]]
  - [[Index Module]]
---

# 인덱싱 API

**인덱싱 API**는 소스 문서에서 지식 그래프 인덱스를 구축하기 위한 프로그래매틱 액세스를 제공합니다.

## 📋 개요

```python
from graphrag.api import build_index

await build_index(config_or_path)
```

## 🔧 함수 레퍼런스

### build_index()

소스 문서에서 지식 그래프 인덱스를 구축합니다.

```python
async def build_index(
    config: GraphRagConfig | str,
    callbacks: WorkflowCallbacks | None = None,
    is_update_run: bool = False,
    memory_profile: bool = False,
) -> None:
    """
    지식 그래프 인덱스를 구축합니다.

    Args:
        config: 설정 객체 또는 settings.yaml 경로
        callbacks: 진행률 추적을 위한 선택적 콜백
        is_update_run: True인 경우 증분 업데이트 실행
        memory_profile: 메모리 프로파일링 활성화

    Raises:
        GraphRAGError: 파이프라인 실패 시
    """
```

## 📝 사용 예제

### 기본 사용법

```python
from graphrag.api import build_index

# 설정 파일 경로 사용
await build_index("./settings.yaml")

# 설정 객체 사용
from graphrag.config import load_config
config = load_config("./project")
await build_index(config)
```

### 콜백과 함께 사용

```python
from graphrag.api import build_index
from graphrag.callbacks import WorkflowCallbacks

class MyCallbacks(WorkflowCallbacks):
    def on_workflow_start(self, name: str):
        print(f"Starting: {name}")

    def on_workflow_end(self, name: str, result):
        print(f"Finished: {name}")

    def on_llm_start(self, text: str):
        print(f"LLM call: {text[:50]}...")

    def on_llm_end(self, text: str, response: str):
        print(f"LLM response: {len(response)} tokens")

await build_index(
    "./settings.yaml",
    callbacks=MyCallbacks()
)
```

### 메모리 프로파일링과 함께 사용

```python
from graphrag.api import build_index

await build_index(
    "./settings.yaml",
    memory_profile=True
)
# 메모리 사용 통계를 출력합니다
```

### 증분 업데이트

```python
from graphrag.api import build_index

await build_index(
    "./settings.yaml",
    is_update_run=True
)
```

## ⚙️ 설정

### 최소 설정

```yaml
# settings.yaml
models:
  default_chat_model:
    type: openai_chat
    model: gpt-4-turbo-preview
    api_key: ${OPENAI_API_KEY}

  default_embedding_model:
    type: openai_embedding
    model: text-embedding-3-small
    api_key: ${OPENAI_API_KEY}

input:
  type: file
  file_type: text
  file_pattern: "input/*.txt"

output:
  type: file
  base_dir: "./output"

chunks:
  size: 1200
  overlap: 100

vector_store:
  type: lancedb
  db_uri: "./output/lancedb"
```

### 전체 설정

모든 옵션은 [[Configuration Module]]를 참조하세요.

## 📊 출력

### 디렉토리 구조

```
output/
├── create_final_documents.parquet
├── create_final_text_units.parquet
├── create_final_entities.parquet
├── create_final_relationships.parquet
├── create_final_communities.parquet
├── create_final_community_reports.parquet
├── create_final_covariates.parquet
└── lancedb/
```

### 결과 읽기

```python
import pandas as pd

# 엔티티 로드
entities = pd.read_parquet("output/create_final_entities.parquet")
print(f"Extracted {len(entities)} entities")

# 커뮤니티 로드
communities = pd.read_parquet("output/create_final_communities.parquet")
print(f"Found {len(communities)} communities")

# 리포트 로드
reports = pd.read_parquet("output/create_final_community_reports.parquet")
for _, report in reports.iterrows():
    print(f"{report['community']}: {report['summary']}")
```

## 🐛 오류 처리

```python
from graphrag.api import build_index
from graphrag.errors import (
    GraphRAGError,
    ConfigurationError,
    LLMError,
    StorageError
)

try:
    await build_index("./settings.yaml")
except ConfigurationError as e:
    print(f"Configuration error: {e}")
except LLMError as e:
    print(f"LLM error: {e}")
except StorageError as e:
    print(f"Storage error: {e}")
except GraphRAGError as e:
    print(f"General error: {e}")
```

## ⚡ 성능 팁

1. **캐싱 활성화**: LLM API 호출 감소
   ```yaml
   cache:
     type: file
     base_dir: "./cache"
   ```

2. **빠른 파이프라인 사용**: 더 빠른 처리
   ```bash
   graphrag index --method fast
   ```

3. **동시성 조정**: API 제한에 맞추기
   ```yaml
   models:
     default_chat_model:
       concurrent_requests: 25
       requests_per_minute: 500
   ```

## 🔗 관련 주제

- [[Query API]] - 인덱스 검색
- [[Indexing API]] - 프로젝트 설정
- [[Indexing API]] - 사용자 정의 프롬프트
- [[Configuration Module]] - 모든 옵션
- [[Index Module]] - 인덱싱 작동 방식

---
*참고: [[Indexing API]], [[Query API]], [[Index Module]]*
