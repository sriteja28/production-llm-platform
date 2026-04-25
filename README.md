# production-llm-platform

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

A multi-tenant LLM serving platform built for reasoning-and-agent
workloads — scheduling, caching, and cost control when output length
is unpredictable (reasoning traces from 10 to 30,000+ tokens) and KV
state persists across hundreds of agent turns. Spans inference, RAG,
agents, evals, observability, and cost economics across 12 milestones,
with deliberate production failures debugged publicly.

## Status

Active build. Scaffolding committed; Milestone 1 (reasoning-aware
baseline + dual-backend abstraction + measurement) is next. The full
milestone plan, GPU budgets, and deliverables structure live in
[docs/build-plan.md](./docs/build-plan.md). Milestone reports land in
`milestones/MX/REPORT.md` as each milestone completes.

## Anchor scenario

A multi-tenant SaaS where lawyers run multi-step reasoning agents over
firm-specific document libraries: variable-length reasoning traces, KV
state persisting across hundreds of agent turns, per-firm LoRA
adapters, SOC 2 compliance pressure, real cost economics. Every
milestone builds features for this workload rather than generic
synthetic benchmarks — the concrete, named workload is what makes the
architecture choices defensible against open-source baselines like
vLLM, SGLang, and llm-d.

## Architecture

The inference layer is backend-agnostic: vLLM (CUDA, cloud) and MLX-LM
(Apple Silicon, local) sit behind a pluggable interface so the
scheduler, RAG retrieval, agent loop, eval harness, and observability
work against either backend. See [CLAUDE.md](./CLAUDE.md) for the full
architecture principles.

_(Diagrams and benchmark output land here with Milestone 1.)_

## Run

_(Quickstart commands land here with Milestone 1, once the dual-backend
harness is functional.)_

## Repo layout

```
production-llm-platform/
├── docs/
│   └── build-plan.md            # 12-milestone plan, GPU budgets, deliverables
├── milestones/                  # M1-M12 build artifacts (PLAN.md, REPORT.md)
├── adr/                         # Architecture decision records
├── content/posts/               # Incremental blog posts
├── CLAUDE.md                    # Claude Code session context
└── learning-foundations.md      # Pre-build curriculum and self-test
```

## License

MIT — see [LICENSE](./LICENSE).
