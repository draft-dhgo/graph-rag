---
title: Context Building Deep Dive
tags:
  - deep-dive
  - context-builder
  - local-search
  - global-search
  - retrieval
created: 2025-01-12
type: deep-dive
links:
  - [[Local Search]]
  - [[Global Search]]
  - [[DRIFT Search]]
  - [[Query Module]]
---

# Context Building Deep Dive

컨텍스트 빌딩(Context Building)은 GraphRAG의 검색 엔진에서 사용자 쿼리에 대한 관련 정보를 수집하고 구성하는 핵심 프로세스입니다.

## 목차

### 1. 개요
- [컨텍스트 빌더의 목적](#-컨텍스트-빌더의-목적)
- [빗대어 보기: 스마트 리서치 어시스턴트](#-빗대어-보기-스마트-리서치-어시스턴트)

### 2. 아키텍처
- [컨텍스트 아키텍처](#-컨텍스트-아키텍처)
- [처리 파이프라인](#-처리-파이프라인)

### 3. 로컬 컨텍스트
- [로컬 컨텍스트 빌더](#-로컬-컨텍스트-빌더)
- [관계 우선순위 전략](#-관계-우선순위-전략)
- [Hop 순회](#hop-순회)

### 4. 글로벌 컨텍스트
- [글로벌 컨텍스트 빌더](#-글로벌-컨텍스트-빌더)
- [Map-Reduce 아키텍처](#map-reduce-아키텍처)
- [동적 커뮤니티 선택](#-동적-커뮤니티-선택)

### 5. 토큰 할당
- [토큰 할당 전략](#-토큰-할당-전략)
- [비율 할당](#-비율-할당)
- [지능형 토큰 관리](#-지능형-토큰-관리)

### 6. 최적화
- [재순위 (Reranking)](#1-재순위-reranking)
- [컨텍스트 압축](#2-컨텍스트-압축)
- [다양성 확보](#3-다양성-확보)

---

## 🎯 컨텍스트 빌더의 목적

```mermaid
flowchart LR
    A[사용자 쿼리] --> B[컨텍스트 빌더]
    B --> C[관련성]
    B --> D[균형]
    B --> E[토큰 효율]
    B --> F[순서]

    style A fill:#e1f5fe
    style B fill:#fff9c4
    style C fill:#c8e6c9
    style D fill:#e1bee7
    style E fill:#ffccbc
    style F fill:#fff59d
```

1. **관련성**: 쿼리와 가장 관련된 정보 선택
2. **균형**: 다양한 정보 소스의 적절한 혼합
3. **토큰 효율**: 제한된 토큰 예산 내에서 최대한 정보 제공
4. **순서**: 중요한 정보가 먼저 표시되도록 순서화

## 📖 빗대어 보기: 스마트 리서치 어시스턴트

컨텍스트 빌더는 **연구자를 위해 관련 자료를 수집하고 정리하는 스마트 어시스턴트**와 유사합니다:

| 리서치 어시스턴트 | 컨텍스트 빌더 |
|-----------------|--------------|
| 연구 주제 이해 | 쿼리 분석 |
| 관련 논문 검색 | 관련 엔티티 검색 |
| 인용 관계 추적 | 관계 순회 |
| 주요 발췌 정리 | 텍스트 유닛 선택 |
| 전문가 의견 수집 | 커뮤니티 리포트 |
| 요약 보고서 작성 | 최종 컨텍스트 구성 |

```mermaid
flowchart TB
    subgraph Assistant["리서치 어시스턴트"]
        A1["📚 주제 분석"]
        A2["🔍 논문 검색"]
        A3["📖 인용 추적"]
        A4["📝 발췌 정리"]
        A5["👨‍🏫 전문가 의견"]
        A6["📊 요약 보고서"]
    end

    subgraph Builder["컨텍스트 빌더"]
        B1["🔤 쿼리 분석"]
        B2["🏷️ 엔티티 검색"]
        B3["🔗 관계 순회"]
        B4["📄 텍스트 선택"]
        B5["🌐 커뮤니티 리포트"]
        B6["💬 최종 컨텍스트"]
    end

    style Assistant fill:#e3f2fd
    style Builder fill:#f3e5f5
```

## 🏗️ 컨텍스트 아키텍처

```mermaid
flowchart TB
    subgraph Input["입력 데이터"]
        Q["❓ User Query"]
        E["🏷️ Entities"]
        R["🔗 Relationships"]
        TU["📄 Text Units"]
        CR["🌐 Community Reports"]
    end

    subgraph Local["로컬 컨텍스트 빌더"]
        direction TB
        EXT["엔티티 추출<br/>Query → Entities"]
        SIM["의미적 유사도<br/>Find related"]
        HOP["Hop 순회<br/>2-3 hop neighbors"]
        REL["관계 선택<br/>In/Out/Single"]
    end

    subgraph Global["글로벌 컨텍스트 빌더"]
        direction TB
        SEL["커뮤니티 선택<br/>All or Relevant"]
        MAP["Map 단계<br/>Parallel processing"]
        RED["Reduce 단계<br/>Aggregation"]
    end

    subgraph Build["컨텍스트 구성"]
        direction TB
        PROP["비례 할당<br/>50% Text + 25%<br/>Community + 25% Local"]
        ORDER["순서화<br/>Relevance ranking"]
        FMT["포맷팅<br/>Prompt construction"]
    end

    Q --> EXT --> SIM --> HOP --> REL --> PROP
    E --> SIM
    R --> REL
    TU --> PROP
    CR --> PROP

    Q --> SEL --> MAP --> RED --> PROP
    CR --> MAP
    CR --> RED

    PROP --> ORDER --> FMT --> CTX["📋 Final Context"]

    style Input fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    style Local fill:#fff3e0,stroke:#f57c00,stroke-width:2px
    style Global fill:#c8e6c9,stroke:#388e3c,stroke-width:2px
    style Build fill:#fce4ec,stroke:#c2185b,stroke-width:2px
    style CTX fill:#e1bee7,stroke:#7b1fa2,stroke-width:3px
```

### 처리 파이프라인

```mermaid
flowchart TB
    START([쿼리 수신]) --> PARSE["쿼리 파싱<br/>엔티티 추출"]

    PARSE --> STRATEGY{"검색 전략"}

    STRATEGY -->|로컬| LOCAL["로컬 빌더<br/>엔티티 중심"]
    STRATEGY -->|글로벌| GLOBAL["글로벌 빌더<br/>커뮤니티 중심"]
    STRATEGY -->|DRIFT| DRIFT["DRIFT 빌더<br/>반복적 확장"]

    LOCAL --> BUILD["컨텍스트 빌드"]
    GLOBAL --> BUILD
    DRIFT --> BUILD

    BUILD --> ALLOC["토큰 할당"]

    ALLOC --> RERANK["재순위"]

    RERANK --> FORMAT["포맷팅"]

    FORMAT --> END([최종 컨텍스트])

    style START fill:#e1f5fe
    style END fill:#c8e6c9
    style LOCAL fill:#fff3e0
    style GLOBAL fill:#c8e6c9
    style BUILD fill:#e1bee7
```

## 🔍 로컬 컨텍스트 빌더

### 상세 알고리즘

```mermaid
flowchart TB
    START([로컬 검색 시작]) --> QE["쿼리에서<br/>엔티티 추출"]

    QE --> SE["의미적 유사 엔티티<br/>검색 (Top-K)"]

    SE --> HOP["관계 순회<br/>2-3 Hop"]

    HOP --> PRIORITY["관계 우선순위<br/>정렬"]

    PRIORITY --> TUS["관련 텍스트<br/>유닛 선택"]

    TUS --> CRS["관련 커뮤니티<br/>리포트 선택"]

    CRS --> BUILD["컨텍스트 빌드"]

    style START fill:#e1f5fe
    style BUILD fill:#c8e6c9
```

```python
class LocalContextBuilder:
    def __init__(
        self,
        text_unit_prop: float = 0.5,      # 50% 텍스트
        community_prop: float = 0.25,     # 25% 커뮤니티
        local_prop: float = 0.25,         # 25% 로컬 데이터
        max_context_tokens: int = 12000,
        top_k_entities: int = 10,
        top_k_relationships: int = 10
    ):
        self.text_unit_prop = text_unit_prop
        self.community_prop = community_prop
        self.local_prop = local_prop
        self.max_tokens = max_context_tokens
        self.top_k_entities = top_k_entities
        self.top_k_relationships = top_k_relationships
```

### 관계 우선순위 전략

```mermaid
flowchart TB
    A["관계 목록"] --> CHECK{엔티티<br/>연결 확인}

    CHECK -->|양방향| IN["In-network<br/>🟢 우선순위 1"]
    CHECK -->|단방향| OUT["Out-network<br/>🟡 우선순위 2"]

    IN --> SORT1["가중치순 정렬"]
    OUT --> SORT2["가중치순 정렬"]

    SORT1 --> MERGE["병합"]
    SORT2 --> MERGE

    MERGE --> FINAL["최종 관계 목록"]

    style A fill:#e1f5fe
    style IN fill:#c8e6c9
    style OUT fill:#fff9c4
    style FINAL fill:#e1bee7
```

| 우선순위 | 유형 | 설명 | 예시 |
|----------|------|------|------|
| 1 | In-network | 쿼리 관련 엔티티 간 연결 | A↔B (둘 다 관련) |
| 2 | Out-network | 한쪽만 관련된 연결 | A→B (A만 관련) |
| 3 | Single | 단일 연결 | 낮은 가중치 |

### Hop 순회

```mermaid
flowgraph TB
    Q["쿼리 엔티티"] --> HOP1["Hop 1<br/>직접 연결"]

    HOP1 --> N1["이웃 엔티티 A"]
    HOP1 --> N2["이웃 엔티티 B"]

    N1 --> HOP2["Hop 2<br/>간접 연결"]
    N2 --> HOP2

    HOP2 --> N3["2-hop 엔티티 C"]
    HOP2 --> N4["2-hop 엔티티 D"]

    style Q fill:#fff9c4
    style HOP1 fill:#c8e6c9
    style HOP2 fill:#a5d6a7
```

```
Hop 0: [쿼리 엔티티]
Hop 1: [직접 연결된 엔티티]
Hop 2: [1-hop의 이웃 엔티티]
Hop 3: [2-hop의 이웃 엔티티]

일반적으로 2-3 hop 까지 순회
```

## 🌐 글로벌 컨텍스트 빌더

### Map-Reduce 아키텍처

```mermaid
flowchart TB
    subgraph Map["Map 단계 (병렬)"]
        M1["커뮤니티 A<br/>LLM → 답변 A"]
        M2["커뮤니티 B<br/>LLM → 답변 B"]
        M3["커뮤니티 C<br/>LLM → 답변 C"]
        M4["커뮤니티 N<br/>LLM → 답변 N"]
    end

    subgraph Reduce["Reduce 단계 (집계)"]
        R1["중간 답변 수집"]
        R2["LLM → 최종 답변"]
    end

    Q["쿼리"] --> Map
    Map --> R1 --> R2 --> F["최종 응답"]

    style Q fill:#e1f5fe
    style Map fill:#c8e6c9
    style Reduce fill:#e1bee7
    style F fill:#fff9c4
```

```python
async def _map_phase(
    self,
    query: str,
    communities: list[dict],
    reports: pd.DataFrame,
    concurrency: int
) -> list[str]:
    """
    각 커뮤니티에서 독립적으로 답변 생성
    """
    semaphore = asyncio.Semaphore(concurrency)

    async def process_community(comm):
        async with semaphore:
            report = reports[reports['community'] == comm['id']].iloc[0]

            prompt = f"""
            Given this community report:
            {report['full_content']}

            And the question: {query}

            Provide a brief intermediate answer focused on this community.
            """

            return await llm.execute(prompt)

    # 병렬 실행
    tasks = [process_community(c) for c in communities]
    results = await asyncio.gather(*tasks)

    return results
```

### 동적 커뮤니티 선택

```mermaid
flowchart TB
    A["모든 커뮤니티"] --> METHOD{선택 방법}

    METHOD -->|Auto| SIM["유사도 기반<br/>쿼리-커뮤니티<br/>임베딩 비교"]
    METHOD -->|Top| RANK["크기 기반<br/>상위 N개<br/>커뮤니티"]
    METHOD -->|All| ALL["모든 커뮤니티"]

    SIM --> THRESH["상위 50%<br/>임계값 필터"]
    THRESH --> SELECTED["선택된 커뮤니티"]
    RANK --> SELECTED
    ALL --> SELECTED

    style A fill:#e1f5fe
    style SIM fill:#c8e6c9
    style RANK fill:#fff9c4
    style ALL fill:#e1bee7
    style SELECTED fill:#ffccbc
```

| 방법 | 설명 | 사용 사례 |
|------|------|----------|
| `auto` | 쿼리와 유사한 커뮤니티 선택 | 일반적 |
| `top` | 크기 기반 상위 선택 | 포괄적 질문 |
| `all` | 모든 커뮤니티 사용 | 완전한 분석 |

## 📊 토큰 할당 전략

### 비율 할당

```mermaid
pie title 토큰 할당 비율 (기본)
    "텍스트 유닛" : 50
    "커뮤니티 리포트" : 25
    "로컬 데이터" : 25
```

```python
def _allocate_tokens(
    self,
    text_units: list[str],
    community_reports: list[str],
    entities: list[dict],
    relationships: list[dict],
    max_tokens: int = 12000
) -> dict:
    """
    정해진 비율로 토큰 할당
    """
    total_available = max_tokens

    # 비율별 할당량 계산
    text_budget = int(total_available * self.text_unit_prop)      # 6000
    report_budget = int(total_available * self.community_prop)   # 3000
    local_budget = int(total_available * self.local_prop)        # 3000

    return {
        'text_units': select_by_token_count(text_units, text_budget),
        'community_reports': select_by_token_count(community_reports, report_budget),
        'local_data': select_local_data(entities, relationships, local_budget)
    }
```

### 지능형 토큰 관리

```mermaid
flowchart TB
    A["쿼리 분석"] --> TYPE{쿼리 유형}

    TYPE -->|구체적| SPECIFIC["구체적 질문<br/>더 많은 로컬 데이터"]
    TYPE -->|포괄적| BROAD["포괄적 질문<br/>더 많은 커뮤니티"]
    TYPE -->|기본| DEFAULT["기본 비율"]

    SPECIFIC --> ALLOC1["Text: 40%<br/>Community: 20%<br/>Local: 40%"]
    BROAD --> ALLOC2["Text: 30%<br/>Community: 50%<br/>Local: 20%"]
    DEFAULT --> ALLOC3["Text: 50%<br/>Community: 25%<br/>Local: 25%"]

    style A fill:#e1f5fe
    style SPECIFIC fill:#c8e6c9
    style BROAD fill:#e1bee7
    style DEFAULT fill:#fff9c4
```

```python
def _allocate_tokens_adaptive(
    self,
    query_type: str,
    available_data: dict,
    max_tokens: int
) -> dict:
    """
    쿼리 유형에 따른 적응형 토큰 할당
    """
    if query_type == "specific":
        allocation = {
            'text_units': 0.4,
            'community_reports': 0.2,
            'local_data': 0.4
        }
    elif query_type == "broad":
        allocation = {
            'text_units': 0.3,
            'community_reports': 0.5,
            'local_data': 0.2
        }
    else:
        allocation = {
            'text_units': 0.5,
            'community_reports': 0.25,
            'local_data': 0.25
        }

    return allocation
```

## 🎓 컨텍스트 최적화

### 1. 재순위 (Reranking)

```mermaid
flowchart TB
    A["검색된 컨텍스트"] --> B["점수 계산"]

    B --> C1["키워드 매칭<br/>40%"]
    B --> C2["최신성<br/>20%"]
    B --> C3["엔티티 관련성<br/>40%"]

    C1 --> SUM["종합 점수"]
    C2 --> SUM
    C3 --> SUM

    SUM --> SORT["점수순 정렬"]

    SORT --> FINAL["재순위된 결과"]

    style A fill:#e1f5fe
    style FINAL fill:#c8e6c9
```

```python
def rerank_context(
    context_items: list[dict],
    query: str
) -> list[dict]:
    """
    검색된 컨텍스트 항목을 재순위
    """
    scores = []

    for item in context_items:
        # 1. 키워드 매칭 점수
        keyword_score = calculate_keyword_match(query, item)

        # 2. 최신성 점수
        recency_score = calculate_recency(item)

        # 3. 엔티티 관련성
        entity_score = calculate_entity_relevance(query, item)

        # 종합 점수
        total = (
            keyword_score * 0.4 +
            recency_score * 0.2 +
            entity_score * 0.4
        )

        scores.append((item, total))

    # 점수순 정렬
    sorted_items = sorted(scores, key=lambda x: -x[1])

    return [item for item, _ in sorted_items]
```

### 2. 컨텍스트 압축

```mermaid
flowchart TB
    A["긴 컨텍스트"] --> B["중요 부분 식별"]

    B --> C1["핵심 문장"]
    B --> C2["관련 엔티티"]

    C1 --> PRIORITY["우선순위 할당"]
    C2 --> PRIORITY

    PRIORITY --> BUILD["압축된 컨텍스트<br/>구성"]

    BUILD --> CHECK{토큰 수<br/>확인}

    CHECK -->|초과| TRUNCATE["자르기"]
    CHECK -->|적절| FINAL["완성"]

    TRUNCATE --> FINAL

    style A fill:#e1f5fe
    style FINAL fill:#c8e6c9
```

### 3. 다양성 확보

```mermaid
flowgraph LR
    S1["선택된 항목 1<br/>벡터: [0.8, 0.3]"]
    S2["선택된 항목 2<br/>벡터: [0.85, 0.35]"]

    C1["후보 항목 1<br/>벡터: [0.82, 0.32]"]
    C2["후보 항목 2<br/>벡터: [0.1, 0.2]"]

    S1 -.-"유사함"-.-> C1
    S1 -.-"다름"-.-> C2

    style S1 fill:#c8e6c9
    style S2 fill:#c8e6c9
    style C1 fill:#ffcdd2
    style C2 fill:#a5d6a7
```

```python
def ensure_diversity(
    selected_items: list[dict],
    all_items: list[dict],
    diversity_threshold: float = 0.7
) -> list[dict]:
    """
    선택된 항목의 다양성 확보
    """
    diverse_items = [selected_items[0]]

    for item in all_items:
        if item in selected_items:
            continue

        # 기존 항목들과의 유사도 확인
        similarities = [
            cosine_similarity(item['embedding'], emb)
            for emb in selected_embeddings
        ]

        max_similarity = max(similarities) if similarities else 0

        # 유사도가 임계값보다 낮으면 추가
        if max_similarity < diversity_threshold:
            diverse_items.append(item)

    return diverse_items
```

## 📊 컨텍스트 품질 메트릭

```mermaid
radar-beta
    title 컨텍스트 품질 지표
    axis Relevance["관련성<br/>LLM 평가", 0.8]
    axis Coverage["커버리지<br/>엔티티/커뮤니티 수", 0.7]
    axis Conciseness["간결성<br/>압축률", 0.6]
    axis Coherence["일관성<br/>연결성", 0.75]
```

| 메트릭 | 설명 | 측정 방법 | 목표 |
|--------|------|----------|------|
| **Relevance** | 쿼리와의 관련성 | LLM으로 평가 | > 0.7 |
| **Coverage** | 주제涵盖 범위 | 포함된 엔티티/커뮤니티 수 | 다양 |
| **Conciseness** | 불필요 정보 제거 | 압축률 | 높음 |
| **Coherence** | 논리적 일관성 | 문장 간 연결성 | > 0.7 |

## 🔗 관련 컴포넌트

- [[Local Search]]: 로컬 컨텍스트 사용
- [[Global Search]]: 글로벌 컨텍스트 사용
- [[Entity]]: 컨텍스트 구성 요소
- [[Community Report]]: 커뮤니티 컨텍스트

---
*See also: [[Query Module]], [[Local Search]], [[Global Search]]*
