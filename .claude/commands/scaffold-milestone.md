---
description: Phase A scaffolding for milestone N (invokes the milestone-scaffolder subagent)
argument-hint: "<milestone number, e.g., 1>"
---

Phase A scaffolding for Milestone $ARGUMENTS of production-llm-platform.

Use the `Agent` tool with `subagent_type: "milestone-scaffolder"` and
this prompt:

> Phase A scaffolding for Milestone $ARGUMENTS of production-llm-platform.
>
> Read in order:
> - `CLAUDE.md` for stable project context
> - `docs/conventions.md` for tooling and cross-cutting conventions
> - `docs/build-plan.md` Milestone $ARGUMENTS section (scope, roadblock,
>   GPU budget)
> - For Milestone 1: `_private/milestones/M1-plan-draft.md` as the
>   pre-staged starting point
> - For Milestones 2-12: the previous milestone's `REPORT.md` for
>   handoff context
> - The memory at
>   `~/.claude/projects/-Users-sri-projects-production-llm-platform/memory/`
>   for user preferences and project state
>
> Generate:
> - `milestones/M$ARGUMENTS/PLAN.md` (scope, learnings, roadblock,
>   local-vs-cloud workflow, GPU cost, deliverables checklist,
>   self-test concept questions, what this milestone is NOT)
> - `milestones/M$ARGUMENTS/REPORT.md` stub (header only)
> - `adr/M$ARGUMENTS-*.md` ADR stubs for each architectural decision
>   identified during scaffolding
> - `content/posts/milestone-M$ARGUMENTS.md` outline
>
> Apply cross-cutting conventions from `docs/conventions.md`:
> cost-per-correct-answer in success criteria, Hamel eval doctrine
> if quality claims, OpenTelemetry GenAI semconv, vLLM disagg
> prefill caveat if relevant, OWASP mapping if guardrails.
>
> Generate 4-6 Phase A clarifying questions covering: model choice,
> framework specifics, scenario specifics (anchor on Spider unless
> overridden), scope (in M$ARGUMENTS vs deferred), GPU budget, any
> milestone-specific tradeoffs.
>
> Stop after scaffolding. Do not write implementation code, do not
> run benchmarks, do not commit. Output the scaffolded file list,
> the clarifying questions, and a preview of what Phase B will
> deliver.

Present the subagent's output to the user. Wait for explicit user
approval ("approved" or amendments) before any Phase B work begins.
Do not commit until approved.
