# MCP server templates

Project-shared templates for adding MCP (Model Context Protocol)
servers to production-llm-platform. The active MCP config lives at
`.mcp.json` (repo root, committed) — this file is the reference for
how to add common servers when needed.

## How to add an MCP server

1. **Verify the server exists.** Search
   https://github.com/modelcontextprotocol/servers or the broader
   MCP ecosystem for the server you want. Don't invent server
   names.
2. **Identify required env vars.** Most servers need credentials —
   API tokens, database URLs, etc. Never put secrets in `.mcp.json`
   directly; use `${VAR}` env-var references and put the actual
   secrets in `.env` at the repo root (already gitignored).
3. **Add the server config to `.mcp.json`.** Copy the relevant
   template from below into the `mcpServers` object.
4. **Set the env vars.** Either in `.env` (loaded by Claude Code
   automatically if present), or in your shell profile.
5. **Restart Claude Code session.** MCP servers load at session
   start.
6. **Verify with a test query.** Ask Claude to use a tool from the
   new server (e.g., "list my recent GitHub issues" if the GitHub
   server is wired).

---

## Templates for common servers

Copy the relevant snippet into the `mcpServers` object in `.mcp.json`.
All templates use the official Anthropic / community MCP servers.

### Filesystem (read files outside this repo)

Useful for cross-repo work — e.g., reading `~/projects/sriteja28/`
(profile repo) or `~/projects/eks-three-tier-aws-infra/` without
cloning into this repo.

```json
"filesystem": {
  "command": "npx",
  "args": [
    "-y",
    "@modelcontextprotocol/server-filesystem",
    "/Users/sri/projects/sriteja28",
    "/Users/sri/projects/eks-three-tier-aws-infra"
  ]
}
```

No env vars needed. Each path argument is a directory the server is
allowed to read.

---

### GitHub (alternative to `gh` CLI)

Useful if you want Claude to query issues, PRs, and repo metadata
without invoking `gh` for each call. The `gh` CLI already handles
most needs in this project — only add this if you find yourself
making many parallel GitHub queries.

```json
"github": {
  "command": "npx",
  "args": ["-y", "@modelcontextprotocol/server-github"],
  "env": {
    "GITHUB_PERSONAL_ACCESS_TOKEN": "${GITHUB_TOKEN}"
  }
}
```

**Env var:** `GITHUB_TOKEN` (or whatever name you use in `.env`).
Use a fine-grained PAT with read scopes for relevant repos. Avoid
write scopes unless explicitly needed.

---

### Hugging Face (model artifacts and dataset access)

Useful starting Day 0 of foundations — Claude can pull model cards,
verify quantization variants exist, fetch dataset metadata directly
without you opening huggingface.co.

```json
"huggingface": {
  "command": "npx",
  "args": ["-y", "@huggingface/mcp-server"],
  "env": {
    "HF_TOKEN": "${HF_TOKEN}"
  }
}
```

**Env var:** `HF_TOKEN` (read-only). Verify the package name on
huggingface.co/blog or the MCP server registry — the canonical name
may differ from `@huggingface/mcp-server` depending on the current
release.

---

### Postgres (M5+ if pgvector is chosen as vector DB)

Useful from Milestone 5 onwards if you select pgvector for tenant
metadata + vector storage. Lets Claude query schemas, run analytical
queries against benchmark results, etc.

```json
"postgres": {
  "command": "npx",
  "args": [
    "-y",
    "@modelcontextprotocol/server-postgres",
    "${PGVECTOR_URL}"
  ]
}
```

**Env var:** `PGVECTOR_URL` (e.g., `postgresql://user:pass@host:5432/db`).

---

### Sentry (M8+ observability)

Useful from Milestone 8 if you wire Sentry for production error
tracking alongside Langfuse / OpenLLMetry.

```json
"sentry": {
  "command": "npx",
  "args": ["-y", "@sentry/mcp-server"],
  "env": {
    "SENTRY_AUTH_TOKEN": "${SENTRY_TOKEN}",
    "SENTRY_ORG": "your-org-slug"
  }
}
```

**Env vars:** `SENTRY_TOKEN`. Verify the package name in Sentry's
docs — naming may have shifted.

---

### Slack (read-only for community presence)

Useful for monitoring vLLM, SGLang, llm-d Discords / Slacks
programmatically — though these are usually Discord, and Discord MCP
support is less mature than Slack as of April 2026.

**Strong recommendation: read-only scope only.** Don't grant Claude
write access to Slack channels — risk of inadvertent posts.

```json
"slack": {
  "command": "npx",
  "args": ["-y", "@modelcontextprotocol/server-slack"],
  "env": {
    "SLACK_BOT_TOKEN": "${SLACK_READ_TOKEN}",
    "SLACK_TEAM_ID": "T01ABCDEFGH"
  }
}
```

**Env var:** `SLACK_READ_TOKEN` with `channels:history`,
`groups:history`, `im:history`, `mpim:history`, `users:read` scopes
only — no write scopes.

---

### Anthropic Files / Workbench

If/when Anthropic releases an MCP server for the Files API or
Workbench, useful for managing prompt artifacts and prompt-cache
state across the project.

Status as of April 2026: verify availability on
https://docs.claude.com/en/docs/agents-and-tools/mcp before
configuring. Skip if not yet released.

---

## Env var management

Secrets handling pattern for this project:

1. **`.env` at repo root** — already gitignored
   (`.env`, `.env.local`, `.env.*.local` all in `.gitignore`).
   Holds all secret values.

2. **`.env.example` at repo root** — committed, shows variable
   names without values:
   ```
   GITHUB_TOKEN=
   HF_TOKEN=
   PGVECTOR_URL=
   SENTRY_TOKEN=
   SLACK_READ_TOKEN=
   ```
   Lets future contributors / future-you know what env vars are
   expected without leaking actual values.

3. **`.mcp.json`** — committed; references env vars via `${VAR}`,
   never inline values.

4. **Shell profile / direnv** — alternative to `.env` if preferred.
   `direnv` with a `.envrc` file (gitignored) is the cleanest
   pattern for project-scoped env vars on macOS.

---

## When NOT to add an MCP server

- If `gh` / `git` / `python` / shell commands already handle the
  use case. Permissions allowlist in `.claude/settings.json`
  reduces friction enough that adding an MCP server for the same
  capability is over-engineering.
- If the server requires write scopes you don't fully understand
  (e.g., Slack write — automation that posts on your behalf can
  embarrass you).
- If the server is alpha / unreleased — wait for stable release.

The default for this project is **no MCP servers** until a specific
need surfaces. The empty `mcpServers: {}` in `.mcp.json` is correct
for now.

---

## After adding a server

Document it in `.claude/commands/README.md` under a new "Active MCP
servers" section, with one line per server explaining what it
provides and which milestones use it.
