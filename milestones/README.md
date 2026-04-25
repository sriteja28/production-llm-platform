# Milestones

This folder will contain one subfolder per milestone (M1 through M12),
each with:

- `PLAN.md` — what we're building, expected learnings, expected
  production roadblocks, local-vs-cloud workflow plan, estimated GPU
  cost
- `REPORT.md` — what we built, what data showed, what broke and how we
  debugged it, lessons learned, "for hiring managers" sections per role
  track this milestone signals to

Each milestone is built incrementally with Claude Code following the
teach → build → break → debug → understand pattern defined in
`/CLAUDE.md`.

## Milestone index (planned)

### Phase 1 — Inference foundation
- M1: Naive baseline + dual-backend abstraction + measurement
- M2: Priority-aware multi-tenant scheduler
- M3: Per-tenant KV cache isolation + accounting
- M4: Prefill/decode disaggregation + multi-LoRA serving

### Phase 2 — Application layer
- M5: Production RAG pipeline with hybrid retrieval
- M6: Agent orchestration with tool execution sandbox
- M7: Structured outputs + safety + guardrails

### Phase 3 — Reliability and operations
- M8: Production-grade observability for non-deterministic systems
- M9: Eval infrastructure + regression detection
- M10: Cost economics + smart routing + budget governance

### Phase 4 — Production hardening + flagship
- M11: Multi-region, deployment safety, chaos engineering
- M12: The flagship benchmark + writeup + launch
