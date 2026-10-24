# Tact — Candidate Brief (Round 2, rank 6)

> Not yet a full design doc. This is the brief a writer would expand into one, with the judges' fixes already folded in. Facts were web-checked in October 2026 and still need the usual review pass.

A proactive assistant that knows when to keep quiet. A Qwen3-0.6B gate on CPU decides whether to spend the user's attention, and conformal risk control gives it a distribution-free false-alarm budget. Its threshold adapts online to each user's patience. The hook: a published fine-tuned 7B on ProactiveBench (recall 100, false-alarm 50.22) scores what 'always interrupt' would score. Tact reframes proactive AI as alert-fatigue engineering.

| | |
|---|---|
| Lens | Personal & private |
| Benchmark | ProactiveBench (thunlp, ICLR 2025; 233 test events; Qwen2-7B-Proactive F1 66.47 / FA 50.22; PRISM gate as main baseline); ProAgentBench stretch only if downloadable |
| Models | Qwen3-0.6B LoRA + head (CPU-served, Q8/ONNX); Qwen3-1.7B ablation; Qwen3-8B Q4 responder (sequential on PC); ProactiveAgent reward model 8B GGUF Q4 (~4.9 GB); Qwen3.5-4B MLX responder on Mac |
| Hardware fit | Gate LoRA in minutes on PC; reward model and 8B run sequentially; gate on CPU keeps GPU free for live ActivityWatch demo |
| Best for roles | AI engineer assistant/copilot products, applied scientist calibration / decision-making, notifications / recsys ML engineer, on-device ML |
| Relation to existing docs | New. Structurally close to Switchboard (a small model decides, a large one acts), but the scarce resource is human attention and the guarantee is user-facing. Maps to Weibo alert thresholds and Pinterest notification volume. | 
## Judge fixes folded in

- Sprint 0 computes class balance and the always-fire baseline; it is row 1 of the first table
- Reproduce PRISM's gate as main baseline
- Score the gate both via the reward model end-to-end and against gold needs-help labels so 'whether' vs 'what' is clean
- Convert reward model to GGUF and check agreement with bf16 on 50 events
- Cross-validate over 6.8k train events for power; keep the 233 as reported test with scenario-level bootstrap
- Report CRC coverage per scenario including where exchangeability breaks
- ProAgentBench stays stretch

## Brief

**Thesis.** Proactivity is a notification-ranking problem, not a generation problem. ProactiveBench's fine-tuned Qwen2-7B-Proactive reports recall 100, precision 49.78, false-alarm rate 50.22 and F1 66.47. That is arithmetically what 'always interrupt' scores if about 50% of test events are positive. Sprint 0 computes the class balance to confirm or refute this. Claims:
- **C1.** A Qwen3-0.6B LoRA classifier with temperature scaling, estimating P(user accepts help | context), beats a prompted 8B and matches or beats PRISM's gate on F1 and false-alarm rate, at 10x+ lower ms/event on CPU.
- **C2.** Conformal risk control gives FA <= alpha on held-out scenarios. Coverage is measured under coding/writing/daily-life domain shift, including where it breaks.
- **C3.** On simulated users with different miss/false-alarm cost ratios, online threshold adaptation (online conformal or Thompson bandit) reaches target utility within N feedback events, against one global threshold.
- **C4 (stretch).** Deferring help to natural breakpoints (window switch, idle).

**Agent loop.**
1. ActivityWatch events (window, app, web, AFK) feed a rolling text summary.
2. The gate scores P(accept).
3. If P(accept) exceeds the per-user CRC-calibrated threshold, the responder drafts help. Otherwise the 8B is never woken.
4. A delivery policy chooses now or the next breakpoint.
5. Accept or dismiss updates the threshold.

Offline, the loop replays ProactiveBench events and scores them with the released reward model plus gold labels.

**Models and memory.**
- Qwen3-0.6B: LoRA plus head, ~3 GB to train, ~0.7 GB to serve on CPU.
- Qwen3-1.7B ablation, ~4-5 GB to train.
- Qwen3-8B Q4_K_M (~5 GB) as responder and prompted-gate baseline on the PC, run sequentially with the reward model (8B, ~4.9 GB Q4; 91.8% F1 agreement with humans, reported).
- Qwen2-7B-Proactive Q4 re-run.
- Qwen3.5-4B MLX on the Mac for the demo.
- No API needed. An optional frontier prompted gate costs under $5.

**Benchmarks.**
- ProactiveBench: 6,790 train / 233 test events, Apache-2.0.
- Published F1: Qwen2-7B-Proactive 66.47, LLaMA-3.1-8B-Proactive 66.25, prompted LLaMA-3.1-8B 55.06 (FA 61.84), Claude-3.5-Sonnet 62.00.
- PRISM (2602.01532) reports 22.78% fewer false alarms.

**Evaluation.**
- Reliability diagrams and expected-utility cost curves (c_miss vs c_fa) instead of a single F1.
- Scenario-level bootstrap CIs on every number (±~6 F1 on n=233).
- CRC coverage tables.
- Simulated-user regret curves.

**Sprint sketch.**
- W1: class balance, always-fire baseline and reward-model GGUF check.
- W2-3: gate LoRA (C1) and PRISM reproduction.
- W4: live ActivityWatch demo.
- W5-7: CRC (C2) and shift analysis.
- W8-10: per-user online adaptation (C3).
- W11: breakpoint stretch.
- W12: write-up.

**Risks and mitigations.**
- The small test set gives wide CIs. Use cross-validation plus bootstrap.
- Events come from a synthetic gym. State the limit, and use ProAgentBench if it becomes available.
- Exchangeability is violated under shift. Report it.
- Reward-model dependence. Label it, and also report gold-label metrics.
- PRISM narrows novelty. Reproduce it as a baseline.

**Sources.**
- https://arxiv.org/abs/2410.12361
- https://github.com/thunlp/ProactiveAgent
- https://arxiv.org/abs/2602.01532
- https://arxiv.org/abs/2602.04482
- https://iclr.cc/virtual/2025/poster/28128
