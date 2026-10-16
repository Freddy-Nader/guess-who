# Spike 007: Evaluate replacing random character generation with a fixed character set

**Status**: VERDICT: adopt  
**Branch**: `advisor/007-spike-fixed-character-set`  
**Committed at**: `6109010`  
**Date**: 2026-06-11

---

## 1. Loading strategy

**Recommendation: Hard-coded table as a Gambas module (option c)**

Three options were weighed:

| Option | Pros | Cons |
|--------|------|------|
| **(a) Hard-code in `caracteristicas()` style** — extend the existing sub with 24 explicit `personajes[i, j] = …` assignments | Zero new components; stays inside existing patterns; trivially diff-able | Long (240+ lines of assignments); mixing data with logic; hard to hand to a translator |
| **(b) CSV shipped and loaded at runtime with `gb` file functions** | Human-editable; translator-friendly; shortest code | Requires `gb.io` or `gb.settings`; adds an I/O failure path; complicates distribution (file must travel with binary); `.project` currently only uses `gb.image`, `gb.gui`, `gb.form` — adding `gb.io` is a new component |
| **(c) Gambas module with table as code** — a dedicated `Personajes.class` with a `Public Sub Load()` that populates `personajes[]` with the fixed 24 rows | Clean separation of data and logic; zero new components beyond a new class; diff-friendly (one row = one block); no I/O failure path | Still verbose; not editable without Gambas IDE |

**Rationale**: Option (c) wins on the zero-new-components constraint and the no-I/O-failure-path constraint. A `Personajes.class` housing a single `Load()` sub is trivially readable as plain text (Gambas `.class` files are UTF-8 source), diff-friendly, and drops in without touching `.project`. Option (a) is acceptable as a fallback if the reviewer prefers fewer files.

---

## 2. What gets deleted

Adopting a fixed character set makes the following code obsolete:

| Location | Code | Estimated lines |
|----------|------|-----------------|
| `Data.class` — `asignacion()` | The entire random-assignment loop (`For i = 0 To 23 … Next`) including `Repeat/Until` for genre, name, rasgos, and remaining traits | ~30 lines removed |
| `Data.class` — `caracteristicas()` | Both `nombres[0, *]` and `nombres[1, *]` blocks (48 + 48 entries) | ~100 lines removed |
| `Data.class` — declaration | `Public nombres[2, 48] As String` | 1 line removed |
| Plan 005 uniqueness machinery | The deduplication loop and collision-detection logic described in plan 005 — not yet landed, but **entirely obsoleted** (see §6) | ~20 lines (projected) |

`oculto[]` selection (2 random indices) stays — see §3.  
`mock[]` initialization loop stays.  
The debug `Print` block in `asignacion()` can be removed as a separate cleanup (plan 003 scope).

**Net line delta**: approximately −150 lines in `Data.class`, +~270 lines in new `Personajes.class` (24 characters × ~11 assignment lines each, plus sub boilerplate). Gross line count increases slightly, but **logic complexity decreases substantially**: zero randomness, zero collision risk, zero Repeat/Until loops.

---

## 3. What stays random

Only one thing must remain random per game:

- **`oculto[i]`** — the hidden character index for each player (lines in `asignacion()` after the main loop). This is a random draw from `[0, 23]` and must remain so; it is the core game mechanic.

Everything else in `asignacion()` can be replaced by deterministic table load. Specifically:
- Character traits — deterministic (fixed table).
- Character names — deterministic (fixed table).
- `mock[]` initialization — deterministic (always `"-"`).

No other source of randomness exists in the codebase.

---

## 4. Variety trade-off

**The honest case against adoption**: A fixed cast means every session of "Adivina Quién?" has the same 24 faces with the same names. Players who play frequently will memorize the board. The game loses the "who might show up today?" surprise that random generation provides. For a two-player competitive game played repeatedly, familiarity eventually allows skilled players to ask optimally-ordered yes/no questions based on memorized trait distributions rather than in-game deduction — this is not necessarily bad (it mirrors the physical board game) but it changes the skill ceiling.

**Mitigations worth considering** (none required for initial adoption):

1. **Multiple fixed decks** — ship 2–3 curated casts and let players choose or randomly select a deck at game start. Preserves determinism within a game while restoring inter-game variety. Costs ~3× the authoring effort.
2. **Seasonal / themed decks** — swap the `Personajes.class` for themed sets (pirates, historical figures, etc.). The fixed-table architecture makes this trivial.
3. **Shuffle presentation order** — even with a fixed cast, randomizing which slot (0–23) each character occupies each game preserves visual novelty without reintroducing trait collisions. This is a one-line change (`gb` array shuffle or index permutation).
4. **Do nothing** — the physical Guess Who board game has used the same 24 characters since 1979. Players still enjoy it. Familiarity is a feature for the target audience (casual / family play).

**Recommendation**: adopt the fixed cast, implement slot-shuffle (mitigation 3) in the same commit as it costs one line and preserves the feel of variety.

---

## 5. Art implications

The current renderer composes characters from **7 independent trait layers** (genre, skin, clothing, eye shape, eye color, freckles, facial features, hair type, hair color, accessories — the sprite stack in `drawPersonaje`). This architecture was necessary precisely *because* characters were random: you cannot pre-draw 24^N combinations.

With a fixed cast of 24, **per-character full portraits become feasible**:

- A single 24-image sprite sheet or 24 individual PNGs replaces the entire layered composition pipeline.
- `drawPersonaje` (or its equivalent in `Plantilla.form` / `Tablero.form`) reduces to a single `Image.Load` / `DrawImage` call indexed by character slot.
- No more layer-ordering bugs, color-blending artifacts, or z-ordering concerns.

