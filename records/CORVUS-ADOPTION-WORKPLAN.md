# Corvus Adoption Workplan (libstats, libhmm, corvus, pylibstats, pylibhmm, ewcalc)

Status: **ACTIVE — written 2026-09-28**, at the return from the
2026-09-17 → 2026-09-28 travel period. This is the one in-progress record
in `records/`; it becomes historical when libstats v2.5.0 and the libhmm
spike verdict have shipped.

**What this file is.** The cross-repo order of work and the machine each
task needs. It exists because a session on one fleet machine cannot see a
session on another, and the ordering spans six repos.

**What it is not.** Project state. Each repo's `PLAN.md` and its GitHub
issues carry the detail and win on any disagreement. A session that
finishes a task updates that repo's `PLAN.md` first, then strikes the row
here.

## Starting state (2026-09-28)

| Repo | Release | Validated on |
|---|---|---|
| corvus | v1.0.1 (2026-09-19) | v1.0.0: all three machines. v1.0.1 (build-system patch): Kaby Lake + CI |
| libstats | v2.4.1 (2026-09-19) | v2.4.0: all three machines. v2.4.1: Kaby Lake + CI |
| libhmm | v4.4.1 (2026-08-26) | All three machines |
| pylibstats | 0.7.1 (2026-09-19) | CI |
| pylibhmm | 0.12.1 (2026-09-19) | CI |
| ewcalc | v1.2.0 (2026-09-13) | — |

Facts that change the plan:

- **corvus is off the critical path.** v1.0.0 froze the surface; all
  remaining adoption work is consumer-side. Consumers pin v1.0.1.
- **The Mac Mini M1 moved from macOS Tahoe 26 to macOS 28 during the
  travel period** [user, 2026-09-28]. Every M1 validation and timing
  record before that date is a Tahoe record. Toolchain and Apple libm
  versions on the new OS are unmeasured until P1 runs.
