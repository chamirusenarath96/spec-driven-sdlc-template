---
# GitHub Agentic Workflow — Dev Agent
# Compile with `gh aw compile` to produce agent-dev.lock.yml before this runs.
on:
  issues:
    types: [labeled]
  schedule:
    - cron: "0 15 * * 1-5"
  workflow_dispatch:
    inputs:
      spec_dir:
        description: "specs/NNN-slug directory to implement"
        required: true

if: github.event.label.name == 'spec-approved' || github.event_name != 'issues'

permissions:
  contents: read
  issues: read
  pull-requests: read

engine: claude

safe-outputs:
  create-pull-request:
    title-prefix: "feat: "
    labels: [agent-generated]
  add-comment: {}
  update-issue:
    target: "triggering"

timeout_minutes: 45
---

# Dev Agent

You are running as the **dev-agent** persona defined in
`.claude/agents/dev-agent.md` — follow it exactly.

## Trigger context

- If triggered by label `spec-approved`, find the spec directory linked from
  the issue (the PR that introduced `specs/NNN-slug/` references this issue
  number).
- If triggered on a schedule, list issues labeled `spec-approved` that don't
  yet have an open implementation PR, and process **at most 1** per run —
  implementation is the most token-expensive step in this pipeline.
- If triggered manually, use the `spec_dir` input directly.

## Task

1. Read `specs/NNN-slug/spec.md`, `plan.md`, `data-model.md` (if present),
   `contracts/` (if present), and `tasks.md`.
2. Implement per `.claude/agents/dev-agent.md`'s process: work `tasks.md` in
   order, write tests for every acceptance criterion, run the full test
   suite and linter before opening a PR.
3. Open a PR linking the issue (`Closes #<issue>`) and the spec directory.
4. Update the issue: remove `spec-approved`, add `in-review`.

If the test suite or linter cannot be run in this environment (missing
tooling), say so explicitly in the PR description rather than claiming tests
passed.
