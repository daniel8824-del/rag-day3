---
title: QMD를 활용한 로컬 RAG의 장점 — vault 실측 케이스
tags:
  - 프로젝트/QMD
  - 프로젝트/wiki-시스템
  - 강의/Day3
  - QMD
  - GBrain
  - Graphify
  - 로컬RAG
date: 2026-05-14
---

# QMD를 활용한 로컬 RAG의 장점 — vault 실측 케이스

## TL;DR

**QMD는 "본인 노트 폴더 검색"이라는 좁은 문제에 최적화된 단일 도구다.**
LangChain·LlamaIndex 같은 일반 리트리버가 RAG 앱 빌드용 범용 부품 모음이라면, QMD는 그 부품을 박스 밖에서 1줄 명령으로 묶어 놓은 통합 결과물이다.
보강이 필요할 때는 GBrain(한글·백링크)과 Graphify(community·god nodes)가 직교 layer로 협력한다.

**핵심 5가지 장점**:
1. **0원·0 토큰** | 외부 API 없이 로컬 임베딩 + SQLite
2. **다국어 무료** | bge-m3 (한·영·일·중) 기본 내장
3. **1줄 시작** | `qmd query "..."` 한 줄로 BM25 + 벡터 + reranker 통합
4. **데이터 외부 유출 없음** | 회사 노트·고객 정보 전부 로컬
5. **3-layer 협력** | QMD 1차 → GBrain 한글 보강 → Graphify 그래프 차별화

---

## 1. 로컬 RAG가 필요한 이유

LangChain·LlamaIndex로 RAG 앱을 빌드하려면 통상 다음 부품을 직접 wire 한다:

```
Embeddings (OpenAI/HuggingFace)
   ↓
Vector store (Chroma/Qdrant/Pinecone)
   ↓
Retriever (BM25 + Vector + Hybrid)
   ↓
Reranker (Cohere/BGE-Reranker)
   ↓
LLM (OpenAI/Anthropic)
```

각 단계마다 **외부 API 호출 + 호스팅 비용 + 코드 작성**이 누적된다.
"본인 노트 폴더 하나 검색"하고 싶을 뿐인데 RAG 앱을 빌드하는 셈.

**QMD는 이 5 부품을 한 바이너리로 묶었다.** 외부 호출 0건.

---

## 2. QMD 단독 장점 7가지

### 2.1 외부 API 0건 → 비용 0원

| 항목 | 일반 RAG | QMD |
|---|---|---|
| 임베딩 호출 | OpenAI text-embedding-3 ~$0.13/1M tokens | 로컬 bge-m3 |
| 벡터 DB 호스팅 | Pinecone Starter $70/월~ | SQLite 단일 파일 |
| Reranker API | Cohere ~$2/1k queries | 로컬 (qmd 내장) |
| **월 비용** | **$80~200+** | **$0** |

### 2.2 다국어 임베딩 기본 내장

bge-m3 (1024차원, BAAI · Apache 2.0). 한·영·일·중 동시 처리. 같은 인덱스에 한글 노트와 영문 논문이 섞여 있어도 cross-lingual 검색 가능.

> 본 vault 실증: 한글 query `"헤르메스 에이전트 자가 개선"` → 영문 노트 `2026-04-23-rejection-sampling-hermes4.md` 상위 매칭.

### 2.3 1줄 시작 (`qmd query`)

```bash
# 일반 RAG: 코드 50~200줄 wire 필요
# QMD:
qmd collection add my-notes ~/notes
qmd update
qmd query "검색어" -c my-notes
```

`qmd query`는 내부적으로 BM25 + 벡터 + 자동 reranking을 하이브리드로 처리. 학습자가 retriever 클래스 조립 안 함.

### 2.4 데이터 로컬 보존

- 회사 노트 · 고객 정보 · 제품 스펙 외부 API로 안 나감.
- 보안 감사 통과 쉬움 (OpenAI 데이터 처리 약관 불필요).
- 항공망·오프라인 환경에서도 동작.

### 2.5 인덱싱 속도 + 증분 갱신

본 강사 vault 실측:
- **2215 .md 파일 + 239 chunks 임베딩 = 1분 47초** (M-tier WSL CPU)
- 증분 갱신: 신규/변경 파일만 재인덱싱 (`qmd update`).
- 모델은 `~/.cache/qmd/models/`에 1회 다운로드 (~600MB) 후 영구 사용.

### 2.6 의미·키워드·하이브리드 3가지 명령 분리

| 명령 | 방식 | 언제 |
|---|---|---|
| `qmd search` | BM25 (정확한 토큰 매치) | 단어가 노트에 그대로 박혀 있을 때 |
| `qmd vsearch` | 벡터만 (의미 유사도) | 단어가 달라도 의미로 찾을 때 |
| `qmd query` | 하이브리드 + reranker | **기본값** (위 두 가지 다 잡음) |

