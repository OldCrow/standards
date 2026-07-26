# CMake House Style — OldCrow Projects

Canonical build-system standard for libhmm, pylibhmm, libstats, pylibstats,
ewcalc, and corvus. Derived from corvus's CMake standard (the reference
implementation) plus the design decisions recorded in
[records/BUILD-STANDARDIZATION-PLAN.md](records/BUILD-STANDARDIZATION-PLAN.md)
(2026-07-21). Each repo's AGENTS.md states
only its *deviations* from this document; when they conflict, the repo's
AGENTS.md wins for that repo and the deviation must be listed there
explicitly.

Rules are written for the libraries; the two Python binding repos and the
ewcalc app follow them wherever the concept applies (targets, options,
scoping) and skip what doesn't (install contract, presets extras noted
per repo).

## 1. Scope and non-goals

- CMake is the only supported meta-build system. Meson/Bazel/etc. are out
  of scope — no parallel build definitions, ever.
- CMakeLists must be **generator-agnostic**: no Makefile- or Ninja-specific
  constructs; anything invocable must work via `cmake --build` /
  `ctest --test-dir`. Ninja is the *preferred* generator (documented, used
  by CI), never a requirement.
- `cmake_minimum_required(VERSION 3.25)` for all six repos (gives
  `PROJECT_IS_TOP_LEVEL` and FetchContent `SYSTEM`). Fleet check 2026-07-21:
  all dev machines and CI images exceed this comfortably (local: 4.4.0).

## 2. Naming

- **Options**: `<PROJ>_`-prefixed, SCREAMING_SNAKE: `LIBHMM_BUILD_TESTS`,
  `CORVUS_WERROR`. Never bare `BUILD_TESTS`/`ENABLE_*` — unprefixed names
  collide in superprojects (FetchContent/add_subdirectory).
- **Targets**: primary library target named for the project; always provide
  build-tree namespaced aliases matching the exported names
  (`corvus::corvus`, `libhmm::hmm`, `libhmm::hmm_static`,
  `libstats::libstats_static`, …). Consumers — including sibling repos —
  link only the `ns::name` form, so add_subdirectory, FetchContent, and
  find_package are interchangeable.
- **Export set**: `<proj>-targets`, namespace `<proj>::`.
- **Package files**: kebab-case — `<proj>-config.cmake`,
  `<proj>-config-version.cmake`, installed to `<libdir>/cmake/<proj>`.
- **Cache/internal variables**: `<PROJ>_` prefix; function-local temps
  `_<proj>_` prefix.

## 3. Target-first scoping

- No directory-scope commands in library code paths: no
  `include_directories()`, `link_directories()`, `link_libraries()`,
  `add_compile_options()` for things expressible per-target. (App-level
  repos like ewcalc may use directory scope deliberately for cross-cutting
  instrumentation — e.g. coverage flags before add_subdirectory — with a
  comment saying why.)
- Visibility rules:
  - PUBLIC: only requirements consumers must inherit to compile/link
    against the target (`target_compile_features(x PUBLIC cxx_std_20)` when
    public headers need it; transitively required libs).
  - PRIVATE: everything about *building* the target — warnings, opt flags,
    internal include dirs, implementation-detail deps.
  - INTERFACE: header-only usage requirements.
- Include dirs on installed targets always split
  `$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>` /
  `$<INSTALL_INTERFACE:include>`.
- Warnings are PRIVATE and gated on `PROJECT_IS_TOP_LEVEL` (a consumer
  embedding the project must never inherit our -Wall/-Werror). `-Werror` /
  `/WX` only via an explicit `<PROJ>_WERROR` option (default OFF), enabled
  by CI. *Deviation*: libstats keeps its `Strict` build type as the -Werror
  vehicle; ewcalc (an app whose subprojects are never embedded) hardcodes
  -Werror on its own targets — both documented in their AGENTS.md.
- PIC via `POSITION_INDEPENDENT_CODE ON` target property (or on the OBJECT
  library), never a global `-fPIC` flag.

## 4. Generator expressions

Use genexes for exactly three things:

1. Build/install interface separation (`$<BUILD_INTERFACE:>` /
   `$<INSTALL_INTERFACE:>`).
2. Object-library composition (`$<TARGET_OBJECTS:>`).
3. Single-level compiler/config selection
   (`$<$<CXX_COMPILER_ID:MSVC>:/W4>`, `$<IF:$<CXX_COMPILER_ID:MSVC>,a,b>`).

Anything requiring nesting beyond one level of conditions goes in plain
`if()` blocks into a variable, applied per-target. (libstats' history: the
failures that motivated its "traditional approach" comments were deeply
nested conditional genexes — those stay banned; the three uses above are
required, not optional.)

## 5. Build types

- Standard set: `Release` (default for single-config generators in the
  numerical libraries — house rule: perf/accuracy numbers only from
  optimized builds), `RelWithDebInfo` (profiling), `Debug` (debugger +
  sanitizers). Constrain via
  `set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS ...)`.
- Single-config generators must get an explicit default via the
  `if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)` guard —
  never rely on CMake's empty default.
- Sanitizers via `<PROJ>_SANITIZE` list option (corvus pattern), not extra
  build types.
- *Deviation*: libstats keeps `Dev` (its default) and `Strict` custom types
  — grandfathered, documented in its AGENTS.md, not to be copied into other
  repos.

## 6. Options

- Declared near the top, one block, all `<PROJ>_`-prefixed.
- Component toggles (`_BUILD_TESTS`, `_BUILD_EXAMPLES`, `_BUILD_TOOLS`)
  default `${PROJECT_IS_TOP_LEVEL}` so embedding a project builds only the
  library.
