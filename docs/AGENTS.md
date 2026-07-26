# Agent pipeline

## State machine

```
[Backlog]
   │ label: needs-spec
   ▼
[Spec In Progress]        ── Research Agent runs ──►  opens PR: specs/NNN-slug/
   │ PR opened
   ▼
[Spec Ready for Review]   ── human (or delegated agent) reviews spec PR
   │ spec PR merged + label: spec-approved
   ▼
[Approved for Dev]        ── Dev Agent runs ──► opens PR: implementation
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
