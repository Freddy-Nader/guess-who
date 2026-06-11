# Plan 004: Fix the broken README run instructions and user-visible Spanish typos

> **Executor instructions**: Follow this plan step by step. Run every
> verification command and confirm the expected result before moving to the
> next step. If anything in the "STOP conditions" section occurs, stop and
> report — do not improvise. When done, update the status row for this plan
> in `plans/README.md` — unless a reviewer dispatched you and told you they
> maintain the index.
>
> **Drift check (run first)**: `git diff --stat f1df899..HEAD -- README.md .src/Plantilla.class .src/Data.class .src/Plantilla.form`
> Changes from plans 001–002 in the `.class` files are expected. If the
> excerpts below are missing, treat it as a STOP condition.

## Status

- **Priority**: P2
- **Effort**: S
- **Risk**: LOW
- **Depends on**: none (string changes are independent; see the one ordering note in Step 3)
- **Category**: docs
- **Planned at**: commit `f1df899`, 2026-06-11

## Why this matters

The README's setup section tells users to run `gb3 guess-who.gambas`. Both halves are wrong: no Gambas binary is named `gb3` (the interpreter components are `gbx3` to run a project from source, `gbr3` to run a compiled `.gambas` archive, `gbc3` to compile), and no `.gambas` file exists in the repository (`*.gambas` is gitignored — see `.gitignore:20`). Anyone who clones and follows the instructions hits a dead end on the very first command. Separately, two user-visible Spanish strings are misspelled in the game UI: "Puntación" (should be "Puntuación", shown on the score label every turn) and "Tes de la piel" (should be "Tez de la piel", shown in the characteristics menu).

## Current state

Gambas `.class` files are **plain text source**; if a tool refuses them by extension, use `cat`/`sed`.

**README.md:35-44** (the broken setup section):

```markdown
### Setup

```bash
# Clone the repository:
git clone https://github.com/Freddy-Nader/guess-who.git
cd guess-who

# Run the compiled Gambas application:
gb3 guess-who.gambas
```
```

**Typo sites** (line numbers at the planned-at commit; plan 002 restructures `MenuClicks2` so the `Puntación` occurrences may have moved — match by string, not line):

- `.src/Plantilla.class:81` — `Label3.Text = "Puntación: " & (puntacion[Data.jugador()] * 10)`
- `.src/Plantilla.class:753` — `Label3.Text = "Puntación: " & (puntacion[Data.jugador()] * 10)` (inside `setup()`)
- `.src/Data.class:48` — `car[1, 0] = "Tes de la piel"`
- `.src/Plantilla.form:14` — `Text = ("Tes de la piel")` (menu item `car10`)

Safety fact verified during planning: `car[i, 0]` (the characteristic *name*) is only ever displayed (`MenuButton1.Text = Data.car[i, 0]` in `Plantilla.class:35`, and the `Print`/`Debug` dump in `Data.class`); no game logic compares against it, so changing the string is safe. The option strings `car[i, 1..3]` ARE compared in logic — do not touch any of those.

The variable name `puntacion` (no typo fix) stays as-is: renaming identifiers is out of scope; only user-visible strings change.

## Commands you will need

| Purpose | Command | Expected on success |
|---------|---------|---------------------|
| Compile (Linux + Gambas only) | `gbc3 -ag .` from repo root | exit 0 |
| Compile (otherwise) | CI `build / compile` job (plan 001) on push | green |
| Residual check | `grep -rn "Puntación\|Tes de la piel\|gb3 " .src README.md` | no matches |

## Scope

**In scope** (the only files you should modify):
- `README.md`
- `.src/Plantilla.class` (two string literals)
- `.src/Data.class` (one string literal)
- `.src/Plantilla.form` (one menu text)

**Out of scope** (do NOT touch):
- Any `car[i, 1..3]` option string (logic compares them).
- The `puntacion` variable name.
- README badges/links section, LICENSE, `.project`.

## Git workflow

- Branch: `advisor/004-readme-and-typos`
- Two commits: `docs: fix run instructions in README setup section` and `fix(ui): correct Puntuación and Tez de la piel spellings`.
- Do NOT push or open a PR unless the operator instructed it.

## Steps

### Step 1: Fix the README setup section

Replace the `### Setup` block (README.md:35-44) with:

```markdown
### Setup

```bash
# Clone the repository:
git clone https://github.com/Freddy-Nader/guess-who.git
cd guess-who

# Run the project from source with the Gambas interpreter:
gbx3 .
```

Alternatively, open the project folder in the Gambas 3 IDE (`gambas3`) and press <kbd>F5</kbd>.
```

Keep the surrounding `### Prerequisites` section and the note about Linux support unchanged.

**Verify**: `grep -n "gb3 \|guess-who.gambas" README.md` → no matches. `grep -n "gbx3 ." README.md` → one match.

### Step 2: Fix "Puntación" → "Puntuación"

Replace **every** occurrence of the string `Puntación` in `.src/Plantilla.class` with `Puntuación` (there are 2 at the planned-at commit; if plan 002 ran first there may still be 2 — replace all found).

**Verify**: `grep -rn "Puntación" .src` → no matches; `grep -c "Puntuación" .src/Plantilla.class` → ≥ 2.

### Step 3: Fix "Tes de la piel" → "Tez de la piel"

Replace the occurrence in `.src/Data.class` (`car[1, 0] = "Tes de la piel"`) and in `.src/Plantilla.form` (`Text = ("Tes de la piel")`).

**Verify**: `grep -rn "Tes de la piel" .src` → no matches; `grep -rn "Tez de la piel" .src` → exactly 2 matches (one in `Data.class`, one in `Plantilla.form`).

### Step 4: Compile

Compile per plan 001 (locally on Linux, else CI on operator push).

## Test plan

String-only changes; the compile gate plus the greps above are sufficient. If a GUI run is available: start the game, confirm the score label reads "Puntuación: 0" and the characteristics menu shows "Tez de la piel".

## Done criteria

- [ ] `grep -rn "Puntación\|Tes de la piel" .src README.md` → no matches.
- [ ] `grep -n "gb3 " README.md` → no matches.
- [ ] Compile gate passes.
- [ ] `git status` shows only the four in-scope files modified.
- [ ] `plans/README.md` status row updated.

## STOP conditions

Stop and report back (do not improvise) if:

- `grep -rn "Tes de la piel" .src` returns matches in files other than `Data.class`/`Plantilla.form` (drift — list them, then proceed only on the two known sites).
- Editing `.src/Plantilla.form` breaks the compile (form files are structured text the IDE regenerates; a stray character corrupts them). On a form-compile error, revert the form change, report, and leave the form edit to someone with the Gambas IDE.

## Maintenance notes

- The README also links an `en` (English) branch; its copy of these strings is a separate codebase and is NOT updated by this plan. See the i18n direction note in `plans/README.md` — translating via `.lang` files would eliminate this class of duplication.
- Other Spanish accents are imperfect elsewhere (e.g. `"Cafe"` for eye color at `Data.class:69`) but those strings are **compared by game logic**; fixing them safely requires changing assignment and comparison together and was deliberately left out of this plan.
