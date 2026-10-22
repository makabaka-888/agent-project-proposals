# BlackBox — Design Document

**A Flight Recorder for LLM Agents: Bit-Exact Replay, Counterfactual Forks That Find the Step That Broke a Run, and Regression CI With Stated Power**

| | |
|---|---|
| Status | Proposal |
| Target duration | ~10–12 weeks (19 mentoring hours / 15 sessions + independent work) |
| Platforms | Windows 11 laptop (16 GB RAM, NVIDIA RTX 3060-class, **6 GB VRAM default**, 8 GB if confirmed) and Apple Silicon Mac (16 GB unified, no CUDA), one repository |
| Deployment | 100% local. No rented servers, no paid GPU cloud. Replay CI runs on free GitHub-hosted runners because replay needs no model. Commercial APIs only as optional comparison baselines, capped at $20 total |
| Core stack | llama.cpp `llama-server` (one pinned build) · Qwen3.5-4B / 2B / 9B and Qwen3-4B-Instruct-2507 GGUF (Apache-2.0) · BFCL v4 multi-turn · the `tau2-bench` repo at tag `v1.0.1` (τ³-bench 1.x line) · OpenTelemetry (OTel) GenAI semantic conventions (semconv, pinned commit) · Arize Phoenix · SQLite + Parquet · statsmodels · GitHub Actions |

---

## 1. The Product Thesis

Agents fail inside long, stochastic trajectories. After a prompt edit, a model swap or a quantization drop (Q4_K_M → Q3_K_M, 4-bit to 3-bit llama.cpp k-quants), the evidence is usually one rerun and a score that moved three percentage points (pp). Worse, or just different? Which of twenty steps broke the run?

**BlackBox treats every agent run as a flight.** Every LLM call, tool call, environment read and user-simulator turn goes into a content-addressed cassette. A run replays exactly with no model loaded, or forks at any step with a resampled, stronger-model or fault-free action. Record/replay tools exist (§4.3); the thesis is behaviour reproducible enough to measure, so failures are found by intervention, `git bisect`-style ("step 7 broke it, here is the minimal cassette"), not by an LLM's opinion. Four measured claims:

1. **One GGUF model file at temperature 0 is not reproducible by default**: BlackBox measures divergent reruns, the first-divergence step and outcome flips under stock defaults, under the bulk configuration and per changed factor, on a PC and on a Mac (§7.6).
2. **Regression CI (continuous integration) with stated power**: paired tests sized by a power curve (N tasks × k runs → minimum detectable effect, MDE) and checked on planted regressions; three-valued verdicts; hosted replay avoids X% of live-model runs (§7.9).
3. **Counterfactual forks beat same-model LLM judges at attribution** on human-labelled natural failures and an unfiltered fault-injected BFCL (Berkeley Function-Calling Leaderboard) multi-turn set, at matched compute or with the cost ratio stated. One Qwen3.5-9B backs every judge, so the gap is interventional (§7.4, §7.7). A live-verified subset checks fork fidelity only; it is never evidence that forks beat judges.
4. **A local 9B investigator is scored on Who&When** against random and the same judges. The logs cannot be re-executed, so this is non-interventional and reported apart (§5.3; planned tier).

## 2. Goals & Non-Goals

### Goals
- **Recorder** (OpenTelemetry GenAI spans + content-addressed cassettes) with three replay modes (§7.1–7.2): a 20-step run replays in < 1 s; strict replay reproducing 100% of tool-call sequences and final state hashes is an acceptance test, not a claim (verify-tools also tests tool determinism).
- **Snapshots, forks, nondeterminism study** (headline; v1 in S0), fault-injection generator and dataset (released), investigator (planned), power-aware regression CI (§7.3–7.9).

### Non-Goals

