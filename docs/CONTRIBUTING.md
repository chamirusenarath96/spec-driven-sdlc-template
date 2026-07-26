# Contributing

This repo's default workflow is agent-run (see [AGENTS.md](AGENTS.md)), but
everything the agents do is also open to humans — you can draft a spec, open
an implementation PR, or leave a review yourself at any point in the
pipeline.

## For humans

1. Open an issue with the **Feature request** template, or pick up an
   existing `needs-spec` issue and write `specs/NNN-slug/` yourself using the
   templates in `.specify/templates/`.
2. Once a spec is approved (`spec-approved` label), implement it or let the
   Dev Agent do it — either is fine, the acceptance criteria in `spec.md` are
   the contract either way.
3. PRs are reviewed the same way whether opened by a human or the Dev Agent:
   against the linked spec's acceptance criteria.

## Conventions

- One spec directory per feature, numbered sequentially — see
  [specs/README.md](../specs/README.md).
- PRs should link their spec and use the PR template.
- Don't hand-edit an approved spec without updating the linked issue — the
  Dev Agent (and any human picking up the same issue) treats `specs/` as the
  source of truth.

## Local setup

See [SETUP.md](SETUP.md) for one-time repo setup (secrets, `gh-aw` compile,
Project board). Once your stack is filled in, add local dev instructions
here (install, run, test commands) — kept separate from `TESTING.md`, which
is specifically about the test suite.
