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

Follow `.claude/agents/dev-agent.md` exactly — in short:

1. Check out the spec's feature branch and read `spec.md`, `plan.md`,
   `data-model.md` (if present), `contracts/` (if present), and `tasks.md`.
2. Run `/speckit.analyze` first — stop and comment on the issue if it
   surfaces a real gap rather than pushing through it.
3. Run `/speckit.implement` to build the feature and its tests.
4. Run the full test suite and linter locally before opening a PR.
5. Open a PR linking the issue (`Closes #<issue>`) and the spec directory.
6. Update the issue: remove `spec-approved`, add `in-review`.

If `.claude/commands/speckit.implement.md` is missing, stop and comment that
Spec Kit isn't installed rather than implementing freehand. If the test
suite or linter cannot be run in this environment (missing tooling), say so
explicitly in the PR description rather than claiming tests passed.

Slack notifications for this workflow are handled entirely outside this
agent — `.github/workflows/agent-notify.yml` watches for a PR labeled
`agent-generated` opening and posts to Slack deterministically. You don't
need to compose or trigger anything Slack-related yourself.

