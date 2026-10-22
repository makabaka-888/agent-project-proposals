# AI Agent Engineering Project Proposals

Design documents for 1:1 mentoring projects in **LLM agents**: test-time search, post-training of small models, terminal / research / GUI agents, evaluation infrastructure, safety, self-improvement, multi-agent simulation, and agent memory & context files. Each doc is a self-contained engineering proposal: thesis and measurable claims, stack, benchmarks, architecture, agent loop, hardware profiles, evaluation plan, milestones, risks, résumé bullets, and interview leverage.

Companion to [recsys-project-proposals](https://github.com/makabaka-888/recsys-project-proposals). Same bar, same 19-hour / 15-session shape, and the same mentee.

> **Private repository.** These documents reference a mentee's employment history and job-search strategy. Do not make this repository public.

## Proposals

| # | Project | Agent paradigm | Headline technique |
|---|---|---|---|
| [01](01_Compass_Design_Doc.md) | **Compass** | Test-time search + post-training | A 1.7B policy proposes actions and a 0.6B process reward model ranks them. The pair is compared with an 8B model at equal latency on ALFWorld and TextCraft, then closes the loop with PRM-filtered rejection-sampling fine-tuning |
| [02](02_Shellwright_Design_Doc.md) | **Shellwright** | Terminal / coding agent | Harness-sensitivity and agent-computer-interface ablations on a Terminal-Bench lite set, then per-turn verified-trajectory distillation into a ~2B student |
| [03](03_Spelunker_Design_Doc.md) | **Spelunker** | Deep research, Wi-Fi off | Budget-bounded context management and an in-agent retriever ablation on BrowseComp-Plus's frozen corpus, plus rejection-sampling distillation |
| [04](04_Crosshair_Design_Doc.md) | **Crosshair** | GUI grounding / computer use | A self-labelling DOM and accessibility-tree data engine, LoRA on a 0.8B vision-language model that is measured against a 2B base, and confidence-gated coarse-to-fine zoom, measured on ScreenSpot-Pro |
| [05](05_BlackBox_Design_Doc.md) | **BlackBox** | Agent eval infrastructure | Record/replay cassettes, counterfactual forks that find the step that broke a run, a temperature-0 nondeterminism study, and regression CI with stated statistical power |
| [06](06_Ctrl-Z_Design_Doc.md) | **Ctrl-Z** | Agent safety runtime | Copy-on-write shadow-world execution and an MCP gateway. Input-side vs outcome-side prompt-injection defence is compared under static and adaptive attack (AgentDojo) |
| [07](07_HarnessForge_Design_Doc.md) | **HarnessForge** | Self-improving agents | An archive of agent-harness code evolves overnight. Successive halving and paired-bootstrap promotion gates stop it from fooling itself, with random-mutation controls |
| [08](08_SimCrowd_Design_Doc.md) | **SimCrowd** | Multi-agent simulation for recsys | Generative user agents on a local 4B run offline A/B tests. They are validated against real behaviour logs and calibrated, and fidelity metrics act as CI gates |
| [09](09_memtidy_Design_Doc.md) | **memtidy** | Agent memory & context files | mdsieve, a public benchmark of what a local 4B agent pays to find one note among thousands of plugin markdown files: its own grep loop vs a noise filter vs an FTS5 index, result format and tool uptake, and a powered, pre-registered kill rule for vectors |

[IDEATION.md](IDEATION.md) records how the first eight were chosen: 30 brainstormed ideas, three judges' scores, and why the others were dropped. Several rejected ideas, such as Ghostrun, Metagame and AgentMall, make good stretch goals. 09 was added on 2026-10-22 and scored by the same three lenses (IDEATION.md, "Added After the Original Selection").

> **09 builds on the mentor's own tool.** memtidy is the mentor's planned open-source project (no code yet). 09's §2 "Who builds what" sets the credit split and the conflict-of-interest safeguards: the mentee owns the benchmark, and memtidy is only a system under test. Mentor and mentee agree it in writing before S0.

## Choosing One

| If the target role is… | Start with | Strong second |
|---|---|---|
| Applied scientist / post-training / RL | 01 Compass | 02 Shellwright |
| AI engineer building agent products | 03 Spelunker | 02 Shellwright |
| Agent platform / ML infra | 05 BlackBox | 07 HarnessForge |
| Multimodal / computer-use | 04 Crosshair | 06 Ctrl-Z |
| Trust & safety / security ML | 06 Ctrl-Z | 05 BlackBox |
| Recsys / ranking (continues the earlier repo) | 08 SimCrowd | 01 Compass |
| Agent quality / evaluation (eval harnesses, judge calibration, regression gates) | 05 BlackBox | 02 Shellwright |
| Context engineering / agent memory and developer tools | 09 memtidy | 03 Spelunker |

The docs cross-reference each other where one project's component helps another, for example BlackBox's τ² and BFCL state serializers reused by Ctrl-Z, or Shellwright's distillation trainer reused by Spelunker. A second project can therefore reuse parts of the first instead of starting from zero. 09 memtidy likewise records every agent run as a BlackBox cassette.

## Shared Design Principles

- **Laptop-only, by rule.** No rented servers and no paid GPU cloud. The fleet is a Windows laptop (16 GB RAM, RTX 3060-class, **6 GB VRAM assumed**, 8 GB if confirmed) and an Apple Silicon Mac (16 GB, no CUDA). The weakest machine sets the defaults: a `portable` profile is the default and `accelerated` is opt-in.
- **Open weights on the critical path.** Models are served by llama.cpp, Ollama or MLX. A commercial API appears only as an optional comparison baseline, capped at $20 in total. Free Colab or Kaggle GPUs are optional accelerators, never a dependency.
- **Public benchmarks that actually run locally.** Each project reports on at least one public benchmark, or a documented lite subset, that fits the fleet. Sprint 0 reproduces one published external number before anything new is claimed.
- **Pre-registered statistics.** Sample sizes are chosen from a variance pilot, minimum detectable effects are stated up front, and an inconclusive result is reported as inconclusive.
- **Honest reporting.** LLM-as-judge scores are labelled as such. No comparison is made against a model the project never ran. Numbers that could not be verified are marked "to be measured".
- **Truthful placement.** Résumé bullets are written in experience voice, with numbers filled in as sprints land. Each doc's Placement Note rules out inventing an employer or dates.
- **One public write-up.** Each project ends with one post under the mentee's own byline (§14 of every doc): hook, context, the work (failure before fix), what I'd do differently, takeaways. Every number traces to the results table, and vendor numbers are labelled.

## Freshness

Model names, library support and baseline numbers were checked against the web in October 2026 and change quickly. Re-verify versions in Sprint 0, which every milestone plan already schedules.
