---
title: "LLM 에이전트 자가 개선(Self-Improvement) — 어떻게 가능한가, 연구 결과는"
source_type: deep_research
source_url: "https://arxiv.org/abs/2507.19457 | https://arxiv.org/abs/2309.16797 | https://arxiv.org/abs/2504.15228 | https://arxiv.org/abs/2508.16153 | https://arxiv.org/abs/2603.18000"
author: "feynman deep-research (Claude via Daniel8824)"
created: 2026-04-22
ingested: 2026-04-22
wiki_page: "[[llm-agent-self-improvement-deep-research]]"
confidence: 0.90
last_reinforced: 2026-04-22
tags:
  - raw/scholarly
  - self-evolution
  - 자가개선
  - LLM에이전트
  - GEPA
  - Promptbreeder
  - SICA
  - Memento
  - AgentFactory
  - feynman/deep-research
---
> **수집 이유**: Hermes Agent의 핵심 차별점인 self-evolution이 "실제로 어떻게 작동하는가"와 "학계의 합의된 연구 결과는 무엇인가"를 원전 단위에서 검증하기 위함. Daniel이 OMC 하네스에 이식할 가치가 있는 지 판단하는 근거 확보.

> **내 관점 (My Take)**: 2023년 Promptbreeder(DeepMind) → 2025년 GEPA(UC Berkeley·Stanford·MIT)·SICA(Bristol/iGent AI) → 2026년 Memento(Huawei·UCL)·AgentFactory(Peking·BAAI)로 이어지는 **"모델 재학습 없는 자가 개선"** 연구 계보가 3년 만에 ICLR Oral·SOTA 성과로 수렴했다. Hermes는 이 흐름의 production 구현체다. OMC 이식 최우선 부품: **(1) reflective mutation (GEPA)**, **(2) 실행 코드 편집 루프 (SICA)**, **(3) case-memory Q-learning (Memento)**.

> **한 줄 통찰**: LLM 자가 개선은 **파라미터 재학습 없이 "외부 아티팩트(프롬프트·스킬·코드·메모리)를 풍부한 텍스트 피드백 + 선택 압력 + 가드레일로 진화시키는" 비-gradient 학습 패러다임**이며, 2025~2026년 5개 대표 논문이 RL/SFT 대비 10~80배 샘플 효율, 35%+ 성능 향상을 일관되게 증명했다.

> **Gold Out**: 5대 논문(GEPA·Promptbreeder·SICA·Memento·AgentFactory) 원전 수치 확보 + **자가 개선 5원리** 메타 프레임(①Gradient-Free ②Rich Feedback ③External Artifact ④Selection Pressure ⑤Safety Guardrail) + OMC 이식 우선순위 7항목 표 + 학계 합의 5건 · 열린 질문 6건 · 3개월 최우선 액션 3개(autoresearch GEPA Pareto+trace / replay-learnings v2 Q-learning / overseer 신규 스킬).

> **Action Intent**: synthesize "self-improvement academic foundation" 1차 인용 source + OMC `autoresearch` 리팩토링 설계 근거 + `replay-learnings` v2 파라메트릭 확장 근거 + `overseer` 신규 스킬 설계 문서 seed + [[harness-wiki-system-justification]] 자매 축 4번째(하네스/메모리/복리/자가개선) + 세종사이버대/경복대 "AI 에이전트 자가 진화" 1주차 강의 소재.

## 핵심 요약

> LLM 에이전트의 self-improvement는 더 이상 "SF 개념"이 아니다. 5개 대표 논문(GEPA, Promptbreeder, SICA, Memento, AgentFactory)이 공통으로 입증한 것:
> 1. **모델 재학습 없이**(gradient-free) 성능이 RL 대비 10~19% ↑, 35× 적은 rollouts로 달성 가능 [GEPA]
> 2. **자기 코드 편집**(self-referential code edit)으로 SWE-Bench 17% → 53% (15 iterations) [SICA]
> 3. **Case-based 메모리 + Soft Q-Learning**으로 GAIA top-1, SimpleQA 95% SOTA [Memento]
> 4. **실행 가능한 subagent 축적**으로 orchestration 토큰 31% ↓ (재사용 효과) [AgentFactory]
> 5. **Self-referential mutation**(mutation 프롬프트도 진화)이 없으면 39~80% 성능 하락 [Promptbreeder ablation]

