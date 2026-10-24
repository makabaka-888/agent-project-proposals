# Crosstalk — Candidate Brief (Round 2, rank 3)

> Not yet a full design doc. This is the brief a writer would expand into one, with the judges' fixes already folded in. Facts were web-checked in October 2026 and still need the usual review pass.

A full-duplex voice agent on one 6 GB laptop that knows when to talk, when to listen and when to stop. It is a scheduled cascade: streaming ASR with an end-of-utterance token, fused acoustic and semantic end-of-turn, a 4B model that thinks while you are still talking, and chunked TTS. Every millisecond is attributed to a stage, and turn-taking is scored on Full-Duplex-Bench next to published Moshi, Freeze-Omni and Gemini Live rows. The interview demo is a live 'interrupt me' test with the latency waterfall on screen.

| | |
|---|---|
| Lens | Voice & embodied |
| Benchmark | Full-Duplex-Bench v1 (offline, published Moshi/Freeze-Omni/Gemini rows per subset) and v1.5 (overlap handling); Big Bench Audio (1,000 Qs, MIT); optional tau2 retail half-duplex read-back arm |
| Models | Parakeet-Realtime-EOU 120M (ONNX-INT8 CPU); Smart Turn v3 + Silero VAD (CPU); Qwen3-4B-Instruct-2507 / Qwen3.5-4B Q4_K_M (~3 GB); Qwen3-0.6B LoRA EOT head; Kokoro-82M CPU; baselines LiveKit turn-detector (Qwen2.5-0.5B), Moshi-MLX q4 and LFM2.5-Audio-1.5B |
| Hardware fit | PC: 4B LLM on GPU (~4 GB with short KV); ASR/VAD/turn/TTS on CPU. Mac: same cascade via MLX plus Moshi-MLX baseline. EOT LoRA trains in hours |
| Best for roles | Voice AI engineer (Sierra, Decagon, LiveKit, Pipecat, ElevenLabs, OpenAI realtime), ML inference / latency engineer, applied scientist speech & dialogue |
| Relation to existing docs | No existing doc or round-1 idea covers realtime speech. Its speculation is like Ghostrun's, but the user's own speech supplies real slack to hide compute in. It absorbs ReadBack's confidence-gated entity read-back as a stretch arm, on half-duplex TTS/ASR wrappers around text tau2, reusing BlackBox's tau2 familiarity. | 
## Judge fixes folded in

- ASR via the community ONNX-INT8 export on CPU on both machines; NeMo in WSL2 only for a parity check
- Build the FDB replayer on a simulated clock; calibrate by reproducing one published Moshi/Freeze-Omni row within CI before any claim
- Take FDB rows per subset (Candor vs synthetic) from the official repo; sources disagree (Moshi pause TOR 0.980 vs 0.983)
- Baseline the open LiveKit turn-detector and Smart Turn v3 alone, plus cite SoulX-Duplug (2603.14877); the new LoRA head must beat both
- Cut to 3 core claims; lead with think-while-listening; backchannel classifier is stretch
- Default to Qwen3-4B-Instruct-2507 for re-drafts if Qwen3.5 hybrid prefix caching is costly
- Merge ReadBack: oracle-transcript swap plus read-back policy on 20-114 retail tasks, half-duplex only

## Brief

**Thesis.** A carefully scheduled cascade on one consumer GPU can match end-to-end speech models on turn-taking while keeping a text LLM's reasoning. Claims:
- **C1.** Per-stage p50/p95 latency budgets on both machines, plus FDB v1 turn-taking latency and takeover rate (TOR), placed next to the published rows from the same subset. One cited source gives Moshi 0.265 s, Freeze-Omni 0.953 s and Gemini Live 1.301 s turn-taking latency.
- **C2.** Fusing the Smart Turn v3 acoustic score with a LoRA semantic end-of-turn head lowers FDB v1 pause TOR against either signal alone and against LiveKit's turn-detector, at equal median latency, with a pre-registered TOR delta.
- **C3 (headline).** Think-while-listening raises Big Bench Audio accuracy under a fixed response-start budget (audio within ~1 s of end of speech) against think-after-turn. The model writes unspoken notes and a speculative draft over ASR partials, then validates or regenerates.
- **Stretch.** A backchannel-vs-interruption classifier on FDB v1.5, and ReadBack's confidence-gated entity read-back on tau2 retail through half-duplex wrappers.

