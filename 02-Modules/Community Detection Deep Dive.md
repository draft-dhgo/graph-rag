---
title: Community Detection Deep Dive
tags:
  - deep-dive
  - community-detection
  - leiden
  - clustering
  - graph-algorithm
created: 2025-01-12
type: deep-dive
links:
  - [[Community]]
  - [[Leiden Algorithm]]
  - [[Index Module]]
  - [[Entity]]
---

# Community Detection Deep Dive

커뮤니티 감지(Community Detection)은 지식 그래프에서 밀접하게 연결된 엔티티 그룹을 식별하여, 문서의 주제적 구조를 발견하는 핵심 알고리즘입니다.

## 목차

### 1. 개요
- [커뮤니티 감지의 목적](#-커뮤니티-감지의-목적)
- [빗대어 보기: 소셜 미디어 그룹 형성](#-빗대어-보기-소셜-미디어-그룹-형성)

### 2. Leiden 알고리즘
- [알고리즘 원리](#-알고리즘-원리)
- [수학적 기초](#-수학적-기초)
- [4단계 상세 분석](#4단계-상세-분석)

### 3. 계층적 구조
- [계층적 커뮤니티](#-계층적-커뮤니티-구조)
- [레벨별 특성](#-레벨별-특성)

### 4. 구현
- [Python 구현](#python-구현)
- [엔티티-커뮤니티 매핑](#엔티티-커뮤니티-매핑)

### 5. 품질 평가
- [품질 메트릭](#-품질-메트릭)
- [커뮤니티 분석](#-커뮤니티-분석)

### 6. 고급 기법
- [다중 레벨 최적화](#다중-레벨-최적화)
- [템포럴 커뮤니티](#템포럴-커뮤니티)
- [오버래핑 허용](#-오버래핑-허용)

---

## 🎯 커뮤니티 감지의 목적

```mermaid
flowchart LR
    A[엔티티 그래프] --> B[커뮤니티 감지]
    B --> C[주제 발견]
    B --> D[요약 생성]
    B --> E[글로벌 서치]
    B --> F[계층적 조직]

    style A fill:#e1f5fe
    style B fill:#fff9c4
    style C fill:#c8e6c9
    style D fill:#e1bee7
    style E fill:#ffccbc
    style F fill:#fff59d
```

1. **주제 발견**: 관련 엔티티를 주제별로 그룹화
2. **요약 생성**: 커뮤니티별로 자연어 요약 생성
3. **글로벌 서치**: 커뮤니티 레벨의 질문 답변
4. **계층적 조직**: 다양한 추상화 수준 제공

## 📖 빗대어 보기: 소셜 미디어 그룹 형성

커뮤니티 감지는 **소셜 미디어에서 자연스럽게 형성되는 관심 그룹**과 유사합니다:

| 소셜 미디어 | GraphRAG 커뮤니티 |
|-------------|------------------|
| 같은 주제에 관심 | 같은 커뮤니티 엔티티 |
| 자주 상호작용 | 높은 관계 가중치 |
| 하위 그룹 형성 | 계층적 레벨 |
| 그룹 규모 다양성 | 다양한 커뮤니티 크기 |
| 그룹 간 연결 | 커뮤니티 간 엣지 |

```mermaid
flowchart TB
    subgraph Social["소셜 네트워크"]
        SG1["📱 AI 그룹<br/>- 딥러닝<br/>- NLP<br/>- CV"]
        SG2["📱 클라우드 그룹<br/>- AWS<br/>- Azure<br/>- GCP"]
        SG3["📱 개발자 그룹<br/>- Python<br/>- JS<br/>- Go"]
    end

    subgraph GraphRAG["GraphRAG 커뮤니티"]
        GG1["🧠 AI 연구<br/>- GPT<br/>- BERT<br/>- Transformer"]
        GG2["☁️ 클라우드 서비스<br/>- Azure<br/>- AWS<br/>- GCP"]
        GG3["💻 프로그래밍<br/>- Python<br/>- TypeScript<br/>- Go"]
    end

    style Social fill:#e3f2fd
    style GraphRAG fill:#f3e5f5
```

## 🏗️ Leiden 알고리즘 심층 분석

### 알고리즘 원리

Leiden 알고리즘은 모듈성(Modularity)을 최적화하여 그래프를 커뮤니티로 분할합니다.

```mermaid
flowchart TB
    START((시작)) --> INIT["각 노드를<br/>자체 커뮤니티로 초기화<br/>🏁 초기 상태"]

    INIT --> PHASE1["📍 1단계: 로컬 이동<br/>노드를 이웃 커뮤니티로 이동<br/>모듈성 개산 확인"]

    PHASE1 --> CHECK1{"더 이상<br/>개선 없음?"}

    CHECK1 -->|아니오| PHASE1
    CHECK1 -->|예| PHASE2["🔧 2단계: 정제<br/>커뮤니티 분할<br/>연결성 보장"]

    PHASE2 --> REFINE{"연결성<br/>확인"}

    REFINE -->|미달성| PHASE2
    REFINE -->|달성| PHASE3["📦 3단계: 집계<br/>슈퍼노드 생성<br/>집계된 그래프 구축"]

    PHASE3 --> LEVEL{"더 집계할<br/>레벨 있음?"}

    LEVEL -->|예| PHASE1
    LEVEL -->|아니오| HIER["🌳 4단계: 계층 구조 생성<br/>부모-자식 관계 설정"]

    HIER --> END((완료))

    style START fill:#c8e6c9
    style END fill:#ffcdd2
    style PHASE1 fill:#e1f5fe,stroke:#1976d2,stroke-width:3px
    style PHASE2 fill:#fff3e0,stroke:#f57c00,stroke-width:3px
    style PHASE3 fill:#e8f5e9,stroke:#388e3c,stroke-width:3px
    style HIER fill:#f3e5f5,stroke:#7b1fa2,stroke-width:3px
```

### 4단계 상세 분석

```mermaid
flowchart TB
    subgraph P1["Phase 1: 로컬 이동"]
        direction TB
        P1A["노드 선택"]
        P1B["이웃 커뮤니티 계산"]
        P1C["최대 이득 확인"]
        P1D["이동 수행"]
    end

    subgraph P2["Phase 2: 정제"]
        direction TB
        P2A["단절 성분 식별"]
        P2B["비연결 커뮤니티 분할"]
        P2C["연결성 검증"]
    end

    subgraph P3["Phase 3: 집계"]
        direction TB
        P3A["커뮤니티를 노드로 축소"]
        P3B["내부 에지 제거"]
        P3C["새 그래프 생성"]
    end

    subgraph P4["Phase 4: 계층 구조"]
        direction TB
        P4A["레벨 기록"]
        P4B["부모-자식 링크"]
        P4C["트리 구조 완성"]
    end

    P1 --> P2 --> P3 --> P4

    style P1 fill:#e1f5fe
    style P2 fill:#fff3e0
    style P3 fill:#e8f5e9
    style P4 fill:#f3e5f5
```

## 📐 수학적 기초

### Constant Potts Model (CPM)

Leiden은 CPM 품질 함수를 최적화합니다:

$$
\mathcal{H} = \sum_{i,j} A_{ij} \cdot \delta(\sigma(i), \sigma(j)) - \gamma \sum_i \sum_{\sigma} n_\sigma(i,j)
$$

여기서:
- $A_{ij}$: 인접 행렬 (엔티티 i, j 간 연결)
- $\delta$: Kronecker delta (동일 커뮤니티 여부)
- $\gamma$: 해상도 파라미터
- $n_\sigma(i,j)$: 가능한 엣지 수

### 모듈성 게인

노드가 커뮤니티 C로 이동할 때의 게인:

$$
\Delta \mathcal{H} = \mathcal{H}_{new} - \mathcal{H}_{old}
$$

양수 게인인 경우에만 이동 수행.

### 파라미터 영향

```mermaid
flowchart TB
    subgraph "해상도(γ) 파라미터 영향"
        GAMMA_LOW["γ 낮음 (0.5)<br/>큰 커뮤니티<br/>적은 수"]
        GAMMA_HIGH["γ 높음 (2.0)<br/>작은 커뮤니티<br/>많은 수"]
    end

    subgraph "max_cluster_size 영향"
        SIZE_SMALL["50<br/>세분화된 주제"]
        SIZE_LARGE["200<br/>큰 주제 그룹"]
    end

    style GAMMA_LOW fill:#c8e6c9
    style GAMMA_HIGH fill:#ffcdd2
    style SIZE_SMALL fill:#fff9c4
    style SIZE_LARGE fill:#e1bee7
```

| 파라미터 | 증가 시 효과 | 감소 시 효과 |
|----------|----------------|----------------|
| `max_cluster_size` | 더 큰 커뮤니티 | 더 작은 커뮤니티 |
| `resolution` | 더 많은 작은 커뮤니티 | 더 적은 큰 커뮤니티 |
| `use_lcc=false` | 모든 노드 포함 | 연결된 노드만 |

## 🔍 GraphRAG에서의 활용

### 구성 매개변수

```yaml
cluster_graph:
  max_cluster_size: 50      # 목표 커뮤니티 크기
  use_lcc: true            # 최대 연결 성분만 사용
  seed: 42                 # 재현성을 위한 시드
  resolution: 1.0          # gamma 값 (해상도)
```

## 📊 계층적 커뮤니티 구조

### 레벨별 특성

```mermaid
flowchart TB
    L0["📍 Level 0: Root<br/>모든 엔티티<br/>1개 커뮤니티<br/>─────────<br/>🌐 전체 데이터셋"]

    L0 --> L1["📍 Level 1: 주요 주제<br/>3-10개 커뮤니티<br/>─────────<br/>도메인 최상위 수준<br/>예: AI, 클라우드, 보안"]

    L1 --> L2["📍 Level 2: 하위 주제<br/>10-50개 커뮤니티<br/>─────────<br/>중간 세부 수준<br/>예: 딥러닝, NLP, 컴퓨터비전"]

    L2 --> L3["📍 Level 3: 세부 주제<br/>50-200개 커뮤니티<br/>─────────<br/>가장 구체적 수준<br/>예: GPT, BERT, ViT"]

    style L0 fill:#c8e6c9,stroke:#2e7d32,stroke-width:3px
    style L1 fill:#fff3e0,stroke:#ef6c00,stroke-width:3px
    style L2 fill:#e1f5fe,stroke:#1565c0,stroke-width:3px
    style L3 fill:#f3e5f5,stroke:#7b1fa2,stroke-width:3px
```

### 실제 예시 구조

```
Level 0: comm-000 (모든 엔티티)
├── Level 1: comm-001 (AI 연구)
│   ├── Level 2: comm-004 (머신러닝)
│   │   ├── Level 3: comm-010 (딥러닝)
│   │   │   └── 엔티티: GPT, BERT, Transformer
│   │   └── Level 3: comm-011 (강화학습)
│   │       └── 엔티티: DQN, PPO, A3C
│   └── Level 2: comm-005 (자연어 처리)
│       ├── Level 3: comm-012 (번역)
│       └── Level 3: comm-013 (요약)
├── Level 1: comm-002 (클라우드)
│   ├── Level 2: comm-006 (인프라)
│   └── Level 2: comm-007 (서비스)
└── Level 1: comm-003 (보안)
```

### 커뮤니티 트리 시각화

```mermaid
flowchart TB
    ROOT["🌐 Root<br/>comm-000"]

    ROOT --> C1["🤖 AI Research<br/>comm-001"]
    ROOT --> C2["☁️ Cloud<br/>comm-002"]
    ROOT --> C3["🔒 Security<br/>comm-003"]

    C1 --> C1A["🧠 ML<br/>comm-004"]
    C1 --> C1B["💬 NLP<br/>comm-005"]

    C1A --> C1A1["🔮 Deep Learning<br/>comm-010"]
    C1A --> C1A2["🎮 RL<br/>comm-011"]

    C2 --> C2A["🏗️ Infrastructure<br/>comm-006"]
    C2 --> C2B["📦 Services<br/>comm-007"]

    style ROOT fill:#c8e6c9
    style C1 fill:#fff3e0
    style C2 fill:#e1f5fe
    style C3 fill:#ffcdd2
    style C1A fill:#fff9c4
    style C1B fill:#fff9c4
    style C2A fill:#e1bee7
    style C2B fill:#e1bee7
```

## 🔧 구현 상세

### Python 구현

```python
import networkx as nx
from graspologic.partition import hierarchical_leiden

def detect_communities(
    graph: nx.Graph,
    max_cluster_size: int = 50,
    seed: int = 42
) -> dict:
    """
    Leiden 알고리즘으로 커뮤니티 감지
    """
    # NetworkX 그래프를 graspologic 형식으로 변환
    adjacency = nx.to_numpy_array(graph)

    # 계층적 Leiden 실행
    partition = hierarchical_leiden(
        adjacency,
        max_cluster_size=max_cluster_size,
        random_seed=seed
    )

    # 결과 파싱
    communities = {}
    for node_id, community_id in partition.items():
        if community_id not in communities:
            communities[community_id] = []
        communities[community_id].append(node_id)

    return communities
```

### 엔티티-커뮤니티 매핑

```mermaid
flowchart TB
    A[엔티티 목록] --> B[관계 목록]
    B --> C[그래프 구성]

    C --> D[Leiden 알고리즘]
    D --> E[커뮤니티 할당]

    E --> F[엔티티-커뮤니티 매핑]
    F --> G[부모-자식 관계]
    G --> H[최종 커뮤니티 구조]

    style A fill:#e1f5fe
    style D fill:#fff9c4
    style H fill:#c8e6c9
```

```python
def assign_entities_to_communities(
    entities: pd.DataFrame,
    relationships: pd.DataFrame
) -> pd.DataFrame:
    """
    엔티티에 커뮤니티 ID 할당
    """
    # 그래프 생성
    G = nx.Graph()

    for _, entity in entities.iterrows():
        G.add_node(entity['id'])

    for _, rel in relationships.iterrows():
        G.add_edge(rel['source'], rel['target'], weight=rel['weight'])

    # 커뮤니티 감지
    partition = detect_communities(G)

    # 엔티티에 커뮤니티 ID 추가
    entity_to_comm = {}
    for comm_id, members in partition.items():
        for entity_id in members:
            entity_to_comm[entity_id] = comm_id

    entities['community_id'] = entities['id'].map(entity_to_comm)

    return entities
```

## 📈 커뮤니티 품질 평가

### 품질 메트릭

```mermaid
radar-beta
    title 커뮤니티 품질 지표
    axis Modularity["모듈성<br/>0.3-0.7", 0.5]
    axis Conductance["전도성<br/>&lt;0.3", 0.2]
    axis Coverage["커버리지<br/>&gt;0.95", 0.98]
    axis Silhouette["실루엣<br/>&gt;0.5", 0.6]
```

| 메트릭 | 설명 | 좋은 값 | 의미 |
|--------|------|---------|------|
| **Modularity** | 커뮤니티 내 밀집도 | 0.3 - 0.7 | 높을수록 좋은 분리 |
| **Conductance** | 외부 연결 비율 | < 0.3 | 낮을수록 좋음 |
| **Coverage** | 할당된 엔티티 비율 | > 0.95 | 높을수록 좋음 |
| **Silhouette** | 클러스터 품질 | > 0.5 | 높을수록 좋음 |

### 커뮤니티 분석

```python
def analyze_community_quality(
    graph: nx.Graph,
    partition: dict
) -> dict:
    """
    커뮤니티 품질 분석
    """
    import networkx.algorithms.community as nx_comm

    # 모듈러리티
    modularity = nx_comm.quality.modularity(
        graph,
        list(partition.values())
    )

    # 커버리지
    coverage = len(partition) / graph.number_of_nodes()

    # 평균 커뮤니티 크기
    comm_sizes = [len(members) for members in partition.values()]
    avg_size = sum(comm_sizes) / len(comm_sizes)

    # 크기 분포
    size_distribution = {
        'min': min(comm_sizes),
        'max': max(comm_sizes),
        'std': pd.Series(comm_sizes).std()
    }

    return {
        'modularity': modularity,
        'coverage': coverage,
        'avg_size': avg_size,
        'size_distribution': size_distribution
    }
```

### 품질 평가 흐름

```mermaid
flowchart TB
    START([커뮤니티 할당 완료]) --> MOD[모듈성 계산]

    MOD --> MOD_OK{modularity<br/>> 0.3?}

    MOD_OK -->|아니오| ADJ1["resolution 감소<br/>재실행"]
    MOD_OK -->|예| COND[전도성 계산]

    ADJ1 --> MOD

    COND --> COND_OK{conductance<br/>&lt; 0.3?}

    COND_OK -->|아니오| ADJ2["max_cluster_size 조정<br/>재실행"]
    COND_OK -->|예| SIL[실루엣 계산]

    ADJ2 --> MOD

    SIL --> SIL_OK{silhouette<br/>> 0.5?}

    SIL_OK -->|아니오| WARN[⚠️ 품질 경고]
    SIL_OK -->|예| SUCCESS[✅ 품질 좋음]

    style START fill:#e1f5fe
    style SUCCESS fill:#c8e6c9
    style WARN fill:#fff9c4
    style ADJ1 fill:#ffcdd2
    style ADJ2 fill:#ffcdd2
```

## 🎓 고급 기법

### 1. 다중 레벨 최적화

```mermaid
flowchart TB
    A[전체 그래프] --> B1[Level 1<br/>max_size=100]
    A --> B2[Level 2<br/>max_size=50]
    A --> B3[Level 3<br/>max_size=25]

    B1 --> C1["주요 주제<br/>5-10개"]
    B2 --> C2["중간 주제<br/>20-50개"]
    B3 --> C3["세부 주제<br/>50-100개"]

    C1 --> D[통합된<br/>계층 구조]
    C2 --> D
    C3 --> D

    style A fill:#e1f5fe
    style D fill:#c8e6c9
```

```python
def optimize_multi_level(
    graph: nx.Graph,
    level_targets: dict = {2: 30, 3: 10}
) -> dict:
    """
    특정 레벨에서 목표 커뮤니티 크기 달성
    """
    partition = {}

    for level, target_size in level_targets.items():
        # 해당 레벨 감지
        level_partition = hierarchical_leiden(
            graph,
            max_cluster_size=target_size
        )

        # 결과 병합
        for node, comm in level_partition.items():
            partition[f"{level}_{comm}"] = node

    return partition
```

### 2. 템포럴 커뮤니티

```mermaid
flowchart LR
    T1["T1: 커뮤니티 A<br/>엔티티: 1,2,3"]
    T2["T2: 커뮤니티 A<br/>엔티티: 1,2,4"]
    T3["T3: 커뮤니티 A<br/>엔티티: 1,4,5"]

    T1 -->|"진화"| T2
    T2 -->|"진화"| T3

    style T1 fill:#c8e6c9
    style T2 fill:#fff9c4
    style T3 fill:#ffcdd2
```

```python
def detect_temporal_communities(
    graphs: list[nx.Graph],  # 시간별 그래프
    window: int = 3
) -> dict:
    """
    시간에 따른 커뮤니티 진화 추적
    """
    communities_over_time = []

    for i, graph in enumerate(graphs):
        partition = detect_communities(graph)
        communities_over_time.append({
            'time': i,
            'communities': partition
        })

    # 커뮤니티 연속성 분석
    return track_community_evolution(communities_over_time, window)
```

### 3. 오버래핑 허용

```mermaid
flowchart TB
    E1["엔티티 A<br/>AI 연구자"]
    E2["엔티티 B<br/>NLP 전문가"]

    E1 --> C1["커뮤니티 1<br/>AI 연구"]
    E2 --> C1
    E1 --> C2["커뮤니티 2<br/>딥러닝"]
    E2 -.->|약한 연결| C2

    style C1 fill:#c8e6c9
    style C2 fill:#fff9c4
    style E1 fill:#e1bee7
    style E2 fill:#e1bee7
```

```python
def allow_overlapping_communities(
    graph: nx.Graph,
    entities: pd.DataFrame
) -> dict:
    """
    엔티티가 여러 커뮤니티에 속하도록 허용
    """
    from sklearn.cluster import SpectralClustering

    # 엔티티 임베딩 기반 클러스터링
    embeddings = np.array([
        e['description_embedding']
        for _, e in entities.iterrows()
    ])

    # 여러 클러스터링 실행
    n_communities = 10
    overlapping = {}

    for run in range(3):  # 3번 실행
        clustering = SpectralClustering(
            n_clusters=n_communities,
            random_state=run
        )
        labels = clustering.fit_predict(embeddings)

        for entity_idx, label in enumerate(labels):
            entity_id = entities.iloc[entity_idx]['id']

            if entity_id not in overlapping:
                overlapping[entity_id] = []
            overlapping[entity_id].append(f"run{run}_comm{label}")

    return overlapping
```

## 🔗 관련 컴포넌트

- [[Community]]: 커뮤니티 데이터 모델
- [[Community Report]]: 커뮤니티 요약
- [[Entity]]: 커뮤니티 구성원
- [[Global Search]]: 커뮤니티 레벨 검색

## 💡 성능 최적화 팁

1. **그래프 필터링**: 약한 연결 제거로 처리 속도 향상
2. **병렬 처리**: 독립적 하위그래프에서 병렬 실행
3. **캐싱**: 재계산 방지
4. **증분 업데이트**: 새로운 엔티티만 재할당

---
*See also: [[Community]], [[Leiden Algorithm]], [[Index Module]]*
