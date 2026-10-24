# Half-Life — Design Document

**Survival Analysis of Long-Horizon Agent Failure: Exact Dead-State Labels, Hazard Shapes per Model, and Test-Time Tricks Priced as Hazard Ratios per 1k Tokens**

| | |
|---|---|
| Status | Proposal |
| Target duration | ~10–12 weeks (19 mentoring hours / 15 sessions + independent work) |
| Platforms | Windows 11 laptop (16 GB RAM, NVIDIA RTX 3060-class, **6 GB VRAM default**, 8 GB if confirmed) and Apple Silicon Mac (16 GB unified, no CUDA), one repository |
| Deployment | 100% local. No rented servers, no paid GPU cloud, no training; survival fits run on CPU. No commercial API is needed; any optional comparison is capped at $20 total. Free Colab/Kaggle GPUs only as optional accelerators |
| Core stack | llama.cpp `llama-server` (one pinned build, CUDA and Metal) · Qwen3-1.7B / 4B / 8B, Qwen3-4B-Instruct-2507, Qwen3.5-2B / 4B / 9B, Gemma 4 E4B, all GGUF Q4_K_M (Apache-2.0) · **Ledger** (project-built seeded tool gym) · Long-Horizon-Execution (Sinha et al.) · BFCL v4 multi-turn (`bfcl-eval`, pinned) · lifelines 0.30.x · statsmodels · SciPy · SQLite + Parquet · `uv` |

---

## 1. The Product Thesis

Turn 37 of a bookkeeping run: a 4B agent pays invoice 42 twice. The overpayment becomes non-refundable vendor credit; the books can never balance again. pass@1 cannot say *how long an agent runs before such a call, or what each fix buys*. Ord and Hamilton fit hazards across tasks to METR's human-minute horizons (Hamilton: Weibull *k* < 1 for every model). Sinha et al. (ICLR 2026) see per-step accuracy degrade within a run as models *self-condition* on their own errors. For stateful tool use, nobody has directly observed the call after which the goal is unreachable.

**Half-Life builds a gym where that call is known exactly, then does reliability engineering on it.** Ledger is a seeded bookkeeping world (accounts, invoices, shipments, files) with irreversible tools. A closed-form reachability check, verified by brute-force breadth-first search (BFS), labels each tool call alive or DEAD and each turn ON- or OFF-plan. Episodes become time-to-event records: format failures and the context cap compete, and censoring is administrative at a fixed turn *t\**. Claims C1–C4:

1. **Hazard shape per model**: Weibull *k* (95% interval) for 8 local configurations (§4.1); a likelihood-ratio (LR) test of *k* = 1; pooled vs within-instance *k*, separating a real decreasing hazard from mixed task difficulty (§7.6).
2. **Own recovered tool errors, randomized**: the model's own pilot failures, each followed by its correction, are spliced into an otherwise identical, length-matched history; stratified Cox hazard ratio (HR) of later DEAD, 30% vs 0% spliced errors (§7.5).
3. **Price list for test-time tricks**: summarize-and-reset, verify-before-commit, capped thinking, per-step best-of-4 and harness-side state echo, each as an HR, a half-life change and a *yield*: log-hazard reduction per 1k decode-equivalent tokens per turn (§7.4).
4. **External validity, pre-registered to fail**: Ledger survival at BFCL's typical length rank-predicts BFCL v4 `multi_turn_base` accuracy across the 8 configurations (one-sided exact Spearman test, n = 8; supported iff Σd² ≤ 30, ρ ≥ 0.643; §7.7).

**Why it is fun.** Mean time between failures (MTBF) for agents: a half-life per model, a token price per trick, decay curves with confidence bands. Headline: *"Self-verify cut my 4B agent's DEAD hazard by X% (half-life +Y turns) for Z extra tokens per turn; thinking cut it X′% at W× the tokens."*

## 2. Goals & Non-Goals

### Goals
- **Ledger v1**: seeded tool gym with exact dead-state labels (§5.1, §7.1–7.2); frozen at session 4, then released (§10).
- **Survival pipeline** (§7.6): person-period event store; Kaplan–Meier (KM), Aalen–Johansen cumulative incidence, grouped Weibull, discrete-time complementary log-log (cloglog) hazards, stratified and time-varying Cox, accelerated failure time (AFT), restricted mean survival time (RMST).
- **C1–C4** with pre-registered n and analysis; **S0**: Sinha et al. Figure-4 reproduction, time-boxed to session 4 (§10).

### Non-Goals

| Out of scope | Owner |
|---|---|
| Training; process reward models (PRMs); best-of-4 here is a majority vote | 01 Compass |
| Harness and agent-computer interface (ACI) sensitivity; distillation | 02 Shellwright |
| Context-management design (summarize-and-reset is only *priced* here) | 03 Spelunker |
| Finding the step that broke *one* run (replay, forks); Half-Life models the *population* hazard | 05 BlackBox |
| Preventing irreversible actions (shadow worlds, rollback) | 06 Ctrl-Z |
| Optimizing harness code against the price list | 07 HarnessForge |
| Refitting METR's human-minute data; leaderboard submissions; models above 9B | — |

Siblings: BFCL runs reuse 05's harness and OpenAI-compatible Qwen3.5 function-calling (FC) handler, else S3 builds a minimal one (≈ 1 session). Ledger episodes can be written as 05 cassettes; their exact labels are free ground truth for 05's fork-vs-judge attribution.

## 3. Alignment With Prior Experience

| Prior experience | Half-Life module |
|---|---|
| **Changsheng**: backtested a two-stage sector→stock strategy (excess return after costs, no look-ahead); AutoML agent | Price list: hazard reduction *net of token cost*; seeds fixed pre-run, never tuned on results |
| **DiDi**: backend; **A/B testing** | Randomized arms paired by world seed; Schoenfeld-sized minimum detectable effects (MDEs); Holm across arms |
| **Sina Weibo**: SQL ETL, standardized metrics, alerting | Person-period tables (one row per turn); fixed metric definitions (t½, RMST, yield); hazard-monitor alert (stretch) |
| **CAS**: binary-encoding retrieval; 20+ configs on Precision@K / latency / storage | 8-configuration ladder; half-life vs tokens frontier |
| **College Application Agent**: LangGraph Writer–Critic | Verify-before-commit is a Writer–Critic loop with a measured price |

