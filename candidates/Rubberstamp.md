# Rubberstamp — Candidate Brief (Round 2, rank 4)

> Not yet a full design doc. This is the brief a writer would expand into one, with the judges' fixes already folded in. Facts were web-checked in October 2026 and still need the usual review pass.

Pick the best of N agent rollouts with an LLM judge and watch the judge get gamed, with no training at all. The judge's estimated success rises with N while the database says otherwise. A 2026 study showed this for single-answer tasks (gap 0.20 at k=1 to 0.59 at k=16) and explicitly left agents out. Rubberstamp brings it to state-verified tool trajectories, adds state-preserving narrative attacks, and tests a shadow-rollout judge: the judge plays the task first, then compares end states.

| | |
|---|---|
| Lens | Eval science |
| Benchmark | BFCL v4 multi-turn (primary, state-checked, no user sim); tau2-bench retail (~40 pre-filtered tasks, N<=8); AgentRewardBench (subsample ~300, text-only; GPT-4o a11y precision 69.8 / recall 83.1, rule-based 83.8 / 55.9) |
| Models | Policy Qwen3-4B-Instruct-2507 Q4_K_M (PC); judges Qwen3.5-9B MLX 4-bit (Mac), Qwen3-8B Q4 (PC at night with q8 KV), Qwen3.5-4B and Gemma 4 E4B small judges |
| Hardware fit | No training; rollouts on PC, judges batched on Mac; judge-size ladder fits both machines |
| Best for roles | Evaluation scientist, applied scientist post-training / reward modeling, AI engineer building eval pipelines |
| Relation to existing docs | Measures when the judges that Compass (RFT filtering), Shellwright and HarnessForge (promotion) rely on start to lie. It reuses BlackBox's BFCL/tau2 harness. Overlap with Compass's PRM-vs-judge comparison is light. | 
## Judge fixes folded in

- BFCL-MT first as the clean arm; tau2 only after the curve appears, ~40 tasks, N<=8
- Pre-filter tasks to 30-70% pilot pass rate; pre-register a >=10 pp gap MDE
- Subsample AgentRewardBench to ~300 trajectories with pre-registered truncation
- Reproduce one AgentRewardBench precision number with a local judge in Sprint 0
- Pre-register a judge-size x shadow-rollout interaction (2607.05904 found commit-first hurts a 1.7B judge)
- Cite 2609.00088: commit-first inherits the judge's own errors; measure shadow-judge failures conditioned on the judge's own rollout success
- Report narrative-attack flip rates as a standalone result so a null curve still has a story

## Brief

**Thesis.** Agent pipelines select and filter trajectories with LLM judges. 2607.05904 ('More Convincing, Not More Correct') shows two things on single-answer tasks. The judge-truth gap grows under best-of-N, from 0.20 at k=1 to 0.59 at k=16 on LiveCodeBench. Commit-first judging cuts false positives on wrong answers from 0.719 to 0.012, though it hurt a 1.7B judge. The paper explicitly does not study agent trajectories. Claims:
- **C1. Goodhart curve on trajectories.** With a local 8-9B judge selecting among N rollouts of a 4B agent, the judge's estimated success rises with N while state-verified success plateaus. Pre-registered: the gap grows by at least 10 pp from N=1 to N=16 on BFCL-MT.
- **C2. Narrative attacks.** State-preserving edits flip verdicts at a measured rate: a false 'booking confirmed' summary, a deleted crucial tool call, confident restatements. A state-diff judge's invariance to them is tested.
- **C3. Shadow-rollout judge.** The judge plays the task in a forked environment, then compares end states. It cuts false positives by at least 50% at matched tokens, but only above a judge-size threshold, which is pre-registered. Following 2609.00088, its residual errors are decomposed by whether the judge's own rollout succeeded.
- **C4. Practical rule.** The N at which judge selection starts to hurt, per judge size.

**Agent loop.**
1. The policy runs N rollouts.
2. Judge variants score each one: standard, rubric, state-diff (DB diff plus task only) and shadow-rollout.
3. The selector takes the argmax.
4. Truth comes from the benchmark's state checker.
5. Proxy and true success are logged per N, with bootstrap CIs.

**Models and memory.**
- Policy: Qwen3-4B-Instruct-2507 Q4_K_M, ~2.5 GB plus KV, on the PC.
- Judges: Qwen3.5-9B MLX 4-bit (~6-6.6 GB) on the Mac; Qwen3-8B Q4_K_M (~5 GB, q8 KV, 8k) on the PC overnight; Qwen3.5-4B (~3 GB) and Gemma 4 E4B (memory measured) for the size ladder.
- Optional GPT-4.1-mini judge on 200 trajectories, under $10.

**Benchmarks.**
- BFCL v4 multi-turn: local, cheap, state-based. Its 4B floor (22.1%, third-party) is re-measured, and only 30-70% pass-rate tasks are kept.
- tau2 retail: local user simulator on the Mac, time-shared.
- AgentRewardBench: cleaned JSON only, not the 38 GB of screenshots.

**Evaluation.**
- Proxy and true success vs N curves, with the gap and its CI.
- False-positive rate, precision and ECE per judge.
- Flip rates under each metamorphic edit.
- Paired bootstrap.
- Multiple-testing framing: best-of-N as data snooping.

**Sprint sketch.**
- W1-2: AgentRewardBench precision reproduction and BFCL pilot pass rates.
- W3-5: N=16 rollouts and the first curve (week-5 demo).
- W6-7: narrative attacks.
- W8-9: shadow-rollout judge and the size ladder.
- W10-11: tau2 retail arm.
- W12: write-up.

**Risks and mitigations.**
- Pass rates too extreme. Pre-filter tasks.
- No Goodhart effect with small judges. The attacks and the shadow judge still stand.
- tau2 user-sim noise. BFCL is the clean arm.
- Long trajectories need truncation, so the truncation rules are pre-registered.
- Throughput: tau2 with a local simulator is ~10x the stated budget, so it is capped.

**Sources.**
- https://arxiv.org/pdf/2607.05904
- https://arxiv.org/pdf/2609.00088
- https://arxiv.org/html/2504.08942v2
- https://huggingface.co/datasets/McGill-NLP/agent-reward-bench
- https://github.com/sierra-research/tau2-bench
- https://rlhfbook.com/teach/course/lec9-chap14-appb-overoptimization/