## 본문

### 1. 연구 지형도 (Taxonomy) — 5가지 접근

자가 개선 기법을 **"무엇을 진화시키는가"** 축으로 분류:

| 접근 | 진화 대상 | 대표 논문 | 연도 | 출판 |
|---|---|---|---|---|
| **Reflective Prompt Evolution** | 시스템 프롬프트 (LLM trace 기반 mutation) | **GEPA** | 2025-07 | ICLR 2026 Oral |
| **Self-Referential Prompt Evolution** | Task prompt + Mutation prompt 동시 | **Promptbreeder** | 2023-09 | DeepMind |
| **Direct Code Self-Edit** | 에이전트 자체 Python 코드 | **SICA** | 2025-04 | Bristol/iGent |
| **Case-Memory Continual Learning** | 에피소드 메모리 + case retrieval 정책 | **Memento** | 2025-08 | Huawei/UCL |
| **Executable Subagent Accumulation** | 재사용 가능 Python subagent 코드 | **AgentFactory** | 2026-03 | Peking/BAAI |

Hermes Agent의 `skill_manage`는 이 중 **SICA 계열(직접 코드 편집) + AgentFactory 계열(실행 가능 아티팩트 축적)** 의 하이브리드다.

---
### 2. GEPA — "Reflective Prompt Evolution Can Outperform Reinforcement Learning" (2507.19457)

**소속**: UC Berkeley, Stanford, BespokeLabs, Notre Dame, Databricks, MIT (17인)
**핵심 인물**: Matei Zaharia (Spark 창시자), Christopher Potts, Omar Khattab (DSPy 저자)

#### 어떻게 가능한가 — 3대 설계 원리

1. **Rich Language Feedback**: RL은 scalar reward만 받지만, LLM은 실행 중 자연어 trace(reasoning chain, tool call, error message)를 생성. GEPA는 이 trace를 **LLM-as-judge가 읽어 "왜 실패했는지" 진단**한 뒤 타겟 mutation 제안.
2. **Pareto-Based Selection (illumination)**: 단일 최고점 후보만 따라가면 local optimum에 빠짐. 각 train instance별 최고 성능을 낸 후보들의 **Pareto frontier**에서 stochastic sampling → 다양성 유지하며 generalization 강화.
3. **Genetic Crossover (Merge variant)**: 서로 다른 모듈에서 강점 있는 두 lineage의 최고 모듈을 조합 → 분산된 개선 사항 통합.

#### 연구 결과 — 충격적인 수치

| 지표 | GEPA vs GRPO (RL) | GEPA vs MIPROv2 (prompt opt) |
|---|---|---|
| 평균 성능 향상 | **+10%** (Qwen3 8B) · 최대 +19% | **+14.29%** aggregate gain |
| Rollout 효율 | **35× 적게** (IFBench: 678 vs 24,000) | — |
| 학습 rollout만 | **최대 78× 적게** (HoVer) | — |
| 프롬프트 길이 | — | **최대 9.2× 짧게** (더 저렴) |
| Generalization gap | — | 더 낮음 (test≈val) |

#### 인퍼런스-타임 코드 최적화 보너스

- **NPU kernel 최적화**: GPT-4o mean vector utilization **4.25% → 30.52%** (GEPA) — compiler error message를 feedback으로 주입
- **CUDA kernel (KernelBench)**: fast_p score **20% 초과** 달성

#### 이식 가치 ★★★★★

Reflective mutation은 GEPA의 핵심. OMC `autoresearch` 스킬을 "단순 반복"에서 "trace 기반 target mutation"으로 업그레이드할 직접 레시피.

---

### 3. Promptbreeder — "Self-Referential Self-Improvement Via Prompt Evolution" (2309.16797)

**소속**: Google DeepMind (2023-09-28)

#### 어떻게 가능한가 — 이중 진화 (Dual-Layer Evolution)