**Interview one-liner:** *"DiDi taught me randomized arms and pre-registered MDEs; Changsheng, judging a strategy after costs. Half-Life does both for agents: hazard ratio per token for each test-time trick."*

## 4. Stack Choices — and Why

### 4.1 Models

| Role | Model | Q4_K_M (unsloth) | KV cache, f16 |
|---|---|---|---|
| Ladder, dense; BFCL row | Qwen3-1.7B | 1.11 GB | 112 KiB/token (28 layers × 8 KV heads × 128) |
| Ladder; S0 text claim; C2/C3 replication | Qwen3-4B (thinking-capable) | 2.5 GB | 144 KiB/token |
| Ladder; BFCL row | Qwen3-4B-Instruct-2507 | 2.5 GB | 144 KiB/token |
| Ladder; S0 reproduction (Mac); BFCL row | Qwen3-8B | 5.03 GB (Q8_0 8.71) | 144 KiB/token |
| Ladder, hybrid-attention | Qwen3.5-2B | 1.28 GB | 12 KiB/token |
| Ladder; **C2, C3, ablations primary** | Qwen3.5-4B (4B LM) | 2.74 GB | 32 KiB/token |
| Ladder, hybrid-attention | Qwen3.5-9B | 5.68 GB | 32 KiB/token |
| Ladder, second vendor | Gemma 4 E4B (4.5B effective, 8B with embeddings) | 4.98 GB | measured in S0 |

**Why these.** Qwen3 vs Qwen3.5 (three Gated DeltaNet layers per full-attention layer, 05 §4.1) contrasts dense and hybrid attention at three sizes; recipes also differ, so the label is *family*, not *architecture*. Sinha et al. ran Qwen3 4/8/14/32B, not 1.7B; their headline Qwen3-32B (19.8 GB at Q4_K_M) does not fit, so S0 targets Qwen3-8B.

**One quantization family.** Q4_K_M on both machines, since a format change across the ladder would confound *k*. The brief's Qwen3.5-9B MLX 4-bit (`mlx-community/Qwen3.5-9B-4bit`, 5.95 GB) is only an optional `accelerated` throughput check, with a 50-episode parity test.

