# Spelunker — Design Document

**An offline deep-research agent that works under a hard token budget, and a 2B student distilled from it, on a laptop with the network off**

| | |
|---|---|
| Status | Proposal |
| Target duration | ~10–12 weeks (19 mentoring hours / 15 sessions + independent work) |
| Platforms | **Windows 11 laptop** (16 GB RAM, RTX 3060-class, **6 GB VRAM assumed**, as the Laptop GPU ships; 8 GB only if confirmed): all scored runs and training. **Apple Silicon Mac** (16 GB unified, no CUDA): development, tests, demo. One repository. |
| Deployment | 100% local: no rented servers, paid GPU cloud or inference API in the agent loop. A commercial API serves only as the judge's reference (est. < $15 worst case); free Colab/Kaggle GPUs are optional. |
| Core stack | Qwen3.5-9B / 4B (GGUF, llama.cpp `llama-server`) · Qwen3-Embedding-0.6B · Qwen3-Reranker-0.6B · Enclave (pinned) · Pyserini BM25 · Postgres + pgvector · Unsloth · MLflow |

**Terms.** **ACM**: Agentic Context Management (Li et al., July 2026), the closest prior work. **CM@16k**: Spelunker's context manager (CM) at a 16k-token budget. **ReAct**: the reason-and-act loop that appends each tool result to the prompt. **GGUF**: llama.cpp's model file format (quantized or bf16). **RFT**: rejection-sampling fine-tuning. **κ**: Cohen's kappa. **MDE**: minimum detectable effect. **RRF**: reciprocal rank fusion. **HNSW**: pgvector's approximate graph index. **FTS**: Postgres full-text search.

---

## 1. The Product Thesis

Deep-research agents have two problems. The live web changes, so evaluations cannot be re-run; and a ReAct prompt grows every step until it overflows a 6–8 GB GPU. **BrowseComp-Plus** fixes the first: 830 human-verified BrowseComp questions frozen over 100,195 documents with relevance judgments, so retriever and policy can be studied apart. **ACM** attacks the second: on its split, Qwen3.5-9B ReAct peaks at 63k tokens and scores 57.0%, below ACM's managed agent (63.5%; 72.7% post-trained; §5.1).

**Spelunker is a deep-research agent with a hard cap on working context, run end to end on hardware the mentee owns.** On a consumer card peak tokens bind first, so the budget is swept and measured.

> **Engineering thesis.** With a cited state ledger, offloaded evidence and a retriever chosen on evidence, a laptop agent inside a 16k-token budget stays within a pre-registered margin of the same agent with a 64k ceiling. Its trajectories, a few thousand tokens per step, are cheap training data for a 2B student.

**Baseline (measured, not a claim): ReAct@64k**: the profile's agent (Qwen3.5-4B on the default 6 GB card, 9B on 8 GB), 64k ceiling, dev and test, BM25 and dense. All margins use this local number. On 8 GB the 9B is *consistent with ACM* if 57.0% lies in its dev 95% CI; otherwise the gap is reported, with split, retriever and 64k-vs-128K cap as candidate causes.

**Floor gate (S1, pre-registered):** the 4B anchor needs ≥ 15% dev accuracy, median ≥ 8 tool calls and a peak prompt above 16k on ≥ 50% of trajectories. Otherwise the budget claims have nothing to test (why: §5.1; fallback: §8).

