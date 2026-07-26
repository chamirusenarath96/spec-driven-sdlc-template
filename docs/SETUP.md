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

## 4. Install Spec Kit and establish the constitution

The Research and Dev Agent personas (`.claude/agents/research-agent.md`,
`dev-agent.md`) drive the real [Spec Kit](https://github.com/github/spec-kit)
slash commands (`/speckit.specify`, `/speckit.plan`, `/speckit.implement`,
etc.) — this step installs them.

```bash
# Requires uv: https://docs.astral.sh/uv/
uv tool install specify-cli --from git+https://github.com/github/spec-kit.git

specify init --here --ai claude
```

`specify init --here` writes one command file per step into
`.claude/commands/speckit.*.md` and scaffolds `.specify/memory/` — **these
command files get committed to the repo**, so the GitHub Actions runner that
later executes `agent-research.md`/`agent-dev.md` only needs the Claude Code
CLI, not a live `specify` install; it reads the committed command files the
same way an interactive session would. (`--ai claude` matches this
template's agents; check `specify init --help` if the flag name has changed
since this was written — Spec Kit's CLI surface evolves between releases.)

Then, once, establish the project's non-negotiable principles — every spec
and plan the agents produce is checked against this:

```
/speckit.constitution Create principles focused on code quality, testing
standards, and whatever else this project must never compromise on.
```

This supersedes the placeholder templates this repo template shipped under
`.specify/templates/` — Spec Kit's own core templates take over from there.
See [`.specify/README.md`](../.specify/README.md).

## 5. Set repo secrets

```bash
gh secret set ANTHROPIC_API_KEY          # or CLAUDE_CODE_OAUTH_TOKEN
gh secret set SLACK_WEBHOOK_URL          # https://api.slack.com/messaging/webhooks
```

## 6. Create the GitHub Project

Follow [PROJECT_SETUP.md](PROJECT_SETUP.md) to create the board, the
`Status` field, and note the field/option IDs the agent workflows need.

## 7. Set branch protection (recommended)

Require the `test` job from `ci.yml` to pass before merge, and decide
whether the Review Agent is allowed to merge directly (see
[AGENTS.md](AGENTS.md#human-checkpoints)) or should only approve.

## 8. Smoke-test the pipeline

1. Open an issue using the **Feature request** template (auto-labeled
   `needs-spec`) — this is how a feature/ticket gets raised in the first
   place; see [AGENTS.md](AGENTS.md#raising-issues) for the second,
   spec-kit-native way to raise issues once a spec exists.
2. Manually dispatch `agent-research.md` (Actions tab → Research Agent →
   Run workflow) rather than waiting for the schedule, pointing it at that
   issue number.
3. Confirm it ran `/speckit.specify` → `/speckit.clarify` → `/speckit.plan`
   → `/speckit.tasks`, opened a `specs/001-.../` PR, and commented on the
   issue.
4. Label the issue `spec-approved`, dispatch `agent-dev.md`, confirm it ran
   `/speckit.analyze` → `/speckit.implement` and opened an implementation PR.
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
