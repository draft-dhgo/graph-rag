---
title: GraphRAG 지식 허브
tags:
  - home
  - graphrag
  - introduction
created: 2025-01-12
type: map-of-content
---

# GraphRAG 지식 허브

Microsoft의 **GraphRAG (Graph-based Retrieval-Augmented Generation)** 시스템에 대한 포괄적인 문서에 오신 것을 환영합니다.

## 📚 빠른 탐색

### 🏠 여기서 시작하기
- [[GraphRAG란 무엇인가?]] - 소개 및 개요
- [[Getting Started]] - 빠른 시작 가이드

### 🏗️ 아키텍처
- [[Architecture Overview]] - 시스템 아키텍처 및 설계
- [[Data Flow]] - 파이프라인을 통한 데이터 흐름
- [[Technology Stack]] - 의존성 및 기술

### 🔧 핵심 모듈
- [[Index Module]] - 지식 그래프 인덱싱 파이프라인
- [[Query Module]] - 검색 및 검색 방법
- [[Configuration Module]] - 구성 관리
- [[Storage Module]] - 데이터 지속성 계층
- [[Language Model Module]] - LLM 통합
- [[Local Search]] - 로컬 검색 방법
- [[Global Search]] - 글로벌 검색 방법
- [[DRIFT Search]] - DRIFT 검색 방법

### ⚙️ 구성 (Configuration)
- [[Configuration Reference]] - 전체 구성 옵션 레퍼런스
- [[Model Configuration]] - LLM 및 임베딩 모델 설정
- [[Storage Configuration]] - 스토리지 및 데이터베이스 설정

### 📊 데이터 모델
- [[Entity]] - 지식 그래프 엔티티
- [[Relationship]] - 엔티티 관계
- [[Community]] - 커뮤니티 탐지
- [[Community Report]] - 커뮤니티 요약
- [[Text Unit]] - 문서 청크
- [[Covariate]] - 추가 메타데이터
- [[Document]] - 소스 문서

### 🔌 API 레퍼런스
- [[Indexing API]] - 지식 그래프 구축
- [[Query API]] - 검색 작업

### 📈 리포트
- [[Summary Report]] - 경영진 요약
- [[Module Comparison]] - 모듈 분석

## 🔍 검색 방법

GraphRAG는 4가지 검색 방법을 지원합니다:

| 방법 | 최적 사용 경우 | 설명 |
|--------|----------|-------------|
| [[Local Search]] | 구체적인 질문 | 엔티티 중심, 2-3 홉 컨텍스트 |
| [[Global Search]] | 광범위한 질문 | 커뮤니티 수준 집계 |
| [[DRIFT Search]] | 복잡한 다중 홉 | 반복적 정제 |

## 🎯 핵심 개념

### 지식 그래프 추출
GraphRAG는 자동으로 식별합니다:
- **엔티티** - 명명된 엔티티 (사람, 조직, 위치)
- **관계** - 엔티티 간의 연결
- **주장** - 텍스트의 사실적 단언

### 커뮤니티 탐지
계층적 구조로 관련 엔티티 클러스터를 발견하기 위해 **Leiden 알고리즘**을 사용합니다.

### 벡터 임베딩
다음에 대한 임베딩을 생성합니다:
- 텍스트 청크
- 엔티티 설명
- 커뮤니티 컨텍스트

## 🚀 빠른 시작

```bash
# 새 프로젝트 초기화
graphrag init --root ./my-project

# 지식 그래프 구축
graphrag index --config ./my-project/settings.yaml

# 그래프 쿼리
graphrag query --method local --query "AI 연구에 누가 관여하고 있나?"
```

## 📖 학습 경로

1. **초급자**: [[GraphRAG란 무엇인가?]] → [[Getting Started]]
2. **중급**: [[Architecture Overview]] → [[Index Module]] → [[Query Module]]
3. **고급**: [[Entity]] → [[Community]] → [[Relationship]]

## 🔗 관련 주제

- [[Leiden Algorithm]] - 커뮤니티 탐지

---
*마지막 업데이트: 2025-01-12*
