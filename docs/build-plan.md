# Build plan — production-llm-platform

This document holds the full 12-milestone build plan, per-milestone
roadblocks and GPU budgets, deliverables structure, and phase-level
success criteria. Unlike `CLAUDE.md`, this document evolves as
milestones complete: append completion notes and link to milestone
reports below each milestone as it finishes.

For stable project context (architecture principles, conventions,
session protocol), see [CLAUDE.md](../CLAUDE.md).

---

## Incremental build plan — 12 milestones across 4 phases

### Phase 1 — Inference foundation (Milestones 1-4)

**Milestone 1: Naive baseline + dual-backend abstraction + measurement**
- Repo structure with dual-backend support (vLLM + MLX-LM)
- Benchmark harness from scratch
- Run baseline locally on M4 Max (Qwen 2.5 3B via MLX) and on rented
  A100 (Qwen 2.5 7B via vLLM)
- **Roadblock:** crank concurrency to 200 mixed prompts → head-of-line
  blocking
- Cloud GPU budget: ~$15-25

**Milestone 2: Priority-aware multi-tenant scheduler**
- Backend-agnostic scheduler with weighted fair queuing
- Tenants: 3 enterprise (premium SLA), 50 standard, 500 free
- **Roadblock:** misbehaving free-tier tenant degrades premium P99
- Cloud GPU budget: ~$10

**Milestone 3: Per-tenant KV cache isolation + accounting**
- Memory budgets with preemption, fairness index
- **Roadblock:** 32K-context query consuming 60% VRAM
- Cloud GPU budget: ~$60-80

**Milestone 4: Prefill/decode disaggregation + multi-LoRA serving**
- Separate prefill/decode workers, S-LoRA-style serving
- 200 LoRAs (one per tenant firm)
- **Roadblock:** 200 LoRAs vs 80GB VRAM, prefill starvation
- Cloud GPU budget: ~$70-90

### Phase 2 — Application layer (Milestones 5-7)

**Milestone 5: Production RAG pipeline with hybrid retrieval**
- Document ingestion, chunking strategies, hybrid retrieval (vector + BM25)
- Per-tenant document isolation (Firm A cannot see Firm B docs)
- **Roadblock:** 100K-page corpora cause retrieval misses + permission
  leaks
- Cloud GPU budget: ~$10

**Milestone 6: Agent orchestration with tool execution sandbox**
- LangGraph-based agent loop with planning, tool calls, recovery
- Customer agent: "find every contract expiring in Q2 and draft renewal
  notices"
- **Roadblock:** stuck loops ($40 bills), hallucinated clauses, multi-
  agent deadlock
- Cloud GPU budget: ~$15

**Milestone 7: Structured outputs + safety + guardrails**
- Grammar-constrained decoding, prompt injection defense, output filtering
- Confidence scoring, audit logging, red-team testing
- **Roadblock:** jailbreak exfiltrates other firm's confidential info
- Cloud GPU budget: ~$15

### Phase 3 — Reliability and operations (Milestones 8-10)

**Milestone 8: Production-grade observability for non-deterministic
systems**
- OpenTelemetry tracing across full request path
- Per-request structured logging with PII redaction
- Replay debugging for non-deterministic failures
- **Roadblock:** customer reports wrong answer, can't reproduce
- Cloud GPU budget: ~$20

**Milestone 9: Eval infrastructure + regression detection**
- Multiple eval types: golden datasets, LLM-as-judge, rule-based
- Statistical significance testing, A/B with shadow traffic
- **Roadblock:** new model "passes evals" but breaks 5% of enterprise
  queries
- Cloud GPU budget: ~$25

**Milestone 10: Cost economics + smart routing + budget governance**
- Per-tenant/feature/model cost attribution
- Smart routing across model tiers, shadow mode for quality validation
- **Roadblock:** $30K weekend bill, smart router quality regression,
  prompt cache corruption
- Cloud GPU budget: ~$30

### Phase 4 — Production hardening + flagship (Milestones 11-12)

**Milestone 11: Multi-region, deployment safety, chaos engineering**
- Blue-green for models, canary rollouts, multi-region failover
- Air-gap mode, backup/restore, chaos test suite
- **Roadblock:** chaos reveals deploy silently breaks 3 customers
- Cloud GPU budget: ~$25

**Milestone 12: The flagship benchmark + writeup + launch**
- Comprehensive benchmark vs vLLM and SGLang on H100
- 5000-7000 word blog post
- Launch on Hacker News, LinkedIn, vLLM Discord, AI infra communities
- Cloud GPU budget: ~$80-100

**Total cloud GPU budget: ~$385 across all 12 milestones.**

---

## Phase-level success criteria

**Phase 1 complete when:** the inference layer serves a multi-tenant
workload across both MLX (local) and vLLM (cloud) backends, with
priority-aware scheduling, per-tenant KV cache accounting, and
prefill/decode disaggregation working under realistic load. Benchmark
data demonstrates each scheduler/cache/disaggregation decision against
a documented alternative.

**Phase 2 complete when:** the platform serves a complete enterprise
workflow end-to-end — a law firm ingests documents, runs hybrid
retrieval with strict per-tenant isolation, executes a multi-step agent
task with tool calls, and returns guardrailed structured output without
leaking other tenants' data.

**Phase 3 complete when:** every production failure mode introduced in
M1-M7 is observable, debuggable from traces alone, validated by
regression evals, and cost-attributable per tenant/feature/model. A new
model can be promoted to production via shadow traffic with statistical
confidence.

**Phase 4 complete when:** the full system survives a chaos test suite
with documented blast radius, the flagship benchmark runs reproducibly
on H100, and the writeup is published with a measurable launch result
(HN front page, LinkedIn engagement, community pickup).

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
