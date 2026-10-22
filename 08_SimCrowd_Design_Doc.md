# SimCrowd — Design Document

**Hundreds of synthetic users per arm on a laptop: offline A/B tests for recommenders, with a fidelity gate that says when not to trust them**

| | |
|---|---|
| Status | Proposal |
| Target duration | ~10–12 weeks (19 mentoring hours / 15 sessions + independent work) |
| Platforms | Windows 11 laptop (16 GB RAM, RTX 3060-class, 6 GB VRAM assumed, 8 GB if confirmed) and Apple Silicon Mac (16 GB, no CUDA), one repository |
| Deployment | 100% local; no rented servers or paid GPU cloud. Optional: a commercial-API baseline (< $20) and free Colab/Kaggle GPUs as accelerators. |
| Core stack | llama.cpp `llama-server` (CUDA / Metal) · Qwen3.5-4B · Qwen3.5-9B · Gemma 4 E4B (GGUF Q4_K_M) · Qwen3-Embedding-0.6B · RecBole · FastAPI · DuckDB · pytest · Unsloth/PEFT |
| Sprints | S0 setup and go/no-go (sessions 1–3) · S1 personas and taste alignment (4–8) · S2 A/B and real-exposure validation (9–10) · S3 calibration (11–12) · S4 analyst and hardening (13–14) · session 15 interview sprint |

---

## 1. The Product Thesis

Offline recommender-system (recsys) metrics disagree with online A/B results, and online tests are slow and risky. LLM user simulators allow offline A/B tests on synthetic users: Agent4Rec (2023) ran 1,000 gpt-3.5-turbo agents on MovieLens-1M and got a plausible ranking of five recommenders.

The catch is fidelity. *Mind the Sim2Real Gap* (COLM 2026) compared 31 LLM simulators with 451 real people on τ-bench: the best scored a User-Sim Index of **76.0**, against **92.7** for human inter-annotator agreement. Simulators were too cooperative, uniform and positive. One that likes everything ranks recommenders by the wrong criterion, silently.

**SimCrowd treats the simulator as a model**: a validation set from real behaviour, calibration, a versioned model card, and a continuous-integration (CI) gate that **refuses to publish an A/B readout** when fidelity fails.

> **Engineering thesis:** small open-weight agents on one consumer GPU can run a credible offline A/B test, *if and only if* their fidelity is measured against real behaviour and enforced as a release gate.

Four claims, each proven or refuted by measurement (decision rules in §9):

1. **Local agents replicate published taste alignment.** On a port of Agent4Rec's MovieLens-1M protocol, Qwen3.5-4B (default), Qwen3.5-9B and Gemma 4 E4B are non-inferior (margin 0.03) to published gpt-3.5-turbo 1:1 accuracy (0.6912) and 1:9 F1 (0.3232). Their lower 95% bounds also clear the trivial floor and a matrix-factorization (MF) score classifier at every ratio. A port delta (§5.3) and title masking (§7.3) measure harness and memorization effects.
2. **Simulated orderings track real behaviour better than cheap baselines.** On real randomized exposures (KuaiRand, §5.2), item-level click AUC (area under the ROC curve) is non-inferior to a logistic click model (G6: AUC ≥ max(0.60, click-model AUC − 0.02), §7.5). Arm-level Kendall τ(sim, real) beats τ(offline Recall@20, real) and τ(click model, real). MovieLens agreement with RecBole's ordering is reported, not targeted.
3. **Calibration fixes agreeableness.** Judge-free like-rate inflation, rating KL (Kullback–Leibler divergence) and 1:9 precision are compared before and after post-hoc logit calibration and a bf16 LoRA (low-rank adaptation) of a small Qwen3.5 on real histories. The bar: a non-LLM rating-model floor, plus behaviours the LoRA never saw (§7.6).
4. **Cheap enough to run weekly.** A 300-user arm in ≤ 2 h on the 6 GB GPU (§8), a 7-arm study in a weekend, $0 inference. Enablers: paired common random numbers (CRN), an append-only cache-friendly prompt, grammar-constrained non-thinking actions.

**Why it is surprising.** We A/B test a recommender on people who do not exist, then A/B test the people. Agent4Rec's 1:1 F1 barely beats "watch everything" (0.698 vs. 0.667, §5.1), and its top three arms differ by only 0.05–0.10 satisfaction points. Whether a simulator can resolve such gaps was never reported. SimCrowd measures it (§9).

## 2. Goals & Non-Goals

### Goals
- Agent4Rec-style **user agents** (§7.3) on a local 4B via `llama-server` parallel slots, facing RecBole arms (Random, Pop, BPR-MF, MultiVAE, LightGCN, SASRec) and a planted **Clickbait** arm (§7.2).
- **Validation** on KuaiRand random exposures, MIND as labelled fallback (§5.2); judge-free pytest **gates** (§7.5); post-hoc, then LoRA, **calibration** (§7.6); a paired, pre-registered design (§9).
- An **analyst agent** that publishes or refuses each readout (§7.7); a versioned **model card** with drift monitors (§7.8).

### Non-Goals
- **A better recommender:** FeedForge, ShopBrain and TuneFlow build those; SimCrowd is the instrument that evaluates them.
- **Replacing online tests:** readouts pre-screen; they never decide launches.
- **A conversational or multimodal simulator:** optional TAU-USI (§5.3) only scores the simulator 05 and 06 use. Also out: pre-training; QLoRA on Qwen3.5 (vendor guidance advises against it).
- **Record/replay and nondeterminism tooling:** that is BlackBox (05). SimCrowd logs only enough to replay a session.

## 3. Alignment With Prior Experience

