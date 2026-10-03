# Windows Toolchain — OldCrow Projects

Canonical Windows build environment (MSVC and clang-cl) for libhmm, pylibhmm, libstats,
pylibstats, ewcalc, and corvus. Consolidated 2026-09-07 from the four
copies that had grown independently in libstats, libhmm, pylibhmm, and
pylibstats. Each repo's AGENTS.md states only its *deviations* and the
repo-specific commands (its own targets, test binaries, and artifacts);
when they conflict, the repo's AGENTS.md wins for that repo.

Paths below are common defaults. Windows tool locations vary by
installation method — direct installer, `winget`, `chocolatey`, Microsoft
Store — and Build Tools and full VS editions install to different
directories. Prefer `vswhere` over hard-coding any of them.

## 1. One-time setup

- **Visual Studio 2022 (17.x) or later**, C++ desktop workload. Build Tools
  alone are sufficient; a full IDE edition is not required. Verified through
  VS 2026 (v18, MSVC 14.5x). Install from
  <https://visualstudio.microsoft.com/downloads/>,
  `winget install Microsoft.VisualStudio.2022.BuildTools`, or
  `choco install visualstudio2022buildtools`.
  - Build Tools: `C:\Program Files (x86)\Microsoft Visual Studio\{version}\BuildTools\`
  - Full editions: `C:\Program Files\Microsoft Visual Studio\{version}\{edition}\`
  - `{version}` is `2022` for VS 17.x and `18` for VS 2026; `{edition}` is
    Community, Professional, or Enterprise.
- **clang-cl**, for the repos that build with it (§5): the Visual Studio
  component "C++ Clang tools for Windows", which installs to
  `{VS}\VC\Tools\Llvm\x64\bin\`, or a standalone LLVM
  (`winget install LLVM.LLVM`). Either way it still needs the MSVC
  installation above for the linker, the CRT, and the Windows SDK.
- **CMake ≥ 3.25** — <https://cmake.org/download/>, `winget install Kitware.CMake`,
  or `choco install cmake`. Generator support for a new VS major version needs
  a correspondingly new CMake: VS 2026 needs CMake ≥ 4.1.
- **Smart App Control must be Off** — Windows Security → App & Browser
  Control → SAC settings. SAC blocks locally compiled executables outright.
  Turning it off is one-way: it cannot be re-enabled without a Windows reset.
- **Add a Defender exclusion for the repo root**, from an elevated shell:
  `Add-MpPreference -ExclusionPath "<repo root>"`. Without it,
  cloud-delivered protection intermittently quarantines freshly linked test
  executables. This surfaces as ctest reporting `BAD_COMMAND` or "Not Run" on
  a rotating subset of tests — on 2026-09-03 two consecutive suite runs each
  lost a different 3–5 binaries. The detection is consistently
  `Trojan:Win32/Wacatac.B!ml`: the `!ml` suffix marks it as Defender's machine
  -learning heuristic firing on unsigned, locally linked binaries, not a real
  finding. **If a test executable vanishes after a successful build, suspect
  quarantine before suspecting the build.**
- **Know the pinentry foreground trap; do not "fix" it by disabling the
  foreground lock.** A background process cannot raise a window while the
  foreground lock is active, and gpg-agent is one — so the pinentry it spawns
  opens *without focus*. It never reports a permission problem: a commit fails
  `gpg: signing failed: Timeout`, an SSH push or fetch fails `agent refused
  operation`, and a foreground command can simply sit for minutes with the
  dialog behind the terminal. On 2026-09-07 that cost four failures across two
  repos and two killed commands. **Until you answer it, a YubiKey touch lands
  in whatever window actually holds focus — for a key in OTP mode, that types
  a one-time password into that window.**
  - **Recovery when it happens:** click the unfocused pinentry in the taskbar
    (it flashes), enter the PIN, retry.
  - **The fleet's mitigation is fewer prompts, not less protection.** The
    native agent's `default-cache-ttl` is 3600 rather than the 600 default, so
    a working session meets the prompt roughly six times less often;
    `max-cache-ttl` stays at 7200, so the outer bound on a single unlock is
    unchanged. Batching helps for the same reason: a signed commit and its
    push share one cache entry, so doing them together costs one prompt.
  - **What we deliberately did not do:** set
    `HKCU\Control Panel\Desktop\ForegroundLockTimeout` to 0. That disables the
    foreground lock for everything, letting any process steal focus mid-typing
    — the mirror of the hazard above — and unexpected focus changes are worse
    than an occasional extra prompt. Two findings if you ever revisit it: the
    registry holds `200000`, which is Windows' own default and not anyone's
    choice, and no Group Policy imposes it; but the *live* value read through
    `SystemParametersInfo(SPI_GETFOREGROUNDLOCKTIMEOUT)` is `0x7FFFFFFF`, set
    at runtime by something we did not identify. A registry-only edit may
    therefore not take effect at all — `SPI_SETFOREGROUNDLOCKTIMEOUT` with
    `SPIF_UPDATEINIFILE | SPIF_SENDCHANGE` is the route that changes both, and
    even then it may not survive the next logon.

## 2. Per-session toolchain activation

Whether you need this depends on the generator, so check before assuming:

- **Visual Studio generator** (the CMake default on Windows): no activation
  needed. The generator locates its own toolchain.
- **Ninja, or invoking `cl.exe` or `clang-cl` directly**: activate MSVC once
  per PowerShell session. It does not persist. clang-cl takes `link.exe`,
  `rc.exe`, `INCLUDE` and `LIB` from this environment; with the
  VS-bundled copy, also put `$vsPath\VC\Tools\Llvm\x64\bin` on `PATH`.

```powershell
# Locate the newest installed Visual Studio, any version or edition:
$vsPath = & "${env:ProgramFiles(x86)}\Microsoft Visual Studio\Installer\vswhere.exe" -latest -products * -property installationPath
$vcvars = "$vsPath\VC\Auxiliary\Build\vcvars64.bat"
# Or pin one explicitly, e.g. VS 2022 Build Tools:
#   "C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\VC\Auxiliary\Build\vcvars64.bat"
$envVars = cmd /c "`"$vcvars`" > nul && set"
foreach ($line in $envVars) {
    if ($line -match "^([^=]+)=(.*)$") {
        [System.Environment]::SetEnvironmentVariable($Matches[1], $Matches[2], 'Process')
    }
}
```

