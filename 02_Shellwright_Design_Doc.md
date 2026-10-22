# Shellwright — Design Document

**A Small Terminal Agent Built From the Interface Up: Harness Attribution, ACI Ablation, and Per-Turn Distillation on a Laptop**

| | |
|---|---|
| Status | Proposal |
| Target duration | ~10–12 weeks (19 mentoring hours / 15 sessions + independent work) |
| Platforms | **Windows 11 laptop** (16 GB RAM, RTX 3060-class, 6 GB VRAM assumed, 8 GB if confirmed; Harbor + Docker in WSL2, the Windows Subsystem for Linux) and **Apple Silicon Mac** (16 GB unified, Metal). One repository. |
| Deployment | 100% local: llama.cpp serves the models, every task runs in local Docker. No rented servers or paid GPU cloud; a commercial API is only an optional baseline under a $20 hard cap. |
| Core stack | Harbor · Docker · llama.cpp (`llama-server`) · LiteLLM · Qwen3.5-9B / 4B / 2B / 0.8B (GGUF) · Unsloth + PEFT · Parquet + DuckDB · MLflow |
| Sprints | S0a–S4, with S0 = S0a + S0b (§10). P0–P4 are compute priorities (§9.4); R4-A, R4-B, R9 and R2 are rate classes (§8.3). |

---

## 1. The Product Thesis

Teams want small open-weight agents that work in a shell: fix a failing continuous-integration build, repair a service config, turn a messy CSV into a report. Small models are cheap, private and run beside the sandbox, but their published scores disagree (§5.2):

- On Terminal-Bench 2.1 (TB2.1), Qwen3.5-9B scores 16.1 ± 3.7 in Tmax's harness (Ai2's bash-tool `Vanillux2Agent`), 1.9 above the 4B (14.2), but 27.34 in FACET's Terminus-2 run. Neither is replicated; Tmax's ± is a standard error (SE) over 3 runs, so its 95% t-interval is about ±16. The 4B's two TB2.1 scores differ by only 3.4.
- The direction flips: within Tmax, the 9B spans 36.0–44.1 on OpenThoughts-TBLite across four harnesses, Terminus-2 second-lowest (36.4), yet Terminus-2 gives FACET's *higher* TB2.1 number. FACET: 2 h per attempt, temperature 1.0, 262k context for base models; Tmax: vLLM on an A100, 64 steps, 120 s per command.

**The harness name is not the variable; its configuration is** (§7.4), and a laptop can match it only in part.

> **Engineering thesis: small-model terminal failures are mostly interface failures, not knowledge failures.** The model drowns in a 10k-line build log, hangs on an interactive prompt, or submits without checking. So: measure what the harness alone moves, fix the interface, then distill the larger model's behavior inside it into a model small enough to ship.
>
> *Pre-registered test:* at rung 0 on the 4B (§9.2), at least 50% of failed episodes carry an interface label (`hang`, `format_error`, `truncation_loss`, `unverified_submit`; §7.5), or the thesis is reported as rejected. `timeout` and `turn_cap` shares are reported, not counted.

Four claims, each tested against a minimum detectable effect (MDE) pre-registered from Sprint-0 noise (§9.3):

1. **Harness sensitivity is measured, then attributed where large enough.** Endpoints A (Tmax-like) and B (FACET-like) run on the 9B (TB2.1) and the 4B (TBLite). If the 4B A−B paired 95% confidence interval (CI) excludes zero, a fractional factorial attributes it to four knob groups (§7.4); otherwise the gaps stand, with CIs. The 4B validates the method; the 11-point question rests on the 9B; if the 9B rule fails (§8.3), the claim narrows to 4B/TBLite.
2. **The interface is worth more than it costs.** A designed agent-computer interface (ACI) beats a Terminus-2-style baseline (rung 0, §9.2) on the 4B by ≥ MDE, each component costed in pass rate, tokens and wall-clock per solved task.
3. **Per-turn distillation fits a consumer GPU and moves a 2B student.** Qwen3.5-2B with bf16 low-rank adaptation (LoRA) beats its base in the same ACI by ≥ MDE on 300 held-out in-distribution (ID) tasks; TBLite is the out-of-distribution check. Only this project asks whether public traces re-rendered through our shaper and context manager (train/inference view parity) beat the same traces natively rendered (§7.6). Also reported: share of the 2B-to-9B gap closed; an untuned-4B comparison.
4. **Cost is a first-class axis.** pass@1, pass@3, pass^3 (§9.1) vs tokens and wall-clock per solved task, per machine profile; the shipped default comes from that Pareto frontier, not the top pass rate.

