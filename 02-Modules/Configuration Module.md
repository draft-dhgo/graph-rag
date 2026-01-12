---
title: Configuration Module
tags:
  - module
  - configuration
  - settings
created: 2025-01-12
type: documentation
links:
  - [[Architecture Overview]]
  - [[Index Module]]
  - [[Configuration Module]]
  - [[Storage Module]]
---

# Configuration Module

**Configuration Module**(`/graphrag/config/`)은 언어 모델, 데이터 처리, 스토리지, 검색, 워크플로우 등 GraphRAG 파이프라인의 모든 측면을 관리하는 포괄적인 시스템입니다.

## Overview

구성 모듈은 Pydantic 모델을 사용하여 타입 안전한 구성 관리를 제공합니다.
- YAML/JSON 파일 로딩
- 환경 변수 치환
- CLI 재정의 기능
- 검증 및 오류 확인

## Configuration Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                   Configuration Sources                     │
├───────────────┬───────────────┬─────────────────────────────┤
│ Settings.yaml │ .env File     │ CLI Arguments               │
└───────┬───────┴───────┬───────┴─────────────┬───────────────┘
        │                   │                     │
        └───────────────────┼─────────────────────┘
                            │
                    ┌───────▼────────┐
                    │ Configuration │
                    │    Loader      │
                    └───────┬────────┘
                            │
                    ┌───────▼────────┐
                    │   GraphRag     │
                    │     Config     │
                    └───────────────┘
```

## Main Configuration Structure

### GraphRagConfig

```python
class GraphRagConfig(BaseModel):
    # Core settings
    root_dir: str
    models: dict[str, LanguageModelConfig]
    input: InputConfig
    chunks: ChunkingConfig
    output: StorageConfig
    cache: CacheConfig

    # Pipeline components
    embed_text: TextEmbeddingConfig
    extract_graph: ExtractGraphConfig
    cluster_graph: ClusterGraphConfig
    community_reports: CommunityReportsConfig

    # Query configurations
    local_search: LocalSearchConfig
    global_search: GlobalSearchConfig
    drift_search: DRIFTSearchConfig
```

## Configuration Sections

### Model Configuration

```yaml
models:
  default_chat_model:
    type: openai_chat  # or azure_openai_chat
    model: gpt-4-turbo-preview
    api_key: ${OPENAI_API_KEY}
    temperature: 0
    max_tokens: 4000

  default_embedding_model:
    type: openai_embedding
    model: text-embedding-3-small
    api_key: ${OPENAI_API_KEY}
```

### Input Configuration

```yaml
input:
  type: file
  file_type: text  # csv, json, text
  file_pattern: "input/*.txt"
  encoding: "utf-8"
  text_column: "text"
  title_column: null
  metadata: []
```

### Chunking Configuration

```yaml
chunks:
  size: 1200        # Chunk size in tokens
  overlap: 100      # Overlap between chunks
  strategy: tokens  # tokens or sentences
```

### Storage Configuration

```yaml
output:
  type: file        # file, blob, cosmosdb
  base_dir: "./output"

# For Azure Blob
# output:
#   type: blob
#   connection_string: ${BLOB_CONNECTION_STRING}
#   container_name: "graphrag"
#   base_dir: "output"
```

### Vector Store Configuration

```yaml
vector_store:
  type: lancedb     # lancedb, azure_ai_search, cosmos_db
  db_uri: "./output/lancedb"

# For Azure AI Search
# vector_store:
#   type: azure_ai_search
#   url: https://my-search.search.windows.net
#   api_key: ${AZURE_SEARCH_API_KEY}
#   index_name: "graphrag"
```

## Loading Configuration

### Basic Loading

```python
from graphrag.config import load_config

# Load with defaults
config = load_config(root_dir="./project")

# Load with custom file
config = load_config(
    root_dir="./project",
    config_filepath="./custom.yaml"
)

# Load with CLI overrides
config = load_config(
    root_dir="./project",
    cli_overrides={"output.base_dir": "custom_output"}
)
```

### Accessing Configuration

```python
# Get model configuration
chat_model = config.models["default_chat_model"]
embedding_model = config.models["default_embedding_model"]

# Access pipeline settings
chunk_size = config.chunks.size
overlap = config.chunks.overlap

# Get search settings
local_search_max_tokens = config.local_search.max_context_tokens
```

## Environment Variables

GraphRAG는 구성 파일에서 환경 변수 치환을 지원합니다:

```yaml
# settings.yaml
models:
  default_chat_model:
    api_key: ${OPENAI_API_KEY}  # Replaced with env var
    api_base: ${AZURE_OPENAI_API_BASE:-https://api.openai.com/v1}  # With default
```

### .env File

```bash
# .env
OPENAI_API_KEY=sk-...
AZURE_OPENAI_API_KEY=...
AZURE_OPENAI_API_BASE=https://...
BLOB_CONNECTION_STRING=...
```

## Configuration Defaults

| Setting | Default | Description |
|---------|---------|-------------|
| `chunks.size` | 1200 | Token chunk size |
| `chunks.overlap` | 100 | Chunk overlap |
| `local_search.max_context_tokens` | 12000 | Local search context |
| `global_search.max_tokens` | 12000 | Global search max tokens |
| `cluster_graph.max_cluster_size` | 50 | Community size target |

## Related Topics

- [[Configuration Module]] - LLM settings
- [[Storage Module]] - Storage backends
- [[Index Module]] - Pipeline configuration
- [[Getting Started]] - Setup guide

---
*See also: [[Architecture Overview]], [[Configuration Module]], [[Storage Module]]*
