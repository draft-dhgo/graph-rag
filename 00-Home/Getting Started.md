---
title: GraphRAG 시작하기
tags:
  - tutorial
  - quick-start
  - installation
created: 2025-01-12
type: guide
links:
  - [[GraphRAG란 무엇인가?]]
  - [[Home]]
  - [[Configuration Module]]
---

# GraphRAG 시작하기

이 가이드를 통해 GraphRAG를 빠르게 시작할 수 있습니다.

## 사전 요구사항

### 필수 요구사항
- **Python**: 3.10 ~ 3.13
- **운영체제**: Windows, macOS 또는 Linux
- **LLM 액세스**: OpenAI API 키 또는 Azure OpenAI 액세스

### 선택 사항
- Azure 계정 (Azure 서비스 통합용)
- GPU (로컬 임베딩용)

## 설치

### 1단계: 새 환경 만들기

```bash
# venv 사용
python -m venv graphrag-env
source graphrag-env/bin/activate  # Windows의 경우: graphrag-env\Scripts\activate

# conda 사용
conda create -n graphrag python=3.11
conda activate graphrag
```

### 2단계: GraphRAG 설치

```bash
pip install graphrag
```

### 3단계: 설치 확인

```bash
graphrag --version
```

## 빠른 시작

### 1단계: 프로젝트 초기화

`init` 명령은 기본 구성으로 새 GraphRAG 프로젝트를 만듭니다:

```bash
graphrag init --root ./my-project
```

이 명령은 다음을 생성합니다:
```
my-project/
├── .env              # 환경변수 템플릿
├── settings.yaml     # 구성 파일
└── prompts/          # 프롬프트 템플릿 디렉토리
    └── *.txt         # 기본 프롬프트 템플릿
```

### 2단계: 환경 구성

`.env` 파일을 자격증명으로 편집하세요:

```bash
# OpenAI API 키
OPENAI_API_KEY=sk-...

# 또는 Azure OpenAI
AZURE_OPENAI_API_KEY=...
AZURE_OPENAI_API_BASE=https://...
AZURE_OPENAI_API_VERSION=2024-02-01
AZURE_OPENAI_DEPLOYMENT_NAME=...
```

### 3단계: 데이터 준비

소스 문서를 프로젝트 디렉토리에 배치하세요:

```bash
mkdir -p ./my-project/input
cp your-documents/*.txt ./my-project/input/
```

지원되는 형식:
- 일반 텍스트 파일 (`.txt`)
- CSV 파일 (`.csv`)
- JSON 파일 (`.json`)

### 4단계: 지식 그래프 빌드

```bash
cd my-project
graphrag index --root . --verbose
```

이 프로세스는 다음을 수행합니다:
1. 문서를 로드하고 청크로 분할합니다
2. LLM을 사용하여 엔티티와 관계를 추출합니다
3. [[Leiden Algorithm]]으로 커뮤니티를 감지합니다
4. 커뮤니티 요약을 생성합니다
5. 벡터 임베딩을 만듭니다

출력은 기본적으로 `./output/`에 저장됩니다.

### 5단계: 지식 그래프 쿼리

#### 로컬 검색
엔티티에 대한 구체적인 질문에 적합합니다:

```bash
graphrag query --root . --method local \
  --query "AI 연구에 참여하는 사람은 누구인가?"
```

#### 전체 검색
광범위한 주제 질문에 적합합니다:

```bash
graphrag query --root . --method global \
  --query "이 문서들의 주요 테마는 무엇인가?"
```

#### DRIFT 검색
복잡한 다중 홉 질문에 적합합니다:

```bash
graphrag query --root . --method drift \
  --query "서로 다른 개념들이 어떻게 관련되어 있는가?"
```

## 구성 기본 사항

### 주요 구성 옵션

`settings.yaml`을 편집하여 사용자 정의하세요:

