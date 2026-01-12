---
title: "Chapter 10: System Architecture"
tags:
  - chapter
  - implementation
  - architecture
  - pipeline
created: 2025-01-12
type: chapter
links:
  - [[00-GraphRAG-Textbook-Outline]]
  - [[Textbook - Entity Extraction]]
  - [[Vector Embeddings Deep Dive]]
  - [[LLM Integration Deep Dive]]
---

# Chapter 10: System Architecture

## 학습 목표

이 장을 마치면 다음을 수행할 수 있습니다:
- GraphRAG 엔드투엔드 아키텍처 설명
- 인덱싱 파이프라인 구성 요소 이해
- 쿼리 파이프라인 흐름 설명
- 확장성 고려 사항 식별
- GraphRAG 시스템을 위한 배포 패턴 설계

---

## 10.1 GraphRAG 파이프라인 개요

### 10.1.1 엔드투엔드 아키텍처

```mermaid
flowchart TB
    subgraph Indexing["오프라인: 인덱싱 파이프라인"]
        direction TB

        INPUT[입력 문서]

        PREP[전처리]

        EXTRACT[엔티티 및 관계 추출]

        GRAPH[그래프 구축]

        COMMUNITY[커뮤니티 탐지]

        SUMMARIZE[커뮤니티 요약]

        EMBED[임베딩 생성]

        STORE[(영구 저장소)]

        INPUT --> PREP --> EXTRACT --> GRAPH --> COMMUNITY --> SUMMARIZE --> EMBED --> STORE
    end

    subgraph Query["온라인: 쿼리 파이프라인"]
        direction TB

        Q[사용자 질의]

        Q_ENT[엔티티 추출]

        MODE[검색 모드 선택]

        RETRIEVE[컨텍스트 검색]

        BUILD[컨텍스트 구축]

        GENERATE[응답 생성]

        OUTPUT[답변 + 인용]

        Q --> Q_ENT --> MODE --> RETRIEVE --> BUILD --> GENERATE --> OUTPUT
    end

    STORE -.->|구조 제공| RETRIEVE
    STORE -.->|요약 제공| BUILD

    style Indexing fill:#e3f2fd,stroke:#1565c0
    style Query fill:#f3e5f5,stroke:#7b1fa2
    style STORE fill:#fff3e0,stroke:#ef6c00
```

### 10.1.2 구성 요소 상호 작용

```mermaid
flowchart LR
    subgraph DataFlow["구성 요소 간 데이터 흐름"]
        direction TB

        subgraph Producers["데이터 생산자"]
            P1[문서 수집]
            P2[엔티티 추출]
            P3[커뮤니티 탐지]
        end

        subgraph Consumers["데이터 소비자"]
            C1[쿼리 처리]
            C2[요약]
            C3[시각화]
        end

        subgraph Storage["저장소 계층"]
            S1[(엔티티 테이블)]
            S2[(엣지 테이블)]
            S3[(커뮤니티 테이블)]
            S4[(텍스트 단위 테이블)]
            S5[(벡터 저장소)]
        end

        P1 --> S4
        P2 --> S1
        P2 --> S2
        P3 --> S3
        S1 --> C1
        S2 --> C1
        S3 --> C2
        S4 --> C1
        S5 --> C1
    end

    style Producers fill:#e8f5e9,stroke:#2e7d32
    style Consumers fill:#e3f2fd,stroke:#1565c0
    style Storage fill:#fff3e0,stroke:#ef6c00
```

### 10.1.3 배포 패턴

```mermaid
flowchart TB
    subgraph Patterns["배포 패턴"]
        direction TB

        subgraph Single["단일 머신"]
            S1["모든 구성 요소를 한 머신에"]
            S2["간단한 배포"]
            S3["제한된 확장성"]
        end

        subgraph Distributed["분산"]
            D1["별도의 인덱싱 및 쿼리 노드"]
            D2["수평적 확장"]
            D3["내결함성"]
        end

        subgraph Cloud["클라우드 네이티브"]
            C1["서버리스 함수"]
            C2["관리형 서비스"]
            C3["자동 확장"]
        end
    end

    style Single fill:#e3f2fd,stroke:#1565c0
    style Distributed fill:#fff3e0,stroke:#ef6c00
    style Cloud fill:#e8f5e9,stroke:#2e7d32
```

