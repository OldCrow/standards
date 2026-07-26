# CI House Style — OldCrow Projects

Standard for GitHub Actions workflows in libhmm, pylibhmm, libstats,
pylibstats, ewcalc, and corvus. Every rule here was paid for by a real
failure; the incident is named so the rule can be re-argued against
evidence rather than restated as dogma.

Two constraints shape everything: hosted runners are small (Linux: 2 vCPU,
~7 GB) and their **hardware is not guaranteed** — a job runs on whatever
silicon it draws. Rules 3 and 4 both follow from the second.

## 1. Runner budget

- Minimize job count. Add a job only when it produces information no
  other job can.
- Prefer **Release-only + `<PROJ>_WERROR`** over Debug/Release matrix
  duplication. Warnings are the reason to run a second configuration;
  the second configuration is not.
- Sweep variants **inside one job** when the setup dominates: corvus's
  Linux job walks AVX2 → SSE4 → SSSE3 → SSE2 sequentially, rebuilding
  only corvus, because a matrix would rebuild Highway four times for the
  same information.
- macOS and Windows jobs must justify themselves by coverage they alone
  give: native arm64/NEON silicon, MSVC-only diagnostics, the
  multi-config generator path. Not tier or version breadth.
- Docs-only pushes skip CI:
  `paths-ignore: ['**.md', 'docs/**', 'LICENSE']` on `push` and
  `pull_request` in real CI workflows (never on release/tag/pages ones).
- Every workflow sets `concurrency: { group: ..., cancel-in-progress: true }`
  and `permissions: contents: read`; every job sets `timeout-minutes`.

## 2. Never `--parallel` unbounded

`cmake --build --parallel` with no argument means unlimited `make -j`,
which OOM-kills a 7 GB runner. Bound every build step to memory:

The fleet idiom, verbatim (libstats `ci.yml`, `avx512-testing.yml`):

```yaml
- name: Build
  run: |
    MEM_GB=$(awk '/MemAvailable/{printf "%d", $2/1048576}' /proc/meminfo 2>/dev/null || echo 4)
    CPU=$(nproc 2>/dev/null || echo 2)
    MEM_JOBS=$(( MEM_GB / 2 ))
    JOBS=$(( MEM_JOBS < CPU ? MEM_JOBS : CPU ))
    JOBS=$(( JOBS < 1 ? 1 : JOBS ))
    echo "Parallel build jobs: ${JOBS}"
    cmake --build build --parallel "${JOBS}"
```

Divide by 3 rather than 2 for sanitizer builds (ASan+UBSan roughly
doubles peak memory per TU). The `|| echo` fallbacks keep the block
portable to runners without `/proc/meminfo` or `nproc`; echoing the
chosen count is what lets a later OOM be diagnosed from the log alone.
macOS runners (14 GB) use the full logical CPU count.

**Recognize the signature**: exit 143, "runner received a shutdown
signal", mass `Terminated` messages, dying at 2–3 minutes. That reads as
a timeout or as infra preemption, and it was misdiagnosed as both for a
full session (libstats `8e12517`, then again in `avx512-testing.yml` at
`6a68200`, where the bug hid because the step usually skips). It is OOM.
The tell that it is *not* preemption: other jobs on the same commit that
already bound their parallelism run to completion.

## 3. Never `-march=native` on a hosted runner

Use the project's portable/baseline mode instead
(`-DLIBHMM_PORTABLE=ON`). Runtime-dispatch TUs keep their per-ISA
targeted flags, so SIMD coverage is unchanged — only the
whatever-this-host-happens-to-be flag goes away.

The incident (libhmm `995ba60`): a runner advertised `avx10.1-256`;
current Clang promotes that to `avx10.1-512` with
`-Winvalid-feature-combination`, fatal under `-Werror`. **Do not fix this
class by suppressing the warning** — the promotion can emit 512-bit code
that a 256-max VM faults on, trading a red build for a SIGILL in tests.
The same reasoning bans CPU-probed `/arch:` selection under MSVC.

## 4. Assert what a job proves

A job that infers its own configuration from the host is not evidence.

- Cap the dispatched SIMD tier explicitly *and* assert the tier reached
  (corvus `CORVUS_DISABLED_TARGETS` + `CORVUS_EXPECT_TARGET`). Uncapped,
  a job silently tests whatever silicon it drew and skips the tier its
  name claims.
- Assert configure-time decisions the job depends on — corvus greps its
  configure log for `using system Highway` before trusting that the
  install rules were even enabled.
- Where a green run only re-exercises a fix if it draws particular
  hardware, say so in the workflow comment and carry it as a watch item.
  A green that skipped the step is not a green.