| Out of scope | Owner |
|---|---|
| Training; models used as published | 02 Shellwright, 03 Spelunker |
| Browser, VM or OS snapshots | 04 Crosshair |
| Rollback, injection defence | 06 Ctrl-Z |
| Simulator validation (recorded and forked, not claimed realistic) | 08 SimCrowd |
| Step-value learning (p̂(k) is 01's Monte-Carlo process-reward estimator; here, diagnosis only) | 01 Compass |
| Gating harness-code edits (BlackBox gates model, quantization and prompt changes; both calibrate on A/A, identical-arm, runs) | 07 HarnessForge |
| Batch-invariant kernels (nondeterminism is measured, not fixed); hosted tracing; UI beyond Phoenix + static report; leaderboard submissions | — |

Siblings: 02 and 04 export traces as cassettes; 06 Ctrl-Z's episodes use them, and its shadow-world snapshots share the BFCL/τ² state adapter (§7.3). 09 memtidy records its agent runs as cassettes; its A/A floor uses the §7.9 power method.

## 3. Alignment With Prior Experience

| Prior experience | BlackBox module |
|---|---|
| **Changsheng**: backtested a two-stage sector→stock strategy (look-ahead bias); AutoML agent | Forks: point-in-time cassettes; no reads of step ≥ *k* |
| **DiDi**: backend; **A/B testing** | Regression CI + pre-registered MDE, A/A, power curve |
| **Sina Weibo**: SQL ETL, standardized metrics, alerting | Recorder, report; the CI gate is the alert |
| **CAS**: binary-encoding image retrieval; 20+ configs on Precision@K / latency / storage | Determinism grid; content-addressed keys |
| **College Application Agent**: LangGraph Writer–Critic | Investigator, judge baseline; SDK demo |

**Interview one-liner:** *"At Changsheng I learned a backtest is only honest if it can't see the future. BlackBox applies that to agents: record every call, replay it exactly, fork at step k without look-ahead, and gate model changes with DiDi's A/B-testing discipline plus power analysis."*

## 4. Stack Choices — and Why

### 4.1 Models

| Role | Model | Params | Quant / file | KV (key-value) cache, f16 |
|---|---|---|---|---|
| Agent under test; R1 quantization regression | Qwen3.5-4B | 4B LM (5B with unused vision encoder) | Q4_K_M 2.74 GB; Q3_K_M 2.29 GB | 32 KiB/token |
| Investigator, *stronger* intervention, τ² user sim | Qwen3.5-9B | 9B | Q4_K_M 5.68 GB | 32 KiB/token |
| Cheap resampler, challenger | Qwen3.5-2B | 2B | Q4_K_M 1.28 GB | 12 KiB/token |
| Pure-attention contrast; S0 reproduction | Qwen3-4B-Instruct-2507 ("2507") | 4B | Q4_K_M 2.5 GB; Q8_0 4.28 GB | 144 KiB/token |

**Why Qwen3.5.** Newest official small Qwen generation (0.8B–9B dense, 2 March 2026, Apache-2.0, 262k context). Three Gated DeltaNet (linear-attention) layers per full-attention layer leave a KV cache on only 8 of 32 layers: 8 × 2 × 4 KV heads × 256 × 2 bytes = 32 KiB/token, vs 144 KiB for 2507 (36 layers × 8 × 128). That affords 32k–64k contexts; the recurrent state may add nondeterminism, hence the pure-attention contrast.

**Thinking** is on by default; agent, judges and simulator send `enable_thinking: false` via `chat_template_kwargs`. The Mac 9B server's `--reasoning-budget 4096` (pre-registered; one ablation vs 0) binds only thinking requests. Mode and budget are logged per span; each is ablated once.

**Card numbers are not anchors.** The Qwen3.5-4B card's BFCL-V4 50.3 is overall, not multi-turn; its TAU2-Bench 79.9 (official setup plus airline fixes from the Claude Opus 4.5 system card; simulator unstated) predates τ³-bench 1.0.0's 75+ task fixes (18 March 2026). Neither compares to a local Q4 run. *Sizes: unsloth GGUF listings, 2026-10-21.*

### 4.2 Runtime

One pinned `bNNNN` `llama-server` build (prebuilt `win-cuda-12.4-x64`, `macos-arm64`; `--jinja`) serves one GGUF on CUDA and Metal. Its README warns that with `cache_prompt` (default on), logits are "not guaranteed to be bit-for-bit identical for different batch sizes". Study flags: `--parallel`, `--no-cache-prompt`, `--kv-unified`, `--ctx-checkpoints`. Stock defaults (cell C4): `--parallel` −1 (auto, unified KV), prompt cache on, `--cache-ram` 8192 MiB, `--cache-idle-slots` on, `--slot-prompt-similarity` 0.10, 32 checkpoints per slot ≥ 8192 tokens apart (`--checkpoint-min-step`).

### 4.3 Libraries

| Concern | Choice | Why |
|---|---|---|
| Benchmarks | `bfcl-eval` (pinned; `2026.3.23` today); `tau2-bench` tag `v1.0.1` | `--skip-server-setup` + `LOCAL_SERVER_ENDPOINT/PORT` target llama-server. τ² `main` is τ-bench 2.0.0 (October 2026, LLM-judged communication); `v1.0.1` keeps airline deterministic |
| Telemetry | `opentelemetry-sdk` + OTLP (the OpenTelemetry protocol); GenAI semconv pinned by commit behind one adapter | `open-telemetry/semantic-conventions-genai` has no tagged release; *Development* status |
| Viewer | Arize Phoenix (Elastic License 2.0, self-hosted) | Converts `gen_ai.*` spans to OpenInference on ingest since 15.10.0 |
| Store / stats / env | SQLite + sha256 blobs, Parquet; `statsmodels`, SciPy; `uv` | One file per study; same commands on both OSes |

**Prior art:** `vcrpy` (HTTP cassettes, strict modes), `agrepl` (man-in-the-middle record/replay for agents), `reeltime` (offline replay debugger), LangGraph time travel (checkpoint forks), AgenTracer (*offline* counterfactual replay and fault injection for training labels). BlackBox adds environment-state forks, *n* samples per fork with intervals, a determinism study and a power-sized CI gate.

## 5. Benchmarks & Datasets

### 5.1 BFCL v4 multi-turn (headline; deterministic state checks)

800 entries (200 each: base, missing-function, missing-parameter, long-context) on Python-object backends (file system, vehicle, trading, travel, messaging, Twitter, tickets, math). Pass = every turn's backend state matches ground truth and the ground-truth minimal call path is a subset of the model's calls. Web search (needs SerpAPI) is excluded.

Published multi-turn accuracy, % (leaderboard CSV, updated 2026-04-12; FC = native function calling, MT = multi-turn); local Qwen3.5 Q4 rows are measured in S1:

| Model | Overall MT | Base | Miss func | Miss param | Long ctx |
|---|---|---|---|---|---|
| xLAM-2-8b-fc-r (FC; CC-BY-NC, reference only) | 70.00 | 76.00 | 72.00 | 65.00 | 67.00 |
| Qwen3-8B (FC) | 41.75 | 50.50 | 42.00 | 40.00 | 34.50 |
| **Qwen3-4B-Instruct-2507 (FC)** | 22.12 | 26.50 | 21.00 | 15.50 | 25.50 |
| Qwen3-1.7B (FC) | 11.00 | 15.00 | 6.00 | 12.00 | 11.00 |

BFCL has no Qwen3.5 handler; S1 adds an OpenAI-compatible chat-completions FC handler and reports its gap to the official one on 2507 (harness sensitivity proper: 02 Shellwright).

**Context caps.** Function docs take 6–17.5 KB of JSON per class (`vehicle_control` 17.5 KB, `travel_booking` 17.0 KB), ~5–9k tokens for two classes. Long-context tool outputs reach ~84k characters (`TRANSACTION_HISTORY_EXTENSION`, `long_context.py`), ~21k tokens at 4 chars/token. S1 measures max prompt tokens per category; `long_context` gets 32k per slot (2 slots, 2 GiB KV; fits 6 GB); 2507 runs only `multi_turn_base`. Over-cap prompts fail as `context_overflow`; their category is labelled "ctx-capped, not leaderboard-comparable."

### 5.2 τ²-bench v1.0.1: airline (50) + retail (114)

Agent, LLM user simulator and domain database, via LiteLLM to llama-server.
- **Topology**: agent Qwen3.5-4B on Windows, simulator Qwen3.5-9B on the Mac, over the LAN; endpoints and build hashes go into the cassette. Labelled fallbacks: a Qwen3.5-4B simulator on the agent's server, or both on the Mac at ≤ 16k each (if S0 memory allows).
- **Scoring**: airline rewards DB state + communication, both deterministic. Retail lists `NL_ASSERTION` in 112 of 114 reward bases, but only 40 tasks carry assertions and empty lists score full, so 74 stay deterministic under the default judge (`gpt-4.1-2025-04-14`). Retail primary is DB-only; a local Qwen3.5-9B judge scores the 40 separately.
- **Simulator at temperature 0.7** in every run, baseline or fork (τ² default: 0.0), so a failure and its forks share one user distribution, labelled "local user sim, temperature 0.7 — not leaderboard-comparable."

### 5.3 Who&When (offline attribution)

184 failure logs on GAIA and AssistantBench (`Kevin355/Who_and_When`), labelled with responsible agent and decisive step: 126 algorithm-generated (CaptainAgent; median ~9k characters) and 58 hand-crafted (Magentic-One; median ~45k, p90 ~151k, max ~330k characters ≈ 11k / 38k / 82k tokens at ~4 chars/token). Lengths are parquet-measured (Qwen-tokenizer recount in S3).

Published accuracy (algorithm-generated / hand-crafted; with ground truth unless stated), context only. Sources: Who&When Table 1 (first four rows, checked against the PDF), CHIEF Table 1, FALAT Table 2.

| Method (backbone) | Step-level | Agent-level |
|---|---|---|
| Random | 19.06 / 4.16 | 29.10 / 12.00 |
| All-at-once (GPT-4o) | 12.50 / 5.26 | 54.33 / 55.17 |
| Step-by-step (GPT-4o) | 25.51 / 7.02 | 35.20 / 34.48 |
| Binary search (GPT-4o) | 23.98 / 6.90 | 44.13 / 51.72 |
| AgenTracer (8B fine-tuned; as reported by CHIEF) | 42.90 / 20.70 | 69.62 / 69.10 |
| CHIEF (DeepSeek-V3.2, thinking) | 52.00 / 29.31 | 76.80 / 77.59 |
| FALAT, best backbone per subset (MiniMax 2.5 / DeepSeek V3.2; ground-truth use not stated) | 46.0 / 29.1 | — |

On algorithm-generated logs random beats the GPT-4o all-at-once judge on step-level accuracy (19.06 vs 12.50); on hand-crafted logs it does not (4.16 vs 5.26). Hence random is a required baseline.

### 5.4 Project-built data
- **Fault-injected BFCL sets U and V** (§7.7), released with the generator.
- **60 natural failures** (100 stretch) from Qwen3.5-4B on BFCL and τ² airline (τ² only if τ² forks survive week 8). A written rubric labels the last recoverable step and the earliest decisive error (§7.4), plus failure-class tags (§7.7); forks never produce labels. 40 are double-labelled by a peer or the mentor (~1.5 h in S3) for Cohen's κ with its 95% interval.

## 6. Architecture

```
  ALL LOCAL: Windows + CUDA laptop, Apple Silicon Mac, one repo

  Harness: BFCL v4 multi-turn | tau2-bench v1.0.1 | any OpenAI-client agent
    agent loop --> blackbox SDK
                    |- httpx shim ---- chat spans -----> llama-server (pinned build)
                    |  role: agent | user_sim | judge     Qwen3.5-4B/2B/9B, Qwen3-4B-Instruct-2507
                    |- tool adapter -- execute_tool ----> BFCL API objects / tau2 DB
                    '- snapshot hook - state per step --> snapshot registry
                              |
     RECORDER  OTel GenAI spans (pinned semconv, adapter) --> Phoenix (local UI)
               cassettes: sha256 blobs + SQLite index --> Parquet
                              |
     +---------------+--------+---------+------------------+
   REPLAY          FORK ENGINE        STATS              INVESTIGATOR (Qwen3.5-9B)
   pure /          restore S(k-1),    pass^k, McNemar,   reads span tree,
   verify-tools /  intervene at k,    paired bootstrap,  fork() / diff_runs()
   strict;         suffix x n,        power curve, A/A   under budget ->
   no model        Wilson CI on p(k)                     ranked steps + repro cassette
                              |
   CI GATE  hosted runner (no GPU): unit + verify-tools replay -> affected tasks
            mentee runs `blackbox ci live --sha <commit>` locally: power-sized
              N x k on affected tasks -> results JSON keyed by SHA + GGUF hash
            hosted job reads that JSON (no self-hosted runner) -> verdict:
              REGRESSION | NO REGRESSION >= MDE | INCONCLUSIVE
   REPORT   pass^k trend, divergence heatmap, fork timelines (static HTML)
```

**Walk-through.** Wrapping the OpenAI client's `httpx` transport (where BFCL handlers and τ²'s LiteLLM end) leaves harnesses unmodified; a localhost proxy covers non-Python agents.

