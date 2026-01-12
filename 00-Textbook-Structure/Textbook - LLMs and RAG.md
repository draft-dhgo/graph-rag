---
title: "Chapter 4: LLMs and RAG"
tags:
  - chapter
  - foundations
  - llm
  - rag
created: 2025-01-12
type: chapter
links:
  - [[00-GraphRAG-Textbook-Outline]]
  - [[Textbook - NLP Basics]]
  - [[Textbook - Entity Extraction]]
  - [[Textbook - System Architecture]]
---

# Chapter 4: LLMs and RAG

## 학습 목표

이 장을 마치면 다음을 수행할 수 있습니다:
- 최신 LLM을 강화하는 트랜스포머 아키텍처 설명
- 어텐션 메커니즘이 작동하는 방법 이해
- 검색 증강 생성(RAG)의 개념 설명
- 기본 RAG 시스템의 한계 식별
- GraphRAG가 기존 RAG를 어떻게 향상시키는지 이해

---

## 4.1 LLM 아키텍처와 기능

### 4.1.1 대규모 언어 모델이란 무엇인가?

**대규모 언어 모델(LLM)**은 시퀀스에서 다음 토큰을 예측하도록 광범위한 텍스트로 학습된 신경망입니다. 이 간단한 목적함수를 대규모로 적용하면 일관된 텍스트를 생성하고, 질문에 답하고, 복잡한 추론 작업을 수행할 수 있는 모델이 생성됩니다.

```mermaid
flowchart TB
    subgraph Training["LLM 학습 과정"]
        direction TB

        DATA[대규모 텍스트 말뭉치<br/>인터넷, 도서, 논문]

        OBJ[목적: 다음 토큰 예측]

        INPUT["프랑스의 수도는"]

        OUTPUT["파리 (예측)"]

        DATA --> OBJ
        OBJ --> INPUT
        INPUT --> OUTPUT
    end

    style Training fill:#e3f2fd,stroke:#1565c0
```

### 4.1.2 트랜스포머 아키텍처

최신 LLM은 **트랜스포머** 아키텍처("Attention Is All You Need", 2017)를 기반으로 구축됩니다.

#### 핵심 구성 요소

```mermaid
flowchart TB
    subgraph Transformer["트랜스포머 아키텍처"]
        direction TB

        subgraph Input["입력 처리"]
            EMB[토큰 임베딩]
            POS[위치 인코딩]
        end

        subgraph Encoder["인코더 스택"]
            ATT1[셀프 어텐션]
            FF1[피드 포워드]
            N1[층 정규화]
        end

        subgraph Decoder["디코더 스택"]
            ATT2[마스크드 셀프 어텐션]
            ATT3[크로스 어텐션]
            FF2[피드 포워드]
            N2[층 정규화]
        end

        OUTPUT[출력 확률]

        EMB --> POS --> Encoder --> Decoder --> OUTPUT
    end

    style Encoder fill:#e3f2fd,stroke:#1565c0
    style Decoder fill:#f3e5f5,stroke:#7b1fa2
```

#### 인코더 vs. 디코더 모델

| 유형 | 아키텍처 | 예시 | 가장 적합한 |
|------|-------------|----------|----------|
| **인코더 전용** | 트랜스포머 인코더 | BERT, RoBERTa | 이해, 분류 |
| **디코더 전용** | 트랜스포머 디코더 | GPT-4, Claude | 생성, 창작 작업 |
| **인코더-디코더** | 둘 다 | T5, BART | 번역, 요약 |

**GraphRAG는 일반적으로 디코더 전용 모델**(GPT-4, Claude)을 생성에 사용합니다.

### 4.1.3 어텐션 메커니즘

**어텐션**을 통해 모델은 각 출력을 생성할 때 입력의 관련 부분에 집중할 수 있습니다.

#### 셀프 어텐션 직관