```
Layer 1: Task prompt 진화 (문제 해결 프롬프트)
Layer 2: Mutation prompt 진화 (어떻게 변이시킬지 지침)  ← self-referential
```

**5가지 mutation 연산자**:
1. Direct mutation
2. Estimation of Distribution mutation (성공 프롬프트 패턴 기반 생성)
3. **Hypermutation** — mutation prompt 자체를 mutation (메타 학습)
4. Lamarckian mutation — 성공한 working-out에서 task prompt 역산
5. Prompt crossover + context shuffling

#### 연구 결과 — Ablation이 증명한 self-reference의 가치

- 산술·상식 추론에서 Chain-of-Thought·Plan-and-Solve 같은 손-설계 프롬프트 상회
- **Ablation**: Self-referential task-prompt 컴포넌트 제거 시 대부분 태스크에서 **39~80% 성능 하락**
- "자기 언급 없이는 자가 개선이 성립하지 않음" — GEPA·SICA·AgentFactory 설계 철학의 이론적 기반

#### 한계

- 평가 비용 높음 (LLM 호출 다수)
- 다른 LLM 간 프롬프트 전이성 보장 안 됨
- 명확한 fitness metric 정의 어려운 태스크에 부적합

---

### 4. SICA — "A Self-Improving Coding Agent" (2504.15228)

**소속**: University of Bristol + iGent AI (2025-04-21)

#### 어떻게 가능한가 — Meta Agent Loop

```
archive (과거 버전 + 성능) → 최고 버전 선택 (utility 함수)
        ↓
meta-agent = target-agent (같은 에이전트!)
        ↓
자기 Python 코드 직접 편집 → 새 버전 생성
        ↓
benchmark 평가 → archive에 추가
```

**핵심 차별점 vs ADAS**: Meta-agent와 target-agent 구분 제거. 에이전트가 **자기 자신의 Python 코드 전체**를 수정 (Gödel Agent/STOP은 DSL 안에서만 수정).

**Utility 함수**:
```
U = 0.5·p_score + 0.25·p_cost + 0.25·p_time
   (성능 50% + 비용 25% + 시간 25%)
timeout 페널티 τ=0.5
```

**안전 장치 — Asynchronous LLM-based Overseer**:
- 별도 LLM이 동시 실행, call graph·event stream 모니터링
- Looping·이탈·pathological 행동 감지 → 알림 또는 강제 종료
- "자가 수정 에이전트"의 실전 안전 패턴

#### 연구 결과 — 15 iterations로 얻은 극적 개선

| 벤치마크 | 초기 | 최종 | 비고 |
|---|---|---|---|
| **SWE-Bench Verified** | 17% | **53%** | 코드 네비게이션·파일 편집 SOTA |
| **LiveCodeBench** | 65% | 71% | 알고리즘 난제 |
| **File Editing (자체)** | 82% | 91~96% | Smart Editor·diff tool 자체 구현 |
| **Codebase Navigation** | 35% | 40~43% | AST Symbol Locator 자체 구현 |
| **추론 (AIME/GPQA)** | 76% | 77% | **Agent Framework Saturation** — o3-mini 단독이 더 나음 |

**총 비용**: 15 iterations에 $7,000 (API 호출). 단 반복 진행 시 token cache hit이 증가해 효율 점증.

#### 에이전트가 스스로 만든 도구들 (학습 궤적)

- Iter 1~3: Smart Editor (diff 기반), Quick Overwrite Tool
- Iter 5~8: Code summarization, edit verification
- Iter 9~13: **AST Symbol Locator** (ripgrep + AST 하이브리드), diff minimization

→ "에이전트가 자기 개선에 필요한 도구를 자기 스스로 개발" 자체가 결과.

#### 한계 — Agent Framework Saturation

이미 강한 LLM(o3-mini for math)은 scaffolding이 오히려 방해될 수 있음. 즉 자가 개선의 성과는 **"LLM 단독은 약하지만 scaffolding으로 커버 가능한 도메인"에서 가장 큼** (long-horizon agentic tasks).

#### 이식 가치 ★★★★

