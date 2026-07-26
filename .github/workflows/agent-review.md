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

Slack notifications for this workflow are handled entirely outside this
agent — `.github/workflows/agent-notify.yml` watches for a review being
submitted on an `agent-generated` PR (via the real GitHub
`pull_request_review` event, regardless of who/what submitted it) and posts
to Slack deterministically, reading the review's actual `state` (`approved`
vs `changes_requested`) rather than trusting a self-report. You don't need
to compose or trigger anything Slack-related yourself.