## 7. Component Design

### 7.1 Recorder and span model
- One `invoke_agent` span per task run, child `chat` and `execute_tool` spans with standard `gen_ai.*` request, tool and message attributes, plus `blackbox.*`: role, step, endpoint, cassette key, model sha256, llama build, `server_flags` (§4.2 flags + `--reasoning-budget`), env snapshot sha256, `fork_of`, intervention. Only the adapter knows semconv names.

### 7.2 Cassette store and replay
- **Key**: sha256 of the canonical request (model hash, messages, tools, sampling, seed, server flags). Sorted keys and normalized numbers prevent `25` vs `25.0` divergence, the int-vs-float bug τ²-bench v1.0.1 fixed in its banking_knowledge domain (#397).
- **Index**: SQLite (run, step, span, parent, role, key, snapshot hash, outcome); content-deduplicated blobs.
- **Modes**: *pure* (model and tools from cassette; viewing, diffing); *verify-tools* (model from cassette, tools live, final state hash asserted; CI). *Strict* applies to both: an unmatched request fails with a field-level diff, VCR-style.
- **What a change re-runs**: a report or viewer change re-renders stored spans only (Phoenix, report); a new checker, rubric or judge re-scores stored spans and §7.3 snapshots without a harness run (all of Who&When; a checker costs seconds, a §9 judge 9B tokens); harness code alone re-runs on recorded responses, VCR-style (*pure*, then *verify-tools*), and only that catches R0.
- **No look-ahead**: a reader opened at fork step *k* refuses entries with step ≥ *k*.

### 7.3 Environment snapshots
- **BFCL**: after each step, pickle every backend instance in play, including private state the checker ignores. Travel, messaging, vehicle and trading hold a seeded `self._random`; without its `getstate()`, fork suffixes draw different booking IDs.
- **State hash**: canonical JSON of the public attributes the checker compares. The file system's `_last_modified` (`datetime.now()`) is excluded or frozen by a cassette-recorded clock shim, so verify-tools replays hash-match.
- **τ²**: domain database as JSON, plus simulator conversation and seed.
- **Fidelity test (S2 exit)**: on 100 recorded runs, restore *S(k−1)* at a random *k* and replay *k…T* in verify-tools mode; 100% of final state hashes must match.

### 7.4 Fork engine
`fork(run, k, I, n)` on steps *s₁…s_T*: (1) restore *S(k−1)* and the cassette's messages; (2) produce step *k* by intervention *I*: *resample* (agent policy, new seed), *stronger* (Qwen3.5-9B acts), *oracle* (BFCL ground truth for the turn; upper bound) or *edit* (a supplied argument fix); (3) run the suffix live *n* times with distinct seeds (τ²'s simulator varies too); (4) return p̂(k) = successes / *n* with a Wilson interval.