Daniel 하네스에 Overseer 패턴(별도 LLM 모니터링) 이식 가치 높음. 코드 직접 편집까지는 현재 상태로는 위험.

---

### 5. Memento — "Fine-tuning LLM Agents without Fine-tuning LLMs" (2508.16153)

**소속**: Huawei Noah's Ark Lab + UCL AI Centre + Jilin Univ + CAS (2025-08-22)
**핵심 인물**: Jun Wang (UCL, 강화학습 권위)

#### 어떻게 가능한가 — M-MDP + Soft Q-Learning

**Memory-augmented MDP**: `⟨S, A, P, R, γ, M⟩` — 표준 MDP에 **메모리 공간 M** 추가.

Loop:
```
Retrieve (case c_t from M_t using μ(·|s_t, M_t))
    ↓
Reuse & Revise (LLM adapts c_t to s_t → action a_t)  ← LLM 고정!
    ↓
Evaluate (reward r_t)
    ↓
Retain ((s_t, a_t, r_t) → M_{t+1})
```

**핵심**: LLM은 frozen. **case-retrieval policy μ만 학습** — Soft Q-Learning으로 retrieval 정책의 Q-function을 업데이트.

**두 가지 메모리 구현**:
- **Non-parametric**: Append + kNN cosine similarity (SimCSE embedding)
- **Parametric**: 2-layer MLP가 Q(s, c; θ) 학습 (binary cross-entropy, success/failure)

#### 연구 결과 — 4개 벤치마크 SOTA

| 벤치마크 | Memento | 비교 |
|---|---|---|
| **GAIA** | **Top-1 validation 87.88% / test 79.40%** | Manus·Aworld·OWL 초월 |
| **DeepResearcher** | **66.6 F1 / 80.4 PM** | SOTA training-based보다 **+14.8 F1** |
| **SimpleQA** | **95.0%** | 단일 홉 factual SOTA |
| **HLE** | 24.4% PM (2위) | GPT-5 25.32%에 근접, Gemini-2.5-Pro 초월 |

**Continual learning 검증**: Iteration 증가에 따라 non-parametric·parametric CBR 모두 성능 향상. Parametric이 약간 더 빠른 수렴.

**OOD 일반화**: MusiQue, Bamboogle, PopQA에서 **4.7~9.6% 절대 개선** — case bank 축적이 실제로 일반화 돕는다는 증거.

**재미있는 발견**: "Fast" planner (GPT-4.1) > "Slow/deliberative" planner (o3, Qwen3) — 빠른 분해·간결한 지시가 오히려 downstream 실행 효율 높임.

#### 이식 가치 ★★★★★

Daniel의 `replay-learnings`는 이미 Memento의 non-parametric case memory와 같은 구조. **Parametric Q-learning 추가**가 다음 진화 방향.

---

### 6. AgentFactory — "Self-Evolving Framework Through Executable Subagent Accumulation" (2603.18000)

**소속**: Peking University + BAAI + Hong Kong Polytechnic (2026-03-18)

#### 어떻게 가능한가 — 3-Phase Lifecycle

```
Install:    새 문제 → Meta-Agent가 Python subagent 코드 생성 → SKILL.md 저장
    ↓
Self-Evolve: 유사 문제 → 저장된 subagent retrieve → 실패 시 modify_subagent로 코드 수정
    ↓
Deploy:     성숙한 subagent를 pure Python으로 export → Claude Code·LangChain 등에서 재사용
```

**핵심 설계**:
- **Meta Skills** (built-in): `create_subagent`, `modify_subagent`, `run_subagent`, `list_saved_subagents`
- **Tool Skills** (built-in): `web_search`, `browser_automation`, `shell_command` 등 atomic tools
- **Subagent Skills** (dynamic): 에이전트가 생성·수정하는 Python 스크립트

**텍스트 경험 vs 실행 경험 — 결정적 차이**:
| 구분 | Reflexion 계열 | AgentFactory |
|---|---|---|
| 저장 단위 | 자연어 reflection | **실행 가능한 Python 코드** |
| 재실행 신뢰성 | 낮음 (LLM이 다시 해석) | 높음 (직접 실행) |
| 포팅성 | 프레임워크 종속 | Python이 돌면 어디든 (SKILL.md 포함) |

