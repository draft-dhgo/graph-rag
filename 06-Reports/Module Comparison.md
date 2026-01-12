---
title: GraphRAG 모듈 비교
tags:
  - report
  - comparison
  - modules
created: 2025-01-12
type: report
links:
  - [[Architecture Overview]]
  - [[Summary Report]]
  - [[Index Module]]
  - [[Query Module]]
---

# GraphRAG 모듈 비교 보고서

## 모듈 의존성 그래프

```
                    ┌─────────────┐
                    │     CLI     │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │     API     │
                    └──────┬──────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
┌───────▼───────┐ ┌────────▼────────┐ ┌─────▼──────┐
│  [[Configuration Module]] │  [[Index Module]]   │  │ [[Query Module]] │
└───────────────┘ └────────┬────────┘ └─────┬──────┘
                         │                │
        ┌────────────────┼────────────────┤
        │                │                │
┌───────▼──────┐  ┌──────▼──────┐  ┌─────▼─────┐
│Language Model│  │Data Model   │  │Vector Store│
└──────────────┘  └─────────────┘  └───────────┘
        │                │
┌───────▼────────────────▼────────┐
│      [[Storage Module]]         │
└─────────────────────────────────┘
```

## 모듈 비교 표

| 모듈 | 주요 책임 | 입력 | 출력 | 의존성 |
|--------|------------------------|-------|--------|--------------|
| [[Configuration Module]] | 구성 관리 | YAML/JSON 파일 | GraphRagConfig | Pydantic |
| [[Entity]] | 타입 정의 | N/A | 데이터 클래스 | None |
| [[Index Module]] | 명령 인터페이스 | CLI 인자 | API 호출 | Typer, 모든 모듈 |
| [[Indexing API]] | 공개 API 계층 | Config | Results | 모든 모듈 |
| [[Index Module]] | 파이프라인 실행 | Documents | Knowledge graph | config, llm, storage |
| [[Query Module]] | 검색 작업 | Query text | Search results | config, llm, storage |
| [[Language Model Module]] | LLM 통합 | Prompts | LLM responses | OpenAI, LiteLLM |
| [[Storage Module]] | 데이터 지속성 | Data | Stored data | Azure SDK |
| [[Storage Module]] | 벡터 작업 | Embeddings | Search results | LanceDB, Azure |
| [[Storage Module]] | 응답 캐싱 | Keys | Cached values | None |
| [[Configuration Module]] | 프롬프트 템플릿 | Config | Prompt strings | None |

## 복잡도 분석

### 순환 복잡도 (추정)

| 모듈 | 낮음 | 중간 | 높음 | 매우 높음 |
|--------|-----|--------|------|-----------|
| Configuration Module | ✓ | | | |
| [[Entity]] | ✓ | | | |
| [[Index Module]] | ✓ | | | |
| API | | ✓ | | |
| [[Storage Module]] | ✓ | | | |
| [[Configuration Module]] | ✓ | | | |
| Tokenizer | | ✓ | | |
| [[Storage Module]] | | ✓ | | |
| [[Storage Module]] | | ✓ | | |
| [[Language Model Module]] | | | ✓ | |
| [[Query Module]] | | | ✓ | |
| [[Index Module]] | | | | ✓ |

### 근거

**낮은 복잡도:**
- [[Entity]]: 간단한 데이터클래스 정의
- [[Storage Module]]: 직관적인 캐싱 로직
- [[Configuration Module]]: 정적 문자열 템플릿
- [[Index Module]]: 최소한의 로직이 포함된 명령 라우팅
- [[Configuration Module]]: Pydantic을 통한 구성 파싱

**중간 복잡도:**
- [[Storage Module]]: 여러 백엔드 구현
- [[Storage Module]]: 여러 구현이 있는 추상 인터페이스
- Tokenizer: 여러 인코딩을 지원하는 토큰 계산
- API: 여러 모듈에 대한 파사드 패턴

**높은 복잡도:**
- [[Language Model Module]]: 여러 제공업체, 재시도 로직, 속도 제한
- [[Query Module]]: 컨텍스트 빌더가 있는 4가지 다른 검색 알고리즘

