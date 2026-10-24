# Half-Life — Candidate Brief (Round 2, rank 2)

> Not yet a full design doc. This is the brief a writer would expand into one, with the judges' fixes already folded in. Facts were web-checked in October 2026 and still need the usual review pass.

Survival analysis for agents. A seeded, contamination-proof tool gym (Ledger) labels the exact step where a small model falls into an unrecoverable state. Each model then gets a Weibull hazard curve, and a test asks whether the hazard is really constant. Popular test-time tricks (context reset, self-verify, thinking, per-step best-of-k) are priced as hazard ratios per 1k tokens. It is the agent analogue of MTBF: cheap, exact, and fun to explain.

| | |
|---|---|
| Lens | Eval science |
| Benchmark | Ledger (project gym with exact BFS dead-state labels, released with seeds); Long-Horizon-Execution (Sinha et al., public) as Sprint-0 replication; BFCL v4 multi-turn binned by step count for external validity; METR/Hamilton fits as reference only |
| Models | Qwen3 1.7B/4B Q4_K_M (PC), Qwen3-8B (Mac); Qwen3.5 2B/4B (PC), Qwen3.5-9B MLX 4-bit (Mac); Gemma 4 E4B (memory measured in S0) |
| Hardware fit | Short steps, <=8k context, models <=4B on PC and 8-9B on Mac; no training; lifelines/scipy on CPU |
| Best for roles | Applied scientist (evaluation / scaling), research engineer (test-time compute), agent reliability / observability engineer |
| Relation to existing docs | Complements BlackBox, which finds the step that broke one run, with a population-level reliability model. Compass's test-time search enters only as one intervention hazard ratio. Shares nothing with Compass's PRM. The Ledger gym can feed BlackBox cassettes. | 
## Judge fixes folded in

- Freeze the Ledger spec by week 2 with a bounded state space for exact BFS labelling
- Model format/parse failures as a competing risk from day 1
- Sinha et al. replication is a hard Sprint-0 gate
- Cite and position against 2609.29508 (discrete-time survival of multi-turn consistency) and Time-To-Inconsistency (2510.02712)
- Release Ledger with seeds

## Brief

**Thesis.** Long-horizon agent reliability is currently inferred indirectly. Ord models METR horizons as a constant hazard. Hamilton's Weibull refit finds a decreasing hazard, with shape k about 0.6 for agents and 0.37 for humans. Sinha et al. find self-conditioning on a key-value execution task. No one has directly observed time-to-first-irrecoverable-error for stateful tool-using small models with step-level state verification, or measured which interventions change the hazard's shape. Claims:
- **C1.** Weibull k with CI per model, for Qwen3 1.7B/4B/8B and Qwen3.5 2B/4B/9B. Test k=1 (constant hazard) and compare the direction with Hamilton's k of about 0.6.
- **C2.** Self-conditioning in tool agents. Splicing a wrong earlier step into the context raises the later hazard, estimated as a Cox time-varying hazard ratio.
- **C3.** Interventions as hazard ratios per 1k tokens: summarize-and-reset every m steps, verify-before-commit, thinking on/off, per-step best-of-k. The output is a 'half-life bought per token' table.
- **C4.** Per-step hazards from Ledger rank-predict the success of about 8 configs on BFCL-MT tasks binned by step count. The falsification line is pre-registered.

**Agent loop.**
1. The goal arrives, optionally with the full plan to isolate execution.
2. The agent calls Ledger tools (inventory/bank ledger and file operations, horizon H=1..200, knobs for distractor tools, observation noise and dependency depth).
3. After each call, the environment diffs the state against the reachable-goal set from the solver.
4. The first transition into a dead state is the event time.
5. Error-then-recovery transitions are logged for multi-state models, and parse failures are logged as a competing risk.
6. Intervention wrappers sit around the loop.

**Models and memory.**
- Q4_K_M sizes: Qwen3-1.7B ~1.1 GB, Qwen3-4B ~2.5 GB, Qwen3.5-2B <2 GB, Qwen3.5-4B ~3 GB, all on the PC.
- Qwen3-8B (~5 GB) and Qwen3.5-9B MLX 4-bit (~6 GB) run on the Mac.
- Gemma 4 E4B memory is measured in S0.
- Contexts stay under 8k (the reset arm bounds them), so the PC runs thousands of episodes per night.

**Benchmarks and baselines.**
- Long-Horizon-Execution (HF arvindh75/Long-Horizon-Execution, public code): Qwen3-32B falls below 50% within 15 turns, and Qwen3-4B fails to reach near-perfect first-step accuracy. Exact small-model horizons are only in figures, so they are read or measured in S0.
- BFCL v4 multi-turn: Qwen3-4B-Instruct-2507 at 22.1% is a third-party figure and is re-measured.
- METR/Hamilton fits are reference only and not run.

**Evaluation.**
- Kaplan-Meier curves, a Weibull AFT model, and Cox models with time-varying covariates.
- Competing risks (format vs reasoning errors) and a multi-state recovery model.
- The MDE is stated as a minimum detectable hazard ratio from a variance pilot.
- Ablations: plan given vs not; distractor tools; noise; architecture (dense vs hybrid).

**Sprint sketch.**
- W1-2: Sinha replication and Ledger spec freeze.
- W3-4: Ledger, solver and the first KM curves. The week-4 demo shows curves plus k for 3 models.
- W5-6: full ladder, then C1.
- W7-8: counterfactual injection (C2).
- W9-10: intervention table (C3).
- W11: BFCL external validity (C4).
- W12: write-up.

**Risks and mitigations.**
- Parse failures dominate small models. Handle them as a competing risk.
- A BFS-tractable state space constrains the design. Bound it explicitly.
- C4 may fail. Report it honestly; C1-C3 stand alone.
- Novelty against 2609.29508 (delayed-gratification termination, 84,540 trajectories) and Sinha et al.: Half-Life measures first irrecoverable error in stateful tool use, competing risks and intervention hazard ratios, not termination timing.

**Sources.**
- https://arxiv.org/html/2509.09677v3
- https://huggingface.co/datasets/arvindh75/Long-Horizon-Execution
- https://www.tobyord.com/writing/hazard-rates-for-ai-agents-decline
- https://gushamilton.github.io/lab/2026/01/23/petos-paradox-ai-agents/
- https://arxiv.org/pdf/2609.29508
- https://arxiv.org/html/2510.02712v1
- https://arxiv.org/pdf/2509.26553