- String-valued cache settings use `set(... CACHE STRING ...)` +
  `set_property(CACHE ... PROPERTY STRINGS ...)` where enumerable — never
  `option()` (BOOL only).
- Renames ship with a one-release deprecation shim: map old→new with a
  `message(DEPRECATION ...)`, remove at the next minor release.

## 7. Install contract (libhmm, libstats, corvus)

What installs: static + shared libraries, public headers, CMake package
files, pkg-config file. Never tests, examples, tools, or benchmarks.

- Paths via `include(GNUInstallDirs)`: `${CMAKE_INSTALL_LIBDIR}`,
  `${CMAKE_INSTALL_INCLUDEDIR}`, `${CMAKE_INSTALL_BINDIR}` (Windows DLLs).
- `install(TARGETS ... EXPORT <proj>-targets)` + `install(EXPORT ...
  NAMESPACE <proj>:: ...)` + `configure_package_config_file` +
  `write_basic_package_version_file(... COMPATIBILITY SameMajorVersion)`.
- Config files call `find_dependency()` for every dependency of an exported
  target (Threads, TBB, OpenMP, hwy, …).
- Shared libs: `VERSION ${PROJECT_VERSION}` + `SOVERSION
  ${PROJECT_VERSION_MAJOR}`; macOS `INSTALL_NAME_DIR "@rpath"` +
  `MACOSX_RPATH`; Linux SONAME via the standard properties (no manual
  `-Wl,-soname` string splicing).
- **Loader registration**: none automated. CMake never runs `ldconfig`
  (breaks non-root and DESTDIR staging). READMEs note that Linux users
  installing to a system prefix run `sudo ldconfig` once; rpath handles
  everything else.
- Every installable library keeps a consumer smoke test
  (`consumer_example/` find_package + `consumer_example_fetchcontent/`,
  libstats pattern) and CI exercises the installed-package path.
- Python repos: wheel-only (`install(TARGETS _core LIBRARY DESTINATION
  <pkg>)` for scikit-build-core), no package export. ewcalc: no `install()`
  — platform scripts own packaging.

## 8. Dependencies

- Pattern: `find_package` first, pinned `FetchContent` fallback (corvus
  Highway, libstats GTest, pylib* parent libs). Pins are exact release tags,
  bumped deliberately (with revalidation where the pin tracks an audit).
- `FetchContent_Declare` uses `SYSTEM` (fetched headers must not trip our
  warnings) and `GIT_SHALLOW TRUE`.
- When adding a subproject, force its component toggles off via
  `set(<DEP>_BUILD_TESTS OFF CACHE BOOL "" FORCE)` — which is exactly why
  rule 2 (prefixed options) exists.
- No vendored copies of dependencies in-tree.

## 9. Presets

Every repo carries `CMakePresets.json` (schema version ≥ 6) with the shared
vocabulary; presets set `binaryDir`, `CMAKE_BUILD_TYPE`, and cache options
— **never a `generator` field** (user default rules; CI passes `-G Ninja`
explicitly).

- Common: `release` → `build/`, `debug` → `build-debug/`,
  `rel-with-debug` → `build-relwithdebinfo/` (libhmm's existing dir naming
  is the precedent).
- Project extras (same file, documented in AGENTS.md): e.g. libstats `dev`,
  `strict`; corvus `sanitize`; ewcalc `frontend`. Extras list finalized in
  Phase 2.
- `CMakeUserPresets.json` is gitignored and free for machine-local use.

## 10. Misc mechanics

- `CMAKE_EXPORT_COMPILE_COMMANDS ON` when `PROJECT_IS_TOP_LEVEL` (clangd).
- Version header / version constants generated from `project(VERSION)` via
  `configure_file` — the CMake version is the single source of truth.
- Status messages: one concise configuration summary block at the end;
  verbose diagnostics behind a `<PROJ>_VERBOSE_BUILD`-style option, not
  unconditional.
- `cmake-format` is the formatter of record where adopted (libhmm pre-commit
  today); repo-by-repo adoption tracked in Phase 3, not required yet.
- Compiler minimum-version guards: fail at configure time with actionable
  messages (libhmm/libstats pattern — keep).

## Known per-repo deviations (summary)

| Repo | Deviation | Why |
|---|---|---|
| libstats | `Dev`/`Strict` custom build types; `Dev` default; include shim | Grandfathered; shim removal needs include/ restructure (tracked post-v2) |
| ewcalc | Unconditional -Werror; directory-scope coverage flags; no install | App, not a library; subprojects never embedded elsewhere |
| pylibhmm | Prefers local `../libhmm` sibling over FetchContent | Dev-loop speed; documented in its AGENTS.md/PLAN.md |
| pylibstats | Prefers installed find_package over sibling | Deliberate contrast to pylibhmm (exercises installed-package path) |
| corvus | `install` gated on system-Highway provider | Fetched Highway isn't in the export set; see corvus PLAN.md |

## Distribution (resolved 2026-07-26)

This file is the single canonical copy, versioned in
[OldCrow/standards](https://github.com/OldCrow/standards) and referenced
from the six repos by URL. Both halves of the old Phase 2 question apply:
each repo's AGENTS.md CMake section stays **self-sufficient** for
day-to-day work (an agent that cannot reach this repo is still correct
about the repo it is in), and it links here for the full rules and for
anything cross-repo. Edit this file first; propagate deviations to the
affected AGENTS.md in the same change.
