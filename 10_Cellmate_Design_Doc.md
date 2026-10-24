# Cellmate — Design Document

**A Laptop Spreadsheet Agent That Writes Live Formulas, Recalculates in LibreOffice After Every Edit, and Fuzzes Its Own Sheet Before Handing It In**

| | |
|---|---|
| Status | Proposal |
| Target duration | ~10–12 weeks (19 mentoring hours / 15 sessions + independent work) |
| Platforms | Windows 11 laptop (16 GB RAM, NVIDIA RTX 3060-class, **6 GB VRAM default**, 8 GB if confirmed) and Apple Silicon Mac (16 GB unified, no CUDA); one repository |
| Deployment | 100% local; no rented servers or paid GPU cloud. Headless LibreOffice on both CPUs; overnight QLoRA on the 6 GB card. Optional: free Colab/Kaggle GPUs (accelerator), commercial APIs (comparison baselines, ≤ $20 total) |
| Core stack | llama.cpp `llama-server` (one pinned build) · Qwen3-4B-Instruct-2507 GGUF (Apache-2.0) · Spreadsheet-RL-4B / -8B and Qwen3-8B GGUF (Apache-2.0) · LibreOffice 26.2.x headless + Python-UNO (Universal Network Objects API) workers · openpyxl · SpreadsheetBench Verified-400, 912 v0.1 and SpreadsheetBench 2 (CC BY-SA 4.0) · Unsloth + PEFT QLoRA (WSL2) · SQLite + Parquet · statsmodels / SciPy · `uv` |

---

## 1. The Product Thesis

Excel is the most-used programming environment, and a spreadsheet agent's answer is the workbook. Small models fail two ways. They cannot see what they wrote: openpyxl writes `=VLOOKUP(A2,Sheet2!A:C,4,FALSE)`, it evaluates to `#REF!`, and Python never says so. Or they paste literals computed elsewhere, which pass today's check and go stale when inputs change: a classic audit finding (European Spreadsheet Risks Interest Group, EuSpRIG; §13).

The gap (§4.3, §5): vendor scaffolds score 86–99% on SpreadsheetBench Verified-400, we found no score for any model ≤ 8B on it (on the original 912 set the 4B scores 9.3%, §5.2), and no one ablates the engine alone: GRID credits its engine without a matched ablation, and Spreadsheet-RL adds recalculation together with other tools (12.0 → 15.6 → 19.3; §4.3, §5.2).

**Cellmate's thesis: for a small spreadsheet agent, the harness (what it observes, how the sheet is encoded, whether it tests its own edits) buys more than laptop post-training or doubled parameters.** Test: on Verified-400, harness gain (A0 → A2) vs RFT gain on the same 4B (C4) vs 4B → 8B gain under one harness, as differences of paired differences with bootstrap confidence intervals (CIs; §9).

1. **C1, engine in the loop.** With LibreOffice recalculating after every write and returning values and `#REF!`/`#VALUE!`/`#NAME?`, a 4B agent beats its own Python/openpyxl-only variant on Verified-400, paired by task, at matched turn and token caps; nothing else differs (§7.1, §7.5). Planning minimum detectable effect (MDE): 5.3 percentage points (pp), §9.
2. **C2, metamorphic self-tests.** Before submission the harness replays the edit log onto perturbed scratch copies and checks invariants (§7.6). On the 912 set's hidden cases 2–3 they cut constant-shadowing failures (a literal where a live formula was needed) ≥ 50% relative against a placebo review turn; built only if S1 finds enough such failures (§7.7).
3. **C3, structure map.** Region detection, SheetCompressor-style anchors and format aggregation cut median prompt tokens per task ≥ 3× against a full-sheet encoding; accuracy cost descriptive (§7.3).
4. **C4, laptop RFT.** Overnight QLoRA rejection-sampling fine-tuning (RFT) of the 4B on the 6 GB card, on verified trajectories from Spreadsheet-RL's deduplicated ExcelForum split, against its base. Spreadsheet-RL-4B/-8B (Group Relative Policy Optimization, GRPO) are descriptive references (§7.8).

**Why it is fun.** A 4B model writes `=SUMIFS`, sees `#VALUE!`, fixes its criteria range, then fuzzes its own sheet. In the demo its factor-screen ranks move when one net income changes; a Python-only run's ranks freeze (§7.9).

## 2. Goals & Non-Goals

### Goals
Build and score C1–C4 (§5, §7), pre-registered (§9); reproduce a published number in S0 (§10). **Released**: harness, T IDs, divergent and dedup lists, run logs, demo.

### Non-Goals

| Out of scope | Owner |
|---|---|
| Generic per-step distillation trainer (segmenter, loss masking, view parity); C4 calls it, adding only a spreadsheet trajectory converter | 02 Shellwright |
| Test-time search, process reward models (self-tests verify one candidate, never rank several) | 01 Compass |
| Long-horizon context management (the map encodes one workbook) | 03 Spelunker |
| Driving Excel or LibreOffice GUIs | 04 Crosshair |
| Record/replay cassettes (optional run export), nondeterminism study, regression CI | 05 BlackBox |
| Sandboxing model-written code, rollback, injection defence | 06 Ctrl-Z |
| Evolving harness code automatically | 07 HarnessForge |
| Charts, pivot tables, formatting, VBA, SpreadsheetBench 2 Visualization; leaderboard submissions | — |

02 tests interface vs knowledge on terminals; Cellmate isolates one interface variable: engine readback.

## 3. Alignment With Prior Experience

| Prior experience | Cellmate module |
|---|---|
| **Changsheng** (fund management): two-stage sector→stock strategy, backtested (look-ahead bias); AutoML hyper-parameter agent | Factor-screen demo; transfer replay as out-of-sample backtest discipline. Confirm with mentee whether his team's models live in Excel before claiming it |
| **DiDi**: Java backend modules; **A/B testing** (page layouts) | Pre-registered MDE, A/A pilot, Holm across claims; per-task McNemar pairing is new |
| **Sina Weibo**: SQL ETL, standardized metrics, alerting | Format aggregation in the map; error readback as the alert |
| **CAS**: binary-encoding retrieval; Precision@K / latency / storage over 20+ configs | C3 as a compression–accuracy frontier |
| **College Application Agent**: LangGraph Writer–Critic | Executable critic: recalculation and self-tests, not an LLM's opinion |