```mermaid
flowchart TB
    subgraph Example["셀프 어텐션: 'it' 이해"]
        direction LR

        SENT["동물은 거리를 건너지 못했습니다. 너무 피곤해서 it였습니다"]

        FOCUS["'it'가 무엇을 참조하는가?"]

        ATT["어텐션 가중치:<br/>동물: 0.1<br/>거리: 0.05<br/>it: 0.85<br/>피곤: 0.9"]

        RESULT["모델 학습: 'it'는 '동물'을 참조, '거리' 아님"]

        SENT --> FOCUS --> ATT --> RESULT
    end

    style Example fill:#e8f5e9,stroke:#2e7d32
```

#### 어텐션 작동 방식 (간소화)

```mermaid
flowchart LR
    subgraph Attention["어텐션 계산"]
        direction TB

        Q[쿼리 Q]

        K[키 K]

        V[값 V]

        SCORE["어텐션 점수 = Q × K^T"]

        WEIGHTS["정규화된 가중치 = softmax(점수)"]

        OUTPUT["출력 = 가중치 × V"]

        Q --> SCORE
        K --> SCORE
        SCORE --> WEIGHTS
        V --> OUTPUT
        WEIGHTS --> OUTPUT
    end

    style Attention fill:#f3e5f5,stroke:#7b1fa2
```

**직관:**
- **쿼리**: 찾고 있는 것
- **키**: 검색 대상
- **값**: 검색하고 싶은 것

### 4.1.4 토큰 제한과 컨텍스트 윈도우

모든 LLM에는 **컨텍스트 윈도우**가 있습니다. 한 번에 처리할 수 있는 최대 토큰 수입니다.

| 모델 | 컨텍스트 윈도우 | 의미 |
|--------|----------------|---------------|
| GPT-3.5 | 16K 토큰 | ~12,000 단어 |
| GPT-4 | 128K 토큰 | ~96,000 단어 |
| Claude 2 | 100K 토큰 | ~75,000 단어 |
| Claude 3 | 200K 토큰 | ~150,000 단어 |

```mermaid
flowchart TB
    subgraph Constraint["컨텍스트 윈도우 제약"]
        direction TB

        WIN[컨텍스트 윈도우]

        SYS[시스템 프롬프트]

        RETR[검색된 컨텍스트]

        QUERY[사용자 쿼리]

        ANS[생성된 답변]

        WIN --> SYS
        WIN --> RETR
        WIN --> QUERY
        WIN --> ANS

        RETR -.->|내에 맞아야 함| WIN
    end

    style RETR fill:#ff5252,color:#fff
    style WIN fill:#e3f2fd,stroke:#1565c0
```

**문제:** 지식 베이스가 컨텍스트 윈도우보다 크면 모든 것을 포함할 수 없습니다. **검색**해야 합니다.

**이것이 우리가 RAG가 필요한 이유입니다!**

### 4.1.5 생성 vs. 이해

LLM에는 두 가지 주요 기능이 있습니다.

```mermaid
flowchart TB
    subgraph LLM["LLM 기능"]

        subgraph Understanding["이해 (인코더)"]
            U1[분류]
            U2[감정 분석]
            U3[개체 인식]
            U4[의미적 검색]
        end

        subgraph Generation["생성 (디코더)"]
            G1[텍스트 생성]
            G2[질문 답변]
            G3[요약]
            G4[번역]
        end
    end

    style Understanding fill:#e3f2fd,stroke:#1565c0
    style Generation fill:#f3e5f5,stroke:#7b1fa2
```

**RAG는 둘 다 사용합니다:**
1. **이해**: 사용자 쿼리 해석
2. **생성**: 최종 답변 생성
3. **검색**: 관련 컨텍스트 찾기 (이해와 생성 사이의 다리)

---

## 4.2 프롬프트 엔지니어링

### 4.2.1 좋은 프롬프트의 조건

**프롬프트 엔지니어링**은 LLM에서 원하는 출력을 이끌어내는 입력을 설계하는 기술입니다.

#### 프롬프트 구조

