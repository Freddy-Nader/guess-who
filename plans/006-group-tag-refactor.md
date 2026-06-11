# Plan 006: Collapse copy-paste event handlers and array wiring with Gambas Group/Tag idiom

> **Executor instructions**: Follow this plan step by step. Run every
> verification command and confirm the expected result before moving to the
> next step. This plan is **incremental by design**: each conversion is
> proven on one control before being applied to the batch. If anything in
> the "STOP conditions" section occurs, stop and report — do not improvise.
> When done, update the status row for this plan in `plans/README.md` —
> unless a reviewer dispatched you and told you they maintain the index.
>
> **Drift check (run first)**: `git diff --stat f1df899..HEAD -- .src/Plantilla.class .src/Plantilla.form .src/Data.class`
> Changes from plans 001–005 are expected; this plan assumes plan 002's
> version of `MenuClicks2`/`adivinar()`. If `Plantilla.class` matches neither
> the original nor the post-002 shape, treat it as a STOP condition.

## Status

- **Priority**: P3
- **Effort**: L
- **Risk**: MED (touches the form file and all event wiring; no automated tests — verification is compile + a full manual play-through)
- **Depends on**: plans/001-ci-compile-baseline.md (compile gate), plans/002-fix-elimination-engine.md (same file)
- **Category**: tech-debt
- **Planned at**: commit `f1df899`, 2026-06-11

## Why this matters

`.src/Plantilla.class` is 833 lines, of which roughly 500 are mechanical repetition: 24 one-line `ButtonN_Click` handlers, ~40 one-line `carXY_Click` menu handlers, and a 300-line `macro()` procedure that hand-assigns every named control into arrays (`botones[24]`, `pb[7,25]`, `inferior[24]`, `menus1/2/3`). Gambas has first-class idioms for exactly this: the **Group** property routes events from many controls to one `<Group>_<Event>` handler with `Last` identifying the sender, **Tag** carries per-control data, and controls can be looked up by iterating `Me.Children` and matching `.Name`. Applying them cuts the file to roughly 250 lines and makes every future change (new characteristic, different board size) a data change instead of 30 synchronized edits. Also included: removing a fragile hack in `tachar()` that toggles global player state twice per loop iteration just to peek at the opponent's board.

## Current state

Gambas `.class`/`.form` files are **plain text source**; if a tool refuses them by extension, use `cat`/`sed`.

### The repetition being removed (all in `.src/Plantilla.class`, line numbers at planned-at commit)

- Lines 41–70: ten `carX0_Click` handlers, each `MenuClicks1(X)`.
- Lines 110–208: thirty `carXY_Click` handlers (X=0..9 characteristic, Y=1..3 option), each `MenuClicks2(Y, X, Y - 1)`. Note `op0` (género) has only Y=1..2 and `op3` (forma de los ojos) only Y=1..2 in the form; `car03_Click`/`car33_Click` exist in the class but have **no corresponding menu item** — they are dead code and confirm deletion is safe.
- Lines 344–415: twenty-four `ButtonN_Click` handlers, each `MenuClick3(N - 1)`.
- Lines 444–741: `macro()` — assigns `botones[i] = Button{i+1}`, `menus1[..]`, `menus2[..]`, `pb[j,i] = {piel|ropa|ojos|pecas|rasgos|pelo|acc}{i+1}`, `inferior[i] = inf{i+1}`.
- `menus2[10]` and `menus3[10]` fields (lines 16–18): `menus2` is filled in `macro()` but **never read**; `menus3` is never even filled. Both delete.
- `menus1[10, 4]` is filled but never read either (menus are referenced by name string: `MenuButton1.Menu = Data.menu[i]`). Delete.

### `tachar()` hack (`.src/Plantilla.class:293-309`)