**Interview one-liner:** *"A number pasted over a formula is a classic finance audit finding. My laptop agent writes live formulas, recalculates in LibreOffice after every edit and fuzzes its own sheet; I tested each piece against a Python-only agent, DiDi A/B style: effect sizes fixed in advance, an A/A check, then the test."*

## 4. Stack Choices — and Why

### 4.1 Models

| Role | Model | Quant / file (Hugging Face, 2026-10-23) |
|---|---|---|
| PC agent; RFT base | Qwen3-4B-Instruct-2507 (non-thinking) | Q4_K_M 2.50 GB (`unsloth`) |
| PC RL comparison | Spreadsheet-RL-4B (Qwen3-4B-Thinking-2507 + GRPO) | Q4_K_M 2.72 GB (`mradermacher`) |
| Mac agent | Qwen3-8B, `enable_thinking: false` | Q4_K_M 5.03 GB (`unsloth`) |
| Mac RL comparison | Spreadsheet-RL-8B (Qwen3-8B + GRPO) | Q4_K_M 5.03 GB (`mradermacher`) |
| Mac dedup embedder | Qwen3-Embedding-0.6B | Q8_0 0.64 GB |

`mradermacher` GGUFs are third-party: hashes pinned, re-converted in S0 if parse tests fail.

**KV math** (`config.json`, same for all four agent models): 36 layers × 8 KV heads × 128 dims × 2 = 73,728 values per token: 144 KiB at f16, 76.5 KiB at q8_0 (34 bytes per 32 values).

**Why Qwen3, not Qwen3.5.** Qwen3 has the only controlled small-model SpreadsheetBench table (§5.2), so S0 can reproduce a row; both Spreadsheet-RL checkpoints are Qwen3; 6 GB RFT means QLoRA, which Unsloth advises against on Qwen3.5 (bf16 LoRA of a 4B does not fit). **GGUF, not MLX, on the Mac**: `mlx-community/Qwen3-8B-4bit` (4.61 GB) works, but one llama.cpp build on CUDA and Metal means one quantization family and one tool-call parser; MLX is the speed-gate fallback (§8).

**Thinking.** Only the Spreadsheet-RL checkpoints think, as trained, under a pre-registered `--reasoning-budget 4096` per turn (one 50-task ablation at 8192), with the Qwen3 thinking sampling their generation config ships (temperature 0.6, top-p 0.95, top-k 20).

### 4.2 Runtime

One pinned `llama-server` build (prebuilt `win-cuda`, `macos-arm64`): `--jinja`, `-fa on`, `--cache-type-k q8_0 --cache-type-v q8_0` (a quantized V cache needs flash attention); slot sizes per arm in §8. Instruct-2507 uses card sampling (temperature 0.7, top-p 0.8, top-k 20, min-p 0), seed logged; only S0 uses Spreadsheet-RL's decoding (§10).

### 4.3 Spreadsheet engine and libraries

| Concern | Choice | Why |
|---|---|---|
| Engine | LibreOffice 26.2, one exact point release (newest at S0, 26.2.5 or later) on both OSes, installer hashes archived | Free, headless, both OSes. 26.2 hits end of life 2026-11-30, mid-project; frozen for reproducibility (26.8 shipped 26 August 2026). XLOOKUP, FILTER, SORT, UNIQUE, SEQUENCE, LET and kin since 24.8 |
| Evaluator | SpreadsheetBench `open_spreadsheet.py` + `evaluation.py` at commit `49b73a9`, `--backend libreoffice` forced | Windows defaults to win32com when Excel is installed, splitting the machines. Parity PR #32 (merged 2026-03-12): recalculation changed 0 of 400 verified_400 outcomes, 16 of 912 (10 regressions, 3 real LibreOffice issues) |
| In-loop recalculation | Own Python-UNO worker on LibreOffice's bundled Python, one persistent `soffice` each | `unoserver` has this design (persistent listener per port), but its README calls Windows and Mac untested |
| Excel | win32com cross-check only if installed | Spreadsheet-RL keeps Excel as truth, citing LibreOffice function gaps; Cellmate measures that risk (§7.2) |

Also `formulas`, a stretch pure-Python engine arm (483 of 536 Excel functions, 90.1%, per Spreadsheet-RL): does that coverage get most of the gain?

**Prior art**: SpreadsheetLLM's SheetCompressor (anchors, inverse-index translation, format-aware aggregation; 25× average compression, 78.9% table-detection F1); Spreadsheet-RL (Excel gym with `recalculate_and_read`, GRPO; its ablation adds recalculation together with other tools, §5.2); SpreadsheetAgent (ACL 2026: structural sketch and targeted-inspection verification, each ablated); SheetBrain (AAAI 2026: LLM checklist validator triggering re-execution); SheetCopilot, SheetAgent; vendor scaffolds (GRID, July 2026, credits its engine without a matched ablation). All verify by LLM or inspection; none ablates engine readback alone or replays edits onto hidden cases. Cellmate does both and adds metamorphic self-tests and laptop RFT.

## 5. Benchmarks & Datasets

### 5.1 SpreadsheetBench Verified-400 (headline)