- **Travel-period decisions are recorded, not pending:** libstats v2.5.0
  scope (full swap, milestone #6), the wheel cost-out (pylibstats #20),
  and the libhmm spike's paper half with pre-registered go/no-go criteria
  (libhmm #106).

## Working rules

- Timing numbers come from Release builds on a quiet machine under the
  repo's own gate. A run that misses the gate is INDICATIVE and creates a
  record that must be redone.
- Tasks 3 and 7 both need quiet machines. Schedule them on different
  days per machine.
- Let the M1 settle after its OS upgrade before any timing run. Under
  Tahoe, background media analysis kept the 5% gate out of reach (corvus
  `docs/PERFORMANCE.md`); an OS upgrade usually restarts that indexing.

## Precursor work (catch-up, per machine)

| # | Machine | Task | Why |
|---|---|---|---|
| P1 | M1 | Pull all repos; check the toolchain after the OS upgrade (Command Line Tools, AppleClang version, Homebrew, Highway 1.4.0, Python venvs); wipe build directories | The build environment changed under every checkout |
| P2 | M1 | Re-run native correctness: libstats v2.4.1 ctest, libhmm v4.4.1 ctest with the NEON ULP gates, corvus v1.0.1 tier-asserted NEON | Re-establishes the NEON baseline on macOS 28 |
| P3 | M1 | Reconcile the three stale libstats branches and the loose NEON patch | libstats `PLAN.md` Next Steps 5. Until done, no stale-branch sweep on the M1 checkout of libstats |
| P4 | M1 | Update the fleet tables from Tahoe to macOS 28, using P1's measured versions | libstats `AGENTS.md`, corvus `docs/ENVIRONMENT.md`, libhmm `PLAN.md` Local Machine State |
| ~~P5~~ | ~~Zen 4~~ | ~~Pull all repos; fresh builds; native smoke of libstats v2.4.1 and corvus v1.0.1, including the corvus toolchain guard under clang-cl, MSVC and mingw~~ | DONE 2026-09-28: libstats ctest 74/74 (MSVC, AVX-512); corvus ctest 34/34 tier-asserted AVX3_ZEN4 (clang-cl); guard verdicts as designed under all three compilers |
| P6 | Kaby Lake | None | The travel work was done on this machine |

## Main spine (libstats)

| # | Task | Issues / milestones | Machines |
|---|---|---|---|
| 1 | Pre-swap baseline: regenerate the characterization sweep at v2.4.1 | Owed confirmation in libstats `PLAN.md` In Progress (x86 contract violations 63 → 61; NEON geometric logpdf) | All three |
| 2 | v2.5.0 core swap on a dev branch, corvus pinned at v1.0.1 | libstats milestone #6; starts from the call-site inventory and the no-broadcast design point | Any one |
| 3 | Post-swap sweep, validation matrix, same-machine x86 erf timing; correct the unmeasured `~5×` erf comment in `dispatch_thresholds.h` from the result | libstats `PLAN.md` Next Steps 3(a)–(b); capped tiers on Kaby Lake | All three |
| 4 | libstats v2.5.0 release | Milestone #6 close | Any, with the signing key |
| 5 | pylibstats v0.8.0: pin bump, LICENSE and NOTICE, Windows wheel job | pylibstats #20; needs side task J done first | Any, plus CI |
| 6 | libstats post-adoption patch | libstats milestone #8; #144 needs the first post-swap Zen 4 session | All three |

Task 1 precedes task 2 for a reason beyond the owed confirmation: on the
M1 it separates what macOS 28 changed from what corvus changes. Without
it, the post-swap NEON differences confound the two.

## Parallel track (libhmm)

| # | Task | Issues / milestones | Machines |
|---|---|---|---|
| 7 | Spike fleet half: branch, measurements, apply the pre-registered criteria | libhmm #106 (runbook on the issue) | All three, quiet |
| 8 | Fit accuracy and kernel hygiene patch, trimmed or not by task 7's verdict | libhmm milestone #5; #94 needs its benchmark pass before the trim | All three |
| 9 | pylibhmm pin bump | Follows task 8; needs side task J done first | Any, plus CI |

## Side tasks (no dependency on the spine)

| # | Task | Issues | Machines |
|---|---|---|---|
| A | Fix the libstats coverage measurement, then port to libhmm | libstats #152, libhmm #108 | CI only |
| B | Uniform speedup gate flake | libstats #129 | Zen 4 |
| C | M1 quiet-bench retry on macOS 28 | corvus `docs/PERFORMANCE.md` M1 rows are INDICATIVE | M1 |
| D | corvus v1.1.0 kernel work | corvus #31, #22, #21, #18; #37 capped rows only if task 3 needs them | Kaby Lake first, then all |
| K | Linux aarch64 validation gap: the aarch64 wheel will ship corvus NEON compiled by GCC, a compiler × ISA pair no fleet machine validates | libstats `PLAN.md` Cross-Repo Dependencies; decide the remedy before task 5 ships wheels | CI (aarch64 runner) |
| E | `load_hmm` wrapper decision | pylibhmm #31 | Any |
| F | `errorf_inv` exposure investigation | libhmm #103 | Any |
| G | Radar jamming calculator | ewcalc #88; needs the physical book pins | Windows and Linux UI |
| H | Plan hygiene | libstats GitHub reconcile (last done 2026-09-05) and milestone #8 bucketing pass | Any |
| ~~I~~ | ~~Merge the open dependabot PRs~~ | DONE 2026-09-28: all eight merged; `main` CI green in libstats, libhmm, corvus and ewcalc | — |
| J | scikit-build-core 1.1.0 breaks free-threaded Python discovery in pylibstats and pylibhmm | Capped `<1.1` on 2026-09-28; CI confirmed green on the capped commits. Open, in order: (1) reproduce on CI, not locally — a throwaway branch with a `workflow_dispatch` job on `ubuntu-latest` / 3.14t / 1.1.0 that prints the generated `CMakeInit.txt` and `Python_FIND_ABI`, changing one variable at a time (drop `wheel.py-api`; newer CMake); (2) decide: upstream defect, or a change to this project's `find_package(Python ...)`; (3) file upstream at scikit-build/scikit-build-core if a defect; (4) lift the cap in both repos together. Detail in both repos' `PLAN.md` | CI only |

## Later milestones (not scheduled)

Listed so this record names every open milestone. None starts before the
spine and the libhmm track finish, and none has an order yet.

| Repo | Milestone | Depends on |
|---|---|---|
| libstats | v2.6.0 — New Distributions (#58–#62) | v2.5.0 shipped; pylibstats bindings and stubs follow |
| libstats | v3.0.0 — Architecture Refactor (#40–#43, #128) | After v2.6.0 |
| libhmm | v4.5.0 — Algorithm Coverage (#47, #48, #51, #52, #96, #97) | After task 8 |
| libhmm | v5.0.0 — API (#95, #98) | After v4.5.0 |
| libhmm | Unmilestoned features (#50 HSMM, #53 IOHMM) | No milestone assigned |
| corvus | v1.1.0 conditional and watch items (#19, #20 wait on libstats consumers; #28, #29 wait on upstream) | Side task D covers the rest of v1.1.0 |
| pylibhmm | Parity-ledger audit of pre-0.12.0 surfaces | Opportunistic; no issue filed |

## Open decisions [user]

- **No-broadcast design point.** corvus takes same-length spans only.
  How libstats calls it where one argument is a scalar parameter blocks
  task 2.
- **Windows wheel.** Raise the job timeout and accept the AVX2 cap under
  MSVC, or move the wheel to clang-cl. Blocks task 5 only.

## Suggested session order

1. M1: P1 to P4 in one session.
2. Zen 4: P5.
3. Task 1 on each machine, then task 2.
4. Task 7 alongside task 2, on whichever machine is free.

Side tasks A, E, F, H, J and K fit any gap.
