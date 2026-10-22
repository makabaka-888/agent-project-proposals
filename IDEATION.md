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
| **NightShift** | Infra, memory & safety | Agent memory that sleeps on it: a local MCP memory server that consolidates, versions and forgets facts while the laptop is idle. | 7 | 7 | 6 | 6.7 | Not selected (see below); its question re-scoped in [09 memtidy](09_memtidy_Design_Doc.md) |
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
- **NightShift**: The memory space is crowded (Mem0, Zep, Letta), and the headline LongMemEval score depends on a swapped-in local judge. Ingestion takes 1-3 nights, and the 'agent sleeps at night' hook repeats across several ideas. Its most novel experiment, memory vs long context on a hybrid model, could be a section of Spelunker. Re-scoped on 2026-10-22 as 09 memtidy, which drops the memory-quality claim and answers each of these reasons point by point (09 §2).
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
| Governed writes | 8 | 5.5 | 6.5 | 6.7 | Grafted as the Planned tier (create, duplicates, scopes, two-laptop relay). Not the headline: it rested on an internal synthetic suite, Glitchhound's flaw |

3. **Synthesize, write, review.** The winner absorbed the grafts. The doc was reviewed adversarially through four lenses (template, facts, feasibility and statistics, privacy and credit), fixed over two rounds, then scored as an idea:

| Idea | Lens | Pitch | Feas. | Career | Novelty | Mean | Outcome |
|---|---|---|---|---|---|---|---|
| **memtidy** | Infra, memory & safety | Grep or index, settled for small agents: a public benchmark measures what a laptop 4B agent pays in tokens to find one note under thousands of plugin markdown files, and whether a noise filter, an FTS5 index or a one-line MCP pointer earns the saving. | 7.5 | 7.5 | 7.5 | 7.5 | Added: [09 memtidy](09_memtidy_Design_Doc.md) |

**What changed since NightShift.** No memory-quality claim and no LLM judge in any headline; indexing in minutes, not nights; no "sleeps at night" hook; memory vs long context left to 03 Spelunker.

**Conflict of interest.** memtidy is the mentor's own pre-release tool. 09 therefore makes the mentee's benchmark (mdsieve) the deliverable and memtidy a system under test, with credit, upstreaming and publication rules agreed in writing before S0 (09 §2, §11).

**Weaknesses the judges raised**, kept visible as for the other ideas:
- The headline is narrower than Shellwright's or Spelunker's, and the likeliest winner may be a simple path-rule filter. 09 pre-registers "filter suffices" as a publishable outcome.
- The S0 reproduction (gitleaks on CredData) checks the secret gate, not the search headline, so LongMemEval's BM25 retrieval row is a second S0 target.
- The headline sits close to the mentor's own evaluation plans for memtidy, so publication order and citation are agreed before S0.
