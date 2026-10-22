# HarnessForge — Design Document

**A Local Coding Agent That Rewrites Its Own Harness Overnight on a Gaming Laptop — and Promotion Gates That Stop It Fooling Itself**

| | |
|---|---|
| Status | Proposal |
| Target duration | ~10–12 weeks (19 mentoring hours / 15 sessions + independent work; the search runs overnight, unattended). Sprints S0a, S0b, S1–S4 (§10) |
| Platforms | Windows 11 laptop (16 GB RAM, RTX 3060-class, **6 GB VRAM assumed**, 8 GB if confirmed) and Apple Silicon Mac (16 GB, Metal, no CUDA). One repository. |
| Deployment | 100% local. No rented servers, no paid GPU cloud. A commercial API appears only as a capped (< USD 20 total) **proposer** comparison arm; the evaluated agent is always local. |
| Core stack | Python 3.12 · uv · llama.cpp `llama-server` (CUDA / Metal) · Qwen3.5-4B / Qwen3.5-9B GGUF · Docker · SQLite · MLflow · Aider polyglot · Harbor · GEPA |

---

## 1. The Product Thesis

Teams shipping small-model agents hand-tune the **harness** (prompt, tools, edit format, context policy, retry, verification) on 20–50 examples; winners fail to replicate or quietly spend more tokens. Yet the harness matters:

- Aider polyglot leaderboard: Qwen3 32B made at least one malformed `diff` edit on 16.4% of tasks (83.6% well-formed) despite format-repair retries.
- Qwen3.5-9B spans 36.0–44.1 on OpenThoughts-TBLite across four harnesses (Tmax, Table 5, as cited in 02 Shellwright).

> **Engineering thesis: for a fixed small model, the harness is a hyper-parameter space that can be searched automatically on a laptop — provided evaluation is cheap and promotion is statistically honest.**

The reference is the **Darwin Gödel Machine (DGM)**: full Polyglot 14.2% → 30.7% over 80 iterations (o3-mini agent, o1 diagnosing, a Claude 3.5 Sonnet (New) agent writing self-modifications). Its paper puts one SWE-bench run at **about USD 22,000** of API spend. Harness optimization is a named problem (Meta-Harness, AutoSaddler, Task-CoEvolve, PRISM, HarnessOpt-Bench, which seals a test partition in a trusted execution environment); a NeurIPS 2026 paper co-evolves validation gates for Meta-Harness. HarnessForge adds a **measured false-promotion rate** and **matched-GPU-hour controls** on a 4B model, a 6 GB GPU and under USD 20.

Four claims, each with a pre-registered test (§9):

1. **Laptop DGM-lite works:** evolved harness code lifts Qwen3.5-4B's held-out pass@2 (solved within two attempts, §5.1) over the seed harness (exploratory: vs. the seed-harness 9B).
2. **It beats controls at equal GPU-hours** (§7.8); an API-proposer arm separates proposer weakness from method failure.
3. **Overfitting and transfer are measured:** search-vs-held-out gap along the champion lineage; transfer to Qwen3.5-9B and 02 Shellwright's 30-task TBLite-laptop slice.
4. **Adaptive validation is cheap; the gate is calibrated:** variance-weighted sampling plus successive halving finds the full-set winner with X% of the evaluations (with a confidence interval, CI), testing Task-CoEvolve's reported 80% reduction; false promotions are measured on null children.

**Why it is fun:** a program improves its own source overnight (*"Night 3: champion changed — syntax-check each file after writing it; +X, confirmed on fresh seeds"*), and few harness-search results report how often their selection rule promotes a change that does nothing.

## 2. Goals & Non-Goals

### Goals
Build a **tamper-resistant evaluator** for Aider polyglot in Docker (§7.1), the **forge** (§7.2–7.5), a calibrated gate (§7.6) and matched-budget controls (§7.8); measure overfitting, transfer and adaptive validation (§9); ship a ten-minute morning report (§7.9).

### Non-Goals
- Weight updates (01 Compass, 02 Shellwright, 03 Spelunker): the model is frozen. Hand-run harness ablations: 02 Shellwright, whose agent-computer interface (ACI) variants can seed this library.
- A nondeterminism study (05 BlackBox); only the noise floor needed for power is measured. "Replay" here re-runs selection on this project's stored outcome matrix, not BlackBox's replay-based test selection; both calibrate on A/A runs (a harness against itself).
- Like-for-like DGM comparison (§5.1); full DGM self-reference (the evolving harness modifying itself) beyond a stretch arm; evaluator, grader or budget changes; cloud dependencies; full SWE-bench or Terminal-Bench 2.

## 3. Alignment With Prior Experience

| Prior experience | HarnessForge module |
|---|---|
| **Changsheng**: AutoML hyper-parameter agent, staged search, backtest promotion discipline (return after costs) | **Forge + halving + gate**; return after costs → score at matched GPU-hours |
| **DiDi**: A/B testing | **Power analysis, paired bootstrap, minimum-detectable-effect (MDE) pre-registration, A/A calibration** |
| **CAS**: 20+ retrieval configs benchmarked on Precision@K, latency, storage | **Ablation ladder, control arms** |
| **College Application Agent**: LangGraph Writer–Critic loop | **Proposer → checks → audit → gate**, with a statistics engine as critic |
| **Sina Weibo**: scheduled ETL, metrics, alerting | **Nightly queue, outcome-matrix schema, morning report, drift sentinel** |
| **Music Genre**: K-means clustering | **Failure taxonomy** conditioning the proposer |

**Interview one-liner:** *"At Changsheng my AutoML agent screened strategy parameters in stages and refused to promote anything that only won the backtest. HarnessForge points that machine at an agent's own source: the harness is the hyper-parameter, Aider polyglot the backtest, a sealed held-out split the out-of-sample period."*

