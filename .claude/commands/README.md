# Project-shared slash commands

Custom slash commands for production-llm-platform. Invoke from any
Claude Code session in this repo as `/<command-name>`.

## Available commands

- **`/scaffold-milestone <N>`** — Phase A scaffolding for milestone N
  via the `milestone-scaffolder` subagent at
  `.claude/agents/milestone-scaffolder.md`. Use at the start of
  every milestone (`/scaffold-milestone 1` after foundations
  complete on 2026-05-24).

- **`/cloud-runbook <context>`** — generate a pre-flight runbook
  before any cloud GPU session. Required by CLAUDE.md architecture
  principle 8 ("write the runbook before starting the meter").
  Example: `/cloud-runbook M1-baseline-bench`.

- **`/check-conventions <M>`** — verify milestone M's deliverables
  follow the cross-cutting conventions in `docs/conventions.md`
  (cost-per-correct-answer, Hamel eval doctrine, OTel GenAI
  semconv, OWASP coverage, vLLM disagg prefill caveat). Use at
  end-of-milestone before declaring it complete.

## Adding a new command

1. Create `.claude/commands/<name>.md` with frontmatter:
   ```yaml
   ---
   description: Short description for the command picker
   argument-hint: "<argument-hint>"
   ---
   ```
   followed by the prompt body. Use `$ARGUMENTS` to inject the
   user's args.
2. Commit. Available immediately in any Claude Code session in this
   repo.
3. Document above in this README.

## Built-in skills also available (no project setup needed)

- `/init` — initialize CLAUDE.md from codebase
- `/review` — review the current branch / PR
- `/security-review` — security review of pending changes
- `/schedule` — schedule a remote agent (used for the 2026-05-23
  M1 progress check-in)
- `/web-setup` — sync GitHub credentials for remote agents
- `/loop` — run a prompt on a recurring interval
- `/simplify` — review changed code for reuse / quality / efficiency
- `/fewer-permission-prompts` — scan transcripts and add common Bash
  / MCP commands to the permissions allowlist (already partially
  done in `.claude/settings.json`)