**Decisive step (pre-registered).** p̂(k) stays near the task's own success rate until the first unrecoverable error, then collapses.

| Element | Rule |
|---|---|
| Reference | p̂_ref = p̂(1), resampled from step 1, *n* = 5 (a relative bar) |
| *k\** | last step with p̂(k) ≥ τ · p̂_ref; τ = 0.5 (sensitivity 0.3, 0.7) |
| Boundary | the *k\** → *k\**+1 drop passes one-sided Fisher exact, α = 0.05, *n* = 5 per side (4/5 vs 0/5 passes; 4/5 vs 1/5 does not) |
| Floor | p̂_ref ≤ 1/5 locates no boundary: labelled "no recoverable step under resample", passed to *stronger* and *oracle* |

*k\** is the last recoverable step; Who&When labels the earliest decisive error (the first mistake whose fix turns failure into success). They differ only if that error is recoverable later (BFCL allows up to 20 steps per turn); forks and judges are prompted for and scored on the same label.

**Search and cost.** Binary search assumes p̂(k) non-increasing in *k*; an exhaustive sweep on 30 cases reports the non-monotone share and agreement. Per failure: 5 (p̂_ref) + ~log₂T × 3 (search) + 2 × 5 (boundary) ≈ 27 suffix samples at T = 16; a suffix ≈ half a run (planning value, measured in S2).

### 7.5 The agent under test — loop
- **BFCL**: sees user turn, history and tool results; calls available functions (missing-function entries withhold some) or replies. A call-free reply ends the turn, 20+ steps force-end it, and the entry ends after its last turn. Output-token and context caps per profile are recorded (`context_overflow`, §5.1).
- **τ²**: sees a simulator message or tool result; acts with one tool call or message; ends on simulator stop, transfer or step cap.
- **Sampling**: card non-thinking settings (temperature 0.7, top_p 0.8, top_k 20), except `presence_penalty` 0 on agent calls: the card's 1.5 can penalise the repeated argument tokens tool calls need, and the card warns of "a slight decrease in model performance". S1 ablates 0 vs 1.5 on 50 entries.

### 7.6 Nondeterminism study

50 BFCL `multi_turn_base` entries × 5 reruns per cell, Windows/CUDA unless noted. One-factor cells attribute effects; C4 and C7 are the configurations actually run.

| Cell | Setting | Isolates |
|---|---|---|
| C0 | `--parallel 1`, `--no-cache-prompt`, `--cache-ram 0`, solo, temperature 0, fixed seed | baseline |
| C1 | `--parallel 4`, solo | slot layout |
| C2 | `--parallel 4`, 4 concurrent requests | co-batching vs C1 (`--parallel 1` never co-batches) |
| C3 | C0 + prompt cache | prompt caching |
| C4 | stock defaults: only model, context, `--jinja`; solo | "by default" (claim 1) |
| C5 | 2507 at C0 | pure-attention contrast |
| C6 / C7 | card sampling, fixed seed, at C0 / C2 | seeded sampling; C7 = bulk configuration of forks and CI; its reruns are the A/A pairs |
| M0 / M4 | Metal (Mac): C0 / C4 on 30 entries | cross-backend vs the same 30 CUDA entries (same build, GGUF hash) |

- **Measures**: divergent-rerun share; first-divergence step and token; outcome-flip rate vs the cell majority; cross-backend agreement; top-1/top-2 logprob margin at divergence, where returned.
- **Unit: the entry** (reruns of one entry are not independent). A clean cell reads "0 of 50 entries diverged, 95% upper bound 6%" (3/n).