## 4. Stack Choices — and Why

### 4.1 Models

| Role | Model, file (placement: §8) | Why |
|---|---|---|
| **Inner loop** | **Qwen3.5-4B** (hybrid Gated DeltaNet + gated attention), Q4_K_M **2.74 GB** | Apache-2.0; fast on the weakest GPU; small per-slot state |
| **Proposer, 9B transfer** | **Qwen3.5-9B**, Q4_K_M **5.68 GB** (speed on 6 GB measured in S0a) | Same family, so 4B → 9B isolates scale; published TBLite anchor |
| **Fallback** | **Qwen3-8B** (8.2B, standard attention), Q4_K_M 5.03 GB | Only if Qwen3.5 tool templates or caching misbehave |
| **API proposer arm** | mid-tier API model (candidates: Claude Haiku 4.5, GPT-5-mini) | Proposer only; hard spend cap |

Rejected: gpt-oss-20b (on 6 GB VRAM most experts sit in RAM, crowding out Docker); Qwen3.6-35B-A3B (too large). Qwen3.5 thinks by default; the seed turns it off, so when to think is a searchable choice paid from the same token budget. *Confirm cards and sizes in S0.*

### 4.2 Runtime and Libraries
**llama.cpp `llama-server`** serves the same GGUF files on both machines (OpenAI-compatible endpoint, per-slot prompt caching, `--metrics`). MLX is faster on M-series, but its quantization kernels would make Mac and Windows rows non-comparable. Seed actions are text-parsed; native tool calls are a mutation, since Qwen3.5 tool calling in llama.cpp depends on chat template and build (community templates exist because calls sometimes surface as raw text).

**Libraries:** Docker (Desktop + WSL2; Aider's README requires it because the benchmark runs model-written code unreviewed) · SQLite (write-ahead log) · git worktrees · MLflow · `ruff`, `pytest`, Python `ast` · `numpy`/`scipy` · `aider` (Apache-2.0, reference harness) · GEPA (`gepa`, MIT) · Harbor.

## 5. Benchmarks & Datasets

### 5.1 Aider polyglot (primary, fully local)
**225 Exercism exercises** (C++ 26, Go 39, Java 47, JavaScript 49, Python 34, Rust 30) that ≤ 3 of 7 reference models solved. Each ships instructions, a stub, tests and a **reference solution** named by `files.example` in `.meta/config.json` (`.meta/example.*` for C++/Go/Python/Rust, `.meta/proof.ci.js` for JavaScript, `.meta/src/reference/java/*.java` for Java; 96 of 225 not at `example.*`, verified in the repo); workspaces are stripped (§7.1). Leaderboard *percent correct* is "pass rate 2" (a second attempt after seeing test output); DGM reports pass@1 without ground-truth test feedback. Not comparable: we log both, headline pass@2.

| Split | Tasks | Use |
|---|---|---|
| **S — search** | 100: Python 28, Go 32, JavaScript 40 (stratified, seeded) | Everything the forge sees |
| **H — held-out** | 125: the other 22 Py/Go/JS + all 103 C++/Java/Rust | Final claims and gap curve; **sealed until S4** (§7.6). Reported as H-in-language (n = 22, underpowered) and H-cross-language (n = 103) |

**Pooled H and the compiled floor.** Py/Go/JS hold only 122 tasks, so most of H tests cross-language transfer (all six languages, n = 125); mutations helping only S languages dilute the pooled lift (accepted; shown per language). A near-zero compiled floor would make pooled H mostly noise, so S0b runs the seed on a 40-task H sample (11 each of C++/Java/Rust + 7 Py/Go/JS; aggregates only). **Rule:** if < 3 of the 33 compiled tasks pass in either seed, the split is redrawn before `prereg-v1` as S = 61 / H = 61 Py/Go/JS; compiled languages become descriptive and the MDE is restated.

**Published baselines** (% correct, pass rate 2; leaderboard updated 2025-11-20; no sub-10B model listed, checked):

| Model | % | Format; note |
|---|---|---|
| Qwen3 32B | 40.0 | diff |
| **gpt-4.1-mini** | **32.4** | diff, $1.99; **S0 reproduction target** |
| Qwen2.5-Coder-32B-Instruct | 16.4 | whole |
| gpt-4.1-nano | 8.9 | whole, $0.43; retirement: §10 |
| gemma-3-27b-it | 4.9 | whole |

The Qwen3-4B-Instruct-2507 card's 12.9 (Qwen3-4B non-thinking 13.8; harness and attempts unstated; it lists gpt-4.1-nano at 9.8 vs. the leaderboard's 8.9) is a floor-gate prior only. Qwen3.5-4B/9B are measured in S0 (seed and Aider harnesses).

### 5.2 Transfer

| Benchmark | Local setup | Published anchors |
|---|---|---|
| **OpenThoughts-TBLite**: 100 Terminal-Bench tasks (40 easy / 26 medium / 26 hard / 8 extreme), r = 0.911 with Terminal-Bench 2 | Harbor + Docker via WSL2 (verify in S0); TBLite-30; Qwen3.5-4B; harness unchanged | **Qwen3.5-4B 31.8 ± 3.8** (Tmax harness; the anchor 02 Shellwright reproduces); Qwen3.5-9B 41.9 ± 2.7 (Tmax); Ai2's OpenThoughts-Agent fine-tune 53.0 ± 0.7; Tmax-9B (reinforcement learning) 57.2 ± 2.5 |
| **SWE-bench Verified Mini** (optional, ledger slack only): 50 instances | ~5 GB of images (EvalScope docs); x86, Windows only | Small-model baselines unknown |

