---
title: GraphRAG 모델 구성
tags:
  - configuration
  - models
  - llm
  - embedding
created: 2025-01-13
type: documentation
links:
  - [[Configuration Reference]]
  - [[Language Model Module]]
  - [[Index Module]]
  - [[Query Module]]
---

# GraphRAG 모델 구성

**모델 구성** 문서는 GraphRAG에서 사용하는 언어 모델(LLM)과 임베딩 모델을 설정하는 방법을 설명합니다.

## 📋 개요

GraphRAG는 다양한 LLM 프로바이더와 임베딩 모델을 지원합니다. 모델 구성은 다음을 제어합니다:

- **LLM 선택**: OpenAI, Azure OpenAI, 기타 프로바이더
- **인증**: API 키, Azure Managed Identity
- **성능 파라미터**: 동시 요청, 재시도, 제한
- **모델 동작**: Temperature, Top-P, Max Tokens

## 🤖 지원되는 모델 타입

### Chat 모델

| 타입 | 설명 | 예시 모델 |
|------|------|-----------|
| `openai_chat` | OpenAI chat 모델 | gpt-4-turbo, gpt-4o, gpt-3.5-turbo |
| `azure_openai_chat` | Azure OpenAI chat | gpt-4-turbo, gpt-35-turbo |
| `azure_openai_embedding` | Azure OpenAI 임베딩 | text-embedding-ada-002 |

### Embedding 모델

| 타입 | 설명 | 예시 모델 |
|------|------|-----------|
| `openai_embedding` | OpenAI 임베딩 | text-embedding-3-small, text-embedding-3-large |
| `azure_openai_embedding` | Azure OpenAI 임베딩 | text-embedding-ada-002 |

## 📝 모델 구성

### 기본 Chat 모델 구성

```yaml
models:
  chat:
    # 모델 타입
    type: openai_chat
    model: gpt-4-turbo-preview

    # 인증
    api_key: ${OPENAI_API_KEY}

    # 생성 파라미터
    temperature: 0.0
    max_tokens: 2000
    top_p: 1.0
    n: 1
    frequency_penalty: 0.0
    presence_penalty: 0.0
```

### Azure OpenAI 구성

```yaml
models:
  chat:
    type: azure_openai_chat
    model: gpt-4-turbo
    deployment_name: gpt-4-turbo-deployment

    # Azure 인증
    api_base: ${AZURE_OPENAI_ENDPOINT}
    api_key: ${AZURE_OPENAI_API_KEY}
    api_version: 2024-02-01
    auth_type: APIKey  # 또는 AzureManagedIdentity

    # Azure 특정
    organization: null
    audience: null
```

### Embedding 모델 구성

```yaml
models:
  embed:
    type: openai_embedding
    model: text-embedding-3-small
    api_key: ${OPENAI_API_KEY}

    # 배치 처리
    batch_size: 16
    concurrent_requests: 25

    # 임베딩 차원 (모델별)
    dimensions: 1536  # text-embedding-3-small
```

## ⚡ 성능 설정

### 동시 요청

```yaml
models:
  chat:
    # 동시 요청 수
    concurrent_requests: 10
    # 비동기 모드: threaded 또는 asyncio
    async_mode: threaded
```

### 속도 제한

```yaml
models:
  chat:
    # 자동 감지 또는 설정
    tokens_per_minute: auto
    requests_per_minute: auto
    # 제한 전략
    rate_limit_strategy: null
```

### 재시도 설정

```yaml
models:
  chat:
    # 재시도
    max_retries: 5
    retry_strategy: exponential_backoff
    max_retry_wait: 60.0
    # 타임아웃
    request_timeout: 60.0
```

## 🎯 사용 사례별 구성

### 추출 최적화

```yaml
models:
  chat:
    model: gpt-4-turbo-preview
    temperature: 0.0  # 결정적 출력
    max_tokens: 2000
    concurrent_requests: 10
```

### 요약 최적화

```yaml
models:
  chat:
    model: gpt-4o
    temperature: 0.0
    max_tokens: 3000
    top_p: 1.0
```

### 쿼리 최적화

```yaml
models:
  chat:
    model: gpt-4o-mini  # 더 빠른 응답
    temperature: 0.2  # 약간의 창의성
    max_tokens: 1500
```

### 비용 최적화

```yaml
models:
  chat:
    model: gpt-4o-mini  # 더 저렴
    max_tokens: 1000
    concurrent_requests: 5

  embed:
    model: text-embedding-3-small  # 더 저렴
```

## 🔄 추론 모델

OpenAI o1/o3 시리즈 추론 모델을 위한 설정:

```yaml
models:
  chat:
    model: o1-preview
    type: openai_chat
    api_key: ${OPENAI_API_KEY}

    # 추론 모델 특정
    reasoning_effort: medium  # low, medium, high

    # 추론 모델에서는 무시됨
    temperature: 1.0  # 항상 1.0
    top_p: 1.0
```

## 🌐 LiteLLM 프로바이더

LiteLLM을 통해 다른 프로바이더 사용:

```yaml
models:
  chat:
    type: openai_chat
    model_provider: litellm
    model: anthropic/claude-3-opus
    api_key: ${ANTHROPIC_API_KEY}
```

## 📊 모델 선택 가이드

### 엔티티 추출

| 요구사항 | 추천 모델 | 이유 |
|----------|-----------|------|
| 최고 품질 | gpt-4-turbo | 정확한 추출 |
| 비용 효율 | gpt-4o-mini | 빠르고 저렴 |
| 빠른 프로토타입 | gpt-3.5-turbo | 매우 빠름 |

### 커뮤니티 요약

| 요구사항 | 추천 모델 | 이유 |
|----------|-----------|------|
| 긴 요약 | gpt-4o | 긴 컨텍스트 |
| 간단 요약 | gpt-4o-mini | 빠르고 정확 |

### 쿼리/생성

| 요구사항 | 추천 모델 | 이유 |
|----------|-----------|------|
| 복잡한 질문 | gpt-4-turbo | 깊은 이해 |
| 간단한 질문 | gpt-4o-mini | 빠른 응답 |
| 추론 필요 | o1-preview | 복잡한 추론 |

## 🔧 구성 검증

### 모델 연결 테스트

```python
from graphrag.index import GraphRAGConfig

config = GraphRAGConfig("path/to/config.yaml")
# 연결 테스트
```

### CLI 테스트

```bash
# 구성 유효성 검사
graphrag init --validate --config ./settings.yaml

# 간단한 인덱싱 테스트
graphrag index --root ./test --input.file_pattern "test.txt"
```

## 📖 관련 문서

- [[Configuration Reference]] - 전체 구성 옵션
- [[Language Model Module]] - LLM 통합 상세
- [[Entity Extraction]] - 엔티티 추출에 모델 사용
- [[Community Report]] - 커뮤니티 요약에 모델 사용

---

*마지막 업데이트: 2025-01-13*