**Agent loop.**
1. A replayer feeds 20 ms frames.
2. Silero VAD plus Parakeet-EOU stream partials and <EOU>.
3. A listener thread updates fused P(end-of-turn) and appends notes and a draft.
4. When the threshold is passed, the draft is validated against the final transcript or regenerated.
5. TTS streams sentence chunks.
6. During playback, overlapping speech is classified. An interruption stops TTS, truncates the context to the words actually played (word timestamps) and replans.
7. Every event is span-traced.

**Models and memory.**
- nvidia/parakeet_realtime_eou_120m-v1, ONNX INT8 on CPU, ~0.3-0.5 GB.
- Smart Turn v3 (8 MB, ~12 ms on CPU, claimed) and Silero VAD, both on CPU.
- Qwen3-4B-Instruct-2507 or Qwen3.5-4B Q4_K_M, ~3 GB plus short KV, on the GPU.
- Qwen3-0.6B LoRA head, ~2-3 GB to train. Train on non-Candor dialogue to avoid contamination.
- Kokoro-82M on CPU.
- Baselines: LiveKit turn-detector (Qwen2.5-0.5B fine-tune, LiveKit Model License, 50-160 ms on CPU), Moshi-MLX q4 on the Mac, LFM2.5-Audio-1.5B as GGUF.

**Benchmarks.**
- FDB v1: 727 samples, offline; only interruption quality uses a GPT-4o judge, so a labelled local judge is swapped in.
- FDB v1.5: open code.
- Big Bench Audio: categorical answers, parsed without a judge (a deviation from the AA method, noted).
- Optional API: one realtime model on a 100-question BBA subset, under $20.

**Evaluation.**
- GPU-vs-CPU placement grid as an experiment in its own right.
- Threshold sweeps.
- Arms: acoustic-only, semantic-only, fused and LiveKit.
- Draft modes: think-after vs think-while.
- Paired bootstrap over samples, with the MDE from a pilot.

**Sprint sketch.**
- W1-2: ONNX ASR, replayer, and calibration against one published Moshi row.
- W3-4: cascade plus traces (C1).
- W5-6: EOT fusion (C2), then the live 'interrupt me' demo.
- W7-9: think-while-listening (C3).
- W10-11: stretch arm, either the backchannel classifier or ReadBack read-back.
- W12: write-up.

**Risks and mitigations.**
- GPU jitter between ASR and the LLM. Put ASR on CPU and measure the jitter.
- An offline replay distorts timing. Use the simulated-clock replayer with calibration.
- Novelty of the end-of-turn fusion is thin, given LiveKit, Smart Turn and SoulX-Duplug. Lead with C3.
- The ASR is English-only.

**Sources.**
- https://arxiv.org/abs/2503.04721
- https://github.com/DanielLin94144/Full-Duplex-Bench
- https://arxiv.org/abs/2507.23159
- https://huggingface.co/datasets/ArtificialAnalysis/big_bench_audio
- https://huggingface.co/nvidia/parakeet_realtime_eou_120m-v1
- https://www.daily.co/blog/announcing-smart-turn-v3-with-cpu-inference-in-just-12ms/
- https://docs.livekit.io/agents/logic/turns/turn-detector
- https://huggingface.co/LiquidAI/LFM2.5-Audio-1.5B
- https://github.com/kyutai-labs/moshi
- https://arxiv.org/abs/2510.06917
- https://arxiv.org/abs/2603.13686
