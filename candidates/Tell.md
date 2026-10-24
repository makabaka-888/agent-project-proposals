# Tell — Candidate Brief (Round 2, rank 8)

> Not yet a full design doc. This is the brief a writer would expand into one, with the judges' fixes already folded in. Facts were web-checked in October 2026 and still need the usual review pass.

A lie detector wired into the model's residual stream, tested at a Werewolf table where the game state proves who lied. A linear probe trained off-game on instructed honest/dishonest pairs (Apollo method) is pitted against black-box judges. Those judges reach only 0.55-0.58 AUROC in WOLF. Tell maps detectability across Qwen3 0.6B-8B and dense vs hybrid Qwen3.5. Then a Detective villager gets the probe as a tool.

| | |
|---|---|
| Lens | multi-agent / safety |
| Benchmark | WOLF Werewolf engine (LangGraph, open code; peer detection ROC AUC 0.55-0.58); Apollo-style instructed-pairs probe eval; Among Us sandbox as OOD prior-art comparison |
| Models | Qwen3-0.6B/1.7B bf16, Qwen3-4B nf4 (HF, PC); Qwen3-8B MLX 4-bit with hidden-state hooks (Mac); Qwen3.5-2B/4B (transformers >=5.2); Q4 GGUF for fast self-play; optional GPT-4.1-mini judge (<$5) |
| Hardware fit | Self-play via llama.cpp Q4; activation capture offline in HF (<=4B on PC) and MLX (8B on Mac); probes on CPU |
| Best for roles | AI safety / interpretability research engineer (Anthropic, OpenAI, GDM, Apollo, FAR.AI), trust & safety ML, multi-agent eval |
| Relation to existing docs | Adds white-box interpretability, which none of the 8 docs have. Ctrl-Z defends an agent against external attackers; Tell monitors an agent's own deception toward peers. Pick it only if the mentee targets safety labs. | 
## Judge fixes folded in

- Capture 8B activations on the Mac via MLX hooks; verify hook in week 1
- Keep live monitor-in-loop (C3) at 4B HF nf4, ~200 games
- Deconfound role from falsity: wolf lies vs wolf true statements, plus villagers' sincere-but-wrong claims
- Controls: honest statements about lying, villager role-talk
- Report recall at 1% FPR alongside AUROC
- Sprint 0 reproduces an Apollo-style probe AUROC on Qwen3-4B
- Store mean-pooled fp16 per statement only
- C4 obfuscation LoRA only if C1-C3 land by week 7

## Brief

**Thesis.** As agents talk to agents, a monitor must catch an agent that misleads its peers, and black-box judges are near chance (WOLF peer detection ROC AUC 0.55-0.58, built on self-report labels). Werewolf provides verifiable ground truth: a wolf saying 'I am a villager', or claiming to have inspected someone it did not, tells a lie the game state proves. Claims:
- **C1.** A logistic-regression probe trained only off-game on instructed pairs beats a same-model black-box judge and an API judge at detecting game-state-labelled lies, at 4B and above. Measured by AUROC and recall at 1% FPR.
- **C2.** A detectability scaling curve over Qwen3 0.6B/1.7B/4B/8B. Prior work found Qwen2.5-1.5B probes at chance. Dense Qwen3 is compared with hybrid Gated-DeltaNet Qwen3.5 at matched size, which is an open question.
- **C3.** A Detective villager with a get_suspicion() probe tool raises village win rate by a pre-registered margin. This is framed as an upper bound on the value of a privileged monitor.
- **C4 (stretch).** Rejection-sampling LoRA on undetected wolf lies, then measure probe drift.

**Deconfounding.** The design separates role from falsity: wolf lies vs wolf true statements, and villagers' sincere but mistaken claims (false but not deceptive).

**Agent loop.**
1. The engine runs the night phase (kill, save, inspect), then 2 day discussion rounds, then a vote.
2. Each player returns JSON {speech, claimed_role, factual_claims[]}.
3. The engine labels each claim true or false from game state. Omission and misdirection are labelled heuristically and separately.
4. A teacher-forced HF replay captures residuals at statement tokens across layers. The layer is chosen on a dev split.
5. Probes and judges score identical statements.

**Models and memory.**
- Qwen3-0.6B bf16 (~1.2 GB) and 1.7B (~3.4 GB) in HF.
- Qwen3-4B nf4 (~2.6-3 GB) on the PC.
- Qwen3-8B MLX 4-bit (~5 GB) on the Mac with a custom hook.
- Qwen3.5-2B bf16 (~4 GB) and 4B 4-bit (~3 GB).
- Self-play uses Q4 GGUF, at roughly 200 games per night (estimate, measured in S0).

**Benchmarks.**
- WOLF: Werewolves deceive in ~31% of turns; peer precision 71-73%, recall 48%. Its confusion-matrix counts do not reproduce its headline figures, which is noted.
- Apollo: AUROC 0.96-0.999 on Llama-3.3-70B.
- Among Us: probes reach out-of-distribution AUROC above 95%.

**Evaluation.**
- Held-out games, with probe layer and threshold pre-registered.
- Bootstrap CIs over games.
- Judge-vs-probe comparison on identical statements.
- Win-rate deltas with paired seeds.

**Sprint sketch.**
- W1-2: Apollo probe reproduction, WOLF swapped to local models, MLX hook check.
- W3-4: game farm and labels.
- W5-6: C1 (week-6 demo).
- W7-8: scaling and architecture (C2).
- W9-10: Detective in the loop (C3).
- W11: optional C4.
- W12: write-up.

**Risks and mitigations.**
- Small models play badly. 4B and above carry the result; small models are curve points only.
- The probe may detect deception-related text. Run the controls above.
- Strong prior art (Apollo, Among Us). The distinct contributions are game-state labels, scaling and the monitor in the loop.
- Windows bitsandbytes and hybrid hooks are brittle. Keep the Mac path.

**Sources.**
- https://arxiv.org/abs/2512.09187v1
- https://arxiv.org/pdf/2502.03407
- https://www.apolloresearch.ai/research/deception-probes
- https://arxiv.org/html/2504.04072v3
- https://papers.nips.cc/paper_files/paper/2025/hash/105c4de1195135fae4974aa8c5e27bbf-Abstract-Conference.html
