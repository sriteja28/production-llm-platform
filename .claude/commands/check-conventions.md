---
description: Verify a milestone's deliverables follow the cross-cutting conventions in docs/conventions.md
argument-hint: "<milestone, e.g., M1>"
---

Verify $ARGUMENTS deliverables against the cross-cutting conventions
in `docs/conventions.md`.

Read in order:
1. `docs/conventions.md` "Cross-cutting conventions" section
2. `milestones/$ARGUMENTS/PLAN.md`
3. `milestones/$ARGUMENTS/REPORT.md` (if exists)
4. All `adr/$ARGUMENTS-*.md` files
5. `content/posts/milestone-$ARGUMENTS.md` (if exists)

For each cross-cutting convention, report PASS / FAIL / PARTIAL / N/A
with one-line evidence:

1. **Cost-per-correct-answer** — Does REPORT.md (or PLAN.md if Phase
   A only) include a cost-per-correct-answer section or table? PASS
   if numbers present; PARTIAL if only methodology described; FAIL
   if missing entirely; N/A only if this milestone has no quality
   claims at all.

2. **Hamel Husain eval doctrine** — For any quality claims:
   - Binary pass/fail rubric (not Likert)? PASS/FAIL.
   - Judge-as-classifier validation (precision/recall on a labeled
     set of 100+ examples)? PASS/FAIL.
   - Paired bootstrap or McNemar's test for statistical
     significance? PASS/FAIL.

3. **OpenTelemetry GenAI semconv** — Does the milestone's code
   instrument with OpenLLMetry against GenAI semconv?
   - `opentelemetry-instrumentation` imports present? PASS/FAIL.
   - Span attributes use `gen_ai.*` namespace? PASS/FAIL.
   - `OTEL_SEMCONV_STABILITY_OPT_IN` documented for dual emission?
     PASS/PARTIAL/FAIL.

4. **OWASP coverage** — If this milestone touches retrieval (M5),
   guardrails (M7), or observability (M8): is the threat model
   mapped in `docs/owasp-coverage.md`? PASS/FAIL/N/A.

5. **vLLM disagg prefill caveat** — If this milestone references
   disagg prefill (typically M4, M11, M12): is it framed correctly
   as "improves tail ITL but not throughput" rather than as a
   throughput win? PASS/FAIL/N/A.

Output format:

```
## Convention check for $ARGUMENTS

| Convention | Status | Evidence |
|---|---|---|
| Cost-per-correct-answer | PASS/FAIL/PARTIAL/N/A | <one-line evidence> |
| Hamel eval doctrine — binary rubric | ... | ... |
| Hamel eval doctrine — classifier validation | ... | ... |
| Hamel eval doctrine — stat sig | ... | ... |
| OTel GenAI semconv — instrumentation | ... | ... |
| OTel GenAI semconv — namespace | ... | ... |
| OWASP coverage | ... | ... |
| vLLM disagg prefill caveat | ... | ... |

## Recommended fixes
- <bullet for each FAIL or PARTIAL>

## Ready to declare $ARGUMENTS complete?
- YES if all PASS or N/A
- NO if any FAIL — fix first
- DISCRETIONARY if PARTIAL — user decides
```

Don't fix anything automatically — surface gaps and let the user
decide.
