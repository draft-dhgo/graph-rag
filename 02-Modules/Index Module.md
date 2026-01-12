---
title: GraphRAG 인덱스 모듈
tags:
  - module
  - indexing
  - pipeline
created: 2025-01-12
type: documentation
links:
  - [[Architecture Overview]]
  - [[Data Flow]]
  - [[Query Module]]
  - [[Entity Extraction Deep Dive]]
  - [[Leiden Algorithm]]
---

# GraphRAG 인덱스 모듈

**인덱스 모듈** (`/graphrag/index/`)은 정형화되지 않은 텍스트 문서를 처리하여 임베딩이 포함된 구조화된 지식 그래프로 변환하는 핵심 구성 요소입니다.

## 📋 개요

인덱싱 파이프라인은 자연어 처리, 그래프 이론, 머신러닝을 결합하여 텍스트 데이터에서 엔티티, 관계, 커뮤니티를 추출하는 정교한 일련의 워크플로우를 구현합니다.

## 🏗️ 파이프라인 아키텍처

### 파이프라인 팩토리

`PipelineFactory` 클래스는 워크플로우 등록과 파이프라인 생성을 관리합니다:

```python
class PipelineFactory:
    workflows: ClassVar[dict[str, WorkflowFunction]] = {}
    pipelines: ClassVar[dict[str, list[str]]] = {}

    @classmethod
    def create_pipeline(cls, config: GraphRagConfig, method: IndexingMethod) -> Pipeline:
        workflows = config.workflows or cls.pipelines.get(method, [])
        return Pipeline([(name, cls.workflows[name]) for name in workflows])
```

### 파이프라인 메서드

| 메서드 | 설명 | 사용 사례 |
|--------|-------------|----------|
| **Standard** | 전체 LLM 기반 추출 | 프로덕션, 최고 품질 |
| **Fast** | NLP + LLM 하이브리드 | 개발, 더 빠른 처리 |
| **StandardUpdate** | 증분 표준 업데이트 | 새 문서 추가 |
| **FastUpdate** | 증분 빠른 업데이트 | 빠른 업데이트 |

## 🔄 표준 파이프라인 워크플로우

```
1. create_base_text_units      -> 초기 텍스트 청크 생성
2. create_final_documents      -> 문서 처리
3. extract_graph              -> 엔티티 및 관계 추출
4. finalize_graph             -> 그래프 처리 및 정제
5. extract_covariates         -> 추가 속성 추출
6. create_communities         -> 커뮤니티 감지
7. create_final_text_units    -> 텍스트 유닛 최종화
8. create_community_reports   -> 커뮤니티 요약 생성
9. generate_text_embeddings   -> 텍스트 임베딩 생성
```

## 📝 핵심 작업

### 1. 텍스트 청킹

**위치**: `/graphrag/index/operations/chunk_text/`

**목적**: 큰 문서를 관리 가능한 청크로 분할합니다.

**전략**:
- **토큰 기반**: 정확한 토큰 수를 위해 tiktoken 사용
- **문장 기반**: 문장 경계를 위해 NLTK 사용

**구성**:
```yaml
chunks:
  size: 1200        # 청크 크기 (토큰 단위)
  overlap: 100      # 청크 간 중복
  strategy: tokens  # tokens 또는 sentences
```

**출력**: [[Text Unit]] 객체

### 2. 그래프 추출

**위치**: `/graphrag/index/operations/extract_graph/`

**목적**: LLM을 사용하여 엔티티와 관계를 추출합니다.

**프로세스**:
1. 각 텍스트 청크를 개별적으로 처리
2. 구조화된 프롬프트와 함께 LLM 사용
3. 유형과 설명이 포함된 엔티티 추출
4. 가중치가 포함된 관계 추출
5. 모든 청크의 추출 결과 병합

**엔티티 유형**:
- `organization` - 회사, 기관
- `person` - 사람, 개인
- `geo` - 위치, 장소
- `event` - 이벤트, 사건
- `custom` - 도메인 특화 유형

**출력**: [[Entity]] 및 [[Relationship]] 객체

### 3. 그래프 최종화

**목적**: 추출된 그래프를 정제, 검증, 준비합니다.

**작업**:
- 중복 엔티티/관계 제거
- 그래프 구조 검증
- 노드 차수 및 메트릭 계산
- 약한 연결 pruning

### 4. 커뮤니티 감지

**위치**: `/graphrag/index/operations/cluster_graph.py`

