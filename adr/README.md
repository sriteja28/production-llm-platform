# Architecture Decision Records (ADRs)

This folder contains architecture decision records for production-llm-
platform. Every meaningful design decision gets an ADR.

## Format

Each ADR follows this structure:

```markdown
# ADR-NNNN: Title

## Status
Proposed | Accepted | Superseded by ADR-NNNN

## Context
What problem are we trying to solve? What forces are at play?

## Decision
What did we decide to do?

## Alternatives Considered
What other options did we evaluate and why did we reject them?

## Consequences
What are the tradeoffs we're accepting?

## References
Papers, GitHub repos, blog posts informing this decision.
```

## Naming convention

`MX-NNN-short-description.md` where:
- MX is the milestone number (M1, M2, etc.)
- NNN is sequential within that milestone (001, 002, etc.)
- short-description is kebab-case

Example: `M1-001-pluggable-backend-interface.md`