Released 2 December 2025 by Shortcut and the authors. Review of the 912 tasks removed 216, kept 696 and modified 382 (375 queries, 93 goldens, 45 inputs); 296 too-easy tasks were then dropped. Each task ships **one** input/answer pair (800 files, per PR #32). `evaluation.py` compares `answer_position` values with the golden, floats rounded to 2 decimals; any mismatch fails. CC BY-SA 4.0, 14.96 MB, CPU recalculation only. All published scores are vendor-reported (aggregator snapshot 2026-10-18):

| System | Verified-400 | Source |
|---|---|---|
| JT-DA-Agent (aggregator top row) | 99.25% | benchmarklist, "imported" |
| GRID agent, claude-sonnet-5 | 91.25% best of 7 (median 90.75%) | GRID blog, 2026-07-27 |
| Shortcut | 86% (85.3 / 84.8 / 85.1 / 88.4; pass@4 98.2%) | Shortcut blog |
| Open-weight frontier (GLM-5.1 Thinking … Kimi K2.6 Thinking) | 85.2 … 84.5% | benchmarklist, self-reported, one shared source link; its "open" Qwen3.6 Plus (80.2%) is API-only, excluded |
| Any model ≤ 8B | none found (search 2026-10-23) | — |
| Run-to-run noise | 12 of 100 tasks flip across five identical runs, even near 90% (hence the A/A pilot, §9) | GRID blog |

### 5.2 SpreadsheetBench 912 v0.1 (S0 reproduction; transfer set)

912 instructions, 2,729 test cases (~3 each) from Excel-forum questions (NeurIPS 2024 Datasets and Benchmarks track). Spreadsheet-RL runs one rollout per test case (released `test` split 2,722 rows; paper: 2,726 rollouts per checkpoint). Pass@1 per test case, Spreadsheet Gym on Excel 365 v2512 (Spreadsheet-RL Table 1; -8B row from its card):

| Model | Pass@1 |
|---|---|
| **Qwen3-4B-Instruct-2507** | **9.3** (S0 target) |
| Qwen3-4B / Qwen3-14B / Qwen3-32B | 11.0 / 15.0 / 17.6 |
| Qwen3-8B (with full harness and tools) | 15.9 (16.7) |
| Qwen3-4B-Thinking-2507: base → + native harness → + full tools | 12.0 → 15.6 → 19.3 |
| Spreadsheet-RL-4B (GRPO, 60 steps) / Spreadsheet-RL-8B (step 35) | 23.4 / 22.3 |

Its cited closed rows run from GPT-4o 16.8 to Copilot Agent Mode 57.7. GPT-4o's 16.8 (OpenAI, OSX + LibreOffice) vs 18.4 (Microsoft, Windows + Excel) is uncontrolled (reporter, agent, OS); engine evidence is PR #32 (§4.3).

**Transfer set T**: Verified tasks whose case-1 input and golden `answer_position` match v0.1 case 1 cell for cell, instruction not materially changed. Their v0.1 cases 2–3, never reviewed by Verified, are hidden replay targets (§7.7); S0 measures |T| and ID mapping. **Audit** (before the C2 go/no-go): case 1's golden formulas, replayed onto cases 2–3, must reproduce their goldens; 50 stratified tasks are hand-checked against the Verified instruction; failures leave T. The audit error rate sits beside every constant-shadowing failure (CSF) number; above 10%, transfer metrics are descriptive.

### 5.3 SpreadsheetBench 2: Financial Modeling + Debugging (descriptive)

321 tasks: Template 97, Financial Modeling (FM) 100, Debugging 100, Visualization 24 (arXiv 2606.29955; NeurIPS 2026). Visualization (Excel/WPS COM chart export plus a VLM judge) is skipped. Inputs and goldens are recalculated before scoring, per the README.

| Model | Overall | Financial Modeling | Debugging |
|---|---|---|---|
| Claude Opus 4.6 (best) | 34.89 | 34.00 | 12.00 |
| GLM-5 (best open) | 17.14 | 22.00 | 7.00 |
| Qwen3.5-397B-A17B (open) | 11.22 | 10.00 | 3.00 |
| MiniMax M2.5 (lowest open) | 7.17 | 8.00 | 4.00 |

A 4B is expected near the floor: 100 tasks per split, Wilson intervals, no test.

### 5.4 Training and development pools
- **RFT pool**: Spreadsheet-RL's `train` split, 5,925 ExcelForum tasks from threads after 2024-01-01 (CC BY-SA 4.0). Its card states no decontamination and SpreadsheetBench shares the forums, so we deduplicate against the 912 set, Verified and SpreadsheetBench 2 (sha256 of canonical cell content, then Qwen3-Embedding-0.6B cosine on instructions, threshold fixed on 50 hand-reviewed pairs before training) and publish the list.
- **Dev-200**: 200 non-Verified SpreadsheetBench tasks (review failures and too-easy ones; noisy, accepted), stratified by `instruction_type`. All iteration happens here; Verified-400 sees only pilot and pre-registered runs.

## 6. Architecture

```
  ALL LOCAL: Windows + CUDA (4B arms, QLoRA) | Apple Silicon Mac (8B arms) | one repo

  Tasks: Verified-400 | T (912 v0.1 cases 2-3, hidden) | SpreadsheetBench 2 | dev-200 | RFT pool
                              |
  RUNNER  working copy -> agent loop (<= 12 turns, 8k prompt) <-> llama-server (pinned)
          structure map: regions, anchors,      PC: Qwen3-4B-Instruct-2507, Spreadsheet-RL-4B
          formula groups, format aggregates     Mac: Qwen3-8B, Spreadsheet-RL-8B
                              |
  TOOLS   find_cells | read_range | write_formula | write_values | copy_fill
          insert/delete_rows | run_python | submit
          ONE WRITER (all arms): openpyxl + prefixer/array writer + reference shifter
          untouched cells: cached values of input.xlsx (all arms)
            +-- A0 Python-only: no recalculation, own writes read back as text
            +-- A1 engine: UNO workers (LibreOffice 26.2.x, CPU): calculateAll() ->
            |              values + #REF!/#VALUE!/#NAME?
            +-- A2 + self-tests: 3 perturbed copies -> replay log -> R1-R4 -> repair (<= 2)
                (A2p placebo: fixed checklist, same repair budget)
                              |
  LOGS    per-turn snapshot + action log + tokens -> SQLite / Parquet
  SCORE   evaluation.py (--backend libreoffice) | transfer replay on T | anytime scoring
  STATS   McNemar (run 1), paired bootstrap, non-inferiority, Holm; divergent set apart
  RFT     verified 4B trajectories -> L-token turn segments (4k, else 2k) -> 02 trainer (QLoRA) -> GGUF
```

## 7. Component Design

### 7.1 Working copy and the single writer
- Each run copies `input.xlsx` to its directory; all arms write via one openpyxl writer, snapshotting each turn (sha256 logged) for anytime scoring (§9).
- **Cached values**: openpyxl drops them on save, so every arm serves cells neither written nor downstream of a write from a `data_only=True` load of the original `input.xlsx`. Arms differ only on written cells and dependents (engine: recalculated; A0: formula text, value `None`, dependents stale). S1 exit test: A0 and A1 agree on all other cells of 20 dev workbooks; mismatches join the divergent list.
- **Prefixer and array writer**: newer functions need OOXML prefixes (`_xlfn.XLOOKUP`; `_xlfn._xlws.FILTER`, `_xlfn._xlws.SORT`), else both engines return `#NAME?`. Calc does not spill, so array-returning functions (FILTER, UNIQUE, SORT, SEQUENCE, …) become an openpyxl `ArrayFormula` over the call's target range. S1 golden test: after LibreOffice recalculation each matches Excel's cached values on dev goldens; goldens needing spill join the divergent list. With one shared writer, C1 cannot win on a serializer bug.
- **Reference shifter**: openpyxl does not shift references, so `insert_rows`/`delete_rows` rewrite formulas, defined names and merged ranges via openpyxl's formula tokenizer. S1 exit test: on 50 dev workbooks, recalculated values match LibreOffice's own UNO insert/delete.

### 7.2 Recalculation workers
- 2–4 workers per machine, each a `soffice --headless` with its own local pipe and `-env:UserInstallation` profile, over the UNO bridge.
- Per write: load the snapshot, `calculateAll()` (never the load-time setting, which S0 pins), read the written range, its `answer_position` dependents and all error cells, close unsaved; the writer stays sole author. p50/p95 logged; at 30 s the worker restarts and returns `recalc_timeout`.
- Scoring uses the evaluator's `--convert-to` path; S0 checks both paths agree on all 400 Verified goldens on both OSes. Mismatches, and goldens whose recalculated answers differ from Excel's cached values, enter the engine-divergent list.

### 7.3 Structure map (C3)
- **Regions**: connected components of non-empty cells (one-cell gaps bridged, merges unioned), headers by type change, each with range, headers, row count.
- **Anchors** (SheetCompressor-style): headers, boundaries, rows where a column's type or format changes, 3 samples; interiors become `… 412 rows like row 7 …`. Text value → address ranges; numbers by format (`C8:C419 number 0.00, 12.30–987.10`); formulas as R1C1 groups (`E8:E419 = RC[-2]*RC[-1]`).
- **Budget** ≤ 3k tokens; beyond it, coarse regions collapse to one line, read on demand. **A3**: every non-empty cell as an `address,value|formula` row (SpreadsheetLLM's vanilla encoding), truncated with a marker.

### 7.4 The agent loop
1. **Observe** (prompt ≤ 8k tokens), ordered so the KV prefix cache survives: a static block (system prompt, tool schemas, supplied instruction, `instruction_type`, `answer_position`, turn-1 map), then an append-only log of calls, results (≤ 1.5k tokens) and map deltas of written cells (engine arms: values; A0: formula text). Past 8k, turns older than the last 4 collapse to one-line records, a counted cache break.
2. **Plan** (turn 1, required): JSON listing each target range, `formula` or `literal`, and source ranges for computed targets; it drives §7.6–7.7.
3. **Act**: ≤ 4 read-only calls or exactly 1 write per turn.

| Tool | Engine arms return | A0 returns |
|---|---|---|
| `find_cells(text or number)` | ≤ 20 addresses with values | same |
| `read_range(sheet, range ≤ 400 cells, mode)` | written formulas show computed values | formula text, value `None` |
| `write_formula(sheet, range, A1 formula)` (relative fill); `write_values`, `copy_fill`, `insert_rows`, `delete_rows` | first 20 values; errors in range and dependents in `answer_position` | "wrote N cells" / "shifted N references" |
| `run_python(code)`, openpyxl + pandas; counts as the turn's write | stdout ≤ 1.5k tokens + answer-region diff (values) and readback | stdout + answer-region diff (formula text or literals) |
| `submit` | A2: self-test report or done | done |

**Budgets**: `submit` or 12 assistant turns; per turn ≤ 1k completion tokens (+ ≤ 4,096 reasoning for thinking models); 600 s wall clock. At any cap the current snapshot is scored, reason logged. A malformed call gets one retry (counted as a turn). `run_python` is a subprocess (20 s CPU) whose saves are diffed and re-applied through the single writer. No arm's environment has `formulas`, `pycel` or LibreOffice's Python; a `sitecustomize` blocks and logs sockets, `subprocess` and `os.exec*` (a guard, not a sandbox: 06 Ctrl-Z).

### 7.5 Arms

| Arm | Machine, model, weights | Feedback | Claim |
|---|---|---|---|
| A0 | PC, Qwen3-4B-Instruct-2507 | none | C1 |
| A1 / A1′ | PC, 4B (A1′: new seed) | readback | C1; C2, C3 baseline / A/A |
| A2 | PC, 4B | readback + self-tests | C2 |
| A2p | PC, 4B, T only | readback + placebo (§7.6) | C2 control |
| A3 | PC, 4B, 1 slot, full-sheet encoding (16k cap) instead of the map | readback | C3 |
| A1@L / A4 | PC, 4B, L view (§7.8); A4 = QLoRA RFT weights | readback | C4 |
| A5 | PC, Spreadsheet-RL-4B (published GRPO) | readback | C4 comparison |
| M0 / M1 | Mac, Qwen3-8B | none / readback | C1 replicate or headline |
| M5 | Mac, Spreadsheet-RL-8B (published GRPO) | readback | C4 comparison |

**Fairness**: A0 keeps every tool, the shared writer, cached values and evaluator, so its correct formulas score; only readback differs (§7.1). Spreadsheet-RL trained on other tool names (`inspect_range`, `fill_formula`, `recalculate_and_read`, `code_interpreter`, …); a 100-task check under those names is stretch.

### 7.6 Metamorphic self-tests (C2)
On `submit` (A2) the harness builds K = 3 scratch copies: two scale numeric cells in the plan's source ranges, each by its own factor from [0.5, 1.5] (integers ± 1–5), skipping lookup keys and criteria; one permutes data rows within their region. Formulas, `answer_position` and non-source cells stay fixed. It replays the action log onto each (§7.7), recalculates and checks:

| Relation | Fails when |
|---|---|
| R1 sensitivity | a computed target (plan: `formula`, or sources given) keeps its value on both scaled copies despite changed sources; on the permuted copy, a row-local target does not move with its row, or a SUM/AVERAGE/COUNT-family aggregate changes. Order statistics and counts (MAX, COUNTIF, …) logged apart |
| R2 errors | an error value appears in `answer_position` on any copy |
| R3 fill consistency | a target column mixes literals and formulas, or its formulas differ in R1C1 form |
| R4 code replay | a `run_python` output in a computed target is unchanged on a perturbed copy |

`submit` returns the ≤ 600-token report; ≤ 2 repair rounds fit in the 12-turn cap, so A2 pays for its checks. **Placebo A2p** gets a fixed checklist ("check that computed targets are live formulas; check for error values") and the same repair budget, isolating test results from a second look. `literal` targets without sources are never R1-flagged. **False alarms** (dev-200 runs flagged yet passing all replayed cases): ceiling 15%, else R1 and R3 are tuned on dev before any Verified run.

### 7.7 Transfer replay and failure classes
The **replayer** applies logged writes, structural edits and `run_python` code, in order, to another input (absolute addresses make it deterministic). On T the agent sees only case 1; its log is replayed onto v0.1 cases 2–3, which the evaluator scores. **Transfer pass** = all three pass (the hard restriction, under replay). Failure classes (by rule, non-exclusive):

| Class | Rule |
|---|---|
| Constant-shadowing (CSF) | a replayed case fails on a cell written as a literal by `write_values` or `run_python`, whatever case 1 does |
| Range-shadowing | a replayed case fails on a formula whose references stop short of that case's data |
| Engine error / wrong target | an error value in `answer_position` on case 1 / case 1 fails with no `answer_position` cell written |

**C2 go/no-go**: C2 is built only if A1's CSF count on audited T after the S1 Verified run is ≥ 30, where a 50% cut has ≈ 80% power at Holm's first step (McNemar, 10% reverse discordance assumed; recomputed in `prereg.yaml`). Below that, the frequency is the published finding, C2 leaves the Holm family, and S3 moves to C4.

### 7.8 RFT pipeline (C4)
1. **Pool**: 1,500 deduplicated Spreadsheet-RL train tasks (§5.4), stratified by sheet count.
2. **Collect**: the 4B in the A1 harness at the **L view**, 2 samples per task. L = 4k (system prompt, map ≤ 1.5k, last 2 turns), or 2k if the week-2 QLoRA smoke test fails; fixed before collection. Success = the task's oracle check under our evaluator. If night 1 projects < 300 trajectories, ≤ 500 failed tasks get 2 more samples (ledger row), then labelled Mac Qwen3-8B traces join.
3. **Segment**: one example per assistant turn, rendered exactly as the L view (02's view-parity test); loss on assistant tokens; over-L examples dropped and counted.
4. **Train**: Unsloth QLoRA, r = 16, α = 32, all linear layers, 2 epochs, batch 1 × accumulation 16, gradient checkpointing; one night ≤ 10 h. Merge, GGUF Q4_K_M, same runner (A4).
5. **Compare**: A4 vs A1@L, pre-registered (§9). Spreadsheet-RL-4B used 60 GRPO steps × 1,024 rollouts on 4 × H100 (~40 h), A4 one laptop GPU overnight; ratio stated beside the gap.

### 7.9 Fund factor-screen demo
SEC EDGAR XBRL (eXtensible Business Reporting Language) `companyfacts`, fiscal 2025, 30 US non-financial large caps: net income, equity, revenue, gross profit, liabilities, assets; fair-access User-Agent, no prices. Uneven tags: gross profit falls back to Revenues − CostOfRevenue, liabilities to LiabilitiesAndStockholdersEquity − StockholdersEquity; gaps logged. Task: "ROE, gross margin, leverage; standardize; composite = mean z-score; rank; flag the top five" (live `STANDARDIZE`, `AVERAGE`, `STDEV.P`, `RANK.EQ`). **Week-6 demo, S1 parts only**, Qwen3-4B (Mac 8B if it fails): a real A0 run with a pasted or Python-computed composite beside an A1 run with live formulas; edit one net income and only A1's ranks move. It opens on a real failed transcript; if C2 is go, S3 adds R1 flagging the pasted composite. A screen capture, not a benchmark.

## 8. Hardware Profiles

**Rule: the weakest machine sets the defaults**: memory from the 6 GB card, speed from each machine's week-2 measurement.

- **Profiles**: `portable` (default): 4B arms at 2 × 10k q8_0 on 6 GB, 8B arms on the Mac. `accelerated` (opt-in): 4B at 4 × 10k on a measured 8 GB card or the Mac; S0 at 1 × 32k without fallback.
- **Placement**: Windows also runs S0, QLoRA (WSL2), SpreadsheetBench 2; the Mac also runs dedup embeddings. Golden recalculation runs on both, answer regions hash-compared.

**Memory** (weights per §4.1; KV q8_0; compute buffers ~0.3–0.5 GiB for the 152k-vocabulary logits at ubatch 512, estimated, measured week 2):

| Process | Weights + KV | 6 GB | 8 GB | Mac 16 GB |
|---|---|---|---|---|
| 4B, 2 × 10k (A0–A2p, A1@L, A4) / 1 × 18k (A3); Spreadsheet-RL-4B, 1 × 16k (A5) | 2.33 + 1.49 = 3.8 / 2.33 + 1.34 / 2.53 + 1.20 = 3.7 GiB | fits | fits | fits |
| 1 × 32k (Spreadsheet-RL's 4,096 + 27,648 limits): 4B (S0) / Spreadsheet-RL-4B (23.4 subset) | 2.33 + 2.39 = 4.7 / 2.53 + 2.39 = 4.9 GiB | tight / likely no; 24k = labelled deviation | fits | fits |
| 4B, 4 × 10k (accelerated) | 2.33 + 2.99 = 5.3 GiB | no | tight | fits |
| Qwen3-8B, 2 × 10k (M0, M1) / Spreadsheet-RL-8B, 2 × 14k (M5) | 4.68 + 1.49 = 6.2 / 4.68 + 2.09 = 6.8 GiB | never | never | fits |
| QLoRA RFT, 4B, ≤ L-token segments, batch 1, checkpointing | Unsloth minimums: 3.5 GB (3B), 6 GB (8B); 4B smoke-tested week 2 | tight | fits | no CUDA |

UNO workers use system RAM (planning 0.5 GB each; to be measured).

**Week-2 go/no-go** (end of session 2, before any Verified arm contrast; S0's thin A0/A1 loop, proxy rates re-checked in the S1 pilot): PC 4B prompt/decode tokens/s at 8k; median s/task on 20 dev tasks for A0, A1 (2 slots), Spreadsheet-RL-4B and both Mac 8B models; 1 × 32k fit of both 4B models on 6 GB; a 50-step Unsloth QLoRA smoke test (4B, 4k segments, batch 1, checkpointing, full-vocabulary logits) that fixes L; UNO p50/p95 per write on Verified workbooks; `soffice` RSS; S0 parity (§10). A failed golden parse test or recalculation p95 > 10 s is fixed before S1.

**Compute ledger** (planning s/task until week 2). PC: non-thinking 4B 60 over 2 slots (A2, A2p 75; A3 100, 1 slot); Spreadsheet-RL-4B 180 (A5); S0 at 1 × 32k and 20 turns 120. Mac (base M-series chip assumed, exact chip recorded in S0; §7.4's cache-preserving prompt): Qwen3-8B 240, Spreadsheet-RL-8B 600 (thinking). llama.cpp discussion #4167: a 7B Q4_0 runs 108–221 prompt and 14–24 decode tokens/s on base M1–M4.

| Item | Runs | PC h | Mac h |
|---|---|---|---|
| S0 reproduction 400 × 1; variance pilot 100 × 2; Mac 8B pilot 100 | 700 | 16.7 | 6.7 |
| Dev iteration, dev-200 × 6 passes | 1,200 | 20.0 | |
| C1 (A0, A1, A1′), C3 (A3), C2 (A2 + 200 dev; A2p on T, planning 300) on 400 | 2,500 | 49.9 | |
| RFT collection 1,500 × 2; QLoRA two nights | 3,000 | 70.0 | |
| RFT retry, ≤ 500 failed tasks × 2 (§7.8 yield rule only) | 1,000 | 16.7 | |
| C4: A1@L, A4, A5 (400 each) | 1,200 | 33.3 | |
| SpreadsheetBench 2 FM + Debugging, A1 and A2 | 400 | 7.5 | |
| M0, M1 on 400; M1′ only if the Mac is the C1 headline | 1,200 | | 80.0 |
| M5 on 200 | 200 | | 33.3 |
| **Total** | **11,400** | **≈ 214** | **≈ 120** |

**Gate rules.** Budget: 220 unattended PC hours (22 nights × 10 h), 150 Mac hours, confirmed at S0. PC ≤ 62 s/task: full plan; 62–80 s: cuts 1–4 (≈ 170 planning hours; ≈ 220 at 80 s); > 80 s: cuts 1–6, then RFT pool 600, C4 labelled underpowered. Mac Qwen3-8B > 240 s/task: drop M1′, C1's headline stays on the PC; > 320 s: M0/M1 → 200 tasks, a descriptive replicate. Spreadsheet-RL-8B > 600 s: M5 → 100.

**Compute cuts (pre-registered order)**: (1) SpreadsheetBench 2's A2 arm; (2) dev passes 6 → 4; (3) the RFT retry (Mac traces only, within its ~30 spare hours); (4) RFT pool 1,500 → 1,000; (5) A5 400 → 200; (6) A1′ 400 → 200 (MDE then rests on the pilot).

## 9. Evaluation Plan

| Rule | Pre-registered setting |
|---|---|
| Unit, family | Task, run 1 primary; Holm, family-wise α = 0.05 over C1, C2 (if go), C3's token ratio, C4 |
| MDE | Holm's first step (α/4 two-sided), power 0.8, McNemar normal approximation: (z₀.₉₉₃₇₅ + z₀.₈)·√(d/n) = 3.34·√(d/n); d = discordant-pair share |
| Pilot | n is fixed at 400, so the pilot fixes the MDE: A1, 100 Verified tasks × 2 seeds, yields p̂ and A/A discordance d̂ only; no arm contrast before `prereg.yaml` is committed. Planning d = max(d̂, 0.10): arm-vs-arm discordance includes the treatment, and frontier noise is at that scale (§5.1) |
| Second run | Planned MDE > 6 pp (d > 0.13): A0 and A1 run twice, paid from cuts 1–3 (§8); paired bootstrap on per-task means |
| Headline | If the 4B's pilot p̂ < 12% (likely, given 9.3% published), C1 may move to the Mac (M0 vs M1). It moves only if the 8B, piloted on the same 100 tasks, scores ≥ 5 pp higher and passes the Mac speed gate (§8); the PC C1 then becomes secondary. Else C1 stays on the 4B, labelled low-base-rate |
| F (C3 set) | Verified tasks whose full encoding is ≤ 12k tokens, so A3 sees all of it in its 16k cap; \|F\| measured in S0; outside F, A3 is truncated |

| Area | Metrics | Compared against | n and MDE |
|---|---|---|---|
| C1 engine | Verified-400 Pass@1 excluding engine-divergent tasks; tokens, turns, recalculations per task; accuracy-vs-cumulative-token curves from per-turn snapshots (token-matched when equal caps yield unequal spend) | A0 (M0 vs M1 if the Mac is the headline) | ≈ 400: 5.3 / 6.5 / 7.5 pp at d = 0.10 / 0.15 / 0.20 |
| C2 self-tests (if go) | unconditional CSF indicator per task on T (primary); case-1 Pass@1 (guardrail); transfer pass; false-alarm rate | A2p (test); A1 (descriptive) | n = \|T\|; ≥ 50% relative CSF cut |
| C3 structure map | per-task prompt-token ratio: one-sided Wilcoxon signed-rank of log ratio against log 3 (the Holm p-value); Pass@1 on F; all 400 secondary | A3 | ratio ≥ 3×. Accuracy margin δ = max(2 pp, 20% of A3's Pass@1 on F), descriptive (95% CI against δ): at d = 0.10, 2 pp needs n ≈ 1,550 (one-sided 0.05, power 0.8), far above \|F\| ≤ 400 |
| C4 RFT | Verified-400 Pass@1 | A1@L (test); A1, A5, M5 (descriptive) | 400; as C1 |
| Thesis | per-task harness gain (A0 → A2, or A1) minus RFT gain (A1@L → A4) and minus 4B → 8B gain (A1 → M1) | — | paired bootstrap CI; descriptive |

**Reporting rules**: success only from the evaluator on pinned LibreOffice; "under our harness" beside every published number; vendor numbers labelled, aggregator rows dated; divergent tasks apart; transfer metric labelled ours; `prereg.yaml` (arms, caps, seeds, tests, margins, cuts) hashed after the pilot, deviations listed; build, LibreOffice and GGUF hashes on every table; nulls published; no LLM judge; no comparison with a model never run.

## 10. Milestones (19 hours / 15 sessions)

| Sprint | Sessions | Deliverable |
|---|---|---|
| **S0** | 1–3 | Repo, `uv`; pins (llama.cpp, LibreOffice point release, GGUF hashes); Unsloth in WSL2; evaluator parity on both OSes (divergent list); UNO worker; thin A0/A1 timing loop (raw read/write tools, no map); week-2 go/no-go (§8); T built; dedup started |
| **S1** | 4–6 | Typed tools; single writer (exit tests, §7.1); map v1; A0 vs A1 on dev-200; session 5: variance and Mac 8B pilots, then `prereg.yaml`; Verified-400 with A1′ (C1); transfer replay, T audit; CSF count → C2 go/no-go (session 6). Overnight in S1–S2: Gym-compatible shim, S0 reproduction |
| **S2** | 7–8 | Map v2, A3 on F (C3); Mac M0/M1; week-6 demo (§7.9) |
| **S3** | 9–10 | If go: self-tests A2 and placebo A2p (C2), dev false-alarm rate first; RFT collection overnight; week-8 checkpoint (session 10, as in 05 and 09): planned-tier cuts |
| **S4** | 11–14 | QLoRA via 02's trainer (its QLoRA path; 02 defaults to bf16 LoRA, §4.1); A1@L, A4, A5, M5 (C4); SpreadsheetBench 2; final tables; releases |
| — | 15 | Interview sprint; write-up (§14) |

**S0 reproduction rule (pre-registered).** Target: Qwen3-4B-Instruct-2507's 9.3% (§5.2). One run (the A/A pilot covers run noise): 400 distinct tasks, one random case each, stratified by `instruction_type`; Q4_K_M, PC, 1 × 32k (fallback §8); Gym-compatible tool shim over LibreOffice with Spreadsheet-RL's harness prompt, decoding (§4.1) and caps (20 turns, 8,192 tool response). Pass: 9.3% inside the 95% Wilson interval (≈ ±2.9 pp at 9%; 400 of 912 tasks sampled), else the gap is documented (Excel vs LibreOffice, bf16 vs Q4_K_M, the shim) and no claim depends on it. Parity alongside: recalculation under the pin changes 0 verified_400 outcomes and PR #32's tasks on the 912 set.

**Tiers.**
- **Core** (ships regardless): S0 parity and reproduction, C1 harness and arms, C3, transfer replay and the CSF finding, the demo.
- **Planned**, cut in this pre-registered order if behind at week 8: (1) SpreadsheetBench 2; (2) M5; (3) RFT pool 1,500 → 800; (4) self-test copies K = 3 → 1.
- **Stretch**: Spreadsheet-RL-4B/-8B (§5.2) on 200 tasks of the S0 subset under its caps (memory per §8); `formulas` middle arm; Qwen3.5-4B C1 replicate; tool-name shim; SpreadsheetBench 2 Template; a Domain-Spreadsheet Finance-B sample; win32com Excel cross-check; a 26.8.x parity check.

## 11. Risks & Mitigations

| Risk | Mitigation |
|---|---|
| LibreOffice and Excel compute differently (PR #32, §4.3); Calc does not spill | Pinned release, both-OS golden recalculation, array writer, divergent list, optional win32com (§4.3, §7.1–7.2) |
| 4B floor too low for powered ablations (published 9.3%) | Pilot first, headline rule, anytime curves, second run (§9) |
| Mac slower than planned (base M-series chip) | Cache-preserving prompt; week-2 Mac gate (§8) |
| Forum near-duplicates leak into the RFT pool | Two-stage dedup against every benchmark task, list published (§5.4) |
| T is small, its hidden cases unreviewed | Sized in S0, audited in S1 (§5.2); if \|T\| < 150, non-Verified v0.1 tasks outside dev-200 form a labelled secondary set |
| Constant-shadowing too rare to power C2 | Go/no-go at 30 A1 CSF tasks; the frequency is the finding (§7.7) |
| Self-tests flag correct sheets | Per-cell factors, keys never perturbed, 15% false-alarm ceiling (§7.6) |
| QLoRA at 4k does not fit 6 GB; low RFT yield | Week-2 smoke test can fix L = 2k, view parity kept; optional free Colab/Kaggle GPU; retry row (§7.8, §8) |
| Scope exceeds 19 h + independent time | Tiers and cut orders (§8, §10); S4 holds only C4 and final tables |

## 12. Résumé Bullets (experience voice; numbers filled in as sprints land)

- Built **Cellmate**, a local spreadsheet agent (Qwen3-4B on a 6 GB laptop GPU, Qwen3-8B on a Mac) that recalculates live formulas in headless LibreOffice after every edit; **engine-in-the-loop** lifted SpreadsheetBench Verified-400 Pass@1 **X%** → **X%** vs a Python-only arm at matched caps (paired, n = **X**, p = **X**).
- Designed **metamorphic self-tests** (edits replayed on perturbed scratch copies, invariants checked) that cut hard-coded-constant failures on hidden test cases **X%** vs a placebo review turn (**X** → **X** tasks) at **X%** false alarms.
- Built a **SheetCompressor-style structure map** that cut prompt tokens **X×** (Pass@1 change **X** pp, 95% CI **X** to **X**).
- Fine-tuned Qwen3-4B overnight with **QLoRA rejection sampling** on **X** verified trajectories (deduplicated against every benchmark task, list published): **X%** vs **X%** base, same harness. Spreadsheet-RL-4B (GRPO, 4 × H100) reported beside it as descriptive only: off its trained tool schema, in LibreOffice, at Q4_K_M.
- *Null and no-go variants*: no detectable gain at an MDE of **X** pp; constant-shadowing was **X%** of hidden-case failures, below the pre-registered build threshold.

**Placement note.** These read as experience only under a real engagement: the mentoring organization (engagement record, truthful title), a real collaboration with a startup, lab or open-source org hosting the work, or an open-source project entry with public commits. A Changsheng entry is legitimate only for work actually done there, such as a factor-screen tool built with the firm's approval. Never invent an employer or dates: North American employers verify, and a fabricated entry can rescind an offer.

## 13. Why This Reads as Real Company Work

- **Spreadsheet copilots ship** (Copilot Agent Mode, Shortcut, GRID and WPS AI post SpreadsheetBench numbers). An engine in the loop costs Excel licences on Windows workers or LibreOffice fidelity risk; Cellmate supplies the controlled number for that call.
- **Stale hard-coded values are a known spreadsheet risk** (EuSpRIG); transfer replay asks the auditor's question: does the sheet still work on new inputs?
- **Operational knowledge**: engine divergence, OOXML prefixes, non-spilling arrays, reference shifting, recalculation latency, cross-OS golden parity, and an analysis that admits a low base rate.

## 14. Interview Leverage

- **ML system design** ("design a copilot for Excel"): engine placement, worker pools, single writer, token-budgeted sheet encoding, pre-submission verification.
- **Applied LLM**: KV math, quantized-V constraint (§4.1–4.2), thinking budgets, RL-trained models under a new tool schema. **Post-training**: RFT vs GRPO cost, decontaminating forum data, view parity.
- **Experimentation**: McNemar vs paired bootstrap, MDEs at Holm's first step, placebo arms, power-gated builds, A/A pilots when n is fixed.
- **Drills** (practice prompts): *"It passes the visible test; does it generalize?"* → §7.6–7.7; *"Is the engine ablation fair?"* → §7.1, §7.5; *"Is the self-test gain just a second look?"* → §7.6; *"Decontaminate data from the benchmark's own forums."* → §5.4.
- **Write-up** (session 15, mentee's byline), working title *"Does a 4B spreadsheet agent need a calculator? I measured it on a laptop"*: hook, context, the work (failure first: a pasted literal passes case 1, fails cases 2–3; then recalculation, self-tests, the paired test), what I'd do differently (pre-declared null or fallback, §7.7, §9), takeaways; numbers trace to the results table.

## 15. References

- SpreadsheetBench: https://github.com/RUCKBReasoning/SpreadsheetBench · https://arxiv.org/abs/2406.14991 · https://huggingface.co/datasets/KAKA22/SpreadsheetBench · PR #32: https://github.com/RUCKBReasoning/SpreadsheetBench/pull/32
- Verified-400: https://shortcut.ai/blog/posts/spreadsheetbench-verified · https://grid.is/blog/an-agent-with-a-spreadsheet-engine-beats-one-without · aggregator https://benchmarklist.com/benchmarks/spreadsheetbench_v1_verified/
- SpreadsheetBench 2: https://github.com/RUCKBReasoning/SpreadsheetBench-2 · https://arxiv.org/abs/2606.29955
- Spreadsheet-RL: https://arxiv.org/abs/2605.22642 (Tables 1 and 3; appendix on Excel vs LibreOffice and `formulas`) · https://github.com/Spreadsheet-RL/Spreadsheet-RL · https://huggingface.co/datasets/Spreadsheet-RL/Spreadsheet-RL · https://huggingface.co/Spreadsheet-RL/Spreadsheet-RL-4B · https://huggingface.co/Spreadsheet-RL/Spreadsheet-RL-8B · https://huggingface.co/mradermacher/Spreadsheet-RL-4B-GGUF · https://huggingface.co/mradermacher/Spreadsheet-RL-8B-GGUF
- SpreadsheetLLM: https://arxiv.org/abs/2407.09025 · SpreadsheetAgent: https://arxiv.org/abs/2604.12282 · SheetBrain: https://arxiv.org/abs/2510.19247
- Models: https://huggingface.co/Qwen/Qwen3-4B-Instruct-2507 · https://huggingface.co/unsloth/Qwen3-4B-Instruct-2507-GGUF · https://huggingface.co/unsloth/Qwen3-8B-GGUF · https://huggingface.co/Qwen/Qwen3-Embedding-0.6B-GGUF
- LibreOffice: https://wiki.documentfoundation.org/ReleaseNotes/24.8 · https://borncity.com/blog/2026/08/27/libreoffice-26-8-veroeffentlicht/ · https://wiki.documentfoundation.org/ReleasePlan · https://bugs.documentfoundation.org/show_bug.cgi?id=126573 (spill) · https://github.com/unoconv/unoserver
- Libraries: https://github.com/vinci1it2000/formulas · https://docs.unsloth.ai/get-started/beginner-start-here/unsloth-requirements · https://unsloth.ai/docs/models/qwen3.5/fine-tune · https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md · Apple Silicon rates: https://github.com/ggml-org/llama.cpp/discussions/4167
- Demo, risk: https://www.sec.gov/edgar/sec-api-documentation · https://eusprig.org/
