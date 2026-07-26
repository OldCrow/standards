# Documentation Conventions — OldCrow Projects

How the repo-level documents in this ecosystem are structured and what
each one is *for*. The point is that a person or an agent arriving cold
can tell, without asking, where the current state lives and which
document wins when two disagree.

## 1. The files each repo carries

| File | Purpose | Audience |
|---|---|---|
| `README.md` | What the project is, how to build and install it | Users |
| `AGENTS.md` | Working contract: commands, architecture, conventions, deviations | Agents and contributors |
| `PLAN.md` | Current state: what's decided, what's open, next step | Whoever picks the work up |
| `CHANGELOG.md` | Released, user-visible change history | Users |
| `CONTRIBUTING.md` | Release process, review expectations | Contributors |
| `CLAUDE.md` | One line: `See @AGENTS.md ...` | Tooling |

`AGENTS.md` is canonical. `CLAUDE.md` is never a competing rules file —
it is a pointer, and nothing else.

## 2. AGENTS.md

Required sections: **Overview**, **Build/Test/Run Commands**,
**Architecture**, **Conventions**, **Open Items**. A section with nothing
to say gets a placeholder, not a deletion — its absence should mean "not
yet written", never "does not apply".

On standards: each repo's AGENTS.md states only its **deviations** from
the fleet standards ([CMake](CMAKE-HOUSE-STYLE.md), [CI](CI-HOUSE-STYLE.md)),
and each deviation says why. Where a repo's AGENTS.md conflicts with a
standard, **the AGENTS.md wins for that repo** — which is exactly why an
undocumented deviation is a defect: it wins silently. Keep the section
self-sufficient for day-to-day work, so an agent that cannot reach the
standards repo is still correct about this repo.

## 3. PLAN.md

**One canonical plan file per repo, updated in place.** Never dated
snapshots, never `PLAN-v2.md`.

It holds *state*, not activity: what has been decided, what is still
open, and the next concrete step. A running checklist of what was done
belongs in commit messages and issues, which already record it better and
with timestamps.

- Tag entries `[DERIVED]` (follows from evidence in the repo),
  `[ILLUSTRATIVE]` (example or sketch, not a commitment), or `[OPEN]`
  (undecided — say who or what unblocks it).
- **Absolute dates.** "Last week" is unreadable six months on and
  actively misleading after a compaction.
- **Point at sources of truth; do not restate them.** A dependency pin,
  a version, or a flag value copied into prose becomes wrong on the next
  bump and there is no mechanism that will catch it. Name the file that
  holds the value, or add a check that fails when they drift.
- Write durable state to the plan file *before* ending a session or
  compacting a conversation. Anything living only in conversation does
  not survive.
- Reading the plan file is how a session starts. If it does not answer
  "where does this stand", that is a bug in the plan file.

## 4. Cross-repo references

Fleet-wide standards live in this repo, outside any code repo, because
they belong to no single one. Reference them by **URL**, not by a local
path — `~/Development/...` resolves on exactly one machine, and a link
that only works for the person who wrote it is not a reference.

Cross-repo effort records go in [`records/`](records/) here, for the same
reason: an effort spanning six repos has no home in any of them.

## 5. Writing standard

**ABC: Accurate, Brief, Clear.** Prefer verbs to nouns and active voice
to passive. No hyperbole — especially internally, where "critical" and
"massive" are noise that crowds out the one time something really is.

Say what a thing does and, where it is not obvious, why it is that way.
A comment or doc line that explains a rejected alternative is usually
worth more than one restating what the code plainly says: it is the only
record of a decision that would otherwise be re-litigated. Where a rule
came from a specific failure, name the failure.

## 6. Commit messages

Conventional-commit style — `type(scope): summary` in the imperative,
e.g. `fix(ci): bound build parallelism to available memory`. Types in use:
`feat`, `fix`, `docs`, `ci`, `build`, `refactor`, `test`, `perf`, `chore`.

The body explains *why*, and names the evidence: the failing job, the
measured number, the issue. A commit that changes behavior on the
strength of a measurement should carry the measurement.