**Payoff:** a 1.3 GB GGUF (llama.cpp's model file; ~1.7 GB+ resident at 32k) that fixes a broken nginx config in a container.

## 2. Goals & Non-Goals

### Goals
- Claims 1–4 on one local, reproducible runner (Harbor + Docker + `llama-server`, one trace format) across both machines.
- **Reproduce one published number in Sprint 0**, before any claim (§9.1). Tmax releases no TBLite eval config (its repo targets Terminal-Bench 2.0 on Daytona, 5 attempts), so it is reconstructed, deviations listed.
- A failure taxonomy over every trace; a published **TBLite-laptop** subset and exclusion list.

### Non-Goals (sibling docs in bold)
- Reinforcement learning (RL; Tmax used H100 nodes): we test what laptop supervised fine-tuning (SFT) buys.
- Test-time search, process reward models (**01 Compass**); automated harness search (**07 HarnessForge**; our knobs follow hypotheses).
- Record/replay, counterfactual forking (**05 BlackBox**, which imports our traces); security beyond Docker isolation (**06 Ctrl-Z**).
- Distilling a context-compression step (**03 Spelunker**, which reuses our per-step trainer: segmenter, loss masking, view parity). Neither doc claims the generic per-step rejection-sampling recipe.
- A 20B-class local teacher: trace generation shares the 16 GB machine with the sandboxes.

## 3. Alignment With Prior Experience

| Prior experience | Skill | Shellwright module |
|---|---|---|
| **Sina Weibo**: scheduled SQL ETL, standardized metrics, alerting | Shell/data wrangling; metric pipelines | **Trace store and dashboard**: per-turn Parquet, failure counters, resource-kill alerts (TBLite's largest category: Data Processing & Scripting) |
| **CAS**: cascaded binary-encoding retrieval; Precision@K, latency, storage over 20+ configs | Config benchmarking | **Knob study and ACI ladder**; pass-rate / tokens / wall-clock Pareto table |
| **Changsheng**: AutoML hyper-parameter agent; backtests with promotion gates | Honest tuning | **Pre-registered MDEs**: a config ships only if it clears the noise floor |
| **College Application Agent**: LangGraph Writer–Critic loop behind FastAPI | Tool-using LLM loops | **Verify gate**: the critic becomes an enforced expected-vs-observed check before submit |
| **DiDi**: Java backend; A/B testing | Experiment design | **A/A noise runs, paired bootstrap, Holm correction** |

**Interview one-liner:** *"Two papers were 11 points apart on the same 9B model on Terminal-Bench. I rebuilt both harness configurations on a laptop, measured how much of the gap survives local serving, and [traced it to specific knob groups | showed it sits inside the noise at laptop scale]. Then I built an interface and a distilled 2B agent that ships as a 1.3 GB GGUF."*

## 4. Stack Choices — and Why

### 4.1 Models

Qwen3.5 small models are Apache-2.0, think by default, and are hybrid: three Gated DeltaNet layers per gated-attention layer, so one layer in four keeps a key-value (KV) cache. Sizes: `unsloth/*-GGUF`. f16 KV per token = attention layers × KV heads × head dim × 2 × 2 bytes (`config.json`).

| Role | Model | File | KV/token | Why |
|---|---|---|---|---|
| Headline agent, teacher | **Qwen3.5-9B** | Q4_K_M 5.68 GB | 32 KiB (8 × 4 × 256) | Two conflicting TB2.1 numbers |
| Default; ACI ablation | **Qwen3.5-4B** | Q4_K_M 2.74 GB; Q8_0 4.48 GB control (Mac, `split`) | 32 KiB | Anchors on both benchmarks and harness families; fits 6 GB |
| Student | **Qwen3.5-2B** | bf16 LoRA → Q4_K_M 1.28 GB | 12 KiB | ~5 GB LoRA (§8.2); base scores 5.7 on TBLite |
| Log summarizer (optional) | **Qwen3.5-0.8B** | Q8_0 0.81 GB | 12 KiB | Beside the 4B only if S0a shows it fits |
| Training fallback | Qwen3.5-0.8B (~3 GB) or **Qwen3-1.7B** QLoRA (quantized LoRA) | — | — | Unsloth advises against QLoRA on Qwen3.5 |
| Non-hybrid fallback | **Qwen3-4B-Instruct-2507** (Q4_K_M 2.5 GB), Qwen3-8B | — | 144 KiB (36 × 8 × 128) | If hybrid prefix caching fails; 4.5× the KV, so 16k or q8_0 KV on 6 GB |

### 4.2 Runtimes and tooling

| Concern | Choice | Why |
|---|---|---|
| Tasks | **Harbor**, local Docker | Official TB2.x/TBLite runner; `--agent-import-path module:Class` runs Terminus-2 (built in), Ai2's `Vanillux2Agent` (`hamishivi/tmax`) and ours on the same images and verifiers |
| Serving | **`llama-server`**: CUDA (Windows), Metal (macOS), same GGUF files | vLLM cannot hold a bf16 4B on 6 GB; identical bytes keep machines comparable; MLX only as an optional Mac comparison |
| Model API | **LiteLLM** (`openai/<name>` + `api_base`) | The form Harbor agents accept |
| Windows | Harbor + Docker in **WSL2** under a `.wslconfig` cap; `llama-server` native | Model memory stays outside the cap |
| Training | **Unsloth + PEFT bf16 LoRA in WSL2**; `transformers` v5 (required for Qwen3.5) | Qwen3.5 training needs custom Triton kernels (Linux-first) |
| Bookkeeping | `uv`; hashed Pydantic knob config; MLflow; NumPy bootstrap | Runs keyed by knob hash, model SHA-256, llama.cpp build, Harbor version, task set; never overwritten |

## 5. Benchmarks & Datasets

### 5.1 Measured benchmarks

| Benchmark | Size | Role | Local feasibility |
|---|---|---|---|
| **OpenThoughts-TBLite** (Apache-2.0) | 100 tasks, 9 categories; 40 easy / 26 medium / 26 hard / 8 extreme | All ablations; r = 0.911 with TB2, i.e. Terminal-Bench 2.0 (card) | Card runtimes are from 32 concurrent Daytona sandboxes. S0a parses each `task.toml` (`cpus`, `memory_mb`, timeouts) into **TBLite-laptop**, the tasks fitting the WSL cap; raw (exclusions as failures) and adjusted scores |
| **Terminal-Bench 2.1** | 89 tasks (2.0 with 28 fixed) | 9B endpoints; P4 finals; 3 attempts, exclusions published | TBLite runs 2.6–8.0× faster than TB2 (card) |
| **Dev slice** | 30 stratified `allenai/TMax-15K` tasks (S0a), disjoint from training and held-out data | Debugging, ACI thresholds; **never carries a claim** | Overnight-sized |
| **Held-out ID set** | 300 converted TMax-15K tasks outside all training pools | Claim 3 primary test, 1 attempt per arm | Converted in S2 (§5.3) |

TBLite and TB2.1 are touched only by configs frozen in a tagged commit before the first claim run; all tuning uses the dev slice.

### 5.2 Published baselines

± = Tmax's SE over 3 runs (2 degrees of freedom; §1). FACET reports 3-attempt means without spread; the TBLite card names no harness and reports TB2 (2.0).

| Model | Harness | TBLite | TB2.1 | Source |
|---|---|---|---|---|
| Qwen3.5-2B / Tmax-2B (RL) | Tmax | 5.7 ± 1.6 / 11.8 ± 1.4 | 1.9 ± 1.4 / 4.2 ± 1.2 | Tmax T3 |
| Qwen3.5-4B | Tmax | **31.8 ± 3.8** (S0b target) | 14.2 ± 2.3 | Tmax T3 |
| Qwen3.5-4B / FACET-4B | Terminus-2 | — | 17.60 / 24.72 | FACET T2 |
| Qwen3.5-9B | Tmax | 41.9 ± 2.7 | **16.1 ± 3.7** | Tmax T3 |
| Qwen3.5-9B | mini-SWE-agent / OpenHands | 44.1 ± 3.3 / 36.0 ± 2.8 | — | Tmax T5 |
| Qwen3.5-9B | Terminus-2 | 36.4 ± 2.2 (Tmax run) | **27.34** (FACET run) | Tmax T5 / FACET T2 |
| FACET-9B / Tmax-9B (RL) | Terminus-2 / Tmax | — / 57.2 ± 2.5 | 35.58 / 28.8 ± 1.4 | FACET T2 / Tmax T3 |
| Nemotron-3-Nano-30B-A3B | not stated | 21.5 ± 1.78 | TB2: 9.5 | TBLite card |
| gpt-5-mini | not stated | 50.5 ± 2.23 | TB2: 24.9 | TBLite card |

Compute-scale references, not targets: FACET-tuned models used full-parameter SFT on 8×H200 (~1.2K DeepSeek-V4-Pro trajectories); Tmax used DPPO (its RL algorithm) on H100 nodes.

### 5.3 Training data (decontaminated against both benchmarks)

| Source | Content | Teacher / format | License |
|---|---|---|---|
| `open-thoughts/OpenThoughts-Agent-SFT-10K` (1K–100K variants) | ≥5-turn traces: SWE-Smith, SuperUser, Tezos, IssueTasks | GLM-4.7-AWQ; Terminus-2 keystrokes | Apache-2.0 |
| `allenai/TMax-SFT-16.5K` | 2.2K rows over ~2K envs (card); 16.5K trajectories, 8K successful (paper §3.3) | Qwen3.6-27B; bash tool | ODC-BY* |
| **Self-RFT traces** | Verified 9B successes in our ACI (`split` on 6 GB; 4B if the 9B fails the gate) | native | ours |
| Self-RFT task pool | `allenai/TMax-15K` → Harbor (~14.6K Apptainer envs: paper 14,600, card 14,601 rows; pytest, fuzz-equivalence, HTTP/TCP checks); `OpenThoughts-Agent-v1-RL` (~720 nl2bash tasks) | — | ODC-BY* / per card |

Self-RFT = rejection-sampling fine-tuning (RFT) on verified successes. No frontier model is run or paid for: traces come from open-weight GLM-4.7, Qwen3.6-27B and our 9B. \*Contains Gemini 3.1 Pro outputs under Google's ToS; TMax environments were Gemini-generated (Gemini-3-Pro per the paper, Gemini 3.1 Pro per the cards). The write-up says so.

**Demo data:** eight hand-written Harbor ops tasks (e.g. broken nginx config, silent cron job, bad CSV delimiter, full-disk log rotation, dependency conflict); never used for a claim.

## 6. Architecture

```
Windows laptop, inside WSL2 Ubuntu (memory cap 6-8 GB):
  harbor run -d openthoughts-tblite --agent-import-path shellwright.agent:Agent

  Shellwright agent loop ------------ exec ----------> Task container (Docker)
   |- aci/shaper   ANSI, progress, head/tail, spill  instruction.md, env
   |- aci/tools    run view edit write notes submit  tests/ (run after
   |- aci/context  notes, compaction                 submit or timeout)
   |- aci/gate     verify then submit
   |- knobs/       typed config
   '- traces/      per-turn Parquet
            | LiteLLM, OpenAI-compatible; servers run outside WSL2
    +-------+---------------------+
  llama-server (Windows, CUDA)    llama-server (Mac, Metal)
  4B default; 9B with 8 GB VRAM   9B in "split" profile, LAN

Offline: traces -> labeller -> segmenter (2-4k windows) -> decontaminate
         -> Unsloth bf16 LoRA (2B) -> GGUF Q4_K_M -> same runner
Report:  MLflow -> paired bootstrap -> ablation and Pareto tables
```

The reward from Harbor's tests joins per-turn tokens and latency in the trace. Baselines differ only in the agent flag.

## 7. Component Design

### 7.1 Runner
`shellwright.agent:Agent` subclasses Harbor's `BaseAgent` (`setup`, `run(instruction, environment, context)`). An S0a smoke test runs the oracle and each agent on 5 tasks before any sweep.

### 7.2 The agent loop

| Element | Specification |
|---|---|
| Observation | Turn 0: system prompt, instruction, tool schemas. Then exit code, duration, `cwd`, shaped stdout/stderr, `truncated` flag, spill path |
| Actions | One native Qwen3.5 tool call per turn (`llama-server --jinja`); baselines keep Terminus-2 keystroke JSON or Vanillux2's bash tool |
| Tools | `run(cmd, timeout_s=120, background=false)` · `view(path, start, end)` (≤200 numbered lines) · `edit(path, search, replace)` (exact unique match, else an error with the match count) · `write(path, content)` · `notes(update)` (≤1,000 tokens) · `submit(check_cmd, expected, observed)` |
| Termination | Accepted `submit`; 64 turns (Tmax's step limit); `task.toml` agent timeout × profile multiplier *m* (Harbor's `agent_timeout_multiplier`, in the run hash); 5 consecutive unparseable actions |
| Budgets | 32k window; ≤8,192 output tokens per turn incl. thinking; commands 120 s default, 600 s max; background jobs die at episode end |
| History | Actions only (append-only prefix); reasoning retention is a Decoding knob (§7.4). Harness messages are tool results, since a user message moves the template's last-query index and strips earlier reasoning |

### 7.3 ACI components (each behind a flag; rungs 1–5 of §9.2)
1. **Execution guard** (with structured `run`): `stdin` from `/dev/null`; `PAGER=cat`, `GIT_PAGER=cat`, `DEBIAN_FRONTEND=noninteractive`, `TERM=dumb`. Timeouts kill the process group and return partial output (`timed_out: true`); `background=true` returns a job ID and log path.
2. **Observation shaper:** strip ANSI codes, collapse progress bars, fold repeats; keep head 40 + tail 80 lines within 6 KB, the rest in a spill file whose path is shown. Optional: Qwen3.5-0.8B summarizes logs over 20 KB (reported separately).
3. **View/edit/write tools:** numbered ranges and exact search-replace instead of `sed -i` and heredocs, whose quoting small models get wrong (SWE-agent's argument).
4. **Notes and compaction:** context = system + task + notes + the latest turns that fit. At 70% of the window a tool result requests a `notes` update, then the oldest turns drop. No hidden summarizer, so every context is reconstructable.
5. **Verify gate** (full ACI): `submit` passes only if `check_cmd` ran after the last mutation and `observed` (whitespace-normalized) is in its output. Any filesystem change (`docker diff` per action, or task-directory mtimes) counts as a mutation, `pip install` included. A failing `submit` is rejected once with the reason, then accepted, so the gate never traps an episode. The detector also feeds `unverified_submit` in every arm.

**Append-only prompts.** `llama-server` restores hybrid recurrent state only from context checkpoints (`--ctx-checkpoints`: default 32 per slot, ≥ `--checkpoint-min-step` apart, default 8,192 tokens). Editing earlier turns re-prefills from the previous checkpoint (issue #24714: full re-prefills on Qwen3.5), so compaction is batched, one rewrite per threshold crossing. S0a checks the template never rewrites earlier turns, sweeps `--checkpoint-min-step`, and measures cache hits and checkpoint memory, incl. host/device placement (DeltaNet state ~50 MB per sequence for the 4B and 9B, from config, f32 assumed; 4B ~48 MB; ~1.5 GB per slot at defaults).

### 7.4 Harness knob registry (Claim 1)

| Group | A (Tmax-like) | B (FACET-like) |
|---|---|---|
| Interface (format, parser, error recovery) | bash tool, submit marker, `max_format_errors` 64 | tmux keystrokes, JSON parser |
| Budgets (per command, per episode, turns) | `command_timeout` 120 s kill; task default × *m*; `max_steps` 64 | no per-command kill (model-chosen waits); 2 h per attempt (FACET); `max_turns` 1,000,000 † |
| Context (window, summarization) | none; `n_ctx` = Tmax's eval context, recorded in S0b (≥64k assumed from its 65,536-token response budget) | Base Qwen3.5 at its 262,144 maximum (9B KV alone: 8 GiB), FACET-tuned at 32k; the local cap (4B 64k on 6 GB, 9B 128k q8_0 KV on the Mac) is an unclosable deviation. Summarization at 8,000 free tokens (`enable_summarize`, `proactive_summarization_threshold`) † |
| Decoding (temperature, per-call cap, template) | `temperature` 0.7; `max_tokens` 16,384; Tmax's edited template (thinking kept on intermediate turns) | 1.0 (FACET); stock template; Terminus-2 default cap, recorded in S1 |

† Terminus-2 default, assumed (FACET does not state it). Column-A values (incl. the paper's 64-step and 120 s rollout limits) are confirmed against the released `Vanillux2Agent` in S0b, not assumed; its repo kwarg defaults already match. Tmax served with vLLM on an A100 (deviations: §9.1).

- **Design:** a 2^(4−1) fractional factorial (resolution IV; I = interface·budgets·context·decoding, − = A, + = B), run only if the 4B A−B paired 95% CI excludes zero. Its 8 configs on the 4B include A (all −) and B (all +). A main effect contrasts 4 configs with 4: per-task noise variance σ²/2 vs 2σ² for a one-at-a-time step: half the standard deviation, so about half the MDE, with no flip-order dependence. Two-factor interactions alias in pairs (e.g. interface×budgets with context×decoding), reported as pairs.
- **Build:** budgets are defined per interface, so groups flip independently. A `Vanillux2Agent` fork drops its per-command kill (budgets +) and adds Terminus-2-style summarization (context +); stock Terminus-2 is patched to cap each wait at 120 s, then send `C-c` (budgets −). Episode limits come from Harbor's job config.
- **Null-patch check:** at stock values, each patched class must send byte-identical requests to the stock agent on the 5 smoke tasks, fed its recorded observations, or nothing is attributed.
- Terminus-2 returns the screen as a user message, and the stock Qwen3.5 template keeps reasoning only after the last non-tool user message, so earlier reasoning is stripped every turn. The expected interface×decoding interaction is reported.
- B episodes log whether summarization fired or the local cap was hit; a non-negligible share is reported as an unreproduced part of the gap. Both 9B endpoints use q8_0 KV on the Mac, so KV precision is not a difference.

### 7.5 Traces and failure taxonomy
Parquet rows per turn (run hash, task, attempt, prompt/completion/cached tokens, latency, action, raw vs shown bytes, exit code, duration) and per episode (reward, label, wall-clock, peak container memory). Labels by rule, first match wins:

| Label | Rule |
|---|---|
| `resource_kill` | exit 137 / out-of-memory kill, or disk full |
| `hang` | a command used its full timeout, or output ends at an interactive prompt (`[Y/n]`, `password:`, pager) |
| `format_error` | ended on consecutive unparseable actions |
| `context_overflow` | request exceeded `n_ctx`, or the server truncated it |
| `timeout` / `turn_cap` | wall-clock limit / turn cap hit without submit |
| `unverified_submit` | no `run` after the last mutation (§7.3), in every arm |
| `truncation_loss` | a failing test quotes a string only in a dropped output region |
| `wrong_solution` | everything else |

The mentee hand-labels 50 failures, the mentor 20 of those; rule-versus-human agreement is reported.

### 7.6 Distillation pipeline (Claim 3)
The trainer is shared with 03 Spelunker (§2); new here are cross-harness trace conversion and the train/inference view-parity test.

1. **Collect** verified successes. Terminus-2 keystroke batches ending in Enter become `run(cmd)`; traces with interactive keystrokes (`C-c`, editors) are dropped, yield reported. Self-RFT yield = tasks attempted × the 9B's pool pass rate, measured the first night; below a projected 500 successes, arm (d) is dropped and public traces carry Claim 3.
2. **Segment** each assistant turn under the **`student` profile** (also used to evaluate base and students): §7.3 context manager, window = segment length (§8.1), notes ≤300 tokens, thinking off. If the fixed prompt (S0a) leaves under 1k tokens for turns at 2k, the 6 GB fleet trains at 4k on the 0.8B fallback.
3. **Mask** loss to the assistant action. **Decontaminate:** drop traces whose task ID or any instruction 13-gram matches TBLite, TB2.1, the dev slice or the ID set; MinHash near-duplicate filter.
4. **Train** Unsloth bf16 LoRA on Qwen3.5-2B with gradient checkpointing (`train` profile, §8.1); each arm fits one night (≤10 h), sized in S3's first hour from peak load/train/merge memory. **Ship:** merge, export GGUF Q4_K_M, evaluate in the same runner.

**Arms**, matched on supervised (action) tokens: **(a)** untrained 2B base; **(b)** public traces (OpenThoughts-Agent-SFT, TMax-SFT), actions converted to our tools, observations and history in **native rendering**, left-truncated; **(c)** the same examples **re-rendered through our shaper and context manager**; **(d)** (c) with part of the public examples replaced by self-RFT 9B traces. (b) vs (c) isolates view parity; (c) vs (d), trace source.

## 8. Hardware Profiles

### 8.1 The weakest machine sets the defaults

| | `portable` (default) | `accelerated` (8 GB VRAM) | `split` (6 GB laptop + Mac) |
|---|---|---|---|
| Agent model | Qwen3.5-4B Q4_K_M, 32k (64k for Claim-1 configs), 1 slot | 9B Q4_K_M 32k 1 slot; 4B 2 slots | 9B and the 4B Q8_0 control served by the Mac over LAN; containers on Windows |
| Harbor concurrency | 1 | 2 (4B), 1 (9B) | 1 |
| WSL2 cap | 6 GB | 8 GB | 8 GB |
| Student training | 2B LoRA, 2k segments (0.8B fallback) | 2B LoRA, 4k segments | as `portable` |

- **`train`** (any fleet): separate `.wslconfig`, 12 GB cap, sandboxes stopped; for the bf16 2B load, LoRA training, merge and GGUF export.
- The Mac also hosts 9B trace generation, the demo and an oracle portability check; amd64 images run emulated there, and tasks failing the oracle are listed, not dropped.
- **9B endpoints need `split`** (64k is unsafe on an 8 GB card, §8.2). Mac bandwidth (68–120 GB/s across base M1–M4; laptop RTX 3060: 336 GB/s) bounds 9B Q4_K_M decode at bandwidth ÷ 5.68 GB ≈ 12–21 tok/s before thinking tokens. `split` rows are priced separately (§9.4) and occupy both machines unless the gate shows a `split` and a `portable` job fit the WSL cap together.

### 8.2 Memory budget (§4.1 weights + f16 KV; compute buffers measured in S0a)

| Model | KV | Total | 6 GB card | 8 GB card | Mac 16 GB |
|---|---|---|---|---|---|
| Qwen3.5-9B Q4_K_M | 1.0 GiB @32k; 2.0 @64k; ~2.1 @128k q8_0 | ~6.7 GB @32k; ~8.0 @128k q8_0 | No (partial offload measured only) | 1 slot @32k; 64k unsafe | `split` only (no local Docker); 128k q8_0 tight vs macOS's GPU working-set limit, measured (fallback 64k q8_0) |
| Qwen3.5-4B Q4_K_M | 1.0 / 2.0 GiB per slot @32k / 64k | ~3.8 GB @32k; ~4.8 @64k or 2 slots @32k | 1 slot; 64k tight (~5.3 GB with buffers; fallback q8_0 KV) | 2 slots @32k; 1 @64k | Yes, beside a 6 GB Docker VM |
| Qwen3.5-4B Q8_0 (control) | as above | ~5.5 / ~6.5 GB + buffers | No | 1 slot @32k | Yes, via `split` |
| Qwen3.5-2B Q4_K_M | 0.375 GiB @32k | ~1.7 GB | Yes | Yes | Yes (demo) |
| Qwen3.5-0.8B Q8_0 | ~0.1 GiB @8k | ~0.9 GB | Beside the 4B: measured (else lower `-ub` or CPU) | Yes | Yes |
| Qwen3-4B-Instruct-2507 Q4_K_M | 4.5 GiB @32k | ~7 GB | 16k or q8_0 KV only | 16k | 16k or q8_0 KV, or no local Docker |
| 2B bf16 LoRA training | — | ~5 GB (Unsloth figure; sequence length unstated, 2k assumed, measured in S3) | Tight; 0.8B fallback | Yes | Not used |

Measured in S0a, outside the totals:
- *Compute buffers:* the 248,320-token vocabulary makes each `llama-server` output buffer ~0.5 GB at the default `-ub 512` (512 × 248,320 × 4 B); two instances plus the Windows display-driver (WDDM) reservation can exceed 6 GB.
- *Host RAM:* recent builds keep a host-RAM prompt cache (8,192 MiB default); we pin `--cache-ram` to 1–2 GiB. WSL cap + Windows host + server resident memory must stay under 16 GB (logged).
- *Recurrent state/checkpoints* (§7.3): ~50 MB per sequence; checkpoints up to ~1.5 GB per slot at defaults.

### 8.3 Go/no-go gate (decided at session 3; the constraint is found in week 2)
A timing run precedes the reproduction (TBLite, 1 attempt, stock frozen configs, nothing tuned, so nothing leaks into claims). Per rate class it logs prefill/decode tok/s at 8k/16k/32k/64k, output tokens per turn (thinking on), prefix-cache hits, latency, **effective wall-clock per episode** (wall-clock ÷ concurrency), and `timeout`/`turn_cap` shares.

| Class | Config | Tasks |
|---|---|---|
| **R4-A** | 4B, `portable`, endpoint A (thinking on, 64k, Tmax template) | 20 |
| **R4-B** | 4B under endpoint B's budgets (2 h, uncapped turns) | 10 |
| **R9** | 9B, `split`, endpoints A and B; TB2.1 under A to scale for its longer runtimes | 10 each; 5 TB2.1 |
| **R2** | 2B, `student` profile; priced in S2 once the ACI exists | — |

| Rule | Trigger | Action |
|---|---|---|
| **Go** | Every committed §9.4 row, priced at its class rate, fits ~35 laptop-hours and ~25 Mac-hours a week for nine weeks (assumed; confirmed with the mentee in S0a) | Proceed |
| **9B rule** | Priced 9B rows exceed the Mac budget | P1b → P4; Claim 1 narrows to 4B/TBLite (fallback wording in §1, §3, §12); P2's 9B row dropped; teacher = 4B self-traces; "gap closed" measured against the 4B at rung 5 |
| **B-budget rule** | R4-B rows do not fit | 4B B-budget configs cap episodes at task timeout × *m*, a deviation shown in every Claim-1 table; the 9B TB2.1 endpoint B keeps 2 h |
| **Cut list** | Otherwise | Apply in order, below |

1. Append-only prompts, `--cache-ram`, checkpoint tuning.
2. Claims 2–3 only: thinking off, then 16k window, then turn cap 40. This changes Decoding and Budgets, so never Claim 1's endpoints; if forced, they are relabelled approximations of A and B.
3. Qwen3-4B-Instruct-2507 if cache misses persist.
4. Ladder rungs 0, 2, 5 and 2 attempts for ablations (MDE recomputed).
5. Factorial at 2 attempts, then endpoints only.

## 9. Evaluation Plan

### 9.1 What is measured

| Area | Metrics | Compared against |
|---|---|---|
| Reproduction (S0b) | TBLite pass@1: 4B, `Vanillux2Agent`, Tmax's edited template, `n_ctx` ≥ 64k (Tmax's eval context, from the repo or authors), *m* = 1 of record | 31.8 ± 3.8. **Pass** if \|local − 31.8\| ≤ 1.96 × √(SE_local² + 3.8²) on all 100 tasks (SE_local by per-task bootstrap). The band is wide (~±10.5 at SE_local ≈ 3.8), so the point estimate and both diagnostics are always reported |
| Diagnostics | Fixed 30-task subset, run regardless: Q8_0 control on `split` (quantization), then *m* = 3 (speed) | Record run on those 30 tasks; a miss that shrinks at *m* = 3 is laptop speed, not quantization |
| Harness attribution | Pass@1 per config; endpoint gaps, factorial main effects; paired 95% CI | §5.2 TB2.1 values; 9B TBLite 41.9 / 36.4 within Tmax (P4) |
| ACI | Pass@1, tokens and wall-clock per solved task, failure-label shares | Rung 0 (§9.2) |
| Distillation | Primary: each trained arm vs arm (a), ID set. Also, on TBLite: gap closed = (student − 2B base) ÷ (9B − 2B base); students and base under `student`, the 9B in the full ACI as labelled upper reference | 2B base; 4B at rung 5; Tmax-2B 11.8 (reference only) |
| Cost frontier | pass@1, pass@3, pass^3 vs tokens and wall-clock per solved task | Per machine profile |

pass@k uses the unbiased estimator over n = 3 attempts; pass^3 is the share of tasks solved in all 3 (τ-bench's consistency metric). Endpoint tables list deviations: Q4_K_M vs bf16, llama.cpp vs vLLM, laptop vs A100 wall-clock (*m*, `timeout` share), local context cap, reconstructed TBLite config.

### 9.2 Ablation ladder (Qwen3.5-4B, TBLite-laptop, 3 attempts)
Rung 0 is the Terminus-2 keystroke interface with §7.2 budgets, a 32k window and identical decoding. It is not endpoint B; only interface components change up the ladder. Rungs 1–5 add the §7.3 components in order (1 guard, 2 shaper, 3 view/edit, 4 notes, 5 gate), so rung 5 is the full ACI. The 9B in the full ACI tops the ladder (P2); leave-one-out from rung 5 for up to three components is P4. Components below the MDE are reported as a group, never as individual wins.

### 9.3 Pre-registered MDEs and how n was chosen
n is fixed by the benchmark (~100 TBLite tasks, fewer after exclusions); attempts per task is the free parameter. Per task, d_i = mean pass under X − under Y; paired bootstrap over tasks. **MDE ≈ (z(1 − α′/2) + z(0.8)) × sd(d) / √n** (power 0.8; α′ = Holm level of the family's first test). The last column is illustrative, not measured.

| Comparison | α′ | Factor | MDE at sd(d) = 0.4 |
|---|---|---|---|
| Single endpoint gap, 100 tasks | 0.05 | 2.8 | ≈11 points |
| 9B endpoint gap, 89 TB2.1 tasks | 0.05 | 2.8 | ≈12, about the published gap: a null means "not distinguishable at laptop scale", not "no gap" |
| Single gap, 30 tasks | 0.05 | 2.8 | ≈20, so the dev slice carries no claim |
| Four factorial main effects | 0.05/4 | ≈3.34 | ≈6.7 (half the sd of a two-config difference) |
| Five ladder contrasts | 0.05/5 | ≈3.42 | ≈13.7 (first contrast) |
| Shrunk ladder (rungs 0, 2, 5), two contrasts | Holm over 2 | — | ≈12.3 |

- **Floor:** the S0b A/A run (reproduction config twice, ~100 TBLite-laptop tasks) sees only sampling noise, while real comparisons add per-task effect heterogeneity. Its MDE is a floor; each comparison also reports its observed sd(d) and implied MDE.
- **Claim 3:** TBLite floor from the 2B base's per-task variance over 3 attempts (Var(d_i) ≥ 2 p̂_i(1 − p̂_i)/k); ID-set floor from sd(d) ≤ √(2 p̄(1 − p̄)), p̄ = base ID pass rate (1 attempt). Holm across the three trained arms on the ID set; the one TBLite comparison (best arm vs base) is unadjusted.
- **Shrink rule:** a floor-based ladder MDE above 12 points (true at sd(d) = 0.4) shrinks the ladder to rungs 0, 2, 5.

### 9.4 Compute budget (hours = episodes × the class's effective minutes ÷ 60)

| Priority | Study | Configs × tasks × attempts = episodes | Rate class |
|---|---|---|---|
| P0 | Reproduction (= endpoint A) + A/A arm | 2×100×3 = 600 | R4-A |
| P0 | Diagnostics: Q8_0 control; *m* = 3 | 2×30×3 = 180 | Q8_0 on `split`; R4-A × up to 3 |
| P1 | 4B endpoint B | 1×100×3 = 300 | R4-B |
| P1 | 4B factorial, 6 more configs (if triggered, §7.4) | 6×100×3 = 1,800 | 3 at R4-A, 3 at R4-B |
| P1b | 9B TB2.1 endpoints A and B (9B rule, §8.3) | 2×89×3 = 534 | R9 × TB2.1 factor |
| P2 | Ladder rungs 0–5, 4B | 6×100×3 = 1,800 | R4-A |
| P2 | 9B in the full ACI (ladder top; Claim 3 denominator) | 1×100×3 = 300 | R9 |
| P3 | 9B trace generation, converted TMax-15K | N×1 (N from first-night yield) | R9 |
| P3 | Student arms (a)–(d), held-out ID set | 4×300×1 = 1,200 | R2 |
| P3 | 2B base and best ID arm on TBLite | 2×100×3 = 600 | R2 |
| P4 | 9B TBLite endpoints; leave-one-out; 4B under `student`; TB2.1 finals for ACI and student | 2×100×3 + 3×100×3 + 1×100×3 + 3×89×3 ≈ 2,600 | mixed |

P0–P3 are committed (~7,300 episodes + N), P1b subject to the 9B rule; P4 runs only if measured rates allow. At a reference 3.5 effective minutes, the laptop-GPU rows (R4 and R2, ~6,400 episodes) need ~370 h against a ~315 h laptop budget. Pre-registered resolution: the ladder shrinks to rungs 0, 2, 5 if the factorial runs or the §9.3 shrink rule fires (−900 episodes; ~320 h with the factorial); otherwise the full ladder runs (~270 h without it). The gate prices R4-B, R9 and TB2.1 rows (no reference rate). The optional API arm runs the full ACI on the dev slice under the $20 cap, labelled exploratory.

### 9.5 Honest-reporting rules
- No gain claimed inside the MDE; every table has n, attempts, CI and a cost column.
- Raw and adjusted scores, with the exclusion list; local Q4 numbers labelled, never equated with published bf16 cloud runs.
- A distillation null is reported; the harness study carries the project either way. Tmax's H100 RL moved the 2B from 5.7 to 11.8 on TBLite, less than the TBLite MDE, so a TBLite null is expected and the ID set is primary.

## 10. Milestones (19 hours / 15 sessions)

| Sprint | Sessions | Deliverable |
|---|---|---|
| **S0a** | 1–2 | Repo, WSL2/Docker caps, `llama-server` on both machines; smoke test; **TBLite-laptop published**; TB2.1 timeouts parsed; pinned llama.cpp build measured (tool-call parse rate, cache hits, checkpoint memory, memory table; `--cache-ram` pinned); dev slice converted |
| **Gate** | 3 | Timing run (§8.3); every §9.4 row priced; go/no-go and cuts at the end of week 2 |
| **S0b** | 3–4 | **Reproduce Qwen3.5-4B + `Vanillux2Agent` on TBLite against 31.8 ± 3.8** (§9.1); A/A arm → MDE floor table; Q8_0 and *m* = 3 diagnostics |
| **S1** | 5–8 | Knob registry; null-patch checks; 4B endpoint B; 9B TB2.1 endpoints overnight on `split`; factorial if triggered; **week-6 demo: one model, three scores, how much of the gap survives locally** |
| **S2** | 9–10 | ACI behind flags, thresholds frozen in a tagged config; ladder; 9B in the full ACI; failure dashboards; cost table. Overnight: TMax-15K → Harbor conversion (self-RFT pool, ID set) and 9B trace generation, interleaved with the ladder (parallel if both fit the WSL cap) |
| **S3** | 11–12 | Segmentation, decontamination; `train` memory check; LoRA arms (b)–(d); student eval on ID set and TBLite |
| **S4** | 13–14 | P4 as budget allows; TB2.1 final; **"terminal specialist in 1.3 GB" demo on the Mac**; write-up |
| — | 15 | Interview sprint: the 11-point story, the ladder, the gap-closed number |

## 11. Risks & Mitigations

| Risk | Mitigation |
|---|---|
| Hybrid prefix-cache misses force a full re-prefill every turn | Measured in S0a; append-only prompts, tool-result harness messages, batched compaction, checkpoint sweep (§7.3); Qwen3-4B-Instruct-2507 / Qwen3-8B fallback, KV cost stated |
| `llama-server --jinja` mis-parses Qwen3.5 tool calls on the pinned build | S0a parse-rate test on 200 calls; fallback: JSON-in-text with a strict parser, as a knob |
| Laptop, B-budget (hours each) or Mac 9B episodes too slow (P1b, P2's 9B row, trace generation) | Week-2 gate prices every row by rate class (B budgets as R4-B); 9B rule, B-budget rule, ordered cut list (§8.3); P4 optional |
| Laptop speed hits wall-clock timeouts an A100 would not | *m* = 3 diagnostic; *m* fixed per profile, in the run hash; `timeout` share in every table; Windows and Mac numbers never pooled |
| Reproduction misses the 31.8 band | Diagnostics run regardless, then a config diff against `Vanillux2Agent`; miss reported. Above 15 exclusions: "inconclusive", not "passed" |
| 2B LoRA does not fit 6 GB at 2k | 0.8B student; Qwen3-1.7B QLoRA if the Qwen3.5 stack breaks; optional free Colab/Kaggle GPU for a 4B arm |
| Public traces leak eval tasks or map poorly to our tools; too few self-RFT traces | Decontamination with published overlap counts; conversion yield reported; arm (d) adds self-RFT traces, dropped below a projected 500 (§7.6) |
| Prior-art overlap | Our contribution is knob-level attribution and the view-parity test, not the ACI idea (SWE-agent) or the generic per-step recipe (shared with 03 Spelunker) |

## 12. Résumé Bullets (experience voice; numbers filled in as sprints land)

- Built a **fully local terminal-agent evaluation and training stack** (Harbor, Docker in WSL2, llama.cpp) running OpenThoughts-TBLite and Terminal-Bench 2.1 on a 6 GB laptop GPU; reproduced a published Qwen3.5-4B score within **X points** (pre-registered band ±**X**).
- Tested an **11-point published gap** in Qwen3.5-9B's Terminal-Bench 2.1 score on a laptop (**X** points survived local serving); a fractional factorial over **X episodes** attributed **X** points of the Qwen3.5-4B harness difference to **[group]** (paired bootstrap CIs, pre-registered MDE). *Null wording:* no single knob group cleared the **X**-point MDE; the difference is an interaction of **[groups]** / is within noise at laptop scale.
- Designed an **agent-computer interface** (output shaping with spill files, execution guard, view/edit tools, notes compaction, verify-before-submit gate) that raised Qwen3.5-4B's TBLite pass rate from **X% to X%** with **X%** fewer tokens per solved task.
- Distilled verified trajectories into a **Qwen3.5-2B student** (per-turn segments, observation-masked bf16 LoRA, one consumer GPU): **+X** points on held-out tasks, **X** from re-rendering public traces through the agent's own interface, closing **X%** of the 2B-to-9B gap; shipped as a **1.3 GB GGUF** (~1.7 GB+ resident at 32k).
- Built a **rule-based failure taxonomy** over **X** traces (**X%** agreement with hand labels), cutting unverified submissions from **X% to X%**.

### Placement Note

List these under experience **only with a real organization and engagement**: (1) **the mentoring organization as employer**, truthful title (`ML Engineering Practicum` / `Machine Learning Intern`), engagement record; (2) **a real collaboration** with a startup, lab or open-source org hosting the work; or (3) **an open-source project entry**: a public repo with real commit history, the TBLite-laptop subset and the reproduction table. Never invent an employer or dates: North American employers verify employment, and a fabricated entry is grounds for a rescinded offer. A public repo reproducing a published number is easier to verify than an internship line.

## 13. Why This Reads as Real Company Work

- **It starts from a measurement dispute** ("two papers disagree by 11 points; which config do we ship?") and reproduces the baseline first.
- **Noise is budgeted:** A/A floors, Holm-adjusted MDEs, a factorial design, compute priced per rate class at a week-2 gate.
- **Cost is in every table**, as for an agent embedded in continuous integration with a latency budget.
- **Hardware shapes the design:** KV bytes from the config, append-only prompts for a hybrid model, a two-machine `split` profile.
- **Failures are labelled by rule and audited by hand**, so each change is explained by what it fixed.

## 14. Interview Leverage

- **Agent system design:** loop, sandboxing, budgets, context management, serving on constrained hardware.
- **LLM engineering:** why keystroke interfaces hurt small models; output shaping; verify gates; prefix caching on hybrid attention.
- **Training:** cross-harness trace conversion; view parity (arms b vs c); decontamination; LoRA memory arithmetic.
- **Evaluation:** pass@k vs pass^k, paired bootstrap, MDE and power, Holm, A/A floors, fractional factorials and aliasing, reproducing single-lab numbers.
- **Behavioral:** "a time results didn't reproduce" (S0b and the 11-point investigation).

## 15. References

- OpenThoughts-TBLite — https://huggingface.co/datasets/open-thoughts/OpenThoughts-TBLite
- Terminal-Bench 2.1 — https://tbench.ai/news/terminal-bench-2-1 ; Merrill et al., *Terminal-Bench* (ICLR 2026) — https://arxiv.org/abs/2601.11868
- Harbor — https://github.com/harbor-framework/harbor ; Terminus-2 — https://harborframework.com/docs/terminus-2 ; job config (`agent_timeout_multiplier`) — https://docs.harborframework.com/core-concepts/jobs/configs
- *FACET* (Aug 2026) — https://arxiv.org/abs/2608.18580
- *Tmax: A simple recipe for terminal agents* (Jun 2026) — https://arxiv.org/abs/2606.23321 ; code — https://github.com/hamishivi/tmax
- TMax data — https://huggingface.co/datasets/allenai/TMax-15K , https://huggingface.co/datasets/allenai/TMax-SFT-16.5K
- *OpenThoughts-Agent* (Jun 2026) — https://arxiv.org/abs/2606.24855 ; https://huggingface.co/datasets/open-thoughts/OpenThoughts-Agent-SFT-10K
- Qwen3.5 cards and configs — https://huggingface.co/Qwen/Qwen3.5-9B (and `-4B`, `-2B`, `-0.8B`); https://huggingface.co/Qwen/Qwen3-4B-Instruct-2507
- GGUF files — https://huggingface.co/unsloth/Qwen3.5-9B-GGUF (and sibling `unsloth` repos)
- Unsloth Qwen3.5 fine-tuning — https://unsloth.ai/docs/models/qwen3.5/fine-tune
- llama.cpp issue #24714 — https://github.com/ggml-org/llama.cpp/issues/24714 ; `llama-server` flags (`--ctx-checkpoints`, `--checkpoint-min-step`, `--cache-ram`) — https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md
- *SWE-agent* — https://arxiv.org/abs/2405.15793 ; mini-SWE-agent — https://github.com/SWE-agent/mini-swe-agent
- *τ-bench* (pass^k) — https://arxiv.org/abs/2406.12045 ; Chen et al. (unbiased pass@k) — https://arxiv.org/abs/2107.03374
