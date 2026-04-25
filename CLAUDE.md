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
end-to-end across 12 milestones. Anchor scenario: a multi-tenant SaaS
where lawyers run multi-step reasoning agents over firm-specific
document libraries — per-firm LoRA adapters, SOC 2 compliance
pressure, real cost economics.

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
   Default scenario: a multi-tenant SaaS where lawyers run multi-step
   reasoning agents over firm-specific document libraries —
   variable-length reasoning traces, KV state across hundreds of agent
   turns, per-firm LoRA adapters, SOC 2 compliance pressure, real cost
   economics. All milestones build features for this workload rather
   than generic synthetic benchmarks. The concrete, named workload is
   what makes architecture choices defensible against open-source
   baselines (vLLM, SGLang, llm-d).

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
- **Observability:** OpenTelemetry + Prometheus + structured logging
  with `structlog`. ClickHouse for request analytics.
- **Agents:** LangGraph for the agent loop (Milestone 6).
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

- Foundations are done (`learning-foundations.md` self-test passes 10/12+)
- M4 Max environment is set up (Python 3.11+, MLX-LM installed, can run
  Qwen 2.5 3B locally)
- Modal account created with $50+ credit
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
