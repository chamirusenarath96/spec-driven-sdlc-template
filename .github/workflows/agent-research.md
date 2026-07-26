---
# GitHub Agentic Workflow — Research/Spec Agent
# Compile with `gh aw compile` to produce agent-research.lock.yml before this
# runs. See docs/WORKFLOWS.md for install instructions.
on:
  issues:
    types: [labeled]
  schedule:
    # Staggered from agent-dev/agent-review to avoid stacking all three
    # agents' token usage into the same 5-hour rate-limit window.
    - cron: "0 13 * * 1-5"
  workflow_dispatch:
    inputs:
      issue_number:
        description: "Issue number to draft a spec for"
        required: true

if: github.event.label.name == 'needs-spec' || github.event_name != 'issues'

permissions:
  contents: read
  issues: read

engine: claude

safe-outputs:
  create-pull-request:
    title-prefix: "spec: "
    labels: [spec]
  add-comment: {}
  update-issue:
    target: "triggering"

timeout_minutes: 20
---

# Research Agent

You are running as the **research-agent** persona defined in
`.claude/agents/research-agent.md` — follow it exactly.

## Trigger context

- If triggered by an issue being labeled `needs-spec`, use
  `${{ github.event.issue.number }}` as the target issue.
- If triggered on a schedule, list open issues labeled `needs-spec` that do
  not yet have a linked `specs/NNN-*` PR, and process **at most 2** of them
  (oldest first) to keep token spend bounded.
- If triggered manually, use the `issue_number` input.

## Task

For each target issue:

1. Read the issue and all comments.
2. Determine the next spec number by scanning `specs/` for the highest
   existing `NNN-` prefix.
3. Produce the full spec directory as described in
   `.claude/agents/research-agent.md`, using `.specify/templates/` as the
   structural basis for each file.
4. Open a pull request containing only the new `specs/NNN-slug/` directory
   (this is a declared safe output — do not push directly to `main`).
5. Comment on the originating issue with a 3-5 bullet summary and a link to
   the spec PR.
6. Update the issue: remove `needs-spec`, add `spec-in-review`.

Do not write or modify any file outside `specs/NNN-slug/`. Do not open more
than one PR per issue.
