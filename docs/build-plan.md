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

## Anchor scenario: Spider

**Spider** is a multi-tenant Glean-style knowledge agent platform for
internal teams. Every milestone builds features for this workload
rather than generic synthetic benchmarks.

**Tenants:** organizations — each tenant is a company's instance,
with its own data sources, permissions, and usage tier (3 enterprise,
50 standard, 500 free).

**Data sources per tenant:** heterogeneous mix of Slack, Confluence,
Google Drive, GitHub, and internal wikis. Each source has its own ACL
model (channel membership, doc permissions, repo access). Per-org
isolation is strict: Org A cannot see Org B's data, ever.

**Query types and distribution:**
- 60% short Q&A (~200 input, ~100 output tokens) — "what does CTRP
  stand for in our docs?"
- 30% reasoning queries (~800 input, 5,000-15,000 output tokens
  including reasoning trace) — "summarize the auth migration
  decision and the tradeoffs we discussed"
- 10% long-horizon agent queries (~1,500 input, 20,000-30,000 output
  tokens plus 5-20 tool calls) — "what happened on Project Atlas
  last sprint? Find the PRs, Slack discussions, and design docs"

**Per-org LoRA adapters:** each org has its own fine-tune capturing
their terminology, project naming conventions, code style, and
internal jargon.

**Why Spider stresses the technical anchor:**
- *Variable trace length.* Reasoning queries produce 5x-30x more
  output than short Q&A — head-of-line blocking under mixed workload
  (M1-M2).
- *KV state across turns.* Long-horizon agent queries make 5-20 tool
  calls each, with KV state needing to persist across the full agent
  loop (M3, M6).
- *Per-org isolation.* ACL filtering during retrieval AND during
  reasoning trace generation — leak prevention is non-trivial when
  reasoning traces can cross-reference content (M5, M7).
- *Multi-source heterogeneous retrieval.* Mixed signals (recency,
  semantic, keyword, author/permission filters) across sources with
  different schemas — RAG at production scale (M5).

---

## Incremental build plan — 12 milestones across 4 phases

### Phase 1 — Inference foundation (Milestones 1-4)

**Milestone 1: Reasoning-aware baseline + dual-backend abstraction + measurement**
- Repo structure with dual-backend support (vLLM + MLX-LM)
- Benchmark harness with reasoning-specific metrics: output-length
  distribution, time-to-first-thinking-token, trace-length variance,
  cost-per-correct-answer
- Models: locally DeepSeek-R1-Distill-Llama-8B (4-bit MLX) on M4 Max;
  cloud rented A100 via vLLM with `--reasoning-parser deepseek_r1`
  (online-serving / chat completions endpoint only)
- OpenTelemetry GenAI semconv instrumented via OpenLLMetry from
  day one (basic spans; full coverage at M8)
- Workload: mixed Spider-scenario queries (60% short / 30% reasoning
  / 10% long-horizon agent)
- **Roadblock:** under mixed workload, a single 30K-token reasoning
  trace evicts other tenants' KV state — head-of-line blocking that's
  qualitatively worse than non-reasoning HoL because eviction patterns
  are unpredictable
- Cloud GPU budget: ~$15-25