**Thinking and sampling.** Qwen: non-thinking (`chat_template_kwargs: {"enable_thinking": false}`), temperature 0.7, top_p 0.8 (Sinha et al.'s settings), top_k 20 (model cards), `presence_penalty` 0 (05 §7.5). The T arm changes only thinking, on a separate server with `--reasoning-budget 512` (N > 0 is a token budget per the server README), so its HR is not confounded with temperature. The cards' thinking settings (Qwen3 0.6 / 0.95, Qwen3.5 1.0 / 0.95) go unused. Gemma 4 E4B: card settings (1.0 / 0.95 / 64), thinking off. Settings print beside the *k* table.

### 4.2 Runtime

One pinned `bNNNN` `llama-server` build (prebuilt `win-cuda-12.4-x64`, `macos-arm64`, `--jinja`) serves OpenAI-compatible chat with native tool calls. `timings` give `prompt_n` (prompt tokens processed), `cache_n` (reused) and `predicted_n` (all generated, thinking included); `usage` lacks a reasoning count, so a *derived* split re-tokenizes `reasoning_content`. Bit-exact reruns are unneeded (05 §7.6): prompt cache on, `--parallel` per §8. Hybrid-attention (Qwen3.5) and sliding-window (Gemma 4) caches cannot roll back to an arbitrary prefix, so llama-server may re-process the whole prompt: the build pins `--ctx-checkpoints` (plus `--swa-full` for Gemma if memory allows), and a re-processed turn is charged its true `prompt_n`.

### 4.3 Libraries

| Concern | Choice | Why |
|---|---|---|
| Survival | `lifelines` 0.30.3 (MIT, Python ≥ 3.11): `KaplanMeierFitter`, `WeibullFitter.fit_interval_censoring`, `CoxPHFitter` (strata, cluster-robust), `CoxTimeVaryingFitter`, `AalenJohansenFitter`, `WeibullAFTFitter` | Mature, CPU-only |
| Discrete time | `statsmodels` GLM, binomial, cloglog link | Turns are discrete; cloglog = grouped proportional hazards (PH) |
| Cross-check | PyDTS (GPLv3, Journal of Open Source Software 2025) | Independent competing-risks fitter |
| Custom likelihood / store | SciPy `optimize`; SQLite + Parquet; `uv` | Instance-specific Weibull scale (§7.6); same commands on both OSes |

## 5. Benchmarks & Datasets

### 5.1 Ledger (project-built, headline)

Seeded generator plus reference executor (§7.1); seeds are fixed before any run, so no model has seen an instance. Default cell: H = *t\** plan items, one per turn (`exec` mode; *t\** ≈ 80, below the 16k cap, §7.3); dependency depth d = 1; 4 distractor tools; no observation noise; irreversible share ρ = 0.5. The release (§10 S4) covers every claim.

### 5.2 Long-Horizon-Execution (Sinha et al.; S0 replication)

An in-context dictionary of 100 five-letter keys, values in [−99, 99]; each turn supplies K keys and the model outputs the running sum, testing execution only. HF `arvindh75/Long-Horizon-Execution` (Apache-2.0): 100 rows of ~50k keys at K = 1. The code repo hard-imports vLLM and torch (`vllm>=0.9.0`, no Windows or Metal backend) and has **no license file** (2026-10-23). S0 runs a private patched fork of a pinned commit (vLLM removed, OpenAI-compatible client on llama-server; ≈ half a session); the release ships only a clean re-implementation over the Apache-2.0 data.

Published text-stated results (small-model numbers exist only in plots):

| Claim (v3) | Setting |
|---|---|
| Qwen3-32B task accuracy falls below 50% within 15 turns | Figure 4, Result 1; K = 1, direct answer, no CoT or thinking |
| First-step accuracy near-perfect for all models except Gemma3-4B and Qwen3-4B | same |
| Turn-100 accuracy degrades as induced history error rises | Figure 5, Result 3 |
| Qwen3 models with thinking do not self-condition | Figure 6, Result 5; 32,000-token thinking limit per turn (App. G.4) |
| Qwen3 non-thinking at temperature 0.7, top-p 0.8 (no top-k stated); 100 rollouts per Qwen3/Gemma3 model | Appendix G.4 |

**S0 target:** Qwen3-8B's Figure-4 horizon H₀.₅ (first turn with mean task accuracy < 0.5), digitized and labelled so; rule in §10.

### 5.3 BFCL v4 multi-turn (external validity, C4)

`multi_turn_base`: 200 state-checked entries (scoring: 05 §5.1). Ground truth (`possible_answer/BFCL_v4_multi_turn_base.json`, computed 2026-10-23): 2–10 calls per entry (median 5.5), 1–7 turns (median 4); bins 2–4 calls (52 entries), 5–7 (108), 8–10 (40). **BFCL tests only the first ~10 actions of a Ledger curve**; §7.7 sets the predictor to match.

Official leaderboard rows (CSV fetched 2026-10-23; MT = multi-turn) are sanity checks only; C4 uses local numbers. No Qwen3.5, thinking-capable Qwen3-4B or Gemma 4 rows exist; the 2507 row is Instruct-2507, not Qwen3-4B.

| Model | Overall MT | Base | Miss func | Miss param | Long ctx |
|---|---|---|---|---|---|
| Qwen3-8B (FC) | 41.75 | 50.50 | 42.00 | 40.00 | 34.50 |
| Qwen3-4B-Instruct-2507 (FC) | 22.12 | 26.50 | 21.00 | 15.50 | 25.50 |
| Qwen3-1.7B (FC) | 11.00 | 15.00 | 6.00 | 12.00 | 11.00 |

### 5.4 Reference-only work (not run) and positioning

| Work | What it measures | What Half-Life adds |
|---|---|---|
| METR (Kwa et al.); Ord (2505.05115; blog 2026-02-04); Hamilton | Success vs *human task minutes*, across tasks | Within-run steps, exact events, per-model *k* |
| Sinha et al. (2509.09677) | Running-sum execution; self-conditioning | Stateful tools, irreversibility, competing risks, matched splices |
| HORIZON (2604.11978) | Cross-domain long-horizon failure diagnosis; 3,100+ GPT-5 and Claude trajectories; LLM-judge attribution | Exact events; hazard shape; randomized priced arms |
| Bogdanov et al. (2609.29508); Manakina et al. (2609.29509) | Discrete-time hazards of *termination timing* in delayed-gratification games: 20-step multi-agent, 84,540 trajectories (29508); minute-by-minute ReAct, per-step tool budgets, 19,200 trajectories (29509) | Irrecoverable states, not a choice to stop; intervention HRs |
| Time-To-Inconsistency (2510.02712) | Cox/AFT/survival forest on first inconsistent answer, 36,951 turns | Tool use; randomized priced arms |
| Beyond pass@1 (2603.29231) | Reliability decay over 4 duration buckets, 23,392 episodes | Parametric shape; exact per-call labels |
| FuncBenchGen (2509.26553) | Controllable function-calling graphs; restating prior values lifted GPT-5 from 62.5% to 81.3% | Absorbing dead states; restatement becomes the echo arm |

## 6. Architecture

```
  ALL LOCAL: Windows + CUDA laptop (PC), Apple Silicon Mac, one repo

  LEDGER (Python, seeded)                EPISODE RUNNER                llama-server (pinned build)
  generator(seed, knobs) -> world  ----> turn loop, budgets --chat---> PC (CUDA): Qwen3-1.7B/4B, 2507,
  tools: transfer, pay_invoice, ship,    tool-call parse,  <-calls--            Qwen3.5-2B/4B
    read/write/delete_file,              schema checks               Mac (Metal): Qwen3-8B, Qwen3.5-9B,
    close_account, distractors           |                                        Gemma 4 E4B
  reference executor -> s*_1..s*_H       |- wrappers: reset_m | verify | think | best-of-4 | echo
  alive(s): closed-form invariants       '- splice injector (C2): length-matched prefix, e% own failed calls
    (== backward BFS on tiny worlds)             |
        '--> per-call alive | DEAD ; per-turn ON | OFF ; FORMAT ; QUIT ; CTX ; censor at t*
                                                 |
  EVENT STORE  episodes | person_period (1 row/turn) | transitions | calls (tokens, timings, seed)
                                                 |
  SURVIVAL (CPU)  KM, Aalen-Johansen | grouped Weibull k, lambda, t_half | cloglog discrete time
                  stratified + time-varying Cox | AFT, RMST | PH tests | bootstrap over instances
                                                 |
  OUTPUTS  k table (C1) | splice HR (C2) | price list (C3) | rank test (C4) | static HTML report
                                                 ^
  BFCL v4 multi_turn_base (05 harness): 8 configs x 200 entries   S0: Sinha Fig. 4, K = 1
```

The runner feeds one plan item per turn; the first DEAD is the event time. Wrappers change only what the model sees or how many samples it draws, never the world or labels.

## 7. Component Design

### 7.1 Ledger world

**State**: 3–5 accounts (integer balances, open/closed); 10–60 invoices (due, paid); orders (item, ordered, shipped); stock for 3 items; files in `work/` (writable) and `archive/` (read-only to `write_file`). Goal *G* = the reference executor's final state.

**Items and arrivals.** Each plan item instantiates a template: irreversible (share ρ) `pay` an invoice, `ship` an order, or sweep and `close` an account (2 : 2 : 1); reversible (1 − ρ) `transfer` funds or edit a `work/` file. A seeded arrival process, exogenous to the agent and seen only via read tools, replaces each consumed invoice, order, stock unit and account, and adds receipts into accounts open in *G*, so supply never runs out. Template mix, depth and open dead routes per item are thus stationary in *t* by construction; a pre-freeze test checks their shares in 20-turn bins.

| Tool | Effect | Reversible | Dead-state route |
|---|---|---|---|
| `list_accounts`, `get_balance`, `get_invoice`, `get_order`, `list_files`, `read_file` | read | — | none |
| `transfer(src, dst, amount)` | moves funds; overdraft or closed account → tool error | yes | none |
| `pay_invoice(i, a, amount)` | funds leave; overpayment kept as non-refundable vendor credit | no | overpaying |
| `ship(o, item, qty)` | stock leaves; over-shipment and wrong item accepted | no | over-shipping, wrong item |
| `write_file(p, text)` | overwrite in `work/`; error in `archive/` | yes | none |
| `delete_file(p)` | removes any file, `archive/` included | no | deleting an archive file |
| `close_account(a)` | only at zero balance; no reopening | no | closing an account open in *G* |
| Distractors (0 / 4 / 12): `pay_vendor_all`, `transfer_all`, `ship_rush` (+1 unit), `purge_dir`, deprecated lookalikes | honest docstrings | mostly no | when misused |

**Knobs**: depth d ∈ {0, 1, 2} (d chained reads reach an argument, e.g. a memo names the invoice whose line names the account); distractors {0, 4, 12}; observation noise ν ∈ {0, 0.3} (irrelevant fields, reformatted numbers, never a wrong value); irreversible share ρ ∈ {0.3, 0.5, 0.6}; plan mode `exec` (one item per turn, default), `upfront` (whole plan at turn 1) or `goal` (target state only; the agent plans and calls `submit()`).

### 7.2 Dead-state labelling (exact, BFS-verified)

`alive(s)` holds iff: (a) no invoice paid beyond its due, no payment to an invoice *G* leaves unpaid, no order over-shipped or shipped the wrong item; (b) Σ open balances + scheduled receipts − remaining dues = Σ *G* balances, where remaining dues are the unpaid amounts of the invoices (current or scheduled) that *G* pays, and every account open in *G* is open; (c) per item, stock + scheduled restock − remaining ordered (orders *G* ships) = *G* stock; (d) every `archive/` file exists unchanged. Transfers between open accounts reach any distribution with the same total, and any action can wait for later arrivals, so (a)–(d) suffice; all other tools are monotone, so they are necessary.

- **BFS verifier**: on tiny worlds (≤ 3 accounts, balances ≤ 6; ≤ 2 invoices; 1 order; 1 item, stock ≤ 3; 2 files; 2 arrivals; every integer argument enumerated), backward BFS from *G* over the time-expanded reachable graph must match `alive` on every state. **Freeze gate (session 4)**: 1,000 random tiny worlds, 0 disagreements.
- **Labels**: per call, alive or DEAD (exact). Per turn, ON iff the world at turn end equals *s\*ₜ*, else OFF (alive but off-plan, e.g. wrong paying account or reversible transfer). Per action (C4), a *deviation* is a state-changing call whose post-state lies on no ordering of the item's reference calls; *uncorrected* if its turn ends OFF. `goal` mode lacks per-item references, so it reports alive/DEAD only. Labels cost O(1) per call.

### 7.3 The agent loop

- **Observation**: system prompt ("tools change a real ledger; some actions cannot be undone"), tool schemas (OpenAI tools JSON), one user message per turn with the next plan item (`exec`), terse one-line JSON tool results. The agent is **never** told its label; a DEAD episode ends silently.
- **Actions**: one native tool call per step, or a one-word reply ending the turn.
- **Format failures**: unparseable call, unknown tool or schema-invalid arguments → tool-error observation, logged as a recurrent event; **3 in a row within a turn → FORMAT** (terminal).
- **Termination**: DEAD is the event. FORMAT, QUIT (`goal`-mode `submit()` while alive but ≠ *G*) and CTX (next request would exceed the cap) are pre-registered competing events, shares reported per arm. Censoring is administrative at *t\**, identical across arms and machines. A 3 h wall-clock hang guard logs a technical failure: excluded and counted, never censoring.
- **Budgets**: ≤ 6 tool calls per turn (a 7th ends the turn as a logged stall); ≤ 256 output tokens per call (T: 512 thinking + 256); context cap 16k (`portable`) or 32k (`accelerated`). Planning sizes, to be measured: schemas ≈ 1.5–2k tokens; a d = 1 turn (item, read + result, write + result, one-word reply) ≈ 120–160 tokens; control hits 16k near turn 90–120. **t\*** = largest turn with p95 context ≤ 0.9 × cap for the most verbose (configuration, arm) pair: planned 80, set at week 2, frozen after the session-8 pilot. Plans are generated item by item, so any *t\** is a prefix of the same plan.
- **Time scale**: primary = turn *t* (plan item), as in Sinha; secondary = action index (C4).

**Event store** (fits read only its Parquet and never re-run episodes): `episodes` (seed, model, GGUF sha256, build, arm, knobs, end cause, event turn); `person_period` (one row per turn at risk: state, own failed calls in context, tokens, cause indicators); `transitions`; `calls` (`prompt_n`, `cache_n`, `predicted_n`, derived reasoning tokens, `timings`, label, deviation flag). Ties are inherent: Cox uses Efron; parametric fits use grouped likelihoods.

### 7.4 Intervention wrappers (C3)

Wrappers change only what the model sees or how many samples it draws, never the world or labels.

| Arm | Mechanism | Extra calls |
|---|---|---|
| C, control | — | 0 |
| R25, summarize-and-reset | every 25 turns, a ≤ 150-token self-written state summary; context rebuilt as system + tools + summary + next item (R10 as ablation) | 1 per 25 turns |
| V, verify-before-commit | before each state-changing call, the same model sees item, proposed call and last 3 observations; APPROVE or one corrected call | 1 per state change |
| T, thinking capped at 512 tokens | thinking on, `--reasoning-budget 512`, C's sampling (§4.1); templates strip prior thinking | 0 (longer outputs) |
| B4, best-of-4 | 4 seeded samples; majority vote on the canonical call (tool, sorted args, normalized numbers); tie → lowest seed | 3 per action |
| E, state echo (harness) | appends exact current values of the accounts, invoices and orders the item names | 0 |

**Cost and yield.** Decode-equivalent tokens per turn = `predicted_n` + *c* × `prompt_n` (§4.2: cache hits free, re-processing charged); *c* = decode ÷ prefill tok/s per machine, measured at week 2. Δ tokens per turn = mean over person-periods in turns 1..*t\**; raw counts shown. **Yield = −ln HR ÷ (Δ tokens per turn / 1000)**, higher is better; its reciprocal is the price. An arm with HR < 1 and Δ ≤ 0 (reset can shorten contexts) is *dominant*, not priced.

### 7.5 Splice injector (C2)

- **Prefix**: turns 1–20 of a fresh instance, scripted correct, in the model's own chat template (assistant text from its own successful tuning-seed turns); the turn-20 world is ON and identical in every arm.
- **Dose** e ∈ {0, 0.1, 0.3}: in e × 20 random prefix turns, a failed call (wrong argument → tool error, no state change) precedes the correct one. Failed calls come, matched by tool, from the same model's own tuning-seed failures (pool hashed in `prereg.yaml`). Each is recovered within its turn, so C2 tests *own recovered* errors, not Sinha's uncorrected ones. Lower doses get equally long neutral reads (`get_balance`), so **prompt length matches within ±2%**.
- **Live** from turn 21, hazard counted from there; instances shared across doses (stratum = instance). Secondary splice B: a valid wrong reversible transfer plus its reversal (state again identical).
- **Observational companion**: "own failed calls in context" as a time-varying Cox covariate in control episodes.

### 7.6 Survival models

| Analysis | Specification | Decision rule |
|---|---|---|
| Nonparametric | KM per configuration (DEAD; other causes censored); Aalen–Johansen cumulative incidence of DEAD, FORMAT, QUIT | descriptive |
| Grouped Weibull (C1) | S(t) = exp(−(t/λ)^k); an event in turn t contributes S(t−1) − S(t) (`fit_interval_censoring` on (t−1, t]); t½ = λ(ln 2)^{1/k}; 95% intervals by bootstrap over instances | LR test vs exponential (*k* = 1) |
| Heterogeneity | 30 instances × 10 rollouts per configuration; pooled fit vs common *k* with instance-specific λᵢ (SciPy); mixed difficulty alone pushes pooled *k* below within-instance *k* | report both |
| No-history probe | pre-freeze, Qwen3.5-4B runs each item of 20 tuning instances once from *s\**ₜ₋₁ with an empty history | within-instance *k* ≠ 1 is credited to context growth or self-conditioning only if the probe's error rate has no trend in *t*; else that per-position rate is a cloglog covariate |
| Model check | piecewise-constant cloglog hazard (10-turn bins) vs Weibull, Akaike information criterion (AIC); overlaid hazards | Weibull worse by > 10 AIC → *k* is a summary only |
| C2, C3 | cause-specific Cox stratified by instance, cluster-robust standard errors; PH check by scaled Schoenfeld residuals; censoring arm-independent, CTX competing (§7.3) | PH fails (p < 0.01) → primary effect = RMST difference at *t\**; Weibull AFT with arm-specific shape shows whether the trick changes *k* |
| Multi-state | cloglog hazards per origin (ON, OFF); OFF→ON recovery share | descriptive; PyDTS on one configuration must agree with the GLM within intervals |

### 7.7 External-validity harness (C4)

- **Predictor (pre-registered)**: Ŝ_ON(6), each configuration's probability of no uncorrected deviation (§7.2) in its first 6 actions (BFCL's median 5.5 calls, rounded up) in the default cell. BFCL fails *any* wrong state, so the ON curve, not the alive curve, is the analogue.
- **Outcome**: local `multi_turn_base` accuracy, 200 entries per configuration, non-thinking.
- **Test**: Spearman ρ on midranks across 8 configurations, one-sided exact permutation test (ties permuted as midranks); **supported iff Σd² ≤ 30, i.e. ρ ≥ 0.643** (exact p = 0.048 without ties).
- **Baselines**: Ledger first-turn accuracy and effective parameter count (Gemma 4 E4B at 4.5B; midranks for the three 4B models), each with its own ρ. A baseline ρ ≥ the predictor's means the hazard model "adds nothing for BFCL", descriptively (n = 8 cannot test a difference of correlated ρs). Per-bin calibration is descriptive.