#### 연구 결과 — Orchestration 토큰 비교 (Claude Opus 4.6)

| 방법 | Batch 1 (from scratch) | Batch 2 (w/ saved subagents) |
|---|---|---|
| ReAct (baseline) | 8,298 | 7,022 |
| Self-Evolving (textual) | 8,608 | 6,210 |
| **AgentFactory** | **4,324** | **2,971** |

**Batch 2에서 ReAct 대비 58% 토큰 절감** — 실행 가능 아티팩트 재사용이 텍스트 요약 재사용보다 훨씬 효율적임을 정량 증명.

**Cross-system 검증**: AgentFactory에서 만든 subagent를 **Claude Code**로 그대로 import해서 성공 실행 → SKILL.md 표준의 실효성.

#### 이식 가치 ★★★★

Hermes `skill_manage` + OMC `/skillify`가 이미 비슷한 철학. Cross-system export 표준화가 배울 점.

---

### 7. 공통 메커니즘 — "자가 개선은 어떻게 가능한가" 5개 원리

5개 논문을 교차 분석하면 같은 DNA가 보임:

| 원리 | 설명 | 대표 구현 |
|---|---|---|
| **① Gradient-Free Learning** | 모델 파라미터 안 건드림. 텍스트·코드·메모리 레벨에서 학습 | 전부 공통 |
| **② Rich Feedback Signal** | scalar reward 아니라 trace·error log·execution output 활용 | GEPA, SICA |
| **③ External Artifact** | 학습 결과를 외부화(prompt, skill, code, memory) — LLM 외부 저장 | 전부 공통 |
| **④ Selection Pressure** | 단순 변이 아닌 선택 (Pareto, Borda, utility, Q-value) | GEPA(Pareto), SICA(utility) |
| **⑤ Safety Guardrail** | 자기 수정의 위험 → PR review, overseer, policy matrix | SICA(overseer), Hermes(INSTALL_POLICY) |

**5개가 모두 있어야 self-improvement가 성립**. 하나라도 빠지면:
- ① 없음 → 비용 폭증, RL로 환원
- ② 없음 → sample inefficient (RL과 동일 문제)
- ③ 없음 → 학습이 세션과 함께 증발
- ④ 없음 → local optimum·drift
- ⑤ 없음 → 악성/무의미 변이 축적, 안전 위험

---

### 8. 학계 합의 vs 열린 질문

#### Consensus (합의된 것)

- ✅ **모델 재학습 없이도** 의미 있는 성능 향상 가능 (GEPA +14%, SICA +36%, Memento +14.8 F1)
- ✅ **Rich text feedback > scalar reward** (GEPA가 RL 대비 35~78× 효율)
- ✅ **Self-referential (자기 언급) 구조 필수** (Promptbreeder ablation 39~80% 하락)
- ✅ **Selection pressure + guardrail 없이는 drift 발생** (Promptbreeder·GEPA·SICA 공통)
- ✅ **실행 가능 아티팩트(코드) > 자연어 요약** (AgentFactory 58% 토큰 절감)

#### Open Questions (열린 질문)

- ❓ **Fitness function Goodhart 위험** — LLM-as-judge의 rubric 품질을 누가 검증하나?
- ❓ **장기 drift** — 수백 iteration 후에도 semantic preservation 유지되나? (현재 논문들은 10~15 iter 수준)
- ❓ **Cross-model transfer** — A 모델에서 진화한 프롬프트가 B 모델에서도 작동하나? (Promptbreeder는 미해결)
- ❓ **Agent Framework Saturation 경계선** — 어떤 태스크에서 scaffolding이 오히려 방해?
- ❓ **LLM weight 변경과의 결합** — gradient-free + gradient 하이브리드가 가능한가? (AlphaEvolve가 힌트)
- ❓ **안전성 극단 시나리오** — self-improving 에이전트가 weight 수정 권한까지 갖는 시점의 안전 보장

---

### 9. Daniel OMC 하네스로의 이식 로드맵