**매우 높은 복잡도:**
- [[Index Module]]: 9개 이상의 워크플로우 단계, 병렬 처리, 오류 복구가 포함된 전체 파이프라인

## 성능 특성

### 실행 시간 (일반적)

| 작업 | 시간 | 병목 지점 |
|-----------|------|------------|
| 구성 로딩 | <1초 | 파일 I/O |
| 텍스트 청킹 | 빠름 | CPU |
| 엔티티 추출 | 느림 | LLM API |
| 커뮤니티 감지 | 중간 | 그래프 알고리즘 |
| 임베딩 생성 | 느림 | LLM/Vector API |
| [[Local Search]] | 중간 | LLM + 벡터 검색 |
| [[Global Search]] | 중간 | LLM + 집계 |
| 인덱싱 (전체 파이프라인) | 매우 느림 | 여러 LLM 호출 |

### 리소스 사용량

| 모듈 | CPU | 메모리 | I/O | 네트워크 |
|--------|-----|--------|-----|---------|
| Configuration Module | 낮음 | 낮음 | 낮음 | 없음 |
| [[Index Module]] | 낮음 | 낮음 | 낮음 | 없음 |
| [[Entity]] | 낮음 | 중간 | 낮음 | 없음 |
| [[Storage Module]] | 낮음 | 중간 | 높음 | 선택사항 |
| [[Storage Module]] | 낮음 | 높음 | 높음 | 선택사항 |
| [[Language Model Module]] | 낮음 | 낮음 | 낮음 | **높음** |
| [[Index Module]] | 중간 | **높음** | 중간 | **높음** |
| [[Query Module]] | 중간 | 중간 | 중간 | **높음** |

## 결합도 분석

### 강한 결합 (높은 의존성)

| 모듈 | 의존 대상 | 영향 |
|--------|------------|--------|
| [[Index Module]] | [[Language Model Module]], [[Storage Module]], [[Entity]] | 높음 |
| [[Query Module]] | [[Language Model Module]], [[Storage Module]], [[Storage Module]] | 높음 |
| [[Index Module]] | API, 모든 모듈 | 중간 |

### 느슨한 결합 (낮은 의존성)

| 모듈 | 의존 대상 | 영향 |
|--------|------------|--------|
| [[Entity]] | None | 없음 |
| [[Storage Module]] | None | 없음 |
| [[Configuration Module]] | [[Entity]] | 낮음 |
| [[Configuration Module]] | [[Entity]] | 낮음 |

## 수정 권장사항

### 안전하게 수정 가능

| 모듈 | 이유 |
|--------|--------|
| [[Configuration Module]] | 템플릿 문자열, 로직 없음 |
| [[Index Module]] | 명령 정의 |
| [[Configuration Module]] | 기본값 및 검증 |

### 주의하여 수정

| 모듈 | 이유 |
|--------|--------|
| [[Entity]] | 전체 시스템에 영향 |
| [[Storage Module]] | 모든 I/O 작업에 영향 |
| [[Storage Module]] | 성능과 정확성에 영향 |

### 깊은 이해가 필요

| 모듈 | 이유 |
|--------|--------|
| [[Index Module]] | 복잡한 파이프라인 오케스트레이션 |
| [[Query Module]] | 여러 검색 알고리즘 |
| [[Language Model Module]] | 비동기 패턴, 속도 제한 |
| [[Storage Module]] | 벡터 유사도 계산 |

## 확장성 지점

### 플러그인 기능

| 확장 지점 | 인터페이스 | 예시 |
|-----------------|-----------|----------|
| 스토리지 백엔드 | `PipelineStorage` | S3, GCS, 사용자 정의 |
| LLM 제공업체 | `BaseLanguageModel` | Anthropic, 로컬 모델 |
| 벡터 스토어 | `BaseVectorStore` | Pinecone, Weaviate |
| 임베딩 모델 | `TextEmbedder` | 사용자 정의 임베딩 |
| 파이프라인 단계 | `WorkflowFunction` | 사용자 정의 작업 |
| 콜백 핸들러 | `WorkflowCallbacks` | 모니터링, 로깅 |

---
*참고: [[Architecture Overview]], [[Summary Report]], [[Module Comparison]]*
