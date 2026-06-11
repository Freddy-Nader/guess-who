# Plan 007: Spike — evaluate replacing random character generation with a fixed character set

> **Executor instructions**: This is a **design spike, not a build plan**. The
> deliverable is a written design document plus a prototype data table — no
> production code changes. Follow the steps; if anything in the "STOP
> conditions" section occurs, stop and report. When done, update the status
> row for this plan in `plans/README.md` AND record the spike's verdict there
> (it gates plan 005).
>
> **Drift check (run first)**: `git diff --stat f1df899..HEAD -- .src/Data.class .src/Plantilla.class`
> Drift from plans 001–005 is fine; this spike reads code for context only.

## Status

- **Priority**: P2 (its verdict gates plan 005 — run the spike before investing in 005)
- **Effort**: M (coarse — direction work estimates are softer than fix estimates)
- **Risk**: LOW (no production code is touched)
- **Depends on**: none
- **Category**: direction
- **Planned at**: commit `f1df899`, 2026-06-11

## Why this matters

The classic Guess Who board game uses a **fixed cast of 24 distinct, named characters**. This implementation instead generates 24 random characters every game (`Data.asignacion()`), which causes concrete problems: duplicate names and even fully identical characters appear (≈99% chance of a name collision per game — see plan 005, which patches this), the visual identity of each character is limited to mechanically layered feature sprites (`img/pb*.png`, `img/pelo*.png`, …), and the game cannot ship recognizable characters or difficulty-tuned boards. A fixed character set would make boards deterministic, guaranteed-distinguishable, art-friendly, and would delete the entire random-generation/uniqueness machinery. The trade-off is losing per-game variety. This spike produces the evidence to decide — it does NOT make the change.

## Current state (read-only context)

Gambas `.class` files are **plain text source**; use `cat` if a tool refuses the extension.

- `.src/Data.class` — `caracteristicas()` defines 10 characteristics (`car[10, 4]`: name + up to 3 option strings) and two 48-name pools; `asignacion()` randomly fills `personajes[24, 11]` (10 trait columns + name) per game, picks `oculto[2]`, initializes `mock[2, 24]`.
- `.src/Plantilla.class` — `drawPersonaje()` renders a character by layering 7 picture boxes (`piel`, `ropa`, `ojos`, `pecas`, `rasgos`, `pelo`, `acc`) chosen by matching trait strings against `car[]`; `.src/Ocultos.class` `imagen()` duplicates this for the hidden-character reveal screen.
- `img/` contains the layer sprites: `pb1{1-3}` (piel), `pb2{1-3}` (ropa), `ojos{1-2}{1-3}` (forma×color), `pb5{1-2}` (pecas), `pb6{1-2}` (rasgos), `pelo{1-2}{1-3}` (tipo×color), `pb9{1-2}` (accesorios), `empty.png`.
- Plan 005 (`plans/005-unique-characters.md`) is the tactical alternative: keep random generation, add uniqueness constraints. If this spike's verdict is "adopt fixed set", 005 becomes unnecessary.
- Game rules constraint visible in the data: every characteristic needs each of its options represented well enough that questions are informative on a 24-character board (classic Guess Who balances option frequencies deliberately).

## Commands you will need

| Purpose | Command | Expected on success |
|---------|---------|---------------------|
| Read sources | `cat .src/Data.class`, `cat .src/Plantilla.class` | plain text |
| Place deliverables | files under `plans/spikes/` | — |

## Scope

**In scope** (files you may create):
- `plans/spikes/fixed-character-set.md` (the design document)
- `plans/spikes/personajes-fijos.csv` (the prototype data table)

**Out of scope** (do NOT touch):
- Any file in `.src/`, `img/`, or the repo root. This spike changes zero production files.

## Git workflow

- Branch: `advisor/007-spike-fixed-character-set`
- Single commit: `docs(plans): spike on fixed character set`.
- Do NOT push or open a PR unless the operator instructed it.

## Steps

### Step 1: Design the fixed cast

Produce `plans/spikes/personajes-fijos.csv` with 24 rows and columns matching `personajes[24, 11]` semantics: `genero, piel, ropa, forma_ojos, color_ojos, pecas, rasgos, tipo_cabello, color_cabello, accesorios, nombre`. Constraints to satisfy (justify each in the doc):

- All 24 trait vectors pairwise distinct; all names distinct.
- Option strings drawn **exactly** from the existing `car[]` values in `Data.class` (so the rendering code keeps working unmodified), respecting the 2-option characteristics (género; forma de los ojos per plan 002) and the no-facial-hair-for-women rule (`asignacion()` line ~241).
- Question balance: for each characteristic, no option held by fewer than ~5 or more than ~14 characters (so first questions are informative either way). State the actual distribution you chose.

### Step 2: Write the design document

`plans/spikes/fixed-character-set.md` must cover:

1. **Loading strategy** — recommend one and sketch it: (a) hard-coded assignments in `caracteristicas()` style, (b) CSV shipped in the project loaded with `gb` file functions, or (c) a Gambas module with the table as code. Weigh: diff-friendliness, translator-friendliness (names!), zero-new-components preference (`.project` currently needs only gb.image/gb.gui/gb.form).
2. **What gets deleted** — enumerate the code made obsolete: the random-assignment loop, the uniqueness machinery from plan 005 (if landed), the name pools (names move into the table). Estimate net line delta.
3. **What stays random** — `oculto[]` (hidden character per player) must stay random per game; confirm nothing else needs randomness.
4. **Variety trade-off** — honest paragraph: fixed cast means every game has the same 24 faces (like the physical game). Mitigations worth listing (multiple cast files, random subset of a larger pool — note that a 24-of-N subset reintroduces balance questions).
5. **Art implications** — with a fixed cast, per-character full portraits become possible (replacing 7-layer composition); flag asset cost and that it is strictly optional/later.
6. **Effect on other plans** — explicit verdict line: `VERDICT: adopt | reject | defer`, and what it means for plan 005 (REJECTED if adopt) and plan 006 (orthogonal — note any interaction with `drawPersonaje`).
7. **Open questions for the maintainer** — anything genuinely unresolvable from the repo (e.g. is per-game variety a feature the author values?).

### Step 3: Record the verdict in the plans index

Update `plans/README.md`: status row for 007 → DONE, and add the verdict line to the "Dependency notes" section so the executor of 005 sees it.

## Test plan

Not applicable (no production code). Quality gate: the CSV satisfies the Step 1 constraints — include a short table in the doc showing per-characteristic option counts as proof.

## Done criteria

- [ ] `plans/spikes/fixed-character-set.md` exists and covers all 7 sections of Step 2.
- [ ] `plans/spikes/personajes-fijos.csv` has 24 unique rows using only existing `car[]` option strings.
- [ ] `git status` shows only the two new files under `plans/spikes/`.
- [ ] `plans/README.md` updated with status + verdict.

## STOP conditions

Stop and report back if:

- The existing `car[]` option strings can't express a balanced 24-cast under the constraints (would mean the characteristic system itself needs redesign — that is a finding, not something to improvise around).
- You are tempted to modify `.src/` "while you're in there" — out of scope, full stop.

## Maintenance notes

- If the verdict is "adopt", the follow-up build plan should be written fresh against the then-current code (post 002/005/006), not improvised from this spike.
- The i18n direction noted in `plans/README.md` interacts here: character names in a data table are easier to localize than names baked into `caracteristicas()`.
