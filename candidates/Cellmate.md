# Cellmate — Candidate Brief (Round 2, rank 1)

> **Promoted.** The full proposal is [10 Cellmate](../10_Cellmate_Design_Doc.md). This brief is kept as the input it was written from.

> Not yet a full design doc. This is the brief a writer would expand into one, with the judges' fixes already folded in. Facts were web-checked in October 2026 and still need the usual review pass.

A laptop spreadsheet agent that writes live formulas, recalculates with LibreOffice after every write, and fuzzes its own edits before it hands in the sheet. The claim: engine-in-the-loop beats a Python/openpyxl-only arm at matched turns on SpreadsheetBench Verified-400. Metamorphic self-tests also cut the 'passes test case 1, fails cases 2-3' hard-coded-constant failure. Vendors claim engine beats Python but publish no controlled ablation, and Excel is the most-used programming environment there is.

| | |
|---|---|
| Lens | science / data-work product |
| Benchmark | SpreadsheetBench Verified-400 (local LibreOffice recalc scoring; no published ≤ 8B number). Open baselines are on the full 912 set, per test case, Excel 365 gym: Qwen3-4B-Instruct-2507 9.3 / Qwen3-8B 15.9 / Spreadsheet-RL-4B 23.4 / Spreadsheet-RL-8B 22.3 Pass@1; SpreadsheetBench 2 Financial_Model + Debugging only |
| Models | Qwen3-4B-Instruct-2507 Q4_K_M (PC, q8 KV, 16k); Spreadsheet-RL-4B (PC); Qwen3-8B and Spreadsheet-RL-8B 4-bit (Mac); Qwen3-4B QLoRA RFT |
| Hardware fit | 4B agent + LibreOffice headless CPU workers on PC; 8B arms on Mac; QLoRA RFT at <=4k segments fits 6 GB |
| Best for roles | AI engineer (copilots / agent products), fintech / enterprise AI engineer, applied scientist (agents, post-training) |
| Relation to existing docs | New domain. It is close to Shellwright only in having a harness ablation plus distillation, and close to round-1 MetricPilot only in being data work (MetricPilot was read-only SQL). The executable critic upgrades the College-Agent Writer-Critic story. Strong Changsheng fit: fund models live in Excel. | 
## Judge fixes folded in

- Sprint 0 re-recalculates golden files under pinned LibreOffice and sets engine-divergent tasks aside
- Reproduce Spreadsheet-RL-8B under the local harness (win32com if Excel present), label results 'under our harness'
- 100-task variance pilot before pre-registering the 5-point MDE; if 4B < ~12%, make the Mac 8B arm the headline
- Measure the frequency of constant-shadowing failures before building C2
- Dedup 512 train tasks vs Verified-400 by instruction embedding + file hash; publish the list
- 8B arms on the Mac only; PC stays at 4B
- Add one fund-style workflow demo (factor screen with live formulas)

## Corrections after verification (2026-10-23)

- The Spreadsheet-RL numbers are on the full 912 set (2,722 per-test-case rollouts, Excel 365 v2512), not on Verified-400. The "split unstated" note was wrong.
- A Spreadsheet-RL-4B checkpoint exists (Qwen3-4B-Thinking-2507 base, 23.4) and fits the 6 GB PC.
- Verified-400 has one test case per task. The 3-per-instruction figure applies only to the 912 set.
- Spreadsheet-RL's ExcelForum train split (5,925 tasks after 2024-01-01, CC BY-SA 4.0) is a larger RFT pool than the 512 non-Verified tasks. The full design doc (10) uses it, deduplicated.

## Brief

