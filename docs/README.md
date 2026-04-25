# Docs

Project documentation that's not milestone-specific or ADR-specific.

## Expected contents (created during the build)

- `scenario.md` — the Spider scenario (multi-tenant Glean-style
  knowledge agent for internal teams) that anchors the entire project,
  with workload model, tenant tiers, source mix, and ACL details.
  (Also currently captured inline in `build-plan.md` under "Anchor
  scenario: Spider".)
- `architecture.md` — overall system architecture diagram and component
  responsibilities
- `local-development.md` — M4 Max setup guide for MLX-LM
- `cloud-development.md` — Modal/RunPod setup for vLLM benchmarks
- `benchmarking-methodology.md` — how we measure, what we measure, what
  we deliberately don't measure
- `cost-tracking.md` — running tally of cloud GPU spend per milestone
