# Build plan — production-llm-platform

This document holds the full 12-milestone build plan, per-milestone
roadblocks and GPU budgets, deliverables structure, and phase-level
success criteria. Unlike `CLAUDE.md`, this document evolves as
milestones complete: append completion notes and link to milestone
reports below each milestone as it finishes.

For stable project context (architecture principles, conventions,
session protocol), see [CLAUDE.md](../CLAUDE.md).

---

## Technical anchor

The platform is built around **reasoning-and-agent inference**:
scheduling, caching, and cost control when output length is
unpredictable (reasoning traces from 10 to 30,000+ tokens) and KV
state persists across hundreds of agent turns. Every milestone in
Phase 1-3 sharpens one aspect of that core problem; Phase 4
demonstrates the integrated system at scale.

This anchor is informed by 2026 production pain: vLLM's Q2 2026
roadmap names reasoning-induced preemption and prefill head-of-line
blocking as unsolved scheduler problems; the DeepSeek DualPath paper
(Feb 2026) establishes that in long-horizon agent workloads,
KV-cache I/O — not compute — is the bottleneck. The platform is
designed to confront these problems directly rather than working
around them.

---

## Incremental build plan — 12 milestones across 4 phases

### Phase 1 — Inference foundation (Milestones 1-4)

**Milestone 1: Reasoning-aware baseline + dual-backend abstraction + measurement**
- Repo structure with dual-backend support (vLLM + MLX-LM)
- Benchmark harness with reasoning-specific metrics: output-length
  distribution, time-to-first-thinking-token, trace-length variance,
  cost-per-correct-answer
- Models: locally Qwen3-thinking-7B / DeepSeek-R1-Distill-Llama-8B via
  MLX on M4 Max; cloud rented A100 via vLLM with reasoning mode on
- Workload: mixed short prompts + reasoning queries simulating the
  anchor scenario
- **Roadblock:** under mixed workload, a single 30K-token reasoning
  trace evicts other tenants' KV state — head-of-line blocking that's
  qualitatively worse than non-reasoning HoL because eviction patterns
  are unpredictable
- Cloud GPU budget: ~$15-25

**Milestone 2: Reasoning-budget-aware multi-tenant scheduler**
- Backend-agnostic scheduler with weighted fair queuing and
  reasoning-budget awareness
- Tenants: 3 enterprise (premium SLA, full reasoning), 50 standard
  (capped reasoning budget), 500 free (no reasoning or hard cap)
- **Roadblock:** free tenant requests reasoning, blows past budget
  mid-trace — degrades premium P99 even with WFQ
- Cloud GPU budget: ~$10

**Milestone 3: Per-tenant KV cache isolation under reasoning pressure**
- Memory budgets with preemption, fairness index, prefix-cache
  awareness
- Reasoning traces consume 5-30x more KV than standard requests —
  accounting must reflect this
- **Roadblock:** 32K-token reasoning trace from one tenant consumes
  60%+ VRAM; fairness index breaks
- Cloud GPU budget: ~$60-80

**Milestone 4: Prefill/decode disaggregation + multi-LoRA + reasoning-aware speculative decoding**
- Separate prefill/decode workers, S-LoRA-style serving
- 200 LoRAs (one per tenant firm) plus speculative decoder draft
  models
