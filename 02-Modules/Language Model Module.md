---
title: Language Model Module
tags:
  - module
  - llm
  - language-model
created: 2025-01-12
type: documentation
links:
  - [[Architecture Overview]]
  - [[Index Module]]
  - [[Query Module]]
---

# Language Model Module

**Language Model Module**(`/graphrag/language_model/`)은 OpenAI, Azure OpenAI, LiteLLM(100+ 제공자) 등 다양한 대형 언어 모델(LLM) 제공자와의 포괄적인 통합을 제공합니다.

## Overview

이 모듈의 주요 기능:
- 다중 제공자 지원(OpenAI, Azure, LiteLLM)
- 내장된 속도 제한 및 재시도 로직
- 토큰 관리 및 추적
- 비동기 네이티브 작업
- 응답 캐싱 통합

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                  Language Model Factory                      │
│  ┌────────────┐  ┌────────────┐  ┌────────────────────┐    │
│  │   OpenAI   │  │Azure OpenAI│  │     LiteLLM        │    │
│  │    Chat    │  │    Chat    │  │   (100+ providers) │    │
│  ├────────────┤  ├────────────┤  ├────────────────────┤    │
│  │   OpenAI   │  │Azure OpenAI│  │                     │    │
│  │ Embedding  │  │ Embedding  │  │     LiteLLM         │    │
│  └─────┬──────┘  └─────┬──────┘  └─────────┬──────────┘    │
└────────┼───────────────┼──────────────────┼─────────────────┘
         │               │                  │
         └───────────────┼──────────────────┘
                         │
                ┌────────▼────────┐
                │ BaseLanguageModel│
                │    (Interface)   │
                └────────┬─────────┘
                         │
         ┌───────────────┼───────────────┐
         │               │               │
    ┌────▼────┐   ┌─────▼─────┐   ┌───▼────┐
    │Rate Limit│   │ Retry     │   │ Token  │
    │  Limiter  │   │ Handler   │   │Counter │
    └──────────┘   └───────────┘   └────────┘
```

## Provider Integrations

### OpenAI

**Supported Models**:
- GPT-4, GPT-4 Turbo, GPT-4o
- GPT-3.5 Turbo
- text-embedding-3-small, text-embedding-3-large

**Configuration**:
```yaml
models:
  default_chat_model:
    type: openai_chat
    model: gpt-4-turbo-preview
    api_key: ${OPENAI_API_KEY}
    temperature: 0
    max_tokens: 4000
```

### Azure OpenAI

**Authentication**:
- API Key
- Azure Managed Identity (DefaultAzureCredential)

**Configuration**:
```yaml
models:
  default_chat_model:
    type: azure_openai_chat
    model: gpt-4
    api_base: https://resource.openai.azure.com
    api_version: 2024-02-01
    api_key: ${AZURE_OPENAI_API_KEY}
    deployment_name: my-deployment
    auth_type: api_key  # or azure_managed_identity
```

### LiteLLM

**Providers**: 100+ including Anthropic, Google, Cohere, Hugging Face, Mistral, etc.

**Configuration**:
```yaml
models:
  claude_model:
    type: chat
    model: claude-3-opus-20240229
    model_provider: anthropic
    api_key: ${ANTHROPIC_API_KEY}
```

## Performance Features

### Rate Limiting

이중 속도 제한으로 API 할당량 소진을 방지합니다:

```yaml
models:
  default_chat_model:
    requests_per_minute: 500
    tokens_per_minute: 150000
    rate_limit_strategy: sliding_window
```

**Algorithm**: Sliding window rate limiter
- Tracks request timestamps
- Tracks token consumption
- Waits until capacity available

### Retry Logic

**Strategies**:
- `exponential_backoff` (default): Wait time = min(2^attempt, max_wait)
- `incremental`: Wait time = min(attempt * base_delay, max_wait)
- `native_retry`: Provider-side retry (OpenAI)
- `random`: Randomized jitter

```yaml
models:
  default_chat_model:
    max_retries: 10
    retry_strategy: exponential_backoff
    max_retry_wait: 60
```

### Token Management

```python
# Token counting
from graphrag.language_model.tokens import Tokenizer

tokenizer = Tokenizer(encoding_name="cl100k_base")
count = tokenizer.count("Hello world")  # Returns: 2

# Truncation
truncated = tokenizer.truncate(text, max_tokens=1000)
```

**Supported Encodings**:
- `cl100k_base`: GPT-4, GPT-3.5-Turbo, text-embedding-3
- `o200k_base`: GPT-4o
- `p50k_base`: Code models

## Factory Usage

```python
from graphrag.language_model import LanguageModelFactory
from graphrag.config import LanguageModelConfig

# Create from config
config = LanguageModelConfig(
    type="openai_chat",
    model="gpt-4-turbo-preview",
    api_key="..."
)
llm = LanguageModelFactory.create_model(config)

# Execute
result = await llm.execute("Tell me about GraphRAG")
print(result.content)
print(result.tokens)  # {"total": 100, "input": 80, "output": 20}
```

## Async Operations

모든 작업은 비동기 네이티브입니다:

```python
import asyncio

async def process_batch(texts):
    tasks = [llm.execute(text) for text in texts]
    results = await asyncio.gather(*tasks)
    return results
```

**Configuration**:
```yaml
models:
  default_chat_model:
    async_mode: threaded  # or asyncio
    concurrent_requests: 25
```

## Related Topics

- [[Index Module]] - LLM usage in indexing
- [[Query Module]] - LLM usage in search
- [[Configuration Module]] - LLM configuration

---
*See also: [[Architecture Overview]], [[Index Module]], [[Query Module]]*
