# Agent pipeline

## Raising issues

There is one tool for tickets in this template: **GitHub Issues**, tracked on
a **GitHub Project (v2)** board (see [PROJECT_SETUP.md](PROJECT_SETUP.md)) —
no separate PM tool. There are two distinct ways an issue enters this
pipeline, and they answer different questions ("what should we build" vs.
"how do we track the pieces of something already scoped"):

1. **Human raises the initial ask.** Open an issue with the
   **Feature request** template (`.github/ISSUE_TEMPLATE/feature-request.md`)
   — auto-labeled `needs-spec`. This is a *raw* ask: a paragraph of what's
   needed and why, nothing more. It's the entry point for anything that
   doesn't have a spec yet.
2. **Spec Kit fans a scoped spec out into per-task issues.** Once the
   Research Agent has run `/speckit.tasks` and produced `specs/NNN-slug/tasks.md`,
   running `/speckit.taskstoissues` converts each task into its own tracked
   GitHub issue, linked back to the spec. Use this when a feature is large
   enough that you want individual tasks reviewed, assigned, or scheduled
   independently rather than landing as one Dev Agent PR — e.g. a spec with
   8 tasks where 3 are genuinely independent workstreams. For a typical
   small feature, skip it; one implementation PR covering all of `tasks.md`
   is simpler. The Research Agent persona (`.claude/agents/research-agent.md`)
   treats this as optional for exactly that reason.

Either way, the issue is what carries state through the pipeline via labels
and its linked Project item — there's no second source of truth to keep in
sync.

## State machine

```
[Backlog]
   │ label: needs-spec
   ▼
[Spec In Progress]  ── Research Agent runs /speckit.specify → /speckit.clarify
   │                    → /speckit.plan → /speckit.tasks (→ optionally
   │                    /speckit.taskstoissues) ──► opens PR: specs/NNN-slug/
   │ PR opened
   ▼
[Spec Ready for Review]   ── human (or delegated agent) reviews spec PR
   │ spec PR merged + label: spec-approved
   ▼
[Approved for Dev]  ── Dev Agent runs /speckit.analyze → /speckit.implement
   │                    ──► opens PR: implementation
   │ PR opened
   ▼
[In Review]                ── Review Agent runs
   │
   ├─ approve ──► [Done]
   │
   └─ request changes ──► [In Development] + label: changes-requested
                            (Dev Agent picks up on next scheduled run)
```

These are the custom **Status** field options to create on your GitHub
Project — see [PROJECT_SETUP.md](PROJECT_SETUP.md) for exact setup steps.
The `/speckit.*` commands are real [Spec Kit](https://github.com/github/spec-kit)
slash commands — see [docs/SETUP.md](SETUP.md#4-install-spec-kit-and-establish-the-constitution)
for installing them and [README.md](../README.md#the-full-picture-issue-to-merge)
for the swimlane view of who calls what, when.

## Labels used

| Label | Meaning | Set by |
|---|---|---|
| `needs-spec` | Issue needs a spec drafted | Human (default on the feature-request issue template) |
| `spec` | PR adds/modifies a `specs/` directory | Research Agent |
| `spec-in-review` | Spec PR open, awaiting approval | Research Agent |
| `spec-approved` | Spec approved, ready for implementation | Human |
| `agent-generated` | PR was opened by the Dev Agent | Dev Agent |
| `in-review` | Implementation PR open | Dev Agent |
| `changes-requested` | Review Agent requested changes | Review Agent |

## Agent personas

Full behavioral contracts live in `.claude/agents/`:

- [`research-agent.md`](../.claude/agents/research-agent.md)
- [`dev-agent.md`](../.claude/agents/dev-agent.md)
- [`review-agent.md`](../.claude/agents/review-agent.md)

Each is invoked by the matching workflow in `.github/workflows/agent-*.md`
(see [WORKFLOWS.md](WORKFLOWS.md) for trigger details and budget controls).

## Human checkpoints

This pipeline is agent-run, not agent-only. Two points are deliberately left
to a human by default:

1. **Approving a spec** (`spec-approved` label) — the Research Agent drafts,
   it does not self-approve. You can delegate this to an agent later once
   you trust the spec quality, but it's not wired that way out of the box.
2. **Merging** — the Review Agent approves and can merge if branch protection
   allows it; if you'd rather keep merge as a manual click, remove merge
   permission from `agent-review.md`'s safe-outputs and just let it approve.

## Escalation

If an agent can't proceed (ambiguous spec, failing tests it can't fix,
conflicting instructions), it must stop and comment on the issue/PR rather
than guessing — this is a hard rule in all three persona files. Treat a
stalled item with no progress after 2 scheduled runs as a signal to look at
it yourself.