## 5. Actions hygiene

- **SHA-pin every action**, third-party and `actions/*` alike, with a
  `# vX.Y.Z` comment. Resolve via
  `gh api repos/OWNER/REPO/commits/TAG`, cross-check against
  `gh api repos/OWNER/REPO/tags`. Watch two shapes the naive pin regex
  misses: non-`vN` refs (`pypa/gh-action-pypi-publish@release/v1`) and
  subpath actions (`github/codeql-action/init`).
- `persist-credentials: false` on every checkout.
- `permissions: contents: read` at workflow level; elevate
  (`pages: write`, `id-token: write`, `attestations: write`) on the
  single job that needs it, never workflow-wide.
- Dependabot covers the `github-actions` ecosystem in every repo.
  **Group `github/codeql-action*`** — init and analyze must move
  together, and split bumps are individually un-mergeable by
  construction (each fails with a version-mismatch error).
- Merging a Dependabot PR that touches `.github/workflows/**` through
  `gh` needs the token's `workflow` scope; without it only already
  up-to-date PRs merge.

## 6. Linting the workflows

Every repo carries an identical `lint-workflows.yml`, triggered only on
`paths: ['.github/workflows/**']`: `reviewdog/action-actionlint`
(`fail_level: error`) plus a pinned `zizmor` run with
`--min-severity medium`. The severity floor is deliberate — zizmor's CLI
exits nonzero on *any* finding including informational ones, and the
actual bar is zero medium/high.

Two traps, both cost real time:

- **Two different check-runs are named `actionlint`.** The Actions *job*
  passes or fails on reviewdog's `-fail-level`; the reviewdog
  `github-check` reporter posts a *separate* check-run of the same name
  governed by `-level`, which goes red on findings in changed files. A
  repo's native CodeQL "Default setup" can post a third. Always trace
  job ID → run ID (or the check-run's `.app.name`) before attributing a
  failure.
- **Findings are latent until a PR touches the file.** `filter_mode: file`
  means only changed files report, so pre-existing findings in `ci.yml`
  stay invisible until a Dependabot PR edits it. "Green on main, red on
  the bump PR" is expected, not a regression. Fix at source so it does
  not matter which file the next PR touches.

Two shellcheck findings are known false positives — fix the *expression*,
never the logic: SC2193 on `[[ "${{ matrix.x }}" == gcc* ]]` (actionlint
substitutes a placeholder before shellcheck sees it; route matrix values
through step `env:` and compare the shell variable — also the
injection-safe pattern), and deliberate word-splitting (ewcalc's keychain
list), which takes a targeted `# shellcheck disable=` plus a reason.

## 7. Caching

No caching by default — it is complexity that hides staleness. Add it
when a step's cost actually grows, and say so in a comment. The one
adopted case is corvus building pinned Highway 1.4.0 from source (~2–4
min/run): key = pin + `runner.os` + `runner.arch`, **exact match only, no
`restore-keys`**, so a pin bump always rebuilds rather than silently
resurrecting the old prefix.

## 8. What CI must exercise

- The **installed** path, not just the build tree: install to a prefix,
  configure and build `consumer_example/` against it via `find_package`,
  run the resulting binary, then `pkg-config --exists --print-errors
  <proj>`. One leg (Linux/Release) is enough.
- Sanitizers (ASan+UBSan together, one Debug job) — not separate runners.
  Turn tools and examples off in that job; only `ctest` output counts, so
  instrumenting them buys nothing but compile time.
- A **monthly canary**: `schedule:` cron, staggered across repos, so
  runner-image and toolchain drift surfaces without waiting for a push.
  Publish and packaging steps stay gated on tag refs, so scheduled runs
  build and test only. Check job-level `if:` conditions actually admit
  `github.event_name == 'schedule'` — otherwise the canary silently
  no-ops.
- Release and publish jobs trigger on tag push or `workflow_dispatch`
  only, never on PR. `paths-ignore` is not evaluated for a new tag, so it
  cannot skip a release.

## 9. Verification discipline

- The first real CI run is the gate. Local `actionlint`/`zizmor` do not
  reproduce reviewdog's severity mapping or zizmor's exit behavior, and
  YAML validity proves nothing about a job.
- A local AppleClang build says nothing about a GCC or Linux-Clang flag
  set. GCC reports every diagnostic in a TU but stops the build at the
  first failing TU, so strict-warning fixes arrive one wave per CI round;
  budget for that, or reproduce locally with bare
  `g++-NN -fsyntax-only` over the flag list from `flags.make`.
- Enabling a strict configuration that has never run to completion is a
  code change, not a CI change. Expect it to fail the first time.