### 7.7 Fault-injection generator
- From passing runs, inject one fault at step *k*: corrupted argument, wrong tool, dropped state-changing call, premature end of turn, or an environment fault (a tool errors once, or silently returns a corrupted result).
- **Set U (headline)**: every injection whose run fails, labelled *k\** = *k* by the injector alone, no live filter, until ≥ 150 cases. Recovered injections are not failures; recovery rate per fault type is reported. Label noise (a later natural error causes the failure) hits forks and judges alike.
- **Set V (fidelity)**: 60 U cases whose faulted suffix fails ≥ 4/5 live and whose clean prefix from *k* succeeds ≥ 4/5. That is fork search's own statistic, so forks should be near ceiling: V reports sample cost and noise robustness only, never evidence that forks beat judges.

**Failure classes** (reported for coding, data and multi-agent systems, §15) are non-exclusive tags, by rule, else by the §5.4 rubric (natural failures only); they never move *k\**. Per-class accuracy is descriptive (Wilson intervals, no test).

| Class | Rule |
|---|---|
| Silent wrong output | a tool result the agent relied on was wrong without an error; injectable (the silent fault in U) |
| Redundant exploration | ≥ 3 state-preserving calls off the ground-truth path in one turn, threshold fixed on S1 passing runs (vendor-reported: a data-native harness needed 1.5× fewer tool operations and 2.2× fewer steps than a generic one) |
| False investigation | steps probing correct state as if it were the bug; rubric only |
| Tool loop | one call (tool, canonical arguments) repeated with an unchanged state hash until the step cap (§7.5) |

No rediscovery class (§15): it needs a result to leave the context, and BlackBox runs never compact or span sessions (§5.1); 02 (compaction) owns it; 09 reports the cross-session kind descriptively (Planned relay).

### 7.8 Investigator agent — loop
- **Observes** a span-tree digest (step, role, tool, argument, result and state-diff digests), the checker's failure message and fork results so far.
- **Tools**: `get_step(k)`, `state_diff(k)`, `fork(k, intervention, n)`, `diff_runs(a, b)`, `submit(ranking)`.
- **Budget**: 10–30 fork samples per failure, ≤ 40 tool calls, a token cap, `--reasoning-budget 4096`; wall-clock measured.
- **Policy**: suspects from reading, bracketed by forks; boundary binary-searched; *k\**±1 verified; ends on `submit` or, at budget exhaustion, with the fork-implied ranking.
- **Output**: ranked steps with p̂ and intervals plus a minimal reproducing cassette (prefix to *k\**, snapshot, intervention) for the CI failure. No fork tools on Who&When.

### 7.9 Regression CI and power analysis

| Element | Design |
|---|---|
| Seeded changes | R0 harness bug (tool-result truncation); R1 Q4_K_M → Q3_K_M; R2 system-prompt edit, fixed before measurement; R3 Qwen3.5-4B → 2507, both arms on the new FC handler; an A/A pair for false alarms |
| Planted doses | tool results corrupted with probability *p*, set on a 50-entry pilot for ~5, 10, 15 pp drops |
| Unit | one outcome per task per arm; N × k run-level pairs would be pseudo-replication |
| Tests | primary: McNemar on run 1 (pass^1); secondary: per-task pass^k via unbiased C(c,k)/C(n,k), paired bootstrap over tasks (clusters), never runs |
| Power curve | resample N′ ≤ N tasks, k′ ≤ k runs from A/A (C7 + baseline reruns, S1) and planted runs (S2); detection rate at α = 0.05 vs analytic McNemar power |
| Verdict | REGRESSION / NO REGRESSION ≥ MDE / INCONCLUSIVE; underpowered is inconclusive, never "pass" |

**Gate flow** (§6). Hosted verify-tools replay must catch R0 with no model (checked in S1). The mentee runs tasks with changed requests live at the power table's N × k and pushes results JSON to a `ci-results` branch; the check stays pending until then. No self-hosted runner: GitHub advises against them on public repositories, since a fork's pull request can run code on the machine. CI cassettes: one zstd bundle per suite, ≤ 500 MB, a versioned release asset cached by the hosted job.

**Replay savings**: the share of tasks each change forces live, over all project commits to agent-harness, tool-adapter and parser code plus ~20 seeded change types (tool-schema edit, retry policy, parser fix, prompt edit, model swap). Prompt edits and model swaps miss every cassette.

## 8. Hardware Profiles

**Rule: the weakest machine sets the defaults**: memory from the 6 GB card, speed from each machine's week-2 measurement.

- **Profiles**: `portable` (default) uses the 6 GB column, with one 9B server on the Mac (partial offload on 6 GB only offline); `accelerated` (opt-in) runs the 4B at 4 × 32k on a measured 8 GB card or the Mac, and the 9B fully on GPU (8 GB, tight).
- **Placement**: Mac: 9B server (τ² simulator, judges, investigator, Who&When), S0 Q8_0 reproduction, Metal cells. Windows: τ² agent, bulk BFCL, fault injection, forks, live CI. Both: determinism grid.
- **Host prompt cache**: `--cache-ram 0` in study cells, `2048` in bulk (the 8192 MiB default alone could crowd out a 9B on the 16 GB Mac); `--ctx-checkpoints` pinned.

**Memory** (KV from configs at f16; compute buffers ~0.5 GB for the 248k-vocabulary logits at ubatch 512, estimated, measured in S0):