- **TBLite-30** = 02 Shellwright's 30-task TBLite-laptop slice: difficulty-stratified, resource-filtered from each `task.toml`, whose turn and timeout budgets apply (same filter here if unpublished).
- Ai2's card calls the Tmax harness "similar to mini-swe-agent" and does not define its ±; other-harness full-set numbers are plausibility anchors only.
- **Cost:** the card's 51–84 min per 100 tasks on 32 concurrent cloud sandboxes (API models) implies ≤ ≈ 16–27 sandbox-minutes per task: up to ≈ 4–7 h per TBLite-30 pass at concurrency 2 (S0b times 5 tasks).

### 5.3 Training and Demo Data
None: weights never change; the forge learns from its outcome matrix and failure traces on S. Demo: the archive (lineage tree, morning reports, seed-vs-champion trajectories on one exercise).

## 6. Architecture

*`split` profile (both machines over LAN, §8.1). In default `portable` the 9B swaps onto the Windows GPU between generations.*

```
 MAC (16 GB, Metal)                WINDOWS (16 GB RAM, 6 GB VRAM)
 llama-server Qwen3.5-9B           llama-server Qwen3.5-4B
 role: proposer                    inner loop, 2 slots x 24k, /metrics
        ^ LAN                                ^
 =======|====== FORGE (mutable, no Docker) ==|=====================
 forge-controller                            |
  |- archive.db: variants, lineage, diffs, outcome matrix, cost
  |- parents: sigmoid(score) x 1/(1+children)
  |- proposer client (local 9B | capped API) -> hypothesis + diff
  |- git worktree per child, no network; checks, AST audit
  '- halving 10 -> +30 -> rest of S; gate -> fresh-seed confirm
        | job (variant, task, seed) -> queue |
 =======|==== EVALUATOR (immutable, own OS user, owns Docker) ======
 queue -> agent container (fresh per job)    |
            stub files only (tests, .meta/ stripped)
            harness mounted read-only        |
            network: model gateway only -----'
              (token + wall-clock budgets, cost meter)
       -> grader container (fresh, network off): solution files
            + original tests, Aider unskip rules, test-count check
            -> row (pass@1/2, tokens, s, failure class)
               S rows: forge-readable | H rows: sealed table

 MLflow per child | morning report (S only) | audit log
 Morning: human audit veto.  H, 9B, TBLite: S4 only
```

The forge owns what changes, the evaluator what judges (§7.1). Checks and the abstract-syntax-tree (AST) audit reject bad children before they cost GPU time; the gate promotes mechanically (§7.6). In `split` the Mac queues ready children, so the Windows GPU only evaluates.

## 7. Component Design

### 7.1 Task Protocol and Immutable Evaluator
- **Workspace:** instructions + `files.solution` stubs only; no tests, `.meta/` or test-build outputs. A CI **canary** (hostile harness) hunting `.meta/**` and every `files.example`/`files.test` path must find none.
- **Budgets belong to the task:** per-attempt generated-token cap (gateway; below per-slot context, §8.2) and wall-clock cap (runner), from S0 timings, frozen at pre-registration. Turns, retries and thinking are harness choices: spend the budget better, never more.
- **Two attempts (Aider-compatible):** attempt 2 keeps the workspace plus `feedback.txt` (test output, truncated identically for all arms).
- **Offline toolchains:** images derive from Aider's benchmark image (npm packages in `/npm-install`, no Gradle step). The build pre-fetches Java's Gradle 8.7 wrapper and JUnit/AssertJ into `GRADLE_USER_HOME` (`./gradlew test` on every Java reference solution); Gradle runs `--offline`. CI asserts all 225 reference solutions pass offline.
- **Grader** (fresh container, network off) alone scores. It copies only declared solution files next to the original tests (a planted `conftest.py` never runs) and reproduces Aider's `run_unit_tests`: Rust `cargo test -- --include-ignored`, Java `@Disabled` stripped, JavaScript `xtest(` → `test(` (`npm-test.sh`), 180 s timeout. A **test-count check** (executed = cases in the unskipped file, cross-checked with `tests.toml` and a `files.example` run) fails skipped or short-circuited tests even at exit 0; a grader unit test fails if any skip marker survives.
- **Row:** `(variant, task, seed, attempt, pass, tokens, cached_tokens, wall_s, failure_class, machine, model_sha)`; pairs only within one `(machine, model_sha)`. H rows go to an evaluator-only table that the forge, the morning report and the human cannot read before S4.
- **OS boundary:** the evaluator is its own OS user. The forge gets read-only access, an evaluator-owned job queue and no Docker. `docker-users` members control Docker Desktop and can bind-mount evaluator files, so the forge stays outside that group (or is a WSL2 user without the Docker socket). A CI canary asserts it cannot `docker run`.

### 7.2 Inner Agent Loop (Seed Harness)
A mini-swe-agent-style loop (that agent: bash-only, ~100 lines) plus file writes, as `run(task, llm, env)`. Mutable: `prompts/`, `loop.py`, `context.py`, `actions.py`, `edit_format.py`. Read-only: gateway client, sandbox API.

