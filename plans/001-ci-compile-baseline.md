# Plan 001: Establish a CI compile gate for the Gambas project

> **Executor instructions**: Follow this plan step by step. Run every
> verification command and confirm the expected result before moving to the
> next step. If anything in the "STOP conditions" section occurs, stop and
> report — do not improvise. When done, update the status row for this plan
> in `plans/README.md` — unless a reviewer dispatched you and told you they
> maintain the index.
>
> **Drift check (run first)**: `git diff --stat f1df899..HEAD -- .src/Data.class .github/`
> If any in-scope file changed since this plan was written, compare the
> "Current state" excerpts against the live code before proceeding; on a
> mismatch, treat it as a STOP condition.

## Status

- **Priority**: P1
- **Effort**: S/M
- **Risk**: LOW
- **Depends on**: none
- **Category**: dx / tests
- **Planned at**: commit `f1df899`, 2026-06-11

## Why this matters

This repository is a Gambas 3 desktop game ("Adivina Quién?", a Spanish Guess Who). It currently has **no way to verify it even compiles**: there is no CI, and the development machine is macOS where the Gambas toolchain (`gbc3`) is not installed (Gambas is Linux-first). Every other plan in `plans/` makes code changes to `.src/*.class` files; without a compile gate those changes land blind. There is also a concrete open question this plan resolves: `​.src/Data.class:39` declares a procedure as `Public caracteristicas()` — missing the `Sub` keyword — and we cannot currently tell whether the Gambas compiler accepts that.

## Current state

- The project root **is** the Gambas project (`.project` file at repo root). There is no `src/` wrapper directory; sources live in `.src/`.
- `.project` (repo root) declares the components the compiler needs:

```
# Gambas Project File 3.0
Title=Adivina Quién?
Startup=Lander
Version=0.0.2
Component=gb.image
Component=gb.gui
Component=gb.form
```

- `.src/Data.class:39` — the suspicious declaration (note: Gambas `.class` files are **plain text source**, not Java bytecode; if a file-reading tool refuses them by extension, use `cat`/`sed` from the shell):

```
Public caracteristicas() 
    ' caracteristicas
    ...
End
```

  Every other procedure in the repo uses `Public Sub name()` or `Public Function name() As Type` (see `.src/Data.class:14` `Public Sub Form_Open()` and `.src/Data.class:220` `Public Sub asignacion()`).

- There is no `.github/` directory.
- Commit message convention (from `git log`): conventional commits, e.g. `feat(docs): add link to English version in README.md navigation section`, `chore: Improved code in .class files, without changing the logic`.

## Commands you will need

| Purpose | Command | Expected on success |
|---------|---------|---------------------|
| Validate workflow YAML locally | `python3 -c "import yaml,sys; yaml.safe_load(open('.github/workflows/build.yml'))"` | exit 0, no output |
| Compile (only if on Linux with Gambas installed) | `gbc3 -ag .` from repo root | exit 0, no error lines |

Note: on this macOS machine `gbc3`/`gbx3`/`gbr3` are **not installed** (verified during planning). The real compile verification happens in GitHub Actions on first push. If `python3` lacks the `yaml` module, `ruby -ryaml -e "YAML.load_file('.github/workflows/build.yml')"` is an equivalent check.

## Scope

**In scope** (the only files you should modify/create):
- `.github/workflows/build.yml` (create)
- `.src/Data.class` (one line: the procedure declaration at line 39)

**Out of scope** (do NOT touch):
- Any other `.src/` file.
- `.gitignore`, `.project`, README — no badge additions in this plan.
- Any logic inside `caracteristicas()` — only its declaration line changes.

## Git workflow

- Branch: `advisor/001-ci-compile-baseline`
- Conventional commit messages, e.g. `ci: add Gambas compile workflow` and `fix(data): add missing Sub keyword to caracteristicas declaration`.
- Do NOT push or open a PR unless the operator instructed it.

## Steps

### Step 1: Fix the procedure declaration in Data.class

In `.src/Data.class`, change line 39 from:

```
Public caracteristicas() 
```

to:

```
Public Sub caracteristicas()
```

This is safe regardless of whether the old form was tolerated: `Public Sub` is the canonical Gambas syntax used everywhere else in this repo, and the matching `End` at line 214 is already present.

**Verify**: `grep -n "Public Sub caracteristicas()" .src/Data.class` → exactly one match at line 39. `grep -c "^End$" .src/Data.class` returns the same count as before the edit (the edit adds no `End`).

### Step 2: Create the GitHub Actions workflow

Create `.github/workflows/build.yml` with exactly this content:

```yaml
name: build

on:
  push:
  pull_request:

jobs:
  compile:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Gambas 3
        run: |
          sudo apt-get update
          sudo apt-get install -y gambas3

      - name: Compile project
        run: gbc3 -ag "$GITHUB_WORKSPACE"
```

Notes for the executor:
- The `gambas3` metapackage on Ubuntu pulls the compiler (`gbc3`) and all standard components including `gb.image`, `gb.gui`, `gb.form`. It is large (~2–3 min install) — that is acceptable; do not try to minimize the package set in this plan.
- `gbc3 -ag <dir>` compiles the project in `<dir>`: `-a` removes useless files / recompiles all, `-g` adds debugging information (matches what the Gambas IDE does).

**Verify**: `python3 -c "import yaml,sys; yaml.safe_load(open('.github/workflows/build.yml'))"` → exit 0.

### Step 3: Dry-run compile if (and only if) a Linux + Gambas environment is available

If you are executing on Linux, or the operator provides a container: run `gbc3 -ag .` from the repo root. Expected: exit 0 and no output lines containing `error:`. If you are on macOS (likely), skip — record in your report that compile verification is deferred to the first CI run.

## Test plan

No unit tests exist in this repo and Gambas testing infrastructure is out of scope here. The "test" is the CI job itself: on the first push of this branch, the `build / compile` job must pass. If the operator pushes and the job fails on the `gbc3` step, the failure log is the deliverable — attach it to your report.

## Done criteria

- [ ] `.github/workflows/build.yml` exists and parses as YAML (command in Step 2).
- [ ] `.src/Data.class:39` reads `Public Sub caracteristicas()`.
- [ ] `git status` shows no modified files outside the in-scope list.
- [ ] `plans/README.md` status row updated.

## STOP conditions

Stop and report back (do not improvise) if:

- `.src/Data.class:39` does not contain `Public caracteristicas()` (codebase drifted).
- A CI run is available to you and the `apt-get install -y gambas3` step fails (package missing on the runner image). Report the failure; the known fallback is adding the PPA `ppa:gambas-team/gambas3` before install, but confirm with the operator before changing the workflow.
- A CI run is available and `gbc3` reports compile errors in files **other than** `Data.class` — that means the codebase has pre-existing compile errors this plan didn't anticipate; report them as findings, do not fix them here.

## Maintenance notes

- Plans 002–006 all rely on this compile gate as their primary verification. If the workflow is renamed, update their "Commands you will need" sections.
- Future improvement deliberately deferred: attaching a compiled `.gambas` executable as a release artifact (see the direction notes in `plans/README.md`).
- Reviewer should scrutinize: that the workflow triggers on `pull_request` as well as `push`, and that Step 1 changed only one line.