→ 사용자가 의도를 명시적으로 선택. 일반 RAG 프레임워크에서는 retriever 코드 다시 조립해야 함.

### 2.7 컬렉션 단위 폴더 관리

여러 폴더를 별칭(컬렉션)으로 등록 후 컬렉션별 검색.

```bash
qmd collection add work-docs ~/work
qmd collection add study     ~/study
qmd collection add scholar   /mnt/c/Users/daniel/Desktop/Zettelkasten/10_Raw/07_Scholar

qmd query "..." -c scholar     # 학술 노트만 검색
qmd query "..." -c work-docs   # 회사 자료만 검색
```

---

## 3. QMD vs 일반 리트리버 비교 표

| 항목 | 일반 리트리버 (LangChain · LlamaIndex) | QMD |
|---|---|---|
| **임베딩 모델** | OpenAI · HuggingFace · 로컬 직접 wire | bge-m3 로컬 **기본 내장** |
| **벡터 DB** | Chroma · Qdrant · Pinecone 별도 구성 | SQLite 단일 파일 |
| **하이브리드 검색** | retriever 클래스 직접 조립 + reranker wire | `qmd query` 1줄 |
| **외부 API · 비용** | 임베딩 호출비 + 벡터 DB 호스팅비 | **0건 · 0원** (전부 로컬) |
| **주 용도** | RAG 앱 빌드용 범용 인프라 | 본인 노트 · 로컬 vault 검색 |
| **학습 곡선** | Python + retriever 추상화 이해 | CLI 4 명령 (`collection add` · `update` · `query` · `search`) |

→ **RAG 앱을 만든다면 일반 리트리버 부품 조립.**
→ **본인 노트만 검색한다면 QMD 1개로 끝.**

---

## 4. 3-Layer 직교 협력 (QMD + GBrain + Graphify)

QMD 단독으로 부족한 영역은 GBrain·Graphify가 보강한다. 본 강사 vault에서 4월 28일 확정된 운영 정합:

| Backend | 역할 | 비용 | 활성 시점 |
|---|---|---|---|
| **QMD** | 의미·키워드 검색 (vector + FTS) | 로컬, **무료** | 90%의 자율 검색 — default |
| **GBrain** | 한글 검색 품질 + 백링크 그래프 | 네트워크, 적음 | 한글 query 약하거나 백링크·타임라인 필요 시 |
| **Graphify** | community · god nodes · semantic similarity | LLM, ~$0.1~0.3/일 | 주1회 정기 빌드 + 사용자 수동 query |

### 4.1 각 backend의 고유 가치

**QMD** | 988~2215 파일 직접 임베딩 · 로컬 SQLite · 0 latency · 0 cost.
약점: 한글 임베딩 품질이 GBrain (text-embedding-3-large) 보다 떨어짐. 그래프 분석 불가.

**GBrain** | text-embedding-3-large 한글 우월 · RRF 하이브리드 (tsvector + embedding) · Cathedral II schema (백링크 / 타임라인 / 엔티티).
약점: 네트워크 의존 (Supabase). 좀비 connection 위험.

**Graphify** | 옵시디언 graph view가 절대 못 하는 **community detection + god nodes ranking + cross-cluster semantic similarity**.
약점: LLM 비용. 첫 build $2~3, 주간 incremental $0.1~0.3. 자동화 불가능 (multi-step LLM orchestration).

### 4.2 운영 사이클

```
일상:
  사용자 질문
    ↓
  Claude Tier 2a (replay-learnings: 30_Claude)
    ↓
  Tier 2b — QMD 먼저 (default, 무료)
    ↓
  필요 시 GBrain (한글 품질 / 백링크)
    ↓
  필요 시 Graphify (community / god nodes / semantic_similar)
       → 결과는 이미 build된 graph.json/graph.html
```

---

## 5. 본 강사 vault 실측 케이스

### 5.1 QMD 측 실측

| 지표 | 값 |
|---|---|
| **인덱싱 파일** | 2,215 (.md) + 13 (rag-day3 자료) |
| **임베딩 chunks** | 239 신규 (본 세션) |
| **임베딩 모델** | embeddinggemma-300M-Q8 (1024차원) |
| **임베딩 시간** | 1분 47초 |
| **저장소** | 로컬 SQLite + bge-m3 모델 ~/.cache/qmd/models/ |
| **외부 API 호출** | **0건** |
| **비용** | **0원** |

### 5.2 Graphify 측 실측 (2026-05-14)

| 지표 | 값 |
|---|---|
| **Nodes** | 4,511 |
| **Edges** | 6,952 |
| **Communities** | 189 |
| **추출 신뢰도** | EXTRACTED 88% / INFERRED 12% / AMBIGUOUS 0% |
| **God nodes 1위** | Claude Code (83 edges) |
| **God nodes 2위** | Harness Engineering (80) |
| **God nodes 3위** | Hermes (74) |
| **QMD 노드 순위** | 10위 (43 edges) |
| **Graphify 자체 순위** | 7위 (46 edges) |

