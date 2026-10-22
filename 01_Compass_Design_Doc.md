# Compass — Design Document

**Agent Decisions as a Ranking Problem: a 1.7B Policy Proposes, a 0.6B Process Reward Model Ranks, on One Laptop**

| | |
|---|---|
| Status | Proposal |
| Target duration | ~10–12 weeks (19 mentoring hours / 15 sessions + independent work) |
| Platforms | **Windows 11 laptop + WSL2** (16 GB RAM, RTX 3060-class, **6 GB VRAM assumed**, 8 GB if confirmed) and **macOS on Apple Silicon** (16 GB, no CUDA; chip recorded in S0), one repository. The weaker machine sets every default. |
| Deployment | 100% local. No rented servers or paid GPU cloud. Commercial APIs only as an optional reference row (< $20 total); free Colab/Kaggle only as an optional accelerator. |
| Core stack | Qwen3-1.7B · Qwen3-0.6B / Qwen3-Reranker-0.6B · Qwen3-4B / Qwen3-8B baselines · llama.cpp · vLLM (WSL2, labelling) · PyTorch + PEFT / Unsloth · ALFWorld · TextCraft · Optuna · MLflow · FastAPI |
| Sprints | S0–S4 plus an interview session, defined in §10 |

---

## 1. The Product Thesis

An on-device assistant, a game-NPC planner or an edge robot's task planner gets one chip and one per-step budget. The model that fits makes too many wrong moves; the one that makes fewer does not fit. On a 6 GB laptop GPU, "use the bigger model" means CPU offload and multi-second steps.

**Compass spends the same milliseconds on search instead of scale.** Each step is a two-stage recommender: Qwen3-1.7B **generates** N candidate actions; a 0.6B process reward model (PRM) **ranks** them by estimated probability of eventual success. Recsys baselines carry over: majority vote is popularity, policy log-probability the generator score, scoring every admissible action full-corpus ranking.

> **Thesis: on fixed hardware, a small policy plus a small trained critic beats a larger model at equal per-step latency, and the critic is trained from the environment with Monte-Carlo (MC) labels and no human annotation.**

Claims 1, 2 and 4 carry a pre-registered minimum detectable effect (MDE, §9); claim 3 is descriptive, with confidence intervals (CIs) and a pre-registered falsification line.

1. **Search beats scale at equal latency.** Qwen3-1.7B + PRM best-of-N (BoN) beats Qwen3-8B greedy at equal or lower 95th-percentile (P95) per-step latency on **both** ALFWorld test (274 games, pooled) and TextCraft test (200 tasks). Headline machine: the **Mac**, where both fit without offload. The 6 GB result is labelled memory-bound when the 8B offloads. Qwen3-4B greedy, the largest model that fits 6 GB unoffloaded, is a pre-registered secondary comparator; the claim names the baselines beaten.
2. **Trained beats prompted.** On a frozen held-out set of MC-labelled states, the PRM ranks candidates with higher pairwise accuracy than a prompted Qwen3.5-9B judge. An **oracle** (true MC values) sets the ceiling on a fixed subset. Scoring latency is reported per runtime (PyTorch vs MLX), never as a ratio.
3. **The compute frontier has an edge.** Success vs tokens vs P95 for BoN (N = 1/2/4/8/16, one prompt window for all N), step beam and 1-step lookahead. It locates the **PRM over-optimization point**, where the chosen action's PRM score keeps rising while its oracle MC value falls, and reports Expected Calibration Error (ECE) and an abstain-and-ask threshold. If success is non-decreasing through N=16, the report says: no over-optimization within N ≤ 16.
4. **Post-training closes the loop.** One round of PRM-filtered rejection-sampling fine-tuning (RFT), trained with QLoRA, i.e. low-rank adaptation (LoRA) on a 4-bit base, lifts Qwen3-1.7B's no-search ALFWorld greedy success over base ReAct and over a success-only filter. No fixed target.

**Demo:** a Mac replay of each step: the 1.7B's candidates, the critic's scores, the 8B's greedy move, both clocks running. **Hook:** the critic is smaller than the policy it judges; the headline plot (1.7B + 0.6B critic above an 8B, same laptop, same latency) reads in five seconds. The second shows where search starts fooling its own critic, the Goodhart effect ranking teams know from click models.

## 2. Goals & Non-Goals

### Goals
- One environment interface (ALFWorld text, TextCraft) with snapshot/replay; one search interface with explicit budgets.
- MC labels on branching states only, reporting label variance and informative-state share.
- PRM grid: init (language model (LM) vs reranker) × loss (pointwise vs pairwise) × label budget (K=2 vs K=4 subset).
- A published equal-latency benchmark: one pinned serving stack and quantization pipeline per platform.
- Calibrated abstain-and-ask; one PRM-filtered RFT round; GRPO-lite (small-scale Group Relative Policy Optimization) on TextCraft as a gated stretch.
- S0 reproduces a published number before any new claim.

### Non-Goals
- State of the art: GiGPO (Group-in-Group Policy Optimization), 86.7% at 1.5B on 2×H100 for 150 iterations, is a ceiling reference.
- Reinforcement learning (RL) on ALFWorld (§7.6), the visual `AlfredThorEnv`, long thinking chains (Qwen3 runs with `enable_thinking=False`).
- Competing with decomposition prompting (ADaPT, THREAD): Compass keeps a flat ReAct loop and changes only how compute is spent.
- Temporal-difference labels with generalized advantage estimation (TD+GAE, as in AgentPRM-X): MC is the only labeller; its relative token cost is cited, not re-measured.
- Any runtime network dependency; services use localhost only.

## 3. Alignment With Prior Experience

| Prior experience | Skill | Compass module |
|---|---|---|
| **Changsheng**: two-stage sector → stock strategy, backtested out of sample | Generate-then-rank; held-out gates | Policy proposes, PRM ranks; test splits frozen like an out-of-sample period |
| **Changsheng**: AutoML hyper-parameter agent | Constrained search | **Optuna over test-time compute**, P95 as hard constraint |
| **College Application Agent**: LangGraph Writer–Critic, FastAPI | Generator + critic | Prompted critic → **trained critic** (claim 2); FastAPI |
| **CAS**: Precision@K vs latency vs storage, 20+ configs | Frontiers | Success vs tokens vs P95 |
| **DiDi**: A/B testing | Experimentation | Paired seeds, cluster bootstrap, MDEs |
| **Sina Weibo**: monitoring with alerting | Metrics | Latency, abstain-rate and ECE dashboards |

**Not prior experience:** Enclave, an unexecuted earlier mentor proposal, fine-tunes Qwen3-Reranker-0.6B. If it finishes first, its training code is reused for PRM B; until then PRM B is new work.

