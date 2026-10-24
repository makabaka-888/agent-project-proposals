# Haggle — Candidate Brief (Round 2, rank 5)

> Not yet a full design doc. This is the brief a writer would expand into one, with the judges' fixes already folded in. Facts were web-checked in October 2026 and still need the usual review pass.

Two laptops haggle over a used car through the real A2A protocol. The buyer's money decisions are made by a typed strategy engine and a deterministic shield that enforces an AP2-style intent mandate, never by LLM sampling. A 4B model only parses and talks. The headline is the measured cost of that guarantee: deal rate and surplus versus raw 4-8B negotiators on A2A-NT, where a raw Qwen2.5-7B buyer overspends its budget in 11.76% of deals. It also shows robustness to a red-team seller that cries poverty and smuggles fake system text.

| | |
|---|---|
| Lens | multi-agent / agentic commerce |
| Benchmark | A2A-NT (100 products x 5 budgets; local via LiteLLM; Qwen2.5-7B out-of-budget 11.76%); re-implemented NegotiationArena buyer-seller game; NegMAS/ANL classic agents as non-LLM baselines |
| Models | Buyer Qwen3-4B-Instruct-2507 Q4_K_M (PC, ~2.5 GB + KV); seller Qwen3-8B MLX 4-bit or Qwen2.5-7B Q4 (Mac, ~5 GB); optional GPT-4o-mini judge cross-check (<$5) |
| Hardware fit | Buyer on PC, seller on Mac over LAN A2A; NegMAS and shield on CPU; no training |
| Best for roles | AI engineer agentic commerce / checkout (Shopify, Stripe, PayPal, Amazon, Instacart), agent platform engineer (A2A/MCP), applied scientist marketplaces & pricing |
| Relation to existing docs | New: agent-to-agent bargaining across two machines. The red-team tactic arm touches Ctrl-Z, so it is kept to one arm and cites Ctrl-Z. It differs from round-1 AgentMall (marketplace ranking) and Metagame (adversarial game search). Optuna concession tuning reuses the Changsheng AutoML story. | 
## Judge fixes folded in

- Frame the shield as an AP2 intent-mandate enforcer; present C1 as a constraint, headline the cost Pareto
- Sprint 0: products_mini with Qwen2.5-7B buyer via LiteLLM/Ollama, compare with published 11.76% before swapping judge
- Validate local extractor/judge vs API judge on ~100 dialogs; measure parser accuracy on counteroffers first
- Add a NegMAS-only templated-text arm to isolate the LLM's contribution
- Re-implement the small NegotiationArena game instead of chasing the repo; drop if unneeded
- Keep injection tactics to one arm and cite Ctrl-Z
- Report relative deltas only (wholesale prices are GPT-4o estimates)

## Brief

**Thesis.** When your shopping agent negotiates with someone else's agent, money decisions should not live inside LLM sampling. Haggle splits economics from eloquence:
- A NegMAS time-based concession strategy (Boulware, Conceder or aspiration) plus a Bayesian truncated-normal posterior over the counterparty's reservation price chooses accept, counter or walk.
- A deterministic shield enforces the budget, the round limit and no-disclosure, framed as an AP2 intent mandate.
- Qwen3-4B only parses the counterparty's message (JSON-schema constrained) and verbalises the next offer.
- Messages travel over a2a-sdk v1.0, with typed DataPart offers.

Claims:
- **C1.** Out-of-budget and out-of-wholesale anomalies are 0 by construction. This is a constraint, not a result. The measured part is the deal-rate loss, which must stay within a pre-registered non-inferiority margin.
- **C2 (headline).** Qwen3-4B plus the engine matches or beats raw Qwen2.5-7B (paper-matched, reproduced locally) and raw Qwen3-8B on price-reduction rate and relative profit at equal wall-clock. It is plotted as a surplus x deal-rate x leak Pareto.
- **C3.** Against a tactic library, the guarded delegate loses less surplus than the raw LLM by a pre-registered margin. Tactics: desperation persona, anchoring, fake competing offers, and fake 'system' text in A2A parts.
- **C4.** Budget leakage, measured as the counterparty's posterior error on the buyer's budget, raw vs guarded.

**Agent loop.**
1. The A2A server receives a message.
2. The parser extracts {price, terms, tactic}, and a text-vs-DataPart cross-check runs.
3. The opponent posterior is updated.
4. The engine decides; its concession parameters are tuned by Optuna on a dev product split.
5. The shield checks the offer and runs a leak checker (regex plus a small LLM), regenerating on failure.
6. The mouth verbalises the offer.
7. A DataPart offer is sent and an OTel span is logged.

Baselines: raw LLM end-to-end, raw LLM plus shield only, and engine plus templated text.

**Models and memory.**
- PC buyer: Qwen3-4B-Instruct-2507 Q4_K_M (~2.5 GB plus ~0.6-1.2 GB KV at 8k).
- Mac seller: Qwen3-8B MLX 4-bit (~5 GB) or Qwen2.5-7B Q4 (~4.7 GB).
- Stretch: QLoRA of the 4B parser on logged dialogs.

**Benchmarks.**
- A2A-NT (NeurIPS 2025) routes calls through LiteLLM. Lite run: 20 products x 5 budgets x 3 seeds = 300 negotiations of at most 30 rounds, roughly 10-20 h per arm worst case.
- Published: Qwen2.5-7B out-of-budget 11.76%. RL prompt optimisation took out-of-budget from 18.4% to 1.3%, but overpayment stayed at 8.1-8.3%. o4-mini out-of-budget 2.98%.
- NegotiationArena: desperation gives about +20% payoff against GPT-4; small-model numbers unknown.
- ANL agents are CPU-only.

**Evaluation.**
- Paired by product, budget and seed; bootstrap CIs.
- Parser accuracy reported first.
- The local judge is validated against the API judge on 100 dialogs before any comparison with published rates.

**Sprint sketch.**
- W1-2: products_mini reproduction, then the judge swap.
- W3-4: engine, shield and A2A across machines (week-4 demo).
- W5-6: C2 arms.
- W7-8: tactic library (C3).
- W9-10: leak metric (C4) and Optuna tuning.
- W11-12: write-up; parser QLoRA as stretch.

**Risks and mitigations.**
- The judge swap invalidates published comparisons. Validate it, and use the structured offers.
- Synthetic wholesale prices. Report relative deltas only.
- C1 is trivial, so lead with its cost.
- Parser errors could dominate. Measure them separately.

**Sources.**
- https://arxiv.org/abs/2506.00073
- https://github.com/ShenzheZhu/A2A-NT
- https://proceedings.mlr.press/v235/bianchi24a.html
- https://negmas.readthedocs.io/en/latest/overview.html
- https://pypi.org/project/anl/
- https://www.pypi.org/project/a2a-sdk
- https://developer.paypal.com/community/blog/PayPal-Agent-Payments-Protocol/
