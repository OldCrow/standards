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

- ~~**corvus is off the critical path.**~~ **On it again (2026-09-29).**
  v1.0.0 froze the surface and the libstats swap is correctness-complete
  on `dev/v2.5.0-corvus`, but its M1 throughput (special-function CDF
  12–24×, quantile 8–66× slower; libstats "perf: v2.5.0 throughput"
  issue, milestone #6) gates the release. Decision [user]: corvus v1.1.0
  throughput first (corvus #42, #43, #31; #37 for x86), then task 3 once
  against the final kernel costs. Consumers still pin v1.0.1.
- **The Mac Mini M1 moved from macOS Tahoe 26 to macOS 27 Golden Gate during the
  travel period** [user, 2026-09-28]. Every M1 validation and timing
  record before that date is a Tahoe record. Toolchain and Apple libm
  versions on the new OS are unmeasured until P1 runs.
- **libstats branch state 2026-10-02** [Zen 4 session]: `main` at
  `d8d3388` (PR #169, side task B). `dev/v2.4.2` CREATED from it, empty —
  the v2.4.2 correctness patch (libstats #157–#167, `main`-only fixes
  filed 2026-09-29/30, independent of 2b) lands there [user: branch
  after the merge so it starts from `main`]. `dev/v2.5.0-corvus` REBASED
  onto that `main` (`22c2250` → `0de7ebb`, 17 commits, no conflicts,
  force-pushed): a checkout of it on any other machine needs `git fetch`
  and `git reset --hard origin/dev/v2.5.0-corvus` before further work.
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
| ~~P1~~ | ~~M1~~ | ~~Pull all repos; check the toolchain after the OS upgrade (Command Line Tools, AppleClang version, Homebrew, Highway 1.4.0, Python venvs); wipe build directories~~ | DONE 2026-09-28: macOS 27.0.1; Xcode and CLT 27.0; AppleClang 21.0.0 (clang-2100.3.34.2); CMake 4.4.3; Ninja 1.13.2; Homebrew 7.0.7; Highway 1.4.0; Python 3.14.7 venvs (pylibstats, pylibhmm). Pre-upgrade `build*/` dirs wiped (Tahoe result logs archived to `~/Archive/` first); `build-m1-gg/` per repo is the fresh build. Stale `SDKROOT` (CLT SDK vs `xcode-select` Xcode) resolved by pulling chezmoi `20ca30d` and applying; today's M1 builds used the CLT 27.0 SDK, same version as Xcode's |
| ~~P2~~ | ~~M1~~ | ~~Re-run native correctness: libstats v2.4.1 ctest, libhmm v4.4.1 ctest with the NEON ULP gates, corvus v1.0.1 tier-asserted NEON~~ | DONE 2026-09-28, all warning-clean: libstats 74/74 (NEON); libhmm 51/51, ULP gates max 1 ULP, means unchanged from Tahoe; corvus 34/34 under `CORVUS_EXPECT_TARGET=NEON` |
| ~~P3~~ | ~~M1~~ | ~~Reconcile the three stale libstats branches and the loose NEON patch~~ | DONE 2026-09-28: all three superseded and deleted (SHAs in libstats `PLAN.md` Next Steps 5); patch moved to `~/Archive/`. The M1 sweep block is lifted |
| ~~P4~~ | ~~M1~~ | ~~Update the fleet tables from Tahoe to macOS 27 Golden Gate, using P1's measured versions~~ | DONE 2026-09-28: libstats `AGENTS.md`, corvus `docs/ENVIRONMENT.md`, libhmm `PLAN.md` Local Machine State |
| ~~P5~~ | ~~Zen 4~~ | ~~Pull all repos; fresh builds; native smoke of libstats v2.4.1 and corvus v1.0.1, including the corvus toolchain guard under clang-cl, MSVC and mingw~~ | DONE 2026-09-28: libstats ctest 74/74 (MSVC, AVX-512); corvus ctest 34/34 tier-asserted AVX3_ZEN4 (clang-cl); guard verdicts as designed under all three compilers |
| P6 | Kaby Lake | None | The travel work was done on this machine |

## Main spine (libstats)

| # | Task | Issues / milestones | Machines |
|---|---|---|---|
| ~~1~~ | ~~Pre-swap baseline: regenerate the characterization sweep at v2.4.1~~ | DONE 2026-09-28 on all three: Zen 4 (AVX-512, 34 → 32); Kaby Lake (AVX2, 34 → 32, same rows); M1 (NEON, 32 → 32, geometric logpdf max_rel 0.865 → 2.9e-11). The M1 leg shows one von Mises `cdf` row moved with macOS 27, not with code: v2.4.0's commit rebuilt on macOS 27 differs from v2.4.1 only in the two #125 rows. The plan's 63 → 61 was the older 6063-row grid | ~~Zen 4~~, ~~Kaby Lake~~, ~~M1~~ |
| 2 | v2.5.0 core swap on a dev branch, corvus pinned at v1.0.1 | libstats milestone #6. **PARKED, correctness-complete** (2026-09-29): `dev/v2.5.0-corvus`, five increments on the M1 — engine + adapters, inverses + hot loops, Bessel, the elementary family, notices/CI/docs; ctest 74/74 and a NEON sweep at each step; accuracy delivered (libstats `PLAN.md` Next Steps (c)). Resumes after 2b; CI green on the branch at `01ade5c` (2026-09-29). Still owed then: ~~Kaby Lake and Zen 4 legs~~ (both done; re-run at the pin bump). Kaby Lake leg DONE 2026-09-30: `6b78cd5` native AVX2, correctness 74/74, sweep 32 → 35 (the M1's rows), AVX2 block regenerated; two timing-label tests regress vs v2.4.1 on that machine (libstats `PLAN.md` Next Steps 6(c)). Zen 4 leg DONE 2026-09-30 at `ead51e0`: MSVC `/arch:AVX512`, ctest 74/74 against both a clang-cl corvus (`AVX3_ZEN4`) and an MSVC corvus (`AVX2` cap), AVX-512 sweep 32 → 35 (the same rows), the two corvus builds bit-identical over all 9798 rows, AVX-512 block regenerated; four exp/log SIMD-speedup timing gates fail on the branch there | M1 ~~done~~; ~~Kaby Lake~~ done; ~~Zen 4~~ done |
| ~~2a~~ | ~~Fleet throughput comparatives of the branch: run `tools/bench/` (v2.4.1 vs branch, elementary family, corvus per-call scaling) on the x86 machines — decides the elementary-family question per ISA and gives corvus its fleet targets~~ | libstats perf issue (milestone #6). DONE: Zen 4 2026-09-30 (`docs/bench-evidence/2026-09-30-zen4-quiet-warm/`), Kaby Lake 2026-09-30 (`docs/bench-evidence/2026-09-30-kaby-quiet/`, quiet, one pass). Same verdict on both ISAs: keep corvus erf on x86, exp/log are the loss (3–6×); scalar CDF 7–20× and quantiles 7–75× are the per-call price. corvus AVX2 targets per element: erf 6.5, exp 8.0, lgamma 58, gamma_p 152, beta_p 1,024, gamma_p_inv 1,886 ns | ~~Kaby Lake~~, ~~Zen 4~~ |
| 2b | corvus v1.1.0 throughput: incomplete gamma/beta family and inverses with a scalar entry point (#42), NEON elementary import from libstats' clean-room kernels (#43), lgamma (#31), x86 erf (#37); release + libstats pin bump | corvus milestone v1.1.0. **IN PROGRESS** (2026-09-30, Kaby Lake): #42 lever 2 landed on corvus `main` at `a78eddd` — the driver pads the masked tail with a live element (single calls now cost one vector: gamma_p 2.9×, lgamma 2.6× faster, bit-identical) plus eight scalar entry points; ctest 34/34 AVX2. Owed: #42 lever 1 (per-element kernel cost — the ≤ 250 ns single-call target lives there), #43, #31, #37; NEON (M1) ctest of `a78eddd` before the release (AVX-512 DONE 2026-09-30 on Zen 4: 34/34 tier-asserted `AVX3_ZEN4`); the libstats pin bump must replace the deducing `corvus_scalar` wrapper with the scalar entry points (overload sets); corvus `docs/PERFORMANCE.md` §4/§8.2 re-run at v1.1.0. Detail: corvus `PLAN.md` Status, corvus #42 | All three (quiet for the timing); M1 owes the `a78eddd` ctest |
| 3 | Post-swap sweep, validation matrix, same-machine x86 erf timing (after 2b — runs once, against the final kernel costs); correct the unmeasured `~5×` erf comment in `dispatch_thresholds.h` from the result | libstats `PLAN.md` Next Steps 3(a)–(b); capped tiers on Kaby Lake | All three |
| 4 | libstats v2.5.0 release | Milestone #6 close | Any, with the signing key |
| 5 | pylibstats v0.8.0: pin bump, LICENSE and NOTICE, Windows wheel job | pylibstats #20; needs side task J done first | Any, plus CI |
| 6 | libstats post-adoption patch | libstats milestone #8, bucketed 2026-09-29: #103, #104, #111, #144 wait for task 3's sweep and timing (#144 needs the first post-swap Zen 4 session; moot if #111 lands NEVER); #114 one pass after the swap. #146 is a task 3 prerequisite (below) | All three |

Task 1 precedes task 2 for a reason beyond the owed confirmation: on the
M1 it separates what macOS 27 changed from what corvus changes. Without
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
| ~~B~~ | ~~Uniform speedup gate flake~~ | libstats #129 — DONE 2026-10-02 on Zen 4, PR #169 merged to `main` (`d8d3388`), with #168: every speedup gate in the timing-label tests timed its first call, and two paths timed in sequence could straddle the Zen 4 frequency step; all 17 gates now measure steady state with the paths interleaved. Timing suite 22/22 ×10 on Zen 4; Kaby Lake CONFIRMED 2026-10-02 (22/22 ×4, three quiet, one under heavy load, Dev build native AVX2); one `ctest -j1 -L timing` owed on the M1 (the label is outside CI) | ~~Zen 4~~; ~~Kaby Lake~~; M1 (confirmation) |
| L | Promote the sustained-crossover method into `threshold_validator` — **before task 3**, so the post-swap threshold re-measure uses the trusted tool; acceptance = reproducing the encoded kAvx2/kNeon rows from the two v2.4.0 bundles | libstats #146 | Any |
| C | M1 quiet-bench retry on macOS 27 | corvus `docs/PERFORMANCE.md` M1 rows are INDICATIVE | M1 |
| D | corvus v1.1.0 kernel work beyond 2b | corvus #22, #21, #18 | Kaby Lake first, then all |
| K | Linux aarch64 validation gap: the aarch64 wheel will ship corvus NEON compiled by GCC, a compiler × ISA pair no fleet machine validates | libstats `PLAN.md` Cross-Repo Dependencies; decide the remedy before task 5 ships wheels | CI (aarch64 runner) |
| E | `load_hmm` wrapper decision | pylibhmm #31 | Any |
| F | `errorf_inv` exposure investigation | libhmm #103 | Any |
| G | Radar jamming calculator | ewcalc #88; needs the physical book pins | Windows and Linux UI |
| H | Plan hygiene | libstats GitHub reconcile (last done 2026-09-05); ~~milestone #8 bucketing pass~~ DONE 2026-09-29 | Any |
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

- ~~**No-broadcast design point.**~~ DECIDED 2026-09-29: libstats fills
  the constant span per block inside its `vector_*` adapters; corvus
  unchanged. Task 2 is unblocked.
- **Windows wheel.** Raise the job timeout and accept the AVX2 cap under
  MSVC, or move the wheel to clang-cl. Blocks task 5 only.

## Suggested session order

1. ~~M1: P1 to P4 in one session.~~ DONE 2026-09-28.
2. ~~Zen 4: P5.~~ DONE 2026-09-28.
3. ~~Task 1 on each machine~~ (DONE 2026-09-28); task 2 M1 leg DONE
   2026-09-29 and parked.
4. ~~2a on the x86 machines~~ (DONE 2026-09-30, both), then 2b (corvus
   v1.1.0; IN PROGRESS — #42 lever 2 landed 2026-09-30, next pickup on
   any machine is #42 lever 1 or #43; M1 ctest `a78eddd` first — Zen 4 done 2026-09-30),
   then task 2 resumes and task 3 runs once.
5. Task 7 alongside 2b, on whichever machine is free.

Side tasks A, E, F, J and K fit any gap; L fits the gaps in task 2 and
must land before task 3. H's milestone #8 bucketing half is done
(2026-09-29); its GitHub reconcile half remains.
