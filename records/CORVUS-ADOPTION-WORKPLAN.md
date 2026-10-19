# Corvus Adoption Workplan (libstats, libhmm, corvus, pylibstats, pylibhmm, ewcalc)

Status: **ACTIVE — written 2026-09-28**, at the return from the
2026-09-17 → 2026-09-28 travel period. This is the one in-progress record
in `records/`; it becomes historical when libstats v3.0.0 (corvus adoption,
formerly v2.5.0) and the libhmm spike verdict have shipped.

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
- **libstats branch state 2026-10-03** [Zen 4 session]:
  `dev/v2.5.0-corvus` at `a036ffb` — the libstats-side levers (task 2c)
  and clang-cl as a supported Windows workflow have landed there; CI
  (dispatched) 12/12 on `003c3ad`. `dev/v2.4.2` is active in a separate
  session and worktree, at `6245e4e` when last read: it carries the
  clang-cl port (`420eaf1` cherry-picked with `-x`) and its own backport
  of the discrete-quantile search (libstats #170, #171, filed from the
  2c work). After v2.4.2 ships, `dev/v2.5.0-corvus` rebases onto `main`;
  ten files will need a decision each (guide in libstats `PLAN.md`,
  "Rebase onto `main` after v2.4.2 ships").
- **libstats release renumbering 2026-10-04** [user; Kaby Lake session]:
  `dev/v2.4.2` ships as **libstats v2.5.0** (correctness fixes, 0
  contract violations, re-derived dispatch tables, clang-cl), and
  corvus adoption (`dev/v2.5.0-corvus`, milestone #6) becomes
  **v3.0.0**: its installed package adds
  `find_dependency(corvus 1.0 CONFIG)`, which a `SameMajorVersion`
  consumer of 2.x would accept and then fail on. New Distributions
  becomes v3.1.0; the Architecture Refactor is post-v3. Branch names
  stay, so `v2.5.0` in branch names and in entries above this one
  means the corvus line. Detail: libstats `PLAN.md`, Decided, "Release
  numbering". GitHub milestone retitles are owed.
- **libstats v2.5.0 RELEASED 2026-10-16** [Kaby Lake session, same
  day]: PR #278 (`dev/v2.4.2` → `main`, 158 commits) closed milestone
  #10's 75 issues; signed tag `v2.5.0`; `dev/v2.4.2` deleted. pylibstats
  0.8.0 released the same day on that pin (OldCrow/pylibstats#31); its
  #20 and #26 are renumbered to the v3.0.0 corvus pin. libstats code is
  frozen until corvus is integrated (v3.0.0). `dev/v2.5.0-corvus` is
  rebased onto the released `main` (`72f9195`, carrying the merge note's
  final state); a checkout elsewhere needs `git fetch` and `git reset
  --hard origin/dev/v2.5.0-corvus`.
- **corvus paused 2026-10-10 → resumed 2026-10-16** [user; Kaby Lake]:
  after the v1.1.0 pre-check (quality sweep, defect hunt, issue order
  rows 0–17 in corvus `PLAN.md`) corvus waited for the libstats release.
  #43 was the exception and is DONE 2026-10-12 (rescoped: no libstats
  import, corvus exp/log stay correctly rounded via fast paths; re-gated
  on all three machines). #62 DONE 2026-10-16. The re-read of libstats
  `PLAN.md` after the release changed nothing in the corvus order. Next
  corvus pickup: order row 2, #53 guard harness (K). **Row 2 DONE
  2026-10-18 on Z** (strict known-defect bands, one per open issue; CI
  green, a native M1 run owed). **Row 3 (#59) DONE 2026-10-18 on Z**
  (`3462e6a`: DdDivDd floor fixed with fail-first both-subnormal rows,
  35/35 at AVX3_ZEN4/SSE4/SSE2; DdSqrt floor documented, prescale
  deferred [user]); CI green. **Row 4 (#58 + #49) DONE 2026-10-19 on Z**
  (`c42f7e3`: gammainv's 2-arg Log1pmxDd and the #49 band deleted;
  Expm1Dd series to 2^-6 with new dd gates; 35/35 at AVX3_ZEN4/SSE4/SSE2;
  CI owed). **Row 5 (#47 + #51) DONE 2026-10-19 on Z** (`50961b3`: R4
  window is R2's own predicate, 1,414 certified seam rows; gamma_q sign
  clamp; #47 and #51 bands deleted; #47's inverse half moved to #55 under
  a new strict band [user]; 35/35 at AVX3_ZEN4/SSE4/SSE2; CI owed). Next:
  row 6, #56 gamma R1/R2 ridge + #57 Temme z.lo (frontier; K, M1, Z).
- **libhmm defect hunt 2026-10-12** [Kaby Lake]: 27 issues (#112–#138)
  filed and split across three milestones, and the release order revised
  [user]: v4.4.2 Safety & contract patch (#7, crash cluster, fail-first
  guards) → Fit accuracy & kernel hygiene patch (#5, now
  corvus-INDEPENDENT) → [spike (#106) → Post-corvus kernel patch (#8)] →
  v4.5.0. Task 8 below is split accordingly; tasks 7 and 9 stand.
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
| 2b | corvus v1.1.0 throughput: incomplete gamma/beta family and inverses with a scalar entry point (#42), NEON elementary import from libstats' clean-room kernels (#43), lgamma (#31), x86 erf (#37); release + libstats pin bump | corvus milestone v1.1.0. **IN PROGRESS.** 2026-10-16 state: ~~#43~~ DONE 2026-10-12 (rescoped, see Facts); ~~#37~~ CLOSED 2026-10-16 (premise retired by 2a/2c: corvus erf stays on x86, 3.4× faster than v2.4.1 on Zen 4); the v1.1.0 correctness block (corvus order rows 2–12, five frontier designs) now precedes #42 lever 1 (row 13) and #31 (row 15) [user, 2026-10-10]. Earlier: (2026-09-30, Kaby Lake): #42 lever 2 landed on corvus `main` at `a78eddd` — the driver pads the masked tail with a live element (single calls now cost one vector: gamma_p 2.9×, lgamma 2.6× faster, bit-identical) plus eight scalar entry points; ctest 34/34 AVX2. Owed: #42 lever 1 (per-element kernel cost — the ≤ 250 ns single-call target lives there), #43, #31, #37; ~~NEON (M1) ctest of `a78eddd` before the release~~ (DONE both halves: AVX-512 2026-09-30 on Zen 4, 34/34 tier-asserted `AVX3_ZEN4`; NEON 2026-10-02 on M1, 34/34 tier-asserted `NEON` at `4534d30`); the libstats pin bump must replace the deducing `corvus_scalar` wrapper with the scalar entry points (overload sets); corvus `docs/PERFORMANCE.md` §4/§8.2 re-run at v1.1.0. Detail: corvus `PLAN.md` Status, corvus #42 | All three (quiet for the timing); ~~M1 owes the `a78eddd` ctest~~ done 2026-10-02 |
| 2c | libstats-side levers from the 2a analysis, on `dev/v2.5.0-corvus`, independent of corvus: (1) revert the elementary-family swap except erf on x86; (2) discrete-quantile search from a normal guess, Student-t quantile with one inverse call off the tail; (3) Windows on clang-cl | libstats #156. **DONE on Zen 4 2026-10-03** (`e62a616`, `45000c6`, `c51e2ed`; clang-cl `420eaf1`, `402f624`, `003c3ad`; AVX-512 block regenerated at `a036ffb`): ctest 74/74, timing 22/22, sweep 35 contract violations, unchanged. INDICATIVE Zen 4 numbers: exp/log/cos/sin back at v2.4.1 cost and erf 3.4× faster than v2.4.1; batch pdf/logpdf for gamma, beta, Student-t back at parity; quantile binomial 145 → 8.3 µs, poisson 9.2 → 2.5 µs, Student-t 43.7 → 27.7 µs. libstats builds with clang-cl (presets `windows-clang-cl`, `-strict`; `-T ClangCL`; a CI leg, 5 min vs MSVC 18) and matches the cl.exe + clang-cl-corvus speed. What stays corvus-owned (2b, #42): scalar CDF 7–17×, batch CDF beta/binomial/Student-t 3–8×, quantile gamma family 10–17×, Student-t 39×. Consequences: corvus #43 is now a deduplication, not a libstats throughput dependency; libstats #107/#108/#110 are open again in substance. Owed: Kaby Lake and M1 legs (ctest, sweep with block regeneration, bench — the M1's NEON kernels are restored untested), a quiet warmed record pass on Zen 4. Detail: libstats `PLAN.md` Status | Zen 4 done; Kaby Lake and M1 owe their legs; all three quiet for the record |
| 3 | Post-swap sweep, validation matrix, same-machine x86 erf timing (after 2b — runs once, against the final kernel costs); correct the unmeasured `~5×` erf comment in `dispatch_thresholds.h` from the result | libstats `PLAN.md` Next Steps 3(a)–(b); capped tiers on Kaby Lake | All three |
| 4 | libstats v3.0.0 release (corvus; formerly v2.5.0) | Milestone #6 close | Any, with the signing key |
| 5 | pylibstats corvus pin bump (was "v0.8.0": 0.8.0 shipped 2026-10-16 on libstats v2.5.0 instead): LICENSE and NOTICE, Windows wheel job (moves to clang-cl with the pin — pylibstats #26) | pylibstats #20, #26, renumbered to the libstats v3.0.0 pin; ~~needs side task J done first~~ J done | Any, plus CI |
| 6 | libstats post-adoption patch | libstats milestone #8, bucketed 2026-09-29: #103, #104, #111, #144 wait for task 3's sweep and timing (#144 needs the first post-swap Zen 4 session; moot if #111 lands NEVER); #114 one pass after the swap. #146 is a task 3 prerequisite (below) | All three |

Task 1 precedes task 2 for a reason beyond the owed confirmation: on the
M1 it separates what macOS 27 changed from what corvus changes. Without
it, the post-swap NEON differences confound the two.

## Parallel track (libhmm)

| # | Task | Issues / milestones | Machines |
|---|---|---|---|
| 7 | Spike fleet half: branch, measurements, apply the pre-registered criteria | libhmm #106 (runbook on the issue) | All three, quiet |
| 8a | v4.4.2 Safety & contract patch: the 2026-10-12 hunt's crash cluster, each fix with a fail-first guard; no number-changing fit fixes. **Next libhmm work, independent of task 7** | libhmm milestone #7 (#112–#125, #139) | Any; nine CI legs |
| 8b | Fit accuracy & kernel hygiene patch, corvus-independent (re-scoped 2026-10-12) | libhmm milestone #5 (#92, #93, #100, #108 + hunt additions) | All three for the per-tier accuracy tests |
| 8c | Post-corvus kernel patch, trimmed or not by task 7's verdict | libhmm milestone #8 (#94, #99, #101, #103, #127, #128, #130); #94 needs its benchmark pass before the trim | All three |
| 9 | pylibhmm pin bump (one per shipped libhmm patch) | Follows 8a, then 8b/8c; ~~needs side task J done first~~ J done | Any, plus CI |

## Side tasks (no dependency on the spine)

| # | Task | Issues | Machines |
|---|---|---|---|
| A | Fix the libstats coverage measurement, then port to libhmm | libstats #152, libhmm #108 | CI only |
| ~~B~~ | ~~Uniform speedup gate flake~~ | libstats #129 — DONE 2026-10-02 on Zen 4, PR #169 merged to `main` (`d8d3388`), with #168: every speedup gate in the timing-label tests timed its first call, and two paths timed in sequence could straddle the Zen 4 frequency step; all 17 gates now measure steady state with the paths interleaved. Timing suite 22/22 ×10 on Zen 4; Kaby Lake CONFIRMED 2026-10-02 (22/22 ×4, three quiet, one under heavy load, Dev build native AVX2); M1 CONFIRMED 2026-10-02 (22/22 ×10, NEON, under ambient load 4–10) | ~~Zen 4~~; ~~Kaby Lake~~; ~~M1~~ |
| ~~L~~ | ~~Promote the sustained-crossover method into `threshold_validator` — before task 3~~ | libstats #146 — DONE, closed 2026-10-05: the method landed as `scripts/analyze_crossovers.py` (`3e9fde3`) with `scripts/derive_threshold_rows.py` on top, used by the v2.5.0 table re-derivation; task 3 uses those scripts | — |
| C | M1 quiet-bench retry on macOS 27 | corvus `docs/PERFORMANCE.md` M1 rows are INDICATIVE. Attempted 2026-10-02 (10% gate, 30 min, locked, Backblaze paused, TM skipped): ABORT, min 11.88%, `mediaanalysisd` on idle in 134/173 samples — corvus `docs/bench-evidence/2026-10-02-m1-gate-abort/`. Next attempt holds off Photos analysis and automatic Time Machine first | M1 |
| D | corvus v1.1.0 kernel work beyond 2b | corvus #22, #21, #18 | Kaby Lake first, then all |
| K | Linux aarch64 validation gap: the aarch64 wheel will ship corvus NEON compiled by GCC, a compiler × ISA pair no fleet machine validates | libstats `PLAN.md` Cross-Repo Dependencies; decide the remedy before task 5 ships wheels | CI (aarch64 runner) |
| E | `load_hmm` wrapper decision | pylibhmm #31 | Any |
| F | `errorf_inv` exposure investigation | libhmm #103 | Any |
| G | Radar jamming calculator | ewcalc #88; needs the physical book pins | Windows and Linux UI |
| H | Plan hygiene | libstats GitHub reconcile (last done 2026-09-05); ~~milestone #8 bucketing pass~~ DONE 2026-09-29 | Any |
| ~~I~~ | ~~Merge the open dependabot PRs~~ | DONE 2026-09-28: all eight merged; `main` CI green in libstats, libhmm, corvus and ewcalc | — |
| ~~J~~ | ~~scikit-build-core 1.1.0 breaks free-threaded Python discovery in pylibstats and pylibhmm~~ | DONE 2026-10-04 in both repos: upstream defect (scikit-build/scikit-build-core#1597; 1.1.0 set `Python_FIND_ABI` on CMake < 4.1.2), fixed in 1.1.1. Bound is `>=0.10,!=1.1.0,<1.2` with `cmake.version = ">=4.1.2"` in both `pyproject.toml`; the `Python 3.14t` job gates any bound change. Detail: pylibhmm `PLAN.md` | — |

## Later milestones (not scheduled)

Listed so this record names every open milestone. None starts before the
spine and the libhmm track finish, and none has an order yet.

| Repo | Milestone | Depends on |
|---|---|---|
| libstats | v3.1.0 — New Distributions (#58–#62; formerly v2.6.0) | v3.0.0 (corvus) shipped; pylibstats bindings and stubs follow |
| libstats | Post-v3 — Architecture Refactor (#40–#43, #128; v3.x or v4, undecided) | After v3.1.0 |
| libhmm | v4.5.0 — Algorithm Coverage (#47, #48, #51, #52, #96, #97) | After task 8 |
| libhmm | v5.0.0 — API (#95, #98) | After v4.5.0 |
| libhmm | Unmilestoned features (#50 HSMM, #53 IOHMM) | No milestone assigned |
| corvus | v1.1.0 conditional and watch items (#19, #20 wait on libstats consumers; #28, #29 wait on upstream) | Side task D covers the rest of v1.1.0 |
| pylibhmm | Parity-ledger audit of pre-0.12.0 surfaces | Opportunistic; no issue filed |

## Open decisions [user]

- ~~**No-broadcast design point.**~~ DECIDED 2026-09-29: libstats fills
  the constant span per block inside its `vector_*` adapters; corvus
  unchanged. Task 2 is unblocked.
- ~~**Windows wheel.** Raise the job timeout and accept the AVX2 cap under
  MSVC, or move the wheel to clang-cl.~~ DECIDED 2026-10-03: clang-cl is
  a supported Windows workflow (libstats `dev/v2.5.0-corvus`: presets,
  `Strict`, `-T ClangCL`, a CI leg; `WINDOWS-TOOLCHAIN.md` §5). The
  wheel moves to clang-cl with the v2.5.0 pin — OldCrow/pylibstats#26.
  Still part of task 5.

## Suggested session order

1. ~~M1: P1 to P4 in one session.~~ DONE 2026-09-28.
2. ~~Zen 4: P5.~~ DONE 2026-09-28.
3. ~~Task 1 on each machine~~ (DONE 2026-09-28); task 2 M1 leg DONE
   2026-09-29 and parked.
4. ~~2a on the x86 machines~~ (DONE 2026-09-30, both), then 2b (corvus
   v1.1.0; IN PROGRESS — #42 lever 2 landed 2026-09-30, next pickup on
   any machine is #42 lever 1 or #43; ~~M1 ctest `a78eddd` first~~ — Zen 4 done 2026-09-30, M1 done 2026-10-02),
   then task 2 resumes and task 3 runs once. 2c (libstats-side levers)
   DONE on Zen 4 2026-10-03, alongside 2b: its Kaby Lake and M1 legs fit
   any session on those machines and should precede task 3, which
   measures against the kernels 2c restored.
5. Task 7 alongside 2b, on whichever machine is free.
6. **From 2026-10-16:** libstats v2.5.0 is out and corvus is the critical
   path again — the v1.1.0 correctness block (corvus order rows 2–12)
   on K, with the frontier rows' M1 and Zen 4 legs as they come up; then
   #42 lever 1, the v1.1.0 release and the libstats pin bump (task 2
   resumes), then task 3 once. libhmm's 8a (v4.4.2 crash cluster) is
   the gap-filler on any machine and does not wait on task 7.

Side tasks A, E, F and K fit any gap (J and L are done). H's milestone
#8 bucketing half is done (2026-09-29); its GitHub reconcile half
remains.
