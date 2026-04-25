---
description: Write a pre-flight runbook before starting a cloud GPU session (Modal/RunPod)
argument-hint: "<context, e.g., M1-baseline-bench, M3-kv-cache-pressure>"
---

Generate a cloud GPU session runbook for: $ARGUMENTS

Per CLAUDE.md architecture principle 8: every cloud GPU session has
a written experiment plan with specific commands, expected duration,
and expected cost **before the meter starts**.

Produce a runbook with these sections:

## 1. Goal
One sentence: what data or outcome are we capturing?

## 2. Provider and instance
- Provider (Modal preferred, RunPod fallback)
- GPU type and count (e.g., 1× A100 80GB)
- Image / base container (e.g., `vllm/vllm-openai:latest` pinned to
  specific version)
- Estimated $ rate per hour at the chosen provider

## 3. Pre-flight checks
- Local code commit hash to deploy
- Model artifacts to download (HF model ID, expected size in GB)
- Workload data to upload (synthetic Spider mix, eval set, etc.)
- Environment variables / secrets needed (HF_TOKEN, etc.)

## 4. Commands to run, in order
- **Setup** (~30 min budget): provision, install vLLM (specific
  version), download model, smoke test with one inference call
- **Workload** (estimated duration): exact CLI commands for the
  benchmark / experiment, with all flags
- **Capture** (~30 min budget): collect logs, metrics CSVs, vLLM
  scheduler outputs, GPU profiles

## 5. Data to capture
- Specific file paths for logs, metrics, screenshots
- Where they get rsync'd into the local repo after the session
- What attaches to `milestones/MX/REPORT.md`

## 6. Estimated total cost and duration
- Total $ = rate × duration
- Verify against the milestone's GPU budget in `docs/build-plan.md`
- Flag explicitly if over budget

## 7. Abort criteria
- Specific conditions to kill the session early (e.g., "model load
  fails after 3 retries"; "benchmark throughput <10% expected after
  30 min"; "OOM on first batch")

## 8. Post-session checklist
- Tear down (`modal app stop`, RunPod terminate)
- Verify dashboard cost matches estimate
- Commit captured data to repo
- Note anomalies in `milestones/MX/REPORT.md`

This command produces the runbook only — do not start the cloud
session from this command. The user reviews the runbook, edits if
needed, then starts the session manually.
