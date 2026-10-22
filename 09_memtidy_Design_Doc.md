# memtidy — Design Document

**Plugin Noise, Result Format and a Powered Vector Kill Rule: What It Costs a 4B Agent to Find One Note on a 6 GB Laptop GPU**

| | |
|---|---|
| Status | Proposal |
| Target duration | ~10–12 weeks (19 mentoring hours / 15 sessions + independent work) |
| Platforms | **Windows 11 laptop** (16 GB RAM, RTX 3060-class, **6 GB VRAM assumed**, 8 GB if confirmed; WSL2 for the CredData and LongMemEval scripts): all scored 4B runs. **Apple Silicon Mac** (16 GB unified, no CUDA): 9B rows, paraphrase generation, property tests, demo, and the second machine of the git sync test. One repository. |
| Deployment | 100% local. No rented servers or paid GPU cloud; at run time the only network use is one git remote shared by the two laptops (a bare repo on the LAN). Commercial APIs (Claude Code, Codex) only as an optional comparison row, capped at $20 total |
| Core stack | llama.cpp `llama-server` (one pinned build) · Qwen3.5-4B / 9B and Qwen3-4B-Instruct-2507 GGUF (Apache-2.0) · Qwen3-Embedding-0.6B · SQLite FTS5 + sqlite-vec · ripgrep · MCP Python SDK (stdio) · LongMemEval as md (code MIT) · Samsung CredData + gitleaks · qmd · Basic Memory · Hypothesis · ranx · statsmodels · 05 BlackBox cassettes |

**Terms.** **memtidy**: the mentor's planned open-source (Apache-2.0), local-first tool that lets agents find, create and share md (Markdown) files; at PM stage with no product code, facts as of commit `7708d37` (2026-10-19). **mdsieve**: the mentee's benchmark and harness (working name). **A**: the agent with `grep`, `glob` and `read` only. **A+F**: A with the noise filter applied to its tools. **B**: A plus `search_md` (noise filter + FTS5 BM25 + metadata filters) and a one-line pointer to it. **B−p**: B without the pointer. **Noise share**: vendor, plugin and cache md as a share of all md under the home directory, by file count. **Delivery**: *inline* (path, heading and a ≤ 3-line snippet per hit) or *paths* (ranked paths only). **grep@noise**: arm A's measured success at full noise. **LME-md**: LongMemEval chat histories written as one md file per session. **Needle**: a fictional fact seeded in exactly one self-written file; a **decoy** is a stale copy of it in a vendor or cache path. **G0**: the mentor's IP check before any public memtidy release (decision-log L14). **RRF**: reciprocal rank fusion. **MDE**: minimum detectable effect. **FBR**: false-block rate. **LOO**: leave-one-out.

---

## 1. The Product Thesis

An agent looking for a note its user wrote last week usually greps the home directory, and most of the md there was not written by the user. One heavy user's laptop scan (n = 1, unpublished, the mentor's) found 51,118 md files; about 370 of them (3.3% of bytes) were self-written, and 62% of installed plugin bytes were exact copies. That is motivation only.

Vendors disagree on the fix. Claude Code moved from a local vector index to agentic grep on "vibes" plus internal benchmarks (secondary reports of a 2025 interview). Cursor reports semantic search plus grep as best (+12.5% offline QA accuracy, vendor-reported). The papers split too, and every result comes from frontier models. On 116 LongMemEval questions stored as files, grep beat vector search on inline-delivery harnesses, and file-based delivery could invert or erase that advantage (*Is Grep All You Need?*, arXiv 2605.15184; direction only, its numbers unverified). On repository QA, semantic search beat a grep subagent, 65.2% vs 46.2% (arXiv 2608.01507; from a search summary, unverified against the PDF). None of them varies **noise share** or runs a laptop open-weight agent.

memtidy's 2026-10-19 direction (decision log L33; one-pager) is "one always-on index over all your md folders, with plugin and cache noise filtered out". L35 defaults to an incremental SQLite FTS5 index with path, frontmatter, scope and recency filters, and adds vectors only if they "recall ≥ 10 pp more than rg on 50 real queries" (translated). L36 turns the old scan into the index's noise filter, makes the secret scan a gate before create and before promotion to shared scopes, and reduces render to a one-line "check memtidy first" hint in native files. The one-pager says how each piece works "is still a proposal", so 09 tests them. memtidy calls no LLM: its filter, index, create path, secret gate and scope rules are the deterministic **tool side**, and the subject of every agent claim is a local open-weight agent that uses them.

**09 treats finding a note as an agent cost problem: it measures what a local 4B agent pays to find one file among tens of thousands of plugin markdown files, and which part of a search tool pays it back.**

> **Engineering thesis: where vendor markdown is most of the files, a small agent's search cost is a noise problem first and a retrieval problem second.** A deterministic noise filter plus a lexical index with metadata filters lets a 4B agent find the right file as often as its grep loop, for fewer tokens and calls; dense vectors should pay off only where lexical overlap is absent.
>
> *Pre-registered test:* rejected if, at full noise, B is not non-inferior to A on success (8-point margin) or the B/A token-ratio upper bound is ≥ 1 (§9.3).

**Baseline (measured, not a claim): grep@noise**: the profile's agent (Qwen3.5-4B on the default 6 GB card; 9B on 8 GB or the Mac) in arm A, inline, on the full-noise fixture (§5.1), dev and test. Success, total prompt tokens, prefilled tokens, tool calls and wall-clock are measured; every margin uses this local number.

**Floor gate (S1, pre-registered).** On the 100 LME-md dev questions, with the 40 dev needles reported beside them, all six must hold: (1) grep@noise success between 30% and 85%; (2) a median of ≥ 3 tool calls in A; (3) tool-call parse failures ≤ 5%; (4) B calls `search_md` in ≥ 70% of episodes; (5) a no-tools path-guessing control (A0) succeeds on ≤ 10%; (6) an oracle-path control (gold path given) answers ≥ 60% of the dev short-answer subset by substring match. Otherwise the cost claims have nothing to test (fallbacks: §8.3).

Four claims, each with a pre-registered decision rule and an MDE re-derived from the S0 variance pilot (§9.3):
1. **Headline: equal success for fewer tokens at full noise, and which part pays.** B vs A, inline, full noise, on 200 LME-md test questions (the primary stratum) × 2 seeds: gold-file success non-inferior (8-point margin) and total prompt tokens lower (geometric-mean ratio B/A). Co-primary attribution, B vs A+F: **index adds** / **filter suffices** (±15% token equivalence) / **inconclusive**. 80 seeded needles form a secondary stratum; their 40 decoy items are set aside as by-construction. Planning: success power 0.81 at discordance 0.2; tokens MDE ≈ 11%.
2. **Noise is what grep pays for.** A and B at 0%, 90% and full noise on the same questions: the A − B log-token gap must grow from 0% to full (difference-in-differences lower bound > 0; planning MDE ≈ 0.17 log points). The filter's rules are frozen on one set of marketplaces; the scored fixture uses held-out ones (§5.1).
3. **The interface decides whether a 4B uses the tool.** (a) {A, B} × {inline, paths}: the tool × delivery interaction on success (planning MDE ≈ 12.5 points) sets the reference tool's default format. (b) Uptake, B vs B−p (no pointer; memtidy's L36 hint): `search_md` call rate (McNemar, n = 280, MDE ≈ 9 points); B's failures split into zero-call / gold never retrieved / retrieved but not returned; spurious calls counted on 80 context-free items, and the pointer is adopted only if they stay ≤ 20%.
4. **Vectors earn their place only where lexical overlap is absent (a powered kill rule).** Retrieval only, no agent: file-level Recall@10 of the better dense or hybrid-RRF arm vs the better of FTS5 and rg (both chosen on dev), on keyword, paraphrase and zh→en strata, n = 210 each. Per stratum, Holm over 3: **keep** (lower bound > 0 and Δ̂ ≥ 10 points), **kill** (upper bound < 10) or **inconclusive**. memtidy's literal 50-query rule runs on the private arm (MDE ≈ 18 points).

**Why it is surprising, and fun.** The industry bet on grep, the papers disagree, and nobody has varied the haystack. Three outcomes would each be news: the filter alone captures the saving; the result format matters more than the retriever; a one-line pointer decides whether a 4B ever uses the index. The demo shows two agents and one question, with live token meters, in a home full of plugin files, replayed from cassettes.

## 2. Goals & Non-Goals

09 is the portfolio's **agent memory & context files** project (context engineering).

### Goals
- **Reproduce one published number in S0.** The co-targets are gitleaks' CredData row and LongMemEval's BM25 retrieval row (§5.4).
- **mdsieve**: a public Apache-2.0 benchmark and harness under the mentee's name. It evaluates any md-context system (an rg/grep loop, qmd, Basic Memory, the reference FTS5 tool, memtidy once it exists) and does not depend on G0.
- **A reference find/create tool** implementing memtidy's current spec (L33–L36 at `7708d37`) through the MCP/CLI boundary.
- **Claims 1–4** plus component acceptance metrics (§7). Results are pre-registered and published whether or not they favour memtidy.