```mermaid
flowchart TB
    subgraph Prompt["프롬프트 구조"]
        direction TB

        SYS[시스템 메시지<br/>"당신은 도움이 되는 어시스턴트입니다..."]

        FEW[피샷 예시<br/>"Q: ...<br/>A: ..."]

        INST[지침<br/>"아래 컨텍스트를 바탕으로 답변하세요"]

        CTX[컨텍스트<br/>검색된 문서]

        Q[쿼리<br/>사용자의 질문]

        SYS --> FEW --> INST --> CTX --> Q
    end

    style SYS fill:#e3f2fd,stroke:#1565c0
    style CTX fill:#e8f5e9,stroke:#2e7d32
    style Q fill:#f3e5f5,stroke:#7b1fa2
```

### 4.2.2 인컨텍스트 학습

**인컨텍스트 학습**을 통해 LLM은 프롬프트에 제공된 예시로부터 가중치 업데이트 없이 학습할 수 있습니다.

```mermaid
flowchart LR
    subgraph ICL["인컨텍스트 학습"]
        direction TB

        EX1["예시 1:<br/>Q: 프랑스의 수도는 어디인가?<br/>A: 파리"]

        EX2["예시 2:<br/>Q: 일본의 수도는 어디인가?<br/>A: 도쿄"]

        QUERY["Q: 스페인의 수도는 어디인가?"]

        MODEL["모델이 패턴 학습 → A: 마드리드"]

        EX1 --> EX2 --> QUERY --> MODEL
    end

    style ICL fill:#e8f5e9,stroke:#2e7d32
```

### 4.2.3 사고 연쇄 프롬프팅

**사고 연쇄(CoT)** 프롬프팅은 모델이 추론 과정을 보여주도록 장려합니다.

```mermaid
flowchart TB
    subgraph CoT["사고 연쇄 프롬프팅"]
        direction TB

        Q["Q: 사과 5개를 가지고 있고, 2개를 먹고 3개를 더 사면 몇 개가 있나?"]

        STANDARD["표준 프롬프트: 답변만 제공 (6)"]

        COT["CoT 프롬프트: '단계별로 생각해봅시다...'"]

        REASONING["추론:<br/>1. 사과 5개로 시작<br/>2. 2개 먹음 → 5 - 2 = 3<br/>3. 3개 더 삼 → 3 + 3 = 6<br/>답: 6"]

        Q --> COT
        COT --> REASONING
    end

    style CoT fill:#f3e5f5,stroke:#7b1fa2
```

### 4.2.4 일반적인 프롬프팅 패턴

| 패턴 | 설명 | GraphRAG 사용 |
|---------|-------------|-----------------|
| **제로샷** | 예시 제공 안 함 | 간단한 쿼리 |
| **피샷** | 예시 제공 | 복잡한 추출 |
| **CoT** | 추론 표시 | 다중 홉 쿼리 |
| **셀프 일관성** | 여러 샘플, 투표 | 신뢰성 향상 |
| **ReAct** | 추론 + 행동 루프 | 대화형 탐색 |

---

## 4.3 검색 증강 생성 (RAG)

### 4.3.1 RAG 패러다임

**검색 증강 생성(RAG)**은 생성 전에 관련 정보를 검색하여 LLM을 강화합니다.

```mermaid
flowchart TB
    subgraph NoRAG["RAG 없이"]
        direction TB
        Q1[쿼리]
        LLM1[LLM]
        A1[답변<br/>(할루시네이션 가능)]

        Q1 --> LLM1 --> A1
    end

    subgraph WithRAG["RAG로"]
        direction TB
        Q2[쿼리]
        RETR[관련 문서<br/>검색]
        LLM2[LLM + 컨텍스트]
        A2[답변<br/>(사실에 기반)]

        Q2 --> RETR --> LLM2 --> A2
    end

    style NoRAG fill:#ffebee,stroke:#c62828
    style WithRAG fill:#e8f5e9,stroke:#2e7d32
```

