# Paperweight — Candidate Brief (Round 2, rank 9)

> Not yet a full design doc. This is the brief a writer would expand into one, with the judges' fixes already folded in. Facts were web-checked in October 2026 and still need the usual review pass.

Make the paper pull its weight. ResearchCodeBench found that giving the paper helps frontier models, but small models gain little, and LLaMA-based models get worse. Paperweight reproduces that anomaly on a laptop, then reverses it. A 0.6B retriever injects only the equation spans tied to the masked snippet, and the coder writes its own tests from that math. Every test runs on CPU in about a second.

| | |
|---|---|
| Lens | science / research engineering |
| Benchmark | ResearchCodeBench (NeurIPS 2025 D&B; 212 snippets, 20 papers; CPU tests ~1.25 s/task; Gemini-2.5-Pro 37.3% scaled pass@1; small-model numbers only in Fig. 2); fresh post-June-2026 30-snippet internal set |
| Models | Qwen2.5-Coder-7B Q4_K_M (PC, q8 KV, 8k) / Qwen3-8B Q4 (Mac fallback); Qwen3.5-9B MLX 4-bit full-paper arm (Mac); Qwen3-Embedding-0.6B Q8 + Qwen3-Reranker-0.6B on CPU |
| Hardware fit | 7B Q4 at 8k with q8 KV just fits 6 GB; full-paper long-context arm Mac-only on a 100-snippet subset; no training on the critical path |
| Best for roles | Research engineer (labs, research tooling), AI engineer (coding agents, RAG), applied scientist LLM evaluation |
| Relation to existing docs | A natural second project that reuses Spelunker's retriever and Shellwright's sandbox, so overlap is deliberate but its unit (one paper, equation spans, math-derived tests) is new. It is not round-1 LabRat: no training runs. | 
## Judge fixes folded in

- Read Fig. 2/4 values and reproduce one 'paper hurts' delta locally before building retrieval
- Week-1 pilot on 40 snippets; if <5%, switch default to Qwen3-8B on Mac or easier papers
- Random-span control at equal token budget to separate relevance from length
- Cluster bootstrap by paper; MDE from pilot
- Full-paper arm at one seed on 100 snippets (Mac prefill time)
- q8 KV on PC
- Self-test precision/recall vs hidden tests is a diagnostic, not a claim
- Use arXiv LaTeX source over PDF

## Brief

**Thesis.** ResearchCodeBench's text states that small models gain little or nothing from the paper and that LLaMA-based models get worse. The authors suspect context dilution, and a whole paper does not fit a 6 GB model anyway. Paperweight tests whether targeted retrieval plus self-verification fix this without more parameters. Claims:
- **C1.** Reproduce the anomaly locally. A 7-9B coder scores no better with the full or truncated paper than with none, paired by snippet.
- **C2.** Targeted grounding reverses it. LaTeX source is parsed into sections, equations, algorithm boxes and hyperparameter tables. A 0.6B embedder plus reranker retrieves spans tied to the masked snippet, at most 2k tokens. At equal prompt budget, scaled pass@1 beats no-paper, full-paper and random-span arms, against a pre-registered MDE.
- **C3.** Equation-derived self-tests raise the pass rate at a fixed attempt budget. The tests check shapes, invariants (e.g. softmax sums to 1), limit cases and a naive reference implementation. Their precision and recall against the hidden tests are reported.
- **C4.** Contamination audit: a cutoff split plus a fresh set of 30 snippets from post-June-2026 papers with code, labelled internal.

**Agent loop.**
1. Read the masked file and its description.
2. Query the span index, rerank, and pack spans.
3. Draft the code.
4. Derive checks from the retrieved equations.
5. Run them on CPU in the benchmark's conda or Docker environment.
6. Repair for up to 3 rounds.
7. Submit, and log the spans for attribution by span type.

**Models and memory.**
- Qwen2.5-Coder-7B-Instruct Q4_K_M: ~4.7 GB plus q8 KV at 8k, on the PC. Its older cutoff helps C4.
- Qwen3-8B Q4 (~5 GB): general coder arm.
- Qwen3.5-9B MLX 4-bit (~6 GB): 32-64k-token full-paper arm on the Mac, with minutes of prefill per snippet.
- Qwen3-Embedding-0.6B Q8 (0.64 GB) and Qwen3-Reranker-0.6B Q6_K (~0.47 GB) on CPU.
- Optional commercial reference on 40 snippets, under $10.

**Benchmark.**
- ResearchCodeBench: scaled pass@1 with line-weighted partial credit.
- Published: Gemini-2.5-Pro-Preview 37.3%, o3-high 32.3%, o4-mini-high 30.8%.
- Open small-model values are only in figures, so they are read in S0 and reported as unknown until then.
- One pass is roughly 1M tokens, or 6-8 h per arm on the PC.

**Evaluation.**
- Arms: none, full or truncated, random-span, and retrieved paper context, each with and without self-tests.
- Cluster bootstrap by paper.
- Attribution of the gain to span types.
- Cutoff split.

**Sprint sketch.**
- W1: harness, 40-snippet pilot, figure values.
- W2-3: C1 reproduction.
- W4-6: parser, index and retrieval (C2). Week-6 demo.
- W7-8: equation tests (C3).
- W9-10: fresh set and contamination split (C4).
- W11-12: full runs and write-up.

**Risks and mitigations.**
- Floor effect. Pilot gate and an 8B fallback.
- Only 212 snippets from 20 papers, so CIs are wide. Use the cluster bootstrap.
- Self-tests may share the code's misunderstanding. Measure their precision.
- Equation parsing is brittle. Use LaTeX source.
- Windows environment. Run tests on the Mac or WSL2.

**Sources.**
- https://arxiv.org/html/2506.02314v1
- https://neurips.cc/virtual/2025/poster/121850
- https://huggingface.co/Qwen/Qwen3-Embedding-0.6B
- https://huggingface.co/paulalesius/Qwen3-Reranker-0.6B-GGUF
- https://huggingface.co/Qwen/Qwen2.5-Coder-7B-Instruct