**Thesis.** Most of the gap between a 4-8B local model and frontier spreadsheet agents on SpreadsheetBench Verified-400 is a harness problem, not a parameter problem. Frontier scaffolds now report about 86-99% on Verified-400 (vendor-reported; an aggregator snapshot of 2026-10-18 tops out at 99.25%, and listings disagree). On the full 912-task set under Spreadsheet-RL's Excel 365 gym (per test case), Qwen3-4B-Instruct-2507 scores 9.3 Pass@1, Qwen3-8B 15.9, Spreadsheet-RL-4B 23.4 and Spreadsheet-RL-8B 22.3. Claims:
- **C1.** Engine-in-the-loop beats a Python/openpyxl-only arm at matched turns and tokens, paired by task, with an MDE from the pilot. In the engine arm, LibreOffice headless recalculates after every write and the agent reads back values plus #REF!/#VALUE! errors.
- **C2.** Metamorphic self-tests cut the hard-coded-constant failure class by a pre-registered fraction. The test is to perturb input cells on a scratch copy, recalculate, and check the invariants. The full 912 set has about 3 test cases per instruction, but Verified-400 ships only one (800 files for 400 tasks), so C2 is measured by replaying the agent's edits onto the 912 set's cases 2-3.
- **C3.** A structure map (table-region detection plus SheetCompressor-style anchors) cuts prompt tokens by at least 3x with no accuracy loss.
- **C4.** QLoRA rejection-sampling fine-tuning of Qwen3-4B on verified trajectories from the 512 non-Verified tasks is compared with Spreadsheet-RL-8B under the same harness.

**Agent loop.**
1. Observe through the compressed structure map.
2. Plan target ranges and choose formula vs literal values.
3. Act through typed tools: read_range, write_formula, write_values, insert_rows, copy_fill, and run_python as a fallback.
4. Recalculate through UNO.
5. Self-test with perturbations on a scratch copy.
6. Repair or submit.
Budget: at most 12 turns and an 8k prompt. Regions that do not fit are read on demand.

**Models and memory.**
- PC (6 GB): Qwen3-4B-Instruct Q4_K_M, ~2.5 GB weights plus ~1.2 GB q8 KV at 16k, with 2 slots.
- Mac (16 GB): Qwen3-8B and Spreadsheet-RL-8B as MLX 4-bit (~5 GB). Per the judges, an 8B with 8k context is too tight for the PC.
- QLoRA RFT of the 4B: Unsloth, at most 4k-token segments, overnight.

**Benchmarks.**
- Verified-400 is public (HF/GitHub), and its evaluator recalculates with LibreOffice 7.5+ on Windows, macOS and Linux. On Windows the default backend is win32com whenever Excel is installed, so `--backend libreoffice` must be forced.
- A stratified 200-task lite subset is used for development.
- One full pass is about 3.6M tokens, or 1-2 nights per arm.
- SpreadsheetBench 2 runs only for the Financial_Model and Debugging categories. Visualization needs Excel COM plus a VLM API, so it is skipped. Its published best is 34.89% and open models score 7-17%.
- Baselines are verified only as listed: Shortcut 86% and GRID 91.25% are vendor-reported, and the aggregators conflict.

**Evaluation and ablations.**
- Arms: engine vs Python-only; with vs without self-tests; full sheet vs structure map; prompted vs RFT; 4B vs 8B.
- Statistics: paired bootstrap at the task level, with a test-case-level breakdown.
- Engine-divergent tasks (dynamic arrays, newer lookups) are reported separately.

**Sprint sketch (12 weeks).**
- W1-2: Sprint 0. Pin LibreOffice, re-recalculate the golden files, run the 100-task variance pilot, and reproduce Spreadsheet-RL-8B.
- W3-4: tool API, recalc loop and the engine-vs-Python ablation (C1).
- W5-6: structure map (C3). Demo a fund factor screen with live formulas.
- W7-8: metamorphic self-tests (C2).
- W9-10: RFT data collection and QLoRA (C4).
- W11-12: full Verified-400 runs, write-up and résumé bullets.

**Top risks and mitigations.**
- LibreOffice and Excel compute differently. Pin the version and separate the divergent tasks.
- A 4B floor below 15% leaves the ablations underpowered. Run the pilot, keep the 8B headline on the Mac, and use stratified subsets.
- Spreadsheet-RL's harness is Excel 365, so its 22.3 may not reproduce. Label every result 'under our harness'.
- Near-duplicate forum tasks could leak from the training pool. Deduplicate by embedding and file hash, and publish the list.

**Sources.**
- https://github.com/RUCKBReasoning/SpreadsheetBench
- https://shortcut.ai/blog/posts/spreadsheetbench-verified
- https://grid.is/blog/an-agent-with-a-spreadsheet-engine-beats-one-without
- https://benchmarklist.com/benchmarks/spreadsheetbench_v1_verified/
- https://huggingface.co/Spreadsheet-RL/Spreadsheet-RL-8B
- https://github.com/RUCKBReasoning/SpreadsheetBench-2
- https://arxiv.org/html/2407.09025v1