### 7.8 S0 replication harness

Sinha's `dict_sum` (K = 1, multi-turn, direct answer, their prompts and HF data) via the patched fork (§5.2): 100 rollouts per model, up to 60 turns or until all fail, scored by their definitions. Rollouts land in the person-period store, so the repo's first KM curve is Sinha's task as survival.

## 8. Hardware Profiles

**Rule: the weakest machine sets the defaults**: memory from the 6 GB card, speed from each machine's week-2 measurement.

- **Profiles**: `portable` (default): 16k cap, its *t\**, the 6 GB column. `accelerated` (opt-in): 32k cap with its own *t\** for hybrid-attention models and the Mac, more slots on a measured 8 GB card. Hazard tables never mix profiles.
- **Placement**: models as in §6; fits and BFS verifier on either CPU.

**Memory** (weights GiB; f16 KV from configs; compute buffers ~0.5 GiB, estimated; usable VRAM measured in S0, as the laptop may drive a display):

| Process | Weights | KV | 6 GB | 8 GB | Mac 16 GB |
|---|---|---|---|---|---|
| Qwen3-1.7B, 2 × 16k | 1.03 | 3.5 GiB | fits (~4.5 before buffers) | fits | fits |
| Qwen3-4B or 2507, 1 × 16k | 2.33 | 2.25 GiB | tight (~4.6 before buffers), measure | 2 slots | fits |
| Qwen3.5-2B, 4 × 16k | 1.19 | 0.75 GiB | fits | fits | fits |
| Qwen3.5-4B, 4 × 16k (+ ~48 MiB recurrent state per slot) | 2.55 | 2 GiB | fits per 05 §8 (~4.7 before buffers; headroom measured) | fits | fits |
| Qwen3-8B, 1 × 16k (Q8_0 S0 fallback: 8.11 + 0.56 at 4k) | 4.68 | 2.25 GiB | no | no | fits (~6.9; Q8_0 tight) |
| Qwen3.5-9B, 4 × 16k | 5.29 | 2 GiB | no | tight, 1 slot | fits (~7.3) |
| Gemma 4 E4B, 1 × 16k | 4.64 | measured | measure | measure | fits |

