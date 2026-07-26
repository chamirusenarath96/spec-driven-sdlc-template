# specs/

Each feature gets a numbered directory here, created by the Research Agent:

```
specs/
  001-feature-slug/
    spec.md
    research.md
    plan.md
    data-model.md   # only if the feature touches persisted data
    contracts/       # only if the feature adds/changes an API surface
    tasks.md
```

Templates for each file live in [`.specify/templates/`](../.specify/templates/).
Numbers are zero-padded and monotonically increasing across the whole repo
(not per-author) — check the highest existing `NNN-` prefix before creating a
new one.

Do not hand-edit a spec after it has moved to **Approved for Dev** on the
project board without also updating the linked issue — the Dev Agent treats
`specs/NNN-slug/` as the source of truth for what to build.
