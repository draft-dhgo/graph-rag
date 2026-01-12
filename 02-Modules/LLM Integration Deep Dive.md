---
title: LLM Integration Deep Dive
tags:
  - deep-dive
  - llm
  - openai
  - azure
  - rate-limiting
  - implementation
created: 2025-01-12
type: deep-dive
links:
  - [[Language Model Module]]
  - [[Index Module]]
  - [[Query Module]]
---

# LLM Integration Deep Dive

GraphRAG의 LLM 통합 계층은 OpenAI, Azure OpenAI, LiteLLM 등 다양한 LLM 제공자와의 통합을 담당하며, 속도 제한, 재시도, 토큰 관리를 포함한 프로덕션 준비 인터페이스를 제공합니다.

## 목차

### 1. 개요
- [LLM 모듈의 책임](#-llm-모듈의-책임)
- [빗대어 보기: 고속도로 톨게이트 시스템](#-빗대어-보기-고속도로-톨게이트-시스템)

### 2. 아키텍처
- [시스템 아키텍처](#-아키텍처)
- [계층별 상세](#-계층별-상세)

### 3. 제공자별 구현
- [OpenAI](#1-openai)
- [Azure OpenAI](#2-azure-openai)
- [LiteLLM](#3-litellm)

### 4. 속도 제한 관리
- [Rate Limiter 구현](⚡-속도-제한-rate-limiting)
- [이중 제한 시스템](#이중-제한-시스템)
- [대기 전략](#대기-전략)

### 5. 재시도 로직
- [지수 백오프](#지수-백오프-exponential-backoff)
- [재시도 전략 비교](#재시도-전략-비교)

### 6. 토큰 관리
- [토큰 카운터](#-토큰-관리)
- [모델별 인코딩](#모델별-인코딩)

### 7. 고급 기법
- [비동기 처리](#-비동기-처리)
- [스트리밍 응답](#스트리밍-응답)
- [함수 호출](#함수 호출-function-calling)

---

## 🎯 LLM 모듈의 책임

```mermaid
flowchart LR
    A[LLM 모듈] --> B[다중 제공자 지원]
    A --> C[속도 제한 관리]
    A --> D[재시도 로직]
    A --> E[비동기 처리]
    A --> F[토큰 추적]

    style A fill:#fff9c4
    style B fill:#c8e6c9
    style C fill:#e1bee7
    style D fill:#ffccbc
    style E fill:#fff59d
    style F fill:#e1f5fe
```

1. **다중 제공자 지원**: OpenAI, Azure OpenAI, 100+ LiteLLM 제공자
2. **속도 제한 관리**: RPM/TPM 이중 제한
3. **재시도 로직**: 지수 백오프 및 점진적 지연
4. **비동기 처리**: 병렬 요청 처리
5. **토큰 추적**: 사용량 모니터링

## 📖 빗대어 보기: 고속도로 톨게이트 시스템

LLM 통합의 속도 제한 관리는 **고속도로 톨게이트의 차량 통제 시스템**과 유사합니다:

| 톨게이트 시스템 | LLM 속도 제한 |
|---------------|--------------|
| 차량 진입 | API 요청 |
| 시간당 차량 제한 | RPM (Requests Per Minute) |
| 하중 제한 | TPM (Tokens Per Minute) |
| 대기 큐 | 요청 대기열 |
| 우선 차량 | 우선순위 처리 |
| 재진입 대기 시간 | 재시도 지연 |

```mermaid
flowchart TB
    subgraph Toll["톨게이트 시스템"]
        T1["🚗 차량 도착"] --> T2["🎫 톨게이트"]
        T2 --> T3{"제한 확인"}
        T3 -->|초과| T4["⏳ 대기"]
        T3 -->|가능| T5["✅ 통과"]
        T4 --> T3
    end

    subgraph LLM["LLM 속도 제한"]
        L1["📡 API 요청"] --> L2["🔒 Rate Limiter"]
        L2 --> L3{"제한 확인"}
        L3 -->|초과| L4["⏳ 대기"]
        L3 -->|가능| L5["✅ LLM 호출"]
        L4 --> L3
    end

    style Toll fill:#e3f2fd
    style LLM fill:#f3e5f5
```

## 🏗️ 아키텍처

```mermaid
flowchart TB
    subgraph Client["클라이언트 코드"]
        API["build_index<br/>local_search<br/>global_search"]
    end

    subgraph Factory["LLM Factory"]
        OF["OpenAI<br/>Factory"]
        AF["Azure OpenAI<br/>Factory"]
        LF["LiteLLM<br/>Factory"]
    end

    subgraph Models["LLM 인스턴스"]
        OAI["OpenAI Chat<br/>gpt-4-turbo"]
        AOA["Azure OpenAI Chat<br/>gpt-4"]
        LLM["LiteLLM<br/>claude-3-opus"]
    end

    subgraph Protection["보호 계층"]
        RL["Rate Limiter<br/>🎯 RPM + TPM"]
        RT["Retry Handler<br/>🔄 Exponential Backoff"]
        TOK["Token Counter<br/>📊 tiktoken"]
    end

    API --> Factory
    Factory --> OAI
    Factory --> AOA
    Factory --> LLM

    OAI --> RL --> RT
    AOA --> RL --> RT
    LLM --> RL --> RT

    RT --> TOK
    TOK --> RESP["LLM Response<br/>✅ 결과 반환"]

    style Client fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    style Factory fill:#fff3e0,stroke:#f57c00,stroke-width:2px
    style Models fill:#e8f5e9,stroke:#388e3c,stroke-width:2px
    style Protection fill:#fce4ec,stroke:#c2185b,stroke-width:2px
    style RESP fill:#c8e6c9,stroke:#2e7d32,stroke-width:3px
```

### 계층별 상세

```mermaid
flowchart TB
    subgraph L1["계층 1: 팩토리"]
        F1["제공자 선택<br/>설정 로드"]
    end

    subgraph L2["계층 2: 모델"]
        M1["LLM 인스턴스화<br/>모델 설정"]
    end

    subgraph L3["계층 3: 보호"]
        P1["속도 제한<br/>재시도 로직<br/>토큰 카운팅"]
    end

    subgraph L4["계층 4: 전송"]
        T1["HTTP 요청<br/>응답 파싱"]
    end

    F1 --> M1 --> P3 --> T1

    style L1 fill:#e1f5fe
    style L2 fill:#fff9c4
    style L3 fill:#ffcdd2
    style L4 fill:#c8e6c9
```

## 📋 제공자별 구현

### 1. OpenAI

```mermaid
flowchart TB
    A["OpenAI Chat"] --> B["API Key 인증"]
    B --> C["gpt-4-turbo-preview<br/>gpt-4o<br/>gpt-3.5-turbo"]

    C --> D["StaticRateLimiter<br/>RPM: 500<br/>TPM: 100,000"]

    D --> E["RetryConfig<br/>max_retries: 10<br/>max_wait: 60s"]

    style A fill:#c8e6c9
    style C fill:#e1f5fe
    style D fill:#fff9c4
    style E fill:#e1bee7
```

```python
class OpenAIChat(BaseLanguageModel):
    def __init__(
        self,
        model: str = "gpt-4-turbo-preview",
        api_key: str | None = None,
        temperature: float = 0.0,
        max_tokens: int = 4000,
        requests_per_minute: int = 500,
        tokens_per_minute: int = 100000,
    ):
        self.client = AsyncOpenAI(api_key=api_key)
        self.model = model
        self.rate_limiter = StaticRateLimiter(
            requests_per_minute,
            tokens_per_minute
        )
        self.retry_config = RetryConfig(
            strategy="exponential_backoff",
            max_retries=10,
            max_retry_wait=60
        )
```

### 2. Azure OpenAI

```mermaid
flowchart TB
    A["Azure OpenAI Chat"] --> B1["API Key 인증"]
    A --> B2["Managed Identity"]

    B1 --> C["Azure Endpoint<br/>API Version<br/>Deployment Name"]

    C --> D["Azure Credential"]

    style A fill:#fff9c4
    style B1 fill:#c8e6c9
    style B2 fill:#e1bee7
    style D fill:#e1f5fe
```

```python
class AzureOpenAIChat(BaseLanguageModel):
    def __init__(
        self,
        model: str,
        api_base: str,
        api_version: str,
        deployment_name: str,
        api_key: str | None = None,
        auth_type: str = "api_key",  # or "azure_managed_identity"
    ):
        if auth_type == "azure_managed_identity":
            # DefaultAzureCredential 사용
            from azure.identity import DefaultAzureCredential
            credential = DefaultAzureCredential()

            self.client = AsyncAzureOpenAI(
                azure_endpoint=api_base,
                api_version=api_version,
                azure_deployment=deployment_name,
                credential=credential
            )
        else:
            self.client = AsyncAzureOpenAI(
                azure_endpoint=api_base,
                api_version=api_version,
                azure_deployment=deployment_name,
                api_key=api_key
            )
```

### 3. LiteLLM

```mermaid
flowchart TB
    A["LiteLLM"] --> B1["Claude"]
    A --> B2["Gemini"]
    A --> B3["HuggingFace"]
    A --> B4["100+ 제공자"]

    B1 --> C["통합 인터페이스"]
    B2 --> C
    B3 --> C
    B4 --> C

    style A fill:#e1bee7
    style C fill:#c8e6c9
```

```python
class LiteLLMModel(BaseLanguageModel):
    def __init__(
        self,
        model: str,  # e.g., "claude-3-opus-20240229"
        model_provider: str,  # e.g., "anthropic"
        api_key: str | None = None,
    ):
        import litellm

        self.model = model
        self.model_provider = model_provider
        self.litellm_client = litellm.aasync_litellm
```

## ⚡ 속도 제한 (Rate Limiting)

### Rate Limiter 구현

```mermaid
flowchart TB
    START([요청 도착]) --> CHECK_RPM{RPM<br/>확인}

    CHECK_RPM -->|초과| WAIT_RPM["⏳ 대기<br/>가장 오래된 요청<br/>1분 경과 대기"]
    CHECK_RPM -->|가능| CHECK_TPM{TPM<br/>확인}

    WAIT_RPM --> CHECK_RPM

    CHECK_TPM -->|초과| WAIT_TPM["⏳ 대기<br/>토큰 확보<br/>시간 계산"]
    CHECK_TPM -->|가능| ACQUIRE["✅ 요청 허용"]

    WAIT_TPM --> CHECK_TPM

    ACQUIRE --> PROCESS[LLM 처리]

    PROCESS --> UPDATE["📊 기록 업데이트<br/>요청 시간<br/>토큰 수"]

    UPDATE --> END([완료])

    style START fill:#e1f5fe
    style END fill:#c8e6c9
    style WAIT_RPM fill:#ffcdd2
    style WAIT_TPM fill:#ffcdd2
    style ACQUIRE fill:#a5d6a7
```

```python
import asyncio
import time
from collections import deque
from typing import Deque

class StaticRateLimiter:
    def __init__(
        self,
        requests_per_minute: int,
        tokens_per_minute: int
    ):
        self.rpm = requests_per_minute
        self.tpm = tokens_per_minute

        # 요청 이력 (시간 스탬프)
        self.request_history: Deque[float] = deque()

        # 토큰 이력 (시간, 토큰 수) 쌍
        self.token_history: Deque[tuple[float, int]] = deque()

    async def acquire(self, tokens: int = 0):
        """
        속도 제한 준수까지 대기
        """
        now = time.time()

        # 1. 요청 제한 확인
        while len(self.request_history) >= self.rpm:
            # 가장 오래된 요청이 1분 이상 전인지 확인
            oldest_request = self.request_history[0]
            if now - oldest_request >= 60:
                self.request_history.popleft()
            else:
                # 대기 시간 계산
                wait_time = 60 - (now - oldest_request)
                await asyncio.sleep(wait_time)
                now = time.time()

        # 2. 토큰 제한 확인
        recent_tokens = sum(
            count for ts, count in self.token_history
            if now - ts < 60
        )

        if recent_tokens + tokens > self.tpm:
            # 대기 시간 계산
            available_in = self._get_time_for_tokens(tokens)
            await asyncio.sleep(available_in)
            now = time.time()

        # 3. 기록
        self.request_history.append(now)
        self.token_history.append((now, tokens))
```

### 이중 제한 시스템

```mermaid
flowchart TB
    A["요청"] --> RPM{"RPM 제한<br/>500/분"}
    A --> TPM{"TPM 제한<br/>100K/분"}

    RPM -->|초과| RPMBLOCK["요청 대기"]
    TPM -->|초과| TPMBLOCK["토큰 대기"]

    RPMBLOCK --> RPM
    TPMBLOCK --> TPM

    RPM -->|통과| BOTH["두 제한 모두<br/>통과 필요"]
    TPM -->|통과| BOTH

    BOTH --> EXEC["LLM 실행"]

    style RPM fill:#fff9c4
    style TPM fill:#e1bee7
    style RPMBLOCK fill:#ffcdd2
    style TPMBLOCK fill:#ffcdd2
    style EXEC fill:#c8e6c9
```

| 제한 유형 | 목적 | 일반적 값 |
|----------|------|-----------|
| **RPM** | 요청 빈도 제한 | 500/분 |
| **TPM** | 토큰 처리량 제한 | 100,000/분 |
| **TPD** | 일일 토큰 제한 | 10,000,000/일 |

### 대기 전략

```mermaid
flowchart TB
    A["속도 제한 도달"] --> B{전략 선택}

    B -->|고정| F["고정 대기<br/>⏸️ 60초"]
    B -->|선형| L["선형 대기<br/>📈 10s × 재시도"]
    B -->|지수| E["지수 대기<br/>📈 2^재시도"]
    B -->|지터| J["지터 추가<br/>🎲 무작위성"]

    F --> NEXT["다음 시도"]
    L --> NEXT
    E --> NEXT
    J --> NEXT

    style A fill:#ffcdd2
    style E fill:#c8e6c9
    style F fill:#fff9c4
    style L fill:#e1bee7
    style J fill:#e1f5fe
```

## 🔄 재시도 로직 (Retry Logic)

### 지수 백오프 (Exponential Backoff)

```mermaid
flowchart TB
    A["API 호출"] --> B{성공?}

    B -->|예| SUCCESS["✅ 결과 반환"]
    B -->|실패| ERROR{에러 유형}

    ERROR -->|RateLimit| RETRY["재시도 결정"]
    ERROR -->|Timeout| RETRY
    ERROR -->|기타| FAIL["❌ 실패"]

    RETRY --> C{재시도<br/>횟수 확인}

    C -->|최대 도달| FAIL
    C -->|가능| D["대기 시간 계산<br/>2^attempt × base"]

    D --> E["지터 추가<br/>× (0.5~1.0)"]

    E --> WAIT["⏳ 대기"]

    WAIT --> A

    style SUCCESS fill:#c8e6c9
    style FAIL fill:#ffcdd2
    style RETRY fill:#fff9c4
    style WAIT fill:#e1bee7
```

```python
async def execute_with_retry(
    llm_call: Callable,
    max_retries: int = 10,
    base_delay: float = 1.0,
    max_delay: float = 60.0,
    jitter: bool = True
):
    """
    지수 백오프를 사용한 재시도 실행
    """
    last_error = None

    for attempt in range(max_retries + 1):
        try:
            return await llm_call()

        except RateLimitError as e:
            if attempt == max_retries:
                raise MaxRetriesError(f"Max retries exceeded: {e}")

            # 대기 시간 계산
            delay = min(base_delay * (2 ** attempt), max_delay)

            # Jitter 추가 (불돌 방지)
            if jitter:
                import random
                delay = delay * (0.5 + random.random())

            logging.warning(f"Rate limited. Retry {attempt + 1}/{max_retries} after {delay:.2f}s")
            await asyncio.sleep(delay)

        except APITimeoutError as e:
            # 타임아웃은 더 긴 대기
            delay = min(base_delay * (2 ** attempt), max_delay * 2)
            await asyncio.sleep(delay)

        except Exception as e:
            # 기타 오류는 즉시 실패
            raise e
```

### 재시도 전략 비교

| 전략 | 대기 시간 공식 | 장점 | 단점 | 추천 상황 |
|------|---------------|------|------|-----------|
| `exponential_backoff` | 2^attempt | 빠른 회복 | 서버 부하 가능 | 일반적 |
| `incremental` | attempt × base | 예측 가능 | 느린 회복 | 안정적인 서비스 |
| `native_retry` | 제공자 의존 | 최적화됨 | 제공자마다 다름 | 단일 제공자 |
| `random` | 무작위 | 충돌 방지 | 느린 회복 | 분산 시스템 |

## 📊 토큰 관리

### 토큰 카운터 구조

```mermaid
flowchart TB
    A[텍스트] --> B{모델 식별}

    B --> C["cl100k_base<br/>GPT-4, 3.5"]
    B --> D["o200k_base<br/>GPT-4o"]
    B --> E["p50k_base<br/>Code"]

    C --> F["tiktoken 인코더"]
    D --> F
    E --> F

    F --> G["토큰 수 반환"]

    style A fill:#e1f5fe
    style F fill:#fff9c4
    style G fill:#c8e6c9
```

```python
class Tokenizer:
    # tiktoken 인코딩 매핑
    ENCODINGS = {
        "cl100k_base": {  # GPT-4, GPT-3.5-Turbo
            "models": ["gpt-4", "gpt-3.5-turbo", "text-embedding-3-*"]
        },
        "o200k_base": {  # GPT-4o
            "models": ["gpt-4o"]
        },
        "p50k_base": {  # Code models
            "models": ["code-davinci-*"]
        }
    }

    def __init__(self, model: str):
        self.encoding = self._get_encoding(model)

    def count(self, text: str) -> int:
        """텍스트의 토큰 수 반환"""
        return len(self.encoding.encode(text))

    def truncate(self, text: str, max_tokens: int) -> str:
        """토큰 제한으로 텍스트 자르기"""
        tokens = self.encoding.encode(text)
        truncated = tokens[:max_tokens]
        return self.encoding.decode(truncated)
```

### 모델별 인코딩

```mermaid
flowchart LR
    subgraph Models["모델별 인코딩"]
        GPT4["GPT-4<br/>cl100k_base"]
        GPT35["GPT-3.5<br/>cl100k_base"]
        GPT4O["GPT-4o<br/>o200k_base"]
        CODE["Code<br/>p50k_base"]
    end

    style GPT4 fill:#c8e6c9
    style GPT35 fill:#a5d6a7
    style GPT4O fill:#fff9c4
    style CODE fill:#e1bee7
```

## 🔧 비동기 처리

### 병렬 요청 아키텍처

```mermaid
flowchart TB
    A["프롬프트 목록<br/>100개"] --> B["세마포어<br/>concurrent=25"]

    B --> C["병렬 실행 시작"]

    C --> D1["요청 1-25"]
    C --> D2["요청 26-50"]
    C --> D3["요청 51-75"]
    C --> D4["요청 76-100"]

    D1 --> E["결결 수집"]
    D2 --> E
    D3 --> E
    D4 --> E

    E --> F["결과 반환"]

    style A fill:#e1f5fe
    style B fill:#ffcdd2
    style E fill:#c8e6c9
    style F fill:#a5d6a7
```

```python
async def parallel_llm_calls(
    prompts: list[str],
    llm: BaseLanguageModel,
    concurrent_requests: int = 25
) -> list[str]:
    """
    여러 프롬프트를 병렬로 처리
    """
    # 세마포어로 동시 요청 수 제한
    semaphore = asyncio.Semaphore(concurrent_requests)

    async def process(prompt: str):
        async with semaphore:
            return await llm.execute(prompt)

    # 병렬 실행
    tasks = [process(p) for p in prompts]
    results = await asyncio.gather(*tasks, return_exceptions=True)

    # 예외 처리
    final_results = []
    for result in results:
        if isinstance(result, Exception):
            logging.error(f"LLM call failed: {result}")
            final_results.append("")  # 실패 시 빈 문자열
        else:
            final_results.append(result)

    return final_results
```

### Async 모드 선택

| 모드 | 설명 | 사용 사례 |
|------|------|----------|
| `asyncio` | 진정한 비동기 I/O | I/O 바운드 작업 |
| `threaded` | 스레드 풀에서 실행 | CPU 바운드 작업 |

## 🎓 고급 기법

### 스트리밍 응답

```mermaid
flowchart LR
    A["요청"] --> B["스트림 시작"]

    B --> C1["청크 1"]
    B --> C2["청크 2"]
    B --> C3["청크 N"]

    C1 --> D["실시간 출력"]
    C2 --> D
    C3 --> D

    style A fill:#e1f5fe
    style D fill:#c8e6c9
```

```python
async def stream_execute(
    self,
    prompt: str
) -> AsyncIterator[str]:
    """
    스트리밍 응답 생성
    """
    async with self.client.chat.completions.create(
        model=self.model,
        messages=[{"role": "user", "content": prompt}],
        stream=True
    ) as response:
        async for chunk in response:
            if chunk.choices[0].delta.content:
                yield chunk.choices[0].delta.content
```

### 함수 호출 (Function Calling)

```mermaid
flowchart TB
    A["텍스트"] --> B["LLM + Tools"]

    B --> C{"함수 호출<br/>필요?"}

    C -->|예| D["Tool 실행"]
    C -->|아니오| E["직접 응답"]

    D --> F["결과 포함<br/>최종 응답"]

    style A fill:#e1f5fe
    style D fill:#fff9c4
    style F fill:#c8e6c9
```

## 📊 성능 최적화

### 최적화 전략

```mermaid
mindmap
    root((최적화))
        배치 처리
            한 번의 API 호출
            여러 청크 처리
        캐싱
            프롬프트 템플릿
            LLM 응답
        병렬 처리
            asyncio
            세마포어 제어
        토큰 최적화
            max_tokens 제어
            효율적 프롬프트
```

## 🔗 관련 컴포넌트

- [[Index Module]]: 인덱싱 중 LLM 사용
- [[Query Module]]: 쿼리 처리 중 LLM 사용
- [[Storage Module]]: 응답 캐싱

## 💡 비용 최적화 팁

1. **캐싱 활성화**: 동일 요청 재처리 방지
2. **배치 처리**: 여러 요청을 한 번에 처리
3. **모델 선택**: 작업에 맞는 모델 선택
4. **토큰 제한**: `max_tokens`로 출력 길이 제어

---
*See also: [[Language Model Module]], [[Index Module]], [[Query Module]]*
