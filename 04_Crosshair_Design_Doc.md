# Crosshair — Design Document

**Self-Labelled Data, Confidence-Gated Zoom and a Planner–Grounder Agent for GUI Grounding on a 6 GB Laptop GPU**

| | |
|---|---|
| Status | Proposal |
| Target duration | ~10–12 weeks (19 mentoring hours / 15 sessions + independent work) |
| Platforms | Windows 11 laptop (16 GB RAM, RTX 3060-class, **6 GB VRAM default**, 8 GB if confirmed) and Apple Silicon Mac (16 GB unified, no CUDA), one repository |
| Deployment | 100% local. No rented servers, no paid GPU cloud. Free Colab/Kaggle only as an optional accelerator; one commercial API only as an optional comparison, capped at $20 total |
| Core stack | Qwen3.5-0.8B / 2B / 4B (Apache-2.0) · Unsloth bf16 LoRA (low-rank adaptation) · llama.cpp `llama-server` (GGUF weights + mmproj vision projector) · mlx-vlm · Playwright · pywinauto (Windows UI Automation, UIA) · pyobjc (macOS Accessibility, AX) · BrowserGym MiniWoB++ · FastAPI · Optuna |

---

## 1. The Product Thesis

A computer-use agent that plans perfectly but clicks four pixels left of the right icon still fails, and on professional software that is common. **ScreenSpot-Pro** (SSPro) targets average **0.07%** of the screenshot (2.01% on the original ScreenSpot); at release the best grounding model scored **18.9%** ([Li et al., 2025](https://arxiv.org/abs/2504.07981)). Small open models closed much of the gap (Qwen3.5-2B card: **54.5**, non-thinking), yet a 2B model still misses about half the targets on high-resolution professional screens.

Local is a product requirement, not only a budget: screenshots are among the most sensitive data a product captures, and enterprise IT resists sending every frame to a hosted model. **How much grounding can a 0.8B–2B model reach on a 6 GB laptop GPU, and what does each point cost in latency and memory?**

Four claims, each against a **locally reproduced baseline** at the served quantization, thinking mode stated:

1. **Self-labelled LoRA lift.** One epoch of bf16 LoRA on ~30k self-minted samples lifts SSPro over our base by at least the pre-registered minimum detectable effect (MDE, §9). Non-inferiority (3-point margin): fine-tuned 0.8B vs base 2B on 6 GB; fine-tuned 2B vs base 4B given 8 GB or the MLX LoRA spike (§8). A ScreenSpot-v2 (SSv2) regression gate must pass.
2. **Confidence predicts errors through quantization and fine-tuning.** Coordinate-token log-probability, multi-sample agreement or zoom agreement reaches AUROC (area under the receiver operating characteristic curve) ≥ 0.70 at bf16, Q8_0 and Q4_K_M, before and after LoRA. Zoom Consistency's 0.60 (§5.1) is the zoom-agreement reference.
3. **Conditional zoom beats equal compute**: extra pixels spent only where the model is unsure buy accuracy per query, not per parameter. At matched mean grounder queries, gated zoom beats always-zoom and k-sample voting without zoom, on a ≥ 20-config accuracy × latency × VRAM Pareto on both machines. A reimplemented UI-Zoomer gate is the external reference.
4. **A 2B grounder plus a 4B planner beats a single 4B agent** in MiniWoB++ successes per hour on the 6 GB card, under the memory configuration S0 selects (§8; rule in §9). An oracle-grounding ablation (§7.7) measures the grounding share of failures.

   **H1: grounding, not planning, binds a small local GUI agent**, and free resources can fix it. Claim 4 tests H1; MiniWoB++, where a 4B planner can bottleneck, may refute it. Crosshair is thus an agent project, not a detector.

**Product angle: adapt on-device.** The DOM (document object model) and accessibility-tree labelling behind OS-Atlas and UGround can run overnight on a user's own apps. It adapts a 0.8B–2B grounder to internal software with no screenshot leaving the laptop. Tested on held-out native apps (§9).

**Prior art.** Crosshair does not claim the gate as new: three 2026 papers already use uncertainty around zoom (rows 2–4). It adds the 6 GB budget, self-minted native-app data and quantized serving, and asks what they leave open: **does the gate survive quantization and LoRA at 0.8B–2B on consumer hardware, and does it raise end-to-end agent throughput?**

| Work | Contribution |
|---|---|
| ScreenSeekeR (SSPro paper); RegionFocus (ICCV 2025) | Zoom search for GUI grounding |
| UI-Zoomer (arXiv 2604.14113) | **Uncertainty-gated zoom**: training-free gate on sample agreement + token confidence (two of our three signals); 7B and 72B models |
| AutoFocus (2605.02630) | **Uncertainty-placed zoom**: where, not whether |
| Zoom Consistency (2604.15376) | **Zoom agreement as confidence**, used to route between two models, not to gate zoom |
| VISTA (inclusionAI, 2606.14579) | Qwen3.5-based 4B and 9B grounders trained for multi-view crop consistency by reinforcement learning (RL; GRPO, group relative policy optimization). Crosshair tests a supervised fine-tuning (SFT) analogue at 0.8B–2B |
| ZonUI-3B (2506.23491) | Small-data consumer-GPU grounder: 24K examples, one RTX 4090 |
| OS-Atlas · UGround · Jedi · GroundCUA | Synthetic corpora: 13M+ elements · 10M elements, 1.3M screenshots · 4M examples · 3.56M human-verified elements |

**Why it is surprising.** Qwen3.5's vision encoder spends one token per 32 × 32 pixel cell (patch 16, 2 × 2 merge). A native 3840 × 2160 screenshot is ~8,100 visual tokens. A 12-pixel icon is under half a token wide, and 4 pixels wide after downscaling to 1280 × 720 for 6 GB. Two zoom levels restore it for ~2,700 tokens across three calls, fewer than one native pass.

## 2. Goals & Non-Goals

### Goals
- **Data engine** with quality gates and a decontamination report (§5.2, §7.1–7.3).
- **bf16 LoRA** on Qwen3.5-0.8B (2B where 8 GB or MLX allows), SSv2-gated (§7.4).
- **Grounder service**: cascade, three confidence signals, threshold calibrated off the test set (§7.5–7.6).
- **Planner–grounder agent** on BrowserGym MiniWoB++ (`file://`), logged per step (§7.7).
- **Pareto sweeps** over size, quantization, pixel cap, depth and threshold on both machines, after **reproducing one published number** (§10).

### Non-Goals
- Pre-training or RL post-training; test-time search with a reward model is sibling **01 Compass**.
- Live websites or OSWorld VMs: everything runs offline from files.
- Agent safety (sibling **06 Ctrl-Z**); record/replay infra (sibling **05 BlackBox**, compatible with our JSONL step logs).
- Hardware beyond the two laptops; anything needing > 8 GB VRAM is optional.

## 3. Alignment With Prior Experience

| Prior experience | Crosshair module |
|---|---|
| **CAS**: patented cascaded binary-encoding retrieval; Precision@K / latency / storage over 20+ configs | **Zoom cascade** and **≥ 20-config Pareto**: the same method in pixel space |
| **Changsheng**: AutoML hyper-parameter agent; two-stage sector → stock selection | **Optuna** over LoRA rank; **scout → grounder** cascade (0.8B proposes, 2B localizes); gated adapter promotion |
| **Sina Weibo**: scheduled ETL (extract-transform-load), metrics, alerting | **Data-engine gates**, per-shard QC (quality-control) reports, alert thresholds |
| **College Application Agent**: embedding-based retrieval (EBR) behind FastAPI; LangGraph Writer–Critic | **Planner–grounder loop**, **`grounder-svc` FastAPI**; the confidence gate plays critic |
| **DiDi**: backend; A/B testing | Paired McNemar tests, champion/challenger adapters |
| **Music Genre**: PCA (principal component analysis), K-means | Screenshot-embedding clustering for dedupe and diversity sampling |

**Interview one-liner:** *"At CAS I patented a cascaded retrieval scheme and measured its accuracy–latency–storage frontier. In Crosshair I moved the cascade into pixel space: a 2B model on a 6 GB laptop GPU zooms only when it is unsure, trained on screenshots that label themselves."*

## 4. Stack Choices — and Why

### 4.1 Models

Qwen3.5 small models: Apache-2.0 image-text-to-text (March 2026), hybrid Gated DeltaNet / gated-attention language model. Thinking is on by default in the 4B/9B cards, off in the 0.8B/2B cards (which report non-thinking numbers only) and in Unsloth's templates for all four. Every request sets and logs `enable_thinking` (false unless stated). Sizes: `unsloth/Qwen3.5-*-GGUF`.

| Role | Model | Published reference | Q4_K_M + mmproj F16 | bf16 LoRA (Unsloth) |
|---|---|---|---|---|
| 6 GB training target; scout | **Qwen3.5-0.8B** | SSPro 46.5, non-thinking (card) | 0.53 + 0.21 = **0.74 GB** | ~3 GB |
| Main artifact; 8 GB training target | **Qwen3.5-2B** | SSPro 54.5, non-thinking (card) | 1.28 + 0.67 = **1.95 GB** | ~5 GB |
| Planner; larger baseline | **Qwen3.5-4B** | SSPro 60.3 (card, mode unstated); SSv2 90.4, OSWorld-G 54.4 (VISTA card) | 2.74 + 0.67 = **3.41 GB** | ~10 GB, free tier only |
| Ceiling (Mac) | **Qwen3.5-9B** | SSPro 65.2 (card); SSv2 91.9, OSWorld-G 63.1 (VISTA card) | 5.68 + 0.92 = **6.60 GB** | out of scope |
| GUI specialist | **UI-TARS-1.5-7B** (Qwen2.5-VL arch.) | SSPro 49.6 (card) | community Q4_K_M ~4.7 GB + mmproj | — |
| Optional | **Holo-3.1-4B** (H Company, Qwen3.5-based) | unknown (card table is an image) | community `g023/Holo-3.1-4B-GGUF`, Q4_K_M 3.07 GB; unofficial, mmproj and fidelity unverified | — |

Holo-3.1-4B's lineage is ambiguous (Hugging Face page: 5B params; model tree rooted at Qwen3.5-0.8B-Base); it stays only if S0 shows a working mmproj and sane coordinates on 50 SSPro items.

**Why this family:** the only open family with published SSPro numbers at 0.8B, 2B, 4B and 9B from one vendor, a clean size axis; Unsloth ships vision LoRA notebooks for 0.8B–4B. **No QLoRA** (quantized LoRA): Unsloth calls it "not recommended" for Qwen3.5 due to higher-than-normal quantization differences.

### 4.2 Runtimes

| Platform | Inference | Training |
|---|---|---|
| **Windows + NVIDIA** | `llama-server` (CUDA) with `--mmproj`: OpenAI-compatible API, JSON-schema output, token log-probs | Unsloth, native Windows pip (transformers v5 required) |
| **macOS** | `llama-server` (Metal) default; `mlx-vlm` 4-bit as the measured alternative | mlx-vlm LoRA **S0 spike** (Qwen3.5 vision LoRA on MLX unverified): the no-cloud route to a 2B adapter when Windows has 6 GB |
| **Both, fallback** | transformers v5 bf16 for reference runs and quantization deltas | — |

- **Not Ollama**: it ships its own `qwen3.5` vision builds (0.8b–9b, including `-mlx` tags), but per Unsloth a Qwen3.5 GGUF with a separate mmproj does not work there. We also need `llama-server` log-probs and JSON-schema output on our exact weights. **Not vLLM**: no native Windows support.
- **Pin everything**: llama.cpp's multimodal doc does not yet list Qwen3.5, so we pin the llama.cpp commit, GGUF revisions and mlx-vlm version. A golden-output test (10 screenshots → fixed coordinates) fails the build on changed predictions. Every launch sets `--image-min-tokens` / `--image-max-tokens` (the default, 0, reads the cap from GGUF metadata and can silently downscale); each request logs its image-token count.
- A free Colab T4 has no bf16; runs there are fp16 and labelled so.

### 4.3 Libraries
Playwright (`elementFromPoint` occlusion checks, device scale factor) · pywinauto `uia` backend · pyobjc `AXUIElement` (role, title, position, size) · `browsergym-miniwob`, MiniWoB++ pinned at commit `7fd85d71` · `imagehash` pHash (perceptual hash) · `statsmodels` McNemar · `uv`, `pathlib`, Pydantic Settings for Windows/macOS parity.

## 5. Benchmarks & Datasets

### 5.1 Measured benchmarks (all offline)

| Benchmark | Size | Role |
|---|---|---|
| **ScreenSpot-Pro** | 1,581 instructions, one per screenshot; 23 apps, 3 OSes; point-in-box; 3.38 GB; MIT | Primary metric, per group (Dev, Creative, CAD, Scientific, Office, OS) and text vs icon; run time per config to be measured (budget ≤ 1 h) |
| **ScreenSpot-v2** | 1,272 samples, mobile / desktop / web; Apache-2.0 | Regression gate for every adapter |
| **OSWorld-G** | 564 samples, no VM, incl. 54 refusal items (target not on screen) | Milestones, on the 510 non-refusal items, as our grounder cannot abstain (published scores include refusals) |
| **MiniWoB++ (BrowserGym)** | 125 tasks × 5 seeds = 625 episodes, 10-step cap; 160 × 210 px task area | End-to-end success, wall-clock, failure attribution |
| **MiniWoB-4K** (ours) | Same tasks in a 3840 × 2160 viewport (task ≈ 0.4% of frame), downscaled to the pixel cap; S0 checks BrowserGym's viewport option | The only agent setting that reports the zoom rung |

**Published baselines** (their own harnesses, sources and modes; never in our columns).

| Benchmark | Model | Score | Source |
|---|---|---|---|
| SSPro | OS-Atlas-7B / UGround-7B / Qwen2-VL-7B | 18.9 / 16.5 / 1.6 | ScreenSpot-Pro paper |
| SSPro | ScreenSeekeR (GPT-4o + OS-Atlas-7B) | 48.1 | ScreenSpot-Pro paper |
| SSPro | RegionFocus (Qwen2.5-VL-72B) | 61.6 | RegionFocus |
| SSPro / SSv2 / OSWorld-G | VISTA-4B (Qwen3.5-4B + VISTA RL) | 64.2 / 93.8 / 61.2 | VISTA card |
| SSPro gain from gated zoom | UI-Zoomer on Qwen2.5-VL-7B · UI-Venus-72B | +13.4 (27.6 → 41.0) · +8.6 | UI-Zoomer, Table 3 |
| Zoom-agreement AUROC | KV-Ground-8B / Qwen3.5-27B | 0.60 (abstract does not say which model) | Zoom Consistency |
| SSv2 / SSPro / OSWorld-G | Jedi-3B · Jedi-7B | 88.6/36.1/50.9 · 91.7/39.5/54.1 | Jedi paper |
| MiniWoB++ | GenericAgent: GPT-4o / Llama-3.1-70B / Claude 3.5 Sonnet | 63.8 / 57.6 / 69.8 | BrowserGym paper |
| MiniWoB++ | Explorer-7B, zero-shot | 53.26 | Explorer (46-task subset × 4 runs) |

### 5.2 Training data — minted, not downloaded
- **Web (~60%)**: synthetic pages (random themes, density, fonts, device scale 1–2, viewports up to 3840 × 2160) plus a one-time S1 snapshot of N public pages (HTML + assets; robots.txt and terms of service respected), rendered offline and listed in the data card. SSv2 web-split sites are excluded; screenshots stay local.
- **Native (~40%)**: 3–5 free apps per OS outside the **benchmark-app ban list**: SSPro's 23 apps; every OSWorld app (Chrome, LibreOffice Calc / Writer / Impress, GIMP, VLC, Thunderbird, VS Code, Ubuntu system UI), since OSWorld-G screenshots come from OSWorld rollouts; OS shells (Explorer, Finder, System Settings). Candidates, fixed in S1 after list and tree checks: Inkscape, Notepad++, Krita, Audacity, Paint.NET; TextEdit, Preview, Keynote.
- **Cap: ~30k rendered samples, one epoch**, counted after 1–3 crop scales per triple (§7.4): ~10–15k distinct triples, sized from samples/s at the week-2 go/no-go (§10). Estimate, to be measured: 30k × ~1,000 visual tokens ≈ 3e7 tokens; at ~6–8 × params FLOPs per token (LoRA, gradient checkpointing), ~2e17 FLOPs for 0.8B and ~5e17 for 2B, so 2B may not fit one night.
- **Run budget**: ≤ 3 full-size runs (0.8B main; 2B main if 8 GB or MLX allows; one re-run); ablations on 0.8B only, 10k cap: web-only vs web + native (core); Jedi slice, frozen vs tuned vision layers (stretch).
- **Control (stretch)**: a 10k Jedi slice, same recipe, tests self-minted vs equal-sized public data. Its OSWorld and WindowsAgentArena (WAA) rollout screenshots are removed first. If provenance cannot be filtered per sample, this arm reports SSPro only and flags OSWorld-G as contaminated.

### 5.3 Demo data
4K screenshots of professional apps captured by the mentee. Any app in a demo is excluded from training.

## 6. Architecture

```
OFFLINE: DATA ENGINE
  web minter (Playwright) | UIA harvester (Win) | AX harvester (Mac)
     -> (screenshot, box, role, name, app, dpi)
  QUALITY GATES  box in viewport, side >= 4 px, unoccluded (elementFromPoint
                 / z-order), non-blank crop, pHash dedupe, benchmark-app ban
                 (SSPro + OSWorld apps + OS shells), per-shard QC + alerts
  INSTRUCTION WRITER  Qwen3.5-4B text-only: templates -> paraphrases
                 (literal / functional / positional) -> look-alike mining
  SHARDS + data card -> TRAINER  Unsloth bf16 LoRA + Optuna
                 (0.8B on 6 GB | 2B on 8 GB) -> merge -> GGUF Q4_K_M + mmproj
                 | MLX 4-bit  -> PROMOTION GATE (ScreenSpot-v2)
                                      |
ONLINE: GROUNDER-SVC (FastAPI)        v
  POST /v1/ground {image, instruction, budget} -> {x, y, conf, depth, ms}
  coarse pass @ pixel cap -> gate g >= tau ? answer
                          : crop 1/2 x 1/2 around guess, upscale, re-query
  backends: llama-server (CUDA | Metal) | mlx-vlm | transformers (ref)
                                      ^ target description(s)
AGENT: BrowserGym MiniWoB++ / MiniWoB-4K (file://)
  goal + AXTree text (no ids) + last 5 actions + last error
     -> PLANNER Qwen3.5-4B (non-thinking; text-only on 6 GB)
  -> {action, target, target2, text, key, direction}
  -> grounder-svc (screenshot) -> (x, y) [, (x2, y2)]
  -> mouse_click / mouse_dblclick / mouse_drag_and_drop /
     keyboard_type / keyboard_press / scroll -> new obs
EVAL: SSPro / SSv2 / OSWorld-G runners, Pareto sweeper, AUROC,
      per-step JSONL (latency, depth, conf, VRAM peak), results table in git
```

**Walk-through.** The data engine reads the element tree each UI already keeps and drops elements that are off-screen, occluded or blank. A local text model turns role and name into instructions, including ones that tell look-alikes apart ("the second *Edit* button, in the toolbar"). Shards train an adapter that is merged, quantized and promoted only if ScreenSpot-v2 holds. The agent and the offline benchmarks call the same grounder service, which decides whether to zoom, so offline and online numbers measure one component.

## 7. Component Design

### 7.1 Web minter
About 40 component templates (toolbars, icon grids, dense tables, menus, dialogs, ribbons) with random labels, icons and themes. Each visible interactive element yields box, accessible name and role, and must pass a center-point test (`elementFromPoint` hits it or a descendant) and the 4 px minimum side.

### 7.2 Native harvesters
Walk the foreground window's UIA / AX tree (depth ≤ 25) for control type, name and on-screen rectangle. **Tree quality is the risk**: Qt and Electron apps expose partial trees, and boxes go stale. Gates: box in window, non-zero area, crop variance above a floor (blank crop = stale box), and a 50-element manual audit per app before its data enters a shard. The data card lists per-app counts and rejection rates.

### 7.3 Instructions and hard negatives
Three types per element: **literal** ("click *Export*"), **functional** ("save the image as PNG"), **positional** ("the icon left of the zoom slider"). **Look-alikes** (same role and string-similar names, or near-duplicate icon crops on one screen) get context-naming instructions and form a held-out "confusable" slice. Qwen3.5-4B writes them under a JSON schema; 200 are hand-audited, error rate reported.

### 7.4 Trainer
- **Target**: `[x, y]` center in a 0–1000 frame (the Qwen3.5 / Qwen3-VL, OS-Atlas and VISTA convention). UI-TARS-1.5 emits absolute pixels in its `smart_resize`d frame, so each model gets a unit-tested coordinate parser.
- **Zoom-consistent training**: each triple is rendered at 1–3 crop scales around the target, matching the cascade's upscaled crops: an SFT analogue of VISTA's view-consistent RL groups, at smaller sizes.
- **Recipe**: Unsloth bf16 LoRA, gradient checkpointing, pixel cap 0.9 MP (megapixels; ~900 visual tokens) fixed from the S0 cap curve. Optuna: ≤ 6 trials on a 3k proxy run, rank ∈ {8, 16, 32} × two learning rates.

### 7.5 Confidence signals

| Signal | Extra cost | Definition |
|---|---|---|
| Token log-prob (default gate) | none | Mean log-prob of coordinate digit tokens (`llama-server` log-probs) |
| Sample agreement | k − 1 decodes; image prefix reused via prompt cache (saving to be measured) | k = 5 at T = 0.7; share within 20 normalized units of the medoid |
| Zoom agreement | one query, after zooming | Distance between depth-(d−1) and depth-d guesses in global pixels. Stop rule only, at depth ≥ 1: stop if within r px |

### 7.6 Cascade
```python
def ground(img, instr, tau, D=2, cap=0.9e6):
    region, best = full_frame(img), None
    for depth in range(D + 1):
        view = resize_to_pixel_cap(crop(img, region), cap)  # may upscale
        (u, v), g = grounder(view, instr)  # 0-1000 frame; g = gate signal
        x, y = to_global(region, u, v)
        if best is None or g > best[2]:
            best = (x, y, g, depth)
        if g >= tau:
            return x, y, g, depth
        region = window_around(x, y, region.w / 2, region.h / 2, clamp_to=img)
    return best  # depth D reached: highest-confidence guess
```
- **Gate signal g** is any §7.5 signal; claim 3 is reported per signal.
- **Query accounting** counts every decode, including the k − 1 extra samples, so claim-3 methods are matched on the same unit.
- **Scout variant**: depth 0 on the 0.8B at low resolution, deeper levels on the 2B.
- **τ is never tuned on a test set**: it is calibrated to a target zoom rate on the held-out minted native-app split. An "oracle τ" tuned on SSPro is reported separately as an upper bound.
- **Budget**: max depth D, max queries per call, wall-clock deadline (on expiry: the best guess and its confidence).

### 7.7 Agent loop (MiniWoB++)
- **Planner**: Qwen3.5-4B, non-thinking, temperature 0, JSON-schema output with the §6 fields (`action`: a verb below or `done`; `direction`: up or down). Input: goal, BrowserGym's flattened accessibility-tree text (AXTree) **without element ids**, last 5 actions (target, outcome), previous error. **No screenshot** on 6 GB (`accelerated` adds one).
- **Grounder**: resolves `target` (and `target2`) to (x, y) via the cascade. It never sees ids, so every click tests pixel grounding.

| Planner action | Executor calls (BrowserGym coordinate actions) |
|---|---|
| `click` / `dblclick` / `type` | `mouse_click(x, y)` / `mouse_dblclick(x, y)` / click, then `keyboard_type(text)` |
| `press` | `keyboard_press(key)`; no grounding |
| `select` | `mouse_click(x, y)` on the dropdown, `keyboard_type(option)`, `keyboard_press("Enter")`: native `<select>` popups are absent from Playwright screenshots |
| `scroll` | `scroll(0, ±delta)` from `direction`; delta fixed on the dev subset |
| `drag` | two grounder calls, then `mouse_drag_and_drop(x1, y1, x2, y2)` (drag-*, use-slider, highlight-text tasks) |

- **Silent-failure checks** (executor, identical in every arm): an action can return no error yet commit the wrong value. **Focus**: before `type` (`select`: before its `keyboard_type`), `document.activeElement` must be editable (for `select`, a `<select>`), else the step fails before any key is sent. Playwright emulates page focus, so page focus proves nothing; an OS-level driver also checks the foreground window.
- **Read-back**: after `type`, the focused field's DOM value must end with `text`. Date and time inputs store a normalised value (*03/15/2016* → *2016-03-15*, *3:45 PM* → *15:45*; `enter-date`, `enter-time`), so `text` is converted first; unparseable → n.a. After `select`, the selected option's label (what MiniWoB's `choose-list` reward reads) must equal `option`, case- and accent-folded. `select` is the substitution case: type-ahead in Blink (Chromium's rendering engine) matches a case- and accent-insensitive prefix, so *Niger* can commit *Nigeria* (both in MiniWoB's country list). S0 tags tasks whose text lives outside `.value` (`terminal` clears its input on every key): n.a. there.
- **No clipboard**: `press` rejects paste chords (`ControlOrMeta+v`, `Shift+Insert`): any application can change the clipboard between copy and paste. `copy-paste` tasks are typed from the AXTree and scored by exact match, trailing space included: S0 checks the AXTree keeps it and tags both, reported apart.
- **Failure classes**: a mismatch fails the step, returns as the previous error and is an **execution** failure (per step, every arm). Grounding is the A-oracle gap (aggregate); a not-editable focus after a click is reported with it.
- **Check tests**: S0 plants one wrong and one correct case per check (correct: `enter-date`, `enter-time` among them); only the wrong one may fire. The first A-oracle dev-subset run (S3) logs checks without failing steps: a mismatch on a rewarded episode counts as a false positive (an upper bound), reported as a rate.
- **Termination**: env reward or `terminated`, planner `done`, or BrowserGym's **10-step cap** (kept for comparability). **Budgets**: ≤ 4 grounder queries per target (a drag has two targets); 30 s per step. S0 checks whether MiniWoB's in-page timers are enforced; if so, slow-inference timeouts are reported per task.
- **Baselines**: **A0** = single Qwen3.5-4B with vision, planning and emitting coordinates in one call. **A-oracle** = the planner sees the AXTree **with** ids and names one; the executor acts on its box center (grounding upper bound). **A-API** (optional) = commercial planner on the 30-task dev subset, $20 cap.
- **Per-step log**: planner ms, grounder ms, depth, confidence, peak VRAM, action, reward, check (ok / not-editable / paste-rejected / mismatch / n.a.).
- **Visual vs non-visual**: A0 sees pixels and the portable planner does not, so tasks whose goal depends on what is drawn (color, shape, canvas content) are tagged in S0, before any agent run, and reported apart.

## 8. Hardware Profiles

> **Rule: the weakest machine on each axis sets the defaults**: memory by the 6 GB card, speed by the slower machine on vision prefill (measured in S0; expected to be the Mac).

| | `portable` (default) | `accelerated` (opt-in) |
|---|---|---|
| Grounder | 2B Q4_K_M (0.8B scout optional) | 2B Q8_0 or 4B Q4_K_M |
| LoRA target | 0.8B | 2B |
| Pixel cap | 0.9 MP (~900 tokens) | 1.6 MP (~1,600) |
| Cascade | conditional, D = 2 | conditional, D = 2–3 |
| Planner | 4B Q4_K_M **text-only** (no mmproj) | 4B Q4_K_M with vision |

**Mac memory:** the GPU gets only part of unified memory by default; S0 reads the limit from llama.cpp's reported working-set size.

**On 6 GB, the `+ LoRA` agent rung swaps the base 2B grounder for the fine-tuned 0.8B** (claim 1's 6 GB target; frees ~1.2 GB); on 8 GB or via the MLX spike, a fine-tuned 2B. The scout runs in agent mode only if memory leaves room for a third model.

**Memory budget, 6 GB card, agent mode:**

| Item | GB |
|---|---|
| Planner Qwen3.5-4B Q4_K_M, text-only | 2.74 |
| Grounder Qwen3.5-2B Q4_K_M + mmproj F16 | 1.28 + 0.67 |
| **Weights subtotal** | **4.69** |
| Left on a 6 GB card before headroom | 1.31 |
| Runtime overhead: two `llama-server` processes' CUDA contexts and cuBLAS workspaces, prefill, 0.9 MP vision buffers (~3,600 pre-merge patches), key-value (KV) cache + Gated DeltaNet state, Windows display reserve | to be measured |
| Headroom target | ≥ 0.5 |

With vision, both models need 5.36 GB of weights alone, hence the text-only planner. Even so, ~0.8 GB is unlikely to cover two processes' overheads, so **fitting 6 GB is a week-2 go/no-go item** (§10). The configurations below are planned, not contingencies; S0 picks the first that passes.

| Config | Planner | Grounder | Cost |
|---|---|---|---|
| **M1** | 4B Q4_K_M on GPU | 2B Q4_K_M + mmproj on GPU | none, if it fits |
| **M2** | 4B Q4_K_M partly or fully on CPU (`-ngl` below all layers; 16 GB RAM holds the 2.74 GB file) | on GPU | planner latency, measured |
| **M3** | 4B IQ4_XS (2.48 GB) on GPU | 2B at 0.6 MP | accuracy at the lower cap |
| **M4** | as M1 or M2 | vision encoder on CPU (`--no-mmproj-offload`) | grounder latency |
| **M5** | on the Mac over the LAN | on the 6 GB card | not single-machine; reported as such |

## 9. Evaluation Plan

**The 0.9 MP number from the cap curve (§10) is the local baseline for all claims.**

**Ablation ladder.** Grounding: base → pixel-cap sweep → always-zoom D = 1, 2 → conditional zoom → + LoRA (web) → + LoRA (web + native) → + scout. Agent: A0 → 4B planner + base 2B → + LoRA (grounder per §8) → + conditional zoom (MiniWoB-4K only) → oracle grounding.

**Pre-registered tests** (α = 0.05, power 0.8; d = discordant-pair rate; CI = confidence interval; SE / SD = standard error / deviation):

| Test | n | Decision rule | Power and MDE |
|---|---|---|---|
| SSPro lift (claim 1): paired McNemar | 1,581 | Lift ≥ MDE, p < 0.05 | **MDE = 3 points** (n ≈ 1,300 at d ≈ 15%; 2 points needs ~2,900, more than exist). d estimated in S0 from base 0.8B vs base 2B (no adapter before S2); LoRA-vs-base d reported once measured |
| "Within 3 points" (claim 1): non-inferiority | 1,581 | One-sided 95% lower bound of (fine-tuned − larger base) > −3 points | n ≈ 6,870 · d: powered only if d ≤ ~23% (≈ 1,030 at d = 15%, ≈ 1,720 at d = 25%). If underpowered, "within X points (95% CI …)", never parity; d reported |
| SSv2 gate: non-inferiority | 1,272 | One-sided 95% lower bound of (adapter − base) > −2.0 points | SE ≈ √(d / 1,272): an unchanged adapter passes with ≥ 80% probability only if d ≤ ~8%; d reported. Mobile / desktop / web splits: CIs and flags, not gates (no mobile screens in minted data) |
| Confidence (claim 2) | 1,581 | "Useful" = AUROC ≥ 0.70, lower bound > 0.65 | 95% CI half-width ≈ ±0.03 at ~45% errors |
| Pareto subset (claim 3) | 400, fixed, stratified | Configs within one MDE of the frontier are re-run on all 1,581 | MDE ≈ 5.4 points at d = 15%, ≈ 7.7 at d = 30% (d runs high: configs differ in size, quantization, cap) |
| MiniWoB++ and claim 4 | 125 tasks × 5 seeds = 625; paired per task (n = 125) | CIs and successes/hour by task-cluster bootstrap; 30-task dev subset never reported. Claim 4 holds if the bootstrap 95% CI of (pipeline − A0) successes/hour excludes 0 **and** pipeline success is not below A0's by more than the paired task-level MDE; else "not supported" | MDE = 2.80 · SD_task / √125 (task-level SD from the dev subset; 0.30 gives ≈ 7.5 points)¹ |

¹ Treating the 625 episodes as independent gives an optimistic MDE (~8 points at ~50% success); intra-task correlation 0.5 (design effect 3) makes it ~14.

**Also reported:** accuracy per SSPro group, text vs icon and on the confusable slice (§7.3); SSPro and OSWorld-G deltas vs the pre-LoRA checkpoint; expected calibration error (ECE) and reliability diagrams per signal × quantization × before/after LoRA; per-domain accuracy (web vs native vs professional; seen app / unseen screens vs unseen app; web-only vs web + native adapters); local base 0.8B / 2B / 4B.

**Honest reporting.** Our numbers state quantization, mode, pixel cap, image-token count, runtime and the bf16 / Q8_0 / Q4_K_M delta. τ, hyper-parameters and prompts are never chosen on a reported test set. MiniWoB results beside Explorer or GenericAgent carry a harness caveat. Negative results are reported. The decontamination report lists excluded apps, pHash matches against all three grounding benchmarks, and Jedi-slice OSWorld/WAA removals (count).

## 10. Milestones (19 hours / 15 sessions)

| Sprint | Sessions | Deliverable |
|---|---|---|
| **S0, part 1** | 1–2 | Repo, `uv`, pinned llama.cpp and GGUF revisions on both machines; reproduction prompt and per-model coordinate parsers; 200-step training probe; memory configs M1–M5 measured |
| **Gate** | end of 2 | **Week-2 go/no-go** (below) |
| **S0, part 2** | 3–4 | **Reproduce Qwen3.5-2B SSPro 54.5 (non-thinking) within ±3 points** (defined below); cap curve native → 1.6 → 0.9 → 0.6 MP, both machines; Q8_0 / Q4_K_M deltas; 0.8B / 4B baselines; base 0.8B vs 2B discordance; thinking runs where they terminate (card warns of 2B thinking loops); SSv2 baselines; MiniWoB task tags (visual, copy-paste, text outside `.value`) and silent-failure planted cases (§7.7); MiniWoB-4K viewport check |
| **S1** | 5–8 | Web minter + UIA harvester with gates; page snapshot; instruction writer; cascade and three confidence signals with AUROC. **Week-6 demo:** 2B hits a 12-pixel icon on a 4K screenshot, zoom path drawn |
| **S2** | 9–10 | LoRA round 1 on 0.8B (30k) with the SSv2 gate; web-only vs web + native ablation (10k); per-domain table; stretch: AX harvester, Jedi-slice control |
| **S3** | 11–12 | MiniWoB agent with A0 and oracle baselines; check false-positive rate (§7.7); MiniWoB-4K with and without zoom; overnight Pareto sweeps on both machines |
| **S4** | 13–14 | grounder-svc hardening, step monitoring, decontamination report, write-up (§14); stretch: MLX export |
| — | 15 | Interview sprint |

**Week-2 go/no-go (end of session 2)**, on the 6 GB Windows card unless stated:

| Check | Measurement | Pass |
|---|---|---|
| **(a) Latency and memory**, both machines | p50/p95 (median / 95th percentile) of one 2B Q4_K_M query at 0.9 MP and of one planner + grounder round trip on a MiniWoB screenshot; `nvidia-smi` peak per process | Grounder p50 ≤ 1.5 s (an SSPro config at ~1.5 queries per item then takes about an hour); round-trip p50 ≤ 6 s on the slower machine; one of M1–M5 (§8) fits with ≥ 0.5 GB headroom |
| **(b) Training** | 200 steps of Unsloth bf16 LoRA on Qwen3.5-0.8B with vision at 0.9 MP, native Windows: s/step, peak VRAM | 30k rendered samples fit in ≤ 10 h |
| **(c) Parsing** | Reproduction prompt on 50 SSPro items per sweep model | Coordinates parseable |

- **(a) fallbacks, in order:** cap 0.6 MP, 0.8B scout at depth 0, IQ4_XS weights, Pareto sweeps on the 400-item subset only.
- **(b) no-go actions, in order:** train under WSL2 (Windows Subsystem for Linux); train on the Mac with mlx-vlm; cut the sample cap to what fits. If all three fail, drop claim 1 and keep the training-free claims 2–4. Unsloth notes custom Mamba Triton kernels can slow Qwen3.5 training; its Windows guide omits this.

**Reproduction, defined.** State model, runtime, quantization, prompt, decoding and maximum visual tokens; the per-request image-token log (§4.2) proves the image was not capped.
- **Reference**: transformers bf16 on the Mac, native resolution, no client-side cap (the processor allows up to ~16.8 MP; fit of a ~8,100-token 4K screenshot is measured). Fallback: free Colab (§4.2), else the highest cap that fits, reported as such. **Then** `llama-server` Q8_0 with `--image-max-tokens` pinned high.
- **Decoding**: the card's non-thinking sampling (T 0.7, top_p 0.8, top_k 20, presence_penalty 1.5) and greedy; both reported.
- **On a miss**: S1 opens by explaining the gap (prompt, coordinate frame, cap, quantization); later lifts use the local baseline (§9).

**Stretch, in cut order** (everything else above is core, including zoom-agreement AUROC from always-zoom logs): 1. A-API planner; 2. frozen-vision ablation; 3. MLX export; 4. Jedi-slice control; 5. AX harvester; 6. 2B adapter (8 GB or MLX spike).

**Overnight compute calendar.** One item per ≤ 10 h night, 7 nights each in S2 and S3; a full MiniWoB run takes up to ~10 h (625 episodes × ≤ 10 steps × ≤ 6 s).

| Nights | Windows (6 GB) | Mac (16 GB) |
|---|---|---|
| S2 n1–n4 | n1 Optuna (≤ 6 trials × 3k, 0.8B); n2 0.8B web + native, 30k; n3–n4 adapter evals, 10k web-only vs web + native pair | n1–n2 claim 3 on base 2B (always-zoom, voting, gated zoom per signal, UI-Zoomer gate); n3–n4 AUROC, signal × quantization on base, then on the adapter's GGUF Q8_0 / Q4_K_M |
| S2 n5–n6 | Stretch: Jedi slice 10k; frozen vision 10k | Stretch: MLX 2B LoRA spike |
| S3 n1–n6 | n1–n4 MiniWoB A0, pipeline base 2B, + LoRA, oracle; n5–n6 Pareto (CUDA configs) | n1–n2 MiniWoB-4K without / with zoom; n3–n6 Pareto (n3: 400-item subset; n5–n6 add 9B, UI-TARS-1.5-7B) |
| S2 n7, S3 n7 | Buffer / re-runs; S3 n7 re-runs near-frontier configs | same |

## 11. Risks & Mitigations

| Risk | Mitigation |
|---|---|
| Card number does not reproduce (prompt unpublished, quantization, runtime) | Claims vs our own baseline; S0 sweep; gap reported |
| Qwen3.5 vision unstable in llama.cpp / mlx-vlm | Pins, golden-output test, transformers fallback (§4.2) |
| Synthetic-to-real gap | Native harvest from session 5 (~week 4); per-domain table |
| Incomplete or stale UIA/AX trees | §7.2 gates and per-app audit; drop failing apps |
| 4K on 6 GB; 6 GB cannot train 2B | Pixel caps, cascade, text-only planner; 0.8B as the 6 GB training target; MLX spike for 2B |
| Planner + grounder do not fit 6 GB | Week-2 check; configs M2–M5 (§8); claim 4 stated under the config that passes |
| Training slower than the overnight budget | Week-2 probe sizes the cap; 0.8B ablations at 10k; WSL2 or MLX fallbacks |
| Gate loses calibration after LoRA or at Q4 | That is claim 2 either way; recalibrate τ per checkpoint on the minted split |
| Prior art | Cited up front (§1); claims cover small, quantized, local models and agent throughput |
| Contamination | Ban list (§5.2); Jedi slice filtered of OSWorld/WAA rollouts; pHash dedupe; report |
| Silent GUI failures: an action returns no error but commits the wrong value, misread as a planning or grounding miss | §7.7 checks in every arm; mismatches are their own failure class. MiniWoB's typed-field and `choose-list` rewards read DOM `.value` or the selected label, as the read-back does; `terminal` (its own element) and `text-editor` (the Quill editor's model) do not, nor would any app with its own document model: read-back n.a. (reading such a model: untested) |
| MiniWoB too slow | 30-task dev subset; overnight full runs for final configs only |
| Compute calendar overruns | Cut stretch items in order (§10) |

## 12. Résumé Bullets (experience voice; numbers filled in as sprints land)

Each bullet is rewritten to the measured result; a failed claim becomes a finding, e.g. "showed confidence-gated zoom does not beat equal-compute voting on a quantized 2B (Δ X, n = 1,581)". The 2B enters bullet 2 only if its adapter exists.

- Built a **self-labelling GUI grounding data engine** minting (screenshot, instruction, box) triples from browser DOM and Windows UIA / macOS accessibility trees, with occlusion, validity and dedupe gates rejecting X% of candidates and decontamination against three public benchmarks.
- Fine-tuned **Qwen3.5-0.8B with bf16 LoRA on X k self-minted samples**, lifting ScreenSpot-Pro from X to X (paired McNemar p < X) with no ScreenSpot-v2 regression — within X points of the larger base 2B (non-inferiority, 95% CI …) on a 6 GB laptop GPU.
- Designed a **confidence-gated coarse-to-fine zoom cascade**; showed grounder confidence predicts errors at AUROC X across quantization levels, and gated zoom adds X points at X mean queries vs X for equal-compute sample voting.
- Ran a **≥ 20-config accuracy × latency × VRAM Pareto study** on CUDA and Apple Silicon, selecting a default serving at p50 X s within 6 GB VRAM.
- Built a **planner–grounder computer-use agent** on BrowserGym MiniWoB++ raising successes per hour X% over a single 4B model; an oracle-grounding ablation attributed X% of remaining failures to grounding.

### Placement Note

List this under EXPERIENCE only with a real engagement: the mentoring organization as employer (truthful title, engagement record), a real collaboration with a startup, lab or open-source organization hosting the work, or an open-source project with public commit history. Never invent an employer or dates: North American employers verify employment, and fabrication gets offers rescinded. A public repo with real commits and a reproducible results table survives scrutiny better than an unverifiable internship line.

## 13. Why This Reads as Real Company Work

- **Data engines are the job**: OS-Atlas, UGround, Jedi and GroundCUA are data pipelines first; this one has gates and a data card.
- **The budget is a spec**: on-device and enterprise-desktop teams ship against "fits 6 GB at p50 ≤ 1.5 s".
- **Costed test-time compute** against an equal-compute control; **reproduction and regression gates before claims**.
- **Failure attribution** answers the first product question: does the agent not know what to do, can it not click it, or did its action silently do something else?

## 14. Interview Leverage

- **System design**: laptop computer-use agent, planner/grounder split, VRAM / latency / step budgets, when to spend test-time compute.
- **ML depth**: VLM (vision-language model) LoRA, visual-token budgets, why no QLoRA, zoom-consistent augmentation, forgetting gates.
- **Evaluation**: McNemar MDE and n, superiority vs non-inferiority, clustered episodes, AUROC and calibration, equal-compute controls, decontamination.
- **Data and efficiency**: accessibility-tree harvesting and QC (the Weibo story); GGUF, mmproj and KV-cache memory, CUDA vs Metal vs MLX (the CAS story); crop-to-global transforms, point-in-box scoring, a bounded cascade.
- **Behavioral**: why local matters, prior art found mid-project, publishing negative results.
- **Drills** (paraphrased themes from public agent-engineering interview reports): *"Each step is 90% reliable; how reliable is a ten-step task, and what do you change?"* → §7.6, §7.7; *"How do you know a keystroke or a dropdown choice did what the agent meant?"* → §7.7; *"The agent failed: did it pick the wrong action or execute the right one badly?"* → §7.7, §1 claim 4.
- **Write-up:** the S4 write-up (§10) is one post under the mentee's own byline, working title *"Four pixels left: what a 2B grounder misses on a 6 GB GPU, and what zoom and a read-back catch"*: hook, context, the work (failure before fix: base-2B misses on 4K icons, then zoom and LoRA, §9 ladder), what I'd do differently (the pre-declared null or fallback: H1 or claim 3 not supported under the §9 rules, worded as in §12; configs M2–M5, §8), takeaways. Every number traces to the results table; vendor numbers are labelled.

## 15. References

- Qwen3.5 cards: https://huggingface.co/Qwen/Qwen3.5-2B · https://huggingface.co/Qwen/Qwen3.5-4B · 2B preprocessor config: https://huggingface.co/Qwen/Qwen3.5-2B/blob/main/preprocessor_config.json
- Unsloth Qwen3.5 fine-tune: https://unsloth.ai/docs/models/qwen3.5/fine-tune · run guide: https://unsloth.ai/docs/models/qwen3.5 · Windows: https://unsloth.ai/docs/get-started/install/windows-installation
- GGUF: https://huggingface.co/unsloth/Qwen3.5-0.8B-GGUF · https://huggingface.co/unsloth/Qwen3.5-2B-GGUF · https://huggingface.co/unsloth/Qwen3.5-4B-GGUF · https://huggingface.co/unsloth/Qwen3.5-9B-GGUF
- MLX: https://huggingface.co/mlx-community/Qwen3.5-4B-MLX-4bit · llama.cpp multimodal: https://github.com/ggml-org/llama.cpp/blob/master/docs/multimodal.md · Ollama qwen3.5: https://ollama.com/library/qwen3.5
- UI-TARS-1.5-7B: https://huggingface.co/ByteDance-Seed/UI-TARS-1.5-7B · coordinates: https://github.com/bytedance/UI-TARS/blob/main/README_coordinates.md
- Holo-3.1-4B: https://huggingface.co/Hcompany/Holo-3.1-4B · community GGUF: https://huggingface.co/g023/Holo-3.1-4B-GGUF
- VISTA: https://arxiv.org/abs/2606.14579 · https://huggingface.co/inclusionAI/VISTA-9B · ZonUI-3B: https://arxiv.org/abs/2506.23491
- ScreenSpot-Pro: https://arxiv.org/abs/2504.07981 · https://github.com/likaixin2000/ScreenSpot-Pro-GUI-Grounding · https://huggingface.co/datasets/likaixin/ScreenSpot-Pro
- ScreenSpot-v2: https://huggingface.co/datasets/OS-Copilot/ScreenSpot-v2 · OS-Atlas: https://arxiv.org/abs/2410.23218 · UGround: https://arxiv.org/abs/2410.05243
- OSWorld-G / Jedi: https://arxiv.org/abs/2505.13227 · https://osworld-grounding.github.io · OSWorld: https://arxiv.org/abs/2404.07972 · GroundCUA: https://arxiv.org/abs/2511.07332
- RegionFocus: https://arxiv.org/abs/2505.00684 · UI-Zoomer: https://arxiv.org/abs/2604.14113 · AutoFocus: https://arxiv.org/abs/2605.02630 · Zoom Consistency: https://arxiv.org/abs/2604.15376
- BrowserGym: https://arxiv.org/abs/2412.05467 · MiniWoB setup: https://github.com/ServiceNow/BrowserGym/blob/main/browsergym/miniwob/README.md · Explorer: https://arxiv.org/abs/2502.11357
- Silent-failure checks (§7.7): BrowserGym actions: https://github.com/ServiceNow/BrowserGym/blob/main/browsergym/core/src/browsergym/core/action/functions.py · MiniWoB task pages: https://github.com/Farama-Foundation/miniwob-plusplus/tree/main/miniwob/html/miniwob · Blink type-ahead: https://github.com/chromium/chromium/blob/main/third_party/blink/renderer/core/html/forms/type_ahead.cc · Playwright focus emulation: https://github.com/microsoft/playwright/blob/main/tests/library/emulation-focus.spec.ts · Windows clipboard: https://learn.microsoft.com/en-us/windows/win32/dataxchg/about-the-clipboard
