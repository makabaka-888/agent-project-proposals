# Levelsmith — Candidate Brief (Round 2, rank 10)

> Not yet a full design doc. This is the brief a writer would expand into one, with the judges' fixes already folded in. Facts were web-checked in October 2026 and still need the usual review pass.

Do not ask the LLM for a level; ask it for a level generator. On the CPU-only PCG Benchmark, a local 4B designer writes and refines generator programs with solver playtests in the loop. It is compared with the benchmark's GA and ES at an equal number of content evaluations. The claim is amortisation: one program hits unseen control targets zero-shot, where the GA must search again for each target. Interviewers can play the levels on the spot.

| | |
|---|---|
| Lens | wild / program search |
| Benchmark | PCG Benchmark (pcg_benchmark, MIT, pure Python CPU; GA feasible 224/360 runs, ES 191, Random 32; appendix LLM baselines only in a figure, re-run) |
| Models | Qwen3.5-4B Q4_K_M designer (PC, 2.74 GB); Qwen3.5-9B Q4 (Mac, 5.68 GB); Qwen3.5-2B Q4 LLM playtester |
| Hardware fit | Evaluation pure CPU on both machines; designer LLM easily fits; no training required |
| Best for roles | Research engineer program synthesis / LLM code generation, applied scientist search & generative modelling, AI engineer at game / UGC platforms |
| Relation to existing docs | It shares HarnessForge's 'LLM evolves code under an evaluator' mechanic, but the evolving artifact is a content generator scored by public game metrics. So it is HarnessForge's cheapest controlled analogue, and it is best chosen only if HarnessForge is not on the mentee's slate. It fixes round-1 Glitchhound's missing public benchmark. | 
## Judge fixes folded in

- Cite Siper et al. and 2608.17947 (evolutionary LLM program search with continual abstraction discovery on Sokoban/Zelda/Lode Runner) and position against them
- Headline C2 (zero-shot controllability / amortisation) rather than raw quality
- Re-run appendix LLM baselines and GA/ES in weeks 1-2
- Sample-efficiency curves over >=10 seeds
- Sandbox generator code with timeout and memory cap from day 1
- Frame as LLM program search vs evolutionary search at equal budget, citing AlphaEvolve/FunSearch
- C4 'hard for AI, easy for A*' kept as a labelled internal side result

## Brief

**Thesis.** In the PCG Benchmark (FDG 2025), search over levels wins. The GA found feasible solutions in 224 of 360 runs (ES 191, Random 32), at 200 generations x 100 individuals. Appendix few-shot LLMs that emit levels directly found very few feasible Zelda levels, with exact values only in a figure. Searching over programs amortises: an LLM-written generate(seed, controls) runs again for any seed or target. Claims:
- **C1.** On Binary, Sokoban, Zelda and 2-3 more problems, the program-search agent reaches GA-level quality using at most 10% of the GA's content evaluations. Shown as equal-evaluation-budget curves over at least 10 seeds.
- **C2 (headline).** One frozen generator hits unseen control targets zero-shot, while the GA must search again for each target.
- **C3.** Solver-trace feedback (unreachable key, unpushable box) vs a scalar score only, measured in iterations to feasibility.
- **C4 (internal, labelled).** Levels solvable by the benchmark solver but failed by a Qwen3.5-2B text player. Report the solve-rate gap vs solution length.

**Agent loop.**
1. Read the problem specification (content space, control space, metrics).
2. Write the generator.
3. Run it in a sandbox for N seeds.
4. Call pcg_benchmark evaluate() for quality, diversity, controllability and solver info.
5. Reflect on failing levels, optionally with PNG renders given to a VLM.
6. Edit the program.
7. Keep a MAP-Elites island archive over diversity descriptors.
8. Freeze and score on held-out seeds and targets.

**Models and memory.**
- Qwen3.5-4B Q4_K_M: 2.74 GB, on the PC.
- Qwen3.5-9B Q4: 5.68 GB, on the Mac.
- Qwen3.5-2B Q4: 1.28 GB playtester.
- Optional bf16 LoRA on verified programs, on the Mac via MLX, since the 2B needs ~5 GB.
- No API.

**Benchmark.**
- pcg_benchmark: 14 env IDs (binary, sokoban, zelda, smb, loderunner, mdungeons, and others). Depends only on numpy and Pillow, CPU only.
- GA and ES are cheap to re-run at matched budgets.
- Prior art: 2608.17947 reports that continual abstraction discovery improves best fitness on Sokoban, Zelda, Dangerous Dave and Lode Runner. Its baseline numbers were not verified here.

**Evaluation.**
- Arms: LLM-program, GA, ES, and direct-LLM levels at matched evaluation counts.
- Feedback-type ablation.
- With vs without the archive.
- Seeds >= 10, with bootstrap CIs.
- Controllability on held-out targets.

**Sprint sketch.**
- W1-2: re-run baselines and set up the sandbox.
- W3-4: designer loop on Binary and Sokoban (week-4 playable demo).
- W5-7: C1 curves on 3-6 problems.
- W8-9: controllability (C2).
- W10: feedback ablation (C3).
- W11: C4 side result.
- W12: write-up.

**Risks and mitigations.**
- The GA may still win on raw quality. Lean on sample efficiency and amortisation.
- Generated programs may collapse diversity. Use the MAP-Elites archive.
- Close prior art (2608.17947, Siper et al.). The differentiators are a laptop-scale 4B designer, an equal-budget comparison with the benchmark's own baselines, and zero-shot controllability.
- Reads as niche. Frame it as program search plus verifiers.
- Overlap with HarnessForge. Choose one of the two.

**Sources.**
- https://arxiv.org/abs/2503.21474v2
- https://github.com/amidos2006/pcg_benchmark
- https://github.com/amidos2006/benchmark_experiments
- https://arxiv.org/pdf/2608.17947
- https://huggingface.co/unsloth/Qwen3.5-4B-GGUF