| Prior experience (skill) | SimCrowd module |
|---|---|
| **DiDi**: backend, A/B testing (experiment design) | CRN pairing, power, confidence intervals (CIs), gates; FastAPI `rec-svc` |
| **Changsheng Fund**: backtests, AutoML hyper-parameter agent (realism, tool use) | A simulated A/B is a recommender backtest, and the fidelity gate tests its realism; the analyst agent upgrades the AutoML agent |
| **Sina Weibo**: SQL ETL, metrics, alerting (pipelines, monitoring) | Persona-builder ETL with data-quality checks; drift monitors on gate margins and Population Stability Index (PSI) |
| **College Application Agent**: embedding-based retrieval (EBR) + FastAPI, Writer–Critic (retrieval, self-critique) | Agent memory retrieval; the analyst's number checker |
| **CAS**: retrieval over 20+ configs (sweeps) | Throughput sweep, ablation ladder |
| **Music Genre**: PCA, K-means (clustering) | Persona segments for stratified fidelity reports |

**Interview one-liner:** *"At DiDi I ran A/B tests; at Changsheng I learned a backtest is only as good as its realism. SimCrowd joins the two: offline A/B tests on LLM users, with a CI gate that refuses to publish when they stop behaving like real ones."*

## 4. Stack Choices — and Why

### 4.1 Models

| Role | Model | Architecture | File (GGUF) | Where |
|---|---|---|---|---|
| **Default population** | **Qwen3.5-4B** | 32 layers = 8 × (3 Gated DeltaNet + 1 Gated Attention) | Q4_K_M 2.74 GB (no vision projector) | Both |
| Size cohort | Qwen3.5-9B | 32 layers, same 3:1 layout | Q4_K_M 5.68 GB | Mac |
| Cross-family cohort | Gemma 4 E4B-it | 4.5B effective / 8B total; 42 layers; 512-token sliding window + global | Q4_K_M 4.98 GB | Mac (tight on 6 GB) |
| Standard-attention fallback | Qwen3-4B-Instruct-2507 | 36 layers, grouped-query attention (GQA) 32 Q / 8 KV; non-thinking | Q4_K_M 2.5 GB | Both |
| LoRA student | Qwen3.5-0.8B / 2B | 24 layers; non-thinking default | bf16 LoRA ~3 / ~5 GB VRAM (Unsloth) | Windows |
| Agent memory | Qwen3-Embedding-0.6B | 0.6B | bf16 ~1.2 GB | CPU / MPS |