**목적**: [[Leiden Algorithm]]을 사용하여 커뮤니티를 식별합니다.

**알고리즘**: graspologic을 통한 계층적 Leiden 클러스터링

**기능**:
- 다중 레벨 커뮤니티 구조
- 부모-자식 관계
- 다양한 세분성 수준에서 계층 보존

**구성**:
```yaml
cluster_graph:
  max_cluster_size: 50
  use_lcc: true  # 최대 연결 컴포넌트 사용
```

**출력**: 계층 구조가 포함된 [[Community]] 객체

### 5. 커뮤니티 리포트

**위치**: `/graphrag/index/operations/summarize_communities/`

**목적**: 각 커뮤니티에 대한 자연어 요약을 생성합니다.

**프로세스**:
1. 각 커뮤니티의 엔티티 컨텍스트 수집
2. 요약을 위해 LLM에 전송
3. 인사이트와 주요 관계 생성
4. 요약과 전체 콘텐츠 모두 저장

**출력**: [[Community Report]] 객체

### 6. 텍스트 임베딩

**위치**: `/graphrag/index/operations/embed_text/`

**목적**: 의미 검색을 위한 벡터 임베딩을 생성합니다.

**임베딩 대상**:
- 텍스트 청크
- 엔티티 설명
- 커뮤니티 컨텍스트

**모델**:
- OpenAI: text-embedding-3-small/large
- 커스텀: 모든 임베딩 함수

**출력**: [[Storage Module]]에 저장된 벡터

## 🚀 업데이트 파이프라인

기존 인덱스에 대한 증분 업데이트:

```
1. update_final_documents      -> 새 문서 추가
2. update_entities_relationships -> 새 그래프 데이터 병합
3. update_text_units           -> 텍스트 유닛 업데이트
4. update_covariates           -> 속성 업데이트
5. update_communities          -> 커뮤니티 업데이트
6. update_community_reports    -> 요약 업데이트
7. update_text_embeddings      -> 임베딩 업데이트
8. update_clean_state          -> 임시 상태 정리
```

## 📊 입력/출력

### 입력
- 문서 (CSV, JSON, TXT)
- 구성
- 기존 인덱스 (업데이트용)

### 출력
| 파일 | 내용 |
|------|----------|
| `create_final_documents.parquet` | 처리된 문서 |
| `create_final_text_units.parquet` | 텍스트 청크 |
| `create_final_entities.parquet` | 추출된 엔티티 |
| `create_final_relationships.parquet` | 엔티티 관계 |
| `create_final_communities.parquet` | 감지된 커뮤니티 |
| `create_final_community_reports.parquet` | 커뮤니티 요약 |
| `create_final_covariates.parquet` | 클레임/메타데이터 |

## ⚙️ 구성

### 파이프라인 구성

```yaml
# 워크플로우 선택
workflows:
  - create_base_text_units
  - extract_graph
  - create_communities
  # ... 등

# 또는 사전 정의된 메서드 사용
method: standard  # standard, fast, standard_update, fast_update
```

### 추출 구성

```yaml
extract_graph:
  entity_types:
    - organization
    - person
    - geo
    - event
  max_gleanings: 1  # 청크당 추출 패스 수
```

### 커뮤니티 구성

```yaml
cluster_graph:
  max_cluster_size: 50
  use_lcc: true
  seed: 42
```

## 🔗 관련 구성 요소

- [[Entity]] - 엔티티 데이터 모델
- [[Relationship]] - 관계 데이터 모델
- [[Community]] - 커뮤니티 데이터 모델
- [[Text Unit]] - 텍스트 청크 데이터 모델
- [[Leiden Algorithm]] - 커뮤니티 감지 알고리즘
- [[Query Module]] - 인덱스된 데이터 검색

## 📚 주요 파일

| 파일 | 용도 |
|------|---------|
| `/graphrag/index/workflows/factory.py` | 파이프라인 팩토리 |
| `/graphrag/index/operations/chunk_text/chunk_text.py` | 텍스트 청킹 |
| `/graphrag/index/operations/extract_graph/` | 그래프 추출 |
| `/graphrag/index/operations/cluster_graph.py` | 커뮤니티 감지 |
| `/graphrag/index/operations/summarize_communities/` | 리포트 생성 |
| `/graphrag/index/operations/embed_text/embed_text.py` | 텍스트 임베딩 |
| `/graphrag/index/run/run_pipeline.py` | 파이프라인 실행 |

---
*참고: [[Architecture Overview]], [[Query Module]], [[Configuration Module]]*