---

## 10.2 인덱싱 파이프라인

### 10.2.1 파이프라인 단계

```mermaid
flowchart TB
    subgraph IndexDetail["상세 인덱싱 파이프라인"]
        direction TB

        subgraph Stage1["단계 1: 입력 처리"]
            INGEST[문서 수집]
            PARSE[구문 분석 및 정규화]
            SPLIT[텍스트 단위로 분할]
        end

        subgraph Stage2["단계 2: 지식 추출"]
            E_ENT[엔티티 추출]
            E_REL[관계 추출]
            E_EMB[텍스트 단위 임베딩]
        end

        subgraph Stage3["단계 3: 그래프 구축"]
            BUILD[그래프 빌드]
            VALIDATE[검증 및 정리]
            STORE1[(엔티티/엣지 테이블 저장)]
        end

        subgraph Stage4["단계 4: 커뮤니티 탐지"]
            LEIDEN[Leiden 알고리즘 실행]
            HIERARCHY[계층 구조 빌드]
            STORE2[(커뮤니티 테이블 저장)]
        end

        subgraph Stage5["단계 5: 요약"]
            GEN[요약 생성]
            EMB_SUMM[요약 임베딩]
            STORE3[(요약 저장)]
        end

        Stage1 --> Stage2 --> Stage3 --> Stage4 --> Stage5
    end

    style IndexDetail fill:#e3f2fd,stroke:#1565c0
```

### 10.2.2 입력 처리

```mermaid
flowchart TB
    subgraph InputProc["입력 처리 세부 정보"]
        direction TB

        SUPPORTED[지원되는 형식:<br/>- TXT, MD<br/>- PDF<br/>- DOCX<br/>- HTML<br/>- JSON (사용자 지정)]

        NORMALIZE[정규화:<br/>- 텍스트 추출<br/>- 서식 제거<br/>- 인코딩 처리<br/>- 공백 정리]

        CHUNK[청킹 전략:<br/>- 고정 크기 또는 의미적<br/>- 중첩 처리<br/>- 크기 제한]

        METADATA[메타데이터 추출:<br/>- 문서 정보<br/>- 섹션 헤더<br/>- 날짜/시간]

        SUPPORTED --> NORMALIZE --> CHUNK --> METADATA
    end

    style InputProc fill:#e8f5e9,stroke:#2e7d32
```

### 10.2.3 추출 단계

```mermaid
flowchart TB
    subgraph Extraction["추출 단계"]
        direction TB

        PHASE1[단계 1: 텍스트 단위 추출<br/>각 텍스트 단위에서<br/>엔티티/관계 추출]

        AGGREGATE[집계<br/>텍스트 단위 간<br/>추출 결합]

        RESOLVE[해결<br/>- 엔티티 중복 제거<br/>- 관계 병합<br/>- 충돌 해결]

        VALIDATE[검증<br/>- 품질 검사<br/>- 필터링<br/>- 신뢰도 점수]

        PHASE2[단계 2: 그래프 구축<br/>모든 엔티티/관계로<br/>최종 테이블 빌드]

        PHASE1 --> AGGREGATE --> RESOLVE --> VALIDATE --> PHASE2
    end

    style Extraction fill:#f3e5f5,stroke:#7b1fa2
```

### 10.2.4 그래프 구축

```mermaid
classDiagram
    class GraphBuilder {
        +addEntity(entity)
        +addRelationship(relationship)
        +addTextUnit(unit)
        +mergeEntities(id1, id2)
        +validateGraph()
        +getStatistics()
    }

    class Entity {
        +id
        +name
        +type
        +description
    }

    class Relationship {
        +id
        +source_id
        +target_id
        +type
        +weight
    }

    class TextUnit {
        +id
        +text
        +entity_ids
    }

    GraphBuilder "1" --> "*" Entity : manages
    GraphBuilder "1" --> "*" Relationship : manages
    GraphBuilder "1" --> "*" TextUnit : manages
```