For Unicode glyphs in tool output, set UTF-8 for the session as well:

```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$OutputEncoding = [System.Text.Encoding]::UTF8
```

## 3. Generator selection

Do not pin a generator locally. CMake's default auto-selects the newest
installed Visual Studio; a hard-coded
`CMAKE_GENERATOR="Visual Studio 17 2022"` breaks the moment VS upgrades in
place, because a 2022→2026 upgrade leaves an empty `2022\` directory behind
and the pin still resolves to it. Toolset reproducibility belongs to CI,
where the runner image pins it. Pass an explicit generator only to
troubleshoot generator selection itself.

This is the Windows case of the generator-agnostic rule in
[CMAKE-HOUSE-STYLE.md](CMAKE-HOUSE-STYLE.md) §1, and of its ban on a
`generator` field in presets (§9). The one exception is the
`windows-clang-cl` preset, which must pin Ninja (§5).

## 4. Multi-config hazard: stale Debug binaries

The Visual Studio generator is multi-config and writes Debug and Release
executables to the *same* output directory. A stale Debug executable next to
a Release DLL is a CRT mismatch, which presents as heap corruption at
runtime rather than as a link error.

`cmake --build --clean-first` does not save you: it cleans Release artifacts
but leaves existing Debug executables alone when their timestamps look
current.

After any clean rebuild, verify a dynamic test executable links the Release
CRT:

```powershell
dumpbin /imports <build>\tests\<some_dynamic_test>.exe | Select-String vcruntime
# VCRUNTIME140.dll  = Release, correct
# VCRUNTIME140D.dll = Debug, stale — delete the EXE and rebuild that target
```

Each repo's AGENTS.md names the specific binaries worth checking.

## 5. clang-cl

clang-cl is Clang behind an MSVC-style command line. It links against the
MSVC CRT and produces MSVC-ABI objects, so its libraries mix freely with
`cl.exe` ones. corvus and libstats support it; it is the full-speed
Windows build for anything that compiles corvus:

- **Highway blocklists every AVX-512 target under `cl.exe`**, so a
  corvus compiled by MSVC dispatches AVX2 at best, on any CPU.
- **`cl.exe` compiles corvus's kernels 3–19× slower** than clang-cl does
  at the same tier (Zen 4, 2026-09-30; libstats
  `docs/bench-evidence/2026-09-30-zen4-clangcl-avx2/`). Results are
  bit-identical.

So Windows performance numbers come from a clang-cl build, and a
`cl.exe` build of a corvus consumer is a correctness and diagnostics
build. libstats warns at configure time when `cl.exe` is compiling a
fetched corvus.

**Selecting it.** After §2's activation:

```powershell
cmake --preset windows-clang-cl        # Ninja, Release, clang-cl for C and C++
```

Beyond the build type, the preset pins two things no other preset does:

- *The generator.* The Visual Studio generator ignores
  `CMAKE_CXX_COMPILER` and `CMAKE_BUILD_TYPE`: without Ninja the preset
  silently yields an MSVC Debug build under a name promising clang-cl
  Release. This is the exception to the no-`generator` rule.
- *C as well as C++.* Highway enables both languages; leaving C to
  default hands MSVC-style flags to a GNU-driver `clang`.

With the Visual Studio generator, select the toolset instead:
`cmake -B build -A x64 -T ClangCL`.

**Three ways to combine the compilers**, all verified on libstats:
everything with clang-cl (the preset; dependencies fetched); `cl.exe`
for the consumer with a clang-cl corvus and Highway installed to a prefix
and found through `CMAKE_PREFIX_PATH`; everything with `cl.exe` (slow
corvus). One FetchContent tree cannot mix compilers, which is why the
second needs the installed prefix.

**Porting a repo to clang-cl** — what broke in libstats, in order of how
much it explained:

- `-Wall` means `cl.exe`'s `/Wall`, i.e. `-Weverything`. Use `/W4`
  (clang-cl's `-Wall -Wextra`). Branch on
  `CMAKE_CXX_COMPILER_FRONTEND_VARIANT STREQUAL "MSVC"`, not on
  `CMAKE_CXX_COMPILER_ID` alone, which is plain `Clang`.
- Both `_MSC_VER` and `__clang__` are defined. An `#if` that tests
  `__clang__` or `__GNUC__` first takes the GNU branch
  (`<cpuid.h>`, `__cpuid_count`) while the includes took the MSVC one
  (`<intrin.h>`, `__cpuidex`). Test `_MSC_VER` first.
- The driver rejects GNU options that are not warnings: `-pedantic` is
  `-Wpedantic`; others pass through as `/clang:<option>`.
- `/arch:SSE2` does not exist on x64. `cl.exe` ignores it; clang-cl
  warns.
- A fetched GoogleTest builds its own sources with `-WX` under an
  MSVC-style driver; 1.17.0 trips Clang 21's `-Wcharacter-conversion`.
  Suppress it on GoogleTest's targets, and fetch GoogleTest `SYSTEM`.
- Expect diagnostics the `cl.exe` build never showed: undiscarded
  `[[nodiscard]]` results, `inline` on a function defined in another
  file, and under a strict warning set sign conversions in Win32 calls.

**CI.** A clang-cl leg asserts its own compiler — `CMakeCXXCompiler.cmake`
must say `Clang` with the `MSVC` frontend — because a leg that falls back
to `cl.exe` still builds and tests green. Hosted runners do not all have
AVX-512, so a leg cannot assert a SIMD tier; tier evidence comes from
native runs.