| Element | Seed definition |
|---|---|
| Observation | Turn 0: system prompt, instructions, file listing, stubs, language and commands, remaining tokens (+ `feedback.txt` on attempt 2); later: last exit code + last 2,000 chars of output, or a parse error |
| Context | Full history to the token cap; ≤ N output tokens per turn (N = longest whole-file write in S0a, frozen in `prereg-v1`) |
| Actions | One fenced block per turn: `bash`, `write <path>` (whole file) or `SUBMIT` |
| Tools | bash + writes in `/workspace`; per-language command timeouts (S0a) inside the wall-clock cap, e.g. 180 s (Aider's test timeout) for Java/C++, whose cold JVM/Gradle and CMake runs can exceed 60 s. Compilers and offline caches present, tests absent: verifying means writing one's own tests |
| Termination | `SUBMIT` · 20 turns · token cap · wall-clock cap |
| Sampling | Thinking off; card's non-thinking preset (temperature 0.7, top_p 0.8, top_k 20, min_p 0) but **presence_penalty 0.0**: 1.5 penalizes repeated identifiers in whole-file writes, so 1.5 is a mutation (the card has no non-thinking coding preset; its thinking-mode one uses 0.0). In `prereg-v1`; seed via gateway |

### 7.3 Seeded Mutation Library
Fourteen unit-tested diffs, written in S1 so night 1 is not wasted:

| Area | Diffs |
|---|---|
| Edit format (2) | search/replace; unified diff with whole-file fallback |
| Verification (2) | compile/syntax check after each write; self-test from the instructions' examples before `SUBMIT` |
| Context (2) | head/tail output truncation; old turns summarized |
| Retry, planning, feedback, prompt (1 each) | format reminder on parse error; requirements checklist first; attempt 2 shows only failing test names and first assertion diff; language conventions |
| Sampling (2) | thinking on the first call only; presence penalty 1.5 (the card's general preset) |
| Tools (2) | `view`/`grep`; native tool calls |

### 7.4 Proposer
Inputs within 32k (lengths measured in S0a): parent source 16k (untouchable files cut to signatures), library diffs as examples 4k (truncated first), failure summary 6k (failure-class histogram, per-language pass rate, three failed trajectories), lineage hypotheses and outcomes 2k, output 4k. **Redacted** trajectories keep failing test names and error class but drop expected values, literal outputs and assertion diffs, so hidden-test content never becomes a copyable string. Output: a ≤ 3-sentence hypothesis plus a unified diff. A failed `git apply` or check gets one repair, then the child is recorded invalid (counted, never hidden).

### 7.5 Archive, Selection, Successive Halving
Every valid child that finishes rung 0 stays in the archive (as in DGM). At 10–30% scores DGM's parent weight is in the sigmoid's exponential tail: a 10-point gap ≈ 2.5–2.6× the weight. One generation, halving factor η = 3:

```
generation g (rng seeded by g):
  parents  = 2 draws, DGM weight sigmoid(10(α − 0.5)) × 1/(1 + n_children)
  children = 3 proposals per parent -> git apply, checks, AST audit
             (invalid or quarantined: logged, never hidden)
  rung 0   = one shared draw of 10 S tasks; every valid child runs it;
             rank by passes, ties -> fewer generated tokens
  rung 1   = top 2 run one shared draw of 30 more S tasks (40 total)
  rung 2   = the best ("promoted child") runs the other 60 (100 total)
  gate     = §7.6 vs. the champion (champion: all its seeds averaged;
             child: its 1 seed)
  if gated: confirmation (all of S, fresh seed), then renamed-task canary
```

- **Cost:** 3 × 60 = 180 task-runs = 1.8 full-S evaluations; a gated child adds 1.0 (confirmation) + 0.2 (canary): worst case 3.0.
- **Adaptive tasks:** at a low base rate most tasks fail for every variant and carry no information. So rung-0/1 draws are sampled ∝ √(p̂ₜ(1 − p̂ₜ)) + ε from archive history and scored with Horvitz–Thompson estimates. The idea is Task-CoEvolve's, tested there on text classification and Terminal-Bench 2.1, not code.
- **Two pre-registered full-evaluation generations** run every valid child on all of S × 2 seeds, after budget B (§7.8) so the main arm at B is standard. They yield an unbiased ≈ 12 children × 100 tasks × 2 seeds matrix for the claim-4 replay. Selection still uses only rows halving would have produced; seed 2 doubles as confirmation.
- **Termination:** night budget spent, or K = 6 generations without a new champion.

### 7.6 Promotion Gate and Calibration
**Champion change** needs: (1) one-sided 95% lower bound of Δ(child − champion) on S > 0 (paired bootstrap, 10,000 resamples, clustered by task, champion seeds averaged); (2) Δ > 0 on a fresh-seed **confirmation run** over S (winner's-curse check); (3) a renamed-task canary within the A/A band (§7.7).

**H never feeds a decision.** The final champion is by rule the main arm's last gated one (each control's: its last gated champion at B); the human veto covers audit findings only; H is unsealed once, in S4, after `prereg-v2`.

**Pipeline calibration.** A real promotion is the best of 6 children after halving, which inflates false promotions, so the whole pipeline is calibrated:
- S0b runs the seed with 8 seeds on S. Each **null generation** randomly assigns those runs to 6 children, champion and confirmation seed, then replays rungs, halving, gate and confirmation from the stored matrix with task resampling.
- 1,000 null generations give the per-generation false-promotion rate with a Clopper–Pearson 95% CI (optimistic: all share 8 runs, the real unit).
- **Rule:** calibrated if the upper bound ≤ 10%; else the bootstrap bound moves to α/6 (Bonferroni over 6 children) before night 1 and the replay re-runs.
- Four **paraphrase near-nulls** (meaning-preserving rewordings) are reported apart and never tune the bound: prompt sensitivity is the thesis.

### 7.7 Reward-Hacking Defenses
DGM's authors report their system removed the markers that detect hallucinated tool use and faked a log of passing tests; HarnessForge assumes the same. On top of the §7.1 evaluator and redaction (§7.4):
- A scan for test-framework internals, `sys.modules` edits and early exits catches import-time patching.
- The **AST audit** quarantines diffs with network use, paths outside `/workspace`, environment reads, gateway-client edits, string literals matching S exercise names, or 8+-character literals found in S test files.
- A **renamed-task canary** re-runs each new champion on 20 S tasks with renamed exercises and paths; a drop outside the A/A band quarantines it (keying on exercise identity is fitting S, not coding).

06 Ctrl-Z covers agents acting on real state; here every action hits a throwaway container.

### 7.8 Controls (Matched Budget)
**Budget B** = the main arm's GPU-hours over its first 3 search nights (Windows wall-clock, one frozen configuration). Arms run one at a time; tokens are reported too. Results are per GPU-hour and per evaluated child: arms without a local proposer (random, API) skip the 9B swap and evaluate more children per hour.

| Arm | Definition |
|---|---|
| (a) Random | Library mutations: does failure conditioning matter? |
| (b) Hand-tuned | Mentee, 6-hour time box, S evaluations capped at B (hours reported), git-tagged **before the first morning report is read** |
| (c) GEPA | External prompt-only baseline (local 9B reflecting); differs in search space *and* engine, so it separates neither |
| (c′) Forge on `prompts/` | Same-engine search-space ablation; slack only |
| (d) API proposer | USD 15 cap (USD 5 kept for S0); the agent stays the local 4B |
| (e) Greedy | Champion-only hill-climb (DGM's key ablation); optional, slack only |
| Reference | Aider's harness on the same 4B |

**Search variance:** one search per arm at B by default, so claim 2's task-bootstrap CI omits run-to-run search variance (HarnessOpt-Bench reports gains varying substantially across seed regimes). With slack, main and random arms re-run with a second search seed and report the spread.

### 7.9 Morning Report
Markdown and plots, **S only**: GPU-hours and ledger burn vs. plan (§8.4), proposals and valid share, champion changes (hypothesis, diff, confirmation Δ, canary), audit flags, tokens, dollars, "H sealed until S4". A **drift sentinel** ignores pass rate (10 tasks at ~10% give about one pass a night). It tracks tokens/s, prefill time and cache-hit rate from `llama-server --metrics`, plus exact match of temperature-0 completions on 5 fixed prompts on one idle slot. A shift flags throttling or driver drift.

## 8. Hardware Profiles

### 8.1 Profiles and Placement

| | `portable` (default) | `split` (opt-in) |
|---|---|---|
| Machines | One: 6 GB Windows laptop or the Mac alone | Both, over LAN |
| Inner loop | Qwen3.5-4B Q4_K_M | Same, so results stay comparable |
| Proposer | 9B swapped in between generations (partial offload on 6 GB) | 9B on the Mac, concurrent |
| 9B transfer (S4) | Windows partial offload, or the Mac alone if the arm64 image passes CI | Mac serves the 9B (`llama-server -c 24576 --parallel 1`, no Docker); Windows runs the containers |

Profile names follow 02 Shellwright; HarnessForge has no separate 8 GB profile: an 8 GB card only lets the 9B run at 32k, 1 slot (confirm in S0). Pre-registered numbers use `portable` settings. **Windows** owns evaluation, Docker and H/TBLite/SWE-bench runs (x86 images); **the Mac** owns proposing, reports and development. Mac 9B-transfer rows pair only with each other (seed-9B vs. champion-9B, same layout); if S0b's hours per 9B H pass exceed the S4 ledger, the transfer drops to 1 seed (pre-registered). Mac-alone evaluation needs the polyglot image to pass on arm64 (Aider's Dockerfile detects aarch64 for Go; the rest is unverified).

### 8.2 Memory Budget
Both Qwen3.5 models have 8 full-attention layers × 4 key-value (KV) heads × head_dim 256 (`config.json`): **32 KiB per token at f16** (1 GiB per 32k slot), plus ≈ 50 MiB fixed state per slot from the 24 Gated DeltaNet layers. llama-server splits `-c` across slots unless `--kv-unified`; the per-slot context printed at startup is logged.

| Component | Windows (16 GB / 6 GB VRAM) | Mac (16 GB unified) |
|---|---|---|
| 4B Q4_K_M + slots | 2.74 GB + state; default `-c 49152 --parallel 2` (24k/slot) ≈ 1.6 GB; `--parallel 4` at 24k/slot needs `-ctk q8_0 -ctv q8_0` (flash attention) ≈ 1.8 GB; compute buffers measured in S0a | same, only if evaluating |
| 9B Q4_K_M + 1 slot | does not fit fully: partial offload, proposer only | 5.68 GB + ≈ 0.8 GB (24k, transfer) or ≈ 1.1 GB (32k, proposer) |
| Docker VM | ≤ 6 GB (`.wslconfig`) | none in LAN layout; ≤ 3 GB if evaluating alone |
| Exercise containers | 2 Py/Go/JS (2–4 with `--parallel 4`); 2 C++/Java/Rust | arm64: verify in S0b; C++/Java/Rust at 1 |
| Controller, SQLite, MLflow; OS | < 1 GB; ~3–4 GB | < 0.5 GB; ~3–4 GB |

**Context per slot ≥ p95 trajectory.** Turn 0 is ≈ 2–3k tokens. Twenty turns of ≤ 2,000-char observations plus whole-file writes may reach ≈ 16–20k (p95 measured in S0a); the token cap sits below it. macOS caps GPU-wired memory below total RAM: read `recommendedMaxWorkingSetSize` on the actual Mac.

### 8.3 S0a Go/No-Go Gates (Session 2, End of Week 2)
On the 6 GB laptop, `portable`: seed harness, Qwen3.5-4B, 30 S tasks × 2 seeds, plus a timed generation skeleton (9B swap-in, 6 proposer calls at real prompt length with repairs, rungs projected from measured s/task).

| Gate | GO if | Else |
|---|---|---|
| **Latency** | A gated generation (1.8 + 1.2 full-S evaluations, §7.5, plus proposer and swap time) projects to **≤ 4.5 h** (two per 9-hour night); context per slot ≥ p95 trajectory | Rungs on pass@1 (rung 2, gate, H stay pass@2); lower token cap; thinking off; `--parallel 4` with q8_0 KV; S cut to 80; then Qwen3.5-2B (re-check floor) |
| **Floor** | Seed pass@2 **≥ 5% with ≥ 3 distinct passing tasks** (1 pass in 20 runs has a Clopper–Pearson 95% CI of 0.1–24.9%) | Seed with Aider's harness, or a 9B inner loop |

**Order:** runner → gates → A/A + power → `prereg-v1`. A/A runs, near-nulls, base rates, H sample and reproduction wait for GO (sessions 3–4): measured once, on the final configuration; either constraint surfaces in week 2, not week 10.

### 8.4 Compute Ledger
**Units:** E = one full-S evaluation (100 tasks, pass@2), target ≤ 1.5 h; E_H = one H pass (125 tasks, 103 C++/Java/Rust at concurrency 2), ≥ 1.25 E ≈ 1.9 h before compiled-language slowdown; T = one TBLite-30 pass, up to ≈ 4–7 h (§5.2). S0b measures E_H and T. A generation is 1.8 E (≈ 2.7 h) plus proposer time, + 1.2 E (≈ 1.8 h) if gated. Available: ≈ 9 nights × 9 h ≈ 81 h per two-session sprint on a 10-week calendar; ≈ 19 nights each for S0 and S1.

| Sprint | Item | Units | Hours |
|---|---|---|---|
| S0a | Gate run (30 tasks × 2 seeds) + generation skeleton | 0.6 E + proposer | ≈ 2–3 |
| S0b | 8 A/A seed runs; 4 paraphrase near-nulls; Aider's harness on the 4B | 13 E | ≈ 20 |
| S0b | 9B floor (30 S tasks); H sample (40 × 2 seeds); 5 TBLite tasks | — | measured |
| S1 | Hand-tuned control; shakedown nights | ≤ B | ≤ 27; rest slack |
| S2 | Main arm to B (3 nights); 2 full-evaluation generations (2 × 12 E); normal nights | B + 24 E + rest | 27 + 36 + 18 = 81 |
| S3 | Random, GEPA, API proposer at B; replay study (CPU) | 3 B | 81 (no slack) |
| S4 | H: seed + final champion × 3 seeds; champion at B + 4 controls × 2; Aider × 1; ≤ 4 lineage champions × 1 | 6 + 10 + 1 + 4 = 21 E_H | ≥ 40 |
| S4 | 9B transfer: seed + final champion × 2 seeds on H | 4 E_H (9B) | measured |
| S4 | TBLite-30: seed + final champion × 2 seeds | 4 T | up to ≈ 16–28 |

**Pre-registered cut order** if a sprint's measured sum exceeds its nights: (1) optional arms (greedy, (c′), second search runs, SWE-bench Verified Mini); (2) lineage gap points beyond the final champion, and champion at B; (3) 9B transfer to 1 seed; (4) TBLite-30 to 1 seed, descriptive. Never cut: claim 1 (3 seeds) and claim-2 controls (2 seeds). S1's spare nights and daytime runs are the only slack; a failed night re-runs from it, not from another arm's budget.

## 9. Evaluation Plan

`prereg-v1` is tagged before night 1 (after the S0a gates); `prereg-v2` freezes forge code at the end of S3, before H is unsealed. Δ is always vs. the seed harness unless stated.

| Test (claim) | n | α / power | Decision rule; metrics |
|---|---|---|---|
| **H lift (1)** | 125 tasks × 3 seeds | one-sided 0.05 / 0.8 | H pass@2 (primary): lift if the one-sided 95% lower bound of Δ > 0 (language-stratified paired cluster bootstrap). Also pass@1, raw Δ, HarnessOpt-Bench's normalized gain; Aider's harness on the 4B as a 1-seed reference |
| Harness vs. scale (1) | H | — | Evolved 4B vs. seed 9B: exploratory, CI only |
| **Controls (2)** | 125 × 2 seeds | Holm over 3 | Same rule vs. random, hand-tuned and GEPA, Holm-adjusted; MDE restated for 2-seed averages; reported as in §7.8 |
| Proposer (2) | H at B | — | Local 9B vs. API: valid-diff rate, promotions per proposal |
| Overfitting (3) | lineage | — | S − H gap (computed in S4); shrinkage promotion Δ → confirmation Δ → H Δ |
| **9B transfer (3)** | 125 × 2 seeds | one-sided 0.1 | "Model-general" if the lower 90% bound of Δ > 0 |
| **TBLite-30 (3)** | 30 × 2 seeds | non-inferiority; power 0.8 at Δ = 0 | **PASS** if the lower 90% bound of Δ > −δ, δ = (z₀.₉ + z₀.₈)·√(σ²_d / 30) from S0b (≈ 15 points at σ²_d = 0.15; a −10-point margin would pass a zero-effect harness only ≈ 55% of the time). **Not testable** if the seed's mean pass count < 5 of 30. Realized power reported |
| **Adaptive validation (4)** | 2 full-evaluation generations (n = 2 is the replication unit), 1,000 replays | — | Succeeds if it picks the full-set winner or a tie-set member (within the A/A 95% band of the best) with probability ≥ 0.9 using ≤ 50% of uniform evaluations. Reports savings with a CI, Kendall τ, regret vs. uniform/fixed subsets, η ∈ {2, 3, 4} |
| **Gate calibration (4)** | 8 seed runs, 1,000 null generations | α = 0.05 | Upper bound ≤ 10%, else α/6 (§7.6) |
| Cost, safety | all runs | — | s, tokens, cache hits per task; audit flags per 100 proposals; canary drops |

**Ablation ladder** (*replay* = no GPU): seed → random mutations (a) → failure-conditioned proposer → archive vs. greedy (e) → successive halving (*replay*) → variance-weighted tasks (*replay*) → gate vs. "highest mean wins" (*replay*).

**How n was chosen.** MDE = (z₀.₉₅ + z₀.₈)·√(σ²_d / n), the effect claim 1 is powered for (not a threshold); σ²_d = per-task variance of the seed-averaged paired difference.
- σ²_d: S0b's A/A runs on S (rescaled to 3-seed averages), cross-checked on the 40-task H sample's seed-only A/A pair (aggregate only; drives no harness choice). The larger is pre-registered, as 82% of H is C++/Java/Rust.
- *Illustration:* σ²_d = 0.15 gives MDE ≈ 9.6 points at n = 100, ≈ 8.6 at n = 125, ≈ 20.5 at H-in-language's n = 22 (hence secondary).
- At a 20% pass rate an unpaired 10-task rung has a 12.6-point standard error, a 40-task rung 6.3: rungs only filter, never decide.
- If S0 implies MDE > 12 points, H seeds are added if the ledger allows; else the larger MDE is stated.

**Honest reporting:** search-split scores are never results; every proposal is counted (invalid, screened, promoted, quarantined); every number carries n, seeds and a CI; DGM, Meta-Harness and AutoSaddler numbers are context only; nulls are published with their power.

## 10. Milestones (19 hours / 15 sessions)

| Sprint | Sessions | Deliverable |
|---|---|---|
| **S0a** | 1–2 | Scaffold, llama-server (CUDA + Metal), offline Docker runner, canaries, **immutable grader** (§7.1), seed harness; **go/no-go at session 2, end of week 2** (§8.3) |
| **S0b** | 3–4 | After GO: S0b runs (§8.4), pipeline calibration, power analysis, frozen budgets, `prereg-v1`. **External reproduction** by day (API + CPU containers): Aider's harness + gpt-4.1-mini on all 225 tasks through this runner. Hard pass: our grader re-grades those outputs with 100% agreement. Sanity range: ±2√2 standard errors (≈ ±8.8 pts) of the published **32.4%** (≈ USD 2), as both runs are noisy |
| **S1** | 5–8 | Mutation library (14 diffs); archive, proposer, checks, audit, halving, gate, gateway, report; hand-tuned control frozen. **Week-6 demo:** first overnight run and morning report |
| **S2** | 9–10 | Main arm: budget B (first 3 nights), both full-evaluation generations, normal nights |
| **S3** | 11–12 | Controls at B (3 nights each); optional arms only with slack; replay study (CPU); `prereg-v2` |
| **S4** | 13–14 | H unsealed once: claims 1–3, lineage gap curve, 9B and TBLite-30 transfer, optional SWE-bench Verified Mini, audit report, dashboards, every pre-registered test |
| — | 15 | Interview sprint |

The reproduction validates runner and grader, not the method. Secondary reports put API removal of gpt-4.1-mini at 2027-01-14 and of gpt-4.1-nano at 2026-10-23 (so nano is no fallback); the mini run must finish before 2027-01-07 (S0 ends around week 3–4). Fallback: a leaderboard open-weight row via a hosted provider, e.g. Qwen3 32B (40.0%, diff), within the USD 5 S0 cap once provider and price are confirmed.

## 11. Risks & Mitigations

| Risk | Mitigation |
|---|---|
| Noise exceeds gains | Power-sized rung 2, confirmation, calibrated gate (§7.6); MDE stated up front, so "no detectable lift" is valid |
| Weak 9B proposer | Library examples, one repair; API arm (§7.8) |
| Reward hacking | Hidden tests, test-count check, AST audit, redacted proposer inputs, canaries (§7.1, §7.7); forge outside `docker-users` |
| Human peeking at H | Evaluator-only H table, S-only reports, one unsealing (§7.1, §7.6) |
| Compute overrun | Measured ledger, pre-registered cut order, nightly burn (§8.4) |
| Compile time; 4B floor on C++/Java/Rust | Compiled languages only in H, lower concurrency; offline Gradle cache, per-language timeouts (§7.1); H-cross-language reported apart; re-split rule (§5.1) |
| Exercism contamination | Paired tests cancel a shared boost; TBLite and SWE-bench Verified Mini are the real checks |
| llama.cpp #24587 (closed "not planned"): context checkpoints invalidated on sliding-window attention (SWA) and hybrid models such as Qwen3.5 when mid-prompt content changes | Context-rewriting mutations pay re-prefill in measured hours, so search prices it in; cache-hit rate logged; Qwen3-8B fallback |
| Qwen3.5 tool-template quirks | Text-parsed actions; native tool calls only as a mutation |
| Contaminated hand-tuned control; drift across nights | Tagged before the first report; fixed power plan; drift sentinel; rows tagged by machine and model hash |
| API model retirement | gpt-4.1-mini reproduction before 2027-01-07; hosted open-weight fallback (§10) |
| Polyglot image fails on arm64 | Mac stays proposer-only; 9B transfer uses the LAN layout |
| Crowded prior art; null result | Cited up front; the contribution is laptop scale, matched controls, a calibrated gate; a powered null is a deliverable |

## 12. Résumé Bullets (experience voice; numbers filled in as sprints land)

- Built a **fully local self-improving coding agent** that evolves its own harness (prompts, tools, context policy, edit format, verification) overnight on a 6 GB laptop GPU, lifting Qwen3.5-4B's held-out Aider-polyglot pass rate from **X% to X%** (paired bootstrap p = X; 125 tasks × 3 seeds).
- Designed **statistically gated promotion** (successive halving, variance-weighted sampling, paired-bootstrap gates, fresh-seed confirmation): **X%** false promotions on null generations; **X%** fewer evaluations to find the winning harness via adaptive task selection.
- Beat **matched-GPU-hour controls** (random +X pts, hand-tuned +X, GEPA +X); API-proposer arm for **USD X**; search-vs-held-out gap X points; transfer to Qwen3.5-9B (+X) and TBLite (pre-registered PASS/FAIL/not testable).
- Built a **tamper-resistant evaluator** (hidden tests, stripped reference solutions, test-count checks, AST audits) that flagged **X of X** proposals, validated by reproducing a leaderboard number (X% vs. 32.4%).

**Rule: the verb follows the measured sign.** A null gets a null bullet, e.g. *"Showed with a pre-registered, powered test (MDE X pts) that evolved harnesses did not beat random mutation at matched GPU-hours on 125 held-out tasks."*

### Placement Note
List these under EXPERIENCE only for (1) a real engagement with a truthful title and record; (2) a real collaboration with a lab, startup or open-source organization that agrees to host the work; or (3) an open-source project entry with real commit history, morning reports and the pre-registration tag. Never invent an employer or dates: North American employers verify employment, and fabricated entries get offers rescinded. A public repo whose pre-registration tag predates its results beats an unverifiable internship line.

## 13. Why This Reads as Real Company Work

- **Harness engineering is the job:** teams change prompts, tools, edit formats and context policy far more often than weights; this automates that loop, with a cost meter.
- **False promotions are production failures:** a prompt change that "won" on 50 examples, then regressed, is familiar; a gate with a *measured* false-promotion rate fixes it.
- **Evaluators get attacked** once an optimizer is rewarded by a metric: hence immutable graders, hidden tests, audit logs.
- **Checkable discipline:** matched compute, a dollar cap, a pre-registration tag dated before results. Automation proposes, statistics decides, a person audits.

## 14. Interview Leverage

- **ML system design:** an evaluation-and-promotion platform for agent changes.
- **Statistics:** paired binary outcomes, cluster bootstrap, Holm, winner's curse, A/A calibration, non-inferiority.
- **LLM / agent engineering:** edit formats for small models, truncation vs. cache reuse on hybrid attention, thinking budgets, harness vs. model size.
- **AutoML / SDE:** successive halving, archive vs. greedy, Docker isolation, OS-user boundaries, crash-safe overnight jobs.
- **Safety and behavioral:** DGM's documented reward hacking and structural defenses; handling a null result; why pre-registration came first.

## 15. References

- DGM: https://arxiv.org/abs/2505.22954 · https://sakana.ai/dgm/ (self-improve prompts with o1 diagnosis: paper Appendix C.3)
- Harness optimization: Task-CoEvolve https://arxiv.org/abs/2608.20169 · Meta-Harness https://arxiv.org/abs/2603.28052 · AutoSaddler https://arxiv.org/abs/2608.23041 · PRISM https://arxiv.org/abs/2609.05736 · HarnessOpt-Bench https://arxiv.org/abs/2608.06301, https://labs.scale.com/papers/harnessopt-bench · "From Static Filters to Adaptive Gates" (NeurIPS 2026) https://neurips.cc/virtual/2026/169856
- GEPA: https://arxiv.org/abs/2507.19457 · https://github.com/gepa-ai/gepa
- Aider: leaderboard https://aider.chat/docs/leaderboards/ · polyglot https://aider.chat/2024/12/21/polyglot.html · exercises https://github.com/Aider-AI/polyglot-benchmark · README https://github.com/Aider-AI/aider/blob/main/benchmark/README.md · `run_unit_tests`, `pct_well_formed` https://github.com/Aider-AI/aider/blob/main/benchmark/benchmark.py · image https://github.com/Aider-AI/aider/blob/main/benchmark/Dockerfile
- Qwen cards: Qwen3-4B-Instruct-2507 https://huggingface.co/Qwen/Qwen3-4B-Instruct-2507 · Qwen3.5-4B (sampling presets, config) https://huggingface.co/Qwen/Qwen3.5-4B
- TBLite: Tmax numbers as cited in 02 Shellwright · https://huggingface.co/datasets/open-thoughts/OpenThoughts-TBLite · Ai2 Qwen3.5-9B card https://huggingface.co/allenai/qwen35-9b-openthoughts · Harbor https://harborframework.com/docs/running-tbench
- GGUF files: https://huggingface.co/unsloth/Qwen3.5-9B-GGUF · https://huggingface.co/unsloth/Qwen3.5-4B-GGUF · https://huggingface.co/Qwen/Qwen3-8B-GGUF
- llama.cpp #24587: https://github.com/ggml-org/llama.cpp/issues/24587 · mini-swe-agent: https://github.com/SWE-agent/mini-swe-agent · SWE-bench Verified Mini (EvalScope): https://evalscope.readthedocs.io/en/latest/benchmarks/swe_bench_verified_mini.html
- Successive halving https://arxiv.org/abs/1502.07943 · Hyperband https://arxiv.org/abs/1603.06560
- GPT-4.1 API removal (secondary): https://www.digitalapplied.com/blog/gpt-4o-gpt-4-1-api-shutdown-january-2027
