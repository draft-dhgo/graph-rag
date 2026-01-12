---
title: GraphRAG 스토리지 구성
tags:
  - configuration
  - storage
  - database
  - cloud
created: 2025-01-13
type: documentation
links:
  - [[Configuration Reference]]
  - [[Storage Module]]
  - [[Index Module]]
  - [[Query Module]]
---

# GraphRAG 스토리지 구성

**스토리지 구성** 문서는 GraphRAG의 데이터 저장 옵션을 설정하는 방법을 설명합니다.

## 📋 개요

GraphRAG는 다양한 스토리지 백엔드를 지원합니다:

- **파일 시스템**: 로컬 디스크 기반 저장
- **Azure Blob Storage**: 클라우드 객체 저장
- **Azure Cosmos DB**: NoSQL 데이터베이스

## 📁 스토리지 타입

### 1. 파일 스토리지 (file)

로컬 파일 시스템에 데이터를 저장합니다.

```yaml
output:
  type: file
  base_dir: ./output

cache:
  type: file
  base_dir: ./cache

reporting:
  type: file
  base_dir: ./reports
```

### 2. Blob Storage (blob)

Azure Blob Storage에 데이터를 저장합니다.

```yaml
output:
  type: blob
  connection_string: ${AZURE_BLOB_CONNECTION_STRING}
  container_name: graphrag-output
  storage_account_blob_url: ${AZURE_BLOB_URL}

cache:
  type: blob
  connection_string: ${AZURE_BLOB_CONNECTION_STRING}
  container_name: graphrag-cache
```

### 3. Cosmos DB (cosmosdb)

Azure Cosmos DB에 데이터를 저장합니다.

```yaml
output:
  type: cosmosdb
  connection_string: ${AZURE_COSMOS_CONNECTION_STRING}
  container_name: graphrag-output
  cosmosdb_account_url: ${AZURE_COSMOS_URL}
```

### 4. 메모리 (memory)

데이터를 메모리에 저장합니다 (테스트용).

```yaml
cache:
  type: memory
```

## 🗂️ 스토리지 구성 요소

### 출력 스토리지 (output)

인덱싱 파이프라인 결과를 저장합니다.

```yaml
output:
  type: file
  base_dir: ./output

# 또는 여러 출력
outputs:
  parquet:
    type: file
    base_dir: ./output/parquet
  json:
    type: blob
    container_name: graphrag-json
```

### 캐시 스토리지 (cache)

중간 결과와 LLM 응답을 캐시합니다.

```yaml
cache:
  type: file
  base_dir: ./cache
```

### 리포팅 스토리지 (reporting)

진행 상황과 메트릭을 저장합니다.

```yaml
reporting:
  type: file
  base_dir: ./reports
```

### 업데이트 인덱스 (update_index_output)

증분 업데이트를 위한 인덱스를 저장합니다.

```yaml
update_index_output:
  type: file
  base_dir: ./output/update
```

## 🗄️ 데이터 모델 스토리지

### Parquet 파일

GraphRAG는 Parquet 형식으로 데이터를 저장합니다.

```
output/
├── create_final_text_units.parquet
├── create_final_entities.parquet
├── create_final_relationships.parquet
├── create_final_communities.parquet
├── create_final_community_reports.parquet
└── create_final_covariates.parquet
```

### 파일 구조

| 파일 | 내용 |
|------|------|
| `create_final_text_units.parquet` | 텍스트 청크 |
| `create_final_entities.parquet` | 추출된 엔티티 |
| `create_final_relationships.parquet` | 엔티티 간 관계 |
| `create_final_communities.parquet` | 커뮤니티 할당 |
| `create_final_community_reports.parquet` | 커뮤니티 요약 |
| `create_final_covariates.parquet` | 추가 메타데이터 |
| `embeddings.parquet` | 벡터 임베딩 |

## 🌐 벡터 스토어 구성

### LanceDB

```yaml
embed_graph:
  vector_store: lancedb

vector_store:
  lancedb:
    type: lancedb
    db_uri: ./lancedb
    overwrite: true
```

### Azure AI Search

```yaml
vector_store:
  azureaisearch:
    type: azureaisearch
    url: ${AZURE_AI_SEARCH_URL}
    api_key: ${AZURE_AI_SEARCH_API_KEY}
    container_name: graphrag-vectors
    audience: null
```

### Cosmos DB Vector Search

```yaml
vector_store:
  cosmosdb:
    type: cosmosdb
    connection_string: ${AZURE_COSMOS_CONNECTION_STRING}
    container_name: graphrag-vectors
    cosmosdb_account_url: ${AZURE_COSMOS_URL}
```

## 🔒 인증 구성

### 연결 문자열

```yaml
output:
  type: blob
  connection_string: ${AZURE_BLOB_CONNECTION_STRING}
```

### Managed Identity

```yaml
output:
  type: blob
  storage_account_blob_url: https://mystorage.blob.core.windows.net
```

## 📊 스토리지 전략

### 로컬 개발

```yaml
# 모든 것을 로컬에 저장
output:
  type: file
  base_dir: ./output

cache:
  type: file
  base_dir: ./cache

reporting:
  type: file
  base_dir: ./reports
```

### 클라우드 프로덕션

```yaml
# Azure에 모든 것을 저장
output:
  type: blob
  connection_string: ${AZURE_BLOB_CONNECTION_STRING}
  container_name: graphrag-prod-output

cache:
  type: blob
  connection_string: ${AZURE_BLOB_CONNECTION_STRING}
  container_name: graphrag-prod-cache

reporting:
  type: blob
  connection_string: ${AZURE_BLOB_CONNECTION_STRING}
  container_name: graphrag-prod-reports
```

### 하이브리드

```yaml
# 로컬 캐시, 클라우드 출력
output:
  type: blob
  connection_string: ${AZURE_BLOB_CONNECTION_STRING}
  container_name: graphrag-output

cache:
  type: file
  base_dir: ./cache

reporting:
  type: console
```

## 🧹 스토리지 관리

### 기존 데이터 삭제

```bash
# 출력 디렉토리 삭제
rm -rf ./output

# 또는 --resume 옵션 없이 재인덱싱
graphrag index --root . --verbose
```

### 증분 업데이트

```yaml
# 업데이트 인덱스 저장
update_index_output:
  type: file
  base_dir: ./output/update
```

```bash
# 증분 업데이트 실행
graphrag index --root . --resume
```

## 📖 관련 문서

- [[Configuration Reference]] - 전체 구성 옵션
- [[Storage Module]] - 스토리지 시스템 상세
- [[Vector Stores]] - 벡터 데이터베이스 옵션
- [[Data Flow]] - 파이프라인을 통한 데이터 흐름

---

*마지막 업데이트: 2025-01-13*