### 4.3.2 검색이 중요한 이유

**문제:** LLM은 고정된 데이터로 학습되며 모든 것을 알 수 없습니다.

**해결책:** 쿼리 시점에 최신 정보, 도메인별 정보를 검색합니다.

```mermaid
flowchart TB
    subgraph Example["예시: 의학 쿼리"]
        direction TB

        Q["쿼리: '약물 X의 부작용은 무엇입니까? (2024년 승인)'"]

        NO_RAG["RAG 없이:<br/>'제 학습 기준 이후 승인된 약물 X에 대한 정보가 없습니다'"]

        WITH_RAG["RAG로:<br/>1. 최신 약물 X 문서 검색<br/>2. 정확한 부작용 목록 제공"]

        Q --> NO_RAG
        Q --> WITH_RAG
    end

    style NO_RAG fill:#ffebee,stroke:#c62828
    style WITH_RAG fill:#e8f5e9,stroke:#2e7d32
```

### 4.3.3 기본 RAG 아키텍처

```mermaid
flowchart TB
    subgraph Indexing["오프라인: 인덱싱 단계"]
        direction TB

        DOCS[문서]

        CHUNK[문서 청킹]

        EMBED[임베딩 생성]

        STORE[(벡터 저장소)]

        DOCS --> CHUNK --> EMBED --> STORE
    end

    subgraph Query["온라인: 쿼리 단계"]
        direction TB

        Q[사용자 쿼리]

        Q_EMBED[쿼리 임베딩]

        SEARCH[벡터 검색]

        CONTEXT[상위 K개 청크<br/>검색]

        PROMPT[컨텍스트로<br/>프롬프트 구축]

        LLM[LLM 생성]

        ANSWER[최종 답변]

        Q --> Q_EMBED --> SEARCH --> CONTEXT --> PROMPT --> LLM --> ANSWER
    end

    STORE -.->|검색 중| SEARCH

    style Indexing fill:#e3f2fd,stroke:#1565c0
    style Query fill:#f3e5f5,stroke:#7b1fa2
```

### 4.3.4 기본 RAG의 한계

| 한계 | 설명 | 영향 |
|------------|-------------|--------|
| **청크 격리** | 각 청크가 독립적으로 검색됨 | 청크 간 연결 누락 |
| **벡터 전용 검색** | 의미적 유사도에만 기반 | 구조적 관계 누락 |
| **전역 컨텍스트 없음** | "큰 그림"을 볼 수 없음 | 주제적 질문에 취약 |
| **출처 상실** | 정보 출처 추적 어려움 | 신뢰성 저하 |
| **키워드 불일치** | 쿼리 용어가 청크 용어와 일치하지 않음 | 관련 정보 누락 |

```mermaid
flowchart TB
    subgraph Problem["청크 격리 문제"]
        direction TB

        QUERY["쿼리: '이 연구자들은 서로 어떻게 관련되어 있나?'"]

        CHUNKS["검색된 청크:<br/>청크 1: '연구자 A가 X 연구'<br/>청크 5: '연구자 B가 X 인용'<br/>청크 12: '연구자 C가 B와 협력'"]

        MISSING["누락: A → X ← B ← C 연결 그래프"]

        RAG_RESULT["RAG: 청크를 나열하여 반환<br/>GraphRAG: 관계 네트워크 표시"]

        QUERY --> CHUNKS --> MISSING --> RAG_RESULT
    end

    style Problem fill:#fff3e0,stroke:#ef6c00
```

---

## 4.4 RAG에서 GraphRAG로

### 4.4.1 GraphRAG가 추가하는 것

GraphRAG는 기존 RAG에 **구조화된 지식**을 추가합니다.

