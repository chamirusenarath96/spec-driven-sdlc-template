---
name: "gha-ci-failure-triage"
description: >-
  Repo convention for GitHub Actions workflows: every job that can fail in a
  way a human needs to act on must auto-file a labeled GitHub issue on
  failure, using the P1/P2/P3 priority convention and the required `issues:
  write` permission. Apply this whenever creating a new workflow under
  .github/workflows/, adding a job to an existing one, or reviewing a
  workflow for completeness.
user-invocable: false
disable-model-invocation: false
---

## Why this exists

This repo's spec template (`.specify/templates/spec-template.md`) doesn't
carry story-level priority — that convention lives in the upstream Spec Kit
template used by other projects in this org (e.g. `card-max`,
`autoshop-takumi`), which tag user stories `P1`/`P2`/`P3` (P1 = most
critical). `card-max` also auto-files a labeled GitHub issue whenever a CI
job fails (`ci.yml`, `crawler.yml`, `enrich.yml`), so failures get triaged
instead of sitting red and ignored.

This skill merges both conventions into one rule for every workflow in this
repo: **on failure, file an issue, and label it with the same P1/P2/P3
priority scale specs use for user stories.**

## Rule

Any `.github/workflows/*.yml` job whose failure represents something a human
must act on (build breakage, failed deploy, failed migration, failed
scheduled job, etc.) MUST:

1. **Declare `issues: write` permission** — either at the workflow level or
   on the specific job that files the issue. Don't grant it workflow-wide if
   only one job needs it; scope it to that job's `permissions:` block
   instead. A job that only reads/tests doesn't need it.

2. **File an issue on failure** via the shared composite action at
   `.github/actions/file-ci-failure-issue`, called from a step with
   `if: failure()`:

   ```yaml
   permissions:
     contents: read
     issues: write

   steps:
     - uses: actions/checkout@v4
     # ...your real steps...

     - name: File issue on failure
       if: failure()
       uses: ./.github/actions/file-ci-failure-issue
       with:
         component: deploy          # short slug: ci, deploy, migrate, crawler...
         priority: P1               # see mapping below
         urgent: "true"             # only for P1
         summary: "Production deploy failed after CI passed."
   ```

   Don't hand-roll a new `actions/github-script` block per workflow — that's
   how `card-max` ended up with the same issue-creation snippet duplicated
   three times (`ci.yml`, `crawler.yml`, `enrich.yml`). Use the composite
   action so the label set and issue format stay consistent repo-wide.

3. **Label with the P1/P2/P3 priority convention**, mirroring how specs
   prioritize user stories (P1 = most critical):

   | Priority | When to use | Labels applied |
   |----------|-------------|-----------------|
   | `P1` | Production-impacting: deploy, promote, migration, or any job that runs on `push` to the default branch and gates what's live for users | `bug`, `urgent`, `P1`, `<component>` |
   | `P2` | Blocks merging/integration but isn't live yet: PR-time lint/type-check/unit/e2e/build failures | `bug`, `P2`, `<component>` |
   | `P3` | Advisory/non-blocking: scheduled warmups, smoke tests, performance/lighthouse audits, crawlers that degrade gracefully | `bug`, `P3`, `<component>` |

   `urgent` is reserved for P1 only — don't add it to P2/P3 issues, or it
   loses meaning as a triage signal.

4. **Component slug** is the workflow's subject, not the workflow filename:
   use `deploy`, `migrate`, `crawler`, `enrich`, etc. — matching what
   `card-max` already uses — so issues from different workflows are still
   groupable by label.

## Applying this to a new workflow

When asked to add a new `.github/workflows/*.yml` file, or add a job to an
existing one, in this repo:

1. Decide the job's priority tier from the table above.
2. Add `issues: write` to that job's (or the workflow's) `permissions:`.
3. Add the `file-ci-failure-issue` step with `if: failure()` as the last
   step of the job, using the resolved `component`/`priority`/`urgent`.
4. If the job already has its own failure notification (e.g. Slack via
   `.github/actions/slack-notify`), keep both — the issue is for tracking
   and assignment, the Slack ping is for immediate visibility. They're not a
   replacement for each other.

## Reference implementation

`.github/actions/file-ci-failure-issue/action.yml` is the shared composite
action. `.github/workflows/ci.yml`'s `notify` job shows the pattern wired
into this repo's own placeholder CI — use it as the template for new jobs.