- **Roadblock:** speculative decoder degrades catastrophically on
  reasoning chains (drafts can't predict reasoning steps); prefill
  starvation under mixed reasoning + non-reasoning workload
- Cloud GPU budget: ~$70-90

### Phase 2 — Application layer (Milestones 5-7)

**Milestone 5: Production RAG pipeline with reasoning-aware retrieval**
- Document ingestion, chunking strategies, hybrid retrieval (vector +
  BM25)
- Per-tenant document isolation (Firm A cannot see Firm B docs)
- Reasoning-aware retrieval: when does the agent reason vs retrieve?
- **Roadblock:** 100K-page corpora cause retrieval misses + permission
  leaks; reasoning agent over-fetches and burns tokens
- Cloud GPU budget: ~$10

**Milestone 6: Multi-turn agent orchestration with KV-cache-aware tool execution**
- LangGraph-based agent loop with planning, tool calls, recovery
- Customer agent: "find every contract expiring in Q2 and draft
  renewal notices" (multi-step reasoning over firm corpus)
- KV state persists across agent turns; tool-call boundaries used as
  natural cache breakpoints (DualPath-style hierarchical KV management)
- **Roadblock:** stuck reasoning loops ($40 bills), hallucinated
  clauses, KV-cache thrash under multi-agent deadlock
- Cloud GPU budget: ~$15

**Milestone 7: Structured outputs + reasoning-trace auditing + guardrails**
- Grammar-constrained decoding, prompt injection defense (including
  injection in reasoning traces)
- Confidence scoring, reasoning-step audit logs, red-team testing
- **Roadblock:** jailbreak exfiltrates other firm's confidential info
  via reasoning-trace leakage
- Cloud GPU budget: ~$15

### Phase 3 — Reliability and operations (Milestones 8-10)

**Milestone 8: Production-grade observability for reasoning + agent workloads**
- OpenTelemetry tracing across the full reasoning + agent loop
- Per-step structured logging with PII redaction
- Replay debugging for non-deterministic reasoning failures
- **Roadblock:** customer reports wrong answer; can't reproduce because
  reasoning trace + tool-call ordering varies between runs
- Cloud GPU budget: ~$20

**Milestone 9: Eval infrastructure for reasoning + regression detection**
- Reasoning evals (is the reasoning sound, not just is the answer
  right?), LLM-as-judge for reasoning quality, rule-based for
  tool-call correctness
- Statistical significance testing, A/B with shadow traffic
- **Roadblock:** new reasoning model "passes evals" but breaks 5% of
  enterprise queries; reasoning-quality regression invisible to
  answer-only metrics
- Cloud GPU budget: ~$25

**Milestone 10: Reasoning cost economics + smart routing + budget governance**
- Per-tenant/feature/model cost attribution including reasoning trace
  cost
- Smart routing: when to use reasoning model vs cheaper non-reasoning
  alternative; when to cap thinking budget; cache-hit-aware routing
- Shadow mode for quality validation
- **Roadblock:** $30K weekend bill from runaway reasoning, smart
  router quality regression, prompt cache corruption
- Cloud GPU budget: ~$30

### Phase 4 — Production hardening + flagship (Milestones 11-12)

**Milestone 11: Multi-region, deployment safety, chaos engineering**
- Blue-green for models, canary rollouts, multi-region failover
- Air-gap mode, backup/restore, chaos test suite
- **Roadblock:** chaos reveals deploy silently breaks 3 customers'
  reasoning workflows
- Cloud GPU budget: ~$25

**Milestone 12: The flagship benchmark + writeup + launch**
- Comprehensive benchmark of reasoning-aware serving vs vLLM and
  SGLang on production-realistic reasoning + agent workload (mixed
  short prompts, multi-turn agents, variable trace length)
- 5000-7000 word blog post
- Launch on Hacker News, LinkedIn, vLLM Discord, AI infra communities
- Cloud GPU budget: ~$80-100

**Total cloud GPU budget: ~$385 across all 12 milestones.**

---

## Phase-level success criteria

**Phase 1 complete when:** the inference layer serves a multi-tenant
reasoning workload across both MLX (local) and vLLM (cloud) backends,
with reasoning-budget-aware scheduling, per-tenant KV cache accounting
under variable trace pressure, prefill/decode disaggregation, and
speculative decoding tuned for reasoning chains. Benchmark data
demonstrates each decision against a documented alternative.

**Phase 2 complete when:** the platform serves a complete enterprise
reasoning-agent workflow end-to-end — a law firm ingests documents, an
agent runs multi-step reasoning over them with strict per-tenant
isolation, tool calls execute against the firm corpus without leaking,
and reasoning-trace auditing catches injection attempts.

**Phase 3 complete when:** every production failure mode introduced in
M1-M7 is observable from reasoning + tool-call traces, debuggable
through deterministic replay, validated by reasoning-quality
regression evals, and cost-attributable per
tenant/feature/model/reasoning-step. A new reasoning model can be
promoted to production via shadow traffic with statistical confidence
on both answer quality and reasoning soundness.

**Phase 4 complete when:** the full system survives a chaos test
suite with documented blast radius on reasoning workloads, the
flagship benchmark runs reproducibly on H100 against vLLM/SGLang
baselines, and the writeup is published with a measurable launch
result.

---

## Deliverables per milestone

For each milestone, generate:

1. `./milestones/MX/PLAN.md` — what we're building, expected learnings,
   expected production roadblocks, local-vs-cloud workflow plan,
   estimated GPU cost
2. The actual code, tests, benchmark scripts, dashboards
3. `./milestones/MX/REPORT.md` — what we built, what data showed, what
   broke and how we debugged it, lessons learned
4. `./adr/MX-*.md` — architecture decision records with alternatives
   considered and tradeoffs accepted
5. `./content/posts/milestone-MX.md` — short blog post draft for
   incremental publication

---

## Completion log

As milestones finish, append entries here with links to the milestone
report, headline benchmark numbers, and the actual cloud GPU spend vs
budget.

_(empty — no milestones completed yet)_