```mermaid
flowchart TB
    subgraph TradRAG["기존 RAG"]
        direction TB
        VEC[(벡터 저장소)]
        SIM[유사도 검색]
        CHUNKS[텍스트 청크]

        VEC --> SIM --> CHUNKS
    end

    subgraph GraphRAG["GraphRAG"]
        direction TB
        GRAPH[(지식 그래프)]
        TRAVERSE[그래프 순회]
        STRUCT[구조화된 컨텍스트]

        GRAPH --> TRAVERSE --> STRUCT
    end

    subgraph Combined["GraphRAG = RAG + 구조"]
        BOTH[벡터 검색 + 그래프 순회]
        RICH[풍부한 연결 컨텍스트]
        BETTER[더 나은 답변]

        BOTH --> RICH --> BETTER
    end

    style TradRAG fill:#fff3e0,stroke:#ef6c00
    style GraphRAG fill:#e8f5e9,stroke:#2e7d32
    style Combined fill:#e3f2fd,stroke:#1565c0
```

### 4.4.2 로컬 vs. 글로벌 검색

GraphRAG는 두 가지 검색 모드를 도입합니다.

#### 로컬 검색 (개체 중심)

```mermaid
flowchart TB
    subgraph Local["로컬 검색"]
        direction TB

        QUERY["쿼리: '마리 퀴리에 대해 말해주세요'"]

        ENTITY[개체 찾기: 마리 퀴리]

        NEIGHBOR[이웃 찾기<br/>발견: 방사선<br/>수상: 노벨상<br/>출생지: 바르샤바]

        CONTEXT[개체 + 관계로부터<br/>컨텍스트 구축]

        QUERY --> ENTITY --> NEIGHBOR --> CONTEXT
    end

    style Local fill:#e3f2fd,stroke:#1565c0
```

#### 글로벌 검색 (커뮤니티 중심)

```mermaid
flowchart TB
    subgraph Global["글로벌 검색"]
        direction TB

        QUERY["쿼리: '이 말뭉치의 주요 주제는 무엇입니까?'"]

        COMM[관련 커뮤니티 식별<br/>커뮤니티: 과학적 발견<br/>커뮤니티: 수상자<br/>커뮤니티: 유럽 과학자]

        SUMM[커뮤니티 요약 검색]

        CONTEXT[커뮤니티 수준 이해로부터<br/>컨텍스트 구축]

        QUERY --> COMM --> SUMM --> CONTEXT
    end

    style Global fill:#f3e5f5,stroke:#7b1fa2
```

### 4.4.3 비교 표

| 측면 | 기존 RAG | GraphRAG 로컬 | GraphRAG 글로벌 |
|--------|----------------|----------------|-----------------|
| **검색 단위** | 텍스트 청크 | 개체 + 이웃 | 커뮤니티 요약 |
| **가장 적합한** | 사실 기반 질문 | 개체별 쿼리 | 주제적 질문 |
| **컨텍스트 유형** | 평평한 텍스트 | 구조화된 관계 | 요약된 주제 |
| **확장성** | 우수함 | 양호함 | 매우 양호함 |
| **복잡성** | 낮음 | 중간 | 중간 |

---

## 4.5 완전한 GraphRAG 파이프라인

### 4.5.1 엔드 투 엔드 아키텍처

```mermaid
flowchart TB
    subgraph Indexing["1단계: 인덱싱 (오프라인)"]
        direction TB

        DOCS[원시 문서]

        CHUNK[텍스트 유닛으로<br/>청킹]

        EXT[개체 및 관계 추출<br/>(LLM 기반)]

        GRAPH[지식 그래프 구축]

        COMM[커뮤니티 탐지<br/>(Leiden 알고리즘)]

        SUMM[커뮤니티 요약 생성<br/>(LLM 기반)]

        EMBED[임베딩 생성<br/>(텍스트 유닛, 개체, 커뮤니티)]

        STORE[(영구 저장소)]

        DOCS --> CHUNK --> EXT --> GRAPH --> COMM --> SUMM --> EMBED --> STORE
    end

    subgraph Query["2단계: 쿼리 (온라인)"]
        direction TB

        Q[사용자 쿼리]

        Q_ENT[쿼리 개체 추출]

        MODE[선택: 로컬 / 글로벌 / 혼합]

        RETR[검색:<br/>- 개체 이웃<br/>- 커뮤니티 요약<br/>- 텍스트 유닛]

        BUILD[컨텍스트 구축]

        GEN[답변 생성<br/>(LLM, 인용 포함)]

        Q --> Q_ENT --> MODE --> RETR --> BUILD --> GEN
    end

    STORE -.->|제공| RETR

    style Indexing fill:#e3f2fd,stroke:#1565c0
    style Query fill:#f3e5f5,stroke:#7b1fa2
```

