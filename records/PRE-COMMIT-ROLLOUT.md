# Pre-commit across the fleet — DRAFT

**Status: DRAFT for review, 2026-11-11. Nothing here is binding and no
repo has been changed.** Section 2 is proposed text for
[CI-HOUSE-STYLE.md](../CI-HOUSE-STYLE.md) §6; it moves there only as the
repos adopt it ("no rule without a follower"). Section 3 is the rollout.
Section 4 lists the decisions that are the owner's to make.

## 1. Why, and where things stand

Three findings on 2026-11-11 prompted this:

- **libstats has the fullest hook set and enforces none of it on the
  Windows machine.** `.pre-commit-config.yaml` exists, but
  `.git/hooks/pre-commit` does not, `pre-commit` is not on `PATH`, and no
  workflow runs the hooks. Every libstats commit made that day skipped
  them. A local hook is opt-in per clone; on a multi-machine fleet with
  agents committing, some clone will always be missing it.
- **libstats CI admits the gap.** Its lint job runs `clang-format-17`
  with `|| true`, because that version does not match the developer
  toolchain. The pre-commit clang-format hook pins its own version, which
  removes the mismatch.
- **libhmm's CI hook job crashed with no source change** when
  `python-version: "3.x"` began resolving to 3.15 (libhmm PR 149). The
  job checks changed files only, so the crash stayed latent until a PR
  touched a CMake file.

| Repo | Config | Hook installed (Windows machine) | Runs in CI | `.gitattributes` |
|---|---|---|---|---|
| libhmm | yes | yes | yes, changed files | `eol=lf` |
| libstats | yes | no | no | none |
| corvus | no | no | no | `eol=lf` |
| ewcalc | no | no | no | none |
| pylibstats | no | no | no | none |
| pylibhmm | no | no | no | none |

Hook state on the two Macs was not checked.

## 2. Proposed standard

1. **One `.pre-commit-config.yaml` per repo, run in two places.** The
   local hook gives feedback before the commit exists. A CI job running
   the same config is the gate, because the local hook cannot be relied
   on to be installed.
2. **Install at session start.** `SESSION-START.md` gains one step:
   `python -m pre_commit install` (and `--hook-type pre-push` where the
   repo uses that stage). Call it as a module; a `pre-commit` executable
   on `PATH` is not guaranteed.
3. **Base hooks in every repo**, from `pre-commit/pre-commit-hooks`:
   `trailing-whitespace`, `end-of-file-fixer`, `check-yaml`, `check-json`,
   `check-merge-conflict`, `check-case-conflict`,
   `check-added-large-files`, `mixed-line-ending --fix=lf`. Plus
   `actionlint`, which today runs only in CI.
4. **Language hooks where the repo already has the config file.**
   clang-format where `.clang-format` exists; ruff where `pyproject.toml`
   configures it. A formatter is never introduced by this standard alone
   (see §4).
5. **Every hook `rev` is pinned, and CI names its Python.** Same
   reasoning as CI House Style §1 and §5. The CI job uses an explicit
   `python-version`, never `"3.x"`.
6. **One pin per tool.** Where CI already installs a pinned linter
   (pylibstats and pylibhmm pin ruff in `ci.yml`), the CI lint step calls
   `pre-commit run` instead, so the hook `rev` is the only pin.
7. **Commit stage stays fast.** Formatters and file checks run at
   `pre-commit`. Anything that takes more than a few seconds (shellcheck
   over many scripts, cppcheck, mypy) runs at `pre-push` or stays
   CI-only.
8. **CI runs changed files on push and PR, all files on the canary.**
   The weekly scheduled run uses `--all-files`, so a hook that breaks
   with no source change fails on schedule instead of on the next PR that
   happens to touch the right file type.
9. **Project hooks are Python, not shell.** `language: python` runs on
   every fleet machine; `language: script` with a `.sh` entry needs Git
   Bash on each Windows clone.
10. **`.gitattributes` with `* text=auto eol=lf` is a precondition.**
    Four repos have none and check out CRLF on Windows, where
    `mixed-line-ending --fix=lf` would rewrite the working tree on every
    run.

## 3. Rollout, cheapest first

Each step is its own PR, and each PR's CI is the test.

1. **`.gitattributes` in libstats, ewcalc, pylibstats, pylibhmm.**
   No content change in the repository, only in Windows working copies.
   ewcalc needs `eol=crlf` exceptions for the files Visual Studio and
   PowerShell own (`*.sln`, `*.vcxproj`, `*.ps1`); list them before
   committing.
2. **pylibstats and pylibhmm: new config.** Base hooks plus ruff check
   and ruff format. Replace the CI `pip install ruff==…` step with the
   pre-commit job. mypy and `scripts/lint-cpp.sh` stay as CI steps: both
   need the built environment.
3. **libhmm: bring the reference up to the standard.** Add `--all-files`
   on the scheduled run, add the missing base hooks, and bump
   `pre-commit-hooks` (v4.6.0; libstats is on v6.0.0). Already pinned to
   Python 3.14.
4. **libstats: make the existing config real.**
   - Run `pre-commit run --all-files` once and measure the diff before
     deciding anything. If it is large, land it as one formatting commit
     and add that commit to `.git-blame-ignore-revs`.
   - Add the CI job; delete the `clang-format-17 … || true` step it
     replaces.
   - Port the four `scripts/check-*.sh` and `validate-includes.sh` hooks
     to Python, or accept Git Bash as a Windows prerequisite (§4).
   - Confirm `.secrets.baseline` exists and is current, or drop
     `detect-secrets`.
5. **corvus and ewcalc: base hooks only.** No formatter until §4 is
   decided.
6. **Then** move §2 into CI-HOUSE-STYLE.md §6, add the install step to
   SESSION-START.md, and retire this record.

## 4. Decisions for the owner

- **[OPEN] clang-format in corvus and ewcalc.** Neither has a
  `.clang-format`. Adopting one costs a whole-codebase reformat commit
  per repo. The base hooks do not depend on it.
- **[OPEN] `cmake-format`.** The hook (cmakelang 0.6.13) has no newer
  release and crashes on Python 3.15. Options: keep it behind a pinned
  hook Python, replace it, or stop formatting CMake. No replacement has
  been evaluated.
- **[OPEN] libstats project hooks: port to Python or require Git Bash.**
  Porting is four small scripts; requiring Git Bash is zero work now and
  a recurring setup step.
- **[OPEN] Hook-version updates.** Dependabot does not update
  `.pre-commit-config.yaml` in the current fleet configuration. Whether
  it can was not checked; the alternative is `pre-commit autoupdate` by
  hand when the canary flags a problem.
- **[OPEN] One active record.** `AGENTS.md` allows one ACTIVE record,
  currently `CORVUS-ADOPTION-WORKPLAN.md`. If this is adopted, either it
  waits, or that rule is relaxed for a second, unrelated effort.

## 5. Costs to expect

- **A fixing hook aborts the commit.** The fix is re-stage and commit
  again; with signed commits that is a second YubiKey signature.
- **First run per machine is slow.** pre-commit builds each hook's
  environment once and caches it.
- **Not proposed:** commit-message hooks, Markdown linting, and any hook
  that needs a built tree.