macOS wires only part of RAM for the GPU (commonly reported as about two-thirds on 16 GB); S0 measures it, changing no system settings. **Pre-registered fallbacks**: 4 Qwen3.5-4B slots do not fit → 3 slots (~4.2 GiB before buffers; throughput only, outputs unaffected), about a third more PC hours (≈ +15 h). Qwen3-4B/2507 does not fit at 16k → it moves to the Mac, so every configuration keeps one *t\**.

**Week-2 go/no-go** (session 2, runner v0; 40 control episodes per machine: Qwen3.5-4B on PC, 4 slots; Qwen3-8B on Mac): per-episode wall clock; tokens and `cache_n` / `prompt_n` per turn; prefill and decode tok/s (sets *c*); planning *t\** from p95 context (§7.3); S0 status; BFS verifier passing 100 tiny worlds. The compute ledger then re-prices at measured rates.

**Compute ledger** (core; planning s/episode at *t\** = 80, aggregate over slots: Qwen3.5-2B/4B 25, Qwen3-1.7B 30, Qwen3-4B/2507 60, Mac 8B/9B 120, Gemma 4 E4B 90; multipliers verify 1.8×, best-of-4 2.5×, thinking 10×; BFCL 60 s/entry PC, 120 s Mac). Effective arms survive longer; the pilot re-prices them.

