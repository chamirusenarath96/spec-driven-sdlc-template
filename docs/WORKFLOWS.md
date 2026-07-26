# Workflows

## Agentic workflows (`gh-aw`)

`.github/workflows/agent-research.md`, `agent-dev.md`, `agent-review.md` are
written in [GitHub Agentic Workflows](https://github.github.com/gh-aw/)
Markdown format — natural-language instructions with YAML frontmatter for
triggers/permissions/safe-outputs. They are **not** runnable as-is; compile
them first:

```bash
gh extension install github/gh-aw
gh aw compile
```

This produces `.github/workflows/agent-*.lock.yml` (gitignored — regenerate,
don't hand-edit, don't commit) which is the actual Actions workflow GitHub
runs.

### Triggers, per workflow

| Workflow | Event trigger | Scheduled fallback | Manual |
|---|---|---|---|
| `agent-research.md` | Issue labeled `needs-spec` | Weekdays 13:00 UTC, ≤2 issues/run | `workflow_dispatch` with `issue_number` |
| `agent-dev.md` | Issue labeled `spec-approved` | Weekdays 15:00 UTC, ≤1 issue/run | `workflow_dispatch` with `spec_dir` |
| `agent-review.md` | PR opened/synchronized, labeled `agent-generated` | — (event-driven only) | `workflow_dispatch` with `pr_number` |

Schedules are staggered on purpose — see
[ARCHITECTURE.md](ARCHITECTURE.md#known-limitations-as-of-this-templates-creation-2026-07)
on the shared 5-hour rate-limit window.

### Safe outputs

Each workflow only writes through declared **safe outputs**
(`create-pull-request`, `add-comment`, `update-issue`, or a `custom` job) —
this is what makes it safe to run these unattended. Do not add `contents:
write` to the top-level `permissions:` block as a shortcut; if an agent needs
a new kind of write, add a new safe output instead so the blast radius stays
declared and auditable. See the
[Safe Outputs reference](https://github.github.com/gh-aw/reference/safe-outputs/).

`agent-review.md` uses a `custom` safe-output job (`pr-review-decision`)
because approving/merging a PR and moving a Project item aren't covered by
the built-in safe-output set — implement that job per the
[Custom Safe Outputs reference](https://github.github.com/gh-aw/reference/custom-safe-outputs/).

### Spec Kit dependency

`agent-research.md` and `agent-dev.md` invoke real
[Spec Kit](https://github.com/github/spec-kit) slash commands
(`/speckit.specify`, `/speckit.plan`, `/speckit.implement`, etc. — see
`.claude/agents/research-agent.md` and `dev-agent.md`). The runner does
**not** need the `specify` CLI installed at workflow-run time: `specify init
--here` (a one-time local step, see [SETUP.md](SETUP.md)) writes the actual
command files into `.claude/commands/speckit.*.md`, and those get committed
to the repo like any other file. Claude Code reads them the same way in a
GitHub Actions runner as it would in your terminal. If those command files
are missing (Spec Kit was never installed), the agents will fail to find
`.claude/commands/speckit.*.md` and stop rather than improvise the spec
structure by hand — that's deliberate, not a bug to work around by re-adding
freeform templates.

### Budget controls already in place

- Scheduled sweeps cap how many issues/PRs get processed per run (2 for
  research, 1 for dev) — raise these only after you've watched real token
  spend.
- `timeout_minutes` is set per workflow to fail fast instead of burning a
  run.
- Consider adding a cheap triage step (Haiku) ahead of the real agent call if
  you find scheduled sweeps frequently find nothing to do — not wired by
  default, since it adds complexity this template doesn't assume you need
  yet.

## CI / Deploy (`ci.yml`, `deploy.yml`)

Plain GitHub Actions, not gh-aw — no agent involvement, just build/test/
deploy plus a Slack notification via
[`slackapi/slack-github-action`](https://github.com/marketplace/actions/slack-github-action).
Fill in the `TODO` steps for your actual stack. Both need the
`SLACK_WEBHOOK_URL` repo secret — see [SETUP.md](SETUP.md).

## Required secrets

| Secret | Used by | Purpose |
|---|---|---|
| `ANTHROPIC_API_KEY` or `CLAUDE_CODE_OAUTH_TOKEN` | `agent-*.md` workflows | Runs Claude Code as the workflow engine |
| `SLACK_WEBHOOK_URL` | `ci.yml`, `deploy.yml`, and `agent-review.md`'s custom safe-output job if you wire agent activity notifications | Posts to your project's Slack channel |
| `GITHUB_TOKEN` (default) | all workflows | Standard Actions token; `gh-aw` safe-outputs use this for their declared writes |

None of these are set in this template — see [SETUP.md](SETUP.md).
