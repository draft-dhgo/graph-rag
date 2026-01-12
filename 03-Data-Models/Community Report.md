---
title: Community Report
tags:
  - data-model
  - community-report
  - summary
  - llm-generated
created: 2025-01-12
type: data-model
links:
  - [[Community]]
  - [[Entity]]
  - [[Global Search]]
---

# Community Report

**Community Report(커뮤니티 보고서)**는 [[Community]]에 대한 LLM 생성 요약 및 분석으로, 해당 커뮤니티 내의 엔티티와 관계에 대한 자연어 인사이트를 제공합니다.

## 정의

```python
@dataclass
class CommunityReport(Named):
    id: str                           # 고유 식별자
    short_id: str | None              # 사람이 읽을 수 있는 ID
    title: str                        # 보고서 제목
    community_id: str                 # 상위 커뮤니티 ID
    summary: str = ""                 # 간략한 요약
    full_content: str = ""            # 전체 상세 내용
    rank: float | None = 1.0          # 중요도 점수
    full_content_embedding: list[float] | None  # 시맨틱 임베딩
    attributes: dict[str, Any] | None # 추가 메타데이터
    size: int | None                  # 보고서 범위
    period: str | None                #覆盖 시기
```

## 목적

커뮤니티 보고서는 두 가지 핵심 목적을 제공합니다:

1. **전역 검색**: 광범위한 질문에 대한 커뮤니티 수준 컨텍스트 제공
2. **사람 가독성**: 그래프 구조의 자연어 요약 제공

## 보고서 구조

### 요약 섹션
다음을 포함하는 간결한 개요(2-3문장):
- 커뮤니티의 주요 테마/주제
- 관련 주요 엔티티
- 주요 관계

### 전체 내용 섹션
다음을 포함하는 상세 분석:
- 커뮤니티의 상세 설명
- 중요한 엔티티 관계
- 주요 발견 및 인사이트
- 주목할 만한 패턴

## 생성 프로세스

```
Community → Entity Context → LLM Summarization → Community Report
```

### 단계

1. **컨텍스트 수집**: 엔티티, 관계, 텍스트 단위 수집
2. **프롬프트 구축**: LLM을 위한 컨텍스트 포맷팅
3. **요약 생성**: 간결한 개요 작성
4. **전체 내용 생성**: 상세 분석 작성
5. **내용 임베딩**: 검색을 위한 벡터 임베딩 생성

### 생성 프롬프트

```
엔티티 커뮤니티에 대한 다음 정보가 주어졌습니다:

엔티티:
{entity_list}

관계:
{relationship_list}

컨텍스트:
{context_text}

다음을 제공하세요:
1. 간략한 요약(2-3문장)
2. 상세한 full_content 분석:
   - 주요 테마/주제
   - 주요 엔티티와 그 역할
   - 중요한 관계
   - 주목할 만한 패턴 또는 인사이트
```

## 저장

커뮤니티 보고서는 Parquet 형식으로 저장됩니다:

```python
# output/create_final_community_reports.parquet
columns = [
    "id", "human_readable_id", "community", "summary",
    "full_content", "rank", "size", "period"
]
```

## 사용 예시

### 커뮤니티 보고서 로드

```python
import pandas as pd

reports = pd.read_parquet("output/create_final_community_reports.parquet")

# 커뮤니티의 보고서 가져오기
community_id = "comm-001"
report = reports[reports["community"] == community_id].iloc[0]

print(f"Summary: {report['summary']}")
print(f"Full Content: {report['full_content']}")
```

### 관련 보고서 찾기

```python
# 키워드를 포함하는 보고서 찾기
keyword = "AI research"
matching = reports[
    reports["full_content"].str.contains(keyword, case=False)
]

# 순위별 상위 보고서 가져오기
top_reports = reports.nlargest(10, "rank")
```

## 검색에서의 활용

### 전역 검색

커뮤니티 보고서는 [[Global Search]]의 주요 컨텍스트 소스입니다:

1. 관련 커뮤니티 선택
2. 보고서 검색
3. 맵-리듀스 프로세스에서 보고서 사용
4. 포괄적인 답변 생성

### 예시 플로우

```
Query: "주요 연구 분야는 무엇인가?"

↓

Community Reports:
- comm-001 (AI): "머신러닝과 딥러닝에 중점을 둡니다..."
- comm-002 (Cloud): "클라우드 인프라와 분산 시스템을 다룹니다..."
- comm-003 (Security): "사이버보안과 개인정보보호를 다룹니다..."

↓

Final Answer: "주요 연구 분야는 AI/ML, 클라우드 인프라,
보안입니다..."
```

## 보고서 품질

### 품질에 영향을 미치는 요소

| 요소 | 영향 |
|--------|--------|
| **커뮤니티 응집성** | 잘 정의된 커뮤니티가 더 나은 보고서 생성 |
| **엔티티 설명** | 풍부한 엔티티 설명이 컨텍스트 개선 |
| **LLM 능력** | 더 나은 모델이 더 통찰력 있는 보고서 생성 |
| **컨텍스트 양** | 포괄적인 보고서에 충분한 컨텍스트 필요 |

### 보고서 품질 향상

```yaml
# 엔티티 추출 개선
extract_graph:
  max_gleanings: 2  # 다중 추출 패스

# 커뮤니티 감지 개선
cluster_graph:
  max_cluster_size: 50  # 최적의 커뮤니티 크기

# 보고서 생성 개선
community_reports:
  max_length: 2000  # 더 긴 보고서 허용
  prompt: "custom_report_prompt.txt"  # 사용자 정의 프롬프트 사용
```

## 관련 주제

- [[Community]] - 요약되는 커뮤니티
- [[Entity]] - 커뮤니티의 엔티티
- [[Global Search]] - 커뮤니티 보고서 사용
- [[Leiden Algorithm]] - 커뮤니티 감지

---
*참고: [[Entity]], [[Community]], [[Global Search]], [[Index Module]]*
