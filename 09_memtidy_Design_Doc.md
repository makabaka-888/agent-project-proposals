# memtidy — Design Document

**Plugin Noise, Result Format and a Powered Vector Kill Rule: What It Costs a 4B Agent to Find One Note Within a 6 GB VRAM Budget**

| | |
|---|---|
| Status | Proposal |
| Target duration | ~10–12 weeks (19 mentoring hours / 15 sessions + independent work) |
| Platforms | **Windows 11 laptop** (16 GB RAM, RTX 3060-class, **6 GB VRAM assumed**, 8 GB if confirmed; WSL2, the Windows Subsystem for Linux, for S0): every scored run unless the floor-low fallback moves claims 1–3 to the Mac. **Apple Silicon Mac** (16 GB, no CUDA): 9B rows, generation, tests, demo. One repository. |
| Deployment | 100% local; no rented servers or paid GPU cloud. Run-time network: a LAN git remote and the optional Claude Code / Codex row (≤ $20 total) |
| Core stack | llama.cpp `llama-server` (pinned) · Qwen3.5-4B / 9B, Qwen3-4B-Instruct-2507 GGUF (llama.cpp's model format) · Qwen3-Embedding-0.6B · SQLite FTS5 full-text search, sqlite-vec · ripgrep · Model Context Protocol (MCP) Python SDK · LongMemEval · CredData, gitleaks · Hypothesis · 05 cassettes |

**Terms.** **memtidy**: the mentor's planned Apache-2.0, local-first md find/create/share tool, no code yet (as of commit `7708d37`, 2026-10-19); **Lnn**: line nn of its decision log; **G0**: a pre-release check the mentor must clear before memtidy is public (L14). **mdsieve**: the mentee's benchmark and harness (working name). **A**: the agent with `grep`, `glob`, `read`; **A+F**: A over noise-filtered files; **B**: A+F + `search_md` (FTS5 BM25 + metadata filters) + a one-line pointer; **B−p**: B without the pointer. **Noise share**: vendor, plugin and cache md as a share of all md under `$HOME`. **Delivery**: *inline* (path, heading, ≤ 3-line snippet) or *paths*. **LME-md**: LongMemEval (LME) histories, one md file per session. **Needle**: a fictional fact in one self-written file; **decoy**: its stale vendor or cache copy.

---

## 1. The Product Thesis

An agent looking for last week's note greps a home directory where most md is not the user's: one heavy user's laptop scan (n = 1, unpublished, the mentor's; motivation only) found 51,118 md files, ~370 (3.3% of bytes) self-written, and 62% of plugin bytes exact copies.

The fix is contested. Claude Code moved to agentic grep on "vibes" and internal benchmarks (podcast, May 2025); Cursor found semantic search plus grep best (vendor blog, ≈ Nov 2025), then dropped its index for Instant Grep (Cursor docs, 2026; L35). With LongMemEval questions as files, grep beat vectors under inline delivery but file delivery could erase the lead (arXiv 2605.15184, metric unverified); on repository QA semantic search won (arXiv 2608.01507, LLM-judge scored); both from search summaries, re-checked in S0. None varies a home directory's vendor share or splits the saving between filter and index.

memtidy's planned deterministic pieces (§7.1) are the **tool side**; a local open-weight agent is the subject of every agent claim. **mdsieve treats finding a note as an agent cost problem: what a local 4B agent pays to find one file among [N] plugin markdown files, and which part of a search tool pays it back.**

> **Engineering thesis: where vendor markdown is most of the files, a small agent's search cost is a noise problem first and a retrieval problem second.** A noise filter plus a lexical index with metadata filters keeps a 4B agent's success within 8 points of its grep loop's (non-inferior) for fewer tokens; vectors pay off only where lexical overlap is absent.
>
> *Pre-registered test:* rejected if B fails 1a; "noise first" holds if the filter's share of the saving, log(A / A+F) ÷ log(A / B), has a lower bound above 0.5, is rejected if its upper bound is below 0.5, and is otherwise "inconclusive" (§9.3).

**Baseline (measured, not a claim): grep@noise**: Qwen3.5-4B on Windows (any card), arm A, inline, full noise, dev and test; every margin uses it.

**Floor gate (S1, pre-registered; point estimates, 100 LME-md dev questions + 40 needles):** grep@noise success 30–85%; median ≥ 3 calls in A; parse failures ≤ 5% of calls; `search_md` in ≥ 70% of B episodes; no-tools path guessing (A0) ≤ 10%; index update ≤ 10 s after 100 edits; an oracle-path control only warns. Otherwise the cost claims have nothing to test (fallbacks: §8.3).

Four claims, each with a pre-registered rule (§9.3):
1. **Headline: success within 8 points of grep (non-inferior) for fewer tokens at full noise, and which part pays.** **1a:** B vs A on 200 LME-md test questions: success non-inferior, fewer total prompt tokens and tokens per successful find (token minimum detectable effect, MDE, ≈ 11%). **1b** (secondary): B vs A+F: **index adds**, **filter suffices** (±15%) or **inconclusive**, plus the filter's share. Needles (80) are secondary; decoys set aside.
2. **Noise is what grep pays for.** The A − B log-token gap grows from 0% to full noise (difference-in-differences lower bound > 0; MDE ≈ 0.17 log points), on marketplaces the filter never saw.
3. **The interface decides whether a 4B uses the tool.** **3a:** tool × delivery interaction on success (MDE ≈ 12.5 points); B's better delivery becomes the default (a tie within ±8 points goes to fewer tokens). **3b** (secondary): B vs B−p (L36's hint) on `search_md` call rate (MDE ≈ 9 points); the pointer stays only if spurious calls on 80 context-free items are ≤ 20%.
4. **Vectors earn their place only where lexical overlap is absent (a powered kill rule).** No agent: Recall@10 of the best dense / hybrid reciprocal rank fusion (RRF) arm minus the best of FTS5 / rg, Holm over four strata (keyword, natural low-overlap, paraphrase, zh→en: Chinese query, English file): **keep** (lower bound > 0 and Δ̂ ≥ 10 points), **kill** (upper bound < 10) or **inconclusive**; low-overlap is counted in S0 and descriptive below 100 test queries. L35's 50-query rule runs on the private arm (Planned; MDE ≈ 18 points).

**Why it is surprising, and fun.** The industry moved to grep and the papers disagree, yet none varies noise share; the filter alone capturing the saving, or a pointer deciding uptake, would be news. Demo: two agents, one question, live token meters.

## 2. Goals & Non-Goals

mdsieve is the portfolio's **agent memory & context files** project (context engineering).

### Goals
- **Reproduce one published number in S0** (§5.4).
- **mdsieve**, a **reference find/create tool** for memtidy's spec (L33–L36; a generic baseline if the mentor objects), and **claims 1–4**, published whether or not they favour memtidy.

### Who builds what (proposed; the mentor can change it before S0)