### 10.2.5 인덱싱 최적화

| 최적화 | 설명 | 영향 |
|--------------|-------------|--------|
| **병렬 추출** | 텍스트 단위를 동시에 처리 | 5-10배 속도 향상 |
| **배치 LLM 호출** | 여러 추출 결합 | API 오버헤드 감소 |
| **캐싱** | 임베딩, 반복 추출 캐시 | 더 빠른 재인덱싱 |
| **점진적 업데이트** | 변경된 문서만 처리 | 더 빠른 업데이트 |
| **벡터 인덱싱** | 벡터용 HNSW/IVF 사용 | 더 빠른 유사도 검색 |

```mermaid
flowchart TB
    subgraph Parallel["병렬 추출 전략"]
        direction TB

        UNITS[100개 텍스트 단위]

        BATCH[10개 배치로 분할<br/>각각 10개 단위]

        WORKERS[10개 병렬 워커]

        RESULTS[10개 결과 세트]

        MERGE[결과 병합]

        FINAL[최종 엔티티/엣지 테이블]

        UNITS --> BATCH --> WORKERS --> RESULTS --> MERGE --> FINAL
    end

    style Parallel fill:#e8f5e9,stroke:#2e7d32
```

---

## 10.3 쿼리 파이프라인

### 10.3.1 파이프라인 단계

```mermaid
flowchart TB
    subgraph QueryDetail["쿼리 파이프라인 세부 정보"]
        direction TB

        PARSE[쿼리 구문 분석]

        EXTRACT_Q[쿼리 엔티티 추출]

        SELECT{쿼리 유형}

        LOCAL[로컬 검색<br/>- 엔티티 이웃<br/>- 관련 텍스트 단위]

        GLOBAL[전역 검색<br/>- 커뮤니티 요약<br/>- 커뮤니티 엔티티]

        MIXED[혼합 검색<br/>- 로컬 + 전역 결합]

        BUILD_CTX[컨텍스트 구축<br/>- 소스 우선순위<br/>- 토큰 예산 관리]

        GENERATE[답변 생성<br/>- LLM 호출<br/>- 인용 포함]

        PARSE --> EXTRACT_Q --> SELECT
        SELECT -->|특정 엔티티| LOCAL
        SELECT -->|주제적| GLOBAL
        SELECT -->|복잡| MIXED
        LOCAL --> BUILD_CTX
        GLOBAL --> BUILD_CTX
        MIXED --> BUILD_CTX
        BUILD_CTX --> GENERATE
    end

    style QueryDetail fill:#f3e5f5,stroke:#7b1fa2
```

### 10.3.2 쿼리 구문 분석

```mermaid
flowchart TB
    subgraph QueryParse["쿼리 구문 분석"]
        direction TB

        RAW[원본 쿼리:<br/>"아인슈타인은 양자 역학에<br/>어떤 기여를 했습니까?"]

        TOKENIZE[토큰화]

        EXTRACT_E[엔티티 추출:<br/>- 아인슈타인<br/>- 양자 역학]

        CLASSIFY[의도 분류:<br/>유형: 엔티티 특정<br/>범위: 기여<br/>도메인: 물리학]

        PARSED[구문 분석된 쿼리 객체]

        RAW --> TOKENIZE --> EXTRACT_E --> CLASSIFY --> PARSED
    end

    style QueryParse fill:#e3f2fd,stroke:#1565c0
```

### 10.3.3 검색 전략