### 4.5.2 핵심 혁신

1. **LLM 기반 추출**: 비정형 텍스트에서 구조화된 지식 추출
2. **커뮤니티 탐지**: 주제 그룹 자동 발견
3. **계층적 조직**: 다중 수준 커뮤니티 구조
4. **이중 검색 모드**: 로컬(개체) 및 글로벌(커뮤니티)
5. **풍부한 인용**: 모든 사실을 소스 텍스트로 추적 가능

---

## 장 요약

이 장에서는 LLM과 RAG를 탐구했습니다.

**LLM 기초:**
- **트랜스포머**는 셀프 어텐션으로 텍스트 처리
- **컨텍스트 윈도우**가 처리할 수 있는 텍스트 양 제한
- **생성**에는 이해 + 검색이 팩트 사실 정확성에 필요

**프롬프트 엔지니어링:**
- **인컨텍스트 학습**이 예시를 통해 교육
- **사고 연쇄**가 추론 개선
- 잘 설계된 프롬프트는 RAG 시스템에 필수적

**RAG 개념:**
- **RAG**는 생성 전에 관련 컨텍스트를 검색
- **해결** LLM 한계: 정적 지식, 할루시네이션
- **기존 RAG**는 텍스트 청크에 대한 벡터 유사도 사용

**GraphRAG 향상:**
- **추가** 구조화된 지식(그래프)를 RAG에
- **두 모드**: 로컬(개체 중심) 및 글로벌(커뮤니티 중심)
- **더 나은** 복잡한 다중 홉 및 주제적 질문

**다음 단계:**
기초가 완료되었습니다. [[Part 2: Core Concepts]]는 [[Textbook - Entity Extraction]]부터 GraphRAG 구성 요소를 자세히 살펴봅니다.

---

## 복습 문제

1. 인코더 전용과 디코더 전용 트랜스포머 모델의 핵심 차이점은 무엇입니까?
2. 어텐션 메커니즘을 직관적으로 설명하세요.
3. 검색 증강 생성이 필요한 이유는 무엇입니까?
4. 기본 RAG의 주요 한계는 무엇입니까?
5. GraphRAG의 로컬 검색과 글로벌 검색 차이점을 설명하세요.
6. GraphRAG 파이프라인의 두 가지 주요 단계는 무엇입니까?

---

## 연습 문제

1. 뉴스 기사에서 개체와 관계를 추출하는 프롬프트를 설계하세요. LLM에 어떤 지침을 주시겠습니까?

2. "양자 역학과 상대성론을 연결하는 것"이라는 쿼리에 로컬 또는 글로벌 GraphRAG 검색을 사용하시겠습니까? 이유는?

3. 기존 RAG와 GraphRAG가 다음을 어떻게 처리하는지 비교하세요: "이 데이터셋의 모든 연구자가 서로 어떻게 연결되어 있는지 요약하세요."

---

## 추가 참고자료

- "Attention Is All You Need" (Vaswani et al., 2017)
- "Retrieval-Augmented Generation for Knowledge-Intensive NLP" (Lewis et al., 2020)
- "From Local to Global: A Graph RAG Approach to Query-Focused Summarization" (GraphRAG 논문)
- 프롬프트 엔지니어링 가이드 (OpenAI, Anthropic)