**Interview one-liner** (last sentence from the Mac result): *"At Changsheng I built a screen-then-rank funnel for stocks. Compass is the same funnel for an agent's next action: a 1.7B model generates candidates and a 0.6B critic trained on Monte-Carlo labels ranks them. On the same Mac at the same per-step latency, it beat an 8B by X points."* If claim 1 fails: *"…it did not beat the 8B, and I traced the gap to X."*

## 4. Stack Choices — and Why

### 4.1 Models

| Role | Model (Apache-2.0) | Architecture | Format by use | Weights |
|---|---|---|---|---|
| Policy | **Qwen3-1.7B** | 1.7B (1.4B non-embedding), 28 layers, GQA 16Q/8KV, 32K ctx | Q4_K_M GGUF (search, benchmark); BF16 (opt-in vLLM labelling); NF4 (QLoRA) | ~1.11 GB Q4_K_M (published Unsloth file); 3.45 GB BF16; QLoRA ~4–5 GB (est.) |
| PRM A | **Qwen3-0.6B** + scalar head | 0.6B (0.44B non-embedding), 28 layers; linear head on the last hidden state, no LM head | BF16 + LoRA | ~1.2 GB; training ~3–4 GB (est.) |
| PRM B | **Qwen3-Reranker-0.6B** | From Qwen3-0.6B-Base; P("yes") from the yes/no logits at the last position | BF16 + LoRA | ~1.2 GB |
| Baseline | **Qwen3-4B** | 4.0B, 36 layers, GQA 32Q/8KV | Q4_K_M | ~2.5 GB + KV; no offload on 6 GB |
| Baseline | **Qwen3-8B** | 8.2B, 36 layers, GQA 32Q/8KV | Q4_K_M | ~5.03 GB + KV; offload expected on 6 GB |
| Prompted judge | **Qwen3.5-9B** | Hybrid Gated DeltaNet + gated attention, multimodal | MLX 4-bit, Mac | ~5–6.6 GB by build (Ollama: 6.6 GB) |
| S0 reproduction | **Qwen2.5-1.5B-Instruct**; **Qwen2.5-7B-Instruct** | 1.5B; 7B | BF16 or Q8_0; Q4_K_M on the Mac | measured in S0 |

**Why Qwen3, not Qwen3.5.** Anything that trains or needs prefix caching uses standard-attention Qwen3; Qwen3.5 is only a frozen judge. Unsloth's Qwen3.5 guide says "It is not recommended to do QLoRA (4-bit) training on the Qwen3.5 models", and 6 GB needs QLoRA. Compass's step cost depends on cache reuse, and llama.cpp #22384 reported full prompt re-processing every turn on hybrid-DeltaNet Qwen3.5/3.6-27B. It closed in April 2026: the maintainer recommended `preserve_thinking`, a Qwen3.6-template flag, and the reporter still saw re-processing with it set. Qwen3's key-value (KV) cache, with grouped-query attention (GQA), is plannable: 2 × layers × 8 KV heads × 128 × 2 bytes = **112 KiB/token** (0.6B/1.7B), **144 KiB/token** (4B/8B).

