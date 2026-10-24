# Ideation Log

How the first eight proposals in this repository were chosen. Thirty ideas were brainstormed through five lenses, scored by three independent judges, and narrowed to a portfolio of eight. A ninth, 09 memtidy, was added on 2026-10-22 at the mentor's request and judged through the same three lenses ([Added After the Original Selection](#added-after-the-original-selection)). The rejected ideas are kept here because several are good seeds for a second project or a stretch goal.

## Method

1. **Brainstorm.** Five ideators, one per lens: dev & terminal agents; GUI, browser & game agents; training better small agents; agent infrastructure, memory & safety; domain & product agents tied to the mentee's background. Each proposed 4-6 ideas and checked models and benchmarks against the web.
2. **Judge.** Three judges scored every idea 1-10, each through one lens only:
   - **Feasibility:** does it really run on a 6 GB VRAM laptop and a 16 GB Mac, on a 10-12 week part-time schedule?
   - **Career signal:** does it match what North-American ML / AI-engineer interviews probe in late 2026?
   - **Novelty & measurability:** is it memorable, and does it produce honest numbers against a public baseline?
3. **Select.** A selector chose a diverse slate, one project per agent paradigm. A fatal feasibility flaw disqualified an idea unless a concrete fix fully resolved it. Complementary ideas were merged.
4. **Write and review.** Each design doc was written from its brief, then fact-checked against the web and reviewed by a mentor/editor, then revised. A final pass checked consistency across the docs.

## All 30 Ideas

Scores are 1-10 per judge; **Mean** is their average. Rows are sorted by mean score.

