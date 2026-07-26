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

## CI / Deploy / Perf / Digest / Agent-Notify (plain GitHub Actions)

`ci.yml`, `deploy.yml`, `perf.yml`, `project-digest.yml`, and
`agent-notify.yml` are ordinary GitHub Actions — no agent involvement, no
compile step. Fill in the `TODO` steps in `ci.yml`/`deploy.yml`/`perf.yml`
for your actual stack; `project-digest.yml` and `agent-notify.yml` need no
stack-specific changes.

## Slack notifications

Every notification in this repo goes through one reusable composite action,
[`.github/actions/slack-notify`](../.github/actions/slack-notify/action.yml),
so they all look and behave the same way (same Block Kit shape, same
`status` → emoji mapping, same "skip silently if `SLACK_WEBHOOK_URL` isn't
set" behavior). There are five sources, and **all five are plain,
deterministic Actions steps — none of them depend on an agent deciding to
emit anything**:

| Source | Workflow | Fires on | Example content |
|---|---|---|---|
| CI status | `ci.yml` | Every push/PR | ✅/❌, tests passed/failed/total, coverage %, duration, link to run |
| Deploy status | `deploy.yml` | Every push to `main` | ✅/❌, environment, commit, deployed URL |
| Performance benchmark | `perf.yml` | Weekly cron, or push to app code, or manual | p50/p95 latency, throughput, bundle size |
| Project digest | `project-digest.yml` | Weekdays, after the agent sweeps | Issues awaiting spec / in review / approved, open PRs, PRs merged in last 24h |
| Agent milestone | `agent-notify.yml` | A PR labeled `spec` or `agent-generated` opens; a review is submitted on one | Spec/PR link, author, reviewer, review verdict |

Every one of them calls the composite action the same way:

```yaml
- name: Notify Slack
  if: secrets.SLACK_WEBHOOK_URL != ''
  uses: ./.github/actions/slack-notify
  with:
    webhook-url: ${{ secrets.SLACK_WEBHOOK_URL }}
    status: success        # success | failure | warning | info
    title: "..."
    summary: "..."
    fields: '[{"label": "...", "value": "..."}]'
    link: "..."
```

### Why agent milestones aren't emitted by the agent

An earlier version of this template had `agent-research.md`, `agent-dev.md`,
and `agent-review.md` each declare a `slack-notify` custom safe-output and
instructed the agent to emit it with a self-composed summary. That's a
weaker design and this template no longer does it, on purpose:

- **Reliability** — a custom safe-output only fires if the agent's run
  reaches that instruction. A run that errors, times out, or gets a
  `timeout_minutes` cutoff produces silence instead of a message. A plain
  Actions trigger (`pull_request: types: [opened]`,
  `pull_request_review: types: [submitted]`) fires whenever the real GitHub
  state changes, independent of whether the agent run that caused it
  finished cleanly.
- **Trustworthiness of content** — an agent-composed notification reports
  what the agent *believes* it did. `agent-notify.yml` instead reads the
  actual event payload (`github.event.pull_request.title`,
  `github.event.review.state`, etc.) — the same facts a human clicking
  around GitHub would see, not a self-report.
- **It doesn't care who did the work** — because `agent-notify.yml` triggers
  on labels (`spec`, `agent-generated`) and the real
  `pull_request_review` event, it fires identically whether the Research/
  Dev/Review Agent did the work or a human did the exact same steps by
  hand. An agent-emitted safe-output would only fire for the agent path.

The three `agent-*.md` personas no longer declare or mention `slack-notify`
at all — see the "Slack notification" note at the end of each file's body,
which just points here.

## Required secrets

| Secret | Used by | Purpose |
|---|---|---|
| `ANTHROPIC_API_KEY` or `CLAUDE_CODE_OAUTH_TOKEN` | `agent-*.md` workflows | Runs Claude Code as the workflow engine |
| `SLACK_WEBHOOK_URL` | `ci.yml`, `deploy.yml`, `perf.yml`, `project-digest.yml`, `agent-notify.yml` | Posts to your project's Slack channel — every caller checks it's set and skips the notification silently if not |
| `GITHUB_TOKEN` (default) | all workflows | Standard Actions token; `gh-aw` safe-outputs and `project-digest.yml`'s `gh` calls use this |

See [`.env.example`](../.env.example) for the same list with local-dev
notes, and [SETUP.md](SETUP.md) for how to actually set each one.