| Process (weights: §4.1) | KV | 6 GB | 8 GB | Mac 16 GB |
|---|---|---|---|---|
| Qwen3.5-4B Q4_K_M, 1 × 16k (study cells) | 0.5 GiB | fits | fits | fits |
| 4B, 4 × 16k or 2 × 32k = 64k (bulk, `long_context`) | 2 GiB | fits (~4.7 GiB before compute buffers; headroom measured) | fits | fits |
| 4B, 4 × 32k = 128k (accelerated) | 4 GiB | no (~6.7 GiB before buffers) | tight, measure | fits |
| Qwen3.5 recurrent state (24 Gated DeltaNet layers × 32 heads × 128 × 128, f32) | ~48 MiB per slot and per context checkpoint | counted in S0 | | |
| 2507 Q4_K_M, 16k | 2.25 GiB | fits (not 32k) | fits | fits |
| 2507 Q8_0, 16k | 2.25 GiB | no | tight | fits |
| Qwen3.5-9B Q4_K_M, 64k | 2 GiB | partial offload | tight at 32k | fits |
| 4B + 9B co-resident (8.42 GB = 7.84 GiB weights), ≤ 16k each (fallback τ² topology only) | 1 GiB | no | no | measured in S0; near macOS's GPU-wired cap (below RAM) |

**Week-2 go/no-go** (end of session 2, before determinism v1 and baselines): Qwen3.5-4B prompt and decode tokens/s at 8k and median wall-clock over 20 `multi_turn_base` entries (cache-off, 1 slot and bulk) on both machines; 9B tokens/s and S0 reproduction status on the Mac. The gate below covers Windows bulk only. A Mac cache-off entry re-prefills ~75k tokens (`--no-cache-prompt`), an estimated 3–6 minutes at llama.cpp's public Apple Silicon rates, so Mac rows are sized in entries with their own cuts.

**Compute ledger** (core tier; Qwen3.5-4B unless noted), at planning rates until week 2: Windows 60 s/entry (aggregate over slots in bulk); Mac 120 s cache-on, 300 s cache-off; τ² 180 s/task. A fork suffix ≈ 0.5 run; parentheses give full-run equivalents. Sprint timing: §10. Planned and stretch items are costed at the week-8 checkpoint.

| Item | Runs | Win h | Mac h |
|---|---|---|---|
| Determinism v1, 20 × 5 × 2 cells (Win), 10 × 5 × 2 (Mac) | 300 | 3.3 | 5.8 |
| S0 reproduction (§10), 3 × 200 | 600 | | 20.0 |
| Handler check, 2507 Q4, 2 handlers × 200 | 400 | 6.7 | |
| Baselines, 4B on 800, 2B on 200, presence-penalty ablation 2 × 50 | 1,100 | 18.3 | |
| τ² airline pass^1–4, 50 × 4 (agent on Win, 9B simulator on Mac, concurrent) | 200 | 10.0 | 10.0 |
| Grid, C0–C7 × 250 (Win; C5 on 2507); M0, M4 × 150 (Mac) | 2,300 | 33.3 | 17.5 |
| Regression CI, A, A′, R1, R2, R3 × 200 × k = 3 (R3 on 2507) | 3,000 | 50.0 | |
| Dose-response, pilot 3 × 50 + 3 doses × 200 | 750 | 12.5 | |
| Fault injection, U (~250 injections, planning), V (60 × 10) | 850 (425) | 7.1 | |
| Resample fork search, (150 U + 60 natural) × 27 | 5,670 (2,835) | 47.2 | |
| *Stronger* + *oracle*, 60-case subset × 27 × 2 | 3,240 (1,620) | 27.0 | one 9B call per fork |
| Monotonicity sweep, 30 cases × 16 steps × 3 | 1,440 (720) | 12.0 | |
| Judges (9B), three rungs on 210 failures; compute-matched on 60 | — | | token-bound, measured at week 2 |
| **Core total** (Runs: Windows full-run equivalents) | ≈ 13,050 | ≈ 228 incl. τ² | ≈ 53 + judges |

**Gate rules.** Budget: 250 unattended Windows machine-hours (~25 nights × 10 h), confirmed by the mentee at S0. s_max = (budget − τ² hours) × 3600 / full-run equivalents = 66 s (full core) or 83 s (after cuts 1–5). Rounded down: ≤ 65 s/entry, full core; 65–80 s, cuts 1–5 in order until it fits (≈ 10,430 equivalents after all five); > 80 s, add cut 6.

