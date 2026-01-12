---
title: GraphRAG 기술 스택
tags:
  - architecture
  - dependencies
  - technology
created: 2025-01-12
type: reference
links:
  - [[Architecture Overview]]
  - [[GraphRAG란 무엇인가?]]
---

# GraphRAG 기술 스택

이 문서는 GraphRAG에서 사용하는 전체 기술 스택을 다루며, 핵심 의존성, 클라우드 서비스, 개발 도구를 포함합니다.

## 핵심 요구사항

### Python 버전
- **지원**: Python 3.10, 3.11, 3.12, 3.13
- **권장**: Python 3.11 또는 3.12

### 설치
```bash
pip install graphrag
```

## 핵심 의존성

### CLI 프레임워크

| 라이브러리 | 버전 | 목적 |
|---------|---------|---------|
| **Typer** | 최신 | 명령줄 인터페이스 프레임워크 |
| **Rich** | 최신 | 터미널 포맷팅 및 진행률 표시줄 |

### 데이터 처리

| 라이브러리 | 목적 | GraphRAG 내 사용 |
|---------|---------|-------------------|
| **Pandas** | 표 형식 데이터 | DataFrame 작업, Parquet I/O |
| **PyArrow** | 컬럼 기반 데이터 | Parquet 형식, 효율적 저장 |
| **NumPy** | 수치 연산 | 벡터 연산 |

### 그래프 처리

| 라이브러리 | 목적 | GraphRAG 내 사용 |
|---------|---------|-------------------|
| **NetworkX** | 그래프 작업 | 그래프 데이터 구조, 알고리즘 |
| **Graspologic** | 그래프 분석 | Leiden 커뮤니티 탐지, 계층 구조 |
| **UMAP-Learn** | 차원 축소 | 그래프 레이아웃, 시각화 |

### NLP 및 토큰화

| 라이브러리 | 목적 | GraphRAG 내 사용 |
|---------|---------|-------------------|
| **NLTK** | 자연어 처리 | 문장 토큰화, 청킹 |
| **SpaCy** | NLP 전처리 | 선택적 NLP 기반 추출 |
| **tiktoken** | 토큰 수 계산 | OpenAI 토큰 인코딩 |

### 검증

| 라이브러리 | 목적 | GraphRAG 내 사용 |
|---------|---------|-------------------|
| **Pydantic** | 데이터 검증 | 설정 모델, 타입 안전성 |

### 비동기 및 동시성

| 라이브러리 | 목적 | GraphRAG 내 사용 |
|---------|---------|-------------------|
| **asyncio** | 비동기 I/O | 동시 작업 |
| **aiofiles** | 비동기 파일 작업 | 비차단 파일 I/O |

## LLM 제공업체

### OpenAI

| 구성 요소 | 목적 |
|-----------|---------|
| **openai** | 공식 OpenAI Python SDK |
| **Models** | GPT-4, GPT-4 Turbo, GPT-4o, GPT-3.5-Turbo |
| **Embeddings** | text-embedding-3-small, text-embedding-3-large |

### Azure OpenAI

| 구성 요소 | 목적 |
|-----------|---------|
| **openai** (Azure) | Azure OpenAI Service 연동 |
| **azure-identity** | Azure 인증(관리 ID) |
| **Models** | OpenAI와 동일, Azure 호스팅 |

### LiteLLM

| 구성 요소 | 목적 |
|-----------|---------|
| **litellm** | 100개 이상의 LLM 제공업체를 위한 통합 인터페이스 |
| **Providers** | Anthropic, Google, Cohere, Hugging Face 등 |

## 저장소 백엔드

### 로컬 저장소

| 구성 요소 | 목적 |
|-----------|---------|
| **aiofiles** | 비동기 파일 작업 |
| **Pathlib** | 경로 조작 |

### Azure 저장소

| 구성 요소 | 목적 |
|-----------|---------|
| **azure-storage-blob** | Azure Blob Storage 연동 |
| **azure-cosmos** | Azure Cosmos DB 연동 |
| **azure-identity** | Azure 인증 |

## 벡터 데이터베이스

### LanceDB

| 구성 요소 | 목적 |
|-----------|---------|
| **lancedb** | 로컬 벡터 데이터베이스 |
| **PyArrow** | 기본 데이터 형식 |

### Azure AI Search

| 구성 요소 | 목적 |
|-----------|---------|
| **azure-search-documents** | Azure Cognitive Search 연동 |

### Cosmos DB 벡터 검색

| 구성 요소 | 목적 |
|-----------|---------|
| **azure-cosmos** | Cosmos DB의 네이티브 벡터 검색 |

## 개발 도구

