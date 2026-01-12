---
title: GraphRAG 구성 레퍼런스
tags:
  - configuration
  - reference
  - settings
created: 2025-01-13
type: documentation
links:
  - [[Architecture Overview]]
  - [[Index Module]]
  - [[Query Module]]
  - [[Language Model Module]]
  - [[Storage Module]]
---

# GraphRAG 구성 레퍼런스

**구성 레퍼런스** 문서는 GraphRAG 시스템의 모든 설정 옵션, 파라미터, 기본값에 대한 포괄적인 가이드를 제공합니다.

## 📋 개요

GraphRAG는 YAML 기반 구성 파일을 사용하여 인덱싱 파이프라인과 쿼리 시스템의 모든 측면을 제어합니다. 구성 시스템은 다음을 지원합니다:

- **다중 환경**: 개발, 스테이징, 프로덕션 환경
- **환경 변수 치환**: 보안 및 유연성
- **타입 안전성**: Pydantic 기반 검증
- **CLI 재정의**: 런타임 설정 변경

## 🏗️ 구성 아키텍처

### 기본 구조

```yaml
# 그래프의 최상위 레벨 구조
root_dir: ./output
models:
  chat: ...
  embed: ...
input: ...
chunks: ...
output: ...
```

### 구성 로딩 순서

1. **기본값**: 내부 기본값 로드
2. **구성 파일**: YAML/JSON 파일 로드
3. **환경 변수**: `${ENV_VAR}` 치환
4. **CLI 인자**: 명령줄 재정의

## 📝 구성 섹션

### 1. 모델 구성 (models)

언어 모델과 임베딩 모델을 설정합니다.

#### Chat 모델

```yaml
models:
  chat:
    type: openai_chat # 또는 azure_openai_chat
    model: gpt-4-turbo-preview
    api_key: ${OPENAI_API_KEY}
    temperature: 0.0
    max_tokens: 2000
    # 성능 설정
    concurrent_requests: 10
    max_retries: 5
    request_timeout: 60.0
    # 속도 제한
    tokens_per_minute: auto
    requests_per_minute: auto
```

#### Embedding 모델

```yaml
models:
  embed:
    type: openai_embedding # 또는 azure_openai_embedding
    model: text-embedding-3-small
    api_key: ${OPENAI_API_KEY}
    # 배치 처리
    batch_size: 16
    concurrent_requests: 25
```

### 2. 입력 구성 (input)

데이터 소스와 형식을 설정합니다.

```yaml
input:
  type: file # file, blob
  file_type: text # text, csv, json
  encoding: utf-8
  file_pattern: "input/*.txt"
  # CSV 특정
  text_column: text
  title_column: title
  # 메타데이터 추출
  metadata: []
```

### 3. 청킹 구성 (chunks)

텍스트 분할 전략을 설정합니다.

```yaml
chunks:
  size: 1200 # 청크 크기 (토큰)
  overlap: 100 # 청크 간 중복
  strategy: tokens # tokens, sentences
```

### 4. 출력 구성 (output)

결과 저장 위치를 설정합니다.

```yaml
output:
  type: file # file, blob, cosmosdb
  base_dir: ./output
```

### 5. 그래프 추출 구성 (extract_graph)

엔티티와 관계 추출을 설정합니다.

```yaml
extract_graph:
  prompt: null # 기본 프롬프트 사용
  entity_types: # 추출할 엔티티 유형
    - organization
    - person
    - location
    - event
    - gpe
  max_gleanings: 1 # 재추출 횟수
  # LLM 설정
  model: chat
  tuple_delimiter: "<|>"
  record_delimiter: "##"
  completion_delimiter: "<|COMPLETE|>"
```

### 6. 커뮤니티 구성 (cluster_graph)

커뮤니티 탐지 파라미터를 설정합니다.

```yaml
cluster_graph:
  # Leiden 알고리즘
  max_cluster_size: 50
  hierarchy_level: null # null = 자동 선택
  use_lcc: true # 최대 연결 컴포넌트 사용
  # Leiden 파라미터
  resolution: 1.0
```

### 7. 커뮤니티 리포트 구성 (community_reports)

커뮤니티 요약 생성을 설정합니다.

```yaml
community_reports:
  prompt: null
  model: chat
  max_length: 2000
  max_input_length: 8000
  # 생성 파라미터
  temperature: 0.0
  top_p: 1.0
  n: 1
```

### 8. 임베딩 구성 (embed_graph)

그래프 요소의 임베딩을 설정합니다.

```yaml
embed_graph:
  model: embed
  # 무엇을 임베딩할지
  embed_graph_embedding: true # 노드 임베딩
  embed_graph_attributes: false # 속성 임베딩
  embed_graph_knowledge: true # 설명 임베딩
  # 벡터 스토어
  vector_store: lancedb
```

### 9. 로컬 검색 구성 (local_search)