**Asset cost**: 24 original character illustrations. This is non-trivial authoring work and is **entirely optional and deferred**. The fixed-cast code change is independent of the art pipeline change. Phase 1 can ship fixed trait data with the existing layered renderer. Phase 2 (whenever art is ready) can swap in portraits with a minimal code change.

**Flag**: plan 006 (`drawPersonaje` group-tag refactor) works on the layered renderer. If portraits are eventually adopted, plan 006 becomes moot. See §6.

---

## 6. Effect on other plans

**Plan 005 — Unique characters (deduplication)**  
VERDICT: **REJECTED** if this spike is adopted.  
Plan 005 adds collision-detection and retry logic to the random assignment loop so that no two characters share an identical trait vector. A fixed cast with 24 hand-verified distinct vectors (see `personajes-fijos.csv` — all 24 trait vectors confirmed unique by automated check) makes the problem plan 005 solves impossible to encounter. Landing plan 005 first and then adopting the fixed cast would result in immediate deletion of plan 005's additions. Do not invest in plan 005 if this spike is adopted.

**Plan 006 — Group-tag refactor of `drawPersonaje`**  
VERDICT: **ORTHOGONAL** in the short term, **potentially superseded** in the long term.  
Plan 006 refactors the layered sprite renderer for maintainability. If the fixed cast is adopted *without* portrait art (Phase 1), plan 006 remains independently valuable — the layered renderer still runs and plan 006 improves it. If portrait art is eventually adopted (Phase 2, §5), plan 006's refactored renderer is discarded anyway. Recommendation: land plan 006 only if there is near-term intent to maintain the layered renderer; defer it if portrait art is on a 6-month horizon.

---

## 7. Open questions for the maintainer

The following questions cannot be resolved from the repository alone:

1. **Target audience and replayability expectation** — Is "Adivina Quién?" intended for repeated play by the same players (familiarity matters) or one-off / classroom use (familiarity irrelevant)? This determines how much weight to give the variety trade-off in §4.

2. **Distribution format** — Is the game distributed as a compiled Gambas binary, as source, or via a package manager? If binary-only, a `Personajes.class` module is invisible to translators. If source distribution is expected, option (b) (CSV) becomes more attractive.

3. **Localization intent** — Are the Spanish trait strings and names intended to stay Spanish permanently, or is l10n planned? If l10n is planned, externalizing strings to a CSV (option b) or a resource file is preferable to hard-coded strings in either `caracteristicas()` or a new module.

4. **Art roadmap** — Is there any intent to commission or create per-character portrait art? The answer determines whether plan 006 is worth landing (§6).

5. **`car[3,3]` bug** — The source has `car[3, 3] = "Cuadrados"` (a duplicate of `car[3, 2]`). This spike treats `Forma de los ojos` as a 2-option characteristic per the plan's instruction. The bug should be tracked separately; confirm whether the fix is in scope for any open plan.

6. **Slot-shuffle** — The design recommends shuffling slot assignment each game (§4, mitigation 3). Confirm whether this is acceptable (it requires one use of `Rnd`) or if the maintainer prefers a fully deterministic board every game.

---

## Balance table

Per-characteristic option distribution across the 24 fixed characters (source: `personajes-fijos.csv`, verified by automated script).

| Characteristic | Option | Count | % |
|----------------|--------|-------|---|
| Género | Hombre | 12 | 50% |
| | Mujer | 12 | 50% |
| Tes de la piel | Blanca | 8 | 33% |
| | Negra | 8 | 33% |
| | Amarilla | 8 | 33% |
| Color de la ropa | Rojo | 8 | 33% |
| | Azul | 8 | 33% |
| | Negro | 8 | 33% |
| Forma de los ojos | Redondos | 13 | 54% |
| | Cuadrados | 11 | 46% |
| Color de los ojos | Azul | 7 | 29% |
| | Verde | 8 | 33% |
| | Cafe | 9 | 38% |
| Pecas | Pocas | 8 | 33% |
| | Muchas | 8 | 33% |
| | Ninguna | 8 | 33% |
| Rasgos faciales | Barba | 4 | 17% |
| | Bigote | 4 | 17% |
| | Ninguno | 16 | 67% |
| Tipo de cabello | Largo | 9 | 38% |
| | Corto | 8 | 33% |
| | Rapado | 7 | 29% |
| Color de cabello | Rubio | 8 | 33% |
| | Castaño | 9 | 38% |
| | Pelirrojo | 7 | 29% |
| Accesorios | Gafas | 8 | 33% |
| | Sombrero | 8 | 33% |
| | Ninguno | 8 | 33% |

**Notes on balance**:

- **Rasgos faciales** is the only structurally imbalanced characteristic: 16/24 characters have "Ninguno" because all 12 women are required to have no facial features. This is inherent to the game's design constraint and mirrors the original board game. The characteristic still eliminates half the board (asking "Does your character have a beard?" eliminates ~83% of characters if yes, ~17% if no — but the 12-Mujer group is a powerful deduction path via the Género question). The imbalance is acceptable and expected.
- All other characteristics fall within the 5–14 target range specified in the plan (actual range: 7–13, excluding the structurally-constrained Rasgos faciales).
- **Forma de los ojos** is 2-option by constraint (bug in `car[3,3]`); the 13/11 split is the closest achievable to 12/12 with a 24-row cast.
- The prototype data is in `plans/spikes/personajes-fijos.csv` and can be used directly as the source of truth for a `Personajes.class` implementation.