**Milestone 2: Reasoning-budget-aware multi-tenant scheduler**
- Build on top of vLLM priority scheduling (PR #5958) + the merged
  Reasoning Budget feature (terminates thinking at a token cap with
  stop-message injection); add weighted-fair-queuing layer on top
- Tenants: 3 enterprise (premium SLA, full reasoning), 50 standard
  (capped reasoning budget), 500 free (no reasoning or hard cap)
- Compare against **llm-d v0.5 inference scheduler** (filter+score
  with P/D-, KV-, SLA-, load-awareness) as the credible production
  baseline — not stock vLLM FCFS
- **Roadblock:** reproduce the failure mode named in vLLM Q2 2026
  roadmap (issue #39749 — preemption storms + prefill HoL under
  reasoning workload) with the Spider mix; demonstrate that the
  budget-aware WFQ layer keeps premium P99 stable while llm-d v0.5
  alone degrades
- Cloud GPU budget: ~$10

**Milestone 3: Per-tenant KV cache isolation under reasoning pressure**
- Build per-tenant accounting on top of vLLM v1 hash-based automatic
  prefix caching + LMCache CPU offload + llm-d tiered prefix cache
  (GPU → CPU → remote)
- Reasoning traces consume 5-30x more KV than standard requests —
  accounting reflects this explicitly
- Ablate against SGLang RadixAttention to compare radix-tree vs
  hash-block prefix structures for tenant isolation
- **Roadblock:** specific failure — DeepSeek-R1-Distill-Llama-8B in
  fp16 on A100 80GB, a single 32K-token reasoning trace consumes 60%+
  of the KV pool; the fairness index breaks before LMCache offload
  kicks in. DualPath framing applies: KV-I/O cost dominates compute
  at this trace length
- Cloud GPU budget: ~$60-80
- ADR: vLLM hash-block APC vs SGLang radix-tree for multi-tenant
  prefix sharing — which structure leaks fewer cache lines across
  tenants?

**Milestone 4: Prefill/decode disaggregation + multi-LoRA + reasoning-aware speculative decoding**
- Multi-LoRA via vLLM Punica/SGMV kernels, 200 LoRAs (one per
  tenant org)
- Speculative decoding stack composed: **EAGLE-3** (NeurIPS 2025)
  token-level + **Lookahead Reasoning** (NeurIPS 2025) step-level
  + **SpecReason** (selective offload of easy reasoning steps to
  small model) — ablate each individually plus the composition
  against vanilla EAGLE-2 to demonstrate why naive token-level spec
  dec breaks on CoT
- Prefill/decode evaluation: **vLLM disagg prefill is still
  experimental and improves tail ITL but not throughput**; measure
  the tradeoff vs Sarathi-Serve chunked prefill rather than asserting
  disagg wins
- **Roadblock:** vanilla EAGLE-2 acceptance rate collapses on
  reasoning chains; 200-LoRA prefill starvation under mixed workload
- Cloud GPU budget: ~$70-90

### Phase 2 — Application layer (Milestones 5-7)

**Milestone 5: Production RAG pipeline with multi-source retrieval and ACL isolation**
- Heterogeneous source ingestion (Slack, Confluence, Drive, GitHub,
  wikis); incremental sync with ACL deltas; per-source rate-limit +
  auth (OAuth/PAT) handling. Connector engineering is non-trivial —
  consider stubbing 2 of 4 sources for portfolio scope, full
  implementation if time allows
- Hybrid retrieval: BM25 + dense embeddings + RRF fusion (k=60) +
  cross-encoder reranker (bge-reranker-v2 local, Cohere Rerank 3.5
  cloud option for ADR comparison)
- Strict per-org ACL isolation following Glean's pattern: filter
  candidates **post-retrieval, pre-context-assembly** — never trust
  the LLM with unfiltered context
- Multi-tenant isolation pattern: Silo per enterprise tenant, Pool
  for standard/free tier (ADR)
- Reasoning-aware retrieval: when does the agent reason vs retrieve?
- **Roadblock:** 100K-document heterogeneous corpora cause retrieval
  misses + ACL leaks across sources; reasoning agent over-fetches and
  burns tokens
- Cloud GPU budget: ~$10

**Milestone 6: Multi-turn agent orchestration with KV-cache-aware tool execution**
- LangGraph (state/flow) + Pydantic AI (typed tool I/O inside nodes)
  composition. ADR justifying choice over OpenAI Agents SDK,
  smolagents, Strands, Microsoft Agent Framework
- Customer agent: "what happened on Project Atlas last sprint?
  Find the PRs, Slack discussions, and design docs" (multi-step
  reasoning across heterogeneous sources)
- Anthropic prompt caching for stable system prompt + tool definitions
  (5-min TTL: 1.25x write cost, 0.1x read; 1-hour TTL: 2x write cost;
  workspace-isolated since Feb 2026 — relevant for multi-tenant)