| Item | Episodes | PC h | Mac h |
|---|---|---|---|
| S0: Sinha, Qwen3-4B + Qwen3-8B × 100 rollouts (+100 Q8_0 if needed) | 300 rollouts | 1.0 | 2.0 |
| No-history probe: Qwen3.5-4B, 20 tuning instances × 80 items | 1,600 single turns | 0.7 | |
| Variance pilot: Qwen3.5-4B, C, R25 and E, 40 instances each | 120 | 0.8 | |
| C1 ladder: 5 PC configurations × 300 | 1,500 | 16.7 | |
| C1 ladder: 3 Mac configurations × 300 | 900 | | 27.5 |
| C2: Qwen3.5-4B and Qwen3-4B, 3 doses × 140 | 840 | 9.9 | |
| C3: Qwen3.5-4B, 6 arms × 220 | 1,320 | 26.5 | |
| C3 replication: Qwen3-4B, C + 2 highest-yield non-T arms × 150 (priced at the worst case, V + B4) | 450 | 13.3 | |
| Ablations: Qwen3.5-4B, 10 cells × 150 (§9 ladder) | 1,500 | 10.4 | |
| C4: BFCL, 5 PC + 3 Mac configurations × 200 entries | 1,600 entries | 16.7 | 20.0 |
| **Core total** | **6,630 Ledger episodes** | **≈ 96** | **≈ 50** |

**Gate rules.** Budget: 150 unattended PC machine-hours and 80 Mac hours, confirmed by the mentee at S0. If the re-priced core exceeds either, **cuts apply in pre-registered order**: (1) C3 replication → C + 1 arm; (2) ablations 150 → 80 per cell, KM only; (3) thinking arm 220 → 150 (MDE recomputed); (4) C1 300 → 200 per configuration (*k* MDE ≈ 0.13 → 0.16); (5) drop the 10% splice dose. C4 keeps all 8 configurations and 200 entries (n = 6 would need Σd² ≤ 6, ρ ≥ 0.829). On a Mac overrun, Gemma 4 E4B moves to the PC if S0 memory allows.

## 9. Evaluation Plan

Two-sided tests at α = 0.05, power 0.8, unless stated. Analysis code and `prereg.yaml` (models, arms, n, primary contrasts, exclusions) are committed and hash-recorded before main runs; pilot episodes never enter confirmatory tests.

| Area | Metrics | Compared against | n and MDE |
|---|---|---|---|
| S0 | H₀.₅; accuracy vs turn | Sinha Fig. 4, digitized | 100 rollouts; rule in §10 |
| Labels | `alive` vs backward BFS | brute force | 1,000 tiny worlds; 0 disagreements |
| C1 shape | *k*, λ, t½, LR test, pooled vs within-instance *k*, AIC (§7.6); family effect | *k* = 1; Hamilton's direction (reference) | 300 per configuration, ≈ 270 events at a planning 90% event share by *t\** → \|k − 1\| ≈ 0.13 |
| Competing risks, multi-state | cause-specific hazards, cumulative incidence, ON↔OFF and →DEAD hazards (§7.6) | — | descriptive; Wald on cause-specific HRs |
| C2 | HR 30% vs 0% (§7.6 Cox); dose trend; PH | 0% splice | 140 per dose; D = 247 events → HR 1.43 |
| C3 | HR, RMST(*t\**), Δt½, Δ tokens/turn, yield; CTX share; shape change (AFT) | control | 220 per arm; Holm over 5, sized at α = 0.01: D = 367 → HR 0.70 |
| C3 replication | same, Qwen3-4B | control | 150 per arm; reported at observed D |
| C4 | Spearman ρ, Ŝ_ON(6) vs BFCL MT-base | first-turn accuracy; effective parameter count | n = 8; supported iff Σd² ≤ 30 (ρ ≥ 0.643) |
| Ablations | KM, *k*, HR per knob | default cell | 150 per cell; claims only where the HR interval excludes 1 |