### 테스트

| 라이브러리 | 목적 |
|---------|---------|
| **pytest** | 테스트 프레임워크 |
| **pytest-asyncio** | 비동기 테스트 지원 |
| **pytest-cov** | 커버리지 보고 |

### 코드 품질

| 라이브러리 | 목적 |
|---------|---------|
| **Ruff** | 린팅 및 포맷팅 |
| **Pyright** | 타입 검사 |
| **black** | 코드 포맷팅(선택 사항) |

### 작업 자동화

| 라이브러리 | 목적 |
|---------|---------|
| **poethepoet** | 작업 실행기 |

## 파일 형식

| 형식 | 라이브러리 | 사용 |
|--------|---------|-------|
| **Parquet** | PyArrow | 구조화된 데이터 저장 |
| **JSON** | 표준 | 설정, 메타데이터 |
| **YAML** | PyYAML | 설정 파일 |
| **GraphML** | NetworkX | 선택적 그래프 내보내기 |
| **Markdown** | - | 보고서, 문서화 |

## 클라우드 서비스

### Azure 서비스

| 서비스 | 목적 | 사용 |
|---------|---------|-------|
| **Azure OpenAI** | LLM 서비스 | 채팅 및 임베딩 모델 |
| **Azure Blob Storage** | 객체 저장소 | 문서 저장, 출력 |
| **Azure Cosmos DB** | NoSQL 데이터베이스 | 구조화된 데이터, 벡터 검색 |
| **Azure AI Search** | 검색 서비스 | 벡터 유사도 검색 |
| **Azure Managed Identity** | 인증 | 자격 증명 없는 인증 |

## 선택적 의존성

### NLP 향상
- **SpaCy**: 고급 NLP 전처리
- **NLTK data**: 문장 토큰화 모델

### 시각화
- **Matplotlib**: 플로팅 및 시각화
- **Plotly**: 대화형 그래프

## 의존성 트리

```
graphrag
├── typer (CLI)
├── pydantic (검증)
├── pandas (데이터)
│   └── pyarrow (Parquet)
├── networkx (그래프)
├── graspologic (커뮤니티 탐지)
│   └── python-igraph (선택 사항)
├── umap-learn (차원 축소)
├── nltk (NLP)
├── tiktoken (토큰화)
├── openai (LLM)
├── litellm (다중 제공업체)
├── azure-storage-blob (Azure 저장소)
├── azure-cosmos (Cosmos DB)
├── azure-search-documents (Azure 검색)
├── azure-identity (Azure 인증)
├── aiofiles (비동기 I/O)
├── lancedb (벡터 DB)
└── rich (터미널 UI)
```

## 보안

### 인증 라이브러리

| 라이브러리 | 목적 |
|---------|---------|
| **azure-identity** | Azure 인증(관리 ID, CLI) |
| **python-dotenv** | 환경 변수 관리 |

## 성능

### 최적화 라이브러리

| 라이브러리 | 목적 |
|---------|---------|
| **numpy** | 벡터화 연산 |
| **pandas** | 효율적인 데이터 조작 |
| **asyncio** | 동시 처리 |
| **aiofiles** | 비차단 I/O |

## 플랫폼 지원

### 운영체제
- Linux (Ubuntu, Debian 등)
- macOS (Intel 및 Apple Silicon)
- Windows (WSL2 권장)

### Python 구현
- CPython (권장)
- PyPy (테스트되지 않음)

## 버전 제약조건

```toml
# 주요 버전 제약조건 (단순화)
python = ">=3.10,<3.14"
pydantic = ">=2.0"
pandas = ">=1.0"
openai = ">=1.0"
azure-storage-blob = ">=12.0"
azure-cosmos = ">=4.0"
lancedb = ">=0.5"
```

## 호환성 매트릭스

| Python | 3.10 | 3.11 | 3.12 | 3.13 |
|--------|------|------|------|------|
| GraphRAG | ✅ | ✅ | ✅ | ✅ |

| LLM 제공업체 | 지원 |
|--------------|---------|
| OpenAI | ✅ 전체 |
| Azure OpenAI | ✅ 전체 |
| LiteLLM | ✅ 전체 |

| 저장소 | 지원 |
|---------|---------|
| 로컬 파일 | ✅ 전체 |
| Azure Blob | ✅ 전체 |
| Cosmos DB | ✅ 전체 |

| 벡터 저장소 | 지원 |
|-------------|---------|
| LanceDB | ✅ 전체 |
| Azure AI Search | ✅ 전체 |
| Cosmos DB | ✅ 전체 |

---
*참고: [[Architecture Overview]], [[Configuration Module]], [[Getting Started]]*
