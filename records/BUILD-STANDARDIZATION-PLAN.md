# Build-Stack Standardization Plan (libhmm, pylibhmm, libstats, pylibstats, ewcalc, corvus)

Status: **EFFORT COMPLETE — all four phases done, all six repos CI-green
(2026-07-23).** This file is now the historical record plus the
still-open follow-up ledger (see Open items at the end). Phase records
follow.

Moved into [OldCrow/standards](https://github.com/OldCrow/standards) on
2026-07-26; it and the house-style doc previously lived unversioned in the
`~/Development` root on one machine.

Phase 0 (2026-07-21). House-style doc written
([CMAKE-HOUSE-STYLE.md](../CMAKE-HOUSE-STYLE.md)). Defects 1–4 fixed, locally verified
(Sonnet subagent; libhmm consumer smoke test passes end-to-end), committed,
pushed, and CI-green on all three repos: libhmm 66a7568 (8 jobs green),
libstats 7a0ba8d + 6a68200 (all 6 workflows green), ewcalc 25d1114
(CI + CodeQL green). Two libhmm fixes landed beyond the plan (both blocked
the consumer test, both pre-existing): exported targets had no include path
(directory-scope include_directories is not captured by install(EXPORT))
and `detail/` headers — directly included by public headers — were excluded
from the header install.

Phase 0 addendum: a latent CI bug surfaced during verification — libstats
avx512-testing.yml still used bare `cmake --build --parallel` and
OOM-died (exit 143) when a run drew an AVX-512-capable runner. Fixed in
6a68200 with the standard mem/2 JOBS bound. Caveat: the fix is only truly
exercised when a future run again draws AVX-512 silicon (the build step
skips otherwise); pattern is identical to the proven ci.yml fix.

Phase 1 status (2026-07-22): **COMPLETE — all CI green** (libhmm 610cdf4,
libstats 412f973, corvus 3bbecf1+c158765). corvus's first CI run failed
usefully: distro libhwy-dev (Highway 1.0.x) predates hn::ReduceMax and
find_package(hwy) had no version floor — fixed with find_package(hwy 1.4)
(floor tied to the accuracy-audit pin; bump together) and a CI step that
builds pinned Highway 1.4.0 from source instead of apt, asserting
"using system Highway" in the configure log. Original execution record:
Commits: libhmm 610cdf4, libstats 412f973, corvus 3bbecf1 (Sonnet agent
executed; all locally verified — 47/47, 49/49, 4/4 tests; consumer
examples build AND run against scratch install prefixes; pkg-config
smoke passes everywhere). Notable finds during execution, all fixed:
- libhmm installed example/test/tool binaries (contract violation) —
  install rules removed from all three subdir CMakeLists.
- libstats never installed the generated libstats_version.h (installed
  find_package consumers couldn't compile) — now installed.
- libstats consumer examples used the removed pre-2.0 Result<T>
  aggregate API — updated to *result/.message().
- corvus exported hwy::hwy (PRIVATE static-lib dep via LINK_ONLY)
  verified to resolve cleanly in an installed consumer — no issue.
[OPEN] All three .pc files bake CMAKE_INSTALL_PREFIX at configure time;
`cmake --install --prefix <other>` leaves stale paths inside the .pc
(pkg-config still resolves; cosmetic for the CI flow). Revisit if
precise relocatable .pc paths ever matter.
[OPEN] pylibhmm/pylibstats CI does not yet exercise the parent library's
installed-package path — deferred (candidate for Phase 4 or next parent
release).

Next: Phase 2 (presets in all six repos).
Created 2026-07-21 after a full survey of all six build stacks. This is the
canonical cross-repo plan file; per-repo PLAN.md files get entries only when
execution touches that repo.

## Decided design parameters (user-approved 2026-07-21)

1. **Install contract** (libhmm, libstats, corvus): static+shared libs, public
   headers, CMake config/version files, pkg-config. GNUInstallDirs paths,
   namespaced exported targets, `SameMajorVersion` compatibility. Tools,
   examples, tests are never installed. Python projects stay wheel-only;
   ewcalc stays app-packaged via platform scripts (no `install()`).
   Loader registration: nothing automated — rpath/@rpath correctness plus a
   README note about `ldconfig` on Linux; CMake never runs it.
2. **Presets**: CMakePresets.json in all six repos, shared vocabulary
   (`release`, `debug`, `relwithdebinfo` + project extras such as
   `strict`/`sanitize`), setting build type/options only — **no pinned
   generator**. Ninja documented as preferred; CI may pass `-G Ninja`.
   Meson (and any non-CMake build system) explicitly out of scope.
3. **libstats depth**: incremental, behavior-preserving modularization.
   Keep Dev/Strict build types, object-library hierarchy, include shim.
   Split warning sets/build-type flags, test registration, tools into
   cmake/ modules and tests/ & tools/ subdirectory CMakeLists; convert
   global add_compile_options to target scope; dedupe TBB block.
4. **libhmm option names**: rename to `LIBHMM_*` with a one-release
   deprecation shim (old names honored with a warning). pylibhmm switches to
   new names in the same coordinated change set.

## House style (to be written as Phase 0 deliverable)

Derived from corvus's AGENTS.md "CMake standard" section — the reference
implementation. Canonical copy:
[CMAKE-HOUSE-STYLE.md](../CMAKE-HOUSE-STYLE.md) (written 2026-07-21); each repo's
AGENTS.md references it and keeps only project-specific deviations.
Core rules:

- Target-first: no directory-scope `include_directories`/`link_libraries`/
  global flags. PUBLIC only for requirements consumers must inherit
  (`target_compile_features(... PUBLIC cxx_std_20)`).
- Warnings PRIVATE, gated on `PROJECT_IS_TOP_LEVEL`; `<PROJ>_WERROR` option
  for CI (libstats' Strict build type is a negotiated exception, kept).
- Generator expressions: allowed for `$<BUILD_INTERFACE:>`/
  `$<INSTALL_INTERFACE:>`, `$<TARGET_OBJECTS:>`, and single-level
  compiler-ID selection. Anything deeper: plain `if()`. (libstats' bad
  experience was with nested conditional genexes, not genexes per se.)
- Options `<PROJ>_`-prefixed, defaults keyed off `PROJECT_IS_TOP_LEVEL`.
- Namespaced ALIAS in build tree (`proj::proj`), exported with the same
  namespace. Config files named `<proj>-config.cmake` (kebab; libhmm's
  CamelCase `libhmmConfig.cmake` migrates).
- `GNUInstallDirs` everywhere install exists; `write_basic_package_version_file`
  with `SameMajorVersion`.
- `CMAKE_EXPORT_COMPILE_COMMANDS ON` when top-level.
- cmake_minimum_required: 3.25 for all six (RESOLVED 2026-07-21: local
  cmake is 4.4.0, CI images and fleet machines all exceed 3.25; corvus
  already requires it).

## Defects to fix (Phase 0, unambiguous under any design)

1. **libhmm install is broken**: `install(TARGETS)` lacks `EXPORT`;
   `libhmmConfig.cmake.in` includes a never-generated `libhmmTargets.cmake`,
   so installed `find_package(libhmm)` hard-fails. Fix = proper
   `install(TARGETS ... EXPORT libhmm-targets)` + `install(EXPORT)`.
2. **libstats duplicated TBB link block** (CMakeLists.txt ~1126–1153 vs
   1158–1178, identical). Delete one.
3. **libhmm version file uses `AnyNewerVersion`** despite v3→v4 break.
   Change to `SameMajorVersion`.
4. **ewcalc `option(EWCALC_FRONTEND_CONFIG ... "Release")`** — string in a
   BOOL option. Change to `set(... CACHE STRING)`.
5. **libhmm `BUILD_SHARED_LIBS` documented no-op** — remove the option (both
   libs always built via OBJECT lib) or make it real; decide during Phase 3
   [OPEN, default: remove and document].

## Phases

### Phase 0 — Standards doc + defect fixes
Write house-style doc; land defects 1–4 (5 deferred to Phase 3 rename batch).
Each fix verified by local configure+build+ctest, then CI.

### Phase 1 — Install contract (libhmm, libstats, corvus)
- libhmm: repair export (defect 1), adopt GNUInstallDirs, add build-tree
  `libhmm::hmm`/`libhmm::hmm_static` aliases, kebab config name, pkg-config
  file (parity with libstats).
- libstats: switch hardcoded `lib`/`bin`/`include` to GNUInstallDirs vars
  (naming/targets already exported and correct).
- corvus: already conforming; verify `find_dependency(hwy)` lands in its
  config (static lib PRIVATE-links hwy → exported LINK_ONLY dep)
  [DERIVED — check during implementation]. Install remains gated on
  system-Highway provider (existing documented behavior).
- Verification: replicate libstats' `consumer_example` +
  `consumer_example_fetchcontent` pattern in libhmm and corvus; exercise the
  installed-package path (find_package) at least once in each library's CI,
  and once from pylibstats (already prefers find_package) / pylibhmm.

### Phase 2 — Presets in all six [COMPLETE 2026-07-22 — all six repos CI green]
Commits: libhmm 5445a0a, libstats 09ee94d, corvus 1c220b3, ewcalc c8494fa,
pylibhmm 7e6db1d, pylibstats 870877d. All locally verified (presets
configure; release-preset build+test per C++ repo: 47/49/4/13; pylibhmm
pip+pytest 165/165 confirming post-Phase-1 add_subdirectory path).
Process note: one commit hit a YubiKey "Bad PIN" GPG failure — resolved
with the documented gpg-agent resync, single retry (never disable
signing, never spam retries: PIN attempts count against the card).
Shared preset vocabulary (schema v6, min 3.25, no generator field);
release/debug/rel-with-debug → build/, build-debug/, build-relwithdebinfo/.
Extras RESOLVED: libstats dev (→build/, its Dev default; release deviates
to build-release/) + strict; corvus sanitize (build-san/); ewcalc frontend
(build-frontend/, own dir to avoid sticky cache); pylib* minimal
release/debug (pip/scikit-build-core stays the primary path).
Folded in: cmake_minimum_required 3.20→3.25 everywhere (corvus already);
policy-shift risk covered by one full build+test per C++ repo, and full
pip install -e + pytest for pylibhmm (doubles as post-Phase-1
add_subdirectory validation). AGENTS.md build sections go preset-first,
and every repo gets a compact self-sufficient "CMake standard" section
(corvus model) — this RESOLVES the earlier [OPEN] house-style-distribution
item: no checked-in doc copies; the local master doc is the editing
source.

### Phase 3 — Target-first refactors [spec user-approved 2026-07-22;
3A COMPLETE (libhmm 8b0b6f7 + pylibhmm 7a06b42, all CI green, incl.
LIBHMM_WERROR on the Clang leg); 3B Batch 1 COMPLETE (libstats 442e9a5,
all 5 workflows green — call-site-placement deviation approved: function
bodies moved, call sites inline to preserve configure ordering);
3B Batch 2 COMPLETE (libstats e12cf7e, all 5 workflows green — approved
deviation: GTEST_FOUND PARENT_SCOPE propagation, required because the
top-level summary reads it after add_subdirectory); 3B Batch 3 COMPLETE
(libstats 9ba4313, all 5 workflows green incl. gcc-14 Strict +
Sanitizers — the decisive legs for the flag refactor). PHASE 3 CLOSED.
Approved deviation in Batch 3: the spec's "no FORCE" cache-default
instruction was wrong — CMake auto-creates CMAKE_CXX_FLAGS_<CONFIG> as
an EMPTY cache entry for custom build types at project(), so unFORCEd
set() is a silent no-op; fixed with the guarded-FORCE idiom
(if(NOT var) set(... FORCE)), empirically proven before use.
Residual cleanup flagged for Phase 4: dead LIBSTATS_OPT_*/
LIBSTATS_DEBUG_INFO_* variables (only LIBSTATS_OPT_FULL_UNIX still
referenced once); stale AGENTS.md line-number pointers.
Batch 3 design as issued: custom-type per-config vars
(Dev "-O1 -g" / "/O1 /Zi /MD", Strict empty), Release/Debug chains drop
duplicate opt flags in favor of CMake per-config defaults (GCC Debug
-fstack-protector-strong preserved via shadow append),
libstats_apply_warnings(target) PRIVATE everywhere except GTest and the
INTERFACE headers target, /utf-8 + _CRT defines + /arch: + TBB fallback
globals unchanged. Accounting protocol: per-TU sorted-multiset flag diff
across Dev/Release/Strict with predicted hunk classes H2-H5; frontier
review of the accounting gates the commit]

Two independent sub-efforts. 3A (libhmm+pylibhmm coordinated pair) and 3B
(libstats) can run as separate agents/sessions. ewcalc/corvus: no Phase 3
work (already conforming; polish landed in earlier phases).

#### 3A — libhmm option rename + global-flag conversion (+ pylibhmm)

Facts verified 2026-07-22:
- CI injects -Werror on the Linux/Clang leg via raw
  `-DCMAKE_CXX_FLAGS=-Werror` (ci.yml:106, matrix `werror: true`).
- `ENABLE_STATIC_ANALYSIS` and `ENABLE_CPPCHECK` are declared but used
  NOWHERE in any CMakeLists — dead options (cppcheck runs from CI
  directly). `ENABLE_CLANG_TIDY` IS used (clang-tidy wiring on
  hmm_objects).
- Old names appear in ci.yml (~6 sites) and in pylibhmm (forced-OFF
  blocks for both its local-subdir and FetchContent paths).
- tests link `hmm_static` (+GTest+Threads), examples `hmm_static`, tools
  `hmm` — all inherit the Phase-1 PUBLIC include dirs, so removing the
  directory-scope `include_directories(include)` is safe once
  `hmm_objects` gets its own target-scope include.

Steps (one libhmm commit, one pylibhmm commit, same change set):
1. Rename options with deprecation shim placed BEFORE the option()
   declarations:
   BUILD_EXAMPLES→LIBHMM_BUILD_EXAMPLES, BUILD_TESTS→LIBHMM_BUILD_TESTS,
   BUILD_TOOLS→LIBHMM_BUILD_TOOLS,
   BUILD_BENCHMARKS→LIBHMM_BUILD_BENCHMARKS,
   ENABLE_CLANG_TIDY→LIBHMM_ENABLE_CLANG_TIDY.
   Shim pattern per name:
   `if(DEFINED <old> AND NOT DEFINED <new>) message(DEPRECATION ...) ;
   set(<new> ${<old>} CACHE BOOL "" )` — old -D spellings keep working
   for one release; remove shim at the next minor release.
2. Defaults: examples/tests/tools default `${PROJECT_IS_TOP_LEVEL}`
   (was ON) — embedding via add_subdirectory/FetchContent now auto-
   disables them. BENCHMARKS stays OFF, CLANG_TIDY stays OFF.
3. DELETE dead options `ENABLE_STATIC_ANALYSIS`/`ENABLE_CPPCHECK`
   (nothing consumes them; shim maps them to nothing with a DEPRECATION
   message so stale -D flags surface rather than silently no-op).
4. DELETE the `BUILD_SHARED_LIBS` option() + comment (defect 5
   disposition: remove; both libs always build from the OBJECT target —
   document in AGENTS.md).
5. New `LIBHMM_WERROR` option (default OFF) appending
   -Werror//WX to the warning set; ci.yml Clang leg switches from
   `-DCMAKE_CXX_FLAGS=-Werror` to `-DLIBHMM_WERROR=ON`.
6. Directory-scope → target scope:
   - `include_directories(include)` → `target_include_directories(
     hmm_objects PRIVATE include)` (final libs already export PUBLIC
     includes from Phase 1); delete the directory-scope line.
   - `add_compile_options(-Wall -Wextra -Wpedantic -Wpointer-arith)` →
     LIBHMM_WARN_FLAGS variable (corvus pattern) applied PRIVATE to
     hmm_objects and to test/tool/example/benchmark targets in their
     subdir CMakeLists; gate the whole set on PROJECT_IS_TOP_LEVEL.
   - global `-fPIC` add_compile_options → delete (hmm_objects already
     has POSITION_INDEPENDENT_CODE ON; executables don't need it).
   - `CMAKE_SHARED_LINKER_FLAGS` mutations (`-undefined dynamic_lookup`
     on macOS, `--no-undefined` on Linux) → `target_link_options(hmm
     PRIVATE ...)` VERBATIM — no behavior change this phase.
     [OPEN, post-Phase-3] whether macOS `-undefined dynamic_lookup`
     should be dropped entirely (it defers symbol resolution and can
     hide link errors; needs its own investigation).
   - MSVC global `/utf-8` stays global — documented exception (source
     encoding must also cover fetched GTest TUs; comment already
     explains).
7. ci.yml: update all old-name -D sites to LIBHMM_* spellings.
8. pylibhmm CMakeLists: local-subdir branch drops ALL forced-OFF lines
   (PROJECT_IS_TOP_LEVEL defaults now handle it — this deliberately
   validates the new defaults); FetchContent branch (pinned v4.2.5 =
   old names) KEEPS the old-name forced-OFF lines with a comment tying
   their removal to the next libhmm tag bump.

Verification 3A: libhmm full matrix locally (release preset build +
47 tests) + configure-log check that a bare `cmake -B` with NO options
still builds everything it used to top-level; deprecation shim
smoke-tested (`-DBUILD_TESTS=OFF` still disables tests, warns);
`-DLIBHMM_WERROR=ON` builds clean with AppleClang; pylibhmm pip
install -e + pytest 165 (local path, no forced-OFF lines); grep proves
no unprefixed spellings remain outside the shim + pylibhmm's
FetchContent branch. CI: libhmm all 8 jobs + pylibhmm matrix green.

Routing 3A: Sonnet, default effort. Escalate if: the shim interacts
badly with CMake option()/cache semantics in any observable way, any
matrix leg's effective flags change beyond the -Werror mechanism swap,
or pylibhmm's FetchContent CI path breaks.

#### 3B — libstats incremental modularization

Fact verified 2026-07-22: NO `CMAKE_CXX_FLAGS_DEV`/`_STRICT` per-config
variables exist — the custom build types receive flags exclusively from
the global add_compile_options chains. Consequence: naive conversion of
those chains to target scope would leave FetchContent deps (GTest) with
NO optimization flags under Dev/Strict. The conversion must instead
define the custom build types properly.

Extraction sequence — three push batches, each locally verified then
CI-green before the next; every step behavior-preserving:

Batch 1 (pure moves, include() preserves directory scope — zero
behavior change by construction):
- B1: `cmake/Threading.cmake` ← detect_threading_systems(),
  detect_tbb_unified(), and the macOS GCD-preference logic, included at
  the exact same point in the top-level file.
- B2: `cmake/CompilerFlags.cmake` ← the warning-set variables
  (LIBSTATS_*_WARNINGS etc.) + the per-compiler build-type
  add_compile_options chains + MSVC compile definitions, included at
  the same point. Still directory-scope at this step, deliberately.
  Verification B1/B2: configure-log diff (must be identical modulo
  timestamps) + compile_commands.json diff (must be EMPTY) for Dev and
  Release presets + full test suite (49) + Strict configure.

Batch 2 (subdirectory moves):
- B3: tests → `tests/CMakeLists.txt` via add_subdirectory. MUST
  preserve: `build/tests/` output location (set RUNTIME_OUTPUT_DIRECTORY
  to `${CMAKE_BINARY_DIR}/tests`, not CURRENT_BINARY), ctest
  WORKING_DIRECTORY `${CMAKE_BINARY_DIR}`, test names, labels, the
  run_tests/run_all_tests/run_tests_timing custom targets, and GTest
  detection (moves with the tests since only tests consume it).
  enable_testing() stays top-level.
- B4: tools → `tools/CMakeLists.txt` (same output-dir rule:
  `${CMAKE_BINARY_DIR}/tools`).
  Verification B3/B4: `ctest --show-only=json-v1` diff pre/post (test
  names, commands, properties identical), binaries exist at documented
  paths (AGENTS.md: build/tests/, build/tools/ — never bin/), full
  suite 49, tools smoke (system_inspector --quick runs).

Batch 3 (the only semantics-touching step):
- B5: define the custom build types properly and de-globalize:
  1. Introduce `CMAKE_CXX_FLAGS_DEV` and `CMAKE_CXX_FLAGS_STRICT`
     (cache init, per compiler) carrying the OPTIMIZATION/debug-info
     portion of what the global chains currently add (e.g. Dev:
     -O1 -g; Strict: none beyond defaults) so every target INCLUDING
     FetchContent deps gets per-config flags the standard CMake way.
  2. Move WARNING flags (common + strict sets + -Werror) to per-target
     PRIVATE application via a `libstats_apply_warnings(target)`
     function in CompilerFlags.cmake, applied to the object libs,
     final libs, tests, tools, examples — NOT to GTest.
  3. The Windows global `/arch:` block STAYS global — documented
     exception (its interaction with LIBSTATS_PORTABLE and the MSVC
     __AVX2__ macro cascade is load-bearing per the in-file comment).
  4. The pkg-config-TBB-path global include_directories inside
     detect_tbb_unified stays (rare fallback path, v1.5.3 hotfix
     comment documents it) — accepted residual.
  Expected observable deltas (the ONLY acceptable ones): GTest TUs
  under Dev/Strict gain the per-config opt flags via the standard
  mechanism instead of injected warnings+opts (they LOSE our warning
  flags — desirable); everything else's flag set must be provably
  identical via compile_commands.json diff per build type
  (Dev, Release, Strict) with a written accounting of every hunk.
  Verification B5: the compile_commands accounting + full suite (49) +
  local Strict configure/build attempt (AppleClang) + real CI Strict
  (gcc-14) + Sanitizers legs — the GCC Strict leg is the true gate
  (AppleClang proves nothing about the GCC flag set; lesson from the
  CI effort).

Routing 3B: Sonnet executes each batch from this spec; frontier
reviews the B5 compile_commands accounting before that batch is
pushed. Escalate if: any compile_commands diff hunk lacks a written
justification, ctest --show-only output differs in B3/B4, Strict/
Sanitizers CI legs fail for reasons beyond the predicted GTest deltas,
or the include-shim/object-library structure has to change (it must
NOT this phase).

### Phase 4 — Verification & documentation
Full local build+test per project on this machine (M1/AppleClang), push,
real CI green on all six (CI is the cross-platform gate — lesson from the
2026-07 CI effort: local AppleClang proves nothing about GCC/MSVC legs).
Update each repo's AGENTS.md/PLAN.md; retire this file's [OPEN] tags.

## Sequencing constraints

- Parent-lib changes that touch option names, target names, or install
  layout must land as coordinated pairs with pylibhmm/pylibstats (they
  consume via add_subdirectory/FetchContent/find_package respectively —
  three different consumption paths, all must stay green).
- pylibhmm's FetchContent pin (libhmm v4.2.5) only exercises on machines
  without a local `../libhmm`; the deprecation shim protects the pinned
  release path until the next libhmm tag.
- Execution is largely pattern-following once started — suitable for a
  mid-tier model per the house Model & Effort Routing convention; escalate
  if a refactor step surfaces a behavior difference.

## Open items
Resolved during execution (retired 2026-07-23): house-style distribution
(self-sufficient AGENTS.md sections, Phase 2); per-repo preset extras
(Phase 2); libhmm BUILD_SHARED_LIBS (removed, Phase 3A).

Post-completion review round (2026-07-23): libstats inter-tier
add_dependencies removed (f05f227 — ~9% clean-build wall-time win,
interleaved A/B on M1, first pair discarded as contaminated); corvus CI
now caches the pinned-Highway prefix (40fef83 — cache-hit path proven by
rerun). Review also identified: libstats include/ restructure to kill the shim
(spec written, filed as libstats issue #83 on the v2.2.0 milestone,
2026-07-23); symbol-visibility/export-header work and Dev/Strict
unification (next-major items); libhmm+corvus README install sections
(editorial). Windows flag accounting (Batch 3 H5) to be closed manually
on the Ryzen box.

SIMD hygiene round (2026-07-24, all CI-green): libstats 52da6c2
(SIMDDetection cmake-format conformance, format-only) + a0ef12a
(application functions split to cmake/SIMDApplication.cmake, verbatim);
libhmm e78c8fa (SIMD block extracted to cmake/SimdDispatch.cmake,
verbatim). e78c8fa's first Linux/Clang run failed ENVIRONMENTALLY: a
runner host advertising avx10.1-256 + Clang's
-Winvalid-feature-combination promotion + -Werror + -march=native.
Extraction exonerated (genexes byte-identical; GCC leg green). Fixed in
995ba60: ALL libhmm CI configures now pass -DLIBHMM_PORTABLE=ON —
hosted runners are a hardware lottery and portable mode is the designed
answer for unknown-ISA machines; per-ISA dispatch TUs keep targeted
flags so SIMD coverage is unchanged. Warning suppression rejected (it
guards real 512-bit-promotion SIGILL risk).

Still open — future candidates, not blockers:
- [OPEN] .pc files bake CMAKE_INSTALL_PREFIX at configure time (Phase 1
  note; cosmetic for current flows).
- [OPEN] pylibhmm/pylibstats CI exercising the parents' installed-package
  path (Phase 1 deferral; natural fit at the next parent release).
- [OPEN] libhmm macOS `-undefined dynamic_lookup` on the shared lib —
  moved to target scope verbatim in 3A; whether to drop it needs its own
  investigation.
- [OPEN] libhmm deprecation shim removal at v4.3.0, together with bumping
  pylibhmm's FetchContent pin and dropping its old-name forced-OFF lines.
- [OPEN] corvus fetched-Highway install gate — tracked in corvus PLAN.md.
- [OPEN] Formatter/linter configs are not fleet-standard and were left
  alone during the 2026-07-26 standards move: libhmm's `.clang-format`
  derives from LLVM, libstats' from Google, with different naming rules
  and include-ordering policy; `.clang-tidy` check sets differ likewise;
  corvus and the two pylib* repos carry neither. Unifying them is a code
  change across every source file (reformat + fixups), not a config move
  — needs its own effort with its own verification, and a decision on
  which base style wins.

Retired 2026-07-26: libstats cmake/SIMDDetection.cmake cmake-format
conformance (closed by 52da6c2 in the SIMD hygiene round above).
