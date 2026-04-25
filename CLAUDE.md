# production-llm-platform — Claude Code project context

This file is auto-loaded by Claude Code on every session in this repo.
It defines stable project context: what this is, the architectural
non-negotiables, the high-level tech stack, the conventions, and the
session protocol. Detailed tooling, anchor papers and projects, and
cross-cutting conventions live in
[docs/conventions.md](./docs/conventions.md) and evolve as the
project progresses.

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

## Tech stack overview

- **Language:** Python 3.11+ primarily. Add Go only if a specific
  component clearly benefits (e.g., high-throughput proxy in front of
  the inference layer).
- **Inference backends:** vLLM (cloud, CUDA) and MLX-LM (local, M4
  Max), behind a pluggable backend interface.
- **Primary dev machine:** MacBook Pro M4 Max with substantial unified
  RAM. Used for ~70% of all work — orchestration, RAG, agents, evals,
  dashboards, integration tests, debugging. MLX-LM serves smaller
  reasoning models locally for free.
- **Cloud GPU rentals:** Modal primarily, RunPod for sustained
  workloads. Used only for CUDA-dependent work — vLLM benchmarks,
  full-scale multi-LoRA, prefill/decode disaggregation, the flagship
  benchmark.
- **Total cloud GPU budget across all 12 milestones: under $400.**

For the **detailed per-milestone tooling** (retrieval, observability,
agents, structured outputs, evals, cost/routing, deployment, diagrams),
**anchor papers and projects (2026)**, **cross-cutting conventions**,
and **doc maintenance cadence**, see
[docs/conventions.md](./docs/conventions.md).

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
├── .claude/
│   ├── settings.json              # Project-shared permissions + hooks
│   └── agents/                    # Custom subagents (e.g., milestone-scaffolder)
├── docs/
│   ├── build-plan.md              # 12-milestone plan, GPU budgets, deliverables
│   └── conventions.md             # Detailed tooling, anchor papers, conventions
├── adr/                           # Architecture decision records (MX-*.md)
├── milestones/                    # M1-M12 build artifacts (PLAN.md, REPORT.md)
└── content/posts/                 # Incremental blog post drafts
```

---

## Pointers to deeper docs

- **Full 12-milestone plan, roadblocks, GPU budgets, deliverables
  structure:** [docs/build-plan.md](./docs/build-plan.md)
- **Detailed tooling, anchor papers and projects, cross-cutting
  conventions, doc maintenance cadence:**
  [docs/conventions.md](./docs/conventions.md)
- **4-week pre-build curriculum and self-test:**
  [learning-foundations.md](./learning-foundations.md)

---

## Claude Code session protocol (best practices for this repo)

This repo uses Claude Code (claude.ai/code) for the build. Apply
these patterns by default in every session:

1. **Specialized agents.** Use the `Agent` tool with appropriate
   `subagent_type` for research that spans multiple sources, parallel
   audits, codebase exploration, or work that would otherwise pollute
   the main context window. Custom subagents for this project live
   in `.claude/agents/` (currently: `milestone-scaffolder.md` for
   Phase A scaffolding work).

2. **Plan mode for complex implementations.** Use Plan mode (or the
   built-in Phase A protocol below) before diving into code on any
   multi-file change. Reach alignment on the design first; execute
   second. Don't write code in Phase A.

3. **Memory.** Write persistent memories at
   `~/.claude/projects/-Users-sri-projects-production-llm-platform/memory/`
   for user / feedback / project / reference patterns that should
   compound across sessions. Read existing memories when relevant
   rather than re-deriving from chat history. The `MEMORY.md` file in
   that directory is the index.

4. **TaskCreate / TaskUpdate.** Use task tracking for multi-step
   work, especially during M1+ build. Mark each task complete as
   soon as it's done; don't batch.

5. **Private operational docs.** Sensitive / strategic / personal-
   positioning content lives in `_private/` (gitignored). When
   context might benefit from one of these files, read it directly
   rather than re-deriving. The `reference_private_docs` memory has
   the index of all `_private/docs/` files.

6. **Hooks and permissions.** `.claude/settings.json` defines
   project-shared permissions (read-only commands allowlisted to
   reduce permission prompts) and hooks (post-Stop notification on
   long-running tasks). User-local overrides go in
   `.claude/settings.local.json` (gitignored).

   **MCP servers.** `.mcp.json` at the repo root configures
   project-shared MCP servers (currently empty by design — the
   default for this project is no MCP servers until a specific
   need surfaces). Templates for common servers (filesystem,
   GitHub, HuggingFace, Postgres, Sentry, Slack) are at
   `.claude/mcp/templates.md`. Secrets reference env vars via
   `${VAR}` syntax; actual values go in `.env` (gitignored, see
   `.env.example`).

7. **Web verification.** Before citing tools, papers, or libraries
   that aren't already in `docs/conventions.md`, verify with
   WebSearch / WebFetch. Don't rely solely on training data — the
   AI infra ecosystem moves fast; April 2026 SOTA differs from
   January 2026 SOTA.

8. **Authorize destructive ops explicitly.** Force pushes, history
   rewrites, file deletions, cross-repo operations all require
   explicit user "go ahead" before execution. Default to flagging
   risk and confirming.

9. **Parallel tool calls when independent.** Batch parallel tool
   calls in a single response when operations don't depend on each
   other. Sequence them when dependent. Surgical Edit beats full
   Write for existing files.

10. **Pull `_private/milestones/M1-plan-draft.md` at M1 kickoff.**
    The pre-staged Milestone 1 plan is the starting point for Phase
    A scaffolding when the user says "Let's start with Phase A —
    Milestone 1 scaffolding."

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
answer fluently before proceeding.

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