**Sizing.** Weibull: Var(k̂) ≈ (6/π²)k²/d for complete data, so d ≈ 270 gives SE(k̂)/k ≈ 0.047; grouping and censoring inflate it, so the pilot's bootstrap SE replaces it. Cox: Schoenfeld D = 4(z₁₋α/₂ + z₀.₈)² / (ln HR)², 1:1 allocation. The session-8 pilot (§8) measures event share under C, R25 and E, per-arm p95 context (freezing *t\**), the instance-clustering design effect and PH diagnostics; n then freezes in `prereg.yaml`. An arm short of its D reports its MDE at the observed D: inconclusive, never "no effect".

**Ablation ladder** (Qwen3.5-4B): the §7.1 knobs in order (plan mode, distractors, noise, depth, ρ), 9 non-default cells plus R10 = the 10 budgeted cells; then C → E → R25 → V → B4 → T from C3, each rung against its neighbour.

**Pre-registered directional predictions** (reported either way): P1, non-thinking control *k* > 1 in `exec` mode (Sinha-style degradation, against Hamilton's between-task *k* < 1); P2, pooled *k* < within-instance *k*; P3, R25 moves *k* toward 1; P4, T has the lowest HR but the lowest yield (best hazard, worst value per token); P5, E has HR < 1.

**Reporting rules**: events from the world labeller, never the model's report; end causes tabulated per arm; digitized numbers labelled; C4 the only external claim; no comparison with an unrun model; build and GGUF hashes on every table; nulls and a failed C4 published.

## 10. Milestones (19 hours / 15 sessions)

| Sprint | Sessions | Deliverable |
|---|---|---|
| **S0** | 1–4 | Repo, `uv`, pinned build, GGUF hashes; **runner v0** (`exec`, exact labels, JSONL log) by session 2; Sinha replication; week-2 go/no-go (§8); spec → BFS verifier → stationarity test, no-history probe → **spec v1 frozen at session 4**; Gemma 4 E4B memory; `prereg.yaml` skeleton |
| **S1** | 5–7 | Hardening: event store, format checks, competing-risk logging; KM and Weibull fits; R25 and E wrappers. **Session-7 demo**: KM curves and *k* with intervals for 3 configurations (beside Sinha's curve as survival if S0 passed) |
| **S2** | 8–10 | Variance pilot → n and *t\** frozen; 8-configuration ladder (C1, heterogeneity, model check); splice injector and C2. Week-8 checkpoint (session 10): planned-tier cut |
| **S3** | 11–12 | V, T and B4 wrappers; C3 price list and frontier plot; Qwen3-4B replication; knob ablations; BFCL handler (or 05's) |
| **S4** | 13–14 | C4 runs and rank test; Ledger release (spec, generator, verifier, seeds, episode Parquet); static report |
| — | 15 | Interview sprint; write-up (§14) |

**S0 reproduction rule (pre-registered, time-boxed).** Qwen3-8B, Figure-4 setting (§5.2), Q4_K_M on the Mac, 100 rollouts. Mentor and mentee digitize H₀.₅ independently (WebPlotDigitizer); the target is their mean. **Pass**: target inside the local bootstrap 95% interval over rollouts, widened by ±1 turn. Text check: Qwen3-4B (PC) first-turn accuracy below Qwen3-8B's. **Fail**: re-run at Q8_0 to split quantization from harness. Still failing at session 4: publish the gap analysis, label S0 *not reproduced*, drop only the Sinha-comparison framing (session-7 overlay, any "replicates Sinha" wording). Ledger continues on a runtime validated independently by 05's S0 check (2507 BFCL `multi_turn_base` 26.50%, run here if 05 is absent).

**Tiers.**
- **Core** ships regardless: Ledger v1 and verifier, S0, C1, C2 (thinking off), C3 on Qwen3.5-4B, C4.
- **Planned**, cut in order at week 8: (1) C2 with thinking capped at 512 tokens (Qwen3.5-4B, 0% vs 30% × 140, ~19 PC h), testing budget-capped thinking, not Sinha's 32,000-token condition; (2) C3 replication on Qwen3-4B; (3) multi-state analysis beyond descriptive tables.
- **Stretch**: Qwen3-14B (9 GB, Mac) as a Sinha anchor; combined arms (R25 + E); an other-authored splice control (same error text attributed to the system); a hazard monitor alerting before DEAD; 05 cassettes.

## 11. Risks & Mitigations

| Risk | Mitigation |
|---|---|
| Format failures dominate small models; few DEAD events | FORMAT competing from day 1 (§7.3); pilot measures event shares before n freezes |
| Reachability predicate wrong in a corner case | BFS gate (§7.2); property tests per tool; post-freeze changes need a version bump and re-run |
| Ledger too easy (no deaths by *t\**) or too hard (turn-1 deaths) | Pre-freeze knob tuning to a median DEAD turn of 15–25 for Qwen3.5-4B (≈ 90% events by *t\** = 80 if *k* = 1), on tuning seeds disjoint from release seeds |
| Context cap ends episodes early, differently by arm | *t\** rule; CTX competing, shares per arm (§7.3) |
| Weibull fits badly (early format deaths, later drift) | AIC check demotes *k* to a summary (§7.6); competing risks separate the early mode |
| Pooled *k* < 1 is heterogeneity, or item difficulty drifts in *t* | Within-instance design; stationary arrivals; no-history probe (§7.1, §7.6) |
| PH fails for thinking or reset | RMST(*t\**) primary for that arm; AFT shape term (§7.6) |
| C4 fails, or BFCL's ≤ 10 calls cover only the curve's start | Pre-registered; C1–C3 stand alone; scope in §5.3 |
| Sinha replication fails; unlicensed code hard-imports vLLM | Session-4 time box, Q8_0 split (§10); private patched fork, clean re-implementation (§5.2) |
| Novelty vs §5.4 work | Exact irrecoverable-state events plus randomized, priced interventions |
| 6 GB card cannot hold Qwen3-4B at 16k or 4 Qwen3.5-4B slots; thinking cost explodes | S0 measures; fallbacks (§8); 512-token budget and cut (3) |

## 12. Résumé Bullets (experience voice; numbers filled in as sprints land)

- Built **Ledger**, a seeded tool-use gym of fresh, unseen instances with exact dead-state labels (closed-form reachability matched brute-force BFS on **X** worlds, 0 disagreements); released **X** seeds and episode-level data.
- Applied **survival analysis to LLM agents** (Kaplan–Meier, grouped Weibull, cause-specific and time-varying Cox, competing risks) over **X** episodes from 8 local models: Weibull *k* **X–X**, constant hazard rejected for **X** of 8, half-lives **X–X** turns.
- Tested **conditioning on own recovered tool errors** via randomized, length-matched history splices: 30% vs 0% own failed calls, **HR X [X, X]**.
- **Priced five test-time tricks** as hazard ratios per 1k tokens: best HR **X** (**arm**), best yield **X** (**arm**); thinking at **X×** control tokens.
- Pre-registered an **external-validity test** against BFCL v4 multi-turn across 8 configurations: **ρ = X** vs a 0.643 line (**supported / not supported**).

**Placement note.** These read as experience only under a real engagement: the mentoring organization (engagement record, truthful title), a real collaboration with a lab, startup or open-source org hosting the work, or an open-source project entry (the Ledger release) with public commits. Never invent an employer or dates: North American employers verify, and a fabricated entry can rescind an offer.

## 13. Why This Reads as Real Company Work

- **Agent platforms toggle mitigations under cost budgets.** "Turn on self-verify in production?" asks failures avoided per token; Half-Life answers with an interval.
- **Reliability vocabulary carries over**: MTBF, hazard, censoring and competing failure modes are how site reliability engineering (SRE) and hardware teams already talk.
- **Operational discipline transfers too**: frozen specs, pre-registration, a time-boxed reproduction gate, a cut-ordered compute ledger.

## 14. Interview Leverage

- **Applied science / evaluation**: Schoenfeld sizing, Holm, stratification, length-matched controls; censoring bias; KM vs Weibull vs Cox; cause-specific hazard vs cumulative incidence; cloglog = grouped PH; RMST when PH fails; how frailty makes a population hazard fall while every unit's rises.
- **Test-time compute and systems**: self-consistency vs verification vs thinking, priced in tokens; prefill vs decode cost; KV budgets (dense vs hybrid attention, §4.1); why recurrent and sliding-window caches cannot roll back.
- **Drills**: *"Success drops on long tasks: compounding error or something else?"* → §7.5, §7.6; *"Is self-verification worth it in production?"* → §7.4, §9; *"Design a contamination-resistant benchmark."* → §5.1, §7.2.
- **Write-up**: one post under the mentee's byline (session 15), working title *"What is the half-life of a 4B agent? Survival analysis on my laptop"*: hook; context; the work (the turn-37 double payment first, then curves, splice, price list); what I'd do differently (the pre-declared null or fallback: *k* as a summary when the Weibull loses, §7.6; a failed C4, §7.7); takeaways. Numbers trace to the results table; vendor and digitized ones labelled.

## 15. References

- Sinha et al., *The Illusion of Diminishing Returns: Measuring Long Horizon Execution in LLMs* (ICLR 2026): https://arxiv.org/abs/2509.09677 · https://arxiv.org/html/2509.09677v3 · https://github.com/long-horizon-execution/measuring-execution · https://huggingface.co/datasets/arvindh75/Long-Horizon-Execution
- METR-data hazards: Kwa et al.: https://arxiv.org/abs/2503.14499 · Ord, *Is there a Half-Life for the Success Rates of AI Agents?*: https://arxiv.org/abs/2505.05115 · https://www.tobyord.com/writing/half-life · Ord, *Hazard Rates for AI Agents Decline as a Task Goes On* (2026-02-04): https://www.tobyord.com/writing/hazard-rates-for-ai-agents-decline · Hamilton, *Peto's Paradox and the Future of AI Agents* (2026-01-23, updated 2026-02-03): https://gushamilton.github.io/lab/2026/01/23/petos-paradox-ai-agents/
- Survival analysis of LLM behaviour: https://arxiv.org/abs/2609.29508 · https://arxiv.org/abs/2609.29509 · Time-To-Inconsistency: https://arxiv.org/abs/2510.02712 · Beyond pass@1: https://arxiv.org/abs/2603.29231 · HORIZON: https://arxiv.org/abs/2604.11978 · FuncBenchGen: https://arxiv.org/abs/2509.26553
- BFCL: https://gorilla.cs.berkeley.edu/leaderboard.html · https://gorilla.cs.berkeley.edu/data_overall.csv · https://github.com/ShishirPatil/gorilla/tree/main/berkeley-function-call-leaderboard
- Models and runtime: https://huggingface.co/unsloth (Qwen3-{1.7B,4B,8B,14B,32B}-GGUF, Qwen3.5-{2B,4B,9B}-GGUF, gemma-4-E4B-it-GGUF) · https://huggingface.co/mlx-community/Qwen3.5-9B-4bit · https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md
- Statistics: https://pypi.org/project/lifelines/ · PyDTS: https://arxiv.org/abs/2204.05731 · https://www.statsmodels.org/stable/glm.html · Schoenfeld, *Sample-size formula for the proportional-hazards regression model*, Biometrics 39 (1983) · Royston & Parmar, restricted mean survival time, BMC Med Res Methodol 13:152 (2013)
- Siblings: [01 Compass](01_Compass_Design_Doc.md) · [02 Shellwright](02_Shellwright_Design_Doc.md) · [03 Spelunker](03_Spelunker_Design_Doc.md) · [05 BlackBox](05_BlackBox_Design_Doc.md) · [06 Ctrl-Z](06_Ctrl-Z_Design_Doc.md) · [07 HarnessForge](07_HarnessForge_Design_Doc.md)