```
Public Sub tachar()
    Dim i As Integer
    Dim j As Byte
    
    For i = 0 To 23
        If Data.mock[Data.jugador(), i] = "*" Then
            pb[0, i].Picture = Picture["img/no.png"]
            For j = 1 To 6
                pb[j, i].Picture = Picture["img/empty.png"]
            Next
        Endif 
        
        Data.oponenteL()
        inferior[i].Picture = Picture[IIf(Data.mock[Data.jugador(), i] = "*", "img/no.png", "img/guess.png")]
        Data.oponenteL()
    Next
End
```

`Data.oponenteL()` toggles the global `Data.oponente` boolean; the pair of calls makes the middle line read the **other** player's board (48 global toggles per call). In `Data.class`: `jugador()` returns `If(oponente, 0, 1)` and `oponenteN()` returns `If(oponente, 1, 0)` — so inside the toggle pair, `Data.jugador()` equals the untoggled `Data.oponenteN()`. The behavior-preserving replacement is `Data.mock[Data.oponenteN(), i]` with no toggles.

### Form facts (`.src/Plantilla.form`)

- Character buttons are `Button1`..`Button24`; `Button25` (adivinanza directa), `Button26` (finalizar), `Button27` (continuar) are functional buttons that must NOT join the group.
- Menus: `Menu1` (10 items `car00`..`car90`) and `op0`..`op9` (items `carX1`..`carX3`, except op0/op3 which have 2). Menu items in Gambas **do support the Group property**, but serialization of hand-edited form files is risky — that is why every form edit in this plan is proven on one control first.
- Form property syntax, as already present in the file (excerpt, lines 163–168):

```
  { Button1 Button
    MoveScaled(38,5,17.1429,5.1429)
    Enabled = False
    Font = Font["+2,Italic"]
    ...
  }
```

  A Group assignment is one more property line inside the braces: `Group = "Botones"`. A Tag line: `Tag = 0`.

- Picture boxes are named `{piel|ropa|ojos|pecas|rasgos|pelo|acc}1`..`25` (index 25 = the hidden-character preview), bottom row `inf1`..`inf24`.

### Gambas idioms used (so you don't have to guess syntax)

- Event group: control property `Group = "Botones"` → events fire `Botones_Click()`; inside it, `Last` is the sending control.
- `Last.Tag` reads the sender's Tag (a Variant; store an Integer).
- Child iteration: `For Each c In Me.Children` (`Dim c As Control`); `c.Name` is the design name. **`Me.Children` only enumerates direct children** — if controls sit inside a container (Panel/HBox), recursion is needed; see STOP conditions.
- String helpers: `Left$(s, n)`, `Mid$(s, start)`, `CInt(s)`, `s Like "piel[0-9]*"`.

### Repo conventions

Spanish identifiers/comments, 4-space indent, banner comments `'-----`, conventional commits.

## Commands you will need

| Purpose | Command | Expected on success |
|---------|---------|---------------------|
| Compile (Linux + Gambas only) | `gbc3 -ag .` from repo root | exit 0 |
| Compile (otherwise) | CI `build / compile` job (plan 001) on push | green |
| Run for manual test (Linux GUI) | `gbx3 .` | game opens |
| Line count progress | `wc -l .src/Plantilla.class` | shrinking |

**This plan should not be executed without access to a Linux GUI environment for manual testing** (locally or by the operator). If you have none, do the work, pass the compile gate, and flag the manual-test checklist as pending in your report — do not mark the plan DONE.

## Scope

**In scope** (the only files you should modify):
- `.src/Plantilla.class`
- `.src/Plantilla.form`

**Out of scope** (do NOT touch):
- `.src/Data.class`, `.src/Ocultos.class` (Ocultos has a similar but much smaller `imagen()`; consolidating it is deferred — see Maintenance notes).
- Game logic semantics: `MenuClicks1/2/3`, `adivinar()`, `opciones()`, `respaldar()`, `setup()`, `drawPersonaje()` bodies stay functionally identical (only how events reach them and how arrays are filled changes).
- `Inicio.class:10` calls `Plantilla.Form_Open()` directly — that entry point must keep working.

## Git workflow

