---
title: Vector Embeddings Deep Dive
tags:
  - deep-dive
  - embeddings
  - vectors
  - semantic-search
  - nlp
created: 2025-01-12
type: deep-dive
links:
  - [[Storage Module]]
  - [[Local Search]]
  - [[Text Unit]]
  - [[Entity]]
---

# Vector Embeddings Deep Dive

벡터 임베딩(Vector Embeddings)은 텍스트를 고차원 수치 벡터로 변환하여, 의미론적 유사도 계산과 시맨틱 서치를 가능하게 하는 핵심 기술입니다.

## 목차

### 1. 개요
- [임베딩의 목적](#-임베딩의-목적)
- [빗대어 보기: 도서관 위치 시스템](#-빗대어-보기-도서관-위치-시스템)

### 2. 아키텍처
- [임베딩 아키텍처](#-임베딩-아키텍처)
- [처리 파이프라인](#-처리-파이프라인)

### 3. 임베딩 모델
- [OpenAI Embeddings](#openai-embeddings)
- [차원 선택 가이드](#-차원-선택-가이드)

### 4. 임베딩 전략
- [텍스트 청크 임베딩](#1-텍스트-청크-임베딩)
- [엔티티 설명 임베딩](#2-엔티티-설명-임베딩)
- [커뮤니티 컨텍스트 임베딩](#3-커뮤니티-컨텍스트-임베딩)

### 5. 유사도 계산
- [코사인 유사도](#코사인-유사도-cosine-similarity)
- [다른 유사도 메트릭](#다른-유사도-메트릭)

### 6. 검색 기법
- [Top-K 검색](#top-k-검색)
- [ANN 검색](#ann-approximate-nearest-neighbor)
- [하이브리드 검색](#하이브리드-검색)

### 7. 최적화
- [임베딩 최적화](#-임베딩-최적화)

---

## 🎯 임베딩의 목적

```mermaid
flowchart LR
    A[텍스트] --> B[임베딩]
    B --> C[의미론적 검색]
    B --> D[유사도 계산]
    B --> E[클러스터링]
    B --> F[추천]

    style A fill:#e1f5fe
    style B fill:#fff9c4
    style C fill:#c8e6c9
    style D fill:#e1bee7
    style E fill:#ffccbc
    style F fill:#fff59d
```

1. **의미론적 검색**: 키워드 매칭을 넘어 의미 기반 검색
2. **유사도 계산**: 텍스트/엔티티 간 의미적 거리 측정
3. **클러스터링**: 유사한 항목들의 그룹화
4. **추천**: 관련 콘텐츠의 추천

## 📖 빗대어 보기: 도서관 위치 시스템

벡터 임베딩은 **도서관에서 책의 주제별 위치를 좌표로 나타내는 것**과 유사합니다:

| 도서관 시스템 | 벡터 임베딩 |
|-------------|------------|
| 책 주제 분류 | 텍스트 → 벡터 변환 |
| 비슷한 주제 근처 배치 | 유사한 의미 가까운 좌표 |
| 주제 거리 계산 | 코사인 유사도 |
| 새 책 위치 배정 | 새 텍스트 임베딩 |
| 분야별 섹션 | 클러스터링 |

```mermaid
flowchart TB
    subgraph Library["도서관 분류"]
        L1["📚 컴퓨터 과학<br/>Q1-100"]
        L2["📚 인공지능<br/>Q1-110"]
        L3["📚 기계학습<br/>Q1-115"]
        L4["📚 딥러닝<br/>Q1-118"]
    end

    subgraph Embedding["벡터 공간"]
        E1["[0.8, 0.3, 0.9]<br/>Computer Science"]
        E2["[0.9, 0.4, 0.8]<br/>AI"]
        E3["[0.95, 0.5, 0.85]<br/>Machine Learning"]
        E4["[0.98, 0.6, 0.88]<br/>Deep Learning"]
    end

    style Library fill:#e3f2fd
    style Embedding fill:#f3e5f5
```

### 벡터 공간 시각화

```mermaid
flowchart LR
    subgraph "2D 벡터 공간 (축약 예시)"
        direction TB

        A["🤖 AI<br/>(0.8, 0.9)"]
        B["🧠 ML<br/>(0.85, 0.88)"]
        C["🔮 DL<br/>(0.9, 0.85)"]

        D["📊 데이터<br/>(0.3, 0.4)"]
        E["📈 통계<br/>(0.35, 0.38)"]

        F["🌐 웹<br/>(0.1, 0.2)"]
        G["🎨 디자인<br/>(0.05, 0.15)"]
    end

    style A fill:#c8e6c9
    style B fill:#c8e6c9
    style C fill:#c8e6c9
    style D fill:#fff9c4
    style E fill:#fff9c4
    style F fill:#ffcdd2
    style G fill:#ffcdd2
```

## 🏗️ 임베딩 아키텍처

```mermaid
flowchart TB
    subgraph Input["입력 데이터"]
        T["📝 Text<br/>문서 청크"]
        E["🏷️ Entity<br/>엔티티 설명"]
        C["🌐 Community<br/>커뮤니티 요약"]
    end

    subgraph Encode["인코딩 단계"]
        TOKEN["🔤 토큰화<br/>tiktoken<br/>문자 → 토큰 ID"]
        EMBED["🔄 임베딩 모델<br/>OpenAI/Cohere<br/>토큰 → 벡터"]
        NORM["⚖️ 정규화<br/>L2 Normalization<br/>단위 벡터"]
    end

    subgraph Store["저장소"]
        VS["💾 Vector Store<br/>LanceDB/Azure<br/>인덱싱"]
    end

    subgraph Search["검색 단계"]
        Q["❓ Query"]
        QE["🔤 쿼리 토큰화"]
        QE2["🔄 쿼리 임베딩"]
        SIM["📊 유사도 계산<br/>Cosine Similarity"]
        TOP["🎯 Top-K Results"]
    end

    T --> TOKEN --> EMBED --> NORM --> VS
    E --> TOKEN
    C --> TOKEN

    Q --> QE --> QE2 --> SIM --> TOP

    style Input fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    style Encode fill:#fff3e0,stroke:#f57c00,stroke-width:2px
    style Store fill:#e8f5e9,stroke:#388e3c,stroke-width:2px
    style Search fill:#fce4ec,stroke:#c2185b,stroke-width:2px
```

### 처리 파이프라인

```mermaid
flowchart LR
    A["원본 텍스트<br/>'GraphRAG is...'"] --> B[토큰화<br/>[1234, 5678, ...]]

    B --> C[임베딩 모델<br/>text-embedding-3-small]

    C --> D["벡터 출력<br/>[0.1, -0.3, 0.8, ...]<br/>1536 차원"]

    D --> E[L2 정규화<br/>단위 벡터]

    E --> F[벡터 DB 저장]

    style A fill:#e1f5fe
    style B fill:#fff9c4
    style C fill:#c8e6c9
    style D fill:#e1bee7
    style E fill:#ffccbc
    style F fill:#fff59d
```

## 📊 임베딩 모델

### OpenAI Embeddings

| 모델 | 차원 | 성능 | 비용 | 속도 |
|------|------|------|------|------|
| `text-embedding-3-small` | 1536 | 우수 ⭐⭐⭐⭐ | 낮음 💰 | 빠름 ⚡ |
| `text-embedding-3-large` | 3072 | 최상 ⭐⭐⭐⭐⭐ | 중간 💰💰 | 중간 ⚡⚡ |
| `text-embedding-ada-002` | 1536 | 좋음 ⭐⭐⭐ | 낮음 💰 | 빠름 ⚡ |

### 모델 비교 시각화

```mermaid
flowchart TB
    subgraph Small["text-embedding-3-small"]
        S1["✅ 1536 차원"]
        S2["✅ 빠른 검색"]
        S3["✅ 낮은 저장 비용"]
        S4["⚠️ 약간 낮은 표현력"]
    end

    subgraph Large["text-embedding-3-large"]
        L1["✅ 3072 차원"]
        L2["✅ 뛰어난 표현력"]
        L3["✅ 미세한 차이 구분"]
        L4["⚠️ 느린 검색"]
        L5["⚠️ 높은 저장 비용"]
    end

    style Small fill:#c8e6c9
    style Large fill:#e1bee7
```

### 차원 선택 가이드

```
┌─────────────────────────────────────────────────────┐
│                  1536 차원 (small)                   │
├─────────────────────────────────────────────────────┤
│  장점: ✅ 빠른 검색                                 │
│        ✅ 낮은 저장 비용                            │
│        ✅ 적은 메모리 사용                          │
│  단점: ⚠️  약간 낮은 표현력                         │
│  추천: 📌 일반적인 문서 검색                         │
│        📌 실시간 검색 시스템                         │
│        📌 대규모 데이터셋                           │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│                  3072 차원 (large)                   │
├─────────────────────────────────────────────────────┤
│  장점: ✅ 뛰어난 표현력                             │
│        ✅ 미세한 차이 구분                           │
│        ✅ 복잡한 의미 이해                           │
│  단점: ⚠️  느린 검색                                 │
│        ⚠️  높은 저장 비용                            │
│        ⚠️  많은 메모리 사용                          │
│  추천: 📌 정밀한 의미 검색이 필요한 경우             │
│        📌 전문 도메인                               │
│        📌 작은 규모 데이터셋                         │
└─────────────────────────────────────────────────────┘
```

## 🔍 임베딩 전략

### 1. 텍스트 청크 임베딩

```mermaid
flowchart LR
    A[문서] --> B[청킹<br/>~2000 토큰]
    B --> C1[청크 1]
    B --> C2[청크 2]
    B --> C3[청크 N]

    C1 --> E1[임베딩 1]
    C2 --> E2[임베딩 2]
    C3 --> E3[임베딩 N]

    E1 --> V[벡터 DB]
    E2 --> V
    E3 --> V

    style A fill:#e1f5fe
    style C1 fill:#fff9c4
    style C2 fill:#fff9c4
    style C3 fill:#fff9c4
    style V fill:#c8e6c9
```

```python
async def embed_text_units(
    text_units: pd.DataFrame,
    embed_model: BaseEmbeddingModel,
    batch_size: int = 100
) -> np.ndarray:
    """
    텍스트 청크 임베딩
    """
    embeddings = []

    for i in range(0, len(text_units), batch_size):
        batch = text_units.iloc[i:i+batch_size]

        # 배치 임베딩
        batch_embeddings = await embed_model.embed_batch(
            texts=batch['text'].tolist()
        )
        embeddings.extend(batch_embeddings)

    return np.array(embeddings)
```

### 2. 엔티티 설명 임베딩

```mermaid
flowchart TB
    A[엔티티 목록] --> B[설명 텍스트 구성]

    B --> C1["Microsoft:<br/>주요 기술 회사"]
    B --> C2["OpenAI:<br/>AI 연구 기관"]
    B --> C3["GPT-4:<br/>언어 모델"]

    C1 --> E1[임베딩 1]
    C2 --> E2[임베딩 2]
    C3 --> E3[임베딩 3]

    E1 --> M[엔티티-임베딩 매핑]
    E2 --> M
    E3 --> M

    style A fill:#e1f5fe
    style B fill:#fff9c4
    style M fill:#c8e6c9
```

```python
def embed_entities(
    entities: pd.DataFrame,
    embed_model: BaseEmbeddingModel
) -> dict[str, np.ndarray]:
    """
    엔티티 설명 임베딩
    """
    # 설명 텍스트 구성
    texts = [
        f"{e['title']}: {e['description']}"
        for _, e in entities.iterrows()
    ]

    # 임베딩
    embeddings = embed_model.embed_batch(texts)

    # 매핑
    return {
        entity_id: emb
        for entity_id, emb in zip(entities['id'], embeddings)
    }
```

### 3. 커뮤니티 컨텍스트 임베딩

```mermaid
flowchart TB
    A[커뮤니티] --> B[엔티티 수집]

    B --> C["컨텍스트 구성<br/>- 커뮤니티 이름<br/>- 엔티티 목록<br/>- 설명 요약"]

    C --> D[임베딩 생성]

    D --> E[커뮤니티 임베딩]

    style A fill:#e1f5fe
    style C fill:#fff9c4
    style E fill:#c8e6c9
```

```python
def embed_community_contexts(
    communities: pd.DataFrame,
    entities: pd.DataFrame,
    embed_model: BaseEmbeddingModel
) -> dict[str, np.ndarray]:
    """
    커뮤니티 전체 컨텍스트 임베딩
    """
    context_embeddings = {}

    for _, community in communities.iterrows():
        # 커뮤니티 엔티티 수집
        comm_entities = entities[
            entities['community_id'] == community['id']
        ]

        # 컨텍스트 구성
        entity_names = comm_entities['title'].tolist()
        entity_desc = comm_entities['description'].tolist()

        context_text = f"""
        Community: {community['title']}
        Entities: {', '.join(entity_names)}
        Descriptions: {' | '.join(entity_desc[:5])}
        """

        # 임베딩
        embedding = embed_model.embed(context_text)
        context_embeddings[community['id']] = embedding

    return context_embeddings
```

## 📐 유사도 계산

### 코사인 유사도 (Cosine Similarity)

```mermaid
flowchart TB
    A["벡터 A<br/>[0.8, 0.3, 0.9]"]
    B["벡터 B<br/>[0.7, 0.4, 0.85]"]

    A --> DOT["내적 계산<br/>A · B = Σ(ai × bi)"]
    B --> DOT

    A --> NORM["노름 계산<br/>||A|| = √(Σai²)"]
    B --> NORM

    DOT --> COS["cos(θ) = (A · B) / (||A|| × ||B||)"]
    NORM --> COS

    COS --> RESULT["유사도 점수<br/>0.97 (매우 유사)"]

    style A fill:#e1f5fe
    style B fill:#e1bee7
    style RESULT fill:#c8e6c9
```

가장 일반적으로 사용되는 유사도 메트릭:

```python
import numpy as np

def cosine_similarity(
    a: np.ndarray,
    b: np.ndarray
) -> float:
    """
    코사인 유사도 계산
    """
    # 내적
    dot_product = np.dot(a, b)

    # 노름
    norm_a = np.linalg.norm(a)
    norm_b = np.linalg.norm(b)

    # 코사인 유사도
    return dot_product / (norm_a * norm_b)
```

### 유사도 점수 해석

```mermaid
flowchart TB
    S1["0.9 - 1.0<br/>매우 유사<br/>동일한 의미"]
    S2["0.7 - 0.9<br/>유사<br/>관련 주제"]
    S3["0.5 - 0.7<br/>약간 유사<br/>간접적 연관"]
    S4["0.3 - 0.5<br/>거의 관련 없음"]
    S5["0.0 - 0.3<br/>전혀 다른 주제"]

    style S1 fill:#c8e6c9
    style S2 fill:#a5d6a7
    style S3 fill:#fff59d
    style S4 fill:#ffccbc
    style S5 fill:#ffcdd2
```

### 다른 유사도 메트릭

| 메트릭 | 공식 | 특징 | 사용 사례 |
|--------|------|------|----------|
| **Cosine** | A·B / (\|A\|\|B\|) | 방향만, 크기 무시 | 텍스트 임베딩 |
| **Euclidean** | √(Σ(a-b)²) | 거리 기반 | 이미지 임베딩 |
| **Dot Product** | A·B | 원점 기반 | 정규화된 벡터 |

## 🔍 벡터 검색

### Top-K 검색

```mermaid
flowchart TB
    A["쿼리: 'AI 기술'"] --> B[쿼리 임베딩]

    B --> C[벡터 DB]

    C --> D["모든 문서와<br/>유사도 계산"]

    D --> E[상위 K개 추출]

    E --> F["결과: 1. GPT (0.95)<br/>      2. BERT (0.92)<br/>      3. AI (0.88)"]

    style A fill:#e1f5fe
    style B fill:#fff9c4
    style F fill:#c8e6c9
```

```python
def vector_search(
    query_embedding: np.ndarray,
    index: np.ndarray,  # (N, D) 임베딩 행렬
    k: int = 10
) -> list[tuple[int, float]]:
    """
    Top-K 유사 문서 검색
    """
    # 모든 문서와 유사도 계산
    similarities = cosine_similarity_matrix(query_embedding, index)

    # 상위 K개 추출
    top_k_indices = np.argsort(similarities)[::-1][:k]

    return [
        (idx, similarities[idx])
        for idx in top_k_indices
    ]
```

### ANN (Approximate Nearest Neighbor)

대규모 데이터에서는 정확한 검색 대신 근사 검색 사용:

```mermaid
flowchart TB
    subgraph Exact["정확 검색"]
        E1["모든 벡터와 비교"]
        E2["O(N) 복잡도"]
        E3["정확하지만 느림"]
    end

    subgraph ANN["근사 검색"]
        A1["인덱스 기반 검색"]
        A2["O(log N) 복잡도"]
        A3["빠르지만 약간 부정확"]
    end

    style Exact fill:#ffcdd2
    style ANN fill:#c8e6c9
```

```python
import lancedb

def create_vector_index():
    """LanceDB 인덱스 생성"""
    db = lancedb.connect("./output/lancedb")

    # 테이블 생성 (IVF_FLAT 인덱스)
    db.create_table(
        "embeddings",
        data=[
            {
                "id": i,
                "vector": embedding,
                "text": text
            }
            for i, (embedding, text) in enumerate(zip(embeddings, texts))
        ]
    )

    # IVF_FLAT 인덱스 생성
    db.create_index(
        "embeddings",
        vector_column_name="vector",
        index_type="IVF_FLAT",
        metric="cosine"
    )
```

### ANN 검색 성능 비교

| 데이터 크기 | 정확 검색 | ANN 검색 | 속도 향상 |
|-----------|----------|----------|-----------|
| 1K | 5ms | 1ms | 5x |
| 10K | 50ms | 5ms | 10x |
| 100K | 500ms | 50ms | 10x |
| 1M | 5000ms | 200ms | 25x |

## 🎓 임베딩 최적화

### 1. 하이브리드 검색

```mermaid
flowchart TB
    Q[쿼리] --> K[키워드 검색<br/>BM25]
    Q --> V[벡터 검색<br/>임베딩]

    K --> RRF[Reciprocal Rank Fusion]
    V --> RRF

    RRF --> FINAL[결과 병합]

    style Q fill:#e1f5fe
    style K fill:#fff9c4
    style V fill:#c8e6c9
    style FINAL fill:#e1bee7
```

```python
def hybrid_search(
    query: str,
    alpha: float = 0.5  # 키워드 vs 벡터 가중치
) -> list[dict]:
    """
    하이브리드 검색 (키워드 + 벡터)
    """
    # 1. 키워드 검색
    keyword_results = bm25_search(query)

    # 2. 벡터 검색
    vector_results = vector_search(query)

    # 3. 결과 결합 (Reciprocal Rank Fusion)
    scores = {}

    for rank, doc in enumerate(keyword_results):
        scores[doc['id']] = scores.get(doc['id'], 0) + 1 / (rank + 1)

    for rank, doc in enumerate(vector_results):
        scores[doc['id']] = scores.get(doc['id'], 0) + 1 / (rank + 1)

    # 4. 정렬
    sorted_results = sorted(scores.items(), key=lambda x: -x[1])

    return sorted_results
```

### 2. 쿼리 확장

```mermaid
flowchart LR
    A[원본 쿼리<br/>'AI 기술'] --> B[관련 엔티티 추출<br/>GPT, BERT, ML]

    B --> C["확장된 쿼리 벡터<br/>= 0.7 × 원본<br/>  + 0.3 × 엔티티들"]

    C --> D[개선된 검색]

    style A fill:#e1f5fe
    style C fill:#fff9c4
    style D fill:#c8e6c9
```

```python
def expand_query_embedding(
    query: str,
    entities: list[Entity],
    embed_model: BaseEmbeddingModel
) -> np.ndarray:
    """
    관련 엔티티로 쿼리 임베딩 강화
    """
    # 원본 쿼리 임베딩
    query_emb = embed_model.embed(query)

    # 관련 엔티티 임베딩
    entity_embs = [
        embed_model.embed(f"{e['title']}: {e['description']}")
        for e in entities[:5]  # 상위 5개
    ]

    # 가중 평균
    alpha = 0.7  # 쿼리 가중치
    beta = 0.3   # 엔티티 가중치

    expanded_emb = alpha * query_emb + beta * np.mean(entity_embs, axis=0)

    # 재정규화
    return expanded_emb / np.linalg.norm(expanded_emb)
```

### 3. 임베딩 캐싱

```python
import pickle

class EmbeddingCache:
    def __init__(self, cache_path: str):
        self.cache_path = cache_path
        self.cache = self._load_cache()

    def get(self, text: str) -> np.ndarray | None:
        return self.cache.get(text)

    def set(self, text: str, embedding: np.ndarray):
        self.cache[text] = embedding
        with open(self.cache_path, 'wb') as f:
            pickle.dump(self.cache, f)
```

## 📊 성능 벤치마크

| 작업 | 작은 크기 (< 1K) | 중간 크기 (10K) | 큰 크기 (100K+) |
|------|-------------------|-------------------|-------------------|
| **임베딩** | 1-2초 | 10-20초 | 2-5분 |
| **정확 검색** | < 1ms | 10-50ms | 100-500ms |
| **ANN 검색** | < 1ms | 5-20ms | 50-200ms |
| **저장소** | 10MB | 100MB | 1GB+ |

## 🔗 관련 컴포넌트

- [[Storage Module]]: 벡터 데이터베이스
- [[Text Unit]]: 텍스트 임베딩 대상
- [[Entity]]: 엔티티 임베딩
- [[Local Search]]: 임베딩 활용 검색

## 💡 성능 최적화 팁

1. **배치 처리**: 한 번의 API 호출로 여러 텍스트 처리
2. **캐싱**: 동일한 텍스트 재임베딩 방지
3. **ANN 인덱스**: 대규모 데이터에서 근사 검색 사용
4. **차원 축소**: 필요한 경우 차원 축소로 저장 비용 절감

---
*See also: [[Storage Module]], [[Local Search]], [[Text Unit]]*