→ god nodes 상위 15위 안에 본 강사가 운영하는 핵심 도메인(Claude Code · Harness · Hermes · MCP · Yurisma · QMD · Graphify · Karpathy · Obsidian)이 정확히 정렬. graphify의 community 매핑이 운영 의도와 일치하는 검증.

### 5.3 surprising connections (옵시디언 graph view 불가)

graphify가 cross-cluster semantic similarity로 발견한 비자명 연결:

- `Context Engineering (Karpathy) ↔ Context Engineering (article)`
- `Google Stitch ↔ Stitch-Claude-Netlify Pipeline`
- `12-Factor Agents ↔ Anthropic Building Effective Agents`
- `study-note Downloader/OCR ↔ study-note UX Redesign`

옵시디언 wikilink만으로는 절대 안 보이는 연결을 LLM 추론이 노드 라벨·문맥 임베딩으로 찾아냄.

---

## 6. 비용 비교 (1년 운영 기준)

본 강사 vault 규모 (2,215 .md / 239 chunks / 주간 ~10% 갱신) 가정.

| 항목 | 외부 RAG (OpenAI + Pinecone + Cohere) | 로컬 QMD | 로컬 QMD + Graphify |
|---|---|---|---|
| 임베딩 (초기) | $2.50 (text-embedding-3 / 5M tokens) | $0 | $0 |
| 임베딩 (월 갱신) | $0.25/월 | $0 | $0 |
| 벡터 DB 호스팅 | $70/월 (Pinecone Starter) | $0 | $0 |
| Reranker (월 1k query) | $2/월 (Cohere) | $0 | $0 |
| Graphify LLM 빌드 | – | – | $0.5/월 (주1회 incremental) |
| **연 합산** | **~$870 + 초기 $2.5** | **0원** | **~$6** |

QMD + Graphify는 외부 RAG 대비 **약 99% 비용 절감** + 데이터 로컬 보존.

---

## 7. 한계 + 보완 (정직한 평가)

QMD가 모든 케이스에 답은 아님. 다음 경우는 부족하다.

| 부족한 영역 | 보완 |
|---|---|
| 한글 임베딩 품질 (영문 대비 약함) | GBrain (text-embedding-3-large) 추가 호출 |
| 백링크 · 타임라인 · 엔티티 그래프 | GBrain Cathedral II schema |
| Cluster 분석 / god nodes / 노드 중심성 | Graphify (주1회 LLM 빌드) |
| Cross-document synthesis (GraphRAG 류) | Graphify + 수동 합성 노트 |
| 5만 파일 이상 대규모 vault | Pinecone 등 외부 인프라로 이전 검토 |

> **핵심 인용 (본 vault `cognee.md`)**: "GraphRAG 엔터프라이즈 도구지만 개인 규모에서는 QMD + 위키링크로 충분하다."

---

## 8. 결론

**QMD = 본인 노트 검색용 통합 RAG 단일 도구.**
일반 RAG 프레임워크가 부품 모음이라면, QMD는 부품을 박스 밖 1줄로 묶은 결과물.
보강이 필요할 때만 GBrain(한글·백링크) · Graphify(그래프·community) 추가.
3 backend는 중복이 아닌 직교 layer — 같은 vault를 다른 lens로 본다.

본 강사 vault 실증:
- 2,215 파일 / 239 chunks / 1분 47초 임베딩 / **0원** (QMD)
- 4,511 nodes / 6,952 edges / 189 communities (Graphify, ~$0.5/월)
- 외부 RAG 대비 **연 ~$870 절감** + 데이터 로컬 보존

본인 노트가 100 ~ 5만 파일 사이라면 QMD를 1차로, 보강이 필요할 때만 다른 layer를 얹는 것이 최소 비용·최대 자율의 운영 안.

---

## 출처

- [[design-2026-04-28-knowledge-system-chain]] — 3 backend 직교 layer 설계 (선행 운영 정합 문서)
- [[design-2026-04-15-knowledge-system-usage-principle]] — Tier 0~3 탐색 순서
- [[session-2026-04-30-qmd-cuda-d3-complete]] — QMD CUDA 가속 구축
- [[session-2026-05-01-harness-skill-system-phase1-complete]] — bge-m3 임베딩 매칭 도입
- Graphify graph_stats (2026-05-14) — 4511 nodes / 6952 edges / 189 communities
- Graphify god_nodes (2026-05-14) — Claude Code · Harness · Hermes · QMD · Graphify 상위
- QMD obsidian collection 인덱싱 (2026-05-14) — 2215 files / 239 chunks / 1m 47s
- github.com/tobi/qmd — QMD 원본 (Tobias Lütke · Shopify CEO)
- github.com/doraemonkeys/qmd-windows — Windows installer
- bge-m3 (BAAI) — 다국어 1024차원 임베딩 (Apache 2.0)
