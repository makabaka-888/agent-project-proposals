# Crosstalk — Design Document

**A Full-Duplex Voice Agent on One 6 GB Laptop: It Thinks While You Talk, Fuses Acoustic and Semantic End-of-Turn, and Attributes Every Millisecond to a Stage**

| | |
|---|---|
| Status | Proposal |
| Target duration | ~10–12 weeks (19 mentoring hours / 15 sessions + independent work) |
| Platforms | Windows 11 laptop (16 GB RAM, NVIDIA RTX 3060-class, **6 GB VRAM default**, 8 GB if confirmed; WSL2 for the official scorer) and Apple Silicon Mac (16 GB unified, no CUDA), one repository |
| Deployment | 100% local. No rented servers, no paid GPU cloud. Commercial APIs only as optional comparison baselines (one realtime model on 100 Big Bench Audio items), capped at $20 total. Free Colab/Kaggle GPUs optional, for the LoRA head only |
| Core stack | llama.cpp `llama-server` (one pinned build) · Qwen3-4B-Instruct-2507 / Qwen3.5-4B GGUF (Apache-2.0) · NVIDIA Parakeet-Realtime-EOU-120M (community ONNX INT8 export) · Silero VAD · Smart Turn v3.2 · Qwen3-0.6B + LoRA end-of-turn head · Kokoro-82M · ONNX Runtime · MLX (Moshi baseline) · Full-Duplex-Bench v1/v1.5 · eot-bench · Big Bench Audio · OpenTelemetry spans (05's adapter) · Parquet · statsmodels |

---

## 1. The Product Thesis

Voice agents fail on timing as often as on content: cut a caller off mid-thought and the call derails; leave 1.5 s of dead air and it feels broken. End-to-end speech models (Moshi, PersonaPlex, GPT-Realtime) hear and speak at once but lose text-LLM reasoning: on Big Bench Audio (BBA), GPT-4o scored 92% on text, GPT-4o Realtime Preview 66% speech-to-speech (Artificial Analysis, December 2024); on τ-Voice, voice agents complete 31–51% of tasks (clean audio) vs 85% for GPT-5 on text. A cascade (ASR → LLM → TTS) keeps the reasoning, but its slowest stage, the LLM, usually waits for the user to stop.

**Crosstalk treats the user's speaking time as compute budget.** A scheduled cascade on one consumer GPU listens with streaming ASR and decides end-of-turn (EOT) from fused acoustic and semantic evidence. While the user talks, it writes unspoken notes and a speculative draft, validated or regenerated at turn end. Round 1's Ghostrun (IDEATION.md) failed because tool time was negligible next to LLM time; here the slack is real: a BBA question lasts 4.5–60.2 s of otherwise idle GPU. Three measured claims:

1. **Think-while-listening raises accuracy under a fixed response-start budget.** On BBA (800 confirmatory questions), with first audio ≤ 1.0 s after the user stops and no canned fillers, notes plus a speculative draft over ASR partials (A3) beat think-after-turn (A0) at α = 0.05. Power targets a minimum detectable effect (MDE) fixed after the pilot (planned ≈ 5 pp). Other arms (STITCH-style A1, unbounded ceiling C): §7.5.
2. **Fused EOT beats each signal alone and LiveKit's open detector.** Fusing Smart Turn v3.2, a LoRA semantic head, silence and Parakeet's end-of-utterance token `<EOU>` lowers the false-cutoff rate at a 300 ms budget on eot-bench English. Baselines, all cross-fitted: Smart Turn alone, the head alone, LiveKit v1-mini (published in-sample rows: context only, §5.3). Full-Duplex-Bench (FDB) v1 pause takeover at matched latency confirms it (§7.2).
3. **Every millisecond is attributed.** In wall mode, each sample's critical-path spans sum to its end-to-end latency measured independently at the playback device (median residual ≤ 5%), with per-stage p50/p95 on both machines. FDB v1 takeover rate (TOR) and latency sit per subset beside published Moshi, Freeze-Omni, Gemini Live and GPT-Realtime rows, only after S0 reproduces Moshi's row through the same replayer and scorer (§7.7, §10).

**Stretch:** an FDB v1.5 backchannel-vs-interruption classifier (§7.6); ReadBack, confidence-gated entity read-back on half-duplex voiced τ²-bench retail (§7.8).

**Why it is fun.** The interview demo is "interrupt me": the mentee talks over the agent, it stops within a measured stop latency, and a live waterfall shows every millisecond; a draft overlay shows the answer ready before the speaker finishes.

## 2. Goals & Non-Goals

### Goals
- **Cascade runtime** (§6–7) on both machines under `portable`: causal replayer on a simulated or wall clock; VAD, streaming ASR, fused EOT, responder, chunked TTS; barge-in truncating context to words actually played; span traces.
- **Benchmarks wired** (§5): FDB v1, v1.5 timing, eot-bench English, BBA.
- **Delivered:** claims 1–3 measured; LoRA EOT head released (weights, script, data card); live demo.

### Non-Goals

| Out of scope | Owner |
|---|---|
| Ranking drafts with a process reward model or search | 01 Compass |
| Distilling the 4B responder into a smaller model | 02 Shellwright (owns the distillation trainer) |
| Bit-exact replay, nondeterminism (Crosstalk reuses 05's span adapter, τ² v1.0.1 pin, DB-only retail scoring, A/A power method) | 05 BlackBox |
| Spoken prompt injection; sandboxing τ² tool calls | 06 Ctrl-Z |
| Validating the τ² user simulator's realism | 08 SimCrowd |
| Echo cancellation (headset demo), diarization, non-English ASR, telephony, agent backchannels, end-to-end speech training, FDB v2 (WebRTC + LLM judge) and v3 (disfluent tool use) | — |

## 3. Alignment With Prior Experience

| Prior experience | Crosstalk module |
|---|---|
| **Changsheng**: backtests that must not see the future (look-ahead bias); AutoML hyper-parameter agent | Causal replayer: no stage reads a frame after `clock.now`; cross-fitted EOT policy sweeps |
| **CAS**: binary-encoding retrieval; 20+ configs on Precision@K / latency / storage | Placement grid, accuracy–latency frontier; INT8 ASR, q4 vs q8 Moshi |
| **DiDi**: backend modules; **A/B testing** | Realtime scheduler with stage budgets; paired McNemar, pre-registered MDE |
| **Sina Weibo**: SQL ETL, standardized metrics, alerting | Spans → Parquet → per-stage p95 budget table; a breach is the alert |
| **College Application Agent**: LangGraph Writer–Critic; FastAPI | Draft-then-validate responder; FastAPI WebSocket demo |
| **Music Genre**: audio features, PCA, K-means | Acoustic EOT features; backchannel classifier (stretch) |

**Interview one-liner:** *"At Changsheng I learned a backtest is honest only if it can't see the future. Crosstalk's replayer applies that to speech: every stage sees only audio that has arrived. At CAS I measured an accuracy–latency frontier; here it is a voice agent's, spending the user's speaking time on thinking."*

## 4. Stack Choices — and Why

### 4.1 Models

| Role | Model | Size / format | Runs on | License |
|---|---|---|---|---|
| Responder + listener (default) | Qwen3-4B-Instruct-2507 ("2507") | Q4_K_M 2.50 GB; KV 144 KiB/token f16 (36 layers × 8 KV heads × 128) | GPU | Apache-2.0 |
| Responder arms | Qwen3.5-4B; Qwen3.5-9B (Mac) | Q4_K_M 2.74 / 5.68 GB; KV 32 KiB/token + Gated DeltaNet state | GPU | Apache-2.0 |
| Streaming ASR + `<EOU>` | `nvidia/parakeet_realtime_eou_120m-v1`: cache-aware FastConformer RNN-Transducer (RNNT), English, no punctuation; `soniqo` ONNX export (INT8 encoder, 320 ms chunks) | ~153 MB; ~420 MB RSS (exporter, macOS) | CPU | NVIDIA Open Model License (export tagged CC-BY-4.0) |
| VAD | Silero VAD | ~2 MB | CPU | MIT |
| Acoustic EOT | Smart Turn v3.2: Whisper Tiny encoder + linear head, ~8M; ≤ 8 s of 16 kHz audio | 8 MB int8 / 32 MB fp32 | CPU | BSD-2-Clause |
| Semantic EOT | Qwen3-0.6B + LoRA (trained here) | Q8_0 0.64 GB merged; KV 112 KiB/token | CPU | Apache-2.0 base |
| TTS | Kokoro-82M v1.0 (`kokoro` ≥ 0.9.2; English per-token `start_ts`/`end_ts`) | 82M | CPU | Apache-2.0 |
| Full-duplex baseline | Moshi, `kyutai/moshiko-mlx-q8` / `-q4` | 8.17 / 4.81 GB + 0.39 GB Mimi | Mac (MLX) | CC-BY 4.0 weights, MIT code |
| Native-audio baseline | LFM2.5-Audio-1.5B GGUF, `llama-liquid-audio-cli` (macOS / Ubuntu runners only; audio input experimental) | Q8_0 ≈ 1.8 GB total | Mac | LFM Open License v1.0 |
| EOT baselines | LiveKit Turn Detector v1-mini (audio); deprecated text `MultilingualModel` (Qwen2.5-0.5B, 396 MB) | — | CPU, LiveKit packages only | LiveKit Model License |

**Why 2507 by default.** Drafts are often discarded, a free KV truncation under pure attention. Qwen3.5's recurrent state rolls back only to a context checkpoint; llama-server's `-cms/--checkpoint-min-step` defaults to 8192 tokens (0 = no minimum; master README, read 2026-10-23; 05 §4.2), above Crosstalk's < 4k contexts, so at defaults each discarded draft re-prefills. S1's rollback probe runs Qwen3.5-4B at the pinned build's defaults and at `-cms 0`; 2507 stays default only if tuned Qwen3.5-4B still loses. Qwen3.5 thinks by default, so all Qwen3.5 calls (arms, judge, simulator) and the Qwen3-1.7B fallback set `enable_thinking=false`, recorded in run hashes; all arms prompt notes as `N:` lines.

**Card numbers are not anchors.** Parakeet-EOU's EOU latency (p50 160 / p90 280 / p95 320 ms, TTS audio plus 3 s of silence) and 9.30% average WER are NeMo figures at 160 ms streaming. The ONNX export streams 320 ms chunks, so the week-2 gate measures its own; S1 checks NeMo parity in WSL2, re-exporting at 160 ms if the export costs > 100 ms of EOT latency. Smart Turn v3's 12.6 ms (AWS c7a.2xlarge) and LiveKit's 50–160 ms (text model) are vendor CPU figures, re-measured here.

### 4.2 Runtime
- **LLM:** one pinned `llama-server` build (`win-cuda-12.4-x64`, `macos-arm64`; `--jinja` for judge and simulator chat calls), `--parallel 1`, 4k context, prompt cache on. One slot holds the responder's append-only log on raw `/completion` (§7.3); no co-batching, so 05's batch nondeterminism does not arise.
- **ONNX Runtime** for Parakeet, Silero, Smart Turn: CPU execution provider (EP), threads pinned per stage. PyTorch `kokoro` gives token timestamps; `onnx-community/Kokoro-82M-v1.0-ONNX-timestamped` is a measured speed option.
- **MLX:** `moshi_mlx` (Python 3.12); its stock CLI takes live audio, so S0 writes a file-driven 80 ms frame loop.
- **NeMo in WSL2** (Ubuntu, CUDA): only the unchanged FDB scorer, ASR parity and the optional re-export, never while the cascade holds the GPU.

### 4.3 Libraries

| Concern | Choice | Why |
|---|---|---|
| Orchestration | Own `asyncio` scheduler, discrete-event clock | Pipecat (BSD-2-Clause) and LiveKit Agents run on wall-clock time; owning the loop makes replays causal |
| LiveKit baselines | `livekit-agents` (pinned; v1-mini ships in 1.6.1), eot-bench's adapters | Licence: use "only with LiveKit Agents", outputs never used to develop other models |
| Benchmarks / tracing | FDB, eot-bench at pinned commits, BBA at a pinned revision; OTel spans via 05's adapter → Parquet, Perfetto waterfalls | Scorer code is part of the measurement; one span model across siblings |

Also PEFT + transformers, statsmodels, soundfile, soxr, sounddevice, FastAPI + WebSocket and `uv`; same commands on both OSes.

**Prior art.** SHANKS (2510.06917) reasons over speech chunks while the user talks (37.1% higher interruption accuracy; 56.9% of tool calls done before the turn ends). STITCH (2507.15375) interleaves unspoken reasoning with spoken chunks (+15% on math at equal latency). SoulX-Duplug (2603.14877, open) is a Qwen3-0.6B streaming state predictor: FDB v1 pause TOR 0.352 (subset mean), turn-taking TOR 0.933, 0.511 s. LiveKit Turn Detector v1 (June 2026, cloud) already fuses audio-semantic and acoustic branches. Crosstalk adds a budgeted BBA test on a laptop cascade, a cross-fitted open fusion and per-stage attribution; fusion novelty is thin (§11).

## 5. Benchmarks & Datasets

### 5.1 Full-Duplex-Bench v1 (turn-taking; official scorer)

Tasks (TOR direction per table below; latency ↓): pause handling, synthetic 137 / Candor (recorded natural conversations) 216; backchannel, ICC (the corpus behind FDB's backchannel set) 55, plus frequency ↑ and Jensen–Shannon divergence (JSD) ↓; smooth turn-taking, Candor 119; user interruption, synthetic 200, plus judge 0–5 ↑.

**Scoring.** All 727 samples are scored time-synchronously on an `output.wav` aligned to `input.wav`. A takeover is ≥ 1 s of model speech, or > 3 ASR chunks, anywhere in the output; latency counts only when TOR = 1, negatives clipped to 0. Transcripts: `nvidia/parakeet-tdt-0.6b-v2` via NeMo `.cuda()`, hence WSL2. The paper names GPT-4o as interruption judge; `eval_user_interruption.py` pins `gpt-4-turbo` (seed 0). Crosstalk applies that rubric with a local Qwen3.5-9B calibrated on 60 mentee-labelled responses (weighted κ, 95% CI); an official-judge parity run is optional within the API cap. S0 logs scorer quirks. Data: the repo's Google Drive link; Candor and ICC are CC BY-NC 4.0 (research use, never redistributed); synthetic and v1.5 sets, MIT.

Published rows (repo README, read 2026-10-23, re-read in S0; only Moshi is re-run):

| Model | Pause TOR ↓ synth / Candor | Backchannel TOR ↓ | Smooth TOR ↑ | Smooth latency (s) | Interrupt TOR ↑ | Interrupt latency (s) |
|---|---|---|---|---|---|---|
| **Moshi** | 0.985 / 0.980 | 1.000 | 0.941 | 0.265 | 1.000 | 0.257 |
| Freeze-Omni | 0.642 / 0.481 | 0.636 | 0.336 | 0.953 | 0.867 | 1.409 |
| Gemini Live 2.0 (deprecated) | 0.255 / 0.310 | 0.091 | 0.655 | 1.301 | 0.891 | 1.183 |
| GPT-Realtime | 0.010 / 0.120 | 0.000 | 1.000 | 1.470 | 0.970 | 1.500 |
| PersonaPlex | 0.584 / 0.662 | 0.327 | 0.992 | 0.070 | 1.000 | 0.400 |

**Why sources disagree.** SoulX-Duplug's Moshi pause TOR 0.983 is the unweighted mean of 0.985 and 0.980 (likewise its Freeze-Omni 0.562 and Gemini 0.283). Crosstalk reports each subset plus that mean, labelled. Backchannel frequency is 0 by design (no agent backchannels, §2); TOR is still scored.

### 5.2 Full-Duplex-Bench v1.5 (overlap handling)

Four scenarios from fixed mixing recipes (attenuation, filtering, echoes): user interruption (200), user backchannel (99), talking to others (100), background speech (100). Stop latency (overlap onset → model stop) and response latency (overlap end → next model speech) use Silero VAD, no judge, so timing is core in all four (§7.6). GPT-4o assigns behaviour labels (Respond / Resume / Uncertain / Unknown) over Parakeet-TDT transcripts (local judge for the stretch arm). Published user-interruption stop latency (Table 2): GPT-4o Realtime 0.23 s, Moshi 1.16 s, Freeze-Omni 1.42 s, Gemini 2.20 s.

### 5.3 eot-bench (end-of-turn; claim 2)

LiveKit's open suite (`livekit/eot-bench`, data `livekit/eot-bench-data`) holds real human-to-agent user turns; every pause ≥ 100 ms is annotated; the last is the true end of turn. English validation: 400 turns (5,454 across 14 languages), each with ≥ 1 annotated pause. Metrics: false-cutoff rate over mid-turn pauses at 300 / 600 ms mean-latency budgets and latency at 5% / 10% cutoff, swept over threshold, action delay and timeout. Latency is endpointing delay, not compute ("conversational dead air"); §7.2 charges ASR lag. Licences recorded: Apache-2.0 (repo), CC-BY-4.0 (HF card). Published English false-cutoff rows (LiveKit-run, vendor-reported):

| Detector | @ 300 ms | @ 600 ms |
|---|---|---|
| LiveKit v1 (cloud only; context) | 9.9% | 4.5% |
| **LiveKit v1-mini** | 27.8% | 12.1% |
| **Smart Turn v3.2** | 35.2% | 14.8% |
| VAD baseline | 55.6% | 21.7% |

eot-bench picks each detector's best policy on the evaluated set itself; Crosstalk reports that number for comparability and, for every claim arm, a two-fold cross-fitted one (policy chosen on half the turns, scored on the other).

### 5.4 Big Bench Audio (claim 1)

`ArtificialAnalysis/big_bench_audio` (MIT): 1,000 Big-Bench Hard (BBH) questions, 250 each of formal fallacies (valid / invalid), navigate (Yes / No), object counting (integer) and web of lies (Yes / No), read verbatim with ". Answer the question" appended by 23 TTS voices (MP3, 4.5–60.2 s). Artificial Analysis has Claude 3.5 Sonnet judge Whisper transcripts over three runs. Crosstalk parses the answer from the text sent to TTS, no judge; a hand-labelled 100-item check transcribes the spoken audio for parse agreement.

**Validity** (pre-registered expectation): navigate and web of lies are incrementally computable, so should gain most; object counting tracks items as they arrive but names what to count last (partial gain); formal fallacies needs every premise (least gain). The primary test pools the 800 items; per-category results are descriptive (95% CIs; MDE ≈ 11 pp at 200). Three categories are binary (chance ≈ 50%), hence the §9 floor gate. BBH has been public since 2022: contamination biases A3 − A0 toward zero (a memorized item needs no reasoning), conservative for claim 1, but inflates absolute accuracy (context only).

### 5.5 S0 targets and project-built data
- **S0 targets:** §10.
- **EOT training data:** LoRA head trains on English rows of `pipecat-ai/smart-turn-data-v3.2-train` (270,946 rows, 23 languages, 41.4 GB, CC BY 4.0, `endpoint_bool` labels, human and synthetic; English share measured in S0). Smart Turn v3.2 trained on that set, so fusion and calibration fit only English rows of its listed test set, `pipecat-ai/smart-turn-data-v3.2-test` (31,527 rows, 23 languages, 4.84 GB, CC BY 4.0; English count measured in S0), unseen by the head. S2 reports Smart Turn's AUC and expected calibration error (ECE) on train vs test rows, exposing the in-sample gap. Crosstalk's Parakeet-EOU transcribes both, so training text matches inference. No Candor, ICC, eot-bench or BBA audio enters training.

## 6. Architecture

```
  ALL LOCAL: Windows + CUDA laptop (portable default), Apple Silicon Mac, one repo

  INPUT    FDB v1/v1.5 wav | BBA mp3 (+3 s silence) | eot-bench turns | live mic (headset)
             |
  REPLAYER 20 ms frames; clock = sim (discrete-event, causal) | wall (1x real time)
             |   AudioSource.read(t) refuses t > clock.now
  CPU  +-----+--------------------+----------------------------+
       Silero VAD (32 ms)       Parakeet-EOU ONNX INT8         Smart Turn v3.2 int8
       speech on/off            append-only words + <EOU>      P_ac(end), on silence
                                     |                             |
                       semantic head: Qwen3-0.6B+LoRA              |
                       P_sem(end | words, last agent msg)          |
                                     '---------> FUSION <---------'
                                     P_end = logistic(logit P_ac, logit P_sem,
                                       log silence, EOU) -> policy(theta, delay, timeout)
             |
  GPU  RESPONDER  llama-server, 2507 Q4_K_M, 1 slot, 4k, prompt cache on
       append-only log:  U: words | N: notes | ... | [D: draft (tail, discardable)]
       listening: notes and drafts when GPU idle    on P_end >= theta: validate | regenerate
             |
  CPU  TTS  Kokoro-82M, clause chunks, token timestamps -> playback buffer + play cursor
             |
  BARGE-IN  VAD during playback -> pause -> interruption? stop, truncate log to played
            words, replan : resume      (stretch: 4-way overlap classifier)
             |
  OUTPUT   time-synchronous output.wav (FDB) | answer text (BBA) | speaker (live)
  TRACE    every stage and queue wait = span -> Parquet, waterfall, budget table
  SCORE    FDB scorer (NeMo, WSL2) | eot-bench harness | BBA parser
  MAC      Moshi-MLX (FDB, S0) | LFM2.5-Audio (BBA) | Qwen3.5-9B (judge, sim, arm)
```

## 7. Component Design

### 7.1 Replayer and clocks
- **Sim clock:** discrete-event loop; each stage call runs for real and its measured service time is charged to the virtual clock, with the GPU and a CPU pool as queues. Skipped idle waits make a run cost compute time, not audio time. Causality is a unit test: no stage, scheduler included, reads a frame stamped after `clock.now` or oracle metadata (true end of speech, labels; scorer only).
- **Wall clock:** a 20 ms timer releases frames at 1×, so contention is real.
- **Calibration (S1 exit):** 50 FDB smooth-turn samples in both modes. Sim passes if p50 first-audio latency differs by ≤ 50 ms, p95 by ≤ 100 ms, and TOR agrees on ≥ 95% of samples; else sim serves policy sweeps only and wall numbers headline.
- **Output:** `output.wav` matches the input length, TTS audio at its virtual play time (FDB's contract). In Windows wall mode it is a WASAPI loopback capture of the output device, so the scorer's latency independently checks the spans. Moshi is frame-synchronous at 12.5 Hz: an offline frame loop reproduces its real-time behaviour at any machine speed, so it calibrates the replayer → scorer chain.

### 7.2 Listening and end-of-turn
- **VAD / ASR:** consumers buffer 20 ms frames. Silero: 32 ms (512-sample) windows, hysteresis. Greedy RNNT over 320 ms chunks with cache carry-over; cache-aware streaming commits tokens, so the log is append-only (revisions counted). Per-token confidence (joint softmax max) feeds the read-back stretch; `<EOU>` is an event.
- **Smart Turn v3.2** runs every 100 ms of VAD silence over the turn's last ≤ 8 s.
- **Semantic head:** last agent message (≤ 64 tokens) and user transcript so far (≤ 128) in Qwen's chat template → P(`<|im_end|>` next), as in LiveKit's text detector. LoRA rank 16 on attention projections, bf16 on the Ampere 3060. Positives: complete clips; negatives: incomplete clips plus word-boundary truncations of complete ones; a 5% dev split sets early stopping. Merged to Q8_0, run on CPU via llama.cpp.
- **Fusion:** logistic regression on [logit P_ac, logit P_sem, log silence ms, `<EOU>` seen, words since `<EOU>`], fitted per §5.5. **Policy:** every 100 ms of silence, fire if P_end ≥ θ and silence ≥ delay, or at the timeout; for the claim, θ, delay and timeout are cross-fitted by turn (two folds).
- **ASR lag on eot-bench:** the adapter releases each feature at its chunk's end plus measured compute (`causal`, primary). Because eot-bench latency excludes compute, a `compute-free` variant matching the vendor rows is also reported. The oracle-`words` arm bounds ASR's cost.
- **Arms:** the §9 claim-2 ladder; LiveKit models join FDB only if wrapped as a LiveKit Agents session (planned tier).

### 7.3 Responder: append-only log, notes, drafts
- **Log:** `[system] U: <words> N: <note> U: <words> … [D: <draft>]` on raw `/completion`, `cache_prompt: true`, in a hand-written Qwen template that opens one assistant turn (Qwen3.5: plus the empty think block `enable_thinking=false` renders). Lines append as plain text; each call adds one suffix (`N:`, `D:` or `Final answer:`) and stops at a newline. Words and notes only append, so the cache reuses the whole prefix (a unit test checks each prompt extends the previous log byte for byte). The draft sits at the tail; a new word truncates it, keeping the notes. A note cut off by a higher-priority call stays as written, so cancellation never forces a rollback.
- **Notes:** ≤ 48 tokens per call, ≤ 512 in total, prompted as working memory ("track state, no answer yet"), written when the GPU is idle after ≥ 4 new words. **Draft:** ≤ 96 tokens, after each note while P_end ≥ 0.3, or after any `<EOU>`.
- **Validation** (rule-based, no LLM call): if no content word (outside a fixed filler list) arrived after the draft, speak it, else regenerate from the log; the validate rate is reported. `commit` freezes the validated or regenerated answer text and hands its first clause to TTS.

### 7.4 The agent loop

| Element | Definition |
|---|---|
| Observation | Events: `vad(on/off)`, `asr_word(w, conf)`, `eou`, `p_ac`, `p_sem`, `p_end`, `tts_ready(chunk, token_ts)`, `played(cursor)`, `clock(t)` |
| Actions (scheduler) | `note`, `draft`, `discard_draft`, `commit`, `speak(chunk)`, `pause`, `resume`, `stop_truncate_replan` |
| Tools | Core: none (BBA and FDB need no external tools). Internal: `llm.complete(prompt_suffix, max_tokens, deadline)`, `tts.synth(text) → (audio, token_ts)`, `head.score(words)`. Stretch: τ² retail tools (§7.8) |
| GPU priority | commit/speak > replan > draft > note; a lower-priority decode yields within one token |
| Termination | Turn: TTS queue drains, or an interruption. Sample: input ends (FDB); BBA: the parsed answer's clause has played (A0, A2, A3) or the answer is spoken (A1, A4), 30 s cap. Live: "stop", or 10 minutes |
| Budgets | Response start B = 1.0 s, scored offline from the true end of speech t_end (the file's last speech frame), which no runtime stage sees; 0.5 / 2.0 s sensitivity; answer ≤ 160 tokens |

**Budget enforcement.** At t_off + B − τ_tts the scheduler forces the prefix "Final answer:" and speaks what follows (τ_tts: p95 Kokoro first-chunk time, measured in S1; t_off: causal VAD offset, the last speech frame VAD has seen, known once EOT fires). A late EOT cannot be rescued: first audio after t_end + B is a miss, scored wrong. No arm may use canned fillers.

### 7.5 Think-while-listening arms (claim 1)

| Arm | While the user speaks | After end of turn |
|---|---|---|
| A0 think-after | ASR only | Reason + answer in the remaining budget |
| A1 think-while-talking (STITCH-style) | ASR only | Generated first clause, then reasoning during playback; answer latency reported |
| A2 notes | Notes | Answer from the log |
| A3 notes + draft (headline) | Notes + draft | Validate or regenerate |
| A4 = A3 + A1 | Notes + draft | Plus think-while-talking |
| C ceiling | — | Text transcript, unbounded reasoning, no budget |

Model arms: Qwen3.5-4B (if the rollback probe passes) and Qwen3.5-9B (Mac) on A0 and A3, 300 items; LFM2.5-Audio-1.5B (Mac, half-duplex, speech in) is a native-audio reference on the same 800.

### 7.6 Barge-in and truncation
- **Pause** at VAD onset during playback. Stop latency is measured at the pause, which fires on any voice activity; hence it is reported beside its cost (pause rate, median pause length under v1.5 backchannel, talking to others, background speech) and a pre-registered arm pausing only after ≥ 2 ASR words. **Interruption** at ≥ 300 ms of speech or ≥ 2 ASR words: playback stops, the agent's turn is cut to tokens whose Kokoro `end_ts` ≤ the play cursor, plus `[interrupted]`, and a replan starts; a shorter overlap resumes playback.
- **Truncation accuracy** on the 200 FDB v1 interruptions: logged played words vs the scorer's `output.wav` transcript.
- **Stretch classifier** (interruption / backchannel / talking to others / background) on duration, ASR words, level vs the user's running baseline, P_ac and P_sem; trained on synthetic overlaps of non-FDB speech built with v1.5 recipes, evaluated on v1.5.

### 7.7 Tracing and latency attribution
- **Spans:** `vad_offset`, `asr_chunk`, `eou`, `eot_decide{ac,sem,fuse}`, `llm_prefill`, `llm_decode`, `tts_first_chunk`, `playback_start`, `queue_wait{stage}`, tagged with profile, machine, build and model hashes.
- **Attribution** (wall mode): end-to-end = first audible output − t_end, measured outside the tracer (sounddevice's per-callback `outputBufferDacTime`; on Windows also Silero onset in the loopback capture). Residual = end-to-end − Σ critical-path spans per sample; claim 3 needs median residual ≤ 5% (sim residuals, 0 by construction, are excluded).
- **Design budget for B = 1.0 s** (pre-registered, reported against measurement): EOT stage ≤ 300 ms wall clock, i.e. decision delay + ASR lag + head compute (eot-bench's 300 ms budget counts decision delay only, reported separately); LLM to first answer clause ≤ 400 ms; TTS first chunk ≤ 200 ms; I/O and playback ≤ 100 ms.

### 7.8 Stretch: ReadBack on τ²-bench retail (half-duplex)
τ² v1.0.1 retail stays text underneath: each message passes through Kokoro (one voice per side) and Parakeet before the other side reads it. Arms: text; voice; voice with an oracle swap of entity spans (order IDs, emails, zips), isolating entity ASR errors; voice + read-back, spelling back for confirmation entities with minimum token confidence below τ. Qwen3.5-9B simulator on the Mac at temperature 0.7 (05 §5.2), labelled "local user sim, temperature 0.7 — not leaderboard-comparable"; DB-only scoring; τ set on a 20-task pilot, then ≤ 114 tasks.

## 8. Hardware Profiles

**Rule: the weakest machine sets the defaults**: memory from the 6 GB card, speed from each machine's week-2 measurement.

| Profile | GPU | CPU |
|---|---|---|
| `portable` (default, both machines) | Responder (CUDA / Metal) | ASR, VAD, Smart Turn int8, semantic head, Kokoro |
| `accelerated` (opt-in: a measured 8 GB card, or the Mac) | Kokoro, Smart Turn fp32, head; 2507 Q8_0 arm; Qwen3.5-9B on the Mac | ASR, VAD |

**Placement.** Windows: scored cascade runs, LoRA training, WSL2 scorer (offline, after runs), placement grid. Mac: Moshi S0, LFM2.5-Audio, the 9B judge, simulator and arm, the cascade's Mac rows.

**Memory** (weights from listings; compute buffers measured in S0):

| Process | Weights | KV / state | 6 GB | 8 GB | Mac 16 GB |
|---|---|---|---|---|---|
| 2507 Q4_K_M, 1 × 4k | 2.50 GB | 576 MiB | fits (~2.9 GiB + buffers) | fits | fits |
| Qwen3.5-4B Q4_K_M, 1 × 4k | 2.74 GB | 128 MiB + recurrent state per checkpoint | fits | fits | fits |
| 2507 Q8_0, 1 × 4k (accelerated); + head 0.64 GB | 4.28 GB | 576 MiB (+ 28 MiB) | tight, measure | fits | fits |
| Moshi MLX q8 / q4, alone | 8.17 / 4.81 GB + 0.39 GB | — | — | — | fits; q8 near macOS's GPU-wired cap, measured |
| Qwen3.5-9B Q4_K_M | 5.68 GB | small at 4k | no | tight | fits |

CPU RAM (Parakeet, head, Kokoro; §4.1) is measured in S0.

**Placement grid** (its own experiment; 50 FDB smooth-turn samples, wall mode, both machines): P0 portable, P1 Kokoro on GPU, P2 head on GPU, P3 ASR on CUDA EP, P4 all on GPU (accelerated). Measures: per-stage and first-audio p50/p95, decode tokens/s under CPU load, GPU jitter (SD of decode tokens/s).

**Week-2 go/no-go** (end of session 2, both machines, before the replayer): component microbenchmarks with Parakeet, Silero and Kokoro in a CPU stress loop beside llama-server running 2507; the Mac's chip is recorded. The end-to-end row waits for the **S1 exit** (session 6; wall mode, 50 smooth-turn samples, A0, Smart Turn EOT). Pre-registered design thresholds, per machine:

| Gate | Pass | If it fails (that machine) |
|---|---|---|
| ASR | chunk compute p95 ≤ 50% of the chunk while the LLM decodes | P3, or the 160 ms re-export |
| LLM | decode ≥ 25 tokens/s; end-of-turn incremental prefill (≤ 64 new tokens on the cached log) ≤ 150 ms; 1k-token cold prefill reported only | Qwen3-1.7B Q4_K_M responder (thinking off), labelled |
| TTS | first chunk (≤ 8 words) p95 ≤ 250 ms on CPU | P1, or shorter first chunks |
| End to end (S1 exit) | first-audio p50 ≤ 1.0 s | B = 1.5 s, fixed before any BBA run |

**Compute ledger** (wall-mode plan until S0 measures durations: FDB 15 s/sample ≈ 3 h/pass; BBA 15 s + 3 s silence ≈ 5 h/1,000 items). Windows ≈ 81 h: eot-bench arms ~6 h, FDB passes ~17 h, placement grid ~3 h, BBA (pilot, 800 × A0–A4, sensitivity, Qwen3.5-4B arm) ~55 h; to be measured: A1/A4 playback (longer by design), Parakeet transcription of smart-turn English (S0), LoRA (Colab optional). Mac ≈ 23 h: Moshi S0 (335 Candor samples × 3 seeds) ~8 h, FDB pass ~3 h, grid ~2 h, model arms ~10 h. **Gate rule:** budget 150 unattended Windows and 60 Mac hours, confirmed at S0; on overrun cut, in order, (1) B sensitivity to A0/A3 at 0.5 s, (2) model arms 300 → 150 items, (3) A4, (4) Mac FDB pass to smooth turn and pause.

## 9. Evaluation Plan

| Pre-registered rule | Setting |
|---|---|
| Test | α = 0.05 two-sided, power 0.8; paired binary: McNemar, normal approximation, MDE ≈ 2.80 · √(d / n) at discordance d |
| Multiplicity | One primary per claim; Holm-adjusted secondaries, MDE at the strictest step (3.08 · √(d / n) for two) |
| Resampling | By question, turn or sample; eot-bench pause-level rates by turn |
| Pilot | 200 BBA items (50 per category, outside the 800): C and A0 right after the S1 exit; in S3, A3 on them estimates d for A3 vs A0, and the MDE is re-fixed before the confirmatory run |
| Floor gate | Proceed at 4B only if the paired 80% CI of C − A0 has lower bound > 5 pp. Else claim 1 is reported "not testable at 4B" and the 9B Mac arm runs it on 300 items (MDE 7.2 pp at d = 0.2, 8.9 pp at d = 0.3), switching model and machine together, stated as such |

| Area | Metrics | Compared against | n and MDE (eot-bench, FDB: n fixed by benchmark) |
|---|---|---|---|
| Claim 1: BBA, B = 1.0 s | Accuracy pooled (primary) and per category (descriptive, §5.4); budget misses; validate rate; answer latency (A1, A4) | A3 vs A0 (primary); A3 vs A1, A4 vs A3 (Holm); C; LFM2.5-Audio | 800: 5.4 pp at d = 0.3, 4.4 pp at d = 0.2; Holm secondaries 6.0 / 4.9 pp |
| Claim 2: eot-bench English | Pause-level false cutoffs @ 300 ms mean latency (primary), @ 600 ms; latency @ 5% / 10%; Pareto; `causal` and `compute-free` (§7.2) | Fused vs Smart Turn (primary); vs semantic, v1-mini (Holm); `<EOU>`, VAD | 400 turns; paired turn-clustered bootstrap re-running two-fold policy selection on cached scores per resample; MDE simulated from the S0 Smart Turn run |
| Claim 2: FDB v1 | Pause TOR per subset, smooth-turn median latency matched ± 50 ms | Smart Turn, semantic (Holm) | 353: 7.3 pp at d = 0.2 |
| Claim 3: latency | Per-sample residual (wall mode), per-stage p50/p95, first-audio p50/p95 (bootstrap CI) | §7.7 budget; published rows (context); Moshi-MLX | 727; 50 per placement |
| Barge-in | FDB v1 interruption TOR, latency, judge (κ on 60); v1.5 stop / response latency and pause cost on the other three scenarios (§7.6); truncation accuracy | Published rows (context); ≥ 2-word pause arm | 200 + 499 |
| Stretch | v1.5 behaviour labels (local judge); τ² retail pass^1 by arm | Text arm; oracle swap | 20 → 114 tasks |

**Ablation ladders.** Claim 1: the §7.5 arms in order. Claim 2: VAD → `<EOU>` → Smart Turn → semantic → fused → fused + `<EOU>`, plus eot-bench's oracle `words` in place of ASR. Claim 3: P0–P4; 2507 vs Qwen3.5-4B rollback cost (re-prefill tokens and ms per discarded draft, checkpoint RAM; settings per §4.1).

**Reporting rules.** FDB numbers come only from the unchanged official scorer, per subset, scorer commit on every table. Labels: published rows "published, not re-run" (except Moshi-MLX: q8/q4, not the published bf16 run); eot-bench vendor rows marked; judge columns "local judge"; BBA "parsed, not judged as Artificial Analysis does". Every table carries profile, machine, build and model hashes. Inconclusive is reported as inconclusive; nulls are published.

## 10. Milestones (19 hours / 15 sessions)

| Sprint | Sessions | Deliverable |
|---|---|---|
| **S0** | 1–3 | Repo, `uv`, pinned builds and hashes; data; ONNX ASR; **week-2 go/no-go** (session 2); replayer (sim + wall) + causality test; WSL2 scorer; Moshi frame loop; **both S0 reproductions**; durations and budget confirmed |
| **S1** | 4–6 | Cascade end to end (A0, Smart Turn EOT), spans, waterfall; NeMo parity (+ re-export if needed, §4.1); sim calibration; placement grid; rollback probe; **S1 exit** (session 6); BBA pilot (C, A0) → **floor gate**; FDB v1 rows (claim 3) |
| **S2** | 7–9 | LoRA head; fusion; eot-bench arms + LiveKit adapters; FDB pause confirmation; barge-in + truncation; **live "interrupt me" demo, barge-in + waterfall** (session 9) |
| **S3** | 10–12 | Notes, drafts, validation; A3 on the pilot → MDE; A0–A4 on 800; Mac model arms; draft overlay in the demo (session 12) |
| **S4** | 13–14 | One stretch arm (v1.5 classifier or ReadBack, chosen at session 12); results tables; release of head, harness, report |
| — | 15 | Interview sprint; write-up (§14) |

**S0 reproduction rules (pre-registered).**
- **Moshi.** `moshiko-mlx-q8` on the Mac, three seeds, Crosstalk's replayer, unchanged scorer; targets Candor smooth-turn TOR 0.941, latency 0.265 s, pause TOR 0.980. Pass: mean TOR within max(2 × seed SD, 0.03), mean latency within max(2 × seed SD, 0.08 s, one Moshi frame). Fail: document the gap and its cause (quantization, sampling, resampling, alignment); the cascade's FDB rows read "scorer chain uncalibrated" until fixed.
- **Smart Turn.** LiveKit's harness with its Smart Turn adapter must give 35.2% / 14.8% exactly (environment check), then Crosstalk's own Smart Turn + Silero adapter within ± 1 pp (pipeline check).

**Tiers.** **Core** ships regardless: claims 1–3, barge-in (FDB v1, v1.5 timing), the demo. **Planned**, cut in this order at the week-8 checkpoint (session 10) if behind: (1) LiveKit-wrapped FDB arm; (2) model arms; (3) A4; (4) LFM2.5-Audio row. **Stretch**: v1.5 classifier; ReadBack; optional API realtime row (100 BBA items; ≤ $20 incl. any judge parity run).

## 11. Risks & Mitigations

| Risk | Mitigation |
|---|---|
| GPU jitter between ASR and the LLM | ASR on CPU by default; jitter measured in the placement grid |
| Fusion novelty is thin (LiveKit v1, Smart Turn, SoulX-Duplug) | Claim 1 leads; claim 2 framed as an open, cross-fitted local measurement |
| BBA floor for a 4B; questions predictable per category | Floor gate → 9B fallback (§9); per-category results; limitation stated |
| Licences: LiveKit (LiveKit Agents only), CC BY-NC subsets | LiveKit models only inside LiveKit packages, never in training data; NC subsets never redistributed |
| A machine misses stage budgets (Mac prefill, Kokoro on CPU), or scope beyond 19 h | Per-machine week-2 gate fallbacks (§8); tiers and cut orders (§8, §10); one stretch arm |

## 12. Résumé Bullets (experience voice; numbers filled in as sprints land)

- Built **Crosstalk**, a full-duplex local voice agent (streaming Parakeet ASR, fused end-of-turn, Qwen3-4B, Kokoro TTS) on one 6 GB laptop GPU and an Apple Silicon Mac: first audio **X / Y ms** p50 / p95; per-stage spans account for all but **X%** (median) of end-to-end latency measured at the playback device.
- Measured whether **thinking while the user talks** changes Big Bench Audio accuracy under a **B** s response-start budget: **X%** vs **Y%** think-after (paired McNemar, n = **N**, pre-registered MDE **M** pp; **significant / null**), vs **Z%** STITCH-style.
- Trained a **Qwen3-0.6B LoRA end-of-turn head** fused with Smart Turn v3.2: false cut-offs at a 300 ms budget went from **X%** to **Y%** on eot-bench English (cross-fitted, turn-clustered CI), vs **Z%** for LiveKit v1-mini.
- Built a **causal simulated-clock replayer** and calibrated the replayer → scorer chain against Moshi's published Full-Duplex-Bench row (TOR **X** vs 0.941; latency **X** vs 0.265 s; pre-registered tolerance: **pass / fail**); barge-in paused playback in **X ms** p50 (false pauses on backchannels **X%**) and truncated context to played words with **X%** accuracy.

**Placement note.** These read as experience only under a real engagement: the mentoring organization (engagement record, truthful title), a real collaboration with a startup, lab or open-source org hosting the work, or an open-source project entry with public commits. Never invent an employer or dates: North American employers verify, and a fabricated entry can rescind an offer.

## 13. Why This Reads as Real Company Work

- **Voice-agent teams live on two numbers**: false cut-offs and p95 response latency. Crosstalk measures both on open benchmarks, re-runs vendor claims locally (Smart Turn, v1-mini eot-bench rows) and refuses comparisons it never ran.
- **Latency as a budget**: per-stage service-level objectives (SLOs), a critical-path residual, a placement grid and a go/no-go gate decide CPU vs GPU placement.

## 14. Interview Leverage

- **ML system design** ("design a voice agent that answers within 800 ms"): streaming ASR, endpointing policies, speculative drafting, KV reuse on an append-only log, barge-in with context truncation, CPU/GPU placement.
- **Applied LLM**: when recurrent state makes draft rollback expensive (checkpoint spacing); prompt-cache layout; q4/q8 and INT8 trade-offs.
- **Experimentation**: McNemar MDE from discordance, cross-fitted thresholds, turn-clustered bootstrap for pause-level rates, a pilot that sets n, a CI-based floor gate, Holm.
- **Research / behavioural**: SHANKS, STITCH, SoulX-Duplug as prior art; why 0.983 and 0.980 disagree; publishing nulls.
- **Drills**: *"Your agent interrupts users; fix it without adding latency."* → §7.2, §9. *"Where do the 900 ms go?"* → §7.7, §8. *"Is your offline eval faithful to real time?"* → §7.1, §10.
- **Write-up:** one post under the mentee's byline (session 15, §10), working title *"Thinking while you talk: a voice agent on a 6 GB laptop"*: hook; context; the work, failure first (a mid-pause cut-off, then fused EOT and the waterfall); what I'd do differently, the pre-declared null or fallback (§9's "not testable at 4B", or a failed sim calibration, §7.1); takeaways. Every number traces to the results table, labelled per §9.

## 15. References

- Full-Duplex-Bench v1: https://arxiv.org/abs/2503.04721 · repo and README table: https://github.com/DanielLin94144/Full-Duplex-Bench (`v1_v1.5/evaluation/eval_user_interruption.py`, `v1_v1.5/get_transcript/asr.py`) · v1.5: https://arxiv.org/abs/2507.23159 · v2: https://arxiv.org/abs/2510.07838 · v3: https://arxiv.org/abs/2604.04847
- eot-bench: https://github.com/livekit/eot-bench · https://huggingface.co/datasets/livekit/eot-bench-data · LiveKit v1: https://livekit.com/blog/solving-end-of-turn-detection · https://docs.livekit.io/agents/logic/turns/turn-detector · licence: https://github.com/livekit/agents/blob/main/MODEL_LICENSE
- Big Bench Audio: https://huggingface.co/datasets/ArtificialAnalysis/big_bench_audio · https://huggingface.co/blog/big-bench-audio-release · BBH: https://arxiv.org/abs/2210.09261
- ASR: https://huggingface.co/nvidia/parakeet_realtime_eou_120m-v1 · https://huggingface.co/soniqo/Parakeet-EOU-120M-ONNX-INT8 · https://huggingface.co/nvidia/parakeet-tdt-0.6b-v2
- Turn and VAD: https://github.com/pipecat-ai/smart-turn · https://www.daily.co/blog/announcing-smart-turn-v3-with-cpu-inference-in-just-12ms/ · https://huggingface.co/datasets/pipecat-ai/smart-turn-data-v3.2-train · https://huggingface.co/datasets/pipecat-ai/smart-turn-data-v3.2-test · https://github.com/snakers4/silero-vad
- TTS: https://huggingface.co/hexgrad/Kokoro-82M · https://github.com/hexgrad/kokoro (`kokoro/pipeline.py`, `join_timestamps`)
- Baselines: https://github.com/kyutai-labs/moshi · https://huggingface.co/LiquidAI/LFM2.5-Audio-1.5B · https://huggingface.co/LiquidAI/LFM2.5-Audio-1.5B-GGUF
- Prior art: SHANKS https://arxiv.org/abs/2510.06917 · STITCH https://arxiv.org/abs/2507.15375 · SoulX-Duplug https://arxiv.org/abs/2603.14877 · https://github.com/Soul-AILab/SoulX-Duplug · τ-Voice https://arxiv.org/abs/2603.13686
- Models: https://huggingface.co/Qwen/Qwen3-4B-Instruct-2507 · https://huggingface.co/Qwen/Qwen3.5-4B · https://huggingface.co/Qwen/Qwen3-0.6B · https://huggingface.co/unsloth (GGUF listings, read 2026-10-23) · llama-server flags: https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md
- τ²-bench (stretch): https://github.com/sierra-research/tau2-bench/releases/tag/v1.0.1
