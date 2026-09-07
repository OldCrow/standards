# Session Start — OldCrow Projects

Canonical session-start ritual for libhmm, pylibhmm, libstats, pylibstats,
ewcalc, and corvus. Consolidated 2026-09-07 from the four copies in
libstats, libhmm, pylibhmm, and pylibstats. Each repo's AGENTS.md states
only its own additions — the specific build paths, presets, and ISA tiers
that its platform notes cover.

## Why this exists

The fleet is deliberately heterogeneous: an Intel Kaby Lake Mac (AVX2, no
AVX-512), an Apple Silicon M1 (NEON), and a Zen 4 Windows box (AVX-512).
A build or a benchmark carried over from the previous session's machine is
not wrong in any way the toolchain will tell you about — it compiles, links,
and runs. It is simply measuring or exercising the wrong thing, and a whole
validation leg can be spent before anyone notices.

So the architecture check is not a formality to skip when you are in a
hurry. It is the step that makes every result after it attributable.

## The ritual

At the start of every session, in order:

1. **Verify the machine** — OS and CPU architecture — before making any SIMD
   or build-path assumption. For the Python binding repos, verify the Python
   interpreter's architecture too: a mismatch between interpreter and
   extension module is a load-time failure that reads like a packaging bug.
2. **Select the build path for this host** from the repo's platform notes.
3. **Reconfigure and rebuild** when the machine or architecture differs from
   the previous session's context. Do not build, install, or test before the
   check is complete.

## Architecture checks

```bash
# macOS / Linux
uname -s
uname -m
```

```powershell
# Windows PowerShell
[System.Runtime.InteropServices.RuntimeInformation]::OSArchitecture
[System.Runtime.InteropServices.RuntimeInformation]::ProcessArchitecture
```

For the Python binding repos, on any platform:

```bash
python -c "import platform, struct; print(platform.system(), platform.machine(), struct.calcsize('P')*8)"
```