- Branch: `advisor/006-group-tag-refactor`
- One commit per step; e.g. `refactor(plantilla): route character buttons through a Botones event group`.
- Do NOT push or open a PR unless the operator instructed it.

## Steps

### Step 1: Prove Group serialization on one button

In `.src/Plantilla.form`, add two property lines inside the `{ Button1 Button ... }` block:

```
    Group = "Botones"
    Tag = 0
```

In `.src/Plantilla.class`, add (do not delete anything yet):

```
Public Sub Botones_Click()
    MenuClick3(Last.Tag)
End
```

and change `Button1_Click()` to do nothing temporarily? **No** — instead delete only `Button1_Click()` (its event now routes to the group handler).

**Verify**: compile passes. If a GUI is available: start a game, use "adivinanza directa" (Button25), click the first character button → the win/lose message must appear exactly as before. If this single-button trial fails to fire, STOP (Group serialization assumption wrong — the form edit needs the Gambas IDE).

### Step 2: Convert all 24 character buttons

- Add `Group = "Botones"` and `Tag = <N-1>` to `Button2`..`Button24` in the form (Tag 1..23).
- Delete `Button2_Click()`..`Button24_Click()` from the class (lines 347–415 region).
- Delete the `botones[24] As Object` fill from `macro()` — but `botones` is still *used* in `Button25_Click` (`botones[i].Enabled = True`) and `setup()` (`botones[i].Text = ...`). Keep the array; fill it generically in Step 4.

**Verify**: compile passes; `grep -c "_Click" .src/Plantilla.class` dropped by 23; `grep -n "MenuClick3(" .src/Plantilla.class` → exactly 2 matches (definition + group handler).

### Step 3: Convert the menu items

Same pattern, two groups:

- `Menu1`'s children `car00`..`car90`: add `Group = "CarGrupo"` and `Tag = <X>` (0..9) to each in the form; delete the ten `carX0_Click` handlers; add:

```
Public Sub CarGrupo_Click()
    MenuClicks1(Last.Tag)
End
```

- All `carXY` items under `op0`..`op9` (Y ≥ 1): add `Group = "CarOpcion"` and `Tag = <X * 10 + Y>`; delete the thirty `carXY_Click` handlers **plus** the dead `car03_Click` and `car33_Click`; add:

```
Public Sub CarOpcion_Click()
    Dim t As Integer = Last.Tag

    MenuClicks2(t Mod 10, t \ 10, (t Mod 10) - 1)
End
```

  (Matches the existing hard-coded calls: `carXY_Click` → `MenuClicks2(Y, X, Y - 1)`.)

Do Step 3 as two sub-commits, each proven on one menu item first (same trial technique as Step 1).

**Verify**: compile passes; `grep -cn "Public Sub car" .src/Plantilla.class` → 0; full question flow works in GUI (pick characteristic → pick option → elimination happens).

### Step 4: Replace `macro()` with name-based lookup

Replace the entire `macro()` body with:

```
Public Sub macro()
    Dim c As Control
    Dim prefijos As String[] = ["piel", "ropa", "ojos", "pecas", "rasgos", "pelo", "acc"]
    Dim j As Integer
    Dim n As Integer

    For Each c In Me.Children
        If c.Name Like "Button[0-9]*" Then
            n = CInt(Mid$(c.Name, 7))
            If n <= 24 Then botones[n - 1] = c
            Continue
        Endif
        If c.Name Like "inf[0-9]*" Then
            inferior[CInt(Mid$(c.Name, 4)) - 1] = c
            Continue
        Endif
        For j = 0 To 6
            If c.Name Like prefijos[j] & "[0-9]*" Then
                pb[j, CInt(Mid$(c.Name, Len(prefijos[j]) + 1)) - 1] = c
                Break
            Endif
        Next
    Next
End
```

Delete the `menus1`, `menus2`, `menus3` field declarations (lines 16–18) and any remaining references (they were only written, never read — verify first with `grep -n "menus" .src/Plantilla.class`).

