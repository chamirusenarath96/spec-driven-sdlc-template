# Architecture

This repo runs a **spec-driven, agent-run SDLC**: a GitHub issue becomes a
spec, an approved spec becomes code, and every PR is reviewed against the
spec that produced it — all coordinated through GitHub-native primitives, no
separate PM tool or agent orchestrator.

## Why these choices

| Decision | Rationale |
|---|---|
| GitHub Projects (v2) + Issues, not Linear/Plane/Huly | Free, no seat/issue caps, GraphQL API agents can drive directly, and issues are already the spec source of truth — no second system to keep in sync. |
| GitHub Spec Kit structure (`specs/NNN-slug/{spec,research,plan,tasks}.md`) | Battle-tested spec schema any human or agent (Copilot, Cursor, Claude) can read without custom tooling. |
| GitHub Agentic Workflows (`gh-aw`) for triggering agents | Native `schedule:`/event triggers, compiles Markdown → Actions YAML, and enforces a **safe-outputs** boundary (writes only through declared actions like `create-pull-request`) so autonomous runs can't freelance. |
| Claude Code subagents (`.claude/agents/*.md`) for the personas | Versioned, reusable, isolated-context agent definitions callable from both `gh-aw` and `claude-code-action`. |
| Plain GitHub Actions + Slack webhook for CI/deploy | Standard, zero new dependency, one webhook per repo. |

## The three agents

See [AGENTS.md](AGENTS.md) for the full state machine. Summary:

1. **Research Agent** (`.claude/agents/research-agent.md`) — issue → spec.
2. **Dev Agent** (`.claude/agents/dev-agent.md`) — approved spec → PR.
3. **Review Agent** (`.claude/agents/review-agent.md`) — PR → approve/merge or
   request changes.

Each agent is invoked by a matching `.github/workflows/agent-*.md` gh-aw
workflow (event-triggered where possible, scheduled as a fallback sweep — see
[WORKFLOWS.md](WORKFLOWS.md)).

## What this template does not decide for you

This is a stack-agnostic template — it has no opinion on your language,
framework, or deploy target. Fill in:

- The `TODO` steps in `.github/workflows/ci.yml` / `deploy.yml`.
- Your actual application code (there is none scaffolded here).
- This section: once you pick a stack, replace this paragraph with what it
  is and why, so agents reading this file have that context too.

## Known limitations (as of this template's creation, 2026-07)

- `gh-aw` was in public preview as of June 2026 — expect its exact
  frontmatter schema to keep evolving; recompile with `gh aw compile` after
  upgrading.
- Scheduled (non-`@claude`-mention) Claude Code Action runs have had
  auth/permission issues in some setups — keep `workflow_dispatch:` as a
  manual fallback on every agent workflow (already done in this template).
- Claude Code Actions usage draws from your normal subscription/API rate
  limits (5-hour rolling window) — the cron schedules in this template are
  staggered deliberately; don't collapse them onto the same time slot.
