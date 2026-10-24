# Twin — Candidate Brief (Round 2, rank 7)

> Not yet a full design doc. This is the brief a writer would expand into one, with the judges' fixes already folded in. Facts were web-checked in October 2026 and still need the usual review pass.

Is your agent smart, or has it seen the test? Twin builds isomorphic twins (bijective renames that keep the optimal plan) and counterfactual twins (policy edits that change the right answer) of agent benchmarks. It calibrates contamination detectors by deliberately poisoning its own model at known doses. A new metric, prior-override rate, asks whether an agent obeys the edited refund policy in its prompt or its memory. A noisy-oracle shrinker (from Shrinkwrap) then finds the minimal twin edit that flips a model's outcome.

| | |
|---|---|
| Lens | Eval science |
| Benchmark | BFCL v4 multi-turn (isomorphic + counterfactual twins, core); ALFWorld valid_seen/unseen isomorphic only (AgentLM-7B 84.0 vs Llama-2-7B-chat 2.0); tau2 retail counterfactual policies on ~30 tasks; WikiMIA detector sanity (Min-K%++) |
| Models | Qwen3-1.7B QLoRA dose ladder (PC, ~3-4 GB); AgentLM-7B Q4 positive control; audit of 4 public models: Qwen3-4B-Instruct-2507, Qwen3.5-4B, Gemma 4 E4B, Llama-3.1-8B (Mac); Qwen3.5-9B MLX user sim |
| Hardware fit | Dose ladder (5 doses x 2 seeds) overnight on PC; ALFWorld and 8-9B on Mac (TextWorld lacks native Windows); no API |
| Best for roles | Applied scientist evaluation / benchmarks, research engineer post-training data, AI engineer model selection |
| Relation to existing docs | No existing doc or round-1 idea measures contamination. It shares the ALFWorld harness with Compass and the BFCL/tau2 harnesses with BlackBox. It absorbs Shrinkwrap's SPRT-gated ddmin as a stretch: shrinking the twin perturbation to the minimal outcome-flipping edit. Quant analogy: a look-ahead bias detector. | 
## Judge fixes folded in

- Core = BFCL-MT twins + dose ladder; ALFWorld isomorphic second; drop ALFWorld PDDL counterfactuals
- tau2 counterfactual policies on ~30 retail tasks only
- Run ALFWorld on Mac or WSL2
- Low doses heavily diluted in clean data so curves do not saturate
- Cut audit to 4 public models, framed as estimates with CIs
- Cite and position against PIPE (2602.01611), Zero-CoT Probe (May 2026) and 2510.02386
- Sprint 0 reproduces AgentLM-7B held-in ALFWorld
- Validate every twin by gold-trajectory replay; TOST equivalence for faithfulness
- Fold Shrinkwrap: SPRT-gated shrinking of perturbation programs as a stretch, baselined on whyflaky

## Brief

**Thesis.** Agent benchmarks now appear in training mixes. AgentLM-7B scores 84.0 on held-in ALFWorld versus 2.0 for Llama-2-7B-chat. Text-membership detectors (Min-K%++) are noisy, and no one has calibrated an agent-contamination detector against a known dose. Claims:
- **C1.** The twin gap detects planted contamination at a lower dose than Min-K%++ on trajectory text, a gold-next-action completion probe, or MinHash/n-gram overlap. Detection AUROC is compared at each dose, pre-registered. Doses: QLoRA on Qwen3-1.7B with 0/1/2/4/8 copies of test trajectories mixed into clean SFT data.
- **C2.** Isomorphic twins are faithful. Uncontaminated base models show a twin gap within ±MDE (TOST).
- **C3.** Prior-override rate on counterfactual twins: the share of tasks where the agent follows the memorised policy rather than the edited one. Reported for 4 public models plus AgentLM-7B.
- **Stretch (from Shrinkwrap).** SPRT-gated delta debugging over the twin perturbation program finds the minimal edit that flips a model, emitted as a pytest counterexample.

**Agent loop.**
1. A per-benchmark twin compiler produces a mapping, the twin task and the remapped gold trajectory.
2. A validator replays the gold trajectory and rejects the twin if it fails.
3. The same ReAct or function-calling agent runs the original and the twin with paired seeds.
4. Outcomes are compared with McNemar tests and bootstrap.
5. In the dose arm, the model is trained at each dose and all detectors are run.

**Models and memory.**
- Qwen3-1.7B QLoRA at <=2k-token segments, ~3-4 GB on the PC.
- AgentLM-7B Q4 (~4 GB).
- Qwen3-4B-Instruct-2507 (~2.5 GB) and Qwen3.5-4B (~3 GB) on the PC.
- Gemma 4 E4B (memory measured) and Llama-3.1-8B Q4 (~4.9 GB) on the Mac.
- Min-K%++ needs local logprobs, which llama.cpp exposes.
- No API.

**Benchmarks.**
- BFCL-MT renames over its Python classes and expected states (Qwen3-4B at 22.1%, third-party, re-measured).
- ALFWorld string-bijection twins.
- tau2 retail: policy.md plus DB edits.
- WikiMIA: Min-K%++ beats Min-K% by 6.2-10.5 AUROC points.

**Evaluation.**
- AUROC-vs-dose curves per detector (the core figure).
- TOST equivalence.
- Prior-override rate with CIs.
- For the stretch: runs per counterexample and fresh-seed replication rate.

**Sprint sketch.**
- W1-2: AgentLM ALFWorld reproduction and BFCL twin compiler.
- W3-4: twin validation and the first gaps (week-4 demo).
- W5-7: dose ladder and detectors (C1).
- W8: ALFWorld isomorphic twins.
- W9-10: tau2 counterfactuals (C3) and the audit.
- W11: shrinking stretch.
- W12: write-up.

**Risks and mitigations.**
- tau2 co-edits are complex. Limit them to 30 tasks.
- Heavy doses saturate every detector. Use diluted low doses.
- Public models may show no gap. That is an honest null; the dose curve stands.
- Prior art (PIPE for interface reliance, Zero-CoT Probe for reasoning benchmarks). Twin's distinct contributions are the agent setting, dose-calibrated detector comparison and counterfactual policy twins.

**Sources.**
- https://arxiv.org/pdf/2310.12823
- https://arxiv.org/abs/2602.01611
- https://proceedings.iclr.cc/paper_files/paper/2025/hash/a2e3b4132ab2e0b7a21e6e75da7f91a9-Abstract-Conference.html
- https://arxiv.org/abs/2510.02386
- https://huggingface.co/datasets/ibm-research/BFCL-FC-robustness
- https://pypi.org/project/whyflaky/
- https://github.com/sierra-research/tau2-bench
