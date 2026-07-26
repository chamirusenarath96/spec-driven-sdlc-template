# Setup checklist for a new project from this template

Run through this once, in order, after clicking **Use this template**.

## 1. Clone and rename

```bash
gh repo clone <you>/<new-repo-name>
cd <new-repo-name>
```

## 2. Pick your stack and fill in the TODOs

- `.github/workflows/ci.yml` and `deploy.yml` have `TODO` steps — replace
  with your real setup/install/lint/test/deploy commands.
- Add a short paragraph to `docs/ARCHITECTURE.md`'s "What this template does
  not decide for you" section describing your stack.

## 3. Install and compile the agentic workflows

```bash
gh extension install github/gh-aw
gh aw compile
```

This turns `.github/workflows/agent-*.md` into runnable
`agent-*.lock.yml` files (gitignored — recompile after editing any
`agent-*.md`, don't hand-edit the lock files).

## 4. Set repo secrets

```bash
gh secret set ANTHROPIC_API_KEY          # or CLAUDE_CODE_OAUTH_TOKEN
gh secret set SLACK_WEBHOOK_URL          # https://api.slack.com/messaging/webhooks
```

## 5. Create the GitHub Project

Follow [PROJECT_SETUP.md](PROJECT_SETUP.md) to create the board, the
`Status` field, and note the field/option IDs the agent workflows need.

## 6. Set branch protection (recommended)

Require the `test` job from `ci.yml` to pass before merge, and decide
whether the Review Agent is allowed to merge directly (see
[AGENTS.md](AGENTS.md#human-checkpoints)) or should only approve.

## 7. Smoke-test the pipeline

1. Open an issue using the **Feature request** template (auto-labeled
   `needs-spec`).
2. Manually dispatch `agent-research.md` (Actions tab → Research Agent →
   Run workflow) rather than waiting for the schedule, pointing it at that
   issue number.
3. Confirm it opens a `specs/001-.../` PR and comments on the issue.
4. Label the issue `spec-approved`, dispatch `agent-dev.md`, confirm it
   opens an implementation PR.
5. Confirm `agent-review.md` fires automatically on that PR (it's PR-event
   triggered, no manual dispatch needed) and leaves a review.

If any step fails, check the workflow's Actions log first — gh-aw runs are
read-only by default, so a missing safe-output declaration is the most
common cause of "the agent said it did X but nothing changed."

## Not included, by design

- No Slack channel is created for you — create one and point the webhook at
  it.
- No live GitHub Project is created for you (Projects have no
  template-repository equivalent) — step 5 is manual, once, per project.
- No Copier CLI wiring for "ask questions, generate a stack-specific
  variant" — see the suggested build order in the original research doc
  (`docs/ARCHITECTURE.md`'s design notes): extract this template into a
  Copier template only after the loop has proven itself on one real project.
