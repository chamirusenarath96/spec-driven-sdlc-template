---
# GitHub Agentic Workflow — Review Agent
# Compile with `gh aw compile` to produce agent-review.lock.yml before this
# runs.
on:
  pull_request:
    types: [opened, synchronize, reopened]
  workflow_dispatch:
    inputs:
      pr_number:
        description: "PR number to review"
        required: true

if: |
  contains(github.event.pull_request.labels.*.name, 'agent-generated') ||
  github.event_name == 'workflow_dispatch'

permissions:
  contents: read
  pull-requests: read
  issues: read

engine: claude

safe-outputs:
  add-comment: {}
  # Approvals/merges and moving Project items are handled by a custom
  # safe-output job — see docs/WORKFLOWS.md for why plain `gh pr review
  # --approve` isn't declared as a built-in safe output here.
  custom:
    - id: pr-review-decision
      description: >
        Submit a PR review (approve or request changes) and update the
        linked GitHub Project item's status field accordingly.
    - id: slack-notify
      description: >
        Post a milestone summary to the project's Slack channel via
        .github/actions/slack-notify. Purely informational, no code/issue
        write.

timeout_minutes: 20
---

# Review Agent

You are running as the **review-agent** persona defined in
`.claude/agents/review-agent.md` — follow it exactly.

## Trigger context

Target PR is `${{ github.event.pull_request.number }}` (or the
`pr_number` input on manual dispatch).

## Task

1. Find the linked spec directory from the PR description or linked issue.
2. Read `spec.md`'s acceptance criteria and `tasks.md`.
3. Review the full diff (`gh pr diff <number>`) against each acceptance
   criterion, test coverage, security, and scope per
   `.claude/agents/review-agent.md`.
4. Emit a `pr-review-decision` safe output:
   - `approve` with a summary of which criteria passed, if all criteria are
     met with test coverage and no security/correctness issues.
   - `request_changes` with specific file:line comments and a list of unmet
     criteria, otherwise.
5. On approve: move the linked Project item to **Done**.
6. On request changes: move the linked Project item to **In Development** and
   add label `changes-requested`.

Never approve a PR that removes or weakens tests without an explicit,
justified reason stated in the PR description.

## Slack notification

After the `pr-review-decision` safe output, also emit `slack-notify` with:

- `status: success` on approve, `warning` on request-changes
- `title`: `"PR approved"` or `"Changes requested"` + the PR title
- `summary`: link to the PR
- `fields`: acceptance criteria met/total, and — on request-changes — a
  short list of which criteria are unmet