현재 OMC 상태와 비교한 우선순위:

| 우선순위 | 이식 대상 | 소스 | 이식 방법 | OMC 대응 스킬 |
|---|---|---|---|---|
| ★★★★★ | **Reflective Mutation** (trace 기반) | GEPA | autoresearch를 trace 읽기 + target mutation으로 업그레이드 | `autoresearch` |
| ★★★★★ | **Case-memory Q-learning** | Memento | replay-learnings에 parametric Q-function 추가 | `replay-learnings` |
| ★★★★ | **Async Overseer** | SICA | 별도 LLM이 세션 모니터링 (pathological 감지) | 신규 skill 필요 |
| ★★★★ | **Pareto 후보 풀** | GEPA | autoresearch가 단일 best 대신 Pareto frontier 유지 | `autoresearch` |
| ★★★★ | **Size/Semantic guardrail** | Hermes 외부 self-evolution | wiki critique gate에 size≤15KB, semantic preservation 추가 | `wiki-critique` |
| ★★★ | **Subagent cross-export** | AgentFactory | SKILL.md 표준을 Claude Code·Hermes 공용으로 확장 | `skillify` |
| ★★★ | **Hypermutation** | Promptbreeder | mutation 프롬프트 자체를 진화 대상으로 | 신규 실험 |

**최우선 3개월 액션**:
1. `autoresearch` 리팩토링 — GEPA Pareto + trace 기반 mutation 도입
2. `replay-learnings` v2 — 파라메트릭 case retrieval Q-function
3. `overseer` 신규 스킬 — 백그라운드 LLM 모니터링

---

## Related

- [[hermes-agent-self-evolution]] — NousResearch 외부 self-evolution 엔진 (DSPy+GEPA 구현)
- [[hermes-skill-self-evolution-architecture]] — tmdgusya의 내부 skill_manage 해부
- [[skill-self-evolution-pattern]] — 외부화된 절차적 기억 진화 패턴
- [[evolutionary-prompt-optimization]] — DSPy+GEPA 패턴 추상화
- [[gepa]] — Genetic-Pareto Prompt Evolution 엔티티
- [[dspy]] — Stanford DSPy 프레임워크 엔티티
- [[hermes-ecosystem-repos]] — Hermes 생태계 레포 카탈로그
- [[hermes-agent-memory-architecture]] — 4계층 메모리 아키텍처
- [[2026-04-20-hermes-agent-self-evolution]] — 원본 README raw
- [[2026-04-11-tmdgusya-hermes-self-evolution]] — 원본 블로그 raw
- [[harness-wiki-system-justification]] — 하네스·볼트 정당성 (동일 thesis: 배포 후 경험 진화)

## 후속 액션

- [ ] `autoresearch` 스킬에 GEPA Pareto + reflective mutation 프로토타입
- [ ] `replay-learnings` 파라메트릭 버전 실험 (Memento Q-learning 복제)
- [ ] `overseer` 신규 스킬 설계 — SICA-style async LLM 모니터
- [ ] Promptbreeder hypermutation을 `/skillify`에 실험적 도입
- [ ] AgentFactory SKILL.md cross-system export 표준 검토 (Claude Code Agent Skills와 정합성)
- [ ] GEPA 원본 레포 (gepa-ai/gepa) 탐색 및 MIT 라이선스 확인
- [ ] 후속 논문 모니터링: AlphaEvolve, Darwin Gödel Machine, Memento 2 (2025-12-27)

## 관련 문서

- [[20_Wiki/01_Sources/llm-agent-self-improvement-deep-research|llm-agent-self-improvement-deep-research]]
- [[20_Wiki/03_Concepts/self-improvement-saturation|self-improvement-saturation]]
- [[20_Wiki/03_Concepts/evolutionary-prompt-optimization|evolutionary-prompt-optimization]]
- [[10_Raw/01_Articles/2026-04-20-hermes-agent-self-evolution|2026-04-20-hermes-agent-self-evolution]]
- [[20_Wiki/01_Sources/hermes-agent-self-evolution|hermes-agent-self-evolution]]
