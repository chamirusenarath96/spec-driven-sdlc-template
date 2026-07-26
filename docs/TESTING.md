# Testing

This template ships no application code, so there is no test suite yet —
this file is a placeholder the Dev Agent and human contributors are expected
to keep accurate once real code exists.

## What to document here once you add a stack

- Where tests live (e.g. `tests/`, colocated `*.test.ts`).
- How to run them locally (exact command).
- What levels exist (unit/integration/e2e) and when each is required — the
  Dev Agent matches whatever a spec's "Testing needed" section asks for
  against this.
- Coverage expectations, if any, and how they're enforced in `ci.yml`.
- How to run migrations/fixtures needed for integration tests, if
  applicable.

## Why this file exists separately from ARCHITECTURE.md

The Dev Agent (`.claude/agents/dev-agent.md`) is instructed to "run the full
test suite and linter locally before opening a PR" — it needs one canonical,
short place to learn how, rather than re-deriving it from `ci.yml` or
guessing. Keep this file in sync with `ci.yml`'s actual test step.
