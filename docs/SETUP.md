# Setup checklist for a new project from this template

Run through this once, in order, after clicking **Use this template**. Each
step says which tool it uses, the exact command, and what you should see
when it worked — if you don't see that, stop and fix it before moving on;
later steps assume earlier ones actually succeeded.

## Tools used, at a glance

| Tool | What it's for | You need an account/install? |
|---|---|---|
| [`gh`](https://cli.github.com/) (GitHub CLI) | Repo/secret/project setup, smoke-testing | Install + `gh auth login` |
| [`gh-aw`](https://github.github.com/gh-aw/) (`gh extension`) | Compiles the agent workflows into runnable Actions | `gh extension install github/gh-aw` |
| [`specify-cli`](https://github.com/github/spec-kit) (Spec Kit) | Installs the `/speckit.*` commands the agents drive | `uv tool install specify-cli` (needs [`uv`](https://docs.astral.sh/uv/)) |
| Claude Code (via `claude-code-action` under the hood) | Runs the three agent personas | An `ANTHROPIC_API_KEY` or `CLAUDE_CODE_OAUTH_TOKEN` secret — no local install needed, it runs in Actions |
| Slack incoming webhook | Where CI/deploy/perf/digest/agent notifications land | A Slack workspace you can add an app to |
| GitHub Projects (v2) | The pipeline's state machine / board | Nothing to install — a few clicks on github.com |

Nothing above is optional if you want the full pipeline working — but you
can stop after step 3 and just have a normal CI-only repo with none of the
agent automation, if that's all you need right now.

Every secret/key any of this needs is listed in one place in
[`.env.example`](../.env.example) — check there if you're ever unsure what
a workflow expects.

## 1. Clone and rename

```bash
gh repo clone <you>/<new-repo-name>
cd <new-repo-name>
```

**Expect:** a local clone with this template's full file tree.

## 2. Pick your stack and fill in the TODOs

- `.github/workflows/ci.yml`, `deploy.yml`, and `perf.yml` each have `TODO`
  steps — replace with your real setup/install/lint/test/deploy/benchmark
  commands. `ci.yml` and `perf.yml` also expect a handful of `$GITHUB_OUTPUT`
  values (test pass/fail counts, coverage, p50/p95 latency, etc.) — the
  comments in each file show the shape and a worked example for common
  tools.
- Add a short paragraph to `docs/ARCHITECTURE.md`'s "What this template does
  not decide for you" section describing your stack.
- Update `docs/TESTING.md` with where your tests actually live and how to
  run them.

**Expect:** no functional change yet — this just replaces placeholders with
real commands so the workflows in step 8 have something to report on.

## 3. Install and compile the agentic workflows

```bash
gh extension install github/gh-aw
gh aw compile
```

**Expect:** `.github/workflows/agent-*.lock.yml` files appear (gitignored —
recompile with `gh aw compile` after editing any `agent-*.md`, never
hand-edit the lock files). This is the actual Actions workflow GitHub runs;
the `.md` files are only the source.

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

**Expect:** new files under `.claude/commands/speckit.*.md` and
`.specify/memory/`. Commit them — the GitHub Actions runner that later
executes `agent-research.md`/`agent-dev.md` only needs the Claude Code CLI,
not a live `specify` install; it reads these committed command files the
same way an interactive session would. (`--ai claude` matches this
template's agents; check `specify init --help` if the flag name has changed
since this was written — Spec Kit's CLI surface evolves between releases.)

Then, once, establish the project's non-negotiable principles — every spec
and plan the agents produce is checked against this:

```
/speckit.constitution Create principles focused on code quality, testing
standards, and whatever else this project must never compromise on.
```

**Expect:** `.specify/memory/constitution.md` populated with your
principles. This supersedes the placeholder templates this repo template
shipped under `.specify/templates/` — Spec Kit's own core templates take
over from there. See [`.specify/README.md`](../.specify/README.md).

## 5. Create a Slack webhook

In Slack: create (or reuse) a channel for this project, add an
[Incoming Webhook](https://api.slack.com/messaging/webhooks) app to it, and
copy the webhook URL. This one webhook feeds every notification this
template sends — see [WORKFLOWS.md](WORKFLOWS.md#slack-notifications) for
exactly what gets posted and when.

**Expect:** a URL shaped like `https://hooks.slack.com/services/…`.

## 6. Set repo secrets

Every key this template uses — what it's for, and whether you need it
locally at all — is listed in [`.env.example`](../.env.example). None of
these are read from a `.env` file at workflow runtime; GitHub Actions only
sees repo secrets, set like this:

```bash
gh secret set ANTHROPIC_API_KEY          # or CLAUDE_CODE_OAUTH_TOKEN
gh secret set SLACK_WEBHOOK_URL          # the URL from step 5
```

**Expect:** `gh secret list` shows both. Every workflow that posts to Slack
checks `secrets.SLACK_WEBHOOK_URL != ''` first and silently skips the
notification if it's unset — so you can defer step 5/6 and everything else
still runs, just without Slack messages. If you also need these values
locally (running the app itself, testing a workflow with `act`), copy
`.env.example` to `.env` and fill it in there — `.env` is already
gitignored, never commit it.

## 7. Create the GitHub Project

Follow [PROJECT_SETUP.md](PROJECT_SETUP.md) to create the board, the
`Status` field, and note the field/option IDs the agent workflows need.

**Expect:** a Project (v2) linked to the repo with the 8 Status options
listed in [AGENTS.md](AGENTS.md#state-machine).

## 8. Set branch protection (recommended)

Require the `test` job from `ci.yml` to pass before merge, and decide
whether the Review Agent is allowed to merge directly (see
[AGENTS.md](AGENTS.md#human-checkpoints)) or should only approve.

**Expect:** the `test` status check listed as required in
Settings → Branches.

## 9. Smoke-test the pipeline

1. Open an issue using the **Feature request** template (auto-labeled
   `needs-spec`) — this is how a feature/ticket gets raised in the first
   place; see [AGENTS.md](AGENTS.md#raising-issues) for the second,
   spec-kit-native way to raise issues once a spec exists.
2. Manually dispatch `agent-research.md` (Actions tab → Research Agent →
   Run workflow) rather than waiting for the schedule, pointing it at that
   issue number.
3. **Expect:** it ran `/speckit.specify` → `/speckit.clarify` →
   `/speckit.plan` → `/speckit.tasks`, opened a `specs/001-.../` PR labeled
   `spec`, and commented on the issue. That PR opening is what triggers
   `agent-notify.yml` — if step 5/6 are done, a "Spec drafted" Slack message
   lands in your channel within a few seconds, independent of the research
   agent run itself (see [WORKFLOWS.md](WORKFLOWS.md#why-agent-milestones-arent-emitted-by-the-agent)
   for why it's wired this way).
4. Label the issue `spec-approved`, dispatch `agent-dev.md`.
   **Expect:** it ran `/speckit.analyze` → `/speckit.implement` and opened
   an implementation PR labeled `agent-generated`, which triggers
   `agent-notify.yml`'s "Implementation PR opened" message.
5. **Expect:** `agent-review.md` fires automatically on that PR (it's
   PR-event triggered, no manual dispatch needed) and leaves a review.
   The review being submitted (whether approved or changes-requested)
   triggers `agent-notify.yml`'s "PR approved" / "Changes requested"
   message.
6. Push a commit to `main` (or open the PR against it).
   **Expect:** `ci.yml` runs and posts a CI status message with real test
   counts once you've filled in step 2's TODOs.
7. Manually dispatch `perf.yml` and `project-digest.yml` once each.
   **Expect:** a benchmark message and a pipeline-snapshot digest message,
   respectively — confirms both are wired before you rely on their
   schedules.

If any step fails, check the workflow's Actions log first — gh-aw runs are
read-only by default, so a missing safe-output declaration is the most
common cause of "the agent said it did X but nothing changed."

## Not included, by design

- No Slack channel is created for you — step 5 is manual, once.
- No live GitHub Project is created for you (Projects have no
  template-repository equivalent) — step 7 is manual, once, per project.
- No Copier CLI wiring for "ask questions, generate a stack-specific
  variant" — see the suggested build order in the original research doc
  (`docs/ARCHITECTURE.md`'s design notes): extract this template into a
  Copier template only after the loop has proven itself on one real project.