All are tagged Apache-2.0 (S0 re-checks Gemma 4's linked license file); arms are RecBole 1.2.1 models. Qwen3.5-4B and 9B think by default and dropped the `/think` switches, so the population sets `chat_template_kwargs: {"enable_thinking": false}`. Thinking-on is an ablation rung, since Sim2Real found capability does not imply fidelity. Three families guard against a Qwen-only artifact.

### 4.2 Runtime and libraries

**`llama-server` on both machines** (CUDA / Metal): OpenAI-compatible API, parallel slots, per-request `seed`, JSON-schema/GBNF output, logprobs, tool calls. llama.cpp #22384, #24587 and #24714 report full prompt re-processing on DeltaNet models as hybrid checkpointing changed across 2026 builds, so the model card pins the build: **a runtime upgrade is a simulator change**. Rejected: vLLM (no native Windows), MLX serving (forks the code path). MLX-LM is an optional Mac LoRA path; S0 checks its Qwen3.5 support.

**Libraries** (four `uv` projects): `sim` (Python 3.12) re-implements Agent4Rec's logic (no `reckit`). `arms` (3.11) runs RecBole 1.2.1 pinned to `numpy<2`, `torch<2.6` (open RecBole #2215: NumPy 2.x and PyTorch ≥ 2.6 break it), CUDA on Windows, CPU/MPS on the Mac, and exports arm pages as Parquet for `rec-svc`. `train` holds Unsloth/PEFT. `a4r-orig` runs Agent4Rec's original code on its tested Python 3.9, for the port delta only. Shared: DuckDB + Parquet run store · pandera · numpy bootstrap, scipy `kendalltau`, statsmodels (power, Holm) · pytest · Streamlit · httpx + asyncio, one semaphore per slot.

## 5. Benchmarks & Datasets

### 5.1 Agent4Rec protocol on MovieLens-1M (measured, fully local)

MovieLens-1M: ~1M ratings, ~6,000 users, ~4,000 movies, 6 MB. Agent4Rec simulates 1,000 agents: 4 movies per page, exit after ≤ 5 pages, 1–10 satisfaction. Taste alignment shows each agent 20 items at ratio 1:m (interacted but unused for the profile : not interacted). Published gpt-3.5-turbo results (Table 1):

| Ratio | Acc | Recall | Precision | F1 | "Watch all" F1 * |
|---|---|---|---|---|---|
| 1:1 | 0.6912 | 0.7460 | 0.6914 | 0.6982 | 0.667 |
| 1:2 | 0.6466 | 0.7602 | 0.5058 | 0.5874 | 0.500 |
| 1:3 | 0.6675 | 0.7623 | 0.4562 | 0.5433 | 0.400 |
| 1:9 | 0.6175 | 0.7753 | 0.2139 | 0.3232 | 0.182 |

\* Computed here, not from the paper: precision 1/(1+m), recall 1.

Table 2, the strategy evaluation we treat as a simulated A/B (view rate, like rate, mean exit page, mean 1–10 satisfaction):

| Arm | P̄view | P̄like | N̄exit | S̄sat |
|---|---|---|---|---|
| Random | 0.312 | 0.269 | 2.99 | 2.93 |
| Pop | 0.398 | 0.360 | 3.01 | 3.42 |
| MF | 0.488 | 0.462 | 3.17 | 3.80 |
| MultVAE | 0.495 | 0.452 | 3.10 | 3.75 |
| LightGCN | 0.502 | 0.465 | 3.02 | 3.85 |

The code (MIT) targets Python 3.9.12, PyTorch 1.13.1 and the OpenAI API; SimCrowd ports its prompts and logic, not its environment. **Memorization:** Di Palma et al. (SIGIR 2025) find GPT and Llama models memorize MovieLens-1M items, profiles and interactions, more so at larger sizes; hence masking (§7.3) and KuaiRand.

### 5.2 Real-exposure validation: KuaiRand-Pure (primary)

| Fact | Value |
|---|---|
| Source | Kuaishou short-video logs, CC BY-SA 4.0 |
| Scale | 27,285 users; standard logs 7,551 items, 1,436,609 interactions; random pool 7,583 items |
| Random exposures | 1,186,059 interactions (4/22–5/08) where a recommendation was replaced by a random-pool item (~0.37% of traffic) |
| Feedback | 12 signals, e.g. `is_click`, `long_view`, `is_like` |
| Text | Supplementary captions, cover text, 4-level categories (examples in Chinese) |

**S0 dataset decision** (criterion: side information to render items as text; a 2026-10 desk check favours KuaiRand):
- **Open Bandit Dataset (OBD, ZOZOTOWN):** truly randomized (Uniform Random vs. Bernoulli TS), but items are anonymized `item_feature_0..3`, no text. Expected to fail.
- **KuaiRand:** expected primary. S0 measures caption coverage on random-exposure items and tests replacement uniformity (χ²); if not uniform, replay uses inverse propensity scoring (IPS) with empirical propensities.
- **MIND (fallback):** ~160k English news, 15M+ impressions, 1M users (MIND-small: 50k); Microsoft Research License. Exposures follow the production policy, so the claim becomes "agreement with real logged behaviour."

**KuaiRand arm study.** Five arms (Random, Pop, BPR-MF, LightGCN, SASRec) train on 4/08–4/21 standard logs only and rank the 7,583-item random pool, so each top-20 (5 pages × 4) can match random events. **Real value:** replay `is_click` rate on random events whose item is in the user's arm top-20. **Simulated value:** mean calibrated P(click) over that top-20, 500 sampled test users. **Non-LLM baselines:** offline Recall@20 / NDCG@20 on held-out standard logs; a logistic click model (user category histogram + item popularity, fitted on calibration users, geometric exit rule). No published LLM-simulator baseline on KuaiRand is known; KuaiSim (NeurIPS 2023), a non-LLM simulator on KuaiRand, is related work only.

### 5.3 Other measurements and data

- **RecBole offline metrics:** Recall@20 / NDCG@20 per arm, both datasets, user-level bootstrap CIs; RecBole's own ML-1M runs sanity-check the arm code.
- **Optional LightGCN reproduction** (S1, overnight): LightGCN-PyTorch reports Gowalla Recall@20 0.1824 / NDCG@20 0.1547 (3 layers, seed 2020, 1,000 epochs). Matching within ±0.005, then running RecBole's LightGCN on that split, checks graph-CF code only.
- **Port delta (S1).** `gpt-3.5-turbo` shuts down on 2026-10-23 and Agent4Rec's 2023 snapshots (`-0301`, `-0613`) were retired on 2024-09-13, so the published run cannot be repeated. Instead `a4r-orig` and the port call one local Qwen3.5-4B endpoint (200 users at 1:9; same users, items, seeds): F1(port) − F1(original), paired bootstrap CI. If the original cannot take a local base URL, compare rendered prompts and parsed decisions on 50 users.
- **Optional API reference:** a current low-cost OpenAI model (e.g. `gpt-5.6-terra`, the replacement the deprecations page lists), 200 users × 1:9, port only, labelled "reference, not a reproduction"; cost measured and capped at $5.
- **Optional, gated: TAU-USI** (495 annotated human τ-bench conversations, 165 tasks: 50 airline, 115 retail; CC BY-NC 4.0, terms prohibit LLM training). Evaluation only: outcome expected calibration error (ECE) and evaluative mean absolute error (MAE) of the τ²-bench user simulator in 05 BlackBox and 06 Ctrl-Z (local Qwen3.5 via LiteLLM). No official scoring code found.
- **LoRA data:** LoRA-train users, M1-masked inputs; targets are real watch (interacted vs. not) and rating tokens on held-out items. Distractors are non-interacted, not observed skips (stated on the model card); KuaiRand's observed skips allow a cleaner variant.
- **Demo:** 4–8 synthetic users (one per slot) browse, rate and exit a MovieLens feed live, beside a 300-user run replay and the publish/refuse gate panel.

## 6. Architecture

```
 OFFLINE ETL                                     FIDELITY CI (pytest, blocking)
 MovieLens-1M ─┐                                 taste-F1 · rating KL · like-inflation
 KuaiRand-Pure ┼─► persona-builder ─► personas   clickbait · uniformity · real-exposure
 (MIND)       ─┘   pandera · splits   .parquet                 ▲
                                          │                    │
 study.yaml ─────────────────────► sim-orchestrator ───► run store (DuckDB/Parquet)
 arms · cohort · seeds · model     CRN seeds · slot pinning      │  per call: prompt
 · calibration version             retries · resume · budgets    │  hash, seed, output
                    ┌─────────────────┴──────────────┐           │
                    ▼                                ▼           ▼
          USER-AGENT RUNTIME                rec-svc (FastAPI)   ANALYST AGENT
          profile · memory · action         Random Pop BPR      read-only tools ──►
          1-token decisions                 MultiVAE LightGCN   publish_readout
                    │                       SASRec Clickbait      (gates pass +
                    ▼                                              numbers check)
          llama-server (CUDA | Metal)                           or refuse_readout
          Qwen3.5-4B Q4_K_M · -np N                                     │
          Qwen3-Embedding-0.6B (CPU | MPS)          readout.md · model card · dashboard
```

**Walk-through.** Logs become personas and user splits (§7.1). The orchestrator expands a study into (arm × seed × user) sessions on pinned slots (§7.4); each loops page by page until exit (§7.3). Pytest runs the gates (§7.5); the analyst publishes or refuses (§7.7).

## 7. Component Design

### 7.1 Persona Builder
- **MovieLens:** taste summary of 25 sampled rated movies (once, by the 9B, cached); Agent4Rec traits (activity = interaction count, conformity = mean squared gap to item-average ratings, diversity = distinct genres); own rating histogram.
- **KuaiRand:** from 4/08–4/21 standard logs (clicked and long-viewed captions, category histogram, like rate); evaluation uses random exposures from 4/22 on, so no time overlap.
- **Splits** by user hash: calibration 20%, test 40%, LoRA-train 40%. Pandera checks every table and fails on bad joins.

### 7.2 Recommender Service and the Clickbait Arm
`GET /arms/{arm}/page?user_id=&page=` returns 4 unseen items.
- **Clickbait arm** (fixed before any run): MovieLens items with ≥ 50 ratings in the top popularity *and* bottom mean-rating quartiles, ordered by MF score, each with a neutral synopsis and a hype rewrite (9B, offline, cached). Known real ratings ground G4; the titles are famous, so G4 runs at M1 (M0 beside) to test hype, not memory.
- **KuaiRand clickbait set** (fixed in S1): items with ≥ 50 random exposures among calibration users, in the top click-rate *and* bottom `long_view`-given-click quartiles; ground truth for report-only G4-K.

### 7.3 User-Agent Loop

| Element | Specification |
|---|---|
| **Observation** (append-only) | Fixed prefix: system block (role, rules, schema; same for all agents) + persona. Each page is **appended** as a user turn: top-5 retrieved memories, page *t* of ≤ 5, 4 item cards (ID, title *or* mask, genres, synopsis ≤ 60 words, optional quality/popularity buckets). Turns are never edited, so call *k*+1 strictly extends call *k*; rendered once via `/apply-template`, grown via `/completion` |
| **Action** (per page) | (a) Per item, a 1-token `yes`/`no` watch call read via `top_logprobs` (S0 checks both are single tokens in each family's tokenizer); calibrated P(watch) is sampled with the CRN uniform and the *sampled* token appended. (b) Per sampled watch, a 1-token `1`–`5` rating call, same handling. (c) One schema-enforced JSON conditioned on (a)–(b): `{feelings: {id: ≤ 20 words}, continue: bool, satisfaction ∈ 1..10 iff continue=false}`. Uncalibrated rungs: identity calibration. KuaiRand: `click`, then `long_view` iff click |
| **Tools / memory** | No external tools. Factual entries ("watched X, rated 2") + one emotional entry per page, embedded; top-5 by cosine |
| **Termination** | `continue=false`; 5-page cap (forced-exit satisfaction prompt); prompt > 5.5k of the 6k slot; 2 consecutive invalid outputs (session invalid) |
| **Budgets** | Per page ≤ 8 one-token calls + 1 JSON call (≤ 192 output tokens); ≤ 5 pages + 2-question interview (≤ 80 tokens each); 120 s timeout; 2 same-seed retries. Port: Agent4Rec's own format, ≤ 1,024 tokens. Thinking rung: ≤ 2,048 thinking tokens per page on `-c 32768 -np 2`, timed separately |

**Taste mode** (as in Agent4Rec): 20 items at 1:m in one turn, then 20 one-token watch calls with no feelings; the positive count is not given (giving it is an ablation rung). **Masking:** M0 shows titles; M1 (default) uses opaque IDs in page and persona history; M2 also drops synopses. Memorization delta = F1(M0) − F1(M1), paired bootstrap CI.

### 7.4 Simulation Orchestrator
- **CRN:** seed = hash(study, user, page), identical across arms; a 50-user pilot also runs independent seeds to measure the variance reduction.
- **Slot pinning:** one slot per session (`id_slot`, `cache_prompt`). Cache metric: tokens re-processed per call (`timings.prompt_n`) minus new tokens. llama.cpp cannot trim hybrid DeltaNet recurrent state to an arbitrary prefix (#24587, #24714), so the prompt never branches or edits.
- **Resumable** (sessions keyed on study, arm, seed, user). **Determinism:** batched inference is not bitwise reproducible; S4 re-runs a 50-user arm with identical seeds and reports the spread.

### 7.5 Fidelity Gates

Thresholds live in a committed `gates.yaml`, frozen **at the start of S1, before any test-user run**; changes need a dated changelog entry. Anchors: calibration users and a real-vs-real reference at the simulated sample size. A/B items mostly lack real ratings, so G2, G3 and G5 use a **probe set** of 50 MovieLens items with ≥ 200 real ratings, fixed in S1. A probe call lists up to 12 probe items the user really rated (study masking level); only those pairs count. Gate runs use 600 test users.

| Gate | Metric (no LLM judge) | Measured on | Threshold |
|---|---|---|---|
| G1 taste | 1:9 F1, 1:1 accuracy | Taste calls, study masking (M0 beside) | ≥ published − 0.03; lower 95% bound above trivial floor |
| G2 ratings | KL(real ‖ sim), 1–5 histograms, add-one smoothing, per segment | Probe pairs | ≤ 95th percentile of real-vs-real KL at the same per-segment n (1,000 bootstrap draws) |
| G3 agreeableness | R = sim P(rating ≥ 4 \| watch) / real P(rating ≥ 4) | Probe pairs | 0.8–1.25 |
| G4 clickbait | Sim − real mean rating, neutral text | Clickbait items at M1; M0 beside | CI upper ≤ +0.3 |
| G5 uniformity | Between-user rating SD per item, sim / real | Probe items | ≥ 0.7 |
| G6 real exposure | KuaiRand click AUC; click-rate inflation | Test users' random exposures | AUC ≥ max(0.60, click-model AUC − 0.02), popularity and category click-through-rate AUCs beside; inflation 0.8–1.25 |
| G7 run health | Invalid sessions; CRN pairing | All sessions | ≤ 2%; 100% |
| Report only | Framing Δ: post-watch rating, hype vs. neutral | Clickbait items | — |
| Report only (G4-K) | Sim P(click) − P(`long_view` \| click) vs. real gap | KuaiRand clickbait set | — |

G2's reference is size-matched: plug-in KL on small histograms is biased up by ~(K − 1)/2 · (1/n₁ + 1/n₂), so an unmatched split-half reference (~100k ratings per half) would fail even a perfect simulator.

### 7.6 Calibration

1. **Post-hoc.** Calibration changes only the distribution each 1-token decision is sampled from. P(watch)/P(click): Platt scaling on `yes`/`no` log-odds. Ratings: temperature + per-class bias, 6 parameters, 5 free (softmax ignores a common shift); full vector scaling is an ablation. Fitted by negative log-likelihood (NLL) on calibration users' taste and probe calls.
2. **bf16 LoRA** of Qwen3.5-0.8B (~3 GB) or 2B (~5 GB per Unsloth); hardware in §8. Free Colab/Kaggle T4 or P100 GPUs lack bf16, so runs there are fp16, labelled. M1-masked inputs; loss on decision tokens and JSON; 10–20k examples of ≤ 2k tokens; one overnight run. Missing or slow Gated DeltaNet kernels on Windows/MPS switch the student to Qwen3-1.7B.

**Regression gate:** a MovieLens-calibrated model loses ≤ 0.02 KuaiRand click AUC.

**Two tiers.** A LoRA trained on real ratings is partly a supervised rating predictor, so G2/G3 gains alone prove little; before/after tables include a non-LLM rating-model simulator (MF-predicted rating + noise calibrated on calibration users).

| Tier | Behaviours | Pass rule |
|---|---|---|
| 1: trained on | G2, G3, G5 | Non-inferior to the *uncalibrated* 9B and the rating model: G3 ratio within 0.05, G2 KL within 10%, G5 ratio within 0.05 |
| 2: unseen by the LoRA | G4 at M1; hype-vs-neutral framing Δ; KuaiRand click AUC transfer | AUC at most 0.02 below the uncalibrated 4B; the rating model ignores text, so has no answer here |

Exit page and satisfaction lack MovieLens ground truth: reported only.

### 7.7 Analyst Agent

| Element | Specification |
|---|---|
| Model | Qwen3.5-9B (Mac) or 4B (Windows), thinking on, OpenAI-style tool calls; own `llama-server` (`-c 32768 -np 1`) while the population server is stopped |
| Observation | Study manifest, read-only `gates.yaml`, tool results |
| Tools | `arm_metrics(arm, metric, segment)` (bootstrap 95% CI, 2,000 resamples); `paired_diff(a, b, metric)` (CRN-paired, Holm); `run_gates()`; `segment_breakdown()`; `fetch_examples()` (≤ 5 transcripts); `compute(expr)`; `publish_readout(md)`; `refuse_readout(reason, gates)` |
| Hard rules | `publish_readout` raises unless every blocking gate passes for this population hash **and** every narrative number matches a tool output at displayed precision. Derived numbers (ratios, percentages, differences) come from `compute`; code renders tables |
| Termination | Publish or refuse; 20 tool calls / 30k tokens, then automatic refusal |

The decision is code; the agent investigates and explains. **Evaluation** on cached run-store data (no new simulation): 20 clean and 20 corrupted studies. Six planted faults pass every gate and show only through tools: swapped arm label; persona segment duplicated from another; cached-output reuse (identical feelings across users); one arm serving stale pages; seed collision breaking CRN in one segment; calibration file from another population hash. Two are gate-visible: over-agreeable population; corrupted persona segment. Reported, with exact Clopper–Pearson CIs: recall on gate-invisible faults and on gate-visible faults, false refusals on clean studies, number-checker rejections. A gate-only script misses gate-invisible faults by construction; no superiority test at this n.

### 7.8 Model Card and Drift

Keyed by sha256 of the GGUF, quantization, prompt templates, calibration parameters and `llama-server` build; any change makes a new card that must re-pass the gates. It states intended use (pre-screening only), validated segments, gate values with CIs, the memorization delta and known failure modes. **Drift alerts:** a gate margin below 25% of its threshold distance (|baseline value − threshold|), or a persona trait's PSI > 0.2.

## 8. Hardware Profiles

**Rule: the weakest machine sets the defaults.** The `portable` profile must run on the 6 GB card *and* the Mac.

| | `portable` (default) | `accelerated` (opt-in) |
|---|---|---|
| Population | Qwen3.5-4B Q4_K_M, 4 slots × 6k (`-c 24576 -np 4`) | 8 × 6k on CUDA; Qwen3.5-9B or Gemma 4 E4B on the Mac |
| Thinking | off | off (on for the analyst and the thinking rung) |
| LoRA student | Qwen3.5-0.8B bf16 (~3 GB, server stopped) | Qwen3.5-2B (~5 GB): bf16 on the RTX 3060 if it fits beside Windows, else an 8 GB card; fp16 on a free T4 |

**Memory budget** (from published configs; S0 confirms against llama.cpp's allocation log). Slots are 6k because the hybrid KV is small and the append-only prompt stays short. Estimates (S0 measures): system ~400 tokens (prefilled once per slot if a checkpoint exists at its end, else per session), persona ~300 (prefilled once per session), page turn ~520 (5 memories ~100 + 4 cards ~90 each), ~160 generated per page. Page 5 starts at ≈ 3.9k (~700 + 4 × ~680 + ~520) and the interview ends at ~4.3k; new prefill falls from ~27% of the prompt (page 2) to ~13% (page 5), reported, not gated.

| Config | Weights | KV (f16) | DeltaNet state | Verdict |
|---|---|---|---|---|
| Win 6 GB · Qwen3.5-4B · 4 × 6k | 2.74 GB | 8 layers × 4 KV heads × 256 × 2 × 2 B = 32 KiB/token → 0.75 GiB | 24 × 32 × 128 × 128 × 4 B ≈ 48 MiB/slot → 0.19 GiB | Expected fit; buffers measured |
| Win 6 GB · Qwen3.5-4B · 8 × 6k | 2.74 GB | 1.5 GiB | 0.38 GiB | Tight; buffers measured |
| Win 6 GB · Qwen3-4B-Instruct-2507 · 4 × 6k | 2.5 GB | 144 KiB/token → 3.4 GiB (q8_0 ≈ 1.8) | — | q8_0 KV only; tight, else 3 slots |
| Mac · Qwen3.5-9B · 4 × 6k | 5.68 GB | 0.75 GiB | 0.19 GiB | Fits |
| Mac · Gemma 4 E4B · 4 × 6k | 4.98 GB | small (sliding window) | — | Fits |
| Analyst · 9B (Mac) / 4B (Win) · 1 × 32k | 5.68 / 2.74 GB | 1.0 GiB | 48 MiB | Fits, population server stopped |
| Thinking rung · 4B · 2 × 16k | 2.74 GB | 1.0 GiB | 0.09 GiB | Expected fit |

**Mac limits.** macOS caps the GPU working set below physical RAM; S0 records llama.cpp's `recommendedMaxWorkingSetSize`. We never raise it with `sysctl` or co-host two population models. Work splits as in the budget table below (the Mac also hosts the analyst and demo); artifacts cross as files.

**Week-2 go/no-go gate** (session 3): a 50-user portable pilot on both machines, Random vs. Pop (no training needed), measuring sessions/h, p50/p95 page latency, tokens re-processed per call, JSON validity and a first σ_d. **Go** only if:
- **Throughput:** projected 300-user arm **≤ 2 h on Windows, ≤ 4 h on the Mac**.
- **Cache:** on pages 2–5, **≥ 95% of calls re-process ≤ their new tokens + 32** (including 1-token calls whose appended token differs from the generated one), and no "forcing full prompt re-processing" line in the server log.
- **Validity:** JSON validity ≥ 98%.

**No-go responses, in order:** (1) cache fails: Qwen3-4B-Instruct-2507 with q8_0 KV, whose standard attention truncates to any prefix; (2) page cap 3, interview on a 20% subsample; (3) a shorter persona; (4) a Qwen3.5-2B population, with the 4B on a 100-user fidelity subset.

**Compute budget**, projected at go-threshold rates (Windows 4B 150 sessions/h, Mac 4B 75, Mac 9B/Gemma budgeted at 37.5 until measured). A taste or probe call is budgeted at 0.5 session-equivalents (SE) until S0 measures it. Capacity is ~30 GPU-h per machine per week (6 h × 5 nights, weekends in reserve). Job sizes are defined in §5, §7 and §9; cells give SE and (GPU-h):

| Sprint (capacity per machine) | Windows | Mac |
|---|---|---|
| S0 (60 h) | Pilot + independent-seed CRN check, 150 SE (1) | Same pilot (2) |
| S1 (120 h) | Taste 4B, 2,500 SE (16.7) · ablation rungs 1–6, thinking rung at 3× time, 2,100 SE (~19) · σ_d re-estimate, 600 SE (4) · port delta, 200 SE (1.3) · RecBole arms, 6 + 5 on 2 datasets, 1,000 epochs, + optional Gowalla (≤ 12, to measure; runs into S2). **~53** | Taste 9B and Gemma, 1,500 SE (40) · 9B persona summaries, 6,040 MovieLens + ~1,500 KuaiRand users (≤ 10, budgeted). **50** |
| S2 (60 h) | MovieLens A/B, 1 seed, 3,052–6,867 SE (20–46) · KuaiRand arm study, 2,500 SE (16.7). **37–63** | KuaiRand item-level G6, 1,000 test users × ≤ 20 random events, 1,000 SE (13.3) |
| S3 (60 h) | Calibration fit + evaluation, post-hoc and LoRA, per model (probes, taste, KuaiRand item-level), 4,200 SE (28) · LoRA training (≤ 8). **36** | Uncalibrated 9B probe, 300 SE (8) |
| S4 (~45 h) | Second A/B seed, final readout only (20–46) · determinism re-run, 100 SE (0.7). **21–47** | Analyst evaluation, 40 cached studies (≤ 8, budgeted) |

S2 and S4 on Windows overrun when σ_d ≈ 0.9. **Pre-stated cut order:** (1) second A/B seed; (2) thinking rung and M2 down to 100 users; (3) cross-family taste cohorts down to 300 users; (4) KuaiRand arm study down to 300 users; (5) A/B n capped at what fits in S2, reporting the achieved minimum detectable effect (MDE).

## 9. Evaluation Plan

Each claim (§1) is scored against its named baselines. The simulated A/B also reports P̄view, P̄like, N̄exit, S̄sat and CRN diffs vs. MF, beside Agent4Rec's and RecBole's orderings (reported only). Cost (sessions/h, GPU-h, re-processed tokens per call, VRAM peak) is reported per machine and profile against the §8 budget.

**Ablation ladder** (300-user probe + 200-user MF arm per rung): base Agent4Rec prompt (the port) → stepwise constrained actions (identity calibration) → remove quality/popularity buckets → memory retrieval → traits block → thinking on → post-hoc calibration → LoRA student.

**Pre-registered decision rules** (power 0.8; a single two-sided test needs z(0.975) + z(0.8) = 2.80):

| Decision | Rule |
|---|---|
| Arm differences (primary) | Target MDE **0.10 satisfaction points** (twice Agent4Rec's MF–LightGCN gap, about a quarter of Pop–MF). 6 tests (each arm vs. MF), Holm, family-wise α = 0.05 two-sided. Power at Holm's first step, Bonferroni α/6 (z = 2.64): n = ((2.64 + 0.84) · σ_d / MDE)², rounded up: **436 per arm at σ_d = 0.6, 981 at 0.9**. First σ_d from the session-3 pilot; the S1 re-estimate (MF vs. LightGCN, 300 users) sets n. Cohort = max(300, n), capped by the §8 budget; if the cap binds, report the achieved MDE, not a claim |
| Taste alignment | 1,000 users for the 4B ({1:1, 1:9} × {M0, M1}, plus M2 at 1:9); 500 per cross-family model (1:1 M1, 1:9 M0, 1:9 M1). Non-inferior on 1:1 accuracy and 1:9 F1 if, at every ratio, the lower 95% bound is ≥ published − 0.03 and above the trivial floor (0.5; 0.182) and the MF-score classifier. 1:1 F1 is reported, not claimed ("watch all" scores 0.667) |
| Calibration | R's CI moves at least halfway to 1.0; G2 KL drops ≥ 30%; 1:9 precision drops ≤ 0.02; tier margins per §7.6 |
| Kendall τ, KuaiRand (5 arms) | A random ordering reaches τ ≥ 0.6 with probability 14/120 ≈ 0.12, so no fixed target: τ is reported with its exact permutation p-value, against τ(offline, real) and τ(click model, real). τ-b over pairs whose real replay CI excludes zero, count shown; tested only if ≥ 6 of 10 pairs resolve, else "untestable at this power" |
| Kendall τ, MovieLens (6 arms vs. RecBole) | Reported only; RecBole CIs from a user-level bootstrap of Recall@20; pairs with a RecBole or simulated CI including zero are excluded |
| Replay power | A top-20 matches 20/7,583 ≈ 0.26% of random events: ≈ 1.25k events per arm from test users (40%; 95% CI ±0.028 on click rate at worst-case p = 0.5), ≈ 1.9k from calibration + test users (60%; ±0.023), which the real side uses. LoRA-train users are excluded only if the KuaiRand LoRA variant trains on their random exposures. Arm pairs closer than ~0.03 (60% of users) to ~0.04 (40%) in click rate should stay unresolved |

**Honest reporting:** failed gates are results; every readout states n, seeds, population hash and achieved MDE; trivial floors, the MF classifier and masked results sit beside every taste number; MIND is "logged," never "randomized." The only cross-protocol comparison is the Agent4Rec anchor, labelled "published, different harness" and shown with the port delta.

## 10. Milestones (19 hours / 15 sessions)

| Sprint | Sessions | Deliverable |
|---|---|---|
| **S0** | 1–3 | Envs (`sim`, `arms`, `train`, `a4r-orig`); pinned `llama-server` on both machines; Agent4Rec port; per-call cache test, Qwen3.5-4B vs. Qwen3-4B-Instruct-2507; Random/Pop pilot; dataset decision; Metal cap; **week-2 go/no-go (session 3)** |
| **S1** | 4–8 | Personas; probe and clickbait sets; **freeze `gates.yaml` before any test-user run**; port delta; RecBole arms (+ optional Gowalla); taste alignment (4B M0/M1/M2, 9B, Gemma); σ_d and power; ablation rungs 1–6; MLX-LM check; **week-6 demo** |
| **S2** | 9–10 | 7 arms + offline metrics; MovieLens A/B with CRN, 1 seed; KuaiRand arms, click model, arm study, item-level G6, replay; Kendall τ |
| **S3** | 11–12 | Post-hoc calibration, then LoRA; G1–G7 in CI; before/after table with rating-model floor; uncalibrated 9B probe; KuaiRand regression |
| **S4** | 13–14 | Orchestrator hardening; analyst evaluation; model card, drift, dashboard; determinism re-run; second A/B seed if budget allows |
| — | 15 | Interview sprint: résumé numbers, 2-minute demo, design and experimentation drills |

## 11. Risks & Mitigations

| Risk | Mitigation |
|---|---|
| MovieLens memorization | M1/M2 masking from week 1 (§7.3); KuaiRand check; delta reported |
| Circular validation | Real random exposures primary; MIND labelled honestly (§5.2) |
| KuaiRand captions sparse or exposure not uniform | S0 coverage and χ² checks; IPS; MIND fallback |
| Replay CIs too wide | τ-b over resolved pairs; 60% of users on the real side; item-level AUC; "untestable" is reportable (§9) |
| DeltaNet prefix-cache misses; 6 GB throughput | Append-only prompt; pinned build; S0 cache test and go/no-go ladder, Qwen3-4B-Instruct-2507 first (§8) |
| Compute overrun | §8 cut order; achieved MDE reported |
| LoRA learns ratings, not behaviour | M1-masked inputs; rating-model floor; Tier 2 on unseen behaviours (§7.6) |
| LoRA does not fit or kernels missing | 0.8B student; Colab fp16; Qwen3-1.7B fallback |
| Chinese captions, English prompts | Bilingual persona summaries; per-segment fidelity; MIND cross-check |
| Anchor model retired (`gpt-3.5-turbo`, 2026-10-23) | Port delta on a local endpoint (§5.3); "different harness" label |
| Agent4Rec code rot; TAU-USI terms | Port logic, not environment (`a4r-orig` only for the delta); TAU-USI evaluation-only |
| "Generative agents are not tool agents" | The analyst's tool loop, refusal rules and planted-fault evaluation are the agentic core |

## 12. Résumé Bullets (experience voice; numbers filled in as sprints land)

Each bullet depends on its tagged claim; a refuted claim's bullet is reworded to the measured result, never silently deleted.

- (Claim 4) Built an **offline A/B testing platform for recommenders** on LLM user-simulator agents (Qwen3.5-4B, llama.cpp): a **7-arm, X-user-per-arm study in X GPU-hours on a 6 GB laptop GPU**, zero inference cost.
- (Claims 1–2) **Tested simulated arm orderings against real randomized exposures** (KuaiRand replay; **Kendall τ = X** vs. X for offline metrics and X for a click-model baseline; X of 10 pairs resolved). Measured Agent4Rec-protocol taste alignment (**1:9 F1 X vs. published 0.323**) and an **X-point memorization effect** via title masking.
- (Claim 3) **Measured synthetic-user agreeableness** before and after logit calibration and a **bf16 LoRA on real rating histories**, against a non-LLM rating-model floor (like-rate inflation X → Y; rating KL change X%).
- Turned fidelity into **blocking CI gates** plus a tool-using **analyst agent that publishes or refuses readouts** (X of Y gate-invisible planted faults caught; X of 20 clean studies refused).
- (Claim 4) Measured the variance reduction from **paired common random numbers** (X%) under a pre-registered power analysis (MDE X); **profiled llama.cpp prefix caching on hybrid DeltaNet models** (X re-processed tokens per call; X sessions/h).

### Placement Note

These bullets go under EXPERIENCE **only in an entry naming a real organization and a real engagement**: (1) the mentoring organization as employer, with a truthful title (`ML Engineering Practicum`) backed by an engagement record; (2) a real collaboration with a startup, lab or open-source org that agrees to host the work; or (3) an open-source project entry with real public commit history. Never invent an employer or dates: North American employers verify employment, and a fabricated entry can get an offer rescinded.

## 13. Why This Reads as Real Company Work

- **It attacks the offline–online gap** every recsys team has, governing the simulator like a model (§1).
- **It knows when to say no:** refusing a readout on failed fidelity is a product decision; demo simulators always return a number.
- **The statistics are the job:** CRN, power, Holm, bootstrap CIs and replay are daily experimentation-platform work, judged by real randomized exposures, not another model's opinion.

## 14. Interview Leverage

- **ML system design and experimentation:** offline vs. online evaluation, exposure bias, metric gaming; paired designs, MDE, multiple testing, replay/IPS, non-inferiority, with the project's own numbers.
- **LLM / agent engineering and ML infra:** structured outputs, persona memory, a tool-using analyst with hard preconditions, logprob calibration, behaviour LoRA; slot pinning, prefix caching, KV and recurrent-state arithmetic on 6 GB, batching nondeterminism.
- **Coding and behavioral:** bootstrap, Kendall τ, replay and power code; freezing thresholds before test data; refusing a readout under pressure.

## 15. References

- Agent4Rec: https://arxiv.org/html/2310.10108v3 · https://github.com/LehengTHU/Agent4Rec
- OpenAI model deprecations (gpt-3.5-turbo shutdown 2026-10-23): https://developers.openai.com/api/docs/deprecations
- Sim2Real: https://arxiv.org/abs/2603.11245 · TAU-USI: https://huggingface.co/datasets/cmu-lti/tau-usi
- MovieLens memorization: https://arxiv.org/abs/2505.10212
- KuaiRand: https://kuairand.com/ · https://github.com/chongminggao/KuaiRand
- Open Bandit: https://github.com/st-tech/zr-obp · https://arxiv.org/abs/2008.07146
- MIND: https://msnews.github.io/ · MovieLens-1M: https://grouplens.org/datasets/movielens/1m/
- KuaiSim: https://arxiv.org/abs/2309.12645 · Replay estimator (Li et al., WSDM 2011): https://arxiv.org/abs/1003.5956 · Model Cards: https://arxiv.org/abs/1810.03993
- Related LLM user simulators: RecAgent https://arxiv.org/abs/2306.02552 · LLM-Powered User Simulator (AAAI 2025) https://arxiv.org/abs/2412.16984
- RecBole: https://github.com/RUCAIBox/RecBole · RecBole #2215 (NumPy 2.x / PyTorch ≥ 2.6): https://github.com/RUCAIBox/RecBole/issues/2215 · LightGCN-PyTorch: https://github.com/gusye1234/LightGCN-PyTorch
- Qwen3.5: https://huggingface.co/Qwen/Qwen3.5-4B · https://huggingface.co/Qwen/Qwen3.5-9B · https://huggingface.co/Qwen/Qwen3.5-2B · https://huggingface.co/Qwen/Qwen3.5-0.8B
- Gemma 4 E4B: https://huggingface.co/google/gemma-4-E4B-it · Qwen3-4B-Instruct-2507: https://huggingface.co/Qwen/Qwen3-4B-Instruct-2507
- GGUF sizes: https://huggingface.co/unsloth/Qwen3.5-4B-GGUF · https://huggingface.co/unsloth/Qwen3.5-9B-GGUF · https://huggingface.co/unsloth/gemma-4-E4B-it-GGUF · https://huggingface.co/unsloth/Qwen3-4B-Instruct-2507-GGUF
- Unsloth Qwen3.5 fine-tuning: https://unsloth.ai/docs/models/qwen3.5/fine-tune
- llama.cpp: https://github.com/ggml-org/llama.cpp/issues/22384 · https://github.com/ggml-org/llama.cpp/issues/24587 · https://github.com/ggml-org/llama.cpp/issues/24714