```mermaid
flowchart TB
    subgraph Retrieval["검색 전략 선택"]
        direction TB

        QUERY{쿼리 분석}

        HAS_ENT[특정 엔티티 있음?]

        SCOPE[광범위 또는 좁음?]

        L1[로컬: 엔티티 중심<br/>엔티티 이웃 순회]

        G1[전역: 커뮤니티 중심<br/>커뮤니티 요약 사용]

        M1[혼합: 결합<br/>로컬 상세 + 전역 컨텍스트]

        QUERY --> HAS_ENT
        HAS_ENT -->|예| SCOPE
        HAS_ENT -->|아니오| G1
        SCOPE -->|좁음| L1
        SCOPE -->|광범위| M1
    end

    style Retrieval fill:#fff3e0,stroke:#ef6c00
```

### 10.3.4 컨텍스트 빌딩

```mermaid
flowchart TB
    subgraph ContextBuild["컨텍스트 빌딩"]
        direction TB

        SOURCES[검색된 소스:<br/>- 엔티티 설명<br/>- 관계<br/>- 커뮤니티 요약<br/>- 텍스트 단위]

        BUDGET[토큰 예산:<br/>총계: 4000 토큰<br/>- 시스템: 500<br/>- 쿼리: 200<br/>- 사용 가능: 3300]

        ALLOCATE[예산 할당:<br/>- 요약: 1000<br/>- 엔티티: 800<br/>- 텍스트: 1000<br/>- 버퍼: 500]

        PRIORITIZE[우선순위:<br/>- 관련성 점수<br/>- 정보 새로움<br/>- 소스 다양성]

        ASSEMBLE[컨텍스트 조립:<br/>- 논리적 순서<br/>- 인용 추가<br/>- LLM용 포맷]

        SOURCES --> BUDGET --> ALLOCATE --> PRIORITIZE --> ASSEMBLE
    end

    style ContextBuild fill:#e8f5e9,stroke:#2e7d32
```

---

## 10.4 확장성 고려 사항

### 10.4.1 분산 인덱싱

```mermaid
flowchart TB
    subgraph Distributed["분산 인덱싱 아키텍처"]
        direction TB

        COORD[코디네이터 노드]

        WORKER1[워커 1<br/>문서 1-1000]

        WORKER2[워커 2<br/>문서 1001-2000]

        WORKER3[워커 3<br/>문서 2001-3000]

        WORKERN[워커 N<br/>...]

        MERGE[결과 병합]

        STORAGE[(공유 저장소)]

        COORD --> WORKER1
        COORD --> WORKER2
        COORD --> WORKER3
        COORD --> WORKERN

        WORKER1 --> MERGE
        WORKER2 --> MERGE
        WORKER3 --> MERGE
        WORKERN --> MERGE

        MERGE --> STORAGE
    end

    style Distributed fill:#e3f2fd,stroke:#1565c0
```

### 10.4.2 병렬 처리

```mermaid
flowchart TB
    subgraph Parallelism["병렬화 기회"]
        direction TB

        subgraph Embarrassing["완전히 병렬 가능"]
            E1[텍스트 단위 추출]
            E2[임베딩 생성]
            E3[요약]
        end

        subgraph Coord["조정된 병렬"]
            C1[엔티티 집계<br/>통신 필요]
            C2[커뮤니티 탐지<br/>전체 그래프 필요]
        end

        subgraph Sequential["순차적 병목"]
            S1[Leiden 알고리즘]
            S2[최종 그래프 병합]
        end
    end

    style Embarrassing fill:#e8f5e9,stroke:#2e7d32
    style Coord fill:#fff3e0,stroke:#ef6c00
    style Sequential fill:#ffebee,stroke:#c62828
```

### 10.4.3 메모리 관리

```mermaid
flowchart TB
    subgraph Memory["메모리 관리 전략"]
        direction TB

        LOAD[로드 전략]

        FULL[전체 그래프 메모리<br/>- 가장 빠른 쿼리<br/>- 큰 RAM 필요<br/>- 100만 엔티티 미만에 적합]

        PARTIAL[부분 로딩<br/>- 핫 서브셋 캐시<br/>- 요청 시 로드<br/>- 균형 잡힌 접근]

        DISK[디스크 기반<br/>- 모든 크기로 확장 가능<br/>- 더 느린 쿼리<br/>- 스트리밍 처리]

        LOAD --> FULL
        LOAD --> PARTIAL
        LOAD --> DISK
    end

    style Memory fill:#f3e5f5,stroke:#7b1fa2
```

