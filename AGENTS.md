# AGENTS.md — OldCrow standards

## Overview

Fleet-wide standards for six sibling projects: libhmm, pylibhmm, libstats,
pylibstats, ewcalc, corvus. Documents only — no code, no build, no tests.
See [README.md](README.md) for what each document covers.

This repo is unusual in one way worth knowing before editing: **changing a
file here changes the rules six other repos are held to.** Treat an edit as
a cross-repo change, not a doc tweak.

## Build/Test/Run Commands

None. Markdown only.

The one check worth running before pushing is that any link you added
resolves — relative links within the repo, and the sibling repos' links
back to it:

```bash
grep -rn "OldCrow/standards" ~/Development/{libhmm,pylibhmm,libstats,pylibstats,ewcalc,corvus}
```

## Architecture

- `*-HOUSE-STYLE.md`, `DOC-CONVENTIONS.md` — the standards themselves.
  Current, binding, edited in place.
- `records/` — completed cross-repo efforts. Historical; edit only to
  correct the record or to retire an item from a ledger it carries.

## Conventions

- **Edit the standard first, then propagate.** A rule changed here without
  the matching change in the affected repos is half a change, and the half
  that ships is the one nobody follows.
- **A repo's `AGENTS.md` wins over a standard for that repo.** So a
  deviation is legitimate only if it is written down there, with its
  reason. An undocumented deviation is a defect — it wins silently.
- **Name the failure.** Every rule here should be traceable to what went
  wrong without it. A rule with no incident behind it is a preference, and
  should either say so or come out.
- **No rule without a follower.** Do not add a standard the six repos do
  not actually meet; write it as an open item in the relevant repo instead.
- Writing follows [DOC-CONVENTIONS.md](DOC-CONVENTIONS.md) — ABC, active
  voice, absolute dates.
- Reference documents by URL when citing them from other repos; a
  `~/Development/...` path resolves on one machine only.

## Open Items

No `PLAN.md` here, deliberately — this repo has no ongoing work of its own.
Open follow-ups belong either to a specific repo's `PLAN.md` or to the
ledger at the end of
[records/BUILD-STANDARDIZATION-PLAN.md](records/BUILD-STANDARDIZATION-PLAN.md).

`main` enforces verified signatures with no bypass, and refuses
force-pushes and branch deletion. An unsigned commit is rejected at push
time, so a stale `gpg-agent` fails loudly rather than landing unsigned
work — resync it (`gpgconf --kill gpg-agent && gpg-agent --daemon
--enable-ssh-support`) rather than working around the rule.