Prefix-collision check (done during planning, re-verify): no control name is a prefix-extension trap — `inf` does not collide (`Plantilla.form` has no control named `info*`), `pelo`/`pecas`/`piel` are mutually exclusive matches. Confirm with: `grep -o "{ [A-Za-z0-9]* " .src/Plantilla.form | sort -u`.

After `macro()` runs, every cell of `botones[0..23]`, `inferior[0..23]`, `pb[0..6, 0..24]` must be non-Null. Add a temporary assertion loop at the end of `macro()` during testing if helpful, and remove it before finishing.

**Verify**: compile passes; GUI: open a fresh game — all 25 character portraits render (24 board + 1 hidden preview), bottom `inf` row shows `guess.png`. `wc -l .src/Plantilla.class` ≈ 250–350.

### Step 5: Fix `tachar()`

Replace the toggle pair (see "Current state") so the middle line reads the opponent directly:

```
        inferior[i].Picture = Picture[IIf(Data.mock[Data.oponenteN(), i] = "*", "img/no.png", "img/guess.png")]
```

and delete both surrounding `Data.oponenteL()` calls inside the loop.

**Verify**: `grep -c "oponenteL" .src/Plantilla.class` → 0 in `tachar()` (one call remains in `Button27_Click` — that one is the real turn switch, keep it). GUI: after player 1 eliminates characters and passes the turn, the bottom row on player 2's screen reflects player 1's crossed-out characters exactly as before the refactor.

### Step 6: Full manual regression play-through

With the GUI: play one complete game covering — characteristic question (hit and miss), repeat-question rejection, direct guess (right and wrong, two separate runs), win by elimination if reachable, "finalizar" button, turn handoff via Button27 → Inicio → back. Every behavior must match the pre-refactor game (same messages, same crossed-out portraits, same score progression).

## Test plan

No automated harness (Gambas). The plan's safety comes from: compile gate after every step, one-control trial before each batch conversion, and the Step 6 regression checklist. Keep each step a separate commit so any regression bisects to one mechanical transformation.

## Done criteria

- [ ] Compile gate passes.
- [ ] `grep -c "Public Sub car\|Public Sub Button[0-9]*_Click" .src/Plantilla.class` → 0 for car handlers; only `Button25/26/27_Click` remain among button handlers.
- [ ] `grep -n "menus1\|menus2\|menus3" .src/Plantilla.class` → no matches.
- [ ] `wc -l .src/Plantilla.class` ≤ 400.
- [ ] Step 6 checklist executed (or explicitly reported as pending — plan stays IN PROGRESS).
- [ ] `git status` shows only the two in-scope files modified.
- [ ] `plans/README.md` status row updated.

## STOP conditions

Stop and report back (do not improvise) if:

- The Step 1 single-button trial does not fire `Botones_Click` (Group property not honored when hand-added to the form text). The fallback is editing the form in the Gambas IDE — that needs a human.
- The compiler rejects `Group` or `Tag` lines in the form file.
- `For Each c In Me.Children` does not yield the buttons/picture boxes (controls nested in containers). Report which container; the recursive-walk extension changes `macro()`'s shape and should be reviewed.
- Any `pb`/`botones`/`inferior` cell is Null after Step 4's `macro()` (name scheme assumption broken — list the missing names).
- No GUI environment exists and the operator is unreachable: finish through Step 5, leave status IN PROGRESS with the Step 6 checklist pending.

## Maintenance notes

- `Ocultos.class:68-137` (`imagen()`) duplicates `Plantilla.drawPersonaje()` almost exactly; once this plan lands, extracting a shared drawing routine (e.g. into `Data` or a new module) is a natural S-effort follow-up — deliberately deferred to keep this plan single-file-pair.
- New characteristics or board sizes after this refactor: add the menu items with Group/Tag in the form and a row in `Data.car` — no class-code fan-out.
- Reviewer should scrutinize: Tag values (off-by-one between `ButtonN` and index `N-1`; `X*10+Y` encoding), and that `Button25/26/27` did NOT receive the Group property.