| 그래프 크기 | 메모리 전략 | 필요 RAM |
|------------|------------------|--------------|
| < 10만 엔티티 | 전체 인메모리 | 8-16 GB |
| 10만-100만 엔티티 | 부분 + 캐싱 | 32-64 GB |
| 100만-1000만 엔티티 | 디스크 기반 + 캐시 | 64-128 GB |
| > 1000만 엔티티 | 분산 | 클러스터 |

### 10.4.4 성능 최적화

```mermaid
flowchart TB
    subgraph Optimization["성능 최적화 기법"]
        direction TB

        subgraph IndexOpt["인덱싱 최적화"]
            IO1[배치 LLM 호출]
            IO2[병렬 텍스트 단위 처리]
            IO3[임베딩 캐싱]
            IO4[점진적 업데이트]
        end

        subgraph QueryOpt["쿼리 최적화"]
            QO1[빈번한 쿼리 캐싱]
            QO2[근사 최근접 이웃]
            QO3[엔티티/요약 사전 필터링]
            QO4[컨텍스트 압축]
        end

        subgraph StorageOpt["저장소 최적화"]
            SO1[벡터 인덱싱 (HNSW)]
            SO2[그래프 분할]
            SO3[칼럼 기반 저장소 (Parquet)]
            SO4[압축]
        end
    end

    style IndexOpt fill:#e3f2fd,stroke:#1565c0
    style QueryOpt fill:#e8f5e9,stroke:#2e7d32
    style StorageOpt fill:#f3e5f5,stroke:#7b1fa2
```

---

## 10.5 배포 아키텍처

### 10.5.1 단일 노드 배포

```mermaid
flowchart TB
    subgraph SingleNode["단일 노드 아키텍처"]
        direction TB

        API[REST API]

        QUERY[쿼리 엔진]

        INDEXER[인덱서]

        STORAGE[(로컬 저장소)]

        LLM[LLM 제공자<br/>OpenAI/Azure/Anthropic]

        API --> QUERY
        API --> INDEXER
        QUERY --> STORAGE
        INDEXER --> STORAGE
        QUERY --> LLM
        INDEXER --> LLM
    end

    style SingleNode fill:#e3f2fd,stroke:#1565c0
```

**장점:** 단순, 낮은 운영 오버헤드
**단점:** 제한된 확장성, 단일 실패 지점

### 10.5.2 분산 배포

```mermaid
flowchart TB
    subgraph Distributed["분산 아키텍처"]
        direction TB

        subgraph Frontend["프론트엔드 계층"]
            LB[로드 밸런서]
            API1[API 서버 1]
            API2[API 서버 2]
            APIN[API 서버 N]
        end

        subgraph Worker["워커 계층"]
            Q1[쿼리 워커 1]
            Q2[쿼리 워커 2]
            I1[인덱서 워커 1]
            I2[인덱서 워커 2]
        end

        subgraph Storage["저장소 계층"]
            CACHE[(Redis 캐시)]
            VECTOR[(벡터 DB)]
            GRAPH[(그래프 저장소)]
            FILES[(파일 저장소)]
        end

        LB --> API1
        LB --> API2
        LB --> APIN

        API1 --> Q1
        API2 --> Q2
        API1 --> I1
        API2 --> I2

        Q1 --> CACHE
        Q2 --> CACHE
        Q1 --> VECTOR
        Q2 --> VECTOR
        Q1 --> GRAPH
        Q2 --> GRAPH

        I1 --> FILES
        I2 --> FILES
        I1 --> GRAPH
        I2 --> GRAPH
    end

    style Distributed fill:#e8f5e9,stroke:#2e7d32
```

**장점:** 확장 가능, 내결함성
**단점:** 복잡, 높은 운영 비용

### 10.5.3 클라우드 네이티브 배포

