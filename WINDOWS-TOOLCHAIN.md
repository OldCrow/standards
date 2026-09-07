# Windows Toolchain — OldCrow Projects

Canonical Windows/MSVC build environment for libhmm, pylibhmm, libstats,
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
- **Set `ForegroundLockTimeout` to 0**, then log out and back in:
  `reg add "HKCU\Control Panel\Desktop" /v ForegroundLockTimeout /t REG_DWORD /d 0 /f`.
  The Windows default is 200000 ms, during which a background process cannot
  raise a window. gpg-agent is one, so the pinentry it spawns opens without
  focus and times out. It never reports a permission problem: a commit fails
  `gpg: signing failed: Timeout`, an SSH push or fetch fails `agent refused
  operation`, and a foreground command can simply sit for minutes with the
  dialog behind the terminal. On 2026-09-07 that cost four failures across two
  repos and one killed command. Recovery without the registry change: click
  the unfocused pinentry in the taskbar, enter the PIN, retry — and batch
  signed operations, since a commit and its push share one 600 s PIN cache.
  **Not build-only:** anything authenticating through the YubiKey is affected,
  including `git` in repos outside this fleet. **Until it is set, a YubiKey
  touch goes to whatever window actually holds focus — for a key in OTP mode,
  that types a one-time password into that window.** The value is per-user, so
  each profile needs it; chezmoi does not manage the Windows registry.

## 2. Per-session toolchain activation

Whether you need this depends on the generator, so check before assuming:

- **Visual Studio generator** (the CMake default on Windows): no activation
  needed. The generator locates its own toolchain.
- **Ninja, or invoking `cl.exe` directly**: activate MSVC once per PowerShell
  session. It does not persist.

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
`generator` field in presets (§ presets).

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