```yaml
# 모델 구성
models:
  default_chat_model:
    type: azure_openai_chat  # 또는 openai_chat
    model: gpt-4-turbo-preview
    api_key: ${AZURE_OPENAI_API_KEY}
    api_base: ${AZURE_OPENAI_API_BASE}
    api_version: 2024-02-01

# 입력 구성
input:
  type: file
  file_type: text  # csv, json, text
  file_pattern: "input/*.txt"

# 청킹 구성
chunks:
  size: 1200        # 청크당 토큰 크기
  overlap: 100      # 청크 간 중복

# 벡터 저장소
vector_store:
  type: lancedb     # lancedb, azure_ai_search, cosmos_db
  db_uri: ./output/lancedb
```

모든 옵션은 [[Configuration Module]]를 참조하세요.

## 프롬프트 튜닝

도메인에 대한 추출 정확도를 높이세요:

```bash
graphrag prompt-tune --root . --domain "space science" --language english
```

이 명령은 `prompts/` 디렉토리에 도메인별 프롬프트를 생성합니다.

## 출력 이해하기

### 출력 구조

```
output/
├── create_final_documents.parquet      # 처리된 문서
├── create_final_text_units.parquet     # 텍스트 청크
├── create_final_entities.parquet       # 추출된 엔티티
├── create_final_relationships.parquet  # 엔티티 관계
├── create_final_communities.parquet    # 감지된 커뮤니티
├── create_final_community_reports.parquet  # 커뮤니티 요약
└── lancedb/                            # 벡터 데이터베이스
```

### 결과 보기

Python을 사용하여 결과를 분석하세요:

```python
import pandas as pd

# 엔티티 로드
entities = pd.read_parquet("output/create_final_entities.parquet")
print(entities.head())

# 커뮤니티 로드
communities = pd.read_parquet("output/create_final_communities.parquet")
print(communities.head())

# 커뮤니티 보고서 로드
reports = pd.read_parquet("output/create_final_community_reports.parquet")
print(reports[["community", "summary"]].head())
```

## 일반 워크플로우

### 새 문서로 인덱스 업데이트

```bash
# input/에 새 문서 추가
graphrag update --root . --verbose
```

### 빠른 파이프라인 사용 (더 빠른 처리)

```bash
graphrag index --root . --method fast
```

### 다른 커뮤니티 수준으로 쿼리

```bash
# 수준 0: 루트 (모든 엔티티)
graphrag query --root . --method global --community-level 0

# 수준 2: 중간 세부 수준 (기본값)
graphrag query --root . --method global --community-level 2
```

## 문제 해결

### 일반적인 문제

#### API 키 오류
```
ApiKeyMissingError: API key not found
```
**해결 방법**: `.env` 파일에 올바른 자격증명이 포함되어 있는지 확인하세요.

#### 메모리 문제
```
MemoryError: Unable to allocate array
```
**해결 방법**: 설정에서 `concurrent_requests` 또는 `chunks.size`를 줄이세요.

#### 속도 제한
```
RateLimitError: Too many requests
```
**해결 방법**: 설정에서 `requests_per_minute` 또는 `tokens_per_minute`를 줄이세요.

## 다음 단계

- [[Configuration Module]] - 모든 구성 옵션
- [[Indexing API]] - 프로그래밍 방식 액세스
- [[Index Module]] - 인덱싱 작동 방식
- [[Query Module]] - 검색 방법 설명

## 고급 사용법

### Python API 사용

```python
from graphrag.api import initialize_project, build_index, local_search

# 초기화
initialize_project("./my-project")

# 인덱스 빌드
await build_index("./my-project/settings.yaml")

# 쿼리
result = await local_search(
    "핵심 테마는 무엇인가?",
    "./my-project/settings.yaml"
)
print(result)
```

자세한 내용은 [[Indexing API]]를 참조하세요.

---
*참조: [[GraphRAG란 무엇인가?]], [[Home]], [[Configuration Module]]*
