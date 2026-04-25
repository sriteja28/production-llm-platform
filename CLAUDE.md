# production-llm-platform — Claude Code project context

This file is auto-loaded by Claude Code on every session in this repo.
It defines stable project context: what this is, the architectural
non-negotiables, the tech stack, the conventions, and the session
protocol. The full build plan, role-track positioning, and learning
anchors live in dedicated docs and evolve as the project progresses.

---

## Project description

This is **production-llm-platform** — a multi-tenant LLM serving
platform built for reasoning-and-agent workloads. The technical
anchor: scheduling, caching, and cost control when output length is
unpredictable (reasoning traces from 10 to 30,000+ tokens) and KV
state persists across hundreds of agent turns. The platform spans
inference, RAG, agents, evals, observability, and cost economics
end-to-end across 12 milestones. Anchor scenario: **Spider** — a
multi-tenant Glean-style knowledge agent platform for internal teams,
where agents reason across heterogeneous sources (Slack, Confluence,
Drive, GitHub) with per-org ACL isolation, per-org LoRA adapters
tuned to each org's terminology and conventions.

---

## Architecture principles (non-negotiable)

1. **Backend-agnostic core.** Scheduler, RAG retrieval, agent loop,
   eval harness, observability, and tenant manager work whether the
   inference backend is vLLM (cloud) or MLX-LM (local). Treat inference
   as a pluggable interface.

2. **Build on existing primitives, don't reinvent.** Wrap vLLM, use
   established vector DBs (Qdrant or pgvector), use OpenTelemetry, use
   ClickHouse for analytics. The wedge is integration depth, not
   recreating libraries.

3. **Pick a specific realistic scenario to anchor the entire project.**
   Default scenario: **Spider** — a multi-tenant Glean-style knowledge
   agent platform for internal teams. Each tenant is a company; agents
   reason across heterogeneous sources (Slack, Confluence, Drive,
   GitHub) with per-org ACL isolation, variable-length reasoning
   traces, KV state across hundreds of agent turns, and per-org LoRA
   adapters tuned to each org's terminology and conventions. All
   milestones build features for this workload rather than generic
   synthetic benchmarks. The concrete, named workload is what makes
   architecture choices defensible against open-source baselines
   (vLLM, SGLang, llm-d).

4. **Deliberately introduce production failures and debug them in
   public.** Each milestone has explicit "what kills this in production"
   step where we inject realistic failures with realistic data volume
   and debug systematically. This is what turns the project from a
   resume bullet into an interview story.

5. **Every architectural decision must be measured, not asserted.**
   Benchmarks before claims. Honest tradeoffs documented in ADRs.

6. **Production-realistic from day one.** Structured logging,
   OpenTelemetry tracing, Prometheus metrics, graceful shutdown, proper
   error wrapping, integration tests, no TODO placeholders.

7. **Documentation and ADRs are first-class deliverables.** Every
   meaningful design decision gets an ADR with alternatives considered.
   The README must be readable by a hiring manager who has 90 seconds.

8. **Local-first development discipline.** Every cloud GPU session has
   a written experiment plan with specific commands, expected duration,
   and expected cost before the meter starts.

---

## Tech stack and dev environment

- **Language:** Python 3.11+ primarily. Add Go only if a specific
  component clearly benefits (e.g., high-throughput proxy in front of
  the inference layer).
- **Inference backends:** vLLM (cloud, CUDA) and MLX-LM (local, M4 Max).
  Both sit behind a pluggable interface so the rest of the platform is
  backend-agnostic.
- **Vector DB:** Qdrant or pgvector (decision deferred to Milestone 5).
- **Retrieval (Milestone 5):** BM25 + dense embeddings + RRF fusion
  (k=60) + cross-encoder reranker (bge-reranker-v2 local /
  Cohere Rerank 3.5 cloud). Glean-style ACL filtering — post-
  retrieval, pre-context-assembly.
- **Observability:** OpenTelemetry GenAI semconv (experimental,
  dual-emit via `OTEL_SEMCONV_STABILITY_OPT_IN`) instrumented via
  **OpenLLMetry** (Traceloop), with **Langfuse** as the default
  backend; **Phoenix Span Replay** for deterministic replay (M8).
  Prometheus + `structlog` for non-LLM signals. Wired from M1
  (basic spans) — full coverage at M8.
- **Agents (Milestone 6):** **LangGraph** for state/flow + **Pydantic
  AI** for typed tool I/O inside nodes.
- **Structured outputs (Milestone 7):** **xgrammar** (vLLM v1 default,
  ~100x faster than Outlines via vocab partitioning).
- **Evals (Milestone 9):** three-layer split — **Inspect AI**
  (model-layer capability/safety), **DeepEval** (pytest-style app-
  layer), **Ragas** (retrieval). Reasoning-model-as-judge with
  binary pass/fail rubric (Hamel Husain doctrine).
