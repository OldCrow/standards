# Corvus Adoption Workplan (libstats, libhmm, corvus, pylibstats, pylibhmm, ewcalc)

Status: **ACTIVE — written 2026-09-28**, at the return from the
2026-09-17 → 2026-09-28 travel period. This is the one in-progress record
in `records/`; it becomes historical when libstats v3.0.0 (corvus adoption,
formerly v2.5.0) and the libhmm spike verdict have shipped.

**Hold lifted 2026-10-25.** The 2026-10-22 hold (adversarial review of
`a8525a7`, rows 12d–12l) ended when R0 re-ran on Z: freeze head `0508ef1`.
R1 (M1) and R2 (Kaby Lake) are DONE 2026-10-25; R3 (PR, tag, release) on Z is next.

**Next up: the corvus v1.1.0 release tail.** On Z, go straight to R3 in the
section "corvus v1.1.0 release tail — ACTIVE" below. It is self-contained: preflight, exact commands, expected
results, how to record them, and what to do if anything differs.

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
  issue, milestone #6) gates the release. Decision [user]: corvus
  throughput first (corvus #42, #43, #31; #37 for x86), then task 3 once
  against the final kernel costs. Consumers still pin v1.0.1.
- **corvus splits correctness from performance (2026-10-20) [user].**
  corvus v1.1.0 ships the correctness block only: order rows 2–12, then
  native M1 and Kaby Lake validation, PR to `main` and the tag. The
  throughput work (rows 13–17: #42 lever 1, #45, #31, #22) moves to the
  new corvus milestone v1.2.0, branched off `main` after v1.1.0. So
  **the consumer pin bump, and libstats' release gate (libstats #156,
  retitled), target corvus v1.2.0.** v1.1.0 is an optional
  accuracy-only bump.
- **Fleet coherence rule [user, 2026-10-20]: libhmm adopts the same
  corvus release as libstats.** Its adoption spike (task 7, libhmm #106)
  is run, and its criteria are judged, against that release, which is now
  corvus v1.2.0, never an earlier pin.
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
  a new strict band [user]; 35/35 at AVX3_ZEN4/SSE4/SSE2; CI green).
  **Row 6 (#56 + #57) DONE 2026-10-19 on Z** (`7f0e18a`: dd series terms
  and a 4-step dd CF tail, which makes gamma R1/R2 correctly rounded or
  1 ulp, re-pinned; Temme tail z.lo back inside its bound, remainder in
  #64; INDICATIVE cost R1 +~40%, R2 +~3% [user accepted]; 35/35 on three
  tiers; CI green, native M1 leg owed). **Row 7 (#48) DONE 2026-10-19 on
  Z** (`07ca2f6`: bracketed Newton phase in gammainv [user's design
  choice], which takes small-a upper-tail failures from 1,648/6,000 to 0;
  388 certified rows; INDICATIVE +17% in two bench bands; 35/35 on three
  tiers; CI green). **Row 8 (#54) defect 1 DONE 2026-10-19 on Z**
  (`e376460`: shifted dd t = −(N + (s−1)/2)·log1p(−w) in the beta
  gamma-limit slice and the betainv transfer; 195 ₂F₁-certified rows,
  HEAD up to 11,592 ULP -> 0; gates re-pinned to 0/0; 35/35 on three
  tiers; CI green). **#65 closed 2026-10-19 as not a defect** (an oracle
  that lost precision in the probe, not the kernel). **Row 8 (#54) defect 2 DONE 2026-10-20 on Z**
  (`a82fd0e`: near-1 prefactor logs taken from the smaller of (ξ, 1−ξ)
  in beta and betainv; B_GL re-derived at the literal 2^-60 target,
  2^59 -> 2^47; 1,098 ₂F₁-certified rows in [2^40, 2^59), HEAD up to
  8 ULP -> within gates; 35/35 on three tiers; CI green). #54 closed.
  **Row 9 (#55, #50, #60) DONE 2026-10-20 on Z** (`0915989`: logit tie
  band on the betainv orientation probe and the #47 router mirror in
  BetaInvForward; a convergence-checked Newton tail; a certified
  backward-interval gate, 752/752; strict bands deleted; 35/35 on three
  tiers; CI green). #55, #50, #60 closed. **Row 10 (#52) DONE 2026-10-20
  on Z** (`806b330`: Bessel tail's last Horner step in dd; 8,730 oracle
  rows, HEAD at 2 ULP -> 1; CI green). **Row 11 (#61) DONE 2026-10-20 on
  Z** (`ae51623`: generator self-checks hardened; 3,001 erf subnormal
  rows; CI green). **Row 12 (#53) DONE 2026-10-20** (`825cc60`: drift,
  harness gaps, beta exact ties). Its re-score found contract excesses,
  added as rows 12a-12c [user]: **12a (#67) DONE 2026-10-21** (`cefaf95`:
  gamma_q R4 4 -> 1 ULP), **12b (#64) DONE 2026-10-21** (`d0fb2e1`: dd erfc
  tail G, gamma R3 3 -> 1), **12c (#69) DONE 2026-10-22** (`87ef42d`: beta's
  copy of gamma's Temme tail brought in step, beta R3 4 -> 1; the twin-drift
  rule now in corvus NUMERICAL-DOCTRINE.md). All CI green. **The v1.1.0
  correctness rows (2-12c) are complete**; next is the pause for native M1
  and Kaby Lake runs, then PR dev/v1.1.0 -> main, tag, release.
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

## corvus v1.1.0 release tail — ACTIVE (written 2026-10-22 on Z, refreshed 2026-10-25)

**Read this first on M1 or Kaby Lake.** corvus v1.1.0 (milestone 5,
"Correctness hardening") is code-complete: order rows 2–12l are done on
`dev/v1.1.0` (12-T and 12d–12k came from the 2026-10-22 adversarial
review and the hunt campaign; 12j is the contract text; 12l (#90), from
the Z audit, restores the gamma-limit slice's published 1 ULP). The R0 row names the freeze head and its green CI run, and
every step below that cites "the R0 CI run" means that run. Nothing on
this list changes code or data. The job is to reproduce the gates natively
and record the results. The per-row detail lives in corvus `PLAN.md`
(Status).

Families whose kernels or reference sets changed in v1.1.0, so their native
numbers are new: gamma_p/q (R1/R2 #56, R3 tail #57/#64, R4 #67),
beta_p/q (#47, the gamma-limit slice #54, the R3 tail #69, exact-tie rows),
gamma_p_inv/gamma_q_inv (#48), beta_p_inv/beta_q_inv (#50/#55/#60,
including the certified backward-interval gate), i0/i1/i0e/i1e (#52),
erf (3,001 subnormal rows), and the dd specials (#58/#49, Log1pmxDd's
unified cell at 2^-67.77). Rows 12d–12k add: gamma_q's R3 core split
(#70) and R4 subnormal a (#71); gammainv deep-subnormal targets (#72);
erfinv subnormal input (#73); erfc's top subnormal binade (#74); the
Bessel series near |x| ≈ 7.4 (#75); beta's gamma-limit ridge (#84) and
R1-complement lgamma (#85). Row 12j (#76) changes gates and reference rows
only: lbeta 1 ULP / 2^-53 absolute, beta R2 complement 1, exp's near-tie
allowance, and witness rows in beta, lbeta, trigamma and gammainv. Row
12l (#90) is the same kind: beta's gamma-limit gates go to 1, with six
witness rows.

| # | Machine | Task | Done when |
|---|---|---|---|
| R0 | Z | ~~REOPENED 2026-10-22; rows 12d–12l, consistency check, fresh configure + ctest, CI green~~ **DONE 2026-10-25. Freeze head `0508ef1`; the R0 CI run is 40238527638** (green on all three jobs). Fresh clang-cl configure, 0-warning build, 36/36 at AVX3_ZEN4, negative check fails, version check OK | First pass `a8525a7` (CI run 39668576528) is superseded by the review rows |
| R1 | M1 | ~~Native NEON validation (steps below)~~ **DONE 2026-10-25 at `0508ef1`** (HEAD `66bf5fa`; macOS 27.0.1, AppleClang 21.0.0, Highway 1.4.0): 0 warnings, 36/36 tier-asserted NEON, negative check fails 34/36; ULP report identical to the R0 CI NEON job (147 max-ULP lines, every not-CR count and worst input); route census 4 unexercised, 0 mismatches, 5 argued unreachable, as on Z. Recorded in corvus `a317ee3`; detail: corvus `PLAN.md` | 36/36 under `CORVUS_EXPECT_TARGET=NEON`; ULP report equals the R0 CI run's NEON job cell for cell; recorded |
| R2 | Kaby Lake | ~~AVX2 native + SSE4/SSSE3/SSE2 caps (steps below); optional GCC diagnostics leg~~ **DONE 2026-10-25 at `0508ef1`** (HEAD `b870466`): 36/36 on all four tiers, tier-asserted, 0 warnings, negative check fails; AVX2 ULP report equals the R0 CI AVX2 job; no-FMA tiers bit-identical; GCC 16.2.0 leg 0 warnings, 36/36. Detail: corvus `PLAN.md` | 36/36 on each of four tiers, tier-asserted; recorded |
| R3 | Z | RELEASING.md: PR `dev/v1.1.0` → `main` (conflict-free: main is merged in), CI green on main, signed tag `v1.1.0`, GitHub release, close milestone 5 | Tag pushed; release published |
| R4 | Z | Branch v1.2.0 off `main` after the tag (rows 13–17, milestone 7). libstats and libhmm adopt corvus v1.2.0, not v1.1.0 (fleet coherence rule) | `dev/v1.2.0` exists |

**Common preflight (M1 and K).**
1. `git fetch` and `git switch dev/v1.1.0 && git pull --ff-only` in
   corvus; pull standards too. Confirm HEAD equals `origin/dev/v1.1.0` and
   contains the freeze head: `git merge-base --is-ancestor 0508ef1 HEAD`.
   Then confirm nothing but docs changed after it:
   `git diff --stat 0508ef1 HEAD -- src include tests tools cmake CMakeLists.txt CMakePresets.json .github`
   must print nothing. Stop if either fails, or if R0 is still open.
   Record HEAD's hash; the code measured is `0508ef1`'s.
2. Record the toolchain: macOS version, `clang --version` (AppleClang),
   CMake, Ninja, `brew info highway` version.
3. Use a fresh build directory (`build-<machine>-v110`; K uses one per
   tier, below). Run no generator: nothing is regenerated.
4. Python 3 (standard library only) must be on PATH when you configure.
   ctest's `claims_check` test runs it to read every documented bound
   against the gates. Without it the configure skips that test silently,
   and ctest reports 35, not 36. `python3 --version` goes in the record.

**CI reference (the R0 run).** Run 40238527638 on `0508ef1`, all three
jobs green. The macOS job is 121305146149: its runner reported
`FEAT_BF16: 0` and native NEON, the M1's class. The Linux job is
121305146417, and its "ULP report" step runs at AVX2. Save each job's
ULP-report step first, with `gh` authenticated. GitHub keeps run logs
about 90 days, so until roughly 2027-01-23. The `awk` keeps that step's
lines and strips the timestamp, so the gate lines read as they do
locally; the echoed commands (ANSI-coloured) have no local counterpart.
```sh
gh run view 40238527638 --log --job 121305146149 | awk -F'\t' '$2=="ULP report"{sub(/^[^ ]+ /,"",$3); print $3}' > ci-ulp-neon.log
gh run view 40238527638 --log --job 121305146417 | awk -F'\t' '$2=="ULP report"{sub(/^[^ ]+ /,"",$3); print $3}' > ci-ulp-avx2.log
```
Each holds 147 "max ULP" lines. gh 2.103.2 (2026-10-23) labels every
step `UNKNOWN STEP`, so this `awk` prints nothing; slice the step at its
`##[group]Run export CORVUS_EXPECT_TARGET=…` line instead (R1 did).

**R1, M1 (NEON native, AppleClang 21).**
1. `sysctl hw.optional.arm.FEAT_BF16` must print 0. The M1 has no BF16,
   so native dispatch is NEON; if it prints 1, stop.
2. `cmake -B build-m1-v110 -G Ninja -DCMAKE_BUILD_TYPE=Release -DCORVUS_WERROR=ON -DCMAKE_PREFIX_PATH="$(brew --prefix)"`,
   then `cmake --build build-m1-v110`. Expect 0 warnings.
3. `CORVUS_EXPECT_TARGET=NEON ctest --test-dir build-m1-v110 --output-on-failure`:
   36/36.
4. Negative check: `CORVUS_EXPECT_TARGET=AVX2 ctest --test-dir build-m1-v110`
   must FAIL, which proves the tier assertion is load-bearing.
5. ULP report: run the gate commands of `.github/workflows/ci.yml`'s macOS
   "ULP report" step verbatim, with `./build/` replaced by
   `./build-m1-v110/`, and tee the output to `m1-ulp-neon.log`. Compare
   it cell for cell with `ci-ulp-neon.log` (CI reference, above). Every max-ULP
   cell, not-CR count and worst input should match: same silicon class,
   same compiler family.
6. Spot-check that the log contains the v1.1.0 results: gamma R3 P/Q
   direct 1, R4 Q direct 1, R4 P complement 1; beta R3 direct 1; Bessel
   tails 1; Log1pmxDd |u| >= 2^-40 at 2^-67.78 or better; betainv
   certified backward intervals 0 outside (p 371 rows, q 381); erf 19,439
   points, max 1; lbeta max 1 ULP; beta gamma-limit slice max 1 ULP (#90); exp's report
   prints a near-ties count. `grep -l deferred: tests/data/*_reference.txt`
   prints nothing: v1.1.0 ships no deferred rows.
7. Optional, a diagnostic and not a gate: the route census. Configure a
   second directory with `-DCORVUS_ROUTE_CENSUS=ON`, build its
   `hunt_probe` target, and run
   `python3 tools/hunt/route_census.py --build <dir>` (needs mpmath through
   `annotate.py`; use a venv that has it). Expect
   0 mismatches. Z reads 4 unexercised and 5 argued unreachable, and the
   script exits 1 while any route is unexercised. Record the summary line;
   a different unexercised count is a finding to record, not a failure.

**R2, Kaby Lake (AVX2+FMA native, AppleClang 15).**
1. Run the per-tier caps of corvus `docs/ENVIRONMENT.md` ("Per-tier
   validation recipe"), but with a fresh directory per tier and a stop on
   the first failure. The recipe's own loop reuses `build-cap` and carries
   on after a failed tier:
   ```sh
   BASE="HWY_AVX10_2|HWY_AVX3_SPR|HWY_AVX3_ZEN4|HWY_AVX3_DL|HWY_AVX3"
   for TIER in AVX2 SSE4 SSSE3 SSE2; do
     case $TIER in
       AVX2)  CAP="$BASE" ;;
       SSE4)  CAP="$BASE|HWY_AVX2" ;;
       SSSE3) CAP="$BASE|HWY_AVX2|HWY_SSE4" ;;
       SSE2)  CAP="$BASE|HWY_AVX2|HWY_SSE4|HWY_SSSE3" ;;
     esac
     B="build-k-v110-$TIER"
     cmake -B "$B" -G Ninja -DCMAKE_BUILD_TYPE=Release -DCORVUS_WERROR=ON \
       -DCORVUS_DISABLED_TARGETS="$CAP" || break
     cmake --build "$B" || break
     CORVUS_EXPECT_TARGET="$TIER" ctest --test-dir "$B" --output-on-failure || break
   done
   ```
   Each tier gets 36/36 under its own `CORVUS_EXPECT_TARGET`. If the loop
   stops early, the tiers after the failed one did not run.
2. ULP report, per tier: ctest prints nothing for passing gates, so run
   the gate commands of `.github/workflows/ci.yml`'s Linux "ULP report"
   step verbatim, under `CORVUS_EXPECT_TARGET=<tier>`, with `./build/`
   replaced by `./build-k-v110-<tier>/`. Tee each to `k-ulp-<tier>.log`,
   and check each log's "active SIMD target" line.
3. Expect SSE4, SSSE3 and SSE2 to be bit-identical to each other (every
   max-ULP cell, not-CR count and worst input), and `k-ulp-AVX2.log` to
   match `ci-ulp-avx2.log` (CI reference, above) cell for cell. FMA and no-FMA tiers may
   differ within their bounds: trigamma, cos(1.0) and one betainv row are
   known cases (corvus ACCURACY.md, under the validation matrix).
4. Optional diagnostics leg (no validation numbers come from it): Homebrew
   GCC 16 with `-DCORVUS_DISABLED_TARGETS=HWY_AVX10_2 -DCORVUS_DEV_WARNINGS=ON -DCORVUS_WERROR=ON`:
   0 warnings, 36/36 under `CORVUS_EXPECT_TARGET=AVX2` (ENVIRONMENT.md,
   "macOS: Homebrew GCC must be capped").
5. Quiet benches are NOT part of this list. The PERFORMANCE.md re-run
   moved to v1.2.0, and the v1.1.0 cost numbers stay marked indicative.

**Recording (R1 and R2; pre-approved bookkeeping, signed commits).**
- corvus `PLAN.md`: a "v1.1.0 native validation" entry with the machine,
  date, toolchain versions, commit, per-tier result, and any deviation.
- corvus `docs/ACCURACY.md`: R1 updates the NEON column date for the
  changed families listed above. R2 adds a line to the matrix's † note (the
  Kaby Lake cross-machine reproduction at v1.1.0: four tiers, 36/36,
  tier-asserted).
- This table: strike R1/R2 and add the date and commit.

**If anything fails or differs.** That means a gate failure, a tier
mismatch against CI, a warning, or a cell that differs from CI's. Stop.
Edit no gate, kernel or reference file on M1 or K. Record the exact output
in corvus `PLAN.md` under an [OPEN] item, push, and hand it back for
diagnosis. Only a pure toolchain or environment problem may be fixed in
place and re-run; record what it was.

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
| 2b | corvus throughput (v1.2.0 since the 2026-10-20 split; was v1.1.0): incomplete gamma/beta family and inverses with a scalar entry point (#42), NEON elementary import from libstats' clean-room kernels (#43), lgamma (#31), x86 erf (#37); release + libstats pin bump | corvus milestone v1.1.0. **IN PROGRESS.** 2026-10-16 state: ~~#43~~ DONE 2026-10-12 (rescoped, see Facts); ~~#37~~ CLOSED 2026-10-16 (premise retired by 2a/2c: corvus erf stays on x86, 3.4× faster than v2.4.1 on Zen 4); the v1.1.0 correctness block (corvus order rows 2–12, five frontier designs) now precedes #42 lever 1 (row 13) and #31 (row 15) [user, 2026-10-10]. Earlier: (2026-09-30, Kaby Lake): #42 lever 2 landed on corvus `main` at `a78eddd` — the driver pads the masked tail with a live element (single calls now cost one vector: gamma_p 2.9×, lgamma 2.6× faster, bit-identical) plus eight scalar entry points; ctest 34/34 AVX2. Owed: #42 lever 1 (per-element kernel cost — the ≤ 250 ns single-call target lives there), #43, #31, #37; ~~NEON (M1) ctest of `a78eddd` before the release~~ (DONE both halves: AVX-512 2026-09-30 on Zen 4, 34/34 tier-asserted `AVX3_ZEN4`; NEON 2026-10-02 on M1, 34/34 tier-asserted `NEON` at `4534d30`); the libstats pin bump must replace the deducing `corvus_scalar` wrapper with the scalar entry points (overload sets); corvus `docs/PERFORMANCE.md` §4/§8.2 re-run at v1.1.0. Detail: corvus `PLAN.md` Status, corvus #42 | All three (quiet for the timing); ~~M1 owes the `a78eddd` ctest~~ done 2026-10-02 |
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
| D | corvus kernel work beyond 2b, all on milestone v1.2.0 ("Throughput & new exports"); #21 and #18 still gated | corvus #22, #21, #18 | Kaby Lake first, then all |
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
5. Task 7 alongside 2b, on whichever machine is free; under the fleet
   coherence rule it measures against the corvus release libstats
   adopts (v1.2.0), so its verdict is final only on that release.
6. **From 2026-10-16:** libstats v2.5.0 is out and corvus is the critical
   path again. The sequence is now (revised 2026-10-20 [user]):
   - the v1.1.0 correctness block (corvus order rows 2–12) on Z and K;
   - a pause for the native M1 and Kaby Lake validation;
   - the corvus v1.1.0 PR, tag and release;
   - corvus v1.2.0 (#42 lever 1 and the rest of rows 13–17) on a branch
     off `main`;
   - the libstats and libhmm pin bumps to v1.2.0 (task 2 resumes, task 7
     judged);
   - task 3 once.

   libhmm's 8a (v4.4.2 crash cluster) is the gap-filler on any machine
   and does not wait on task 7.

Side tasks A, E, F and K fit any gap (J and L are done). H's milestone
#8 bucketing half is done (2026-09-29); its GitHub reconcile half
remains.