- KV state persists across agent turns; tool-call boundaries used as
  natural cache breakpoints (DualPath-style hierarchical KV management)
- Quality bar: reference SWE-bench Verified, TAU-bench, GAIA
- **Roadblock:** stuck reasoning loops ($40 bills), hallucinated
  cross-source claims, KV-cache thrash under multi-agent deadlock,
  prompt-cache breakage when "stable" prefix drifts
- Cloud GPU budget: ~$15

**Milestone 7: Structured outputs + reasoning-trace auditing + guardrails**
- Grammar-constrained decoding via **xgrammar** (vLLM v1 default,
  pushdown-automaton-based, ~100x faster than Outlines via vocab
  partitioning). ADR comparing xgrammar / Outlines /
  lm-format-enforcer / BAML / Instructor
- Prompt injection defense including injection in reasoning traces
- Confidence scoring, reasoning-step audit logs, red-team testing
  mapped to **OWASP LLM Top 10 2026** + **OWASP Agentic Top 10 2026**
  (separate doc, includes Thought/Observation Injection — directly
  maps to the Spider scenario's reasoning-trace forge attack)
- **Roadblock:** jailbreak exfiltrates another org's internal data
  via reasoning-trace leakage (e.g., reasoning trace cites a doc
  the user shouldn't have access to)
- Cloud GPU budget: ~$15

### Phase 3 — Reliability and operations (Milestones 8-10)

**Milestone 8: Production-grade observability for reasoning + agent workloads**
- OpenTelemetry GenAI semconv (experimental, dual-emit via
  `OTEL_SEMCONV_STABILITY_OPT_IN`) instrumented via OpenLLMetry
  (Traceloop). Wired in M1 already; M8 brings full coverage across
  the agent loop
- Backend: Langfuse as primary; ADR comparing
  Langfuse / Arize Phoenix / Helicone / Braintrust for multi-tenant
  LLM observability
- **Phoenix Span Replay** for deterministic replay of
  non-deterministic reasoning + tool-call failures: record LLM
  responses + tool outputs during live run, substitute on replay
- Per-step structured logging with PII redaction
- **Roadblock:** customer reports wrong answer; can't reproduce
  because reasoning trace + tool-call ordering varies between runs.
  Phoenix Span Replay is the fix
- Cloud GPU budget: ~$20

**Milestone 9: Eval infrastructure for reasoning + regression detection**
- Three-layer eval split (ADR justifying):
  - **Inspect AI** (UK AISI) — capability/safety evals at model layer
  - **DeepEval** — pytest-style application-layer
  - **Ragas** — retrieval-specific
- Reasoning-model-as-judge with **binary pass/fail rubric** (Hamel
  Husain doctrine, 2026): treat the judge as a classifier, validate
  with precision/recall on a labeled 100+ example set, paired
  bootstrap / McNemar's test for statistical significance on binary
  outputs (NOT t-test on Likert means)
- A/B with shadow traffic
- **Roadblock:** new reasoning model "passes evals" but breaks 5%
  of enterprise queries; reasoning-quality regression invisible to
  answer-only metrics
- Cloud GPU budget: ~$25

**Milestone 10: Reasoning cost economics + smart routing + budget governance**
- Per-tenant/feature/model/reasoning-step cost attribution: split
  tokens by (tenant, feature, model, reasoning-vs-final-answer,
  cached-vs-uncached input)
- Smart routing: **RouteLLM-style router** for tier selection +
  **Portkey-style gateway** for policy / fallback / cache. ADR on
  the router-vs-gateway distinction. **OpenRouter** as marketplace
  reference
- Anthropic prompt caching dynamics surfaced in cost model: 5-min
  cache pays off after 1 read, 1-hour after 2 reads, 0.1x read cost,
  workspace-isolated
- Shadow mode for quality validation
- **Roadblock:** $30K weekend bill from runaway reasoning, smart
  router quality regression, and **prompt cache corruption** —
  "stable" prefix drifts on a whitespace change, cache writes burn
  money without reads. Detect via cache-hit rate by prefix-hash
- Cloud GPU budget: ~$30

### Phase 4 — Production hardening + flagship (Milestones 11-12)

**Milestone 11: Multi-region, deployment safety, chaos engineering**
- Spider Helm chart deploys onto **KServe LLMInferenceService CRD +
  llm-d backend** (both CNCF as of March 2026 — KServe incubating
  Nov 2025, llm-d sandbox March 2026). Don't reinvent the operator;
  build on the converged ecosystem
- Knative for request/concurrency-based autoscaling with
  scale-to-zero (the cost-control primitive)
- Blue-green for models, canary rollouts, multi-region failover
- Chaos suite via **LitmusChaos** (CNCF incubating; ships an MCP
  server so chaos experiments can be driven from the agent loop —
  high-leverage demo)
- Air-gap mode, backup/restore
- Architecture diagrams auto-generated from the Helm chart and live
  cluster state via [KubeDiagrams](https://github.com/philippemerle/KubeDiagrams).
  CI runs `philippemerle/KubeDiagrams@main` on manifest changes;
  generated SVGs land in `docs/diagrams/` and embed in `README.md`,
  `docs/architecture.md`, and milestone reports. The kubectl plugin
  (`kubectl diagrams`) is also used during dev to capture live
  cluster topologies for incident write-ups
- **Roadblock:** chaos reveals deploy silently breaks 3 customers'
  reasoning workflows
- Cloud GPU budget: ~$25
- ADR: KServe + llm-d over a custom operator (CNCF momentum, March
  2026 ecosystem standardization)
- **Note on licensing for the spinout.** When packaging is spun out
  as a separate `spider` open-source repo (productized Helm chart /
  K8s operator), license it under **Apache-2.0** rather than MIT.
  Apache-2.0 is the AI-infrastructure ecosystem norm (vLLM, SGLang,
  MLX, llm-d, Kubernetes — all Apache-2.0) and its explicit patent
  grant matters once a project is being productized for adoption.
  The portfolio repo `production-llm-platform` stays MIT.

**Milestone 12: The flagship benchmark + writeup + launch**
- Reproducible benchmark harness: fork **vLLM
  `benchmark_serving.py`** + SGLang harness, drive with the Spider
  workload mix (60% short / 30% reasoning / 10% long-horizon agent)
- Compare four serving stacks side-by-side:
  - vLLM v1 + EAGLE-3 (token-level spec dec only)
  - SGLang + RadixAttention
  - llm-d v0.5 + KServe
  - Our reasoning-aware stack (vLLM + Reasoning Budget + WFQ +
    EAGLE-3 + Lookahead Reasoning + Punica multi-LoRA + LMCache)
- Quality on **SWE-bench Verified** subset + **TAU-bench**
- **Primary metric: cost-per-correct-answer** (the metric that
  matters at $400K hire tier — distinguishes this project from
  generic LLM serving portfolios)
- Reproducibility: pinned commits, Modal scripts, exact GPU-hour
  cost published per run; runbook written before the meter starts
- 5000-7000 word blog post on Hashnode publication
- Launch on Hacker News, LinkedIn, vLLM Discord, AI infra
  communities
- Cloud GPU budget: ~$80-100

**Total cloud GPU budget: ~$385 across all 12 milestones.**

---

## Cross-cutting conventions

These apply across every milestone, not just the milestone where the
relevant primitive is introduced.

1. **Cost-per-correct-answer in every milestone report.** Not just
   M12. This is the metric that distinguishes the project from
   generic LLM serving portfolios.
2. **Hamel Husain eval doctrine.** Every milestone with quality
   claims uses binary pass/fail judge + classifier validation
   (precision/recall on labeled set) + paired bootstrap CI.
3. **OpenTelemetry GenAI semconv from M1.** All instrumentation
   goes through OpenLLMetry against the GenAI semconv. M8 brings
   full coverage and Phoenix Span Replay; M1-M7 wire basic spans.
4. **OWASP coverage doc.** `docs/owasp-coverage.md` maps OWASP LLM
   Top 10 2026 + OWASP Agentic Top 10 2026 to specific milestone
   test cases. Updated as milestones complete.

---

## Phase-level success criteria

**Phase 1 complete when:** the inference layer serves a multi-tenant
reasoning workload across both MLX (local) and vLLM (cloud) backends,
with reasoning-budget-aware scheduling layered on vLLM priority
scheduling, per-tenant KV cache accounting on top of vLLM v1 APC +
LMCache, prefill/decode tradeoffs measured against Sarathi-Serve, and
EAGLE-3 + Lookahead Reasoning composition tuned for reasoning chains.
Benchmark data demonstrates each decision against llm-d v0.5 as the
production baseline.

**Phase 2 complete when:** the platform serves a complete enterprise
reasoning-agent workflow end-to-end — an org ingests heterogeneous
source data (Slack, Confluence, Drive, GitHub) with Glean-style
post-retrieval ACL filtering, an agent runs multi-step reasoning
across them via LangGraph + Pydantic AI with strict per-org and
per-source isolation, tool calls execute without leaking cross-org
or cross-permission data, and reasoning-trace auditing catches OWASP
Agentic Top 10 attack patterns including Thought/Observation
Injection.

**Phase 3 complete when:** every production failure mode introduced
in M1-M7 is observable via OpenLLMetry GenAI semconv traces in
Langfuse, deterministically replayable through Phoenix Span Replay,
validated by binary-rubric judge evals (Inspect AI / DeepEval /
Ragas) with paired-bootstrap statistical confidence, and
cost-attributable per tenant / feature / model / reasoning-step /
cache-tier. A new reasoning model can be promoted via shadow traffic
with statistical confidence on both answer quality and reasoning
soundness.

**Phase 4 complete when:** the full system runs as a Spider Helm
chart on KServe + llm-d, survives a LitmusChaos suite with documented
blast radius on reasoning workloads, the flagship benchmark runs
reproducibly across four serving stacks on H100 with SWE-bench
Verified + TAU-bench quality, and the writeup is published with a
measurable launch result on cost-per-correct-answer as the headline
metric.

---

## Deliverables per milestone

For each milestone, generate:

1. `./milestones/MX/PLAN.md` — what we're building, expected learnings,
   expected production roadblocks, local-vs-cloud workflow plan,
   estimated GPU cost
2. The actual code, tests, benchmark scripts, dashboards
3. `./milestones/MX/REPORT.md` — what we built, what data showed, what
   broke and how we debugged it, lessons learned, cost-per-correct-
   answer numbers
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

---

## Post-Milestone 12 follow-ups

Things to do after the M12 launch lands. Don't start any of these
before M12 is published — they all gate on the launch.

1. **Spin out `learning-foundations.md` as a separate repo** (e.g.,
   `ai-infra-foundations`). After M12 ships, the curriculum has
   lived experience baked in — refine it based on what actually
   mattered vs what was wasted time, then publish as a standalone
   shareable resource. Cross-link from the M12 flagship post:
   "here's how I prepared for this work." Two repos owning two
   distinct topics catches more recruiter SEO than one repo trying
   to do both.

2. **Spin out the productized platform as `spider` repo with
   Apache-2.0.** See the licensing note in Milestone 11. The
   portfolio repo `production-llm-platform` stays MIT and remains
   the build journal; `spider` becomes the clean OSS Helm chart /
   K8s operator that companies can adopt.

3. **Submit `production-llm-platform` to DeepWiki.** Per the
   `Diagrams and documentation` entry in CLAUDE.md, DeepWiki
   submission is deferred until M12 launch so the auto-generated
   wiki captures the substantive end state, not scaffolding. Add the
   `[![Ask DeepWiki](https://deepwiki.com/badge.svg)]` badge to
   README the day M12 publishes.