| Piece | Owner | Licence / gate |
|---|---|---|
| mdsieve: builder, converter, query sets, harness, scorers, results, for any md-context system (rg, qmd, Basic Memory, the reference tool, memtidy later) | Mentee | Apache-2.0, public from S0, G0-independent; names memtidy or cites its log only after G0 or the mentor's written consent |
| Reference tool: filter, index, `search_md`, secret gate, MCP/CLI; create and scopes (Planned) | Mentee | Public mdsieve baseline by default, described generically; if the mentor objects in writing before S0, a generic FTS5 + filter baseline outside memtidy's spec replaces it, so claims 1–3 keep a public arm. Upstream only after G0 and written contributor terms (e.g. Developer Certificate of Origin, DCO, sign-off) |
| memtidy: product, name (L30), roadmap, validation, write-ups | Mentor | Untouched |
| Employer IP clause for the mentee's outside open-source work | Mentee | Recorded in writing before S0 |

**Data and conflict of interest.** The mentor's data is never used; the n = 1 aggregates are motivation only. The mentor owns memtidy (possibly commercial later), writes its spec, mentors this analysis and plans a separate grep-vs-index evaluation of memtidy, so `prereg/` is frozen by hash before test opens. The mentee dates mdsieve's public results, by the end of S4 whatever memtidy's status; the mentor's post follows or cites them (agreed in writing before S0; more in §11).

### Non-Goals
- Cloud sync, hosted team servers, adoption or pricing claims, training, redistributing corpora.

