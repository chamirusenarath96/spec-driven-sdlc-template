# Code review: CodeRabbit + the Review Agent

This repo runs **two** automated reviewers on every PR, and they do
different jobs on purpose:

| | [CodeRabbit](https://coderabbit.ai) | Review Agent (`.claude/agents/review-agent.md`) |
|---|---|---|
| What it checks | General code quality, bugs, security (OWASP-style issues, secrets via `gitleaks`), style | The diff against the linked spec's acceptance criteria in `specs/NNN-slug/spec.md` |
| Knows about this repo's specs? | Only what `path_instructions` in `.coderabbit.yaml` tells it — informationally, not as its main job | Yes — reads `spec.md` and `tasks.md` directly, that's its whole purpose |
| Can block merge? | No — `request_changes_workflow: false` in `.coderabbit.yaml`, so it never auto-approves or blocks | Yes — its `pr-review-decision` safe-output is what moves the Project item to **Done** or back to **In Development** |
| Runs on | Every PR (GitHub App, event-driven) | PRs labeled `agent-generated` (see `agent-review.md`) |
| Setup | GitHub App install + Slack connection, both in CodeRabbit's own dashboard | Already wired via `gh-aw` — see [WORKFLOWS.md](WORKFLOWS.md) |

Think of CodeRabbit as a fast, always-on first pass that catches the things
a general-purpose linter-plus-senior-engineer would catch, and the Review
Agent as the pipeline-specific gate that decides whether this PR actually
satisfies what was scoped. Neither replaces the other; a PR you'd want to
merge should have both a clean CodeRabbit pass and a Review Agent approval.

## Setup

Two steps happen outside this repo, in CodeRabbit's own systems, and need
your GitHub/Slack account access to authorize — nothing here can do this
for you:

1. **Install the GitHub App**: go to
   [coderabbit.ai](https://coderabbit.ai), sign in with GitHub, and install
   the CodeRabbit app on this repository (or your whole org). This grants
   CodeRabbit read/write access to PRs — review it like you would any
   third-party GitHub App permission request.
2. **Connect Slack**: in CodeRabbit's dashboard, under Integrations, connect
   the same Slack workspace/channel you set up in
   [SETUP.md](SETUP.md#5-create-a-slack-webhook). This is CodeRabbit's own
   native integration — it does **not** go through this repo's
   `.github/actions/slack-notify` composite action or the
   `SLACK_WEBHOOK_URL` secret; CodeRabbit posts its own review summaries
   directly once you've connected it.

Once both are done, `.coderabbit.yaml` (already committed) takes effect on
the next PR — no further action needed in this repo.

## Why CodeRabbit doesn't auto-approve here

`request_changes_workflow: false` in `.coderabbit.yaml` means CodeRabbit
only ever leaves comments — it never submits an approving or
changes-requested review itself. This is deliberate: this template's merge
gate is the Review Agent's `pr-review-decision` safe-output (see
[AGENTS.md#human-checkpoints](AGENTS.md#human-checkpoints)), and having two
systems independently deciding approve/request-changes on the same PR
would make branch protection rules ambiguous (whose decision wins?) and
risks CodeRabbit approving a PR before the Review Agent has checked spec
conformance at all. If you'd rather let CodeRabbit gate on general code
quality *in addition to* the Review Agent gating on spec conformance, flip
`request_changes_workflow: true` and add CodeRabbit's check as a second
required status check in branch protection (see
[SETUP.md#8-set-branch-protection-recommended](SETUP.md#8-set-branch-protection-recommended)) —
just be aware you now need both to pass.

## What's excluded from CodeRabbit's review, and why

`.coderabbit.yaml`'s `path_filters` excludes `specs/**`, `.specify/**`, and
`docs/**` — those are spec-kit artifacts and this template's own
documentation, not application code, and the Review Agent already reads
`spec.md` directly against the diff. `ignore_title_keywords: ["spec:"]`
also skips the Research Agent's spec-only PRs entirely (see
`agent-research.md`'s `title-prefix: "spec: "`) since there's no code in
them to review.