### Who builds what (proposed; the mentor can change it)

| Piece | Owner | Licence / gate |
|---|---|---|
| mdsieve: fixture builder, LongMemEval converter, query sets, harness, scorers, results | Mentee | Apache-2.0, the mentee's own public repo from S0; G0-independent |
| Reference find tool: noise filter, FTS5 index, `search_md`, secret gate, MCP server/CLI | Mentee | Lives in mdsieve as a baseline. Goes upstream to memtidy only after G0 (L14) clears **and** contributor terms (e.g. DCO) are agreed in writing |
| Reference create and scope parts (Planned) | Mentee | Specified from public prior art (Basic Memory, the Claude memory tool, krimto, Gemini CLI memory tiers, 06's read-set rule) and the one-pager. Published before memtidy is public only if the mentor agrees in writing at S0; otherwise kept in a private branch, with acceptance tables published |
| memtidy: product, name (L30), roadmap, private validation (frontier agents on the mentor's own corpus) and its write-ups | Mentor | Not touched by this project. **The mentor's data is never used** |
| The mentee's employer IP clause for outside OSS work | Mentee | Checked in writing **before S0** |

The mentee's write-up covers the public benchmark only. memtidy's own validation and the n = 1 laptop scan stay the mentor's to publish. Publication order and mutual citation are agreed in writing before S0.

### Non-Goals (sibling docs in bold)
- Retriever choice on web corpora, and memory vs long context, so no "all md in the prompt" arm: **03 Spelunker**.
- Record/replay and A/A-calibrated CI: **05 BlackBox**. 09's runs are its cassettes.
- MCP gateway and injection defence: **06 Ctrl-Z**.
- Searching harness or prompt text: **07 HarnessForge**. 09 varies md context only: noise share, tool, delivery and pointer.
- Cloud sync, hosted team servers, adoption or pricing claims, training, and redistributing corpora.

### Relation to Sibling Proposals
- **03 Spelunker** owns retriever choice on web corpora (its Claim 3) and memory vs long context (Claims 1–2, `note`/`recall`). 09 runs a similar retriever ladder on noisy local md, where noise filtering and result-delivery format are the new factors.
- **05 BlackBox** owns record/replay and A/A-calibrated CI. 09 records every agent run as a cassette, and its A/A floor uses 05's §7.9 power method.
- **06 Ctrl-Z** owns the MCP gateway and injection defence. 09's stdio MCP server is a possible Customs target, and 09's `base_version` rule reuses 06's versioned read-set commit (06 §7.3). Write-path persistence is a stretch run in 06's harness (06 §7.5), where 09's create path (secret gate, PR-only shared writes) is one arm; any injection screening there is 06's.
- **07 HarnessForge** searches harness and prompt text. 09 holds the harness fixed and varies md context.
- **02 Shellwright**: `notes` and compaction are the in-agent analogue; none of its knobs is md context on disk.

### Why NightShift's rejection (IDEATION L100) does not apply

| NightShift reason | 09's answer |
|---|---|
| Crowded (Mem0, Zep, Letta) | No memory system is built to compete. The claims are cost at equal accuracy as noise grows, which component pays (filter vs index), and the tool interface (delivery, uptake). qmd, Basic Memory and rg are *run* in the harness, and "grep is enough" is a publishable null |
| Headline LongMemEval score needs a swapped-in local judge | Every headline metric is judge-free: a gold path among ≤ 3 returned, token and call counts from `llama-server` timings, and Recall@10 against LongMemEval's `answer_session_ids`. No QA-accuracy headline; the secondary answer metric is substring match on a frozen short-answer subset |
| Ingestion takes 1–3 nights | A full FTS5 build of the fixture is gated at ≤ 10 min CPU, and mounting one history at ≤ 10 s incremental. Dense embedding runs only for claim 4, on filtered, hash-deduped chunks, ≤ 1 night (measured) |
| "Sleeps at night" hook | No consolidation, versioning or forgetting while idle. Overnight runs are compute scheduling only |
| Its most novel experiment (memory vs long context) belongs to Spelunker | Agreed: 09 has no long-context or stuffed-context agent arm; the Planned vendor-shelf row (§7.3) is deterministic accounting, not an agent run |

## 3. Alignment With Prior Experience

| Prior experience | memtidy module |
|---|---|
| **CAS**: cascaded binary-encoding retrieval; Precision@K / latency / storage over 20+ configs | **Retrieval bench** (claim 4): Recall@10 vs p95 latency vs index size across rg, FTS5, dense and hybrid |
| **Changsheng**: AutoML hyper-parameter agent; backtest promotion gates | **Kill rule as a promotion gate**: vectors and the default result format ship only through pre-registered tests |
| **DiDi**: A/B testing | **Paired agent A/B**: variance pilot, A/A floor, non-inferiority, equivalence, Holm |
| **Sina Weibo**: scheduled SQL ETL, standardized metrics, alerting | **Indexer/watcher**: freshness lag, ghost-hit checks, one metrics row per run, a leak-scan alert in CI |
| **College Application Agent**: embedding-based retrieval (EBR) behind FastAPI; LangGraph Writer–Critic | **Dense arm + MCP/CLI service**; create's propose → check → write loop (Planned) |
| **Music Genre**: PCA, K-means (clustering) | **Noise-filter error analysis**: clustering misfiled md by path and content features |

**Not prior experience:** memtidy is the mentor's planned product, with no code. Enclave, an earlier mentor proposal, is a stalled Sprint-0 scaffold; 09 reuses only its RRF fusion pattern and tests (§7.4) and its socket-guard idea, with an IP-based loopback allow-list (§7.3).

**Interview one-liner:** *"At CAS I measured a retrieval scheme's accuracy–latency–storage frontier. Here I measured what a 4B agent on a 6 GB laptop GPU pays to find one note among tens of thousands of plugin markdown files, and which part of a search tool pays it back: at equal accuracy [the noise filter alone cut tokens X% and the index added X% | the index added nothing beyond the filter], and result format moved success X points."*

## 4. Stack Choices — and Why

### 4.1 Models

| Role | Model | File / quant | KV/token (f16) | Why |
|---|---|---|---|---|
| Agent, `portable` | Qwen3.5-4B (Apache-2.0) | Q4_K_M 2.74 GB | 32 KiB (8 of 32 layers attend) + ~48 MiB Gated DeltaNet state per slot | Fits 6 GB at 16k with headroom |
| Agent, 8 GB / Mac | Qwen3.5-9B | Q4_K_M 5.68 GB | 32 KiB | 9B rows; floor fallback |
| Standard-attention fallback | Qwen3-4B-Instruct-2507 | Q4_K_M 2.5 GB | 144 KiB | Prefix-cache safety (§11) |
| Dense arm (claim 4) | Qwen3-Embedding-0.6B (1024-d, multilingual) | fp16 ~1.2 GB | — | zh→en and paraphrase strata; a research arm, not a product default |
| Query and probe generator | Qwen3.5-9B (Mac) | Q4_K_M | — | Paraphrase queries and duplicate probes, labelled *generated* |
| Optional rerank rung | Qwen3-Reranker-0.6B | Q8_0 ~0.64 GB | — | Dev rung only |

**Why Qwen3.5.** Its small dense models (2026-03-02, Apache-2.0) are the newest Qwen weights that fit 6 GB; every later Qwen release is ≥ 27B or MoE (as of October 2026; re-verify in S0). **Thinking** is off (`enable_thinking: false`), with one dev ablation rung on. **KV arithmetic.** 8 attention layers × 4 KV heads × 256 × 2 (K, V) × 2 bytes = 32 KiB/token, so 16k costs 0.5 GiB; 2507 is 36 × 8 × 128 × 2 × 2 = 144 KiB/token, so 12k ≈ 1.69 GiB. *Sizes: unsloth GGUF listings; confirm cards and sizes in S0.*

### 4.2 Runtime
One pinned `llama-server` build (CUDA / Metal, `--jinja`); `-np 2` only if S0 shows a gain with reuse intact. vLLM has no native Windows build. llama.cpp #22384, #24587 and #24714 report hybrid re-prefill, so the prompt is append-only and byte-stable with a fixed tool order (IDEATION's prefix-cache rule). #20198 reports tool-call `arguments` returned as an object, so an argument validator sits in front of every tool (both re-checked on the pinned build in S0). `/tokenize` counts startup tokens; per-call `timings` record `prompt_n`, `cache_n` and `predicted_n`.

### 4.3 Libraries

| Concern | Choice | Why |
|---|---|---|
| Index | stdlib `sqlite3` FTS5, `bm25()` (k1 = 1.2, b = 0.75, fixed in SQLite source); unicode61 + trigram dual index for CJK; external-content table | Lexical first (L35); rebuildable from files |
| Vectors (claim 4) | sqlite-vec, flat cosine | Same file as the index |
| Grep | ripgrep, pinned; `--hidden` in A, stock defaults as rung 0 | rg skips hidden dirs by default, so flags are pre-registered |
| Watcher | `watchdog` + per-call mtime/hash check | Freshness without a daemon dependency |
| MCP | MCP Python SDK, stdio, tools only, `readOnlyHint` on reads | Spec 2026-07-28 deprecates Roots, so scope is a tool parameter; deterministic `tools/list` order keeps the prefix cacheable |
| Duplicates (Planned) | `datasketch` MinHash LSH + exact containment / Jaccard | Containment fits superset copies |
| Property tests | Hypothesis `RuleBasedStateMachine` | 06 §7.6 style |
| IR metrics, stats | ranx, statsmodels, SciPy | Recall@k, McNemar, Wilson, cluster bootstrap |
| Baselines run, not quoted | qmd (MIT, 30,551★ on 2026-10-22); Basic Memory (AGPL-3.0, out of process); gitleaks v8.30.1 plus a 2022-era v7.x for S0; Betterleaks; detect-secrets | Build vs buy is measured |

*Vendor versions, spec dates and star counts: as of October 2026; re-verify in S0.*

**Prior art.** The grep-vs-vector papers (§1); LongMemEval-V2, where a coding agent over trajectory files scores ~70–75% vs ~48.5% for the best RAG baseline (search summary, unverified); SWE-ContextBench, where "free retrieval … can even hurt" (search summary); a self-published LongMemEval-S table with BM25-only `recall_any@5` 86.2% and BM25 + vectors 95.2% (agentmemory); qmd, Basic Memory and claude-mem as tools. **09 adds** a laptop open-weight agent, noise share as a crossed factor scored on held-out marketplaces, the filter-vs-index attribution, delivery crossed with tool, measured uptake with a pointer ablation, and a powered three-valued kill rule.

## 5. Benchmarks & Datasets

### 5.1 Synthetic home fixture (public; ground truth by construction)
- **Builder.** A script makes a clean OS user (`$HOME`) per machine and installs public plugin and skill marketplaces for Claude Code, Codex, Cursor, Copilot and Gemini CLI at pinned commits, into the paths those agents use. It adds `.cache` web-page md and third-party clones under `~/code`.
- **Held-out split.** Marketplaces are split by name: a **filter-dev** set (about a quarter of the files), on which the filter's rules are written and frozen, and a **held-out** set, which alone makes the scored fixture and includes at least one vendor layout the rules never saw. Split, commits and file lists are frozen in `prereg/` at the end of S0.
- **Self-written set** (~400 files, measured in S0): public agent files (CLAUDE.md, AGENTS.md, `.claude/rules`, `memory-bank/`) from OSI-licensed repos found by the mentee's own GitHub search, re-fetched at pinned SHAs into own-project folders; hard cases (`/init`-style generated files, same-named `README.md` files, vendor copies of user notes); 120 agent needles (§5.3), each a fictional fact (port, flag, owner, date, command, decision) in exactly one file, with 1–3 stale decoys in cache or vendor paths for half of the test needles; and one mounted LME-md history per episode (§5.2).
- **Noise levels.** 0% (vendor and cache md removed), 90% (a fixed-seed stratified sample of held-out vendor md) and full. The full share is measured in S0; the n = 1 scan implies about 99% by file count.
- **Published.** Builder, file lists and hashes only, never the files.

### 5.2 LongMemEval as md (LME-md)
- **Size.** 500 questions; the 30 abstention items are skipped, as the official retrieval evaluation does, leaving 470. `_S` histories are ~115k tokens and ~40–50 sessions each. Gold = `answer_session_ids`.
- **Licence and version.** The code is MIT (LICENSE checked 2026-10-22); the dataset card licence is confirmed in S0. The README links only the 2025-09 cleaned files, so cleaned data is the default; original files are used only for S0, if still hosted. The user downloads the data; mdsieve never redistributes it.
- **Converter.** A shipped converter writes each session as one md file with `date:` frontmatter and turns under headings; chunks are 400–800 tokens on heading boundaries.
- **Mounting.** Each question's history is mounted at one of five documented md locations by a fixed-seed rule: Claude Code auto memory, Codex memories, `~/notes/`, a project's `docs/`, or a project's `memory-bank/` (vendor paths as of October 2026; re-verify in S0). The location is a reported stratum, and mounting is a timed incremental re-index.
- **Splits.** Pilot 60, dev 100, test 200 for agents. The 110 spare questions allow the n-rule (§9.3) and the `_M` ceiling rung, which keeps the same questions and gold with ~500 sessions, streamed.
- **Excluded and reserve.** LoCoMo (CC BY-NC 4.0) is excluded; MemoryAgentBench (MIT, exact match) is the reserve corpus.

### 5.3 Query sets (frozen by hash in `prereg/` at S0 end)

| Set | Source | Dev / Test |
|---|---|---|
| Agent questions, primary (claims 1–3) | LME-md, stratified by question type | 100 / 200 (+60 pilot) |
| Agent needles, secondary | Seeded needles, keyword-style queries; test: 40 clean + 40 decoy (set aside) | 40 / 80 |
| Context-free items (claim 3b) | 80 LME-md dev questions with the gold session pasted into the task; any search is spurious | 80 |
| Retrieval, keyword (claim 4) | LME questions sharing ≥ 2 content stems (Porter, stopwords removed) with a gold session; deterministic rule, fixed-seed split independent of agent splits | 60 / 210 |
| Retrieval, paraphrase (claim 4) | 270 retrieval-only needles whose query the 9B rewrote until it shares no content stem with the gold file; labelled *generated*; 60 checked by the mentee (≥ 90% answerable, else regenerate) | 60 / 210 |
| Retrieval, zh→en (claim 4) | One sentence of a public `README.zh-CN.md` as the query; the paired `README.md`, placed as an own-project doc, is gold; the zh file removed | 60 / 210 |
| Private real-query arm (Planned) | The mentee's own transcripts: ~50 lookups where an agent could not find an md it wrote (short written protocol: the last time, how it was found, how long it took); gold labelled by the mentee | aggregates only |

### 5.4 S0 external reproduction (pre-registered; two co-targets)

| Source | Metric | Published | Local |
|---|---|---|---|
| Samsung CredData README table ("Updated: April 2022"; vendor-authored; train + test combined) | gitleaks TP / FP / P / R / F1 | 1,120 / 1,011 / 0.5256 / 0.2444 / 0.3336 | **S0** |
| LongMemEval (Wu et al., ICLR 2025), BM25 retrieval row | session-level Recall@5 / NDCG@5 (`recall_all@5`, `ndcg_any@5`) | read from the PDF in session 1, committed before any run | **S0** |
| *Context only, never in our columns:* agentmemory, LME-S cleaned (self-published) | BM25-only `recall_any@5` | 86.2% (BM25 + vector 95.2%) | — |
| *Context:* Sen et al. 2026 (arXiv 2605.15184) | grep vs vectors, 4 harnesses × 5 models | direction only | — |
| *Context:* arXiv 2608.01507 | repo QA, semantic vs grep subagent | 65.2% vs 46.2% (unverified) | — |
| *Context:* Cursor (vendor) | offline QA, semantic + grep | +12.5% | — |

**(a) CredData.** 19,459,282 lines, 11,408 files, 297 repos; 73,842 labelled lines, 4,583 true. `python download_data.py` needs Linux, so it runs in WSL2 ext4 on the Windows laptop (NTFS filenames fail; macOS APFS case-insensitivity may too). The run is the repo's `python -m benchmark --scanner gitleaks`; its wrapper calls the pre-v8 CLI (`--no-git -p… -o …`, fields `file`/`lineNumber`), so a 2022-era v7.x binary is pinned and declared, with v8.30.1 and Betterleaks as context rows. Each data file keeps its source project's licence, so the data stays local. CPU only, about an evening (to be measured). **Pass:** with ≥ 98% of labelled lines downloaded, F1 within ±0.03 and TP within ±5%; otherwise the gap is a port delta.

**(b) LongMemEval.** `bash run_retrieval.sh <file> flat-bm25 session` (rank_bm25 BM25Okapi on whitespace tokens). It runs in WSL2 (`declare -A` needs bash ≥ 4; macOS ships 3.2), in a separate CPU-only env, because the script imports torch, sentence-transformers, openai and transformers even for BM25, with NumPy < 2 (`eval_utils.py` calls `np.asfarray`). If the paper's row is on `_M` (a ~2.5–3 GB JSON loaded whole), the loader streams it with ijson; scoring code is untouched. **Pass:** both metrics within ±2 pp on the original file; on cleaned data only, the gap is a port delta, never a pass. Also reported, not gated: our FTS5 BM25 (k1 1.2) vs rank_bm25 (k1 1.5 by default) on the same md.

**S0 is met** if either target reproduces. Otherwise both gaps are published as port deltas (08's convention) and marked "not reproduced".

### 5.5 Data for Planned components
- **LOO md trees.** kubernetes/website `content/en/docs` and docusaurus `/docs` (both CC BY 4.0); mdn/content `files/en-us` (prose CC-BY-SA 2.5); public `.claude/rules` and `memory-bank/` folders (OSI repos). ≈ 120 test + 60 dev held-out files per tree, ≈ 600 / 300 in all, fetched from a `repo@sha + path` list and never redistributed.
- **Duplicate probes.** ≈ 600 positives (superset copies, edits, reorders, public README translation pairs, 9B paraphrases labelled *generated*); half have the original in a different visible scope. Negatives: the LOO held-out files plus ≈ 300 same-template hard negatives.
- **Secret seeds.** ≥ 20 key types × ~100 format-valid fakes (≈ 2,000), placed in prose, code, tables, frontmatter and URLs, including base64 and URL-encoded variants. Hard negatives: `${VAR}`, SHAs, UUIDs, documented example keys. **False-positive corpus:** ≈ 100k lines of the fixture's public md.

### 5.6 Demo data
One dev question where A and B disagree, replayed from cassettes. It is hand-picked and never used for a claim.

## 6. Architecture

```
ALL LOCAL, NO NETWORK AT RUN TIME (one git remote, LAN only, for the two-machine test)

FIXTURE (pinned SHAs)            TOOL SIDE (deterministic, no LLM)
 synthetic $HOME -------------> NOISE FILTER ---> INDEXER/WATCHER ---> FTS5 + metadata
  |- vendor md: held-out         path rules,       mtime + hash,        path/type/scope/
  |   marketplaces, caches       vendor hashes,    H1-H3 chunks         mtime [+ vec: C4]
  |- own md, needles, decoys     foreign clones   SECRET GATE: redact before index/return
  |- LME-md at 5 memory dirs                      CREATE + SCOPES (Planned): git, PRs
  `- scopes = local git repos                               |
                                       MCP SERVER (stdio) == CLI (one library)
                                                            |
HARNESS (held fixed) ---------------------------------------+---------------------------
 tasks --> AGENT LOOP (Qwen3.5-4B, llama-server) --> A    grep / glob / read (rg)
              |                                      A+F  same tools, filter applied
              |                                      B    A + search_md + pointer
              |                                      B-p  B without the pointer
              |                                           [inline | paths] x noise level
              `--> cassettes (05) --> SCORERS: gold path, prompt/cache/predicted tokens,
                                      calls, uptake split, leak scan --> STATS
RETRIEVAL BENCH (claim 4, no agent): rg | FTS5 | dense | hybrid RRF | qmd | Basic Memory
S0 (CPU, WSL2): gitleaks on CredData | LongMemEval flat-bm25, official scripts
```

**Walk-through.** The builder creates the haystack, with the scored noise drawn from held-out marketplaces. The tool side indexes it in minutes, through the filter and the secret gate. The harness runs one fixed loop with four tool menus and two delivery formats, at three noise levels, and records every call as a 05 cassette. Scorers read only cassettes and gold labels. Claim 4 bypasses the agent, and S0 runs the two external scripts unchanged.

## 7. Component Design

### 7.1 Tool side (deterministic; acceptance metrics, not claims)
The designs below are 09's own reading of the one-pager and L34–L36; memtidy may implement them differently (§10 spec freeze).

| Component | Design | Acceptance |
|---|---|---|
| Noise filter (L36) | Classes: self-written (instruction, memory, agent-artifact, user-doc) vs vendor / cache / unclassified. Signals: path rules for vendor dirs; a content hash seen ≥ 2× in vendor dirs; clones whose git remote is not the user's and that have no user commits; cache bodies that start with a source URL. No symlink following. Rules frozen on filter-dev marketplaces | P/R/F1 by class on the held-out marketplaces; unclassified share; scan time. A 300-file stratified human audit on the mentee's own machines (aggregates only) |
| Indexer / watcher | Watcher + per-call mtime/hash check; H1–H3 chunks prefixed "title > heading path"; FTS5 external-content table with path, scope, `type`, mtime and class; rebuildable from files | Full / incremental build time; write → searchable lag p50/p95; **0 ghost hits** after a ≥ 10k-operation rename/delete storm; **I7**: rebuilt index = incremental index; p95 latency at 0.5k / 5k / full |
| `search_md(query, k=8, path_prefix?, type?, scope?, modified_after?)` | BM25 with a query sanitizer (FTS5 `MATCH` throws on raw `" - : * (`) and an OR-of-terms fallback; hits grouped by file; scope label on every hit; output ≤ 2,000 tokens; inline or paths delivery | Claim 4 port delta; p95 call latency |
| Secret gate (L36) | Regex + entropy rules (gitleaks-compatible); `[REDACTED:<rule>]` before indexing and before any return; output shows rule, file, line and ≤ 4 characters. Motivation only: one heavy user's laptop scan (n = 1, unpublished, the mentor's) found one token in plaintext in 4 agent config files | Recall on ≈ 2,000 seeded fakes (an upper bound; 0 misses ⇒ < 0.15% at 95%, rule of three). False-positive files per 1k md lines vs gitleaks v8.30.1 (`dir` mode), every hit of both on the ≈ 100k-line corpus hand-reviewed; detect-secrets and Betterleaks counts unreviewed. **Leak scan**: no ≥ 6-character substring of any seeded value in tool output, logs, FTS5 content, cassettes or git objects (`git cat-file --batch-all-objects`). Any leak fails acceptance (fix, rerun, report both) |
| MCP server | stdio, tools only: `search_md`, `read_md`, `list_scopes`, plus `create_md` (Planned); `readOnlyHint` on reads; fixed tool order; the CLI is the same library; hash-pinnable for 06 Customs | Tool definitions ≤ 800 tokens (measured); p95 call latency; MCP RSS measured |

### 7.2 The agent loop

| Element | Specification |
|---|---|
| Observation | (1) A byte-stable system prompt: rules and tool definitions in a fixed order. In B only, the last line is the pointer `Your notes are indexed: call search_md before grepping.`, standing in for L36's native-file hint. (2) The task card: `Find the user's own note(s) that answer: <question>. Return up to 3 paths with final.` (3) Append-only tool results |
| Action | One function call per turn in Qwen3.5's native tool format, or `final(paths ≤ 3, answer?)`. Arguments are validated; an invalid call counts as a parse failure and is never silently retried |
| Tools | **A:** `grep(pattern, path?, glob?)` (rg `--hidden`, ignore files honoured), `glob(pattern)`, `read(path, offset?, limit?)`. **A+F:** the same tools over the filter's output (an rg ignore file for path rules, an exclude list for per-file decisions). **B:** A + `search_md`. **B−p:** B without the pointer. *Inline* = matching lines or snippets with path and heading; *paths* = `rg -l` or ranked paths only |
| Termination | (1) `final`; (2) 15 tool calls, then one "answer now" turn; (3) context reaches 16k, then one "answer now" turn; (4) 3 consecutive unparseable calls, scored as failure. A 300 s watchdog is a runaway guard, not a termination rule: a trip is logged as an infrastructure fault, re-queued once, then scored as a failure |
| Budgets | 15 calls; 16k context; **2,000 tokens per tool result in every arm**, with a marker giving the dropped-match count; ≤ 512 generated tokens per turn; tool definitions ≤ 800 tokens; thinking off (one dev rung on); the card's non-thinking sampling (temperature 0.7, top-p 0.8, top-k 20; presence penalty 0, as in 05 §7.5), pinned, seeds logged |
| Cost accounting | Per-call `timings` from the pinned build, semantics verified in S0. **Total prompt tokens = Σ(`prompt_n` + `cache_n`)** is the primary, cache-independent cost; prefilled tokens = Σ `prompt_n`; startup tokens per arm from `/tokenize`. The first 20 episodes per cell run cold (slot erased), the rest warm; the two are reported apart |

The 2,000-token cap is what makes noise expensive: at full noise an unfiltered `grep` for a common term fills its cap with vendor matches, and the dropped-match marker tells the agent to narrow the pattern, which costs another call. The same cap binds `search_md`, so no arm wins by seeing more.

### 7.3 Harness and scorers
- **Runner.** Resumable, one config hash per cell; records 05 cassettes (prompts, timings, tool I/O, wall-clock).
- **Network-off smoke test.** Enclave's socket-guard pattern, with an IP-based loopback allow-list, around a full agent episode and an MCP session.
- **Gold-path scorer.** Paths normalised (NFC, separators, case-insensitive on Windows). Primary: ≥ 1 gold path among ≤ 3 returned; secondary: all gold paths returned; answer substring match on a frozen short-answer subset (LME answers ≤ 5 tokens; needle answers exact).
- **Uptake parser.** Splits B and B−p failures into zero-call / gold never retrieved by any `search_md` result / retrieved but not returned in `final`.
- **Other scorers.** Token, call and parse counters; leak scanner; ranx for claim 4. **No LLM judge.**
- **Vendor-shelf row (Planned, deterministic).** For each test question: would the gold file have been in context at session start under Claude Code's documented load (CLAUDE.md chain; the first 200 lines or 25 KB of `MEMORY.md`) or Codex's (AGENTS.md chain ≤ 32 KiB)? Emulators follow the docs as of October 2026 (checked 2026-10-22; re-verify in S0) and report token cost. No agent run, since memory vs long context stays with 03.

### 7.4 Retrieval bench (claim 4)
- **Arms.** *rg as a retriever*: files ranked by distinct query terms matched, then by match count. *FTS5 BM25*: a file's score is its best chunk. *Dense*: Qwen3-Embedding-0.6B chunk embeddings, flat cosine via sqlite-vec, query instruction per the model card. *Hybrid RRF (k = 60)* of FTS5 and dense; the fusion and its tests (`fake_embedder`, the punctuation cases, the RRF formula) are ported from Enclave's Postgres RRF SQL to SQLite (FTS5 `bm25()` is negative, so ascending order). Optional Qwen3-Reranker-0.6B rung (dev only). qmd and Basic Memory retrieval rows (Planned; context, not in the verdict).
- **Selection.** Best lexical (FTS5 vs rg) and best vector (dense vs hybrid) are chosen on each stratum's dev; literal "vs rg" rows are reported beside them.
- **Corpus.** The filtered fixture plus each stratum's targets. Embeddings run on hash-deduped chunks; build time and index size are logged.

### 7.5 Create, scopes and the two-laptop relay (Planned)
**Create** follows the one-pager (one call proposes path, name and metadata, flags near-duplicates in every visible scope, scans for secrets, writes, indexes):
1. Placement: a layout rule first, then a BM25 vote over folder profiles (top-3).
2. Filename: an NFC kebab slug ≤ 64 characters, with a case-insensitive collision check.
3. Frontmatter: a schema chosen by destination.
4. Duplicate cascade over **visible** scopes only: body hash → MinHash LSH (5-shingles, 128 permutations) → exact containment and Jaccard. The threshold is frozen on dev at FBR ≤ 2%. A hit returns `blocked_duplicate` (path, scope, score, `base_version`).
5. Secret gate (§7.1).
6. Write: personal commits with a `base_version` check, where a stale version is a conflict, never an overwrite (06 §7.3). Team and parent writes become proposal branches (L34); parent writes from child teams are refused.
7. Synchronous index update.

**Create acceptance.** LOO placement top-1/top-3 per tree vs the 4B agent choosing the folder with a capped `list_dir` (depth-limited tree, ≤ 2,000 tokens per call, ≤ 4 calls); frontmatter validity 100%; duplicate recall at the frozen FBR (test Wilson upper bound ≤ 4%), containment vs Jaccard (McNemar); a second annotator's "also acceptable" rate on 100 placements.

**Scopes (L34).** Personal, team and parent are local git repos, cloned on both laptops from one bare remote; a policy file stands in for git-host teams. Results come most specific scope first, each labelled.

**Relay acceptance (S3).** The Windows agent's note is written through `create_md` and pushed; the Mac pulls, and its agent finds the note with `search_md`; a seeded fake key in a pasted log never reaches the team repo. 20/20 CLI-driven relays must succeed, with 0 leaks and lag after pull ≤ 10 s. 20 agent-driven relays report the create → `blocked_duplicate` → `update` compliance rate descriptively. Property suites run in two-clone single-machine mode by default; the true Windows ↔ Mac schedule runs on a labelled 1,000-sequence subset.

### 7.6 Invariants (acceptance tests; 06 owns the pattern)
A Hypothesis `RuleBasedStateMachine` runs write, update, promote, approve, reject, rename, delete, sync-on-machine-X and concurrent-update operations over 2–3 nested scope trees:
- **I1:** no output reveals a note from an unreadable scope, or that one exists. **I2:** the parent is read-only to children. **I3:** shared writes exist only as proposal branches. **I4:** most specific scope first, every hit labelled.
- **I5:** a stale `base_version` gives a conflict, never a lost update. **I6:** the secret gate runs before promotion. **I7:** the rebuilt index equals the incremental index (Core, §7.1). **I8:** cross-OS filename identity holds (NFC/NFD, case, CRLF).

Targets: 0 violations in 10,000 sequences per invariant (< 0.03% at 95%) and in 1,000 cross-machine sequences (< 0.3%). The policy is parameterised, so the suite survives memtidy changes to inheritance.

## 8. Hardware Profiles

> **Rule: the weakest machine sets the defaults.** `portable` assumes 6 GB; `accelerated` is opt-in once an 8 GB card is confirmed.

- `portable` (default): 4B, 1 slot × 16k, f16 KV, on the 6 GB card.
- `accelerated` (opt-in): the 9B at 1 × 16k on a confirmed 8 GB card (tight), or the 4B with `-np 2` if S0 shows reuse intact.
- `mac`: the 9B rows and 4B parity rows on Metal.

### 8.1 Memory budget

| GB (buffers estimated; measured in S0) | 6 GB (`portable`) | 8 GB (`accelerated`) | Mac 16 GB |
|---|---|---|---|
| Agent weights | 4B 2.74 | 9B 5.68 | 9B 5.68 (4B 2.74 for parity rows) |
| KV @16k, 1 slot (32 KiB/token) | 0.50 GiB | 0.50 GiB | 0.50 GiB |
| Gated DeltaNet state, 1 slot | ~0.05 | ~0.05 | ~0.05 |
| Output buffer (248k vocabulary, `-ub 512`) + CUDA context | ~0.5 + ~0.3 | ~0.5 + ~0.3 | ~0.5 |
| **Total, 1 slot** | **~4.1 (fits)** | **~7.0 (tight)** | **~6.7** (under the Metal working-set cap, recorded in S0) |
| 2 slots (`-np 2`, only if reuse holds) | ~4.7 | no | ~7.3 |
| Fallback: Qwen3-4B-Instruct-2507 @12k (1.69 GiB KV) | ~5.0 (tight) | fits | fits |
| Embedder (claim 4 only; agent server stopped) | Qwen3-Embedding-0.6B fp16 ~1.2 | same | MPS |
| Tool side (index, rg, watcher, MCP server) | CPU RAM ≤ 1 GB target | same | same |

Usable VRAM is measured with `nvidia-smi` on day one. **What runs where.** *Windows:* every scored 4B episode; the fixture build; claim-4 embeddings; both S0 scripts in WSL2 ext4; the LOO 4B baseline. *Mac:* the 9B rows or the 9B fallback grid; 9B query and probe generation; Hypothesis suites; the second laptop of the relay; development and demo. *Both:* fixture, filter and index timings (gate row 5); rg latency; the git sync test.

### 8.2 Week-2 go/no-go gate
End of session 2, on Windows, with 30 LME-md dev questions in A and B at full noise; build times on both machines.

| # | Metric | Pass if |
|---|---|---|
| 1 | Tool-call parse failures | ≤ 5% |
| 2 | Median episode | ≤ 90 s (the §8.4 ledger plans 75 s) |
| 3 | Prefix reuse (prefill / prompt tokens per turn after turn 1) | ≤ 0.2 |
| 4 | rg latency per call on the full fixture (NTFS) | p95 ≤ 5 s |
| 5 | FTS5 full build / incremental after 100 edits or one history mount, both machines | ≤ 10 min / ≤ 10 s |
| 6 | Early floor read: grep@noise success | 30–85% (warning only; the S1 floor gate binds) |

### 8.3 Fallback ladder (declared now, so it cannot be renegotiated)

| Fails | Fallback, in order |
|---|---|
| 1 | Built-in GGUF template → newer pinned build → lazy-GBNF constrained calls → Ollama |
| 2 or 3 | `--ctx-checkpoints` sweep → Qwen3-4B-Instruct-2507 at 12k → 10 tool calls → LME-md test 200 → 150 (MDE declared) |
| 4 | Pinned rg `--threads` and `--max-filesize` → third-party clones pruned to 25% (noise share re-measured) → a WSL2 ext4 copy of the fixture (labelled) |
| 5 | Batched inserts in one transaction, WAL → trigram index only for files with CJK text |
| Floor gate, low (< 30%) | LME-md histories mounted at one location (`~/notes`) → the 9B fallback grid on the Mac (§8.4) → claims 1–3 "not testable on 6 GB", and claim 4 carries the doc |
| Floor gate, high (> 85%) | `_M`-size histories (~500 sessions) for the same questions, streamed, MDEs recomputed on dev → if still > 85%, claims run as planned and 3a is reported "ceiling-limited" (tokens still compared) |
| Floor gate, uptake (< 70%) | Three pre-written tool-description and pointer wordings on dev, best frozen → `search_md` first in the fixed order → the 9B → claims 1–3 run and are reported "uptake-limited" (claim 3b is the finding), never a pass |
| Floor gate, A0 (> 10%) or oracle (< 60%) | Needles at conventional paths re-placed; for the oracle, the answer metric is dropped (path success only) and 2507 is tried on dev |

At ceiling, the keyword stratum of claim 4 can only return kill or inconclusive, which is the expected verdict.

### 8.4 Compute budget
Planning rates: 75 s per 4B episode on Windows; 3.75 min per 9B episode on the Mac; both are replaced by the gate medians. Capacity: ~25 GPU-h a week on Windows (~5 h × 5 nights) over 8 run weeks (weeks 3–10) ≈ 200 GPU-h; the Mac has ~25 h a week ≈ 200 Mac-h.

| Block | Machine | Episodes | Hours |
|---|---|---|---|
| S0 variance pilot + A/A floor (60 LME q × {A, B} × 3 seeds) | Win | 360 | 7.5 |
| Week-2 gate (30 × {A, B}) | Win | 60 | 1.3 |
| Floor gate (140 dev × {A, B}; 100 oracle-path; 100 A0, one turn) | Win | 480 | 7 |
| Dev ladder (rungs 0–9 × 100 LME dev × 1 seed) | Win | 1,000 | 21 |
| Claim 1 (A, A+F, B; inline; full; 280 × 2 seeds) | Win | 1,680 | 35 |
| Claim 3 extra cells (A-paths, B-paths, B−p; 280 × 1 seed) | Win | 840 | 17.5 |
| Claim 2 (A, B at 0% and 90%; 280 × 1 seed) | Win | 1,120 | 23.3 |
| Spurious-call set (80 × {B, B−p}, short) | Win | 160 | 1.5 |
| Claim 4 embeddings + retrieval (no agent) | Win | — | ~6 (measured) |
| Planned: create LOO 4B baseline (900 short episodes) | Win | 900 | 5 |
| Planned: second fixture build, A and B on 100 dev | Win | 200 | 4.2 |
| Stretch: qmd, Basic Memory as B's tool (200 LME test × 1 seed) | Win | 400 | 8.3 |
| **Windows total** | | **7,200** | **~138 GPU-h of ~200** (~162 at the 90 s gate ceiling) |
| Planned: 9B rows (A, B; inline; full; 200 LME test × 1 seed) | Mac | 400 | 25 |
| 9B generation (paraphrase queries with regeneration slack, ~330; 300 duplicate probes) | Mac | — | ~2 |
| **Mac total** | | | **~27 Mac-h of ~200** |
| Fallback only: 9B grid (A, A+F, B, B−p × 200 × 1 seed) + 100-q dev floor read; replaces the 9B rows | Mac | 800 + 200 | 50 + 12 |
| Optional API row: Claude Code / Codex, A vs B, 50 LME dev q | API | 100 | ≤ $15 + $5 reserve ($20 total) |

**Pre-registered cut order** at the week-8 checkpoint: (1) stretch agent arms; (2) the API row; (3) the 90% noise level (claim 2's test needs only 0% and full); (4) seed 2 of claim 1 (power is planned on questions); (5) the second-fixture row; (6) the 9B rows; (7) ladder rungs 7–9. **Never cut:** claims 1–3 cells at seed 1, claim 2's 0% cells, the spurious-call set, claim 4, both S0 targets, the leak scan.

## 9. Evaluation Plan

### 9.1 Areas

| Area | Metrics | Against | n / MDE |
|---|---|---|---|
| Agent cost (claims 1–3) | Success (≥ 1 gold path in ≤ 3), total prompt tokens, prefilled tokens (cold / warm), tool calls, wall-clock, tokens per successful find, uptake split | grep@noise (A) | 200 LME-md + 80 needles; §9.3 |
| Retrieval (claim 4) | File-level Recall@10, `recall_all@10`, MRR, nDCG@10, p95 latency, index size, build time | best of FTS5 / rg | 210 per stratum; Holm-adjusted MDE ≈ 10 points |
| Tool side | Filter P/R/F1 on held-out marketplaces + 300-file audit; build, lag, ghost hits, I7; seeded-secret recall (upper bound), FP/1k lines, leak count | gitleaks; path-rules-only filter; index everything | §7.1 |
| Startup accounting | Startup tokens per arm (`/tokenize`); vendor-shelf coverage (Planned) | — | deterministic |
| Planned acceptance | Placement top-1/top-3, frontmatter validity, duplicate recall at FBR ≤ 2%, invariant violations, relay 20/20 | 4B with `list_dir`; Jaccard | §7.5–7.6 |
| Optional API row | Claude Code / Codex, A vs B | — | 50 q, descriptive, ≤ $20 |

### 9.2 Ablation ladder
Full noise, inline, dev; one change per rung: 0 rg stock defaults (hidden dirs skipped) → 1 `--hidden` (A) → 2 + noise filter (A+F) → 3 + `search_md`, BM25 only, no metadata filters, no pointer → 4 + metadata filters → 5 + pointer (= B) → 6 inline → paths → 7 + hybrid (only if claim 4 keeps a stratum) → 8 B without grep → 9 thinking on. Non-agent reference row: one FTS5 call, top-3 paths.

### 9.3 Pre-registered tests
- **Settings.** α = 0.05, power 0.8. Paired-binary MDE: (z₁₋α/₂ + z₀.₈)·√(p_d/n). Results are conditional on the frozen fixture hash.
- **Bootstrap.** Paired, 10,000 resamples over clusters, with seeds and machines nested inside. A cluster is an LME question with its history (questions sharing an evidence session merged) or a needle's generator template. McNemar runs as a check.
- **Families.** The agent family is C1a, C2 and C3a, Holm over 3. C1b and C3b are pre-registered secondary verdicts at α = 0.05, reported whatever C1a's outcome. Claim 4 is its own family, Holm over 3 strata.
- **Variance pilot and A/A floor (S0).** 60 LME questions × {A, B} × 3 seeds; seed-vs-seed runs of A give the A/A floor (05 §7.9 method). MDEs use the larger of the A/A and A/B discordance and log-token SD, replacing the planning values (p_d 0.2, σ_d 0.6) before test sets open.
- **n-rule.** If the pilot's discordance exceeds 0.25, the LME-md test set rises to 300 questions (60 + 100 + 300 = 460 of 470), costing +1,300 episodes ≈ +27 GPU-h (138 → ~165 of 200).

| Claim | Statistic | Decision rule | Planning MDE / power |
|---|---|---|---|
| 1a | Δ success B − A (one-sided); GM token ratio B/A | One-sided 95% lower bound of Δ > −8 points **and** ratio upper bound < 1, at the Holm-adjusted level | Success power 0.81 at p_d 0.2, n 200 (0.66 at the strictest Holm step); tokens ≈ 11% at σ_d 0.6 (≈ 13% at the strictest step) |
| 1b | GM token ratio B/(A+F); Δ success | **Index adds**: ratio upper bound < 1, or B − A+F success lower bound > 0. **Filter suffices**: 90% CI of the ratio inside [0.87, 1.15] and A+F non-inferior to B (8 points); a lower bound > 1 reads "filter suffices; the index adds cost". **Inconclusive** otherwise | TOST power 0.90 at a true ratio of 1 (σ_d 0.6, n 200) |
| 2 | Per-question difference-in-differences of log tokens, (A − B) at full minus at 0% | Lower bound > 0; the 90% level is a dose point | ≈ 0.17 log points, a ratio of ratios ≈ 1.18 (≈ 0.19 at the strictest step) |
| 3a | Tool × delivery interaction on success (seed 1 of all four cells) | CI excludes 0 → direction reported. Default format = higher B success, ties broken by B tokens | ≈ 12.5 points (≈ 14.5 at the strictest step) |
| 3b | McNemar on "called `search_md` ≥ 1", B vs B−p; zero-call share of B failures (Wilson) | "Uptake binds" if the zero-call share's 95% interval is above 50%; "not binding" if below 25%; else "mixed". Pointer adopted only if spurious calls ≤ 20% on the 80 context-free items | ≈ 9 points at q 0.3, n 280 |
| 4 | Δ Recall@10 (best vector − best lexical) per stratum | **Keep**: Holm-adjusted lower bound > 0 and Δ̂ ≥ 10. **Kill**: Holm-adjusted upper bound < 10. **Inconclusive** otherwise | Holm-adjusted MDE ≈ 10 points (n 210, p_d 0.2). Kill-power 0.80 at Δ = 0; keep-power 0.83 at Δ = 13 and 0.5 at Δ = 10. Private arm, literal L35 rule: MDE ≈ 18 points at n 50 |

**Why 8 points.** It is about 2.5 SE at n = 200 (SE ≈ 3.2 points), the smallest margin the budget powers. At the floor's 30% base rate, a 30% token cut with an 8-point loss still lowers tokens per successful find (0.70 / 0.22 ≈ 3.2 vs 1 / 0.30 ≈ 3.3), and higher base rates are more forgiving; tokens per successful find is reported with its CI. **Why ±15% for "filter suffices".** It is the smallest token difference worth an index's build and maintenance cost, and the budget powers it (TOST 0.90), while ±10% would not (≈ 0.45).

The MDE is for planning (smaller test effects may be missed), never a pass bar: requiring observed Δ ≥ MDE passes only ~50% of the time at a true effect of that size. Claim 4's 10-point bar is memtidy's product threshold (L35), not the MDE, which is why its keep-power is stated at both 10 and 13 points.

### 9.4 Honest reporting
- No LLM judge in any headline; the secondary answer metric is substring match on a frozen short-answer subset.
- Nulls are published. "Not testable on 6 GB", "uptake-limited", "ceiling-limited", "not run" and "inconclusive" are never passes.
- Published rows are context only, in separate columns; LME-md results are never compared with published LongMemEval scores. No comparison with an API model the project never ran.
- Build, flags, GGUF hashes, the fixture hash and marketplace commits go on every table.
- Decoy items are a set-aside by-construction stratum (as 06 sets aside its policy arm's by-construction wins). Generated paraphrase queries and probes are labelled; synthetic-secret recall is an upper bound.
- The private arm and the filter audit are reported as aggregates only. The mentor's data is never used.
- Small-model external validity is stated: no transfer to frontier agents is claimed.
- Results are published whether or not they favour memtidy.

## 10. Milestones (19 hours / 15 sessions)

| Sprint | Sessions | Hours | Deliverable |
|---|---|---|---|
| **S0** | 1–4 | 5 | Mentee employer-IP check and the credit / publication terms recorded; mdsieve repo (Apache-2.0); `uv` envs on both OSes (WSL2 for S0 scripts); pinned build + GGUF hashes; fixture builder + noise census, marketplace split frozen; LME-md converter; **CredData reproduction** and **LongMemEval BM25 row**; FTS5 tool v0; minimal A/B loop; **week-2 gate** (session 2); variance pilot + A/A floor; splits, strata and MDEs frozen in `prereg/` |
| **S1** | 5–8 | 5 | Noise filter (frozen on filter-dev marketplaces), watcher, secret gate and leak scan, MCP server/CLI; **floor gate**; dev ladder rungs 0–9; claims 1 and 3 test cells start overnight. **Week-6 demo**: two agents, one question, live token meters, replayed from cassettes |
| **S2** | 9–10 | 3 | Claims 1 and 3 complete on test; claim 2 noise sweep; default format frozen; claim-4 strata built (paraphrase queries, zh→en pairs); **week-8 cut checkpoint** (session 10) |
| **S3** | 11–12 | 3 | **Claim 4** verdicts (dense, hybrid; qmd and Basic Memory rows); Planned tier: private arm, create acceptance, scopes + I1–I8, **two-laptop relay**, 9B rows, second fixture |
| **S4** | 13–14 | 2 | Stretch (qmd / Basic Memory as B's tool, Claude Code / Codex row ≤ $20, the 06 poisoning run); package, runbook, results table; upstream PR only if G0 + written DCO |
| — | 15 | 1 | Interview sprint: replayed demo, 2 × 2 delivery plot, kill-rule verdicts on a whiteboard; the write-up (§14) |

Sessions 4 + 4 + 2 + 2 + 2 + 1 = 15; hours 5 + 5 + 3 + 3 + 2 + 1 = 19.

**S0 reproduction rule (pre-registered).** (a) gitleaks on CredData: ≥ 98% of labelled lines downloaded, F1 within ±0.03 of 0.3336 and TP within ±5% of 1,120, with the pinned v7.x binary in WSL2 ext4. (b) LongMemEval `flat-bm25`: `recall_all@5` and `ndcg_any@5` within ±2 pp of the paper's row (read from the PDF in session 1), on the original file; on cleaned data only it is a port delta. S0 is met if either reproduces; otherwise both are port deltas (08's convention), marked "not reproduced".

**Tiers.**
- **Core** ships regardless: S0 (both targets attempted); mdsieve's builder, converter, harness and scorers; noise filter, indexer/watcher, `search_md`, secret gate and leak scan, MCP server/CLI; claims 1–4, with claim 3 including B−p and the spurious-call set; the week-6 demo.
- **Planned**, cut in this order at week 8 if behind: (1) second-fixture row; (2) 9B rows; (3) the vendor-shelf row; (4) qmd and Basic Memory retrieval rows; (5) create acceptance; (6) scopes, I1–I8 and the relay; (7) the private real-query arm.
- **Stretch:** qmd and Basic Memory as B's tool; the Claude Code / Codex row; write-path poisoning persistence in 06's harness; 06 Customs pinning of the MCP server.

**Spec freeze.** The reference tool follows memtidy as of `7708d37` (2026-10-19). Changes after the mentor's planned 2026-11-15 direction review enter only in S3, and only if they leave frozen test configs untouched.

## 11. Risks & Mitigations

| Risk | Mitigation |
|---|---|
| G0, credit and conflict of interest: the mentee works next to the mentor's pre-release tool | §2 split: mdsieve is G0-independent, under the mentee's name; upstream only after G0 + written DCO; create/scope publication is the mentor's written call; write-ups split, with order and mutual citation agreed before S0; results published regardless |
| The mentee's employer IP clause covers outside OSS work | Checked in writing before S0; own devices, time and accounts; public data only; if blocked, the work stays private research under "Projects" |
| memtidy changes direction at the 2026-11-15 review | mdsieve evaluates any md-context system; spec frozen at `7708d37`; only §1 motivation, §13 and the upstream path change |
| 4B at the floor; 9B does not fit 6 GB | S1 floor gate with A0 and oracle-path controls; single-location mounts; 9B fallback grid on the Mac (~62 Mac-h); claim 4 needs no agent (§8.3) |
| Ceiling: LME-md too easy for grep (third-party BM25 R@5 ≈ 86% on `_S`) | Five mount locations next to vendor noise; `_M`-size histories rung; tokens still compared; 3a labelled ceiling-limited (§8.3) |
| The agent ignores `search_md` | Uptake floor with a pre-written wording rung; claim 3b reports uptake either way |
| Tool-call parse failures (template, #20198) | Gate row 1; built-in template first; argument validator; lazy GBNF; failures counted, never silently retried |
| Qwen3.5 hybrid prefix-cache misses (#22384, #24587, #24714) | Total prompt tokens (cache-independent) are the primary cost; append-only prompt, fixed tool order; gate row 3; 2507 fallback |
| By-construction wins (noise at known paths, decoys) | Filter frozen on filter-dev marketplaces, scored on held-out ones; LME-md primary; decoys set aside; stock rg as rung 0 (§5.1, §9.4) |
| S0 targets drift (CredData repos deleted since 2022; LongMemEval cleaned re-release; script environment) | Two co-targets; ≥ 98% coverage rule; ±2 pp band; WSL2, NumPy < 2, ijson; port deltas declared (§5.4) |
| Crowded field (qmd, Basic Memory, claude-mem; grep-vs-vector papers) | Cited up front, run as baselines; the delta is noise share, filter vs index, delivery and uptake on a laptop open-weight agent; a powered null is a deliverable |
| Corpus licences (LoCoMo CC BY-NC; LongMemEval data card unstated; CredData per-repo; marketplaces mixed; Basic Memory AGPL) | LoCoMo excluded; builders and converters shipped, never data; CredData local only; AGPL tool run out of process |
| Secrets or private text leak (fixture, cassettes, private arm) | Fake keys only; leak scan of every channel, including git objects; the private arm never leaves the mentee's machine; aggregates only |
| Compute overrun | §8.4 buffer (~31%) and cut order; the n-rule pre-costed (+27 GPU-h); tokens keep cut cells comparable |
| Sibling overlap (03 retrieval, 06 MCP/injection, 07 harness) | Non-Goals; no long-context arm; harness frozen; poisoning only in 06's harness (§2) |

## 12. Résumé Bullets (experience voice; numbers filled in as sprints land)

- Built **mdsieve**, an Apache-2.0 benchmark for agent markdown search: a synthetic home with **X** plugin md files from held-out public marketplaces, a LongMemEval-to-md converter and replayable cassettes; ran **X** judge-free episodes of a local Qwen3.5-4B agent on a **6 GB laptop GPU**.
- Measured **agent search cost at equal accuracy under plugin noise**: a noise-filtered SQLite FTS5 tool matched the grep loop's success (**X%** vs **X%**) with **X%** fewer prompt tokens at **X%** noise, and showed **[the noise filter alone recovered X% of the saving | the index added X% beyond the filter]**.
- Showed **the tool interface decides small-agent behaviour**: result format (inline snippets vs ranked paths) shifted success **X points**, and a one-line pointer moved search-tool uptake **X% → X%** at **X%** spurious calls; set the tool's default format from a pre-registered test.
- Ran a **pre-registered three-valued vector kill rule** (Qwen3-Embedding-0.6B dense and hybrid RRF vs FTS5/rg, n = 210 per stratum, Holm-adjusted): Recall@10 moved **X / X / X points** on keyword / paraphrase / zh→en queries, keeping vectors for **[strata]**, after reproducing gitleaks' published CredData F1 within **X** *(append "and LongMemEval's BM25 retrieval row within X pp" only if it reproduced)*.
- Shipped an **MCP/CLI md search tool** with a deterministic noise filter (F1 **X** on held-out marketplaces) and a write-path secret gate: **0** leaked substrings across tool output, index, logs, git objects and cassettes, and **X** false positives per 1k md lines vs **X** for gitleaks. *Append "; added a create path with cross-scope duplicate checks (**X%** recall at a **2%** false-block rate) and git-backed scopes (0 violations in **X** property-test sequences across two machines)" only if the Planned tier ran.*

**Rule: the verb follows the measured sign.** A refuted claim's bullet is reworded to the measured result ("showed grep matched the index below **X%** noise"), never silently deleted.

### Placement Note

These bullets go under EXPERIENCE **only with a real organization and a real engagement**:
1. **The mentoring organization as employer**, with a truthful title (`ML Engineering Practicum (mentored project)`) and an engagement record; "Intern" only if the organization issued that title in writing and will confirm it on a reference check.
2. **A real collaboration**: memtidy as a public open-source project, only after G0 clears, with written contributor terms (e.g. DCO) and merged public commits. memtidy is the mentor's product, so contributions to it are listed as merged commits, never as employment.
3. **By default, an open-source project entry**: mdsieve, the mentee's own public repo, with real commit history, the results table and the demo video (benchmark data excluded).

Never invent an employer or dates: North American employers verify employment, and a fabricated entry can get an offer rescinded. A public repo with a reproducible results table survives scrutiny better than an unverifiable internship line.

## 13. Why This Reads as Real Company Work

- **Cost at equal accuracy is a product SLO**: the question a coding-agent team asks before shipping a search tool, answered with a non-inferiority margin and a token ratio.
- **Attribution before shipping**: the filter-vs-index verdict says which component earns its maintenance cost; "the filter suffices" is a cheaper product.
- **Build vs buy, measured**: qmd, Basic Memory and rg run in the same harness and cassettes.
- **The decision comes before the data**: vectors and the default result format ship only through pre-registered tests, and a 50-query rule's ~18-point MDE is stated up front.
- **Governance in the write path**: secrets are redacted before index and return, every channel is leak-scanned including git objects, and shared writes are proposal-only (L34).

## 14. Interview Leverage

- **ML system design** ("design search over an agent's notes and memory files"): filter vs index, delivery format, per-result token budget, scope ranking, uptake.
- **LLM serving:** hybrid KV arithmetic, prefix reuse with append-only tool results, cold vs warm prefill, tool-call parsing.
- **Search/IR:** FTS5 BM25 constants vs rank_bm25's, CJK trigram, RRF, query strata, why 50 queries detect only ~18 points.
- **Experimentation:** non-inferiority margins, equivalence (TOST) for "filter suffices", cluster bootstrap, Holm, A/A floor, MDE vs pass bar.
- **Coding and security:** an MCP server, a secret gate and a leak scan over git objects, watcher ghost-hit tests, Hypothesis properties.
- **Behavioral:** the pre-declared week-2 fallback; "the filter, not the index, did it"; a credit split agreed in writing up front.
- **Drills** (paraphrased themes from public agent-engineering interview reports): *"Your agent has a search tool and still greps everything. How do you find out why, and what do you change?"* → §7.2, §9.3 claim 3; *"Design search over a user's local files for a coding agent. When would you add embeddings, and how would you decide?"* → §7.1, §7.4, §9.3 claim 4; *"Show that your new tool cut cost without hurting accuracy."* → §9.3 claim 1; *"How do you keep an agent's memory from leaking secrets?"* → §7.1.
- **Write-up:** one post under the mentee's own byline (interview sprint, §10), working title *"Tens of thousands of plugin files and one note: what grep costs a 4B agent, and what pays it back"*: hook, context, the work (failure before fix: a grep loop flooded by vendor matches at the 2,000-token cap, then the filter and the index separated, §9.2), what I'd do differently (the pre-declared null or fallback: a "filter suffices" or "inconclusive" verdict, §9.3, or "not testable on 6 GB", §8.3), takeaways. Every number traces to the results table; vendor numbers are labelled. It covers the public benchmark only; memtidy's own validation and the n = 1 scan stay the mentor's to publish (§2).

## 15. References

- memtidy one-pager and decision log, rows L14, L30, L33–L37 (private repository, commit `7708d37`, 2026-10-19; no URL)
- LongMemEval (Wu et al., ICLR 2025): https://arxiv.org/abs/2410.10813 · code (MIT; README links only the 2025-09 cleaned files, checked 2026-10-22): https://github.com/xiaowu0162/LongMemEval · cleaned data: https://huggingface.co/datasets/xiaowu0162/longmemeval-cleaned · LongMemEval-V2: https://github.com/xiaowu0162/LongMemEval-V2
- CredData (Samsung; benchmark table "Updated: April 2022", checked 2026-10-22): https://github.com/Samsung/CredData · runner: https://github.com/Samsung/CredData/tree/main/benchmark
- Secret scanners: gitleaks (v8.30.1, 2026-03-21): https://github.com/gitleaks/gitleaks · Betterleaks: https://github.com/betterleaks/betterleaks · detect-secrets: https://github.com/Yelp/detect-secrets
- Is Grep All You Need? (Sen et al., 2026): https://arxiv.org/abs/2605.15184
- Deep agentic vs semantic search on repository QA (2026): https://arxiv.org/abs/2608.01507
- LongMemEval-S BM25 rows (third-party, self-published): https://github.com/rohitg00/agentmemory/blob/main/benchmark/LONGMEMEVAL.md (checked 2026-10-22) · https://unpkg.com/open-memex@0.8.1/docs/retrieval-ablation-study.md
- SWE-ContextBench: https://arxiv.org/abs/2602.08316 · ContextBench: https://arxiv.org/abs/2602.05892
- Context files (Gloaguen et al., 2026): https://arxiv.org/abs/2602.11988 · Lulla et al. (2026): https://arxiv.org/abs/2601.20404 · Vercel evals (January 2026): https://vercel.com/blog/agents-md-outperforms-skills-in-our-agent-evals
- Vendor retrieval claims (context only): Cursor semantic search: https://cursor.com/blog/semsearch · Anthropic Contextual Retrieval (2024-09-19): https://www.anthropic.com/news/contextual-retrieval
- MemoryAgentBench (MIT, reserve corpus): https://github.com/HUST-AI-HYZ/MemoryAgentBench · LoCoMo (CC BY-NC 4.0, excluded): https://github.com/snap-research/locomo
- md tools run as baselines (stars on 2026-10-22): qmd (MIT, 30,551★): https://github.com/tobi/qmd · Basic Memory (AGPL-3.0): https://github.com/basicmachines-co/basic-memory · claude-mem: https://github.com/thedotmack/claude-mem · krimto: https://mcpservers.org/servers/krimto-labs/krimto
- Search internals: SQLite FTS5 `bm25()` (k1 1.2, b 0.75): https://github.com/sqlite/sqlite/blob/master/ext/fts5/fts5_aux.c · tokenizers: https://github.com/sqlite/sqlite/blob/master/ext/fts5/fts5_tokenize.c · sqlite-vec: https://github.com/asg017/sqlite-vec · ripgrep: https://github.com/BurntSushi/ripgrep · rank_bm25: https://github.com/dorianbrown/rank_bm25 · ranx: https://github.com/AmenRa/ranx · datasketch: https://github.com/ekzhu/datasketch · Hypothesis: https://github.com/HypothesisWorks/hypothesis
- MCP 2026-07-28 (Roots deprecated; deterministic `tools/list` order): https://github.com/modelcontextprotocol/modelcontextprotocol/tree/main/schema · https://blog.modelcontextprotocol.io/posts/2026-07-28/
- Agent memory locations and startup loads (checked 2026-10-22): https://code.claude.com/docs/en/memory · https://github.com/openai/codex/blob/main/codex-rs/core/src/agents_md.rs · https://github.com/google-gemini/gemini-cli/blob/main/docs/tools/memory.md · Claude memory tool: https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool
- LOO trees (Planned): https://github.com/kubernetes/website · https://github.com/facebook/docusaurus · https://github.com/mdn/content
- Memory poisoning (06 stretch): Bad Memory: https://arxiv.org/abs/2607.14611
- Models: https://github.com/QwenLM/Qwen3.5 · https://huggingface.co/Qwen/Qwen3.5-4B · https://huggingface.co/Qwen/Qwen3.5-9B · https://huggingface.co/Qwen/Qwen3-4B-Instruct-2507 · https://github.com/QwenLM/Qwen3-Embedding · https://huggingface.co/unsloth
- llama.cpp: https://github.com/ggml-org/llama.cpp/issues/22384 · https://github.com/ggml-org/llama.cpp/issues/24587 · https://github.com/ggml-org/llama.cpp/issues/24714 · https://github.com/ggml-org/llama.cpp/issues/20198 · server README: https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md