### Relation to sibling proposals
- **03 Spelunker** owns retriever choice on web corpora and memory vs long context (NightShift's best experiment), so no "all md in the prompt" arm here.
- **05 BlackBox** owns record/replay and A/A-calibrated CI; mdsieve's runs are its cassettes, and its A/A floor uses 05 §7.9.
- **06 Ctrl-Z** owns the MCP gateway and injection defence; mdsieve's server is a possible Customs target, `base_version` follows 06 §7.3's read-set version check, and write-path persistence is a Stretch run there (§10).
- **07 HarnessForge** owns harness search; mdsieve fixes the loop, budgets and task card and compares a pre-registered grid of tool menus, delivery, a one-line pointer and noise share.
- **02 Shellwright** owns `notes` and compaction, the in-agent analogue.

### Why NightShift's rejection (IDEATION.md, "Why Notable Ideas Were Not Selected") does not apply

| NightShift reason | mdsieve's answer |
|---|---|
| Crowded (Mem0, Zep, Letta) | More so now (qmd, claude-mem, Basic Memory), so no better-memory claim: the MCP server is the tool under test; the claims are cost under noise, attribution and interface |
| Headline needs a swapped-in local judge | Judge-free headlines: gold path, `llama-server` token counts, Recall@10 on `answer_session_ids` |
| Ingestion takes 1–3 nights | FTS5 build ≤ 10 min, history mount ≤ 10 s (gated); only claim 4 embeds (~8 GPU-h budgeted, possibly over one night; measured in S0), and no agent arm waits on it |
| "Sleeps at night" hook | No idle consolidation or forgetting; nights are compute scheduling |

## 3. Alignment With Prior Experience

| Prior experience | mdsieve module |
|---|---|
| **CAS**: cascaded binary-encoding retrieval; Precision@K / latency / storage over 20+ configs | **Retrieval bench**: Recall@10 vs p95 latency vs index size |
| **Changsheng**: AutoML hyper-parameter agent; backtest promotion gates | **Kill rule as a promotion gate** for vectors and the default format |
| **DiDi**: A/B testing | **Paired agent A/B**: variance pilot, A/A floor, non-inferiority, equivalence |
| **Sina Weibo**: scheduled SQL ETL, standardised metrics, alerting | **Indexer/watcher**: freshness lag, ghost hits, a leak-scan alert in CI |
| **College Application Agent**: embedding-based retrieval (EBR) behind FastAPI; LangGraph Writer–Critic | **Dense arm, MCP/CLI service**; create's propose → check → write loop |
| **Music Genre**: principal component analysis (PCA), K-means (clustering) | **Noise-filter error analysis**: clustering misfiled md |

**Not prior experience:** memtidy (the mentor's, no code) and Enclave (a stalled earlier mentor scaffold; only its RRF pattern, tests and socket guard are reused).

**Interview one-liner:** *"At CAS I measured a retrieval scheme's accuracy–latency–storage frontier. With mdsieve I measured what a 4B agent within a 6 GB VRAM budget pays to find one note among [N] plugin markdown files: [the noise filter alone cut tokens X% and the index X% more | the index added nothing beyond the filter | grep matched the index at full noise, and I traced why], and result format [moved success X points | made no detectable difference]."*

## 4. Stack Choices — and Why

### 4.1 Models

| Role | Model | File / quant | KV (key-value) cache per token, f16 | Why |
|---|---|---|---|---|
| Agent | Qwen3.5-4B | Q4_K_M 2.74 GB | 32 KiB (8 of 32 layers attend) + ~48 MiB Gated DeltaNet state per slot | Fits 6 GB at 16k |
| Mac rows | Qwen3.5-9B | Q4_K_M 5.68 GB | 32 KiB | 9B rows, query generation; floor fallback |
| Standard-attention fallback | Qwen3-4B-Instruct-2507 | Q4_K_M 2.5 GB | 144 KiB | Prefix-cache safety (§11) |
| Dense arm (claim 4) | Qwen3-Embedding-0.6B (1024-d, multilingual) | fp16 ~1.2 GB | — | zh→en and paraphrase strata |

**Why Qwen3.5:** the newest Qwen weights that fit 6 GB (2026-03-02; Apache-2.0 per Hugging Face metadata, LICENSE checked in S0); thinking off, one dev rung on. **KV arithmetic:** 8 attention layers × 4 KV heads × 256 × 2 (K, V) × 2 bytes = 32 KiB/token; 2507: 36 × 8 × 128 × 2 × 2 = 144 KiB/token. *Sizes from unsloth GGUF listings, architecture from third-party configs; both confirmed from `config.json` in S0.*

### 4.2 Runtime
One pinned `llama-server` build (CUDA / Metal, `--jinja`, `-fa on`); vLLM has no native Windows build. llama.cpp #22384, #24587 and #24714 report hybrid re-prefill, hence an append-only, byte-stable prompt, fixed tool order and measured re-prefill (`preserve_thinking`, suggested on #22384, is checked in S0). #20198 (object-typed tool `arguments`), fixed by #20213 (2026-03-08), gets a regression validator.

### 4.3 Libraries

| Concern | Choice | Why |
|---|---|---|
| Index, vectors, tests | stdlib `sqlite3` FTS5, `bm25()` (k1 1.2, b 0.75, fixed), unicode61 + trigram for Chinese, Japanese and Korean (CJK); `watchdog`; sqlite-vec; `datasketch`; Hypothesis | Lexical first (L35); one file, rebuildable from md |
| Grep | ripgrep pinned; `--hidden` in A, stock defaults as rung 0 | rg skips hidden dirs by default |
| MCP | MCP Python SDK, stdio, tools only | Spec 2026-07-28 deprecates Roots, so scope is a parameter; fixed `tools/list` order caches |
| Baselines | rg (Core); qmd (MIT) and Basic Memory (AGPL-3.0, out of process), Planned; gitleaks v7.6.1 (S0) and v8.30.1; Betterleaks; detect-secrets | Build vs buy; a cut row is "not run" |

*Versions and spec dates as of October 2026; re-verified in S0.* **Prior art:** §1's papers; LongMemEval-V2 (a coding agent over trajectory files beat the best retrieval-augmented baseline; LLM-judge scored; search summary, re-checked in S0); SWE-ContextBench ("free retrieval … can even hurt"); qmd, Basic Memory, claude-mem. **mdsieve adds** a laptop open-weight agent, noise share, filter-vs-index attribution, delivery × tool, uptake and a powered kill rule.

## 5. Benchmarks & Datasets

### 5.1 Synthetic home fixture (public; ground truth by construction)
- **Builder.** A script creates a clean `$HOME` per machine and installs public plugin and skill marketplaces (Claude Code, Codex, Cursor, Copilot, Gemini CLI; availability checked in S0) at pinned commits, plus `.cache` web-page md and third-party clones under `~/code`.
- **Held-out split.** Marketplaces split by name: **filter-dev** (~¼ of files; rules written and frozen here) and **held-out** (alone scored; includes an unseen vendor layout).
- **Self-written set** (~400 files): public CLAUDE.md, AGENTS.md, `.claude/rules` and `memory-bank/` files from repos under Open Source Initiative (OSI) licences; hard cases (`/init` output, same-named `README.md`, vendor copies of user notes); 120 agent needles, half the test ones with 1–3 decoys.
- **Claim-4 items:** 270 retrieval-only needles and 270 `README.zh-CN.md` / `README.md` pairs from OSI-licensed repos. All sources at pinned SHAs.
- **Noise levels:** 0%, 90% and full (the fixture's natural share, measured in S0). Only builder, lists and hashes are published.

### 5.2 LongMemEval as md (LME-md)
470 questions (500 minus the 30 abstention items, as the official retrieval eval does); `_S` histories of ~115k Llama-3 tokens and ~40 sessions; gold = `answer_session_ids`. Code MIT, data licence checked in S0; the cleaned 2025-09 files are user-downloaded, never redistributed. One md file per session, mounted by a fixed-seed rule at one of five locations (Claude Code auto memory, Codex memories, `~/notes/`, a project's `docs/` or `memory-bank/`), a reported stratum. Splits: pilot 60, dev 100, test 200, spare 110. LoCoMo (CC BY-NC 4.0) excluded.

### 5.3 Query sets (frozen by hash in `prereg/` at S0 end)

| Set | Source | Dev / Test |
|---|---|---|
| Agent questions, primary | LME-md, stratified by type | 100 / 200 (+60 pilot) |
| Agent needles, secondary | Keyword-style; test 40 clean + 40 decoy (set aside) | 40 / 80 |
| Context-free items (3b) | Dev questions with the gold session pasted in; any search is spurious | 80 |
| Retrieval, low-overlap | LME questions sharing 0–1 content stems with every gold session (natural); counted in S0 and given every qualifying question first | 60 / the rest (< 100: descriptive) |
| Retrieval, keyword | The remaining LME questions sharing ≥ 2 content stems with a gold session | 60 / ≤ 210 |
| Retrieval, paraphrase | Queries about retrieval needles, rewritten by the 9B to share no content stem with the gold file (~330 generated, 270 kept; *generated*); 60 hand-checked by the mentee, blind to arm (≥ 90% answerable) | 60 / 210 |
| Retrieval, zh→en | A sentence of a public `README.zh-CN.md`; its paired `README.md` is gold | 60 / 210 |
| Private arm (Planned) | ~50 lookups from the mentee's transcripts, failed and successful 1:1; gold labelled blind to arm | aggregates |

**Claim 4 corpus**, per query and the same for every arm: the unfiltered held-out fixture (with retrieval needles and English READMEs, no `README.zh-CN.md`) plus its LME-md history (paraphrase and zh→en queries: one fixed-seed LME history each). Each stratum reports its lexical floor; paraphrase is labelled by-construction. **Private material** (private arm, §7.1 audit): the mentee's personal, non-employer files only, screened before labelling, never sent to a commercial API or a published cassette; aggregates only.

### 5.4 S0 external reproduction (pre-registered target: CredData)

| Source | Metric | Published | Local |
|---|---|---|---|
| CredData README (Samsung, vendor-authored, April 2022; train + test) | gitleaks true / false positives, precision, recall, F1 | 1,120 / 1,011 / 0.5256 / 0.2444 / 0.3336 | **S0** |
| LongMemEval (Wu et al., ICLR 2025), flat-BM25 session row | `recall_all@5`, `ndcg_any@5` | Fixed in session 1, before any run (not pre-registered here) | **S0** |

**(a) CredData** (73,842 labelled lines, 4,583 true). `download_data.py` fetches 297 repos at pinned commits and needs Linux (filenames fail on NTFS, the Windows file system), so it runs in WSL2 ext4. The wrapper calls the pre-v8 CLI and parses v7 report fields, so v7.6.1 (2021, the last v7) is pinned by checksum, a declared assumption since the README omits the version; v8.30.1 is a context row. CPU, about an evening; data stay local.

**(b) LongMemEval.** `run_retrieval.sh <file> flat-bm25 session` in WSL2 (bash ≥ 4, CPU, NumPy < 2). The table, file (`_S` or `_M`) and hash are recorded before `prereg/` freezes; if the original pre-2025-09 file is gone, (b) is a port delta on the cleaned file. FTS5 BM25 (k1 1.2) vs rank_bm25 (k1 1.5) is reported beside it.

### 5.5 Other data
**Secrets (Core):** ≈ 2,000 fakes of ≥ 20 key types (documented prefix + random body) in prose, code, tables, frontmatter, URLs and base64, seeded only in a separate secret-gate corpus and the relay's pasted log, never in the agents' fixture (A's raw `rg` output bypasses the gate); hard negatives (`${VAR}`, SHAs, UUIDs, example keys); a ≈ 100k-line false-positive corpus. **Planned:** leave-one-out (LOO) trees from kubernetes/website, docusaurus (CC BY 4.0) and mdn/content (CC-BY-SA 2.5); ≈ 600 duplicate probes, half across scopes. **Demo:** one dev question where A and B disagree, replayed from cassettes, never used for a claim.

## 6. Architecture

```
ALL LOCAL; RUN-TIME NETWORK = ONE LAN GIT REMOTE (relay) + OPTIONAL CAPPED API ROW

FIXTURE (pinned SHAs)            TOOL SIDE (deterministic, no LLM)
 synthetic $HOME -------------> NOISE FILTER ---> INDEXER/WATCHER ---> FTS5 + metadata
  |- vendor md: held-out         path rules,       mtime + hash,        path/type/scope/
  |   marketplaces, caches       vendor hashes,    H1-H3 chunks         mtime [+ vec: claim 4]
  |- own md, needles, decoys     foreign clones   SECRET GATE: redact before index/return
  |- LME-md at 5 memory dirs                      CREATE + SCOPES (Planned): git, PRs
  `- scopes = local git repos                               |
                                       MCP SERVER (stdio) == CLI (one library)
                                                            |
LOOP (held fixed; tool menu and pointer vary per cell) -----+---------------------------
 tasks --> AGENT LOOP (Qwen3.5-4B, llama-server) --> A    grep / glob / read (rg)
              |                                      A+F  same tools, filter applied
              |                                      B    A+F + search_md + pointer
              |                                      B-p  B without the pointer
              |                                      + A/B as paths; A/B at 0%, 90%
              `--> cassettes (05) --> SCORERS: gold path, prompt/cache/predicted tokens,
                                      calls, uptake split, leak scan --> STATS
RETRIEVAL BENCH (claim 4, no agent): rg | FTS5 | dense | hybrid RRF [| qmd | Basic Memory]
S0 (CPU, WSL2): gitleaks v7.6.1 on CredData | LongMemEval flat-bm25, official scripts
```

**Walk-through.** The tool side indexes the haystack through filter and secret gate. One fixed loop runs ten pre-registered cells (§8.4: four tool menus at full noise, inline; A and B as paths; A and B at 0% and 90%), every call a 05 cassette scored against gold. Claim 4 bypasses the agent.

## 7. Component Design

### 7.1 Tool side (deterministic; acceptance metrics, not claims)
memtidy's 2026-10-19 direction (L33; one-pager), as mdsieve reads it (spec freeze, §10): one always-on FTS5 index over all md folders, plugin and cache noise filtered out, with path, frontmatter, scope and recency filters; vectors only if they recall ≥ 10 points more than rg on 50 real queries (L35, paraphrased); the old scan reused as noise filter, a secret gate on create and promotion, render cut to a one-line hint (L36). L35–L36 are marked changeable; the earlier no-LLM plan is not restated after that date.

| Component | Design | Acceptance |
|---|---|---|
| Noise filter (L36) | Self-written vs vendor / cache / unclassified, from vendor-dir path rules, hashes seen ≥ 2× in vendor dirs, clones with a foreign remote and no user commits, and cache bodies opening with a source URL | Precision / recall / F1 by class on held-out marketplaces; unclassified share; a 300-file audit on the mentee's personal machines (§5.3) |
| Indexer / watcher | Watcher + per-call mtime/hash check; H1–H3 chunks prefixed "title > heading path"; external-content FTS5 table (path, scope, `type`, mtime, class) | Build times; write → searchable lag p50/p95; **0 ghost hits** after a 10k-operation rename/delete storm; **I7** rebuilt = incremental |
| `search_md(query, k=8, path_prefix?, type?, scope?, modified_after?)` | BM25 behind a token whitelist (every term quoted, since raw punctuation, e.g. `-` `.` `/` `:` `'`, and bare AND / OR / NOT break `MATCH`) with an OR fallback; hits grouped by file, scope-labelled; ≤ 2,000 tokens | Claim 4 port delta; p95 latency at 0.5k / 5k / full |
| Secret gate (L36 gates create and promotion; redaction before indexing is mdsieve's addition) | Gitleaks-compatible regex + entropy rules; `[REDACTED:<rule>]` before indexing and any return. Motivation: GitGuardian counts 24,008 unique secrets in the MCP configuration files it studied (2026 report) | Recall on ≈ 2,000 fakes (upper bound); false-positive files per 1k lines vs gitleaks v8.30.1. **Leak scan:** no fake's random body (prefix excluded, ≥ 12 characters) in `search_md` / `read_md` output over that corpus, logs, FTS5, their cassettes or git objects, beside an unseeded run's chance matches; any leak fails |
| MCP server | stdio, tools only: `search_md`, `read_md`, `list_scopes`, `create_md` (Planned); `readOnlyHint` on reads; fixed order; CLI = same library | Tool definitions ≤ 800 tokens; p95 latency; resident memory |

### 7.2 The agent loop

| Element | Specification |
|---|---|
| Observation | (1) Byte-stable system prompt, tools in fixed order; in B only, a last line `Your notes are indexed: call search_md before grepping.` stands in for L36's hint. (2) Task card: `Find the user's own note(s) that answer: <question>. Return up to 3 paths with final.` (3) Append-only tool results |
| Action | One native function call per turn, or `final(paths ≤ 3, answer?)`; an invalid call is a parse failure, never silently retried |
| Tools | **A:** `grep(pattern, path?, glob?)` (rg `--hidden`), `glob(pattern)`, `read(path, offset?, limit?)`. **A+F:** the same over the filter's output. **B:** A+F + `search_md`; **B−p:** no pointer. *Paths* delivery = `rg -l` or ranked paths |
| Termination | (1) `final`; (2) 15 tool calls, then one "answer now" turn; (3) 16k context, likewise; (4) 3 consecutive unparseable calls, a failure. A 300 s watchdog is a runaway guard, not a termination rule: logged, re-queued once, then a failure |
| Budgets | **2,000 tokens per tool result in every arm**, with a dropped-match count; ≤ 512 generated tokens per turn; thinking off; the card's non-thinking sampling except presence penalty 0 (card: 1.5), as in 05 §7.5 |
| Cost accounting | **Total prompt tokens = Σ(`prompt_n` + `cache_n`)**, primary and cache-independent; prefilled = Σ `prompt_n`; **re-prefill share** = Σ max(0, `prompt_n` − tokens appended since the last call) / Σ(`prompt_n` + `cache_n`), after 08's re-processed-token metric. First 20 episodes per cell cold, the rest warm |

The result cap makes noise expensive: at full noise an unfiltered `grep` for a common term is expected to fill its 2,000 tokens with vendor matches (measured in S0), and narrowing costs a call. `search_md` has the same cap, so no arm wins by seeing more.

### 7.3 Harness and scorers
- **Runner:** resumable, one config hash per cell, 05 cassettes (a minimal recorder in 05's format if 05 is not run). **Network-off smoke test:** Enclave's socket-guard pattern with an address-based loopback allow-list (127.0.0.1, ::1), plus one OS-level network-off run, since a Python guard misses llama-server, rg and git (03 §7.4).
- **Gold-path scorer:** paths normalised (Unicode NFC, separators, Windows case); success = ≥ 1 gold path among ≤ 3 returned; substring match on a frozen short-answer subset (≥ 50 dev items). **No LLM judge.**
- **Uptake parser:** B and B−p failures → zero-call / gold never retrieved / retrieved but not returned. Within-episode repeat searches count as redundant searches; **rediscovery** of the cross-session kind (re-finding a fact an agent wrote to md in an earlier session; 02 §7.5 flags the in-episode analogue) is reported descriptively from the agent-driven relay cassettes (§7.5).

### 7.4 Retrieval bench (claim 4)
Arms over the §5.3 corpus: *rg* (files ranked by distinct query terms matched); *FTS5 BM25*; *dense* Qwen3-Embedding-0.6B, flat cosine; *hybrid RRF* (k = 60), with fusion and tests ported from Enclave's Postgres RRF SQL; qmd and Basic Memory (Planned). The best lexical and vector arms are picked per stratum on dev. **Embedding** (estimate, measured in S0): the hash-deduplicated fixture plus up to 470 histories × ~115k ≈ 54M tokens before cross-question session dedup (≈ 3 GPU-h at an assumed ~5k tokens/s); budgeted at ~8 GPU-h, possibly more than one 5-h night.

### 7.5 Create, scopes and the two-laptop relay (Planned)
**Create** follows the one-pager: one call proposes path, name and metadata, flags near-duplicates in every visible scope, scans for secrets, writes and indexes. Placement: a layout rule, then a BM25 vote over folder profiles. Duplicates: body hash → MinHash locality-sensitive hashing (LSH) → containment over **visible** scopes only, frozen on dev at a ≤ 2% false-block rate, returning `blocked_duplicate`. Writes: personal commits with a `base_version` check (06 §7.3) or team and parent proposal branches (L34). **Acceptance:** LOO placement top-1/top-3 vs the 4B with a capped `list_dir`; duplicate recall at the frozen rate.

**Scopes (L34) and relay (S3).** Personal, team and parent scopes are local git repos cloned on both laptops from one bare remote, with a policy file standing in for git-host teams. The Windows agent writes a note via `create_md` and pushes; the Mac's agent pulls and finds it; a fake key in a pasted log never reaches the team repo. Pass: 20/20 CLI-driven relays, 0 leaks, lag ≤ 10 s; 20 agent-driven relays are reported descriptively.

### 7.6 Invariants (acceptance tests; 06 owns the pattern)
A Hypothesis `RuleBasedStateMachine` over nested scope trees checks **I1** no output reveals an unreadable scope's note or its existence; **I2** parent read-only; **I3** shared writes only as proposals; **I4** most specific scope first, labelled; **I5** stale `base_version` → conflict; **I6** secret gate before promotion; **I7** rebuilt = incremental (Core); **I8** cross-OS filename identity. Target: 0 violations in 10,000 two-clone and 1,000 Windows ↔ Mac sequences, under a parameterised policy memtidy can change.

## 8. Hardware Profiles

> **Rule: the weakest machine sets the defaults.** `portable` assumes 6 GB; `accelerated` is opt-in once an 8 GB card is confirmed.

`portable` (default): 4B, 1 slot × 16k, f16 KV; `-np 2` (~4.7 GB) is the first time fallback. `accelerated` (8 GB confirmed): the Planned 9B rows may also run on Windows (~6.8 GB with `-ub 256`, measured in S0); Core cells stay on the 4B. `mac`: 9B rows and 4B parity rows.

### 8.1 Memory budget

| GB (estimated; measured in S0) | Windows, 4B | Mac 16 GB, 9B |
|---|---|---|
| Weights | 2.74 | 5.68 |
| KV @16k, 1 slot (32 KiB/token) + DeltaNet state | 0.54 + 0.05 | 0.54 + 0.05 |
| Output buffer (248k vocabulary, `-ub 512`) + CUDA context | 0.5 + 0.3 | 0.5 |
| **Total, 1 slot / 2 slots** | **~4.1 / ~4.7** | **~6.8 / ~7.4** (Metal cap recorded in S0) |
| Fallback: Qwen3-4B-Instruct-2507 @16k, q8_0 KV (1.29) | ~4.4 | fits |

`-fa on` is pinned (without it the f32 attention-score buffer adds ~0.5 GB at 16k). Host RAM: `--ctx-checkpoints 8` (~0.4 GB), `--cache-ram 2048`. CUDA's system-memory fallback is disabled, so an overflow fails instead of spilling. The embedder (~1.2 GB) runs only with the agent server stopped.

### 8.2 Week-2 go/no-go gate
End of session 2: 30 LME-md dev questions in A and B at full noise on Windows; build times on both machines; 10 questions × {A, B} with the 9B on the Mac.

| # | Metric | Pass if |
|---|---|---|
| 1 | Tool-call parse failures, share of calls | ≤ 5% |
| 2 | Projected load at the measured **mean** episode time: weeks 6–8 / weeks 6–10 Core with the n-rule | ≤ 75 / ≤ 125 GPU-h (≈ ≤ 80 s per 4B episode) |
| 3 | Re-prefill share (§7.2) | ≤ 0.2 |
| 4 | rg latency per call on the full fixture (NTFS) | p95 ≤ 5 s |
| 5 | FTS5 full build / one history mount | ≤ 10 min / ≤ 10 s |
| 6 | Early floor read: grep@noise success | 30–85% (warning) |
| 7 | Mac 9B mean episode | ≤ 4 min (warning: above it the fallback 9B grid overruns the Mac) |

The first full night's throughput then replaces the planning rate (a 1.5 h burst misses thermal throttling).

### 8.3 Fallback ladder (declared now, so it cannot be renegotiated)

| Fails | Fallback, in order |
|---|---|
| 1 | Built-in GGUF template → newer pinned build → lazy GBNF grammar-constrained calls → 2507 (llama-server only: Ollama hides `cache_n`) |
| 2 | `-np 2` → 10 tool calls → test 200 → 150 (1a power ≈ 0.71 (0.53); tokens ≈ 13% (15%); interaction ≈ 14.5 (16.7); DiD ≈ 0.19 (0.22); the n-rule outranks this) → 2507 at 16k, q8_0 KV |
| 3 alone | Logged, no fallback: the primary cost is cache-independent and row 2 gates time |
| 4 | Windows Defender exclusion → labelled WSL2 ext4 copy → clones pruned to 25% (noise re-measured) |
| 5 | Batched inserts, write-ahead log (WAL) → trigram only for CJK files |
| Floor, low (< 30%) | One mount location (`~/notes`) → the 9B grid on the Mac → claims 1–3 "not testable on 6 GB"; claim 4 carries the doc |
| Floor, high (> 85%) | `_M`-size histories, MDEs recomputed on dev → 3a "ceiling-limited" (tokens still compared) |
| Floor, calls (median < 3) | `_M`-size histories → 1a's and 2's token claims "floor-limited" (success still compared) |
| Floor, uptake (< 70%) | Three pre-written wordings, best frozen → `search_md` listed first → "uptake-limited" (3b is the finding), never a pass |
| Floor, A0 (> 10%) | Needles at conventional paths re-placed |
| Floor, index update (> 10 s) | Row 5's rungs → per-call mtime/hash check only (watcher off) |

### 8.4 Compute budget
Planning rates: 75 s per 4B episode and 3.75 min per 9B episode, replaced by measured means. Capacity per machine: ~25 GPU-h a week (~5 h × 5 nights) over weeks 3–10 ≈ 200 GPU-h. A-only cells run sealed, results unopened until B's cells finish.

| Block (Windows unless named) | When | Episodes | GPU-h |
|---|---|---|---|
| S0 pilot + A/A floor + pilot extras (§9.3); week-2 gate | S0 | 520 | 10.8 |
| A-only test cells (A × 2 seeds, A-paths, A at 0% and 90%; × 280) + ladder rung 0 | Pre-filter, to week 5 | 1,500 | 31.3 |
| Floor gate (140 dev × {A, B}; 100 oracle; 100 one-turn A0) | Weeks 6–10 | 480 | 7 |
| Dev ladder: rungs 2–4, 6, 8; rung 9 at ~2× (1 and 5 reuse the floor gate) | Weeks 6–10 | 600 | 14.6 |
| Filter-dependent cells (A+F and B × 2 seeds, B-paths, B−p, B at 0% and 90%; × 280) + spurious-call set (80 × {B, B−p}) | Weeks 6–10 | 2,400 | 48.2 |
| Claim 4 embedding (~8, §7.4) + retrieval; rung 7 (S3) | Weeks 6–10 | 100 | ~10 (estimates) |
| Planned: LOO 4B baseline (900 short), second fixture (A, B × 100), qmd embedding | Cuttable | 1,100 | ~11 |
| Stretch: qmd, Basic Memory as B's tool (× 200) | Cuttable | 400 | 8.3 |
| **Windows total** | | **7,100** | **~141 of ~200** |
| Mac, Core: paraphrase-query generation (9B, ~330 queries; §5.3) | S0 | — | ~2 |
| Mac, Planned: 9B rows (A, B × 200) | Cuttable | 400 | ~25 (Mac **~27 of ~200**) |
| Mac, fallback only: the 8-cell 9B grid (A, A+F, B, B−p, A-/B-paths, A/B at 0%; × 200) + floor read | After floor-low | 1,800 | ~113 of ~125 left after week 5 |
| API, optional: Claude Code on Claude Sonnet 5.5, A vs B, ≤ 50 public LME-md dev questions (~10 turns over a ~20k-token cached prefix: ~$0.10–0.30 an episode at $2 / $10 per million input / output tokens, October 2026 list price; a 5-episode pilot sets n) | Any | ≤ 100 | ≤ $15 stop + $5 reserve |

**Critical path.** With the filter frozen by the end of week 5, weeks 6–10 give ~125 GPU-h for ~80 of Core work (~97 with the n-rule): a 36% (22%) margin. A floor-gate fallback that changes the fixture, mounts or histories (floor-low, high, calls or A0) reruns the A cells (~29 GPU-h), leaving ~13%; a rerun plus the n-rule invokes the week-8 cut order (§10). The ~70 due by week 8 is tight. History mounts, Hypothesis suites (~8 h), the 1,000 Windows ↔ Mac sequences (~17 h) and CPU indexing run in daytime. **API:** a pay-as-you-go key with a provider spend limit; a subscription counts at its full monthly price; Codex runs only from what remains.

## 9. Evaluation Plan

### 9.1 Areas

| Area | Metrics | Against | n / MDE |
|---|---|---|---|
| Agent cost (claims 1–3) | Success, total and prefilled tokens (cold / warm), calls, wall-clock, tokens per successful find, uptake split | grep@noise (A) | 200 LME-md + 80 needles |
| Retrieval (claim 4) | File-level Recall@10, mean reciprocal rank (MRR), nDCG@10 (normalised discounted cumulative gain), lexical floor, p95 latency, index size | best of FTS5 / rg | ≤ 210 per stratum |

### 9.2 Ablation ladder
Full noise, inline, dev, one change per rung: 0 rg stock defaults → 1 `--hidden` (A) → 2 + filter (A+F) → 3 + bare `search_md` → 4 + metadata filters → 5 + pointer (= B) → 6 paths → 8 no grep → 9 thinking on (≤ 2,048 generated tokens per turn; prefill re-measured, as the template strips earlier thinking). Rungs 1 and 5 reuse the floor gate. Rung 7 (+ hybrid) is a labelled post-hoc S3 row, run only if claim 4 keeps a stratum (query embeddings on CPU).

### 9.3 Pre-registered tests
- **Settings.** α = 0.05, power 0.8; p_d = discordant-pair share on success, σ_d = SD of the per-question log-token difference, q = discordant share on calling `search_md`; paired-binary MDE (z₁₋α/₂ + z₀.₈)·√(p_d/n) on the frozen fixture hash; paired bootstrap, 10,000 resamples over question (with history) and needle-template clusters; token MDEs use two-sided z. Decisions use Holm-adjusted p-values, one-sided at α = 0.05 for directional bounds (1a, 2, claim 4; as in 03 and 04), two-sided for 3a's interaction; printed Bonferroni intervals at α/3 (1a, 2, 3a) and α/4 (claim 4; α/3 if low-overlap is descriptive) can differ from a Holm verdict.
- **Families.** 1a, 2, 3a: Holm over 3. 1b, 3b and the format simple effect: secondary, α = 0.05. Claim 4: Holm over 4 strata (3 if low-overlap is descriptive).
- **Variance pilot (S0).** 60 questions × {A, B} × 3 seeds, plus 20 × {A+F (path-rules-only v0 filter), A-paths, B-paths} at full noise and {A, B} at 0%; A's seed-vs-seed runs give the A/A floor (05 §7.9). It re-derives 1a's MDE (planning p_d 0.2, σ_d 0.6) and checks the independence the claim 2–3 MDEs assume; claim 4's come from 60 dev queries per stratum. Power assumes one seed; seed 2 is averaged and cuttable. **n-rule:** if p_d > 0.25 or σ_d > 0.8, test grows to 300 (+27 GPU-h; the pool's cap) and seed 2 stays; above p_d 0.31 the lower power is stated (≈ 0.70 at 0.4). It outranks the §8.3 cut to 150.

| Claim | Statistic | Decision rule | Planning MDE / power |
|---|---|---|---|
| 1a | Δ success B − A; geometric-mean (GM) token ratio B/A; tokens-per-find ratio | Lower bound of Δ > −8 points **and** both ratio upper bounds < 1 | Power 0.81 at p_d 0.2, n 200 (0.66); tokens ≈ 11% at σ_d 0.6 (≈ 13%); joint ≈ 0.65 (≈ 0.53) |
| 1b | GM ratio B/(A+F); Δ success; filter share log(A/A+F) ÷ log(A/B) (cluster bootstrap) | In order: **index adds** if the ratio upper bound < 0.87 or the success lower bound > 0 (each at α/2); else **filter suffices** if the ratio's 90% CI lies in [0.87, 1.15] and A+F is non-inferior; else **inconclusive** | Two one-sided tests (TOST) power 0.90 at ratio 1 |
| 2 | Difference-in-differences of log tokens, (A − B) at full minus 0% | Lower bound > 0; 90% is a dose point | ≈ 0.17 log points (≈ 0.19) |
| 3a | Tool × delivery interaction on success; simple effect B-inline − B-paths (secondary) | Interaction CI excludes 0 → direction reported. Default: B's better delivery if the simple effect's CI excludes 0; a tie (90% CI inside ±8 points) goes to fewer tokens; else inline | ≈ 12.5 points (≈ 14.5); simple effect ≈ 8.9 |
| 3b | McNemar on "called `search_md`", B vs B−p (n 280, decoys included); zero-call share of B failures (Wilson) | "Uptake binds" if the share's interval is above 50%, "not binding" if below 25%, else "mixed"; pointer adopted only if spurious calls are ≤ 20% (point estimate, n 80) | ≈ 9 points at q 0.3 |
| 4 | Δ Recall@10 (best vector − best lexical) per stratum | **Keep**: lower bound > 0 and Δ̂ ≥ 10. **Kill**: upper bound < 10. Else **inconclusive**; low-overlap below 100 test queries gets bounds only | ≈ 10 points at n 210, p_d 0.2; kill-power 0.84 at Δ = 0, keep-power 0.83 at Δ = 13. Low-overlap at its S0 count, recorded in `prereg/` (n 140: MDE ≈ 12.6 points, kill-power 0.66, keep-power 0.79). Private arm ≈ 18 points |

Parenthesised values and claim-4 powers: strictest Holm step. **Why 8 points:** ~2.5 SE at n = 200; the tokens-per-find condition stops a small token cut passing beside a success loss (at a 30% base rate, an 8-point loss needs a ≥ 27% cut). **Why ±15%:** the smallest powered saving worth an index's upkeep (±10%: ≈ 0.45). The MDE is for planning (smaller test effects may be missed), never a pass bar: requiring observed Δ ≥ MDE passes only ~50% of the time at a true effect of that size. Claim 4's 10-point bar is L35's changeable product proposal, not the MDE.

### 9.4 Honest reporting
- No LLM judge in any headline; nulls published; "not testable on 6 GB", "uptake-limited", "ceiling-limited", "floor-limited", "not run" and "inconclusive" are never passes.
- Published rows are context only, in separate columns, LLM-judge scores labelled; LME-md is never compared with published LongMemEval scores or unrun API models; build, flags and hashes on every table.
- Decoys are a set-aside by-construction stratum (as 06 sets aside its policy arm's wins); generated queries and the paraphrase stratum labelled; secret recall an upper bound.

## 10. Milestones (19 hours / 15 sessions)

| Sprint | Sessions | Hours | Deliverable |
|---|---|---|---|
| **S0** | 1–4 | 5 | Employer-IP check, credit terms and any mentor objection recorded; repo, `uv`, pinned build + GGUF hashes; fixture builder; LME-md converter; **CredData** and **LongMemEval BM25** reproductions; FTS5 tool v0; A/B loop; **week-2 gate** (session 2); variance pilot; paraphrase queries (Mac); low-overlap count; `prereg/` frozen; A-only cells start |
| **S1** | 5–8 | 5 | Filter, watcher, secret gate, leak scan, MCP/CLI; **floor gate**; dev ladder; B cells start. **Week-6 demo**: two agents, one question, live token meters |
| **S2** | 9–10 | 3 | Claims 1 and 3 on test; claim 2 sweep; default format frozen; claim-4 strata; **week-8 cut checkpoint** (session 10) |
| **S3** | 11–12 | 3 | **Claim 4** verdicts; rung 7; Planned: private arm, create, scopes + I1–I6, I8, **two-laptop relay**, 9B rows, second fixture |
| **S4** | 13–14 | 2 | Stretch; package, runbook, **results public**; upstream PR only after G0 (§2) |
| — | 15 | 1 | Interview sprint: replayed demo, 2 × 2 delivery plot, kill-rule verdicts on a whiteboard; the write-up (§14) |

**S0 reproduction rule (pre-registered).** gitleaks v7.6.1 on CredData within F1 ±0.03 of 0.3336 and true positives ±5% of 1,120, on ≥ 98% of labelled lines; else the gap is published as a port delta (08's convention), "not reproduced". LongMemEval's BM25 row is a second target: ±2 points on both metrics, original file, against the value fixed in session 1 (§5.4).

**Tiers.**
- **Core** ships regardless: S0 (both targets attempted); mdsieve; filter, indexer with I7, `search_md`, secret gate, leak scan, MCP server/CLI; claims 1–4; the week-6 demo.
- **Planned:** second fixture; 9B rows; qmd and Basic Memory retrieval rows; create acceptance; scopes, I1–I6, I8 and the relay; the private arm.
- **Stretch:** qmd and Basic Memory as B's tool; write-path persistence in 06's harness (06 §7.5).
- **Optional API track** (≤ $20, outside the cut order): the Claude Code / Codex row.

**Pre-registered cut order (week 8; Planned before Core extras).** Windows: Stretch → second fixture → qmd and Basic Memory rows → create acceptance → claim 2's 90% cells → claim 1's seed 2 (kept if the n-rule fired). Mac and CPU: Stretch → 9B rows → scopes, I1–I6, I8 and relay → private arm. **Never cut:** claims 1–3 at seed 1, claim 2's 0% cells, the spurious-call set, claim 4, S0, the leak scan.

**Spec freeze.** The reference tool follows memtidy at `7708d37`; later direction changes enter only in S3, and only if frozen test configs stay untouched.

## 11. Risks & Mitigations

| Risk | Mitigation |
|---|---|
| G0, credit and conflict of interest beside the mentor's pre-release tool | §2 split, `prereg/` hash; the mentor's post follows or cites mdsieve (agreed in writing before S0; the mentee's date, by S4, stands); after S0 the split changes only by mutual written agreement; the mentee may decline memtidy-specific parts at no cost and keeps mdsieve's copyright |
| The mentee's employer IP clause covers outside open-source work | Checked in writing before S0; own devices and time; if blocked, private research under "Projects" (§12) |
| memtidy changes direction or pauses | Frozen at `7708d37`; a change rewords §1, §2, §7.1, §7.5, §13 and the upstream path; a pause closes that path |
| 4B at the floor; 9B does not fit 6 GB | Floor gate with an A0 control; 9B grid on the Mac; claim 4 needs no agent (§8.3) |
| Ceiling: LME-md too easy (agentmemory, self-published: stemmed BM25 with synonym expansion, R@5 ≈ 86% on `_S`) | Five mount locations beside vendor noise; `_M` rung; tokens still compared |
| The agent ignores `search_md`; tool calls fail to parse (#20198, fixed) | Uptake floor and wording rung; 3b reports uptake either way; argument validator; lazy GBNF |
| Hybrid prefix-cache misses (#22384, #24587, #24714) | Cache-independent primary cost; append-only prompt; gate row 3 logged; 2507 only if time (row 2) also fails |
| By-construction wins (known noise paths, decoys, generated paraphrases) | Held-out marketplaces; LME-md primary; decoys set aside; natural low-overlap stratum; stock rg as rung 0 |
| S0 targets drift (deleted CredData repos; cleaned LongMemEval) | CredData pre-registered; coverage rule; pinned environments and checksums; port deltas (§5.4) |
| Secrets, private or employer text leak | Fake keys only; leak scan down to git objects; screened personal material, never sent to an API (§5.3) |
| Compute overrun | Critical-path margin (§8.4), projected-hours gate, cut order, pre-costed n-rule (§9.3) |

## 12. Résumé Bullets (experience voice; numbers filled in as sprints land)

- Built **mdsieve**, an Apache-2.0 benchmark for agent markdown search (**X** plugin md files from held-out public marketplaces, a LongMemEval-to-md converter, replayable cassettes); ran **X** judge-free episodes of a local Qwen3.5-4B agent within a **6 GB VRAM budget**.
- Measured **agent search cost at non-inferior accuracy under plugin noise**: a noise-filtered SQLite FTS5 tool stayed within 8 points of grep's success (**X%** vs **X%**, lower bound **−X**) with **X%** fewer prompt tokens at full (**X%**) noise; **[the filter alone recovered X% of the saving (CI X–X%) | the index added X% beyond the filter]**.
- Measured **how the tool interface changes small-agent behaviour**: result format shifted success **X points**; a one-line pointer moved search-tool uptake **X% → X%** at **X%** spurious calls.
- Ran a **pre-registered three-valued vector kill rule** (dense and hybrid RRF vs FTS5/rg, four strata): Recall@10 moved **X / X / X / X points** on keyword / low-overlap / paraphrase / zh→en queries, keeping vectors for **[strata]**.
- Shipped an **MCP/CLI md search tool** with a deterministic noise filter (F1 **X** on held-out marketplaces) and a secret gate: **X** leaked substrings (acceptance: 0) across output, index, logs, git objects and cassettes; **X** false-positive files per 1k lines vs **X** for gitleaks.

**Conditional wording.** Bullets 1 and 5 are descriptive (counts and acceptance metrics, no pre-registered test). Bullet 4 appends "after reproducing LongMemEval's BM25 row within **X** points" and bullet 5 "after reproducing gitleaks' CredData F1 within **X**" only if that target reproduced (if neither did, bullet 4 ends "S0 port deltas of **X** published"); bullet 5 adds "; added cross-scope duplicate checks (**X%** recall at **2%** false blocks) and git-backed scopes" only if the Planned tier ran. **The verb follows the measured sign:** a refuted claim's bullet states the measured result ("measured grep matching the index below **X%** noise"), never silently deleted.

### Placement Note

These bullets go under EXPERIENCE **only with a real organization and a real engagement**:
1. **The mentoring organization as employer**: a truthful title (`ML Engineering Practicum (mentored project)`) and an engagement record; "Intern" only if issued in writing and confirmable on a reference check. The entry discloses the mentor's conflict of interest (naming memtidy only after G0 or written consent) and does not present mdsieve as work for memtidy.
2. **A real collaboration**: a startup, lab or open-source org that agrees to host the work.
3. **By default, an open-source project entry**: mdsieve, the mentee's public repo, with real commit history, the results table and the demo video. Merged memtidy commits (only after G0, under written contributor terms) count here, never as employment. If the employer check blocks publication: a private research project under "Projects", without "Apache-2.0", "public", "shipped" or a demo link.

Never invent an employer or dates: North American employers verify employment, and a fabricated entry can get an offer rescinded.

## 13. Why This Reads as Real Company Work

- **Cost at non-inferior accuracy is a product service-level objective (SLO)**: what a coding-agent team asks before shipping a search tool, answered with a non-inferiority margin and tokens per successful find.
- **Attribution before shipping**: the filter-vs-index verdict says which component earns its upkeep; "the filter suffices" is a cheaper product.
- **The decision comes before the data**: vectors and the default format ship only through pre-registered tests; a 50-query rule's ~18-point MDE is stated up front.
- **Governance in the write path**: redaction before index and return, leak scans down to git objects, proposal-only shared writes (L34).

## 14. Interview Leverage

- **ML system design** ("search over an agent's notes and memory files"): filter vs index, delivery format, per-result token budget, scope ranking, uptake.
- **LLM serving:** hybrid KV arithmetic, re-prefill share with append-only results, cold vs warm prefill, tool-call parsing.
- **Search/IR and experimentation:** FTS5 vs rank_bm25 constants, CJK trigram, RRF; non-inferiority, TOST, cluster bootstrap, Holm; why 50 queries detect only ~18 points.
- **Behavioural:** the pre-declared week-2 fallback; attributing the saving between filter and index (whichever the data said); a written credit split with a disclosed conflict of interest.
- **Drills** (practice prompts on common agent-engineering themes): *"Your agent has a search tool and still greps everything: how do you find out why?"* → §7.3, §9.3 claim 3; *"When would you add embeddings to local-file search, and how would you decide?"* → §7.4, §9.3 claim 4; *"Prove a cheaper tool is not worse: what margin, and why?"* → §9.3 claim 1.
- **Write-up:** one post under the mentee's own byline (session 15, §10), working title *"[N] plugin files and one note: what grep costs a 4B agent"*: hook, context, the work (failure before fix, if observed: grep flooded at the 2,000-token cap, then filter and index separated, §9.2), what I'd do differently (the pre-declared null or fallback: "filter suffices" or "inconclusive", §9.3, or "not testable on 6 GB", §8.3), takeaways. Every number traces to the results table; vendor numbers are labelled. Public benchmark only (§2).

## 15. References

- memtidy one-pager and decision log, lines L14, L30, L33–L37 (private repository, commit `7708d37`, 2026-10-19; no URL)
- LongMemEval (Wu et al., ICLR 2025): https://arxiv.org/abs/2410.10813 · code (MIT): https://github.com/xiaowu0162/LongMemEval · LongMemEval-V2: https://github.com/xiaowu0162/LongMemEval-V2
- CredData (Samsung; table "Updated: April 2022"): https://github.com/Samsung/CredData · GitGuardian, secrets in MCP configs (2026): https://blog.gitguardian.com/the-state-of-secrets-sprawl-2026-pr · gitleaks (v7.6.1; v8.30.1, 2026-03-21): https://github.com/gitleaks/gitleaks · Betterleaks: https://github.com/betterleaks/betterleaks · detect-secrets: https://github.com/Yelp/detect-secrets
- Grep vs vectors: *Is Grep All You Need?* (Sen et al., 2026): https://arxiv.org/abs/2605.15184 · repository-QA study (2026): https://arxiv.org/abs/2608.01507 · Cursor (vendor): https://cursor.com/blog/semsearch · https://cursor.com/docs/agent/tools/search · Claude Code's "vibes" remark (Latent Space podcast, May 2025; secondary write-up): https://officechai.com/ai/claude-researcher-explains-how-agentic-search-performed-better-than-rag-for-code-generation/
- LongMemEval-S BM25 rows (self-published): https://github.com/rohitg00/agentmemory/blob/main/benchmark/LONGMEMEVAL.md · SWE-ContextBench: https://arxiv.org/abs/2602.08316
- Corpora: LoCoMo (CC BY-NC 4.0, excluded): https://github.com/snap-research/locomo · LOO trees: https://github.com/kubernetes/website · https://github.com/facebook/docusaurus · https://github.com/mdn/content
- md tools: qmd (MIT): https://github.com/tobi/qmd · Basic Memory (AGPL-3.0): https://github.com/basicmachines-co/basic-memory · claude-mem: https://github.com/thedotmack/claude-mem
- Search internals: SQLite FTS5 `bm25()` (k1 1.2, b 0.75): https://github.com/sqlite/sqlite/blob/master/ext/fts5/fts5_aux.c · sqlite-vec: https://github.com/asg017/sqlite-vec · ripgrep: https://github.com/BurntSushi/ripgrep
- MCP 2026-07-28 (Roots deprecated; deterministic `tools/list` order): https://github.com/modelcontextprotocol/modelcontextprotocol/tree/main/schema · https://blog.modelcontextprotocol.io/posts/2026-07-28/
- Memory mount locations (§5.2): Claude Code https://code.claude.com/docs/en/memory · Codex memories https://developers.openai.com/codex/memories
- Models: https://github.com/QwenLM/Qwen3.5 · https://huggingface.co/Qwen/Qwen3.5-4B · https://huggingface.co/Qwen/Qwen3.5-9B · https://huggingface.co/Qwen/Qwen3-4B-Instruct-2507 · https://github.com/QwenLM/Qwen3-Embedding · https://huggingface.co/unsloth
- llama.cpp: https://github.com/ggml-org/llama.cpp/issues/22384 · https://github.com/ggml-org/llama.cpp/issues/24587 · https://github.com/ggml-org/llama.cpp/issues/24714 · https://github.com/ggml-org/llama.cpp/issues/20198 · https://github.com/ggml-org/llama.cpp/pull/20213 · https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md