**One quantization pipeline.** Qwen ships no official Q4_K_M for Qwen3-1.7B (its GGUF repository has only Q8_0, 1.83 GB), and community files differ (Unsloth's are Dynamic 2.0 quants). S0 re-quantizes 1.7B, 4B and 8B from the official BF16 safetensors with the pinned tag's `convert_hf_to_gguf.py` + `llama-quantize Q4_K_M`, no importance matrix, and fingerprints each GGUF's sha256. Table sizes are published files', for planning.

**Judges.** Qwen3.5 thinks by default (model card), which breaks one-pass scoring, so the judge runs with `enable_thinking=False`, reading P("yes") at the first assistant token after the empty think block; S0 checks the rendered template. A Qwen3-8B judge on `llama-server` removes dependence on the Qwen3.5 runtime. *Confirm model cards before S0.*

### 4.2 Runtimes — One Pinned Stack per Platform

| Use | Windows (WSL2 Ubuntu, CUDA) | macOS (Metal) |
|---|---|---|
| Policy, 4B/8B baselines | `llama-server`, pinned build tag | `llama-server`, same tag |
| PRM scoring | PyTorch CUDA | PyTorch MPS |
| MC labelling | `llama-server`; vLLM BF16 opt-in | `llama-server` |
| Prompted judges | not run (would offload) | mlx-lm (Qwen3.5-9B), `llama-server` (Qwen3-8B) |
| Environments | ALFWorld + TextCraft | TextCraft arm64; ALFWorld in a separate osx-64 (Rosetta) env-pool process |
| Training | Unsloth (QLoRA 1.7B), PEFT (LoRA 0.6B) | PEFT on MPS, PRM only (fallback) |

Claim-1 arms share runtime, build tag and quantization, or the claim is about runtimes. Windows work runs in WSL2 because TextWorld, ALFWorld's text engine, supports only Linux and macOS (TextWorld 1.7.0, PyPI); one userspace keeps cross-OS hops out of latency.

**Mac split, from day 1.** TextWorld 1.7.0 has no macOS arm64 wheel (only manylinux x86_64, macOS x86_64 and an sdist). ALFWorld's README recommends an x86 conda env (`CONDA_SUBDIR=osx-64`) on Apple Silicon, and current PyTorch has no macOS x86_64 wheels. So ALFWorld runs in its own Rosetta process, calling the arm64 `llama-server` and PRM over localhost HTTP. Policy+PRM latency excludes environment time, so the Mac comparison stays fair; total latency, which includes it, is reported beside it (§7.7).

### 4.3 Libraries
`alfworld` (MIT, text-only `AlfredTWEnv`); TextCraft from the ADaPT repo (MIT; recipes downloaded separately); `uv` with one lockfile for the services, plus a separately pinned osx-64 conda environment for the Mac's ALFWorld env-pool; Unsloth/PEFT, with TRL and OpenPipe ART (Apache-2.0) as GRPO references, not dependencies; Optuna with a constraint function; MLflow; Parquet; FastAPI; own cluster bootstrap (resampling unit = game).

## 5. Benchmarks & Datasets

### 5.1 ALFWorld (headline 1)
CPU-only text environment. **Test = valid_unseen (134 games) + valid_seen (140), pooled to 274** for claims 1 and 4; valid_unseen alone is the out-of-distribution (OOD) breakdown, not separately powered. Training games: ~3,553 in common reports, varying by data version (S0 records count and list hash). **Protocol A** (primary): admissible commands (`info['admissible_commands']`) in the prompt, 2-turn history, as in GiGPO's template. **Protocol B** (secondary): list hidden, free-form commands. Step cap 30, as in AgentPRM-C; the S0 reproduction uses GiGPO's 50.

| System | Setting | Success | Source |
|---|---|---|---|
| AgentPRM-C π0, Llama-3.2-3B, supervised fine-tuning (SFT) init | 30-action cap; OOD (paper states 134 and 136 games) | 64.9% | AgentPRM-C T1 |
| AgentPRM-C π3 / BoN(π3, Q2) N=16 | same | 88.1% / 91.0% | AgentPRM-C T1 |
| ReAct, GPT-4o | as reported there | 65.7% | AgentPRM-C T1 |
| Qwen2.5-1.5B-Instruct prompting / ReAct | 50 steps, admissible list, verl-agent defaults† | 4.1% / 12.8% | GiGPO T1 |
| Qwen2.5-1.5B GRPO / GiGPO | same; 2×H100, 150 iterations | 72.8 ± 3.6% / 86.7 ± 1.7% | GiGPO T1 |
| Qwen2.5-7B-Instruct ReAct | same | 31.2% | GiGPO T1 |
| **Qwen3-1.7B / 4B / 8B ReAct** | Compass A and B, cap 30 | **S0** | — |

†GiGPO names no split. Its rows assume verl-agent defaults (`eval_dataset: eval_in_distribution`, i.e. valid_seen; `val_data_size=128`; validation temperature 0.4), so they are not directly comparable with AgentPRM-C's OOD rows.

### 5.2 TextCraft (headline 2)
From ADaPT: Minecraft 1.16.5 recipes as text tasks. Actions are `get`, `craft <item> using <ingredients>` and `inventory`; reward is 1 when the target enters the inventory. Each task lists the gold craft commands plus up to 10 distractors. Pure Python, deep-copyable.

**Splits come from code.** No split lists ship. `TextCraft.reset(seed)` sorts `item_recipes_min_depth(2)` deepest first and picks index `seed % len`; `run_textcraft.py` runs seeds 0–199. So **test = seeds 0–199 (the 200 deepest items)** and **dev = seeds 200…len−1** (seed = task index). The paper's counts may not match (test 123 depth-3 + 11 depth-4 + 77 depth-2 = 211; 220 depth-2 dev tasks), so S0 commits per-depth counts, hashes and the measured dev count.

**Depth shift, declared.** By the paper's counts all depth-3/4 items are in test, so labels, tuning and GRPO-lite data are depth-2 while about two thirds of test is depth 3–4. Pre-registered: **a depth-2-trained critic, evaluated per depth** (2 vs 3–4). Deeper goals are not synthesised (they would be test items); depth-3/4 items S0 finds beyond seed 199 join dev.

**No admissible list.** Crafting commands are listed, but `get` quantities and argument binding are free-form. A **full-ranking** arm scores the listed craft commands plus `get` of each listed ingredient at its recipe quantity, so ranking vs generation is measured on both benchmarks; TextCraft is therefore a required co-headline.

| System | Success | Source |
|---|---|---|
| ReAct / ADaPT, GPT-3.5 | 19.0% / 52.0% | ADaPT |
| ReAct / ADaPT, GPT-3.5 (re-run) | 20.5% / 52.5% | THREAD T3 |
| ReAct / ADaPT / THREAD, Llama-3-8B | 12.5% / 23.5% / 92.0% | THREAD T3 |
| Qwen2.5-3B-Instruct + AgentPRM-X, beam @8×8 (ORM 43.3%, PVM 44.3%) | 56.7% | AgentPRM-X T1 (setting differs) |
| **Qwen3-1.7B / 4B / 8B ReAct** | **S0** | — |

THREAD's 92.0% shows decomposition is strong here; Compass does not compete with it (§2).

### 5.3 WebShop (optional, S4)
GiGPO reports Qwen2.5-1.5B-Instruct ReAct at score 40.1, success 11.3%. Needs Java and a pyserini index; runs only if S4 has slack.

### 5.4 Training and Demo Data
Test items never feed labels, tuning, RFT or the demo. ALFWorld training games are split by game-id hash before any rollout; the partition is committed to `prereg/`.

| Partition | ALFWorld | TextCraft dev | Used for |
|---|---|---|---|
| **Label** | rest of training | rest (~70) | Set **(a)**: PRM-train labels, K=2; K=4 on a 10% subset (label-noise check, label-budget arm) |
| **Tuning** | 100 games | 60 tasks | Set **(b)**, K=4: early stopping, label-noise check; episodes for Optuna, τ, N*, gates, demo, S0 pilot |
| **Claim-2 eval** | 100 games | 40 tasks | Set **(c)**: ~200 states, K=4 (K=8 if throughput allows), hashed into `prereg/` before PRM training; only pairs whose q̂ differs count |
| **Policy training** | 500 games | 50 tasks | RFT (ALFWorld), GRPO-lite (TextCraft); never labelled, so PRM filter and reward score out of sample |

TextCraft counts assume ~220 dev tasks and scale with the measured count. The S0 1.5B/7B reproduction runs on valid_seen, as a baseline only.

## 6. Architecture

```
 ONE MACHINE, NO NETWORK AT RUN TIME: 6 GB card (WSL2) or 16 GB Mac (Metal)

 bench CLI, demo UI --> search-controller (FastAPI)  /v1/episode, /v1/step
                        budgets: N, beam, lookahead, step cap, P95, tau
       +-------------------+--------------------------+
   env-pool           policy-svc                   prm-svc
   ALFWorld/TextWorld llama-server                 PyTorch CUDA | MPS
   TextCraft          Qwen3-1.7B Q4_K_M            A: Qwen3-0.6B + head
   snapshot/replay    N samples, T=0.8             B: Qwen3-Reranker-0.6B
   determinism test   (4B, 8B greedy baselines)    prefix-shared batch scoring
   (Mac: osx-64 proc) one prefill per step (n_cmpl)
       +--> step record: candidates, scores, choice, tokens, ms
            --> trajectories.parquet --> MLflow, HTML report
 Offline
  labeler  label-game rollouts -> branching states -> K MC continuations
           (replay-from-reset | deepcopy) -> labels.parquet (q, var, K, depth)
  trainer  PRM LoRA (pointwise | pairwise) -> temperature scaling -> gate
           PRM-filtered RFT (QLoRA 1.7B) -> greedy gate; GRPO-lite (0.6B) stretch
  bench    pinned stack per platform -> equal-latency table, frontier, reliability
```

Per step, the controller validates and dedups the N samples from `policy-svc`, scores survivors in one `prm-svc` batch (beam and lookahead also take `env-pool` snapshots), then acts or abstains and logs the step. Offline, the labeller drives the same services in batch; the trainer turns labels into a PRM and searched trajectories into an RFT adapter. Each artefact passes a tuning-split gate before a service loads it.

## 7. Component Design

### 7.1 Environment Layer
- `reset(game_id)`, `step(action)`, `snapshot()`, `restore(handle)`. TextCraft snapshots with `deepcopy`. ALFWorld has no clone: a snapshot is the action log, and `restore` replays it from `reset`. S0 replays 100 random prefixes, asserts identical observations and times a replay.
- W workers (default 8), one environment each, reset per game; `restore` resets rather than reloads. Per-environment resident memory (RSS) is measured in S0 (§8).

### 7.2 The Agent Loop

| Element | Specification |
|---|---|
| **Observation** | [static system + few-shot + goal] → [last 2 turns] → [**rule-based progress note**, ≤120 tokens, no LLM call: held object, visited receptacles, last failed action (ALFWorld); inventory, crafted items (TextCraft)] → [observation]. Static block cached across steps; whole prompt shared by the N samples; ≤1.5K tokens, oldest turn truncated first. |
| **Actions** | TextWorld commands (`go to`, `take`, `put`, `open`, `heat/cool/clean … with`, `examine`, …), one admissible item in protocol A; TextCraft's three actions (§5.2) behind a regex syntax filter. Controller-only: `ASK_HELP`; tools `prm.score`, `env.snapshot/restore`, `judge.score`. |
| **Generate** | N ∈ {1, 2, 4, 8, 16} from one prefill (`llama-server` `n`/`n_cmpl`, `-np ≥ N`), T=0.8 (tuned), top-p 0.8, ≤64 output tokens: optional ≤48-token thought, then `<action>…</action>`. |
| **Invalid** | Drop unparseable, non-admissible or filter-failing samples; resample up to 2 more times. Then protocol A takes the admissible command nearest the likeliest sample (normalised edit distance); TextCraft emits `inventory`. Rate logged per arm. |
| **Dedup** | Lower-case, collapse whitespace, strip articles; protocol A: exact admissible string. |
| **Rank / act** | PRM scores (state, action) with the thought stripped, so a persuasive thought cannot inflate it. Argmax calibrated score; ties go to higher log-probability. A lone survivor is still scored for abstention. |
| **Budgets** | Per-step P95 budget = 8B greedy P95 on the same machine; each chip's service-level objective (SLO) comes from S0. N, beam and lookahead are fixed per config and in the run hash. |

```python
def run_episode(env, game_id, searcher, budget):
    state, low, t0 = State.initial(env.reset(game_id)), 0, clock()
    for t in range(budget.max_steps):  # 30: success or this cap, nothing else scores
        if clock() - t0 > budget.runaway_s:  # runaway guard, never scored
            return Result(None, infra_abort=True)
        d = searcher.select(state, budget)  # generate N, validate, dedup, score
        low = low + 1 if d.best_score < budget.tau else 0  # tau = -inf in claims 1, 4
        if low >= 3:  # abstain: k=3 consecutive steps below tau (claim-3 arms)
            return Result(False, abstained=True)
        obs, reward, done, _ = env.step(d.action)
        state = state.advance(d.action, obs); log_step(state, d)
        if done:
            return Result(reward > 0)
    return Result(False)
```

**Runaway guard:** per-arm wall-clock cap of 1.5 × 30 × that arm's S0-pilot P99 total step latency. A hit (**infra-abort**) re-runs once from reset, same seed; a second drops the game from both arms of every pair it is in (pre-registered).

### 7.3 Search Controller
`Searcher.select(state, budget) → Decision` implements:

| Searcher | Rule |
|---|---|
| Greedy; majority vote; log-prob rerank | N=1, T=0; popularity; generator score |
| Prompted judge | P("yes") from one forward pass, its fastest form |
| **PRM best-of-N** | Argmax calibrated PRM score |
| **PRM full ranking** | All admissible actions (A); TextCraft: listed craft commands plus `get` of each listed ingredient |
| Step beam\* | Width 2, N/2 samples per beam, depth 2 in snapshots, ranked by last-step score; commit the best beam's first action, re-plan every step |
| 1-step lookahead\* | Execute the top 2 in snapshots; value = best next-step score of 2 samples |
| Oracle\* | Each candidate's Q from K=2 MC rollouts on the test game; fixed subset (§9) |

\*Simulator-access arms: labelled so everywhere, never called deployable. If full ranking matches policy+PRM on ALFWorld, the gain is short-list ranking; TextCraft tests that with free-form arguments.

### 7.4 Process Reward Model
- **Input:** instruction + goal + window + note as shared prefix, action as suffix. **A:** linear head on the last hidden state, no LM head. **B:** reranker format (query = state, document = action); P("yes") from the yes/no LM-head rows at the last position only (`logits_to_keep=1`, or two rows sliced from `lm_head`). Full-vocabulary logits over 1.5K tokens would add ~0.47 GB per sequence in BF16 (1,536 × 151,669 × 2 B); an S0 test asserts peak memory.
- **Prefix-shared scoring:** one ~1.5K-token prefill, its KV cache expanded across N suffixes of ~20 tokens. A unit test against unshared scoring asserts |Δlogit| < 1e-2 and identical ranking (Spearman 1.0); exact equality only in fp32 on CPU, since BF16 kernels of different shapes round differently.
- **Losses:** pointwise binary cross-entropy (BCE) on soft labels q̂ ∈ [0, 1]; pairwise RankNet on same-state pairs with q̂ᵢ > q̂ⱼ.
- **Training:** LoRA rank 8 or 16, split by game id, early stopping on set-(b) pairwise accuracy; temperature scaling; ECE (10 bins); τ set on tuning episodes for a target abstention precision.

### 7.5 MC Labeller and RFT
**Labelling.** Roll out the policy (T=0.8) on label games and tasks, N=4 candidates per state; keep **branching states** (≥2 distinct canonical actions). Per distinct candidate, run K continuations (restore, execute, continue to success or the cap). q̂ = mean success, stored with variance, depth, game id and policy hash. Label counts are reported with the **informative share**: states with at least one candidate pair whose q̂ differs.

**Base-rate gate (S0).** Nearest published base rate: Qwen2.5-1.5B-Instruct ReAct, 12.8% (GiGPO). At p ≈ 0.13 both K=2 continuations fail with probability 0.87² ≈ 0.76, so most labels would be 0 (AgentPRM-C labelled with a 64.9% SFT-initialized policy). S0 measures 1.7B tuning-split success and the share of branching states with ≥1 success among their K·D continuations. Below ~30%, costed fallbacks apply in order:
1. Continuations from 1.7B majority-vote best-of-4 (~4× the calls).
2. Continuations from the 8B; labels then estimate the 8B's values, and the report says so.
3. SFT warm-start of the 1.7B on ALFWorld's expert plans and a scripted TextCraft expert built from listed recipes; the SFT model becomes the policy and its greedy row joins claims 1 and 4.

| Cost term (assumed until S0) | Value |
|---|---|
| Time | Σ over label sets of S × D × K × L̄ (states × distinct candidates × K × mean remaining steps) ÷ measured calls/s |
| L̄ | ≈ 15–18: most continuations fail and run to the 30-step cap; mean branching depth 12–15 |
| Plan | (a) S≈1,000 at K=2, K=4 on its 10%; (b) ~100 and (c) ~200 states at K=4. D=4 upper bound: ≈13.6K × L̄ = 204K–245K calls (AgentPRM-C: 10K rollouts per iteration) |
| Per call | ~0.6–1K uncached prompt tokens (the 2-turn window shifts; only the static block stays cached) + ≤64 output |
| Throughput | 3–10 calls/s on 6 GB; 0.5–1 on a base M1/M2 Mac, scaled from llama.cpp's Apple Silicon table (7B Q4_0 prefill 108–180 t/s; the 1.7B is ~5× cheaper per token) |
| Projection | ~5–19 h on both machines (1–3 nights), plus ~13.6K ALFWorld restores × measured replay time over W workers |

**Mismatch guard.** Labels must describe the Q4_K_M test-time policy. If vLLM BF16 labels, S0 compares the two on ~500 labelled states (top-action agreement; KL divergence over the union candidate set) against thresholds committed to `prereg/` first (proposed: agreement ≥ 0.90); below them, labelling stays on Q4_K_M. A 50-game success-rate check was dropped: its ~18-pt MDE (2.8·√(0.2/50)) would almost always pass.

**PRM-filtered RFT.** From searched rollouts on policy-training games (§5.4), keep successful episodes and, within them, steps whose chosen score beats the state's mean candidate score. QLoRA NF4 via Unsloth, loss on action tokens, one epoch, length ≤2K (1K if memory runs out). Gate: tuning greedy gain, then one test run. A **success-only filter** ablation isolates the PRM's contribution.

### 7.6 GRPO-Lite (Stretch, Gated)
Qwen3-0.6B on TextCraft policy-training tasks, G=4 × 8 tasks per iteration, ≤60 iterations; one sample per turn, 2-turn window, loss on that turn's action tokens. Group-normalized return = success + λ · mean PRM progress; KL to the reference by switching the adapter off. A success-only reward arm checks whether the policy learns to please the PRM instead of solving tasks.

| Per iteration (turn counts assumed; measured in S0) | ALFWorld, GiGPO-scale | TextCraft GRPO-lite |
|---|---|---|
| Episodes | 8 × 16 = 128 | 4 × 8 = 32 |
| Turns × tokens per window | 30 × ~750 | 15 × ~600 |
| **Backprop tokens** | **~2.9M** (1.7B) | **~0.29M** (0.6B) |

Ten times fewer tokens on a third-size model fits; ALFWorld does not. S0 throughput decides whether the arm runs.

### 7.7 Equal-Latency Protocol and Observability
- **Latency:** per step, from observation ready to action chosen. **Policy+PRM** (sampling and scoring) is claim 1's measure, since neither arm touches the environment while deciding. **Total** adds environment and replay time; it is reported beside policy+PRM and is the frontier's measure for beam, lookahead and oracle. P50/P95 after 20 warm-up steps; CIs resample whole episodes.
- **Pinned:** llama.cpp tag after PR #18663 (Jan 2026; `n_cmpl` skips repeated prompt processing), CUDA/Metal versions, one Q4_K_M pipeline, KV type, context, thinking off, one template. Each model gets its fastest `-ngl` split (layers offloaded to the GPU) from a sweep, so the 8B is not handicapped. AC power, no other GPU process, clocks and temperature logged. Fingerprint: GGUF sha256s; on the Mac, chip, GPU core count, memory bandwidth.
- **Matching:** configs with policy+PRM P95 ≤ 8B greedy P95 on the same machine are eligible. N* maximises tuning-split success among them, is frozen by hash and runs once on test. `bench.py` prints the table and fingerprint.
- **Observability:** a Parquet row per step, an MLflow entry per run (config hash, commit, hardware). Dashboards cover success by arm, per-stage latency, tokens, invalid actions, infra-aborts, abstain rate, ECE and the frontier; the demo viewer shows candidates, scores, choice and τ per step.

## 8. Hardware Profiles

> **Rule: defaults are tuned for the 6 GB card and the Mac. An 8 GB card or vLLM is an opt-in accelerator, never an assumption.**

| | `portable` (default) | `accelerated` (opt-in) |
|---|---|---|
| Policy | `llama-server` Q4_K_M, KV q8_0 | same, KV f16 |
| N | from the S0 latency gate (expect 4) | from its own frontier |
| Labelling | `llama-server`, both machines overnight | vLLM BF16 in WSL2, if the mismatch check passes |

**Memory (estimates, verified in S0).** With `n_cmpl` and the unified KV buffer (`--kv-unified`; check the pinned tag's default), the N samples share one prefix: KV = one prefix + N × ≤64 decode tokens. No sharing is the upper bound.

| Component | 6 GB card | 8 GB card | Mac 16 GB |
|---|---|---|---|
| Qwen3-1.7B Q4_K_M | 1.11 GB | 1.11 GB | 1.11 GB |
| KV, shared prefix (`-c 4096`, covers N ≤ 16) | ~0.23 GiB (q8_0) | ~0.44 GiB (f16) | ~0.44 GiB |
| KV, no sharing (8 seqs × 2,048) | ~0.93 GiB (q8_0) | ~1.75 GiB (f16) | ~1.75 GiB |
| PRM BF16 + scoring activations (last-position logits) | ~1.5 GB | ~1.5 GB | ~1.5 GB |
| CUDA context, 2 processes | ~0.6–1.0 GB | ~0.6–1.0 GB | shared |
| **Compass total** (shared KV to upper bound) | **~3.5–4.6 GB** | ~3.7–5.5 GB | ~3.1–4.5 GB |
| Qwen3-4B Q4_K_M + 2K KV | ~2.5 GB + 0.28 GiB → **fits alone** | fits | fits |
| Qwen3-8B Q4_K_M + 2K KV | 5.03 GB + 0.28 GiB → **partial offload** | fits | fits |
| Qwen3-8B judge beside the 1.7B policy | ~7–8 GB → not run (offloaded; latency not comparable) | not run | fits |
| Qwen3.5-9B judge | not run | not run | ~5–6.6 GB, loaded alone |

**Host RAM (16 GB).** WSL2's VM defaults to 50% of host RAM (8 GB), which must hold the env pool, the 8B's offloaded layers, the PRM and labelling, so the repo ships a `.wslconfig` (`memory=12GB`, `swap=8GB`) with this reason beside it. S0 measures the env pool (W × per-env RSS), the offloaded layers (`-ngl` sweep) and the Python services (PRM, controller, labeller).

**What runs where.** Mac: demo, headline latency table, judge arms. WSL2: training, memory-bound table, claim 4. Both label overnight; adapters and label shards cross as files. The frontier keeps the 1.5K-token window at every N, so a large-N change cannot come from a shorter prompt; if prefix sharing fails on 6 GB, that frontier stops at N=8 and N=16 runs on the Mac only.

**S0 go/no-go latency gate (session 2, week 2).** No trained PRM is needed: latency does not depend on head weights. On both machines, over 20 tuning games, Compass P95 at N=4 (`llama-server` with `n_cmpl`, random PRM-A head, prefix-shared scoring) is measured against Qwen3-8B and Qwen3-4B greedy P95.
- **Pass:** Compass P95 ≤ 8B greedy P95 on at least one machine. That chip's absolute P95 becomes its SLO, not a pass/fail number: base M1 to M4 differ about 2× in llama.cpp prefill (7B Q4_0 PP512: 108 t/s on a 7-core M1, 221 t/s on a 10-core M4; discussion #4167), and Pro/Max chips are faster still.
- **On failure, in order:** drop the thought (48 → 0 tokens); shrink the window (1.5K → 1K, for every N); KV q8_0; verify prefix-shared scoring; N 4 → 2; lookahead off.
- **Declared now, so it cannot be renegotiated:** if N=2 matches the 8B nowhere, claim 1 is reported not testable on this hardware and the project re-centres on claims 2–3. If it matches only on 6 GB, claim 1 runs memory-bound only.

## 9. Evaluation Plan

| Area | Metrics | Against |
|---|---|---|
| Claim 1 | Success (ALFWorld pooled + valid_unseen; TextCraft by depth), policy+PRM P50/P95, tokens/episode, infra-aborts, per platform | 8B greedy; 4B greedy (secondary) |
| Claim 2 | Pairwise accuracy on set (c) (primary); AUROC (area under the receiver-operating-characteristic curve) and Spearman vs K=4 labels; BoN success at fixed N (tuning); scoring ms per runtime | Qwen3.5-9B judge (primary), Qwen3-8B judge, oracle |
| Claim 3 | Success vs N/beam/lookahead vs tokens vs P95 (total latency for simulator arms); N*; chosen action's PRM score vs oracle MC value as N grows; ECE; selective success vs coverage | Majority, log-prob, PRM full ranking |
| Claim 4; GRPO-lite | ALFWorld greedy success, no search; TextCraft greedy success (0.6B) by depth | Base model and success-only variant of each |
| Labels, repro, portability | K=2 vs K=4 agreement; informative share; GiGPO reproduction (§10); both machines within seed noise | GiGPO 12.8% (valid_seen, T=0.4), 7B 31.2% |

**Ablation ladder** (one change per rung, matched N): greedy → BoN + majority → + log-prob → + prompted judge → + PRM A pointwise → + reranker init (B) → + pairwise loss → + beam/lookahead (simulator-access) → + abstention → oracle ceiling (subset). Protocol B and full ranking run alongside; an ALFWorld-only PRM tested on TextCraft measures transfer.

**Pre-registered statistics**

| Item | Rule |
|---|---|
| n | ALFWorld 274 pooled (valid_unseen 134: breakdown only, §5.1); TextCraft 200; 3 paired sampling seeds |
| MDE | (z₁₋α/₂ + z₀.₈)·√(p_d/n_eff) for paired binary outcomes with discordant fraction p_d; n_eff = n·m/(1+(m−1)ρ), m seeds, intra-game correlation ρ |
| Multiplicity | Holm over claim 1, claim-2 pairwise accuracy and claim-4 greedy lift. Worst step α = 0.05/3 ≈ 0.0167 → multiplier 2.39 + 0.84 = **3.24**, not 2.8 |
| Claim 1 | Intersection-union: beat the 8B on both benchmarks at the Holm level, p = max(p_ALFWorld, p_TextCraft), no further correction. Joint power ≈ 0.8² ≈ 0.64 at tabled MDEs (independent benchmarks); 0.80 needs ~12% larger effects |
| Seeds | Greedy on a deterministic game repeats under every seed, so ρ is far from 0; claim 4 (greedy vs greedy) has ρ = 1. As m grows n_eff → n/ρ: only more games lower the floor |
| Claim 2 | Per informative pair, does the PRM / the judge order it correctly (paired); bootstrap over states clustered by game; ≥400 pairs from ≥100 games and tasks. Illustrative: p_d = 0.30, effective n 200 → 3.24·√(0.30/200) ≈ 12.5 pts. Design effect measured when (c) is labelled, before PRM training |
| Claim 3 | Descriptive; episode-bootstrap CIs, no MDE |
| Pilot (S0) | 1.7B and 8B, 50 tuning games + 50 tuning tasks, 3 seeds → p_d, ρ (TextCraft depth-2 only: a lower-difficulty proxy). MDEs, endpoints, analysis script into `prereg/` before headline runs |
| Underpowered | Seed-saturated floor 3.24·√(p_d·ρ/n) > 10 pts → endpoint pre-registered as underpowered, or claim narrowed before the run, never after |
| CIs | Paired bootstrap over games (10K resamples); McNemar secondary. Uncorrected, labelled secondaries: vs Qwen3-4B, TextCraft by depth, valid_unseen alone |

| MDE at p_d = 0.20 | n | ρ=1 (claim 4; claim-1 bound) | ρ=0.5, m=3 (claim 1, illustrative) |
|---|---|---|---|
| ALFWorld pooled | 274 | ~8.8 pts | ~7.1 pts |
| valid_unseen (breakdown) | 134 | ~12.5 pts | ~10.2 pts |
| TextCraft test | 200 | ~10.2 pts | ~8.4 pts |

**Compute tiers (pre-registered; hours filled in from S0 throughput).** Hours per arm per machine = episodes × mean steps × measured s/step (total latency). Illustration: 20 steps × 1.5 s on full test at 3 seeds ((274 + 200) × 3 = 1,422 episodes) ≈ 12 h per machine.

| Tier | Arms | Items × seeds | Machine |
|---|---|---|---|
| 1 | Compass frozen config, 8B and 4B greedy (claim 1); base and RFT 1.7B greedy (claim 4, ALFWorld 274) | Full test; Compass × 3 seeds; greedy × 1 if S0 finds repeat greedy runs identical on 20 games, else × 3 | Mac + WSL2; claim 4 WSL2 |
| 1 | PRM vs Qwen3.5-9B judge (claim 2) | Set (c), no episodes | Mac |
| 2 | Ablations, N sweep, beam, lookahead, protocol B, full ranking, judge BoN, transfer, GRPO-lite | Tuning × 1; test × 1 for reported rungs | Either |
| 3 | Oracle (subset ceiling) | 30 ALFWorld + 40 TextCraft test items × 1 | WSL2 |

The oracle needs ≈ N × K × L̄ = 4 × 2 × 15 = 120 policy calls per step: ~126K calls for the subset (~3.5–12 h on WSL2, est.). If Tiers 1–2 project past 20 machine-nights, Tier 2 drops its test seed.

**Honest reporting.** A claim whose CI includes zero is reported **not supported** beside the passes. Other papers' numbers are reference rows (step caps, prompts, splits, decoding, quantization differ), and tables say so. Simulator-access arms are never called deployable; the 6 GB result is labelled memory-bound when the 8B offloads; judge latency is per runtime, never a ratio; TextCraft success is always shown by depth (2 vs 3–4). Infra-aborts and invalid rates per arm sit in the headline table. Nothing is tuned on test; test runs load a frozen config by hash. The optional API row (model, date, cost) is never part of a claim.

## 10. Milestones (19 hours / 15 sessions)

| Sprint | Sessions | Hours | Deliverable | Gate |
|---|---|---|---|---|
| **S0** | 1–4 | 5 | Repo, `uv`, WSL2 (+ `.wslconfig`) and Mac (osx-64 env-pool) envs; env layer; pinned llama.cpp; one-pipeline Q4_K_M for 1.7B/4B/8B; TextCraft split enumeration; GiGPO reproduction (below); Qwen3-1.7B/4B/8B ReAct on tuning; base rate, informative share; throughput, latency and variance pilot; MDEs committed | **Latency** go/no-go (§8, session 2); **repro** (below); **base rate** under ~30% → §7.5 fallbacks; **throughput:** labelling projected past 3 nights → cut S or shrink set (c) |
| **S1** | 5–8 | 5 | BoN with majority, log-prob and prompted judges (**week-4 demo**); MC labeller; label sets (a)–(c), (c) frozen into `prereg/`; first PRM (A, pointwise) | **PRM quality:** PRM A beats log-prob on tuning-label pairwise accuracy, and its BoN beats majority vote on tuning success |
| **S2** | 9–10 | 3 | PRM grid; frontier (fixed window) + over-optimization; calibration + abstention; beam, lookahead (Tier 2); oracle subset (Tier 3); TextCraft transfer | Grid winner and N* frozen by hash on tuning; beam, lookahead and oracle slip to S3 if S2 overruns |
| **S3** | 11–12 | 2.5 | PRM-filtered RFT + success-only ablation; GRPO-lite if the S0 gate allows | Tuning gain before any test run |
| **S4** | 13–14 | 2.5 | Tier-1 frozen-config headline runs (§9); controller service with budgets; dashboards; `bench.py`; optional WebShop | All `prereg/` endpoints reported |
| — | 15 | 1 | Interview sprint: headline plot, two-minute walkthrough, mock design round | — |

**S0 reproduction.** Reproduce GiGPO's Qwen2.5-1.5B-Instruct ReAct 12.8% with verl-agent's default evaluation (its template, valid_seen, T=0.4, 50 steps, history 2).
- **Primary:** 3 seeds × all 140 valid_seen games (n=420); valid_unseen is reported as our own number. Pass by two one-sided tests (TOST) for equivalence: ours within ±6 pts of 12.8%, about the 95% sampling half-width of a 128-environment reference (our 90% CI at n=420 is ≈ ±2.7 pts).
- **Second point:** Qwen2.5-7B-Instruct ReAct (31.2%), 1 seed × 140 games, one night on the Mac; its 95% CI (±7.7 pts at n=140) must cover 31.2%.
- **Fallback:** Llama-3-8B ReAct on TextCraft (12.5%, THREAD).

## 11. Risks & Mitigations

| Risk | Mitigation |
|---|---|
| ALFWorld's admissible list makes search easy | TextCraft co-headline, protocol B, full ranking on both (§5.2) |
| Weak base policy gives mostly-zero MC labels | Base-rate gate and fallback ladder (§7.5); informative share reported |
| MC labelling takes 1–3 nights | K=2, S≈1,000, branching states only, both machines; throughput gate (§10) |
| Evaluation compute exceeds machine time | Tiers, oracle subset, Tier 2 falls back to tuning (§9) |
| TextCraft PRM trained on depth 2; ~two thirds of test is depth 3–4 | Pre-registered per-depth reporting (§5.2); ALFWorld-trained PRM's TextCraft transfer as a robustness check |
| Equal latency depends on serving details | One pinned stack and quantization pipeline per platform; fastest offload split per model; published `bench.py` (§7.7) |
| 6 GB win is a memory artefact | Mac fair fight is the headline; the 4B, which fits without offload, is secondary (§1) |
| TextWorld has no arm64 wheel | osx-64 (Rosetta) env-pool process from day 1 (§4.2) |
| Prefill sharing across the N samples | `llama-server` `n`/`n_cmpl` (PR #17775, Dec 2025; prompt-skip fix PR #18663, Jan 2026) on a post-fix build, `-np ≥ N`; S0 checks the prompt-eval token count per request; vLLM `n=` as accelerator |
| QLoRA or RL arm over budget on 6 GB | 1K sequences, checkpointing, optional free Colab; GRPO capped and gated by S0 |
| Prior art: AgentPRM-C (Choudhury); AgentPRM-X (Xi et al.: agent PRMs on WebShop, BabyAI, TextCraft, with beam search and TD+GAE labels); Liu et al. 2025 (a small policy plus a PRM beats far larger models on math); AgentRM; GiGPO; Math-Shepherd | Cited. The delta none of them covers: equal per-step wall-clock on a fixed consumer device (not equal samples or FLOPs), a reranker-initialised value function, and over-optimization and abstention under a latency SLO |
| Sibling overlap | Teacher→student distillation stays in Shellwright (02) and Spelunker (03); Compass's RFT is self-improvement through its own PRM. HarnessForge (07) evolves harnesses and configs; Compass searches only N, beam, lookahead and τ. BlackBox (05) records and forks LLM calls; Compass's snapshot/restore is a ~100-line env wrapper for MC labels, not a replay framework. Siblings use Qwen3.5; Compass uses Qwen3 on purpose (§4.1) |

## 12. Résumé Bullets (experience voice; numbers filled in as sprints land)

**Rule:** a bullet whose claim comes out **not supported** is rewritten as a finding (e.g. "showed that a 1.7B + PRM does not beat an 8B at equal latency on a 16 GB Mac; traced the gap to X") or deleted, never kept with a weaker number. Bullet 1 keeps only the clause the results support.

- Built a **test-time search agent** in which a Qwen3-1.7B policy proposes N actions per step and a **0.6B process reward model ranks them**, beating Qwen3-8B greedy decoding by **X points on ALFWorld and X on TextCraft** at equal or lower P95 per-step latency on a 16 GB Apple Silicon laptop where both models fit (and on a 6 GB GPU where the 8B must offload).
- Trained the PRM on **X Monte-Carlo-labelled branching states with no human annotation**; reranker initialization plus a pairwise RankNet loss raised held-out pairwise accuracy from **X to X**, above a prompted Qwen3.5-9B judge (**X vs X**), with scoring latency of **X ms vs Y ms** on the same Mac.
- Mapped the **compute-optimal frontier** (best-of-N, step beam, lookahead) over success, tokens and P95, located over-optimization at **N = X**, and shipped a calibrated abstain-and-ask threshold (ECE **X**) avoiding **X%** of failed episodes at **X%** coverage.
- Raised greedy success with no search from **X% to X%** via **PRM-filtered rejection-sampling fine-tuning** (QLoRA), **X points** above a success-only filter.
- Built a **reproducible equal-latency benchmark** pinned per platform (WSL2/CUDA, macOS/Metal) with pre-registered MDEs and paired cluster-bootstrap CIs, after reproducing a published ALFWorld baseline within **X points**.

### Placement Note
List these under EXPERIENCE only in an entry naming a **real organization and a real engagement**:
1. **A real engagement**, such as the mentoring organization with a truthful title (`ML Engineering Practicum`) where a record exists.
2. **A real collaboration** with a lab, startup or open-source org hosting the work.
3. **An open-source project entry** with real commit history.

**Never invent an employer or dates.** North American employers verify employment, and a fabricated entry is grounds for a rescinded offer. A public repo with a reproducible `bench.py` survives scrutiny better than an unverifiable internship line.

## 13. Why This Reads as Real Company Work

- **The constraint is a latency SLO**: better quality without breaking P95 on shipped hardware, as inference teams set goals.
- **Generate-then-rank is production architecture** with recsys baselines, and **labels come from the environment**, not annotators.
- **The statistics are an experimentation platform's**, and **the benchmark ships** as a pinned, rerunnable script.
- **Abstention is a product decision** with measured coverage.

## 14. Interview Leverage

- **ML system design** ("an on-device agent under a fixed per-step budget"): candidate generation, ranking, budgets, abstention, where each millisecond goes.
- **Recsys / ranking**: pointwise vs pairwise, calibration, full ranking vs candidate generation, Goodhart.
- **LLM / agents**: PRMs, test-time compute, MC labelling, RFT, multi-turn GRPO masking.
- **ML depth**: compute-optimal inference (Snell et al.; Liu et al.), reward over-optimization (Gao et al.), K=2 label noise.
- **Experimentation**: MDE arithmetic, paired designs, intra-cluster correlation, Holm and intersection-union tests, honest nulls.
- **Coding**: replay-from-reset wrapper, prefix-shared scorer, cluster bootstrap.
- **Behavioural**: the pre-declared S0 latency fallback answers "a plan that did not survive the data".

## 15. References

- AgentPRM-C (Choudhury): https://arxiv.org/html/2502.10325v1
- AgentPRM-X (Xi et al., Nov 2025): https://arxiv.org/abs/2511.08325
- Liu et al., "Can 1B LLM Surpass 405B LLM? Rethinking Compute-Optimal Test-Time Scaling": https://arxiv.org/abs/2502.06703
- AgentRM (Xia et al., ACL 2025): https://arxiv.org/abs/2502.18407
- GiGPO: https://arxiv.org/html/2505.10978 · verl-agent (evaluation defaults in `ppo_trainer.yaml` and `examples/gigpo_trainer/run_alfworld.sh`): https://github.com/langfengQ/verl-agent
- ALFWorld: https://arxiv.org/pdf/2010.03768 · https://github.com/alfworld/alfworld
- TextWorld 1.7.0 (PyPI): https://pypi.org/project/textworld/
- ADaPT / TextCraft (Findings of NAACL 2024): https://arxiv.org/html/2311.05772v2 · https://github.com/archiki/ADaPT
- THREAD: https://arxiv.org/html/2405.17402
- Math-Shepherd: https://arxiv.org/abs/2312.08935
- Snell et al., test-time compute: https://arxiv.org/abs/2408.03314
- Gao et al., reward model overoptimization: https://arxiv.org/abs/2210.10760
- Qwen3 cards: https://huggingface.co/Qwen/Qwen3-1.7B · https://huggingface.co/Qwen/Qwen3-0.6B · https://huggingface.co/Qwen/Qwen3-4B · https://huggingface.co/Qwen/Qwen3-8B-GGUF · https://huggingface.co/Qwen/Qwen3-1.7B-GGUF (Q8_0 only) · https://huggingface.co/unsloth/Qwen3-1.7B-GGUF (size reference) · https://huggingface.co/Qwen/Qwen3-Reranker-0.6B
- Qwen3.5-9B: https://huggingface.co/Qwen/Qwen3.5-9B
- Unsloth Qwen3.5 guide: https://unsloth.ai/docs/models/qwen3.5/fine-tune
- llama.cpp #22384: https://github.com/ggml-org/llama.cpp/issues/22384
- llama.cpp server `n`/`n_cmpl`: https://github.com/ggml-org/llama.cpp/pull/17775 · fix https://github.com/ggml-org/llama.cpp/pull/18663
- llama.cpp Apple Silicon benchmarks: https://github.com/ggml-org/llama.cpp/discussions/4167
- WSL `.wslconfig`: https://learn.microsoft.com/en-us/windows/wsl/wsl-config
- OpenPipe ART: https://github.com/OpenPipe/ART
- WebShop: https://github.com/princeton-nlp/WebShop
