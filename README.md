# OldCrow Engineering Standards

Fleet-wide standards for the OldCrow C++/Python projects — [libhmm],
[pylibhmm], [libstats], [pylibstats], [ewcalc], and [corvus].

These documents live here rather than in any one project because they
belong to all six. Each project's `AGENTS.md` states only its
**deviations** from them, and where a project's `AGENTS.md` conflicts
with a standard, the `AGENTS.md` wins for that project — so an
undocumented deviation is a defect.

| Document | Covers |
|---|---|
| [CMAKE-HOUSE-STYLE.md](CMAKE-HOUSE-STYLE.md) | Target-first CMake: naming, scoping, generator expressions, build types, options, the install contract, dependencies, presets |
| [CI-HOUSE-STYLE.md](CI-HOUSE-STYLE.md) | GitHub Actions: runner budget, bounded parallelism, ISA hazards on hosted runners, action pinning, workflow linting, what CI must exercise |
| [DOC-CONVENTIONS.md](DOC-CONVENTIONS.md) | What `AGENTS.md`, `PLAN.md`, and the other repo documents are each for, and how they cross-reference |

## Records

[`records/`](records/) holds completed cross-repo efforts — the reasoning
and the evidence behind decisions that are now standing rules, plus any
follow-ups those efforts left open.

- [BUILD-STANDARDIZATION-PLAN.md](records/BUILD-STANDARDIZATION-PLAN.md)
  — the 2026-07 build-stack standardization across all six repos
  (complete; carries the open follow-up ledger).

## Using these

Every rule here was paid for by a real failure, and the incident is named
so the rule can be re-argued against evidence rather than restated as
dogma. If a rule no longer earns its keep, change it here — then update
the projects, not the other way round.

Reference these documents by URL. A `~/Development/...` path resolves on
exactly one machine; these are meant to work from any of them, from CI,
and from a browser.

[libhmm]: https://github.com/OldCrow/libhmm
[pylibhmm]: https://github.com/OldCrow/pylibhmm
[libstats]: https://github.com/OldCrow/libstats
[pylibstats]: https://github.com/OldCrow/pylibstats
[ewcalc]: https://github.com/OldCrow/ewcalc
[corvus]: https://github.com/OldCrow/corvus