```yaml
local_search:
  # 텍스트 유닛
  text_unit_prop: 0.5 # 텍스트 유닛 비율
  # 커뮤니티
  community_prop: 0.1 # 커뮤니티 비율
  # conversations
  conversation_history_max_turns: 5
  # 컨텍스트
  top_k_mapped_entities: 10
  top_k_relationships: 10
  max_context_tokens: 8000
  # LLM
  model: chat
  temperature: 0.0
  top_p: 1.0
  n: 1
```

### 10. 글로벌 검색 구성 (global_search)

```yaml
global_search:
  # 커뮤니티
  max_level: 3 # 계층 레벨
  use_community_summary: true
  # 컨텍스트
  max_context_tokens: 12000
  token_budget: 4000
  # LLM
  model: chat
  temperature: 0.0
  top_p: 1.0
  n: 1
  # 정렬
  rank_conversation_history: false
```

### 11. UMAP 구성 (umap)

차원 축소를 설정합니다.

```yaml
umap:
  num_neighbors: 15
  min_dist: 0.1
  metric: cosine
```

### 12. 캐시 구성 (cache)

```yaml
cache:
  type: file # file, memory, blob, cosmosdb
  base_dir: ./cache
```

### 13. 리포팅 구성 (reporting)

```yaml
reporting:
  type: file # file, blob, console
  base_dir: ./reports
```

## 🔧 환경 변수

### 지원되는 환경 변수

| 변수 | 설명 | 예시 |
|------|------|------|
| `OPENAI_API_KEY` | OpenAI API 키 | `sk-...` |
| `AZURE_OPENAI_API_KEY` | Azure OpenAI 키 | `...` |
| `AZURE_OPENAI_ENDPOINT` | Azure OpenAI 엔드포인트 | `https://...` |
| `AZURE_OPENAI_API_VERSION` | Azure API 버전 | `2024-02-01` |
| `GRAPHRAG_ROOT_DIR` | 기본 루트 디렉토리 | `./output` |
| `GRAPHRAG_CACHE_DIR` | 캐시 디렉토리 | `./cache` |

### 환경 변수 사용

```yaml
models:
  chat:
    api_key: ${OPENAI_API_KEY}
    api_base: ${AZURE_OPENAI_ENDPOINT:-https://api.openai.com/v1}
```

## 📊 전체 예제 구성

```yaml
# GraphRAG 구성 파일 예제
---
# 기본 경로
root_dir: ./output

# 언어 모델
models:
  chat:
    type: openai_chat
    model: gpt-4-turbo-preview
    api_key: ${OPENAI_API_KEY}
    temperature: 0.0
    max_tokens: 2000
    concurrent_requests: 10
    max_retries: 5
    request_timeout: 60.0

  embed:
    type: openai_embedding
    model: text-embedding-3-small
    api_key: ${OPENAI_API_KEY}
    batch_size: 16
    concurrent_requests: 25

# 입력
input:
  type: file
  file_type: text
  encoding: utf-8
  file_pattern: "input/*.txt"

# 청킹
chunks:
  size: 1200
  overlap: 100
  strategy: tokens

# 출력
output:
  type: file
  base_dir: ./output

# 그래프 추출
extract_graph:
  model: chat
  entity_types:
    - organization
    - person
    - location
    - event
    - gpe
  max_gleanings: 1

# 커뮤니티 탐지
cluster_graph:
  max_cluster_size: 50
  use_lcc: true

# 커뮤니티 리포트
community_reports:
  model: chat
  max_length: 2000
  max_input_length: 8000

# 임베딩
embed_graph:
  model: embed
  embed_graph_embedding: true

# 로컬 검색
local_search:
  text_unit_prop: 0.5
  community_prop: 0.1
  top_k_mapped_entities: 10
  top_k_relationships: 10
  max_context_tokens: 8000

# 글로벌 검색
global_search:
  max_level: 3
  use_community_summary: true
  max_context_tokens: 12000

# 캐시
cache:
  type: file
  base_dir: ./cache

# 리포팅
reporting:
  type: file
  base_dir: ./reports
```

## 🚀 빠른 시작

### 1. 새 구성 파일 생성

```bash
graphrag init --root ./my-project
```

### 2. 구성 파일 사용

```bash
# 기본 구성
graphrag index --root ./my-project

# 사용자 정의 구성
graphrag index --config ./custom-config.yaml

# CLI 재정의
graphrag index --root . --input.file_pattern "data/*.txt"
```

### 3. 구성 검증

```bash
# 구성 유효성 검사
graphrag init --validate --config ./settings.yaml
```

## 📖 관련 문서

- [[Index Module]] - 인덱싱 파이프라인 구성
- [[Query Module]] - 쿼리 시스템 구성
- [[Language Model Module]] - LLM 설정 상세
- [[Storage Module]] - 스토리지 옵션

---

*마지막 업데이트: 2025-01-13*