**Claims** (each with a closed-book, no-tools control and the judge's κ; decision rules in §9.3):
1. **Headline:** CM@16k vs ReAct@64k on test, best retriever frozen, beside peak prompt, total prefill and wall-clock.
2. **Graceful degradation:** CM@16k non-inferior to CM@64k within 8 points on test, peak prompt capped by construction (§7.1), less total prefill. 8k and 32k are dev-only.
3. **Retriever choice is a first-order agent parameter:** BM25 → Qwen3-Embedding-0.6B → hybrid / +Qwen3-Reranker-0.6B inside ReAct@64k, on accuracy and search calls; a 4B–9B test of the gain BrowseComp-Plus reports for gpt-5 (§5.1).
4. **A 2B student recovers a measured fraction of the teacher** via per-step RFT on successful compressed trajectories (§7.6); reported with its queries/hour gain.

**Why it is surprising, and fun.** More context should mean better research, yet ACM's 9B managed agent wins at a lower peak. Spelunker tests that at a quarter of ReAct's peak, on a laptop, and asks whether a 4B can do it at all.

## 2. Goals & Non-Goals

### Goals
- Claims 1–4 (§1), built as a precise agent loop and budget-bounded context manager (§7.1–7.2) in a plain Python state machine that owns every prompt byte.
- A retriever ablation scored end to end, not only by nDCG; the 2B student served as GGUF on the same runtime.
- A calibrated local judge beside exact match; one published number reproduced in S0.

### Non-Goals
- Building a retrieval engine: Enclave (07 in the earlier recsys/search series) is pinned, code unchanged, or replaced by a thin adapter (§7.4).
- Live-web tools at run time; leaderboard chasing against frontier models.
- Multi-agent orchestration; full reinforcement learning (GRPO is a stretch, §7.6).
- Redistributing the corpus, decrypted questions or plaintext trajectories.

### Relation to Sibling Proposals
- **02 Shellwright** owns per-turn distillation into Qwen3.5-2B; Spelunker reuses its trainer, masking and view parity, adding compression-step targets (ledger JSON) and the student's budget curve. A mentee doing both treats Claim 4 as a stretch.
- **05 BlackBox** owns cache-hit and seed-variance methodology and record/replay; Spelunker uses its metrics and cassettes.
- **01 Compass** sidesteps the hybrid-cache hazard (llama.cpp #22384) with standard-attention Qwen3; Spelunker keeps Qwen3.5 and measures prefix reuse in S0 (§7.2).
- **07 HarnessForge** could search the context-policy knobs (trigger, ledger cap) set here by hand.

## 3. Alignment With Prior Experience

| Prior experience | Spelunker module |
|---|---|
| **CAS**: patented cascaded binary-encoding image retrieval; Precision@K / latency / storage over 20+ configs | Retriever ablation and S0 reproduction, scored on answer accuracy |
| **Enclave** (row only if it shipped): local hybrid retrieval, egress test | The `search` tool; the egress test becomes the Wi-Fi-off demo |
| **College Application Agent**: EBR (embedding-based retrieval) + FastAPI; LangGraph Writer–Critic | `verify`: clue-by-clue cited verdicts |
| **Changsheng**: AutoML hyper-parameter agent; backtest gates | Budget sweep; gated student promotion |
| **Sina Weibo**: ETL, metrics, monitoring | Per-query telemetry in MLflow |
| **DiDi**: A/B testing | Paired designs, MDEs, bootstrap CIs |

**Interview one-liner:** *"I put a research agent on a local retrieval stack that answers X% of BrowseComp-Plus test questions with the network off, inside a 16k-token budget on a 6 GB laptop GPU, measured what each thousand tokens and each retriever buys, and distilled it into a 2B model."* Lead with "I built a retrieval engine that runs with the network off" only if Enclave shipped.

## 4. Stack Choices — and Why

### 4.1 Models

| Role | Model | Format | Disk | Why |
|---|---|---|---|---|
| Agent, 8 GB anchor | Qwen3.5-9B | GGUF Q4_K_M, text only | 5.68 GB | ACM's model; Apache-2.0 |
| Agent, 6 GB default | Qwen3.5-4B | GGUF Q4_K_M | 2.74 GB | 9B's layout; fits 6 GB with the reranker |
| Student | Qwen3.5-2B | bf16 LoRA → GGUF Q4_K_M / Q8_0 | 1.28 / 2.01 GB | Teacher's tokenizer, template, layout; serves on the Mac |
| Student fallback | Qwen3.5-0.8B | bf16 LoRA | measured | 3 GB LoRA vs 5 GB for the 2B (Unsloth) |
| Query encoder | Qwen3-Embedding-0.6B (1024-d) | fp16 | ~1.2 GB | Matches a prebuilt index |
| Reranker | Qwen3-Reranker-0.6B | fp16 | ~1.2 GB | Enclave's stage 2 |
| 8B-index encoder | Qwen3-Embedding-8B-GGUF (4096-d) | Q4_K_M, CPU | 4.68 GB | fp16 (15.1 GB) does not fit; S0 parity-gated |
| Cache fallback, 6 / 8 GB | Qwen3-4B / Qwen3-8B (standard attention) | GGUF Q4_K_M | 2.5 / 5.03 GB | Only if hybrid caching fails (§7.2) |
| Local judge | Qwen3.5-9B | Q4_K_M, partial offload on 6 GB | 5.68 GB | Batch, offline |

**KV arithmetic.** The 9B and 4B have 8 gated-attention layers of 32 (4 KV heads, head dim 256); the other 24 are Gated DeltaNet with fixed-size state. At q8_0: ~17 KB of KV per token, ~0.29 GB at 16k, ~1.14 GB at 64k. The 2B (6 attention layers of 24, 2 KV heads): ~6.5 KB. Standard-attention Qwen3-4B / Qwen3-8B (36 layers, 8 KV heads, head dim 128): ~78 KB, i.e. 1.28 GB at 16k, 5.1 GB at 64k. The hybrid layout makes a 9B plausible on 8 GB, and causes the cache risk (§7.2).

**Thinking** is on by default for the 9B and 4B; the 2B needs it enabled, and its card warns it is more prone to thinking loops than larger Qwen3.5 models (loop rate reported). Sampling: the card's thinking mode (T = 1.0, top-p 0.95, top-k 20, presence penalty 1.5), per-call generation cap. Thinking vs non-thinking is measured once on 30 dev queries; the untrained 2B runs both ways.

### 4.2 Runtimes and Libraries

| Concern | Choice | Why |
|---|---|---|
| Serving | llama.cpp `llama-server`, one pinned build (CUDA / Metal) | One path on both machines; prompt caching, q8_0 KV (`-ctk q8_0 -ctv q8_0`, flash attention), per-request timings |
| Prompting | Raw `/completion` + `cache_prompt`; Spelunker renders bytes | Qwen3.5's template keeps `<think>` only after the last real user turn, so an injected user turn (ledger instruction, loop warning) or whitespace change forces a re-prefill; `preserve_thinking` exists only in the Qwen3.6 template |
| Agent framework | Plain Python state machine, no LangGraph | A framework that owns serialization hides the bytes the cache depends on |
| Training | Unsloth bf16 LoRA | Unsloth advises against QLoRA (quantized LoRA) for Qwen3.5; needs `transformers` v5 |
| BM25 | Pyserini, prebuilt index | Paper parity; Java 21, documented for Mac/Linux only, so a WSL2/Docker sidecar on Windows |
| Storage / tracking | Postgres 16 + pgvector (Enclave's compose); MLflow + results table in git | Memory beside Enclave; regressions show in a diff |

## 5. Benchmarks & Datasets

### 5.1 BrowseComp-Plus

| Property | Value |
|---|---|
| Queries | 830 human-verified of BrowseComp's 1,266 |
| Corpus | 100,195 docs; mean 5,179 words (~32k characters) |
| Labels | Means of 6.1 evidence and 2.9 gold docs per query; TREC relevance files (qrels) |
| Access | Queries/answers obfuscated (`Tevatron/browsecomp-plus`), decrypted locally by the repo script; corpus plain |
| Prebuilt indexes | `Tevatron/browsecomp-plus-indexes`: BM25 and Qwen3-Embedding-0.6B / 4B / 8B; 5.25 GB; dense vectors cover each doc's first 4,096 tokens; no embedding job. Fits both machines; agent time is the cost (§8) |
| Protocol | `search` top-5, 512 tokens each; gpt-4.1 judge (the repo script defaults to Qwen3-32B) |

**Splits** (fixed seed, stratified by evidence count): dev 100 for iteration; test 230 for claims, once per frozen config; train 500 for teacher trajectories only. ACM's released rollouts are unused: they may overlap this test split.

**Published baselines** (all 830 queries, gpt-4.1 judge; Chen et al. Tables 1 and 4):

| Agent | BM25 | Qwen3-Embed-8B | Search calls (BM25 / 8B) |
|---|---|---|---|
| gpt-5 | 55.90% | 70.12% | 23.23 / 21.74 |
| gpt-oss-20B (medium) | 16.39% | 29.88% | 13.72 / 13.64 |
| Qwen3-32B | 3.49% | 10.36% (0.6B: 4.10%, 4B: 7.83%) | ~0.9 |

**Retrieval only** (full question as the query, evidence qrels; Table 2), nDCG@10 / Recall@100: BM25 1.6 / 4.7; 0.6B 8.0 / 26.5; 4B 14.0 / 40.2; 8B 20.3 / 47.7.

**ACM** Pass@1 (150-question split; retriever and judge model unstated; ReAct capped at 128K): Qwen3.5-9B ReAct 57.0% (19.5 calls, 63k peak); ReSum (periodic history summarisation; Alibaba Tongyi, 2025) 60.8%; ACON (learned compression guidelines; Kang et al., 2025) 61.4%; ACM base 63.5% (30.8 calls); ACM post-trained 72.7% (46.2 calls, 54k peak); Qwen3.5-397B-A17B 65.3%.

**Qwen3.5-4B and 2B: no published result.** In ACM's Appendix D, previous-generation Qwen3-4B-thinking collapses every rollout into two turns (2.0 turns, 1.2 searches, 3.4%; 9B: 19.4, 16.2, 57.3%), leaving context management nothing to do. Qwen3-32B above makes under one search per question; hence the floor gate (§1).

**S0 external reproduction (pre-registered).** Encode the 830 decrypted questions with Qwen3-Embedding-0.6B via Enclave's query path, search the prebuilt index, score on evidence qrels: a deterministic, minutes-long check that decryption, query instruction and encoder match the paper.
- **Targets:** 0.6B nDCG@10 8.0 ± 0.5, Recall@100 26.5 ± 1.0; BM25 nDCG@10 1.6 ± 0.3. In S2 the same test gates the 8B row (Q4 encoder within 1.0 of 20.3, else "not run").
- **HNSW depth:** `hnsw.ef_search` (default 40) caps returned rows, so Recall@100 needs `SET hnsw.ef_search = 200`; HNSW-vs-exact recall is reported (the paper used a flat index). If pinned Enclave cannot set it per query, Recall@100 uses the flat reference path and only nDCG@10 is gated through Enclave.

### 5.2 Training data
Once CM@16k freezes, the profile's agent (4B on `portable`, 9B on `accelerated`) samples the train split once per question; a second sample runs only if the first yields < 150 kept trajectories and budget allows. *Kept*: judge and exact match both correct, and every final citation is a document the trajectory opened. Yield (≤ teacher accuracy × trajectories sampled) is measured; kept trajectories become per-step examples (§7.6).

### 5.3 Demo data
Ten hand-picked dev questions most people could not answer in an afternoon, recorded network-off and replayed at speed from 05 BlackBox-style cassettes (a live run takes minutes); each searches 100K web documents and ends in field notes with a document behind every fact. One short query runs live on the Mac (2B student), Wi-Fi off. An unscored run on Enclave's (or the adapter's) docs shows it is not tied to one benchmark.

## 6. Architecture

```
 [ NO OUTBOUND NETWORK AT RUN TIME -- enforced by the demo and a smoke test ]

 question -> spelunker-agent (plain Python state machine)
             |- renderer  [SYS+tools][QUESTION+C1..Cn][LEDGER vK] [TAIL ...]
             |            `-------- stable prefix ---------------' `- append-only -'
             |- ctx-manager  token accounting; threshold; compress -> validate -> vK+1
             |- llama-server :8080  Qwen3.5-9B/4B GGUF; q8_0 KV; /completion
             |                      cache_prompt; timings -> cache-hit metric
             `- tools
                 search -> Enclave /v1/search (pinned) -> Postgres FTS + pgvector + reranker
                        `-> reference: Pyserini BM25 sidecar; flat index (4B/8B vectors)
                 open, find -> corpus store (doc_id -> tokenized pages)
                 note, recall -> Postgres `spelunker` schema (offloaded evidence + vectors)
                 verify -> <=3 searches; evidence appended; verdict = next turn
                 answer -> terminate

 offline: judge-svc  batch Qwen3.5-9B grading; κ vs gpt-4.1 + human; exact match
          distill/   rejection filter -> step segmenter -> Unsloth LoRA -> GGUF
 ops:     MLflow per-query telemetry; results table in git; egress-blocked smoke test
```

At step 0 the model rewrites the question as constraints C1..Cn (BrowseComp questions are conjunctions of clues): `verify`'s checklist. The first three blocks stay fixed between compressions, so their KV is reused and only the tail grows. Each tool result is appended to the tail, but if it would push the prompt past the threshold T, the manager compresses first (§7.1–7.2). Every call logs prompt length, prefilled and generated tokens, and wall-clock. Grading is a later batch job, so the judge never competes for VRAM.

## 7. Component Design

### 7.1 The Agent Loop

**Observation**, four blocks:
- `SYS`: rules, 7 tool schemas; fixed.
- `QUESTION`: text + C1..Cn; fixed after step 0.
- `LEDGER vK`: `facts[]` (`doc_id:page` cite, constraint IDs), `leads[]`, `ruled_out[]` (cite), `candidates[]` (per-constraint status), budget used; changes only at compression.
- `TAIL`: verbatim turns since compression; append-only.

**Action**: one call per turn. Thinking is unconstrained (it still counts toward G); the call after `</think>` must match its JSON schema via a lazy GBNF grammar (`llama-server`'s `grammar_lazy` / `grammar_triggers`, checked on the pinned build). Without it on `/completion`, a second append-only, grammar-constrained call emits the tool call, keeping the prefix.

| Tool | Returns | Cost |
|---|---|---|
| `search(query, mode=bm25\|dense\|hybrid\|rerank)` | k hits: `doc_id`, title, first s tokens (64-token stub if seen) | 1 call |
| `open(doc_id, page=None)` | 1,024-token page (768 at 8k, 2,048 at ≥ 32k), "page i of n"; default: BM25-best page for the last query | 1 call |
| `find(doc_id, pattern)` | ≤ 5 matching sentences with page numbers | 1 call |
| `note(fact, cite, constraints)` | Ack, staged for the next ledger; over 3 notes per 5 turns fires the loop guard | tokens only |
| `recall(query, k=3)` | Offloaded evidence spans (≤ 256 tokens at 8k, ≤ 512 above), `doc_id:page` | 1 call |
| `verify(candidate, constraint_ids)` | Evidence for ≤ 3 constraints (§7.3) | 1 per internal search |
| `answer(text, cites, confidence)` | Terminates | — |

At ≥ 16k, search follows the paper's protocol (§5.1), so agent-issued search counts are *approximately* comparable with its Table 1 (`search`-only agents); verify-internal searches are reported apart. Paging exists because a ~5k-word document does not fit 8k (S0 measures the share over 8k tokens). The prompt requires `verify` before `answer`; unverified answers are logged, not blocked.

**Termination**, checked before every call (cap and ceiling hit rates reported per arm and budget):
1. `answer`.
2. **Tool-call cap** of 40, then a forced answer from the ledger (ReAct: the tail). ACM's managed agents averaged 30.8 and 46.2 calls, so 40 vs 60 is checked on dev before freeze.
3. **Generated-token cap** per query, set at the week-2 gate, frozen with the config (deterministic, unlike wall-clock).
4. **Loop guard:** an identical call three times, or excess notes, triggers a warning; two more steps without progress force an answer.
5. ReAct only: the **64k ceiling** (fires when the next call would exceed 64k), then a forced answer from the tail.

A 30-minute watchdog is not a termination rule: it flags an infrastructure failure and re-queues the run.

**Budgets.** B: per-call cap on prompt + generation, compression included. G: generation cap. O: largest observation. L: ledger cap. I: compression instruction (~128 tokens). S: `SYS` + `QUESTION` (measured in S0; 2,500 assumed).

| B | G | Search k × s | O | L | Threshold T = B − G − I − L | Post-compression cap T − G − O |
|---|---|---|---|---|---|---|
| 8k | 1,024 | 3 × 256 | ~900 | 1,024 | 6,016 | ~4,090 |
| 16k | 2,048 | 5 × 512 | ~2,720 | 1,920 | 12,288 | ~7,520 |
| 32k | 3,072 | 5 × 512 | ~2,720 | 4,096 | 25,472 | ~19,680 |
| 64k | 4,096 | 5 × 512 | ~2,720 | 6,144 | 55,168 | ~48,350 |

**Invariants** (unit-tested with maximum-size turns):
1. **Hard cap.** Compression fires *before* an observation that would cross T is appended; the held observation follows the new ledger. Agent calls start at ≤ T (T + G ≤ B); compression prompts are ≤ T + G + I, and T + G + I + L = B.
2. **Progress.** S + new ledger + kept tail (the call that produced the held observation, without thinking, ~300 tokens) + held observation is ≤ T − G − O, so one worst-case turn fits.

At 16k this holds with S = 2,500 (7,440 ≤ 7,520). If S0 measures more, L at 16k shrinks by the excess (floor 1,536) and the table is re-derived; 8k needs S ≤ ~1,850, else "not run". A ledger hitting L loses its last incomplete entry (counted). Calls over B are logged (expected: zero).

**ReAct baseline:** `search`, `open`, `find`, `answer` only; no ledger or compression. Given ACM's 63k mean peak (§5.1), many 64k runs may hit the ceiling; hit rate and non-truncated accuracy sit beside the anchor. If thinking reaches G, the harness closes the block and asks for the call; truncation rate is reported.

### 7.2 Context Manager and the Prefix Cache

**Compression** (only on the §7.1 trigger):
1. Hold the triggering observation; append a fixed *"update the ledger"* instruction to the current prompt (a cache hit).
2. Emit ledger vK+1: grammar-constrained JSON, ≤ L tokens, thinking off.
3. **Validate:** each fact must cite a document seen in this trajectory, on a page containing one of its content words; failures are dropped and counted (*note-drop rate*).
4. Embed offloaded tail outputs with Qwen3-Embedding-0.6B for `recall`.
5. Re-prefill `SYS` + `QUESTION` + vK+1 + kept tail + held observation (≤ T − G − O): the only full prefill, since thinking stays verbatim between compressions and the prompt is append-only.

**Thresholds, not ACM's agent-triggered compression** (an ablation rung, §9.2), give predictable peak memory on 6 GB and countable cache breaks. At 16k a compression may come every one to three turns, each a counted re-prefill of ≤ ~7.5k tokens.

**Hybrid-model cache risk.** Gated DeltaNet state cannot be truncated to an arbitrary prefix, so `llama-server` restores from context checkpoints. Two llama.cpp issues report whole-prompt re-processing every turn:
- **#22384** (Qwen3.5/3.6-27B): the maintainer closed it ("nothing to fix in llama-server") and recommended `preserve_thinking`; the reporter still saw a full re-process with it set.
- **#24587** (Qwen3.5-35B-A3B, variable-length retrieval-augmented prompts): closed as stale.

| | |
|---|---|
| Mitigations | Byte-stable rendering, earlier thinking verbatim; append-only tail (unit-tested); `--ctx-checkpoints` tuned on the pinned build; no side calls on the agent's slot |
| S0 gate | Per-call cache-hit rate, 1 − (prompt tokens evaluated ÷ prompt length) from `llama-server` timings: **≥ 80% on non-compression turns**, reported with and without verify-adjacent turns |
| On failure | Qwen3-4B (6 GB) or Qwen3-8B (8 GB), standard attention (§4.1); neither fits 64k. The largest budget is 16k on 8 GB; on 6 GB it comes from measured usable VRAM (expected 16k; 32k only with q4_0 KV). Claim 2 then compares 16k with that budget, or with 8k; 64k points say "not run". The floor gate still applies (Qwen3-4B-thinking is ACM's collapsing model, §5.1). Reported as a finding |

### 7.3 `verify`: the Critic, Repurposed
Per call: ≤ 3 constraints (1 at 8k), one search each (`"<candidate> <constraint keywords>"`, k=3, 256-token snippets), appended as the observation, so bounded by O. The next turn gives a cited verdict per constraint (*satisfied / contradicted / unknown*) under a verdict-only grammar; a contradiction moves the candidate to `ruled_out`. No separate LLM call runs, so the single slot keeps its prefix and verify tokens count like any turn's. Reported: verification coverage; accuracy on verified vs unverified answers.

### 7.4 Retrieval Through Enclave
- **Pinned: code unchanged, configuration overridden.** Enclave's `portable` defaults (256-d Matryoshka embeddings, 256–320-token passages, conditional reranking on) do not fit this benchmark; Spelunker sets 1024 dims, 512-token passages, conditional reranking off, the BrowseComp-Plus query instruction and `hnsw.ef_search` ≥ max(100, rerank depth N). S0 parity runs under exactly these. A loader writes the prebuilt 0.6B vectors into Enclave's pgvector table.
- **Modes:** `bm25` = Pyserini sidecar; `dense` = Enclave HNSW; `hybrid` = Enclave RRF over FTS + pgvector; `rerank` = hybrid top-N + Qwen3-Reranker-0.6B, N from a measured knee per profile, N ≤ `ef_search`. The reranker stays on one device across budgets (CPU on `portable`), so wall-clock is not confounded.
- **Larger indexes:** pgvector HNSW caps at 2,000 dims (`vector`) or 4,000 (`halfvec`). The 4B vectors (2560-d) fit as `halfvec`; the 8B vectors (4096-d) use a flat reference index (100K × 4096 fp16 ≈ 0.8 GB, one matrix product per query).
- **Without Enclave**, a thin adapter (Pyserini + flat index + reranker) serves the same contract.

### 7.5 Judge
- **Local judge:** Qwen3.5-9B with the paper's grading prompt (Appendix F), as a batch job; exact match and calibration error (from stated confidence, as in Table 1) always sit beside it.
- **Calibration (S1):** gpt-4.1, the paper's judge, grades 200 dev answers (~0.3M tokens, est. < $5); 100 are also hand-labelled. The local judge is primary if κ ≥ 0.80 with a 95% bootstrap lower bound ≥ 0.70 **against both**. At 10–20% positives the κ CI from 200 answers is about ±0.1, so a point estimate is near a coin flip at the cut.
- **Otherwise** gpt-4.1 grades each test configuration as it freezes (~1,600 answers, est. < $10), with a **hard stop on 2027-01-07**. Later test answers get the local judge plus human adjudication of every local-judge vs exact-match disagreement (answers are short).
- **API risk:** third-party coverage reports a gpt-4.1 API shutdown on 2027-01-14, inside this project's window; no primary OpenAI page was found, so OpenAI's deprecations page is re-checked in S0. Calibration labels are cached; hand labels need no API.

### 7.6 Distillation
1. **Rejection sampling** (§5.2).
2. **Per-step examples in student view V_s:** `SYS` + `QUESTION` + ledger + the newest tail turns fitting in B_s tokens (older ones behind an *"[n earlier turns omitted]"* marker; thinking only in the last). Target: the teacher's thinking + tool call, or the ledger JSON at a compression step; loss masked to the target, never on observations. The student runs V_s at inference too (re-prefilling ≤ B_s per turn, cheap at 2B), matching views as in 02 Shellwright. Teachers still run at CM@16k, since S plus a ledger fills most of a 4k B_s; a 100-query dev run of the teacher in V_s separates view loss from capacity loss.
3. **Unsloth bf16 LoRA** on Qwen3.5-2B, with the Gated DeltaNet projections among the targets, or ~75% of token-mixing layers stay frozen (unit-tested).
   - *Memory:* sequence length binds. Unsloth's 5 GB figure assumes `max_seq_length = 2048`; bf16 weights alone are ~3.8 GB (BF16 GGUF: 3.78 GB) and the vocabulary is ~248k, so a 13k-token step cannot fit 6 GB beside the display reserve.
   - *S0 probe:* a one-step dummy LoRA at 2k / 4k / 8k on the card pre-registers B_s as the largest that fits (expected ~4k on 6 GB). If S + ledger cap + one turn exceeds it, the 0.8B is probed. Training time = kept examples × mean length ÷ probed throughput.
   - *16k-view student:* needs 8 GB or an optional free GPU (Kaggle P100 16 GB or 2 × T4, Colab T4; fp16, as Pascal and Turing lack bf16): an accelerator, never a dependency.
4. **Export** a merged GGUF (Q4_K_M, Q8_0) to the same `llama-server`; it runs on the Mac unchanged.
5. **Gate and pass rule:** Claim 4, §9.3.
6. **Stretch, GRPO** (exact-answer reward): fast vLLM rollouts need Linux/WSL2 and vLLM > 0.16.0, which Unsloth notes lacked Qwen3.5 support (model cards now list vLLM serving); with `fast_inference=False`, multi-turn rollouts are impractical. Expected "not run"; re-checked before S4; 0.8B only, if S4 ends early.

## 8. Hardware Profiles

> **Rule (from Enclave): the weakest machine sets the defaults.** `portable` assumes 6 GB; `accelerated` is opt-in once an 8 GB card is confirmed.

| Item (GB; buffers, CUDA context estimated) | `portable` (6 GB, default) | `accelerated` (8 GB) | `mac` (16 GB unified) |
|---|---|---|---|
| Agent weights (§4.1) | 4B, 2.74 | 9B, 5.68 | 4B, or the 2B student for the demo |
| KV @16k / @64k | 0.29 / 1.14 (q8_0) | 0.29 (q8_0) / ~0.6 (q4_0) | — |
| Buffers + CUDA context | ~0.5 + ~0.3 | ~0.5 + ~0.3 | — |
| **Total @16k / @64k** | **~3.8 / ~4.7** | **~6.8 / ~7.1** | — |
| Reranker / query encoder | CPU / CPU, at every budget | CPU / CPU | MPS (Metal) / MPS |
| Owns | All scored runs | Scored runs + the 9B anchor | Development, tests, native Pyserini, demo |

Usable VRAM (Windows reserves some for the display) is measured with `nvidia-smi` on day one; a profile within 0.5 GB of it moves a component to CPU. The 9B@64k anchor is inside that margin even with q4_0 KV, so it plans partial offload of a few layers (tok/s reported) or a 32k reference. Training never runs beside serving; artifacts cross machines as files.

**Week-2 go/no-go gate** (session 3: Windows, 4B, BM25, 20 dev queries, minimal ReAct loop with a naive 16k truncation stub; also the first floor read). **Go** if median ≤ 5 min/query, p90 ≤ 10 min and cache reuse ≥ 80%. **Otherwise, in order:** fix caching → lower the generation cap → no thinking on `note` turns → 4B primary on 8 GB, if confirmed → standard-attention fallback (only if caching is the cause) → one dev seed.

**S1 re-plan** (100 dev queries): re-tabulate from measured means and apply the floor gate. If the 4B fails:
- **8 GB:** the 9B is the scored agent, as planned.
- **6 GB:** the 9B runs with partial CPU offload; test shrinks to the largest n the minutes allow and the margin is recomputed (n = 150: SE ≈ 4.1 points, a ~10-point margin at q = 0.25).
- **Offloaded 9B over a ~10 min/query median at 16k:** Claims 1, 2 and 4 read "not testable on 6 GB". The project reframes around the measured 4B floor, the retrieval and serving results and, time allowing, a dev-only 9B budget curve (screening, not a claim).

**Compute budget.** Planning means until the gate measures them: 6 min/query at ≤ 16k, 8 min at 32k–64k, 2 min for the 2B, 0.5 min closed-book. Capacity: ~5 h a night, ~35 laptop-hours a week.

| Sprint (weeks) | Runs (configuration × split × seeds) | Laptop-h | Capacity |
|---|---|---|---|
| S0 (1–2) | Week-2 gate: 20 dev | ~2 | 70 |
| S1 (3–5) | ReAct@64k dev: BM25 × 2 seeds, dense, hybrid, rerank × 1 (500 × 8 min); closed-book dev + test (330 × 0.5); thinking check (30 × 6); **ReAct@64k-BM25 test** (230 × 8) | ~103 | 105 |
| S2 (6–7) | **ReAct@64k-best test** (230 × 8); CM smoke runs (20 × 6); CM@16k dev × 2 seeds (200 × 6); − `verify` rung (100 × 6) | ~63 | 70 |
| S3 (8–9) | CM@64k, CM@32k dev (200 × 8); CM@8k dev if runnable (100 × 6); **CM@16k test** (230 × 6); teacher sampling begins (~100 × 6) | ~70 | 70 |
| S4 (10–11) | Teacher sampling completes (~400 × 6); **CM@64k test** (230 × 8); 2B training (measured); student and untrained 2B on test (460 × 2) | ~86 + training | 70 |
| Buffer (12) | Overflow from S4 | — | 35 |

Total ~325 laptop-hours plus training; the week-12 buffer absorbs S4's overflow at planning means. Local-judge batches share this capacity (measured in S1). **Levers first:** two `llama-server` slots (`-np 2`) at 16k (4B KV 0.29 GB each) if S0 shows a throughput gain with reuse intact; teacher sampling on a free Kaggle/Colab GPU, never assumed. **Then cuts, in order:** − `recall` and ACM-style rungs (slack only, else "not run"); 8k rung; 8B row; teacher-in-V_s diagnostic; CM@32k dev cut to 50 queries; full-830 union (needs a post-freeze dev re-run, 100 × 6 min); second teacher sample.

## 9. Evaluation Plan

### 9.1 Areas

| Area | Reported |
|---|---|
| Accuracy (every run) | Judge accuracy, exact match, calibration error vs the local anchor (published rows as context only) |
| Telemetry (every run) | Per-call telemetry (§6); compressions, note-drop rate, cap and ceiling hit rates, calls over B |
| Serving | Cache-hit rate, prefill/decode tok/s, VRAM peak per profile |
| Retrievers | Search calls (verify-internal apart), in-trajectory evidence recall (Table 1's Recall), rerank latency; vs BM25 at the same budget |
| Distillation | Student ÷ teacher accuracy on the same GPU |
| Closed-book | Expected ≈ 0; hits reported apart |

### 9.2 Ablation Ladder
1. ReAct @64k, BM25 (anchor) → 2. dense-0.6B → 3. best of hybrid / rerank → 4. + ledger (CM @64k) → 5. compression at 32k / 16k / 8k (8k only if runnable, §7.1) → 6. − `recall` → 7. − `verify` → 8. ACM-style agent-triggered compression @16k → 9. 2B student in V_s, untrained 2B as the floor. Rungs 6–8 are dev-only, one seed; 6 and 8 run only from slack (§8).

### 9.3 Pre-registered Effects and How n Was Chosen
Paired binary outcomes with discordance rate q have SE(Δ) ≈ √((q − Δ²)/n). Planning uses q = 0.25, re-estimated in S1 from seed-vs-seed disagreement (also reported as the noise floor). CIs: paired bootstrap over queries, 10,000 resamples; dev averages 2 seeds where run, test uses 1.

Two-sided MDE at α = 0.05, 80% power: **dev** n = 100, SE ~5.0 pts, MDE ~14 pts (screening only); **test** n = 230, SE ~3.3 pts, MDE **~9 pts** (all claims). The MDE is for planning (smaller test effects may be missed), never a pass bar: requiring observed Δ ≥ MDE passes only ~50% of the time at a true effect of that size.

| Claim | Statistic | Decision rule |
|---|---|---|
| 1. Headline | Δ = CM@16k − ReAct@64k, two-sided 95% CI | "CM@16k better" if the CI lies above 0, "worse" if below, else "no difference detected at this n" |
| 2. Non-inferiority | One-sided 95% upper bound of (CM@64k − CM@16k) | Pass if **< 8 points**. Testable only if the floor gate passed; the estimate on queries whose CM@64k peak exceeded 16k is reported beside it |
| 3. Retriever | Δ = best retriever − BM25 inside ReAct@64k, two-sided 95% CI | Detected if the CI excludes 0, else "not detected at this n". Only the dev-chosen retriever goes to test; Holm correction if more than one goes |
| 4. Student | CI of (student − untrained 2B) | Runs only if the teacher yields ≥ 150 kept trajectories. Pass if the CI excludes 0 **and** the student gives ≥ 2× the teacher's queries/hour. Reported: fraction of teacher accuracy recovered, with its CI |

**Why 8 points:** at n = 230 and q = 0.25, Claim 2 has ~80% power when the true difference is 0; hence test n = 230. It is the smallest margin testable at this n (a quarter of a 32% anchor), not a judgement that 8 points do not matter.

### 9.4 Honest-Reporting Rules
- ACM's 57.0% always carries its split and unstated retriever. No re-splitting (§5.1).
- The full-830 number is the union of the final teacher's dev, test and train-sample-1 runs, only if the config froze before all three.
- Judge accuracy always appears with exact match, κ, the grading judge and cap/ceiling hit rates; failed claims carry CIs.
- Blocked claims say "not testable on 6 GB" (§8), never a vacuous pass; rows not run (8B index, GRPO, the ~20 GB Qwen3-32B judge) say "not run", never filled from other papers.
- Decrypted data and plaintext trajectories stay out of git (`.gitignore` + canary pre-commit check); artifacts use query IDs.

## 10. Milestones (19 hours / 15 sessions)

Runs per sprint: §8. Each test configuration runs, and is graded, as it freezes.

| Sprint | Sessions (weeks) | Deliverable |
|---|---|---|
| S0 | 1–3 (1–2) | Repo, Enclave config, corpus, decryption, indexes, Pyserini sidecar; minimal ReAct loop; **Table 2 reproduced**; `llama-server` pinned; reuse, VRAM, S tokens measured; LoRA probe; splits frozen; **week-2 gate** (session 3) |
| S1 | 4–7 (3–5) | Full loop + 7 tools; dev anchor; rerank-depth knee; closed-book; judge calibration; **floor gate and re-plan**; ReAct@64k-BM25 frozen and run on test. Week-5 demo: Wi-Fi-off ReAct trace with a cited answer |
| S2 | 8–9 (6–7) | Best retriever frozen; **ReAct@64k-best on test**; 8B parity check; ledger, compression, `recall`, `verify` with the §7.1 invariant tests |
| S3 | 10–11 (8–9) | Dev budget curve; **CM@16k frozen in week 8, run on test**; teacher sampling starts overnight. Demo: Wi-Fi-off run with field notes |
| S4 | 12–14 (10–11) | Teacher sampling completes; **CM@64k on test**; 2B RFT in V_s; GGUF export; student on test; full-830 union if eligible; runbook; week 12 is the compute buffer |
| — | 15 (12) | Interview sprint: replayed demo, budget-curve walk-through, cache measurements on a whiteboard (the bug and its fix only if one was found) |

## 11. Risks & Mitigations

| Risk | Mitigation |
|---|---|
| Hybrid-model cache misses force a re-prefill every turn | §7.2: mitigations, S0 gate, standard-attention fallback |
| 4B at the floor: too few tool calls for the budget claims to bind | Week-2 read; S1 floor gate; offloaded 9B, or "not testable on 6 GB" (§8) |
| ~325 laptop-hours, loaded towards the end | §8 re-plan, buffer, ordered cuts; free GPU only as an accelerator |
| 9B does not fit 6 GB; Mac too slow | 4B is the stated 6 GB default; the Mac owns no scored number |
| Judge dependence; gpt-4.1 retirement | §7.5 κ rule, exact match beside, cached labels, hard stop 2027-01-07 |
| Student training: VRAM or yield | S0 probe sets B_s; 0.8B fallback; ≥ 150 kept trajectories; underpowered claims labelled |
| Prior art (ACM, ReSum, ACON, Context-Folding) | Cited as the start; the delta is the laptop budget curve, small-scale retriever study and 2B student |

## 12. Résumé Bullets (experience voice; numbers filled in as sprints land)

- Built an **offline deep-research agent** over a frozen 100K-document corpus (BrowseComp-Plus), running Qwen3.5 via llama.cpp on a **6 GB laptop GPU** at **X% accuracy** (exact match X%, judge κ = X).
- Designed a **budget-bounded context manager** (cited ledger, offloaded evidence, threshold compression, append-only prompt) within **X points** of a 64k-context agent at a **16k cap**, cutting prefill **X%**.
- Measured **prefix-cache reuse on a hybrid Gated DeltaNet model** at **X%** per call (gate ≥ 80%) with byte-stable, append-only prompts. *Append "; diagnosed and fixed a re-prefill bug, raising reuse from X% to X%" only if a below-gate baseline was measured and a change raised it.*
- Ran an **in-agent retriever ablation** (BM25, Qwen3-Embedding-0.6B, hybrid RRF, Qwen3-Reranker-0.6B): retrieval moved accuracy **X points**, after reproducing published nDCG@10 within **X**.
- **Distilled the agent into Qwen3.5-2B** via per-step rejection-sampling fine-tuning, recovering **X% of teacher accuracy** at **Xx** throughput.

### Placement Note (read before putting this under EXPERIENCE)

These bullets go under EXPERIENCE **only with a real organization and a real engagement**:
1. **The mentoring organization as employer**, with a truthful title (`ML Engineering Practicum (mentored project)`), only where an engagement record exists; "Intern" only if the organization issued that title in writing and will confirm it on a reference check.
2. **A real collaboration**: a startup, lab or open-source org that agrees to host the work.
3. **An open-source project entry**: a public repo with real commits, the results table and the demo video (benchmark data excluded).

Never invent an employer or dates: North American employers verify employment, and a fabricated entry is grounds for a rescinded offer; a public repo with a reproducible results table survives scrutiny better than an unverifiable internship line.

## 13. Why This Reads as Real Company Work

- **A deployment constraint, swept and priced:** "the prompt must fit the memory we have" is how on-device and on-prem agent teams work.
- **Anchor before claim:** S0 reproduction, then the local baseline and floor gate.
- **Serving behaviour as a result:** per-call reuse on hybrid models, a known llama.cpp pain point (§7.2; both issues closed without a server-side fix).
- **Retrievers judged on agent accuracy**, as by a search team's agent customer.
- **Budgeted experiments:** laptop-hours, a cut list, pre-registered MDEs, one touch of test.

## 14. Interview Leverage

- **ML system design** ("research assistant over internal docs"): loop, tools, budgets, termination, memory, with numbers.
- **LLM serving:** hybrid-attention KV arithmetic, quantization, prefix caching on recurrent layers, VRAM budgeting.
- **Search/IR:** BM25 vs dense vs rerank end to end; pgvector dimension limits.
- **Training:** rejection sampling, per-step distillation, loss masking, LoRA targets on non-standard layers, why not QLoRA.
- **Experimentation:** paired designs, MDEs, non-inferiority margins, judge κ, contamination controls.
- **Behavioral:** compute and floor gates, dropping LangGraph for byte-stable prompts, reporting failed claims.

## 15. References

- BrowseComp-Plus (Chen et al., 2025): https://arxiv.org/abs/2508.06600
- BrowseComp-Plus code and index recipe: https://github.com/texttron/BrowseComp-Plus · https://github.com/texttron/BrowseComp-Plus/blob/main/scripts_build_index/qwen3-embed.md
- BrowseComp (Wei et al., 2025): https://arxiv.org/abs/2504.12516
- ACM (Li et al., 2026), including Appendix D (Qwen3-4B-thinking): https://arxiv.org/abs/2607.23809
- ReSum (Alibaba Tongyi, 2025): https://arxiv.org/abs/2509.13313
- ACON (Kang et al., 2025): https://arxiv.org/abs/2510.00615
- Context-Folding (Sun et al., 2025): https://arxiv.org/abs/2510.11967
- Qwen3.5 cards: https://huggingface.co/Qwen/Qwen3.5-9B · https://huggingface.co/Qwen/Qwen3.5-4B · https://huggingface.co/Qwen/Qwen3.5-2B
- Qwen3.5 chat template (thinking retention): https://huggingface.co/Qwen/Qwen3.5-4B/blob/main/chat_template.jinja
- GGUF sizes: https://huggingface.co/unsloth/Qwen3.5-9B-GGUF · https://huggingface.co/unsloth/Qwen3.5-4B-GGUF · https://huggingface.co/unsloth/Qwen3.5-2B-GGUF · https://huggingface.co/Qwen/Qwen3-4B-GGUF · https://huggingface.co/Qwen/Qwen3-8B-GGUF
- Unsloth Qwen3.5 fine-tuning: https://unsloth.ai/docs/models/qwen3.5/fine-tune
- Qwen3-Embedding: https://huggingface.co/Qwen/Qwen3-Embedding-0.6B · https://huggingface.co/Qwen/Qwen3-Embedding-8B-GGUF
- llama.cpp hybrid-cache issues: https://github.com/ggml-org/llama.cpp/issues/22384 · https://github.com/ggml-org/llama.cpp/issues/24587
- pgvector HNSW limits: https://docs.pgedge.com/pgvector/v0-8-2/hnsw/ · `hnsw.ef_search` (default 40 caps returned rows): https://github.com/pgvector/pgvector#query-options
- Pyserini install: https://github.com/castorini/pyserini/blob/master/docs/installation.md
- gpt-4.1 API retirement (third-party): https://thenewstack.io/openai-gpt4o-api-shutdown · https://www.digitalapplied.com/blog/gpt-4o-gpt-4-1-api-shutdown-january-2027