| Idea | Lens | Pitch | Feas. | Career | Novelty | Mean | Outcome |
|---|---|---|---|---|---|---|---|
| **BlackBox** | Infra, memory & safety | A flight recorder for agents: OpenTelemetry traces, bit-exact replay, and what-if forks that find the step that broke the run. | 9 | 7.5 | 7.5 | 8 | Selected: [05 BlackBox](05_BlackBox_Design_Doc.md) |
| **Compass** | Training small agents | Agents as a ranking problem: a 1.7B policy proposes, a 0.6B process reward model ranks, and test-time search beats a 5x bigger model at equal latency. | 7 | 8 | 7.5 | 7.5 | Selected: [01 Compass](01_Compass_Design_Doc.md) |
| **Ctrl-Z** | Infra, memory & safety | Undo for the real world: agents act inside a copy-on-write shadow world, and only reviewed diffs get committed. | 8 | 7 | 7.5 | 7.5 | Selected: [06 Ctrl-Z](06_Ctrl-Z_Design_Doc.md) |
| **SimCrowd** | Domain & product | A thousand synthetic users on a laptop: offline A/B tests for recommenders, with a fidelity score that says when not to trust them. | 8 | 7 | 7.5 | 7.5 | Selected: [08 SimCrowd](08_SimCrowd_Design_Doc.md) |
| **HarnessForge** | Dev & terminal | AutoML for agent scaffolds: a coding agent that rewrites its own harness overnight on a gaming laptop, behind promotion gates so it cannot fool itself. | 6 | 8 | 7.5 | 7.2 | Selected: [07 HarnessForge](07_HarnessForge_Design_Doc.md) |
| **MuscleMemory** | GUI, browser & games | Agents that stop re-thinking what they already learned. A JIT compiler turns successful trajectories into self-healing skills, and the VLM is called only when the page surprises them. | 8 | 7 | 6.5 | 7.2 | Not selected (see below) |
| **ToolSmith** | Training small agents | Forge a laptop-sized function-caller: verified-trajectory distillation into a 1.7B/4B student, then settle 'gradients vs. grammars' with numbers. | 8 | 7.5 | 6 | 7.2 | Not selected (see below) |
| **Spelunker** | Domain & product | Deep research with the Wi-Fi off: a laptop agent that digs through a frozen 100K-page web on a token budget it is not allowed to exceed. | 6 | 8 | 7.5 | 7.2 | Selected: [03 Spelunker](03_Spelunker_Design_Doc.md) |
| **AgentMall** | Domain & product | Ranking for robots: what a recommender looks like when the shopper is an LLM agent and some sellers are trying to prompt-inject it. | 6 | 7.5 | 8 | 7.2 | Not selected (see below) |
| **Ghostrun** | Dev & terminal | Speculative execution for agents: a 0.8B drafter guesses the next shell command and runs it in a forked sandbox while the 9B model is still thinking. | 6 | 6.5 | 8.5 | 7 | Not selected (see below) |
| **Switchboard** | Infra, memory & safety | Spend the 9B only where it matters: per-step model routing and tool-schema retrieval for agents on a 6 GB GPU, priced in joules. | 7 | 7 | 7 | 7 | Not selected (see below) |
| **Customs** | Infra, memory & safety | Border control for MCP: every tool output gets a passport stamp, and every outbound call is inspected before it leaves. | 7 | 8 | 6 | 7 | Merged into [06 Ctrl-Z](06_Ctrl-Z_Design_Doc.md) |
| **Shellwright** | Dev & terminal | Teach a 2-4B model to live in a terminal: agent-computer-interface engineering plus trajectory distillation, scored on Terminal-Bench. | 5 | 8.5 | 7 | 6.8 | Selected: [02 Shellwright](02_Shellwright_Design_Doc.md) |
| **Crosshair** | GUI, browser & games | A 2B model that clicks like a 9B. Every rendered page labels itself for free, and a coarse-to-fine zoom cascade adds accuracy at test time. | 7 | 7 | 6.5 | 6.8 | Selected: [04 Crosshair](04_Crosshair_Design_Doc.md) |
| **Metagame** | GUI, browser & games | A 4B model that out-thinks its opponent: LLM-guided minimax + Bayesian opponent modeling for competitive Pokémon on a local Showdown server. | 6 | 6 | 8.5 | 6.8 | Not selected (see below) |
| **Tripwire** | GUI, browser & games | Red team vs. blue team for computer-use agents. An attacker LLM evolves pop-ups and fake buttons, and an action firewall stops a 4B agent from being hijacked. | 7 | 6.5 | 6.5 | 6.7 | Not selected (see below) |
| **NightShift** | Infra, memory & safety | Agent memory that sleeps on it: a local MCP memory server that consolidates, versions and forgets facts while the laptop is idle. | 7 | 7 | 6 | 6.7 | Not selected (see below); its space is revisited by [09 memtidy](09_memtidy_Design_Doc.md) |
| **PocketGym** | Training small agents | Multi-turn GRPO on a gaming laptop: take a 1.7B model from near-zero to most-of-the-time in a household simulator, overnight. | 5 | 8.5 | 6 | 6.5 | Merged into [01 Compass](01_Compass_Design_Doc.md) |
| **Respawn** | Training small agents | A 4B game agent that learns from every death: memory vs. playbook vs. nightly weight consolidation, measured on BALROG. | 7 | 6 | 6.5 | 6.5 | Not selected |
| **PatchFunnel** | Dev & terminal | A retrieval-first funnel from issue to patch that lets a 4B-9B laptop model compete on SWE-bench by adding structure instead of parameters. | 7 | 7 | 5 | 6.3 | Not selected (see below) |
| **Greenlight** | Dev & terminal | A local CI doctor that turns a red Java build into a test-proven patch, or into an honest 'this is flaky' label with evidence. | 7 | 6 | 6 | 6.3 | Not selected (see below) |
| **Glitchhound** | GUI, browser & games | An autonomous playtester that explores games, sniffs out glitches and files reproducible bug reports, graded on bugs we planted ourselves. | 6 | 5 | 8 | 6.3 | Not selected (see below) |
| **MetricPilot** | Domain & product | A self-hosted analytics agent that answers metric questions in SQL, reads A/B tests like a staff data scientist, and refuses to ship a result that fails a sanity check. | 8 | 5.5 | 5.5 | 6.3 | Not selected (see below) |
| **Mutineer** | Dev & terminal | Red team vs blue team on a laptop: one local agent plants realistic bugs, another must write the test that catches them, and the arms race becomes training data. | 4 | 7 | 7.5 | 6.2 | Not selected (see below) |
| **TradingPit** | Domain & product | A stock exchange full of LLM day-traders on a gaming laptop: do bubbles, fat tails and spoofers emerge, and can a regulator agent catch them? | 7 | 4.5 | 7 | 6.2 | Not selected (see below) |
| **Sparring** | Infra, memory & safety | Two laptops, one fight: a sub-2B attacker learns to hijack agents with RL on the PC while the defender learns to resist on the Mac. | 4 | 7 | 7 | 6 | Merged into [06 Ctrl-Z](06_Ctrl-Z_Design_Doc.md) |
| **LabRat** | Domain & product | An ML-engineering agent that runs real Kaggle experiments on a gaming laptop, and has to evict its own brain from the GPU so its model can train. | 6 | 6 | 5.5 | 5.8 | Not selected (see below) |
| **Heckler** | Training small agents | Train an AI customer from hell to break your support agent, then train the agent on every failure: two LoRAs, one base model, one laptop. | 4 | 7 | 5.5 | 5.5 | Not selected (see below) |
| **PocketPilot** | GUI, browser & games | Say it and your phone does it. A voice-driven Android agent for low-vision users runs on the laptop next to the phone, and asks before anything irreversible. | 5 | 5 | 6 | 5.3 | Not selected (see below) |
| **Daydream** | Training small agents | Let the agent practice inside its own dream of the website: a 1.7B world model becomes the gym, and real-environment steps become a budget. | 3 | 6 | 7 | 5.3 | Not selected (see below) |

## Why the Slate Looks Like This

Eight proposals (nine since 09 memtidy was added), one per agent paradigm, all runnable on the stated fleet: a Windows laptop with an RTX 3060-class card, assumed to be the 6 GB laptop part until the mentee confirms otherwise, and a 16 GB Apple Silicon Mac. Each proposal has at least one public, deterministic or judge-calibrated benchmark that runs locally or as a documented lite subset.