- **Cost / routing (Milestone 10):** **RouteLLM**-style router +
  **Portkey**-style gateway; **OpenRouter** as marketplace reference.
- **Deployment (Milestone 11):** **KServe LLMInferenceService CRD +
  llm-d** backend (both CNCF), **Knative** for autoscaling,
  **LitmusChaos** for chaos suite.
- **Diagrams and documentation:**
  - **Mermaid** in markdown for sequence diagrams, agent flow
    diagrams, and component diagrams in milestone reports and ADRs
    (renders natively on GitHub — no separate build step). Use from
    Milestone 1.
  - [**KubeDiagrams**](https://github.com/philippemerle/KubeDiagrams)
    for auto-generated K8s architecture diagrams from manifests, Helm
    charts, and live cluster state. Used from Milestone 11 onwards
    when the Helm chart / K8s operator packaging lands;
    opportunistically earlier via the `kubectl diagrams` plugin to
    capture live cluster topologies for milestone reports. SVGs land
    in `docs/diagrams/`, regenerated by CI.
  - [**Structurizr**](https://structurizr.com/) (C4 model) for
    high-level architecture documentation in `docs/architecture.md`
    when the system is mature enough to warrant it (Milestone 11+).
  - [**DeepWiki by Cognition**](https://deepwiki.com) for
    auto-generated repo wiki with architecture diagrams, module
    explanations, dependency maps, and ask-the-codebase chat.
    **Submission deferred to Milestone 12** (end of build). Rationale:
    DeepWiki crawls current state — submitting during scaffolding or
    mid-build surfaces an early-stage repo to hiring managers
    clicking the badge. After M12, when the architecture is
    substantive and benchmarks are published, DeepWiki's
    auto-generated wiki becomes a high-signal surface area. Add
    `[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/sriteja28/production-llm-platform)`
    to README the day M12 launches.
- **Primary dev machine:** MacBook Pro M4 Max with substantial unified
  RAM. Used for ~70% of all work — orchestration, RAG, agents, evals,
  dashboards, integration tests, debugging. MLX-LM serves smaller
  models locally for free.
- **Cloud GPU rentals:** Modal primarily, RunPod for sustained
  workloads. Used only for CUDA-dependent work — vLLM benchmarks,
  full-scale multi-LoRA, prefill/decode disaggregation, the flagship
  benchmark.
- **Total cloud GPU budget across all 12 milestones: under $400.**

---

## Anchor papers and projects (2026)

The following are referenced by short name across milestones — name
them, don't paraphrase. They are the 2026 reasoning-and-agent
infrastructure stack.

**Anchor papers:**
- **DualPath** (DeepSeek, Feb 2026) — KV-cache I/O bottleneck in
  long-horizon agent workloads
- **EAGLE-3** (NeurIPS 2025) — current SOTA token-level speculative
  decoding
- **Lookahead Reasoning** (NeurIPS 2025) — step-level draft+verify
  for reasoning chains
- **SpecReason** (Apr 2025) — selective small-model offload for easy
  reasoning steps
- **PagedAttention** (vLLM original) — KV cache fragmentation
  solution
- **FlashAttention** — IO-aware attention algorithm (intuition only)
- **RadixAttention** (SGLang) — radix-tree prefix cache
- **S-LoRA** — multi-LoRA serving (paper); production stack uses
  vLLM Punica/SGMV kernels
- **Sarathi-Serve** (OSDI 2024) — chunked prefill (alternative to
  full P/D disaggregation)

**Anchor projects:**
- **vLLM v1** + Punica/SGMV multi-LoRA + Reasoning Budget feature
- **SGLang** + RadixAttention
- **llm-d** (CNCF sandbox, March 2026) — distributed inference
  scheduler with tiered prefix cache
- **KServe** (CNCF incubating, Nov 2025) — LLMInferenceService CRD
- **LMCache** — KV offload layer
- **xgrammar** — structured output (vLLM v1 default)
- **LangGraph + Pydantic AI** — agent state + typed tools
- **Inspect AI / DeepEval / Ragas** — eval layers
- **OpenLLMetry / Langfuse / Arize Phoenix** — LLM observability
- **LitmusChaos** — CNCF chaos engineering
- **RouteLLM / Portkey / OpenRouter** — routing + gateways
- **KubeDiagrams** — auto-generated K8s architecture diagrams

---

## Conventions and constraints

- Every dependency must be justified with what alternatives we rejected
- All benchmark numbers from actual runs, never estimates. If not yet
  measured, the report must say "not yet measured"
- Cite real papers, real GitHub repos, real file paths in vLLM, MLX-LM,
  SGLang source. Do not invent sources. If unsure, say "verify this
  reference"
- Production-grade code from day one: structured logging with structlog,
  OpenTelemetry, proper error handling, graceful shutdown, integration
  tests
- Documentation readable by Staff-level hiring managers, not beginners
- No emojis in code, comments, commit messages, or docs
- Every milestone produces something I can demo end-to-end
- ADR filename format: `adr/MX-<slug>.md` (e.g., `adr/M1-dual-backend-interface.md`)
- Milestone artifacts: `milestones/MX/PLAN.md`, `milestones/MX/REPORT.md`
- Blog post drafts: `content/posts/milestone-MX.md`

---

## Cross-cutting conventions (apply to every milestone)

1. **Cost-per-correct-answer in every milestone report.** Not just
   M12. This is the metric that distinguishes the project from
   generic LLM serving portfolios at the $400K hire tier.
2. **Hamel Husain eval doctrine.** Every milestone with quality
   claims uses binary pass/fail judge + classifier validation
   (precision / recall on a labeled set of 100+ examples) + paired
   bootstrap or McNemar's test for statistical significance.
3. **OpenTelemetry GenAI semconv from M1.** All LLM instrumentation
   goes through OpenLLMetry against the GenAI semconv from the first
   milestone. M8 brings full agent-loop coverage and Phoenix Span
   Replay; M1-M7 wire basic spans against the same schema.
4. **OWASP coverage doc.** `docs/owasp-coverage.md` maps OWASP LLM
   Top 10 2026 + OWASP Agentic Top 10 2026 to specific milestone
   test cases. Updated as milestones complete.
5. **vLLM disagg prefill caveat.** Disagg prefill is still
   experimental as of April 2026; improves tail ITL but not
   throughput. Frame it as a tail-latency vs throughput tradeoff,
   with Sarathi-Serve chunked prefill as the alternative — never
   claim throughput win.

---

## Directory map

```
production-llm-platform/
├── README.md                      # 90-second orientation for any reader
├── CLAUDE.md                      # This file — stable project context
├── learning-foundations.md        # 4-week pre-build curriculum + self-test
├── .gitignore
├── docs/
│   └── build-plan.md              # 12-milestone plan, GPU budgets, deliverables
├── adr/                           # Architecture decision records (MX-*.md)
├── milestones/                    # M1-M12 build artifacts (PLAN.md, REPORT.md)
└── content/posts/                 # Incremental blog post drafts
```

---

## Pointers to deeper docs

- **Full 12-milestone plan, roadblocks, GPU budgets, deliverables
  structure:** [docs/build-plan.md](./docs/build-plan.md)
- **4-week pre-build curriculum and self-test:** [learning-foundations.md](./learning-foundations.md)

---

## Start a session

### Pre-build checklist (verify once, before Milestone 1)

- **Day 0 of `learning-foundations.md` complete** — Python 3.11+ venv
  with MLX-LM working on Metal, DeepSeek-R1-Distill-Llama-8B (4-bit
  MLX) generates reasoning traces locally, Modal account active with
  $50 credit, vLLM Discord joined for lurking
- Foundations curriculum complete (`learning-foundations.md`
  self-test passes 10/12+)
- GitHub repo initialized with proper `.gitignore`
- This `CLAUDE.md` is in the repo root and committed

### Session protocol — teach, build, break, debug, understand

For Milestone 1, and repeated for every subsequent milestone:

**Phase A — Scaffold and align.** Initialize the milestone scaffolding
(directory structure, interface designs, harness designs, scenario
docs). Stop. Walk me through the design. Wait for my approval before
implementing.

**Phase B — Implement and break.** Implement the milestone fully — all
backends, the harness, the deliberate roadblock, the debug walkthrough.
Stop. Walk me through what was built, why each decision was made, what
tradeoffs were considered, what the data showed, what I should learn
before moving on. Generate 5-10 concept questions I should be able to
answer fluently before proceeding (append them to
[docs/learning-anchors.md](./docs/learning-anchors.md)).

**Phase C — Confirm and proceed.** After I confirm understanding by
answering the concept questions, proceed to the next milestone. Repeat
the teach → build → break → debug → understand pattern.

This pacing is intentional. The point is to make me genuinely deep
across the AI engineering stack, not to produce a finished artifact I
cannot defend in interviews.

### Clarifying questions before scaffolding a new milestone

Before starting any milestone scaffolding, ask 4-6 clarifying questions
covering: model choice, vector DB choice (Milestone 5+), agent
framework specifics (Milestone 6+), vLLM positioning, scenario
specifics, and any milestone-specific decisions. Do not scaffold until
the answers are in.

### Kickoff phrase

When I'm ready to start Milestone 1, I'll say: "Let's start with
Phase A — Milestone 1 scaffolding."
