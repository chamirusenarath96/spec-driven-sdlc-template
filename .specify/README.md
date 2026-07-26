# .specify/

`templates/` in this directory currently holds **hand-written placeholders**
that mirror the shape of a real [Spec Kit](https://github.com/github/spec-kit)
project (`spec-template.md`, `plan-template.md`, `tasks-template.md`,
`data-model-template.md`, `research-template.md`) — written before this repo
had the real tool installed, so a spec drafted by hand would still follow a
consistent structure.

Once you run `specify init --here` (see
[docs/SETUP.md](../docs/SETUP.md#4-install-spec-kit-and-establish-the-constitution)),
real Spec Kit takes over this directory:

- `.specify/templates/` gets Spec Kit's own core templates — expect them to
  overwrite or sit alongside the placeholders here. That's expected; the
  placeholders exist only so this template repo has *something* structurally
  correct before Spec Kit is installed.
- `.specify/memory/constitution.md` is created by `/speckit.constitution` —
  your project's non-negotiable principles, checked against every later
  `/speckit.plan`.
- `.claude/commands/speckit.*.md` (not under this directory, but installed by
  the same command) are the actual slash-command definitions the Research
  and Dev Agent personas drive — see `.claude/agents/research-agent.md` and
  `dev-agent.md`.

Do not hand-edit `.specify/templates/` after Spec Kit is installed except
through Spec Kit's own override mechanism
(`.specify/templates/overrides/` — see the
[Extensions & Presets](https://github.com/github/spec-kit#-making-spec-kit-your-own-extensions--presets)
section of Spec Kit's README) — a raw edit will be silently clobbered the
next time someone runs `specify self upgrade` or reinstalls a preset.