Coverage by paradigm:
- RL/post-training plus test-time search: 01 Compass, which absorbs PocketGym's training loop as a bounded arm.
- Coding/terminal: 02 Shellwright, which has the highest career score.
- Deep-research/search agents: 03 Spelunker. It continues the Enclave story (07 in the earlier recsys series) without duplicating it. Enclave stalled as a Sprint-0 scaffold, so Spelunker brings its own search adapter (03 §7.4).
- GUI/vision: 04 Crosshair. It is the best CAS-patent fit and includes real LoRA.
- Agent infra/evaluation: 05 BlackBox. It is the most feasible idea in the pool.
- Agent safety: 06 Ctrl-Z. The four-idea security cluster is merged into one input-side vs outcome-side defense study. Customs supplies the gateway layer and Sparring supplies a bounded adaptive-attack arm.
- Self-improving agents: 07 HarnessForge. It is the most direct upgrade of the mentee's AutoML agent.
- Domain multi-agent simulation for recsys: 08 SimCrowd. It bridges directly to the mentor's Pinterest context.
- Agent memory & context files: 09 memtidy, added on 2026-10-22 ([below](#added-after-the-original-selection)).

Wild picks: HarnessForge (a program that rewrites its own harness overnight), SimCrowd (A/B tests on people who do not exist, plus a test of the simulated people themselves) and Ctrl-Z (parallel-universe execution).

Post-training of a small model, with stacks matched to vendor guidance (Qwen3 QLoRA, Qwen3.5 bf16 LoRA only, short sequences):
- Compass: PRM LoRA plus RFT, with a GRPO-lite stretch.
- Shellwright: per-turn trajectory distillation.
- Crosshair: grounding LoRA.
- Spelunker: RFT student.
- SimCrowd: calibration LoRA.

Disqualified for unresolved feasibility fatal flaws:
- Daydream: compute.
- Heckler: throughput.
- Sparring: wrong training stack and throughput. It survives only as a bounded arm inside Ctrl-Z.
- Mutineer: floor effect.

Shellwright's fatal flaw (8-16k training sequences on 8 GB) is fully resolved by per-turn 2-4k-token segments, so it stays. Fatal flaws from the other judges disqualified PocketPilot, Glitchhound, LabRat, TradingPit and MuscleMemory, or their fixes were too heavy.

Ordering:
- Clusters were deduplicated: one security project, one ALFWorld project, and only BlackBox and Ctrl-Z share the BFCL/tau2 harnesses (AgentDojo is Ctrl-Z's alone).
- Recommended reading order starts with the recsys-bridge training project (Compass). The flagship coding and research agents follow, then GUI, infra and safety, and the two wildest ideas come last. 09 memtidy, added later, reads best after 03 Spelunker and 05 BlackBox, its two closest siblings.

Rules applied to every brief:
- A Sprint-0 local reproduction of one external number.
- Pre-registered minimum detectable effects.
- Qwen3.5 prefix-cache measurement with a standard-attention fallback.
- Explicit prior-art citations.

## Why Notable Ideas Were Not Selected

- **Ghostrun**: Novelty 8.5, but its premise is unvalidated. With a 9B model on one 6-8 GB GPU, LLM prefill and decode probably dominate wall-clock time while most tool calls take milliseconds, so overlapping tools with generation may save almost nothing. A co-resident drafter also slows the target model. Draft decoding with Qwen3.5 hybrid-state rollback and copy-on-write forks of Harbor containers are both unproven. A stretch idea for Shellwright once a critical-path profile shows that tool time matters.
- **Metagame**: Fun and rigorous, but its thesis that test-time search beats model size duplicates Compass. Realistic per-turn cost is 10+ LLM calls, or 15-40 s per turn. That gives 4-12 battles per hour, so ratings with confidence intervals would take weeks of nights. The PokéChamp reproduction is a port from API models.
- **AgentMall**: A distinctive ranking-for-agents question, but it overlaps SimCrowd as the recsys-agent bridge and Ctrl-Z on injection defense. Local-endpoint configuration of Magentic Marketplace is unverified, and four claims x arms x seeds exceeds the compute budget. A good follow-on if SimCrowd lands early.
- **ToolSmith**: Safe and feasible, but verified-trajectory function-calling distillation is a crowded genre (xLAM, APIGen-MT, ToolACE). A Qwen3-4B student will likely lose to off-the-shelf Qwen3.5-4B on BFCL. The tool-calling cluster is already covered by BlackBox and Ctrl-Z.
- **MuscleMemory**: No public benchmark exercises its core compile/replay/de-optimize loop. MiniWoB re-randomizes each seed, Mind2Web is offline, and the drift suite is internal. The real fix, a self-hosted WebArena site, is heavy on 16 GB RAM, so Crosshair took the GUI slot.
- **NightShift**: The memory space is crowded (Mem0, Zep, Letta), and the headline LongMemEval score depends on a swapped-in local judge. Ingestion takes 1-3 nights, and the 'agent sleeps at night' hook repeats across several ideas. Its most novel experiment, memory vs long context on a hybrid model, could be a section of Spelunker. 09 memtidy, added on 2026-10-22 at the mentor's request, works in the same space but drops the memory-quality claim and answers each of these reasons (09 §2).
- **PocketGym**: Merged into Compass as a bounded GRPO-lite arm on TextCraft. As a standalone project, multi-turn ALFWorld GRPO needs about 2.9M backprop tokens per iteration, or 30-50 h per arm, which does not fit the schedule.
- **Customs**: Merged into Ctrl-Z as the input-side gateway layer, so the four-idea security cluster becomes one input-vs-outcome defense comparison.
- **Sparring**: Fatal feasibility flaw: it plans QLoRA on Qwen3.5, which Unsloth advises against, and vLLM GRPO rollouts are unsupported there. Victim-query throughput is several times too high. Kept only as a frozen evolutionary attacker plus an optional single-generation Qwen3-1.7B GRPO stretch inside Ctrl-Z.
- **Daydream**: Fatal feasibility flaw: it needs both all-real and dream GRPO runs, 2-3x PocketGym's budget, which already does not fit. Its fallback, world-model lookahead, is essentially Compass.
- **Heckler**: Fatal flaw: episode throughput is overestimated 3-5x because a 9B on the Mac serves both user simulator and referee, with Qwen3.5 cache misses. Its headline control variable, realism, is an LLM-judge score.
- **Mutineer**: Fatal flaw: a floor effect. A LoRA'd 2B test writer on about 45 SWT-Mini instances likely sits near 0% success, so the self-play gain is unmeasurable. Close prior art (Self-play SWE-RL), and SWT harness disk needs are unverified.
- **PatchFunnel**: Solid but largely an Agentless replication. 50 django-heavy instances give about +/-13-point CIs, nothing is trained, and Shellwright gives a stronger coding signal.
- **Greenlight**: Defects4J program repair is a dated, contaminated genre for 2026 agent roles. There are no public flaky-test labels in its benchmark, and 'SBFL beats LLM localization' is a known result.
- **Switchboard**: Reasonable, but tier swapping on 6 GB dominates latency, laptop energy metering is noisy, and step-routing prior art (RouteLLM, R2V) narrows the novelty. It overlaps BlackBox's BFCL/tau2 harness.
- **Tripwire**: Part of the over-full security cluster. Its unique visual-injection angle is too thin alongside Ctrl-Z, and 4B GUI victims may have such low utility that ASR is uninterpretable.
- **PocketPilot**: Fatal flaw: android_world on Windows/WSL2 is undocumented and Apple Silicon has emulator issues. If the Sprint-0 gate fails, only low-signal offline AndroidControl numbers remain. It is mostly integration.
- **Glitchhound**: Fatal flaw from the career judge: the headline rests on an internal planted-bug suite, and the public numbers measure only side components. Wide engineering scope across unfamiliar game codebases.
- **LabRat**: Fatal flaw: medal rate on 12 Kaggle tasks with a 4-9B model under 2-h budgets is at the floor and too small-n to separate scaffolds. Kaggle download and rule friction.
- **MetricPilot**: Text-to-SQL is the most crowded demo category, and BIRD numbers mostly reflect an off-the-shelf Arctic model. DABstep test answers are hidden and scored only through the Hugging Face leaderboard, so it is not locally scorable. The stats-linter is evaluated on a self-planted suite.
- **TradingPit**: Fatal flaw: the agentic content is thin (one JSON order per tick), and stylized-facts simulation science does not transfer to target roles. It also overlaps AlphaDesk's quant identity.

## Added After the Original Selection

### 09 memtidy (2026-10-22)

The mentor asked to add memtidy, the mentor's own planned open-source tool for finding, creating and sharing the md files agents read and write (CLAUDE.md, AGENTS.md, memory files, notes), as a project. It went through a smaller version of the method above:

1. **Three design briefs**, each from one angle: agent cost under noise (the agent's grep loop vs a noise filter vs an index); whether md context helps at all (nothing, files stuffed at startup, a self-written digest, or a librarian tool); and governed writes (placement, cross-scope duplicates, a secret gate, team scopes).
2. **The same three lenses** scored each brief:

| Brief | Feas. | Career | Novelty | Mean | Outcome |
|---|---|---|---|---|---|
| Agent cost under noise | 7.5 | 8 | 7.5 | 7.7 | Base of 09 |
| Does md context help? | 5.5 | 7.5 | 6 | 6.3 | Grafted: tool uptake and the pointer ablation (claim 3b). Dropped: the stuffed-at-startup arm, a near by-construction win that also repeats NightShift's memory-vs-long-context experiment (03 owns it), and an AGENTbench-lite arm likely at 0% with a 4B |
| Governed writes | 8 | 5.5 | 6.5 | 6.7 | Grafted: the S0 CredData target, the leak scan and invariant I7 (Core); create, duplicates, scopes, the other invariants and the two-laptop relay (Planned). Not the headline: it rested on an internal synthetic suite, Glitchhound's flaw |

3. **Synthesize, write, review.** The winner absorbed the grafts. The doc was reviewed adversarially through four lenses (template, facts, feasibility and statistics, privacy and credit), fixed over two rounds, then scored as an idea:

| Idea | Lens | Pitch | Feas. | Career | Novelty | Mean | Outcome |
|---|---|---|---|---|---|---|---|
| **memtidy** | Infra, memory & safety | Grep or index for small agents, measured: a public benchmark of what a laptop 4B agent pays in tokens to find one note under thousands of plugin markdown files, how much of any saving the noise filter and the FTS5 index each earn, and whether a one-line pointer changes tool uptake. | 7.5 | 7.5 | 7.5 | 7.5 | Added: [09 memtidy](09_memtidy_Design_Doc.md) |

**What changed since NightShift.** No memory-quality claim and no LLM judge in any headline; the FTS5 index builds in minutes, only claim 4's embedding may take a night, and no agent arm waits on it; no "sleeps at night" hook; memory vs long context left to 03 Spelunker.

**Conflict of interest.** memtidy is the mentor's own pre-release tool. 09 therefore makes the mentee's benchmark (mdsieve) the deliverable, with a mentee-built reference tool to memtidy's spec as the system under test; memtidy itself (no code yet) stays the mentor's and is benchmarked only once it exists. Credit, upstreaming and publication rules are agreed in writing before S0 (09 §2, §11).

**Weaknesses the judges raised**, kept visible as for the other ideas:
- The headline is niche, and the likeliest winner may be a simple path-rule filter. 09 pre-registers "filter suffices" as a publishable outcome.
- The S0 reproduction (gitleaks on CredData) checks the secret gate, not the search headline, so LongMemEval's BM25 retrieval row is a second S0 target.
- The headline sits close to the mentor's own evaluation plans for memtidy, so publication order and citation are agreed before S0.

---

# Round 2 (2026-10-23)

A second brainstorm aimed at what round 1 missed. Six new lenses were used: voice and embodied agents; multi-agent and social play; personal and private agents; science and data work; benchmarks and evaluation science; and creative wildcards. All 30 round-1 ideas were excluded as near-duplicates. The same three judges scored the results, and a selector shortlisted ten. Each shortlisted idea has a brief in [candidates/](candidates/) that is ready to expand into a full design doc.

## Shortlist

| Rank | Idea | Lens | Pitch | Main benchmark |
|---|---|---|---|---|
| 1 | [**Cellmate**](candidates/Cellmate.md) | science / data-work product | A laptop spreadsheet agent that writes live formulas, recalculates with LibreOffice after every write, and fuzzes its own edits before it hands in the sheet. The claim: engine-in-the-loop beats a Python/openpyxl-only arm at matched turns on SpreadsheetBench Verified-400. Metamorphic self-tests also cut the 'passes test case 1, fails cases 2-3' hard-coded-constant failure. Vendors claim engine beats Python but publish no controlled ablation, and Excel is the most-used programming environment there is. | SpreadsheetBench Verified-400 (local LibreOffice recalc scoring; open baseline Qwen3-8B 15.9 / Spreadsheet-RL-8B 22.3 Pass@1, split unstated); SpreadsheetBench 2 Financial_Model + Debugging only |
| 2 | [**Half-Life**](candidates/Half-Life.md) | Eval science | Survival analysis for agents. A seeded, contamination-proof tool gym (Ledger) labels the exact step where a small model falls into an unrecoverable state. Each model then gets a Weibull hazard curve, and a test asks whether the hazard is really constant. Popular test-time tricks (context reset, self-verify, thinking, per-step best-of-k) are priced as hazard ratios per 1k tokens. It is the agent analogue of MTBF: cheap, exact, and fun to explain. | Ledger (project gym with exact BFS dead-state labels, released with seeds); Long-Horizon-Execution (Sinha et al., public) as Sprint-0 replication; BFCL v4 multi-turn binned by step count for external validity; METR/Hamilton fits as reference only |
| 3 | [**Crosstalk**](candidates/Crosstalk.md) | Voice & embodied | A full-duplex voice agent on one 6 GB laptop that knows when to talk, when to listen and when to stop. It is a scheduled cascade: streaming ASR with an end-of-utterance token, fused acoustic and semantic end-of-turn, a 4B model that thinks while you are still talking, and chunked TTS. Every millisecond is attributed to a stage, and turn-taking is scored on Full-Duplex-Bench next to published Moshi, Freeze-Omni and Gemini Live rows. The interview demo is a live 'interrupt me' test with the latency waterfall on screen. | Full-Duplex-Bench v1 (offline, published Moshi/Freeze-Omni/Gemini rows per subset) and v1.5 (overlap handling); Big Bench Audio (1,000 Qs, MIT); optional tau2 retail half-duplex read-back arm |
| 4 | [**Rubberstamp**](candidates/Rubberstamp.md) | Eval science | Pick the best of N agent rollouts with an LLM judge and watch the judge get gamed, with no training at all. The judge's estimated success rises with N while the database says otherwise. A 2026 study showed this for single-answer tasks (gap 0.20 at k=1 to 0.59 at k=16) and explicitly left agents out. Rubberstamp brings it to state-verified tool trajectories, adds state-preserving narrative attacks, and tests a shadow-rollout judge: the judge plays the task first, then compares end states. | BFCL v4 multi-turn (primary, state-checked, no user sim); tau2-bench retail (~40 pre-filtered tasks, N<=8); AgentRewardBench (subsample ~300, text-only; GPT-4o a11y precision 69.8 / recall 83.1, rule-based 83.8 / 55.9) |
| 5 | [**Haggle**](candidates/Haggle.md) | multi-agent / agentic commerce | Two laptops haggle over a used car through the real A2A protocol. The buyer's money decisions are made by a typed strategy engine and a deterministic shield that enforces an AP2-style intent mandate, never by LLM sampling. A 4B model only parses and talks. The headline is the measured cost of that guarantee: deal rate and surplus versus raw 4-8B negotiators on A2A-NT, where a raw Qwen2.5-7B buyer overspends its budget in 11.76% of deals. It also shows robustness to a red-team seller that cries poverty and smuggles fake system text. | A2A-NT (100 products x 5 budgets; local via LiteLLM; Qwen2.5-7B out-of-budget 11.76%); re-implemented NegotiationArena buyer-seller game; NegMAS/ANL classic agents as non-LLM baselines |
| 6 | [**Tact**](candidates/Tact.md) | Personal & private | A proactive assistant that knows when to keep quiet. A Qwen3-0.6B gate on CPU decides whether to spend the user's attention, and conformal risk control gives it a distribution-free false-alarm budget. Its threshold adapts online to each user's patience. The hook: a published fine-tuned 7B on ProactiveBench (recall 100, false-alarm 50.22) scores what 'always interrupt' would score. Tact reframes proactive AI as alert-fatigue engineering. | ProactiveBench (thunlp, ICLR 2025; 233 test events; Qwen2-7B-Proactive F1 66.47 / FA 50.22; PRISM gate as main baseline); ProAgentBench stretch only if downloadable |
| 7 | [**Twin**](candidates/Twin.md) | Eval science | Is your agent smart, or has it seen the test? Twin builds isomorphic twins (bijective renames that keep the optimal plan) and counterfactual twins (policy edits that change the right answer) of agent benchmarks. It calibrates contamination detectors by deliberately poisoning its own model at known doses. A new metric, prior-override rate, asks whether an agent obeys the edited refund policy in its prompt or its memory. A noisy-oracle shrinker (from Shrinkwrap) then finds the minimal twin edit that flips a model's outcome. | BFCL v4 multi-turn (isomorphic + counterfactual twins, core); ALFWorld valid_seen/unseen isomorphic only (AgentLM-7B 84.0 vs Llama-2-7B-chat 2.0); tau2 retail counterfactual policies on ~30 tasks; WikiMIA detector sanity (Min-K%++) |
| 8 | [**Tell**](candidates/Tell.md) | multi-agent / safety | A lie detector wired into the model's residual stream, tested at a Werewolf table where the game state proves who lied. A linear probe trained off-game on instructed honest/dishonest pairs (Apollo method) is pitted against black-box judges. Those judges reach only 0.55-0.58 AUROC in WOLF. Tell maps detectability across Qwen3 0.6B-8B and dense vs hybrid Qwen3.5. Then a Detective villager gets the probe as a tool. | WOLF Werewolf engine (LangGraph, open code; peer detection ROC AUC 0.55-0.58); Apollo-style instructed-pairs probe eval; Among Us sandbox as OOD prior-art comparison |
| 9 | [**Paperweight**](candidates/Paperweight.md) | science / research engineering | Make the paper pull its weight. ResearchCodeBench found that giving the paper helps frontier models, but small models gain little, and LLaMA-based models get worse. Paperweight reproduces that anomaly on a laptop, then reverses it. A 0.6B retriever injects only the equation spans tied to the masked snippet, and the coder writes its own tests from that math. Every test runs on CPU in about a second. | ResearchCodeBench (NeurIPS 2025 D&B; 212 snippets, 20 papers; CPU tests ~1.25 s/task; Gemini-2.5-Pro 37.3% scaled pass@1; small-model numbers only in Fig. 2); fresh post-June-2026 30-snippet internal set |
| 10 | [**Levelsmith**](candidates/Levelsmith.md) | wild / program search | Do not ask the LLM for a level; ask it for a level generator. On the CPU-only PCG Benchmark, a local 4B designer writes and refines generator programs with solver playtests in the loop. It is compared with the benchmark's GA and ES at an equal number of content evaluations. The claim is amortisation: one program hits unseen control targets zero-shot, where the GA must search again for each target. Interviewers can play the levels on the spot. | PCG Benchmark (pcg_benchmark, MIT, pure Python CPU; GA feasible 224/360 runs, ES 191, Random 32; appendix LLM baselines only in a figure, re-run) |

## All 30 Round-2 Ideas

Scores are 1-10 per judge. A dash means the career judge returned no score for that idea under a matching name. **Mean** averages the scores that exist.

| Idea | Lens | Pitch | Feas. | Career | Novelty | Mean | Outcome |
|---|---|---|---|---|---|---|---|
| **Half-Life** | Eval science | Survival analysis for agents: record the exact step where a small model fails on a long task, fit its hazard curve, and price each test-time trick as a hazard ratio. | 8.5 | – | 8 | 8.3 | Shortlisted #2 |
| **Tact** | Personal & private | A proactive assistant that knows when to keep quiet: a 0.6B gate with a guaranteed false-alarm budget, tuned to each user's patience. | 8.5 | 7 | 7.5 | 7.7 | Shortlisted #6 |
| **Cellmate** | Science & data work | A laptop spreadsheet agent that writes live formulas instead of pasted values, and fuzzes its own edits so they pass the hidden test cases. | 7 | 8.5 | 7.5 | 7.7 | Shortlisted #1 |
| **Crosstalk** | Voice & embodied | A full-duplex voice agent on a 6 GB laptop that knows when to talk, when to listen and when to stop. | 7 | 8 | 7.5 | 7.5 | Shortlisted #3 |
| **Haggle** | Multi-agent & social | Two laptops haggle over a used car over A2A. One of them has a calculator for a brain and a muzzle on its wallet. | 8 | 7.5 | 7 | 7.5 | Shortlisted #5 |
| **Shrinkwrap** | Eval science | Like git bisect for an agent that flips coins: find the smallest API drift or prompt change that breaks a tool-calling agent, and prove the failure reproduces. | 7.5 | 7 | 8 | 7.5 | Merged / dropped (see below) |
| **Tell** | Multi-agent & social | A lie detector wired into the model's residual stream, tested at a Werewolf table where the game itself says who lied. | 7 | 6.5 | 8.5 | 7.3 | Shortlisted #8 |
| **Twin** | Eval science | Is your agent smart, or has it seen the test? Isomorphic and counterfactual twins of agent benchmarks, with a contamination detector calibrated by poisoning our own model on purpose. | 6.5 | 7.5 | 8 | 7.3 | Shortlisted #7 |
| **Rubberstamp** | Eval science | Run best-of-N against an LLM judge on agent trajectories and watch the judge get gamed without any training. Then test whether making the judge solve the task first fixes it. | 6 | 8 | 8 | 7.3 | Shortlisted #4 |
| **Shoebox** | Personal & private | Ask your camera roll anything with the Wi-Fi off: a phone-class agent over 15k personal photos whose memory index is 32x smaller thanks to binary codes. | 8.5 | 7 | 6 | 7.2 | Merged / dropped (see below) |
| **Paperweight** | Science & data work | Make the paper pull its weight: a small coder that reads only the equation it needs, then writes tests from that equation to check its own code. | 7.5 | 6.5 | 7.5 | 7.2 | Shortlisted #9 |
| **Mimic** | Eval science | Can an LLM-written sandbox stand in for the real API? Sim-to-real fidelity for synthetic agent environments, measured where the real code exists. | 7 | 7.5 | 7 | 7.2 | Merged / dropped (see below) |
| **Rapport** | Multi-agent & social | A Codenames spymaster that reads its teammate's mind: a retrieval funnel proposes clues, and a simulated partner picks the one they will actually get. | 6.5 | 6 | 8.5 | 7 | Merged / dropped (see below) |
| **Levelsmith** | Wildcards | Do not ask the LLM for a level; ask it for a level generator. Program search with solver playtests on the PCG Benchmark, plus a designer that tries to stump AI players. | 9 | 5.5 | 6.5 | 7 | Shortlisted #10 |
| **Hindsight** | Science & data work | A forecasting agent built like a quant backtest: point-in-time data only, no peeking at the future, and a live public scoreboard it cannot game. | 7 | 6.5 | 7 | 6.8 | Merged / dropped (see below) |
| **Kepler** | Science & data work | A 4-8B scientist on a gaming laptop runs experiments in alien universes, where gravity might fall off as r^-1.5, and has to discover the law itself. | 7.5 | 6.5 | 6.5 | 6.8 | Merged / dropped (see below) |
| **ReadBack** | Voice & embodied | Pay down the voice tax: a laptop phone agent that never books the wrong confirmation code. | 5 | 8 | 7 | 6.7 | Merged / dropped (see below) |
| **Discreet** | Personal & private | A 1.7B bouncer between your inbox and any agent. It decides which personal facts may leave in this context, and it is trained with RL on about 590 synthetic examples on a gaming laptop. | 7.5 | 7 | 5.5 | 6.7 | Merged / dropped (see below) |
| **Peephole** | Wildcards | -O0 to -O3 for agent programs: a pass manager that makes a local 4B-8B multi-hop pipeline faster and more accurate, measured like a compiler paper. | 7.5 | 6.5 | 6 | 6.7 | Merged / dropped (see below) |
| **Retake** | Personal & private | An agent that coaches a blind user's camera. It answers, admits it cannot tell, or says 'tilt left', and it is trained in a simulator built from real blind users' photos. | 7 | 6 | 6.5 | 6.5 | Merged / dropped (see below) |
| **Gizmo** | Wildcards | Small VLMs fail at Blender because they do not know its API. Give one the manual, let it write and unit-test its own helper SDK, and measure how far a 4B model gets on BlenderGym. | 5.5 | 6.5 | 7.5 | 6.5 | Merged / dropped (see below) |
| **Lorekeeper** | Wildcards | An interactive-fiction game master that writes its world's rules as Python while you play, so the dragon it killed stays dead. | 7 | 5.5 | 7 | 6.5 | Merged / dropped (see below) |
| **Telepath** | Multi-agent & social | The agents stop talking in words and pass their thoughts (hidden states and KV caches) to each other. Then we build the wiretap that makes that safe to ship. | 6 | 5.5 | 7.5 | 6.3 | Merged / dropped (see below) |
| **Lemma** | Science & data work | A pocket Lean prover that works inside real math repositories: it finds the right definitions, reads the compiler's complaints, and learns only from proofs the kernel accepted. | 6 | 6 | 7 | 6.3 | Merged / dropped (see below) |
| **Cutroom** | Wildcards | Two hours of footage plus a brief in, a 60-second trailer out, on a 6 GB GPU, and it never cuts on a half-spoken word. | 7 | 6 | 5.5 | 6.2 | Merged / dropped (see below) |
| **Hivecraft** | Multi-agent & social | Three 4B bots must bake a cake together in Minecraft. The paper says talking is what breaks them, so we replace chat with a protocol. | 5.5 | 6 | 6.5 | 6 | Merged / dropped (see below) |
| **Tempo** | Voice & embodied | How slow can a robot brain be before the robot drops the cup? A success-latency-memory frontier for VLAs on a laptop and a Mac. | 5.5 | 5.5 | 6.5 | 5.8 | Merged / dropped (see below) |
| **Canary** | Personal & private | Your assistant learns your writing voice overnight. A planted canary tells you whether it also learned your secrets, and a privacy gate refuses to ship it if it did. | 7 | 5 | 5.5 | 5.8 | Merged / dropped (see below) |
| **Foreman** | Voice & embodied | A 2B VLM supervises a 0.45B robot policy: decompose, watch, retry, on a 6 GB laptop. | 4.5 | 6 | 6 | 5.5 | Merged / dropped (see below) |
| **Backseat** | Voice & embodied | Yell 'stop!' at a simulated robot and it stops in under half a second, then takes 'no, the other bowl' as a correction. | 4 | 4.5 | 6 | 4.8 | Merged / dropped (see below) |

## What Round 2 Adds

Round 2 adds paradigms the 8 docs lack, and every pick keeps a public anchor that runs locally. The new paradigms: spreadsheet agents with an executable critic (Cellmate), realtime full-duplex voice (Crosstalk), survival-analysis reliability science (Half-Life), judge overoptimization on state-verified trajectories (Rubberstamp), agent-to-agent commerce over A2A (Haggle), calibrated proactivity (Tact), dose-calibrated contamination detection (Twin), white-box deception monitoring (Tell), paper-grounded research coding (Paperweight) and LLM program search for content (Levelsmith). Three merges remove duplicates: ReadBack's read-back policy becomes a stretch arm of Crosstalk, Shrinkwrap's noisy-oracle shrinking becomes Twin's stretch, and Canary's release gate is dropped. The embodied/LIBERO cluster is cut entirely: it is Linux-only, has a 70-vs-90 replication gap, and is out-probe risk for this mentee. Each brief folds in the judges' fixes, chiefly 8-9B on the Mac and 4B on the 6 GB PC, Sprint-0 reproduction gates, floor-effect go/no-go thresholds, and the prior art the ideators missed (2609.29508, 2609.00088, LiveKit turn-detector, 2608.17947, Zero-CoT Probe, PIPE). Recommended pairing with existing docs: Cellmate or Crosstalk for product roles, Rubberstamp or Half-Life as the eval-science slot beside BlackBox, Haggle for agentic-commerce/platform roles.

## Why the Others Were Not Shortlisted

- **ReadBack**: Merged into Crosstalk as a half-duplex stretch arm. Standalone it has a fatal integration risk: tau-Voice targets realtime APIs and its caller voices are Sierra-internal.
- **Shrinkwrap**: Merged into Twin as its stretch: SPRT-gated shrinking of the perturbation program. Standalone it sits in BlackBox's eval-infra slot, and with a 4B model at ~22% on BFCL-MT too few tasks reach the pass rate it needs.
- **Tempo**: LIBERO is Linux-only in LeRobot, and a 6 GB study replicated SmolVLA at 70% vs the published 90%. Its 8,000-episode grid also exceeds the budget. Only the contention result is novel.
- **Foreman**: Fatal flaw: the executor fine-tune on LIBERO-90 cannot be done on 6 GB, and lerobot/libero lacks LIBERO-90. Robotics interviewers would also out-probe the mentee.
- **Backseat**: Fatal scope: it combines three stacks on an unverified Linux-only simulator. Its headline metrics come from a self-designed operator with no public baseline.
- **Hivecraft**: Floor effect: zero-shot 8B scores ~0 on MineCollab. Episodes run in real time under tmux/Java (WSL2), so trials are too few for clean CIs.
- **Rapport**: Novel, with good retrieval fit, but the 6x6 cross-play matrix is ~10x the stated compute. The paper dropped small models for rule-breaking, and the agentic depth is thin.
- **Telepath**: Its fair-speed claim C2 is confounded by mixing engine and precision. LatentMAS has no MPS or quantized support, a causal audit already exists (2607.26773), and there is no production path.
- **Discreet**: A crowded area (CI-RL, PrivacyChecker, FLOWSEAL), and its gateway architecture echoes Ctrl-Z/Customs. Its numbers depend on a local judge that cannot be compared with published results.
- **Shoebox**: Novelty is low: sign-binarised embeddings are commodity practice, and it is close to replicating camroll-agent on a subset with an unknown QA count. It also overlaps Crosshair's CAS slot.
- **Canary**: LaMP gains (+0.01-0.05 ROUGE) sit within noise, the agent loop is thin, and the main result (rank and epochs increase memorisation) is already known.
- **Retake**: The main agentic claim lives entirely in a self-built simulator with an invented motion model. The accessibility hiring market is small.
- **Lemma**: Heavy Lean/Mathlib multi-toolchain setup, a niche AI-for-math market that probes formal-methods depth, and a forced CAS mapping.
- **Hindsight**: It doubles down on the quant identity AlphaDesk already provides, the overlap that sank TradingPit. Few live questions resolve within 12 weeks, and a 2023-cutoff 8B likely sits near base rates.
- **Kepler**: Pre-empted. LLM-AutoSciLab (2605.24043) already pairs an LLM with PySR at 67.6% on NewtonBench, and Model Discovery Agent already designs experiments by disagreement. Its C1 target is a strawman.
- **Mimic**: It repeats SimCrowd's 'when not to trust the simulator' pitch, and C-World (2601.06328) already reports ranking fidelity of 0.883. Its generators may also have memorised the BFCL code.
- **Peephole**: It overlaps HarnessForge, ToolSmith and Switchboard. Full GEPA budgets take 30-48 h per run, and the structure-pass latency gains are likely a prefill-dominated null.
- **Cutroom**: The public metrics are weak (TVSum/SumMe F1 is near-random) and QVHighlights video availability is shaky. The real product, the trailer, has no benchmark.
- **Gizmo**: BlenderGym's Blender version and render settings are undocumented, so comparability is at risk. Blender also contends with the VLM for VRAM, the 3D market is niche, and mentee fit is weak.
- **Lorekeeper**: Code world models are prior art (ByteSized32, WorldCoder), and the FlawedFictions data download was not found. Its public benchmarks test parts of the system, not the GM product.