**Compute cuts (pre-registered order)**: (1) monotonicity sweep 30 → 15 cases; (2) *stronger*/*oracle* subset 60 → 30; (3) cells C5, C6 → 30 entries; (4) CI k 3 → 2 (pass^1 test unaffected); (5) dose-response 3 → 2 doses; (6) fork, fault and CI rows → Qwen3.5-2B (the 4B keeps grid, baselines, handler check). Mac overruns: judge subset 60 → 30, then Metal cells 30 → 20 entries; the S0 reproduction stays 3 × 200, cache on.

## 9. Evaluation Plan

Replay fidelity (hash match, strict-miss rate, latency vs live rerun) is tested in §7.3; determinism in §7.6. Pre-registered tests use α = 0.05 two-sided, power 0.8, McNemar normal approximation.

| Area | Metrics | Compared against | n and MDE |
|---|---|---|---|
| Agent baselines | BFCL multi-turn, 4 categories, `context_overflow` counted; τ² airline and retail-DB pass^1…pass^k | §5.1 rows (ctx-capped labelled); none for τ² | — |
| Regression CI, pass^1 (run 1), BFCL `multi_turn_base` | R0–R3 verdicts; planted-dose detection; A/A false alarms; live runs avoided per change type | analytic McNemar power; all-live CI | 10 pp at 155 tasks, discordance d = 0.20 |
| Attribution: set U | exact-step and ±1 accuracy; fork samples, tokens, wall-clock; non-monotone p̂ share | random; ladder below; binary-search judge | 15 pp at ≥ 103, d = 0.30; ≥ 150 targeted → ≈ 12.4 pp |
| Attribution: natural failures | same, plus κ (§5.4) | same | 20 pp: 60 → 19.4 pp at d = 0.30; 100 (stretch) → 15.2 pp; Wilson intervals per method |
| Attribution: set V | fidelity and sample cost only (§7.7) | no comparison | 60 |
| Who&When | agent- and step-level accuracy, both subsets, ± ground truth | random; local same-model judges | — |

**CI sample size.** N is re-fixed from S1 A/A discordance. All 200 base entries give 8.8 pp (d = 0.2) or 10.8 pp (d = 0.3); all 800 multi-turn entries, 4.4 pp at d = 0.2. pass^k uses the paired task-cluster bootstrap.

**Ablation ladder**: random → all-at-once judge → + per-step state diffs → step-by-step judge → compute-matched judge (self-consistency over *m* votes, *m* set so judge tokens ≈ fork-search tokens per failure; 60 cases) → *resample* forks → + *stronger* → + *oracle* → investigator.

**Reporting rules**: success comes from the benchmark checker (backend state, BFCL's call-path subset, τ² airline's communication check; §5.1–5.2), as τ² and BFCL score it, not from the agent's final message; the retail NL judge is reported apart; a compute ratio beside every judge-vs-fork gap; build, flags and GGUF hashes on every table; injected and natural failures apart; no comparison with an API model never run; nulls published.

## 10. Milestones (19 hours / 15 sessions)

| Sprint | Sessions | Deliverable |
|---|---|---|
| **S0** | 1–4 | Repo, `uv`, pinned build and GGUF hashes; week-2 go/no-go (§8); published-number reproduction (below); τ² airline smoke test across machines; Mac 4B + 9B co-residence at ≤ 16k; httpx shim spans; determinism v1 |
| **S1** | 5–7 | Cassettes, replay, Phoenix; Qwen3.5 handler + check; BFCL and τ² airline baselines; full grid; R0 caught by hosted replay. Exit: 20-step replay < 1 s. Week-6 demo: scrub a failed run in Phoenix, replay it |
| **S2** | 8–10 | Snapshots + fidelity test (exit); fork engine; sets U and V; attribution v1 (§9 ladder through *oracle*, monotonicity sweep); power curve. Week-8 checkpoint (session 10): planned-tier cuts |
| **S3** | 11–12 | 60 natural failures labelled, κ on 40 (§5.4); fork vs judge on them; planned tier if kept (Who&When, τ² forks and retail, investigator) |
| **S4** | 13–14 | R1–R3 live, `blackbox ci live` + hosted gate, three-valued verdict, replay savings; docs, package, dataset and generator released |
| — | 15 | Interview sprint; write-up (§14) |

**S0 reproduction rule (pre-registered).** 2507 (FC) BFCL `multi_turn_base` 26.50%, official handler, local llama-server (Q8_0, Mac, prompt cache on), ≥ 3 runs. Pass: mean within 2 × the run-to-run standard deviation (SD) of 26.50%, floored at 2 pp; else document the gap. Not a binomial interval (±6.1 pp): leaderboard and local runs share the same 200 entries, so it would model absent task sampling and pass a ~5 pp handler or quantization bug.

**Tiers.**
- **Core** ships regardless: all that claims 1–3 need (§7.1–7.7, §7.9), plus τ² airline recording and baseline.
- **Planned**, cut in this pre-registered order at week 8 if behind: (1) Who&When (claim 4); (2) τ² forks and retail (forks become BFCL-only); (3) investigator (fork search alone carries claim 3); (4) natural failures 60 → 30, descriptive only.
- **Stretch**: the mentee's LangGraph Writer–Critic agent under the SDK shim (demo), 100 natural failures, a Qwen3.5-9B BFCL baseline.

## 11. Risks & Mitigations

| Risk | Mitigation |
|---|---|
| Record/replay is crowded (agrepl, reeltime, vendor tracers, LangGraph) | Cite it; differentiate as in §4.3 |
| Scope exceeds 19 h + independent time | Tiers and cut orders (§8, §10); S4 holds only R1–R3 live and the hosted gate |
| Compute exceeds the plan (cache-off Mac entries take minutes) | Derived gates and cuts (§8); Mac cells at 30 entries; Mac prompt cache on outside determinism cells |
| The grid finds no divergence | Still a result (entry-level bound, §7.6); claims 2–4 do not depend on it |
| Injected-fault labels are circular with the fork rule | Headline on unfiltered U and natural failures; V is fidelity only |
| τ² forks are stochastic via the user sim; injected faults may be unrealistic | Sim at temperature 0.7, *n* samples, intervals; natural failures reported separately |
| BFCL long-context outputs (~21k tokens) overflow 16k slots | 32k slots, `context_overflow` type, ctx-capped label (§5.1) |
| Qwen3.5 tool-call parsing changes between llama.cpp builds | One pinned build; golden parse tests; official-handler cross-check on 2507 |
| GenAI semconv churn (Development, no release) | Pinned commit behind one adapter |
| 9B does not fit 6 GB; 4B + 9B may not co-reside on the Mac; Who&When logs exceed context | Topology in §5.2 and §8; step-by-step mode for long logs; truncation reported per case |
| Hosted-runner minutes on a private repo (2,000/month on Free) | Replay suite under ~5 minutes; public repo under the open-source placement; never self-hosted (§7.9) |

## 12. Résumé Bullets (experience voice; numbers filled in as sprints land)

- Built **BlackBox**, a local flight recorder for LLM agents (OpenTelemetry GenAI spans + content-addressed cassettes); strict replay reproduced **X%** of tool-call sequences and final state hashes on BFCL v4 multi-turn and τ²-bench, a 20-step run in **X ms** with no model.
- Measured **temperature-0 nondeterminism** of one GGUF across llama.cpp settings and CUDA vs. Metal: **X%** of reruns diverged (median first divergence at step **X**), flipping **X%** of outcomes.
- Designed **power-analysed regression CI** (McNemar on pass^1, paired bootstrap on pass^k), power curve validated on planted regressions: caught **X** of **Z** planted regressions ≥ the **X** pp MDE at an **X%** A/A false-alarm rate, classified **Y** seeded changes (quantization, prompt, model swap) and skipped **X%** of live-model runs over **X** code changes via free hosted replay.
- Built **counterfactual fork attribution** and a local Qwen3.5-9B triage agent: **X%** exact-step accuracy vs. **X%** for a same-model judge at matched compute on **X** unfiltered fault-injected failures (generator and dataset released), **X%** on **X** human-labelled natural failures, **X%** step-level on Who&When.

**Placement note.** These read as experience only under a real engagement: the mentoring organization (engagement record, truthful title), a real collaboration with a startup, lab or open-source org hosting the work, or an open-source project entry with public commits. Never invent an employer or dates: North American employers verify, and a fabricated entry can rescind an offer.

## 13. Why This Reads as Real Company Work

- **Every model upgrade asks** whether the new model, prompt or quantization is worse; BlackBox gives a verdict that admits when it cannot tell, and the postmortem teams want: "step 7 broke it, here is a reproducing cassette".
- **Operational knowledge**: CI split between GPU runs and free replay; which server flag makes evals flaky; statistics pre-registered.

## 14. Interview Leverage

- **ML system design and coding** (eval infra for an agent platform): capture points, content-addressed cassette keys, canonical serialization, transport-level instrumentation, replay modes, CI split by GPU need.
- **Applied LLM**: why temperature 0 is not deterministic; hybrid-attention KV budgets; quantization effects.
- **Experimentation**: paired designs, McNemar vs. bootstrap, the pass^k estimator, power curves, "not significant" ≠ "no regression".
- **Research / behavioural**: last recoverable vs. earliest decisive step; why a set filtered by your own method cannot test it; pre-registration; publishing nulls.
- **Drills** (practice prompts on common agent-engineering themes): *"Your eval says your method wins: what else could explain it?"* → §7.7, §7.9, §9; *"Reconstruct what an agent saw and changed at each step from its logs."* → §7.1, §7.3, §7.4; *"Calibrate an LLM judge against human labels."* → §5.4, §9.
- **Write-up:** one post under the mentee's own byline (session 15, §10), working title *"Is temperature 0 deterministic on my laptop? I measured it"*: hook, context, the work (failure before fix: a temperature-0 rerun that flips its outcome, then replay, forks and the power-sized gate), what I'd do differently (the pre-declared null or fallback: a clean grid's entry-level bound, §7.6; INCONCLUSIVE verdicts, §7.9), takeaways. Every number traces to the results table; vendor numbers are labelled.

## 15. References

- BFCL: https://gorilla.cs.berkeley.edu/leaderboard.html · https://gorilla.cs.berkeley.edu/data_overall.csv · https://gorilla.cs.berkeley.edu/blogs/13_bfcl_v3_multi_turn.html · https://github.com/ShishirPatil/gorilla/tree/main/berkeley-function-call-leaderboard · backends: https://github.com/ShishirPatil/gorilla/tree/main/berkeley-function-call-leaderboard/bfcl_eval/eval_checker/multi_turn_eval/func_source_code
- τ²-bench: https://github.com/sierra-research/tau2-bench · https://github.com/sierra-research/tau2-bench/releases · https://github.com/sierra-research/tau2-bench/releases/tag/v1.0.1 (retail `tasks.json` and `src/tau2/config.py` at that tag) · fix #397: https://github.com/sierra-research/tau2-bench/pull/397 · τ-bench (pass^k): https://arxiv.org/abs/2406.12045 · τ²-bench: https://arxiv.org/abs/2506.07982
- Who&When: https://arxiv.org/abs/2505.00212 · https://huggingface.co/datasets/Kevin355/Who_and_When · CHIEF: https://arxiv.org/abs/2602.23701 · FALAT: https://arxiv.org/abs/2606.00765 · AgenTracer: https://arxiv.org/abs/2509.03312
- Prior art: https://arxiv.org/abs/2607.16200 · https://pypi.org/project/reeltime/ · https://pypi.org/project/vcrpy/ · https://docs.langchain.com/oss/python/langgraph/use-time-travel
- Telemetry: https://github.com/open-telemetry/semantic-conventions-genai · https://arize.com/docs/phoenix/release-notes/05-2026/05-15-2026-otel-semconv-conversion
- llama.cpp server: https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md · https://github.com/ggml-org/llama.cpp/releases · Apple Silicon throughput: https://github.com/ggml-org/llama.cpp/discussions/4167
- Failure classes: vendor posts (vendor-reported; seen via search 2026-10-22, full text re-read in S0) https://www.snowflake.com/en/blog/engineering/data-eng-bench-data-engineering-agent-benchmark/ · https://www.snowflake.com/en/blog/engineering/ai-agent-memory-experience-cards/ · https://www.snowflake.com/en/blog/engineering/arcticmem-persistent-memory-ai-agents/ · unbounded agent loops: https://arxiv.org/abs/2607.01641 · multi-agent failure taxonomy (MAST: step repetition, missing or incorrect verification; re-check in S0): https://arxiv.org/abs/2503.13657
- Nondeterminism: https://thinkingmachines.ai/blog/defeating-nondeterminism-in-llm-inference/ · https://docs.vllm.ai/en/stable/features/batch_invariance/
- Models: https://huggingface.co/Qwen/Qwen3.5-4B · https://huggingface.co/Qwen/Qwen3-4B-Instruct-2507 · https://huggingface.co/unsloth (Qwen3.5-{2B,4B,9B}-GGUF, Qwen3-4B-Instruct-2507-GGUF) · https://artificialanalysis.ai/articles/qwen3-5-small-models
- GitHub Actions billing: https://docs.github.com/en/billing/concepts/product-billing/github-actions · self-hosted runners: https://docs.github.com/en/actions/how-tos/manage-runners/self-hosted-runners/add-runners