```mermaid
flowchart TB
    subgraph Cloud["클라우드 네이티브 아키텍처 (AWS 예시)"]
        direction TB

        subgraph Compute["컴퓨팅"]
            LAMBDA[AWS Lambda<br/>서버리스 함수]
            ECS[Amazon ECS<br/>컨테이너 작업]
        end

        subgraph Storage["저장소"]
            S3[S3<br/>문서 저장소]
            OPENSEARCH[OpenSearch<br/>벡터 + 그래프]
            DDB[DynamoDB<br/>메타데이터]
        end

        subgraph Queue["대기열"]
            SQS[SQS<br/>작업 대기열]
        end

        subgraph API["API 게이트웨이"]
            APIGW[API 게이트웨이]
        end

        APIGW --> LAMBDA
        APIGW --> ECS

        LAMBDA --> SQS
        ECS --> SQS

        LAMBDA --> S3
        LAMBDA --> OPENSEARCH
        ECS --> OPENSEARCH

        LAMBDA --> DDB
        ECS --> DDB
    end

    style Cloud fill:#f3e5f5,stroke:#7b1fa2
```

---

## 장 요약

이 장에서는 GraphRAG 시스템 아키텍처를 다루었습니다:

**아키텍처 개요:**
- 두 단계 설계: **인덱싱** (오프라인) 및 **쿼리** (온라인)
- 데이터 생산자와 소비자 간의 명확한 분리
- 단일 노드에서 클라우드 네이티브까지 다양한 배포 패턴

**인덱싱 파이프라인:**
- **입력 처리**: 문서 수집, 정규화, 청킹
- **추출 단계**: 엔티티 및 관계 추출과 집계
- **그래프 구축**: 지식 그래프 빌드 및 검증
- **커뮤니티 탐지**: Leiden 알고리즘 실행
- **요약**: 커뮤니티 요약 생성

**쿼리 파이프라인:**
- **쿼리 구문 분석**: 엔티티 추출 및 의도 분류
- **검색 전략**: 쿼리 유형에 따른 로컬, 전역 또는 혼합
- **컨텍스트 빌딩**: 예산 할당 및 우선순위 지정
- **답변 생성**: 인용이 포함된 LLM 호출

**확장성:**
- 완전히 병렬 가능한 작업에 대한 **병렬 처리**
- 대규모 문서 컬렉션을 위한 **분산 인덱싱**
- 다양한 그래프 크기를 위한 **메모리 관리** 전략
- 캐싱 및 인덱싱을 통한 **성능 최적화**

**다음 단계:**
[[Vector Embeddings Deep Dive]]에서 GraphRAG 시스템에 임베딩이 통합되는 방법을 다룹니다.

---

## 복습 질문

1. 인덱싱 파이프라인과 쿼리 파이프라인의 주요 차이점은 무엇입니까?
2. 인덱싱 파이프라인의 다섯 단계를 설명하세요.
3. 쿼리 파이프라인은 로컬 검색과 전역 검색 사이에서 어떻게 결정합니까?
4. GraphRAG의 확장성 고려 사항은 무엇입니까?
5. 단일 노드, 분산, 클라우드 네이티브 배포 패턴을 비교하세요.
6. 분산 GraphRAG 시스템의 주요 병목 현상은 무엇입니까?

---

## 연습문제

1. 100만 개 문서를 처리하고 초당 100개 쿼리를 처리해야 하는 GraphRAG 시스템을 위한 배포 아키텍처를 설계하세요.

2. 인덱싱 파이프라인이 너무 오래 걸립니다. 세 가지 잠재적 병목 현상을 식별하고 각각에 대한 최적화를 제안하세요.

3. AWS에 배포합니다. 각 구성 요소에 어떤 서비스를 사용하시겠습니까, 그 이유는 무엇입니까?

---

## 추가 참고자료

- GraphRAG 아키텍처 문서
- 분산 시스템 설계 패턴
- 클라우드 아키텍처 모범 사례
- 성능 최적화 기법
