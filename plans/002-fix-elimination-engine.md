# Plan 002: Fix the character-elimination engine (wrong option mapping, corrupt live-count, phantom scoring, duplicate eye option)

> **Executor instructions**: Follow this plan step by step. Run every
> verification command and confirm the expected result before moving to the
> next step. If anything in the "STOP conditions" section occurs, stop and
> report — do not improvise. When done, update the status row for this plan
> in `plans/README.md` — unless a reviewer dispatched you and told you they
> maintain the index.
>
> **Drift check (run first)**: `git diff --stat f1df899..HEAD -- .src/Plantilla.class .src/Data.class`
> (Exception: changes from plan 001 — `Public Sub caracteristicas()` at
> `Data.class:39` — are expected and fine.) If anything else changed, compare
> the "Current state" excerpts against the live code before proceeding; on a
> mismatch, treat it as a STOP condition.

## Status

- **Priority**: P1
- **Effort**: M
- **Risk**: MED (rewrites the core game loop; no automated tests exist — verification is compile + a scripted manual play-through)
- **Depends on**: plans/001-ci-compile-baseline.md
- **Category**: bug
- **Planned at**: commit `f1df899`, 2026-06-11

## Why this matters

This is a Gambas 3 implementation of Guess Who. The core mechanic — ask a question about a characteristic, eliminate non-matching characters — has four interlocking bugs that together make the game frequently play out wrong: a correct guess on a *third* menu option eliminates the wrong character groups (including the hidden character itself); the per-player "characters remaining" counter (`vivos`) drifts upward on every correct guess and is never restored after a rollback; a rejected question still awards points and permanently burns the question; and the eye-shape characteristic has two options with the identical string `"Cuadrados"`, which can wipe the entire board in one question. These bugs have existed since the first commit. After this plan, the elimination logic is correct for both 2-option and 3-option characteristics and the remaining-count is always the number of uncrossed characters.

## Current state

Gambas `.class` files are **plain text source** (not Java bytecode). If a file-reading tool refuses them by extension, use `cat`/`sed`/shell editing.

### Game model (needed to understand the fixes)

- `Data.personajes[24, 11]` (`.src/Data.class:3`) — one shared board of 24 randomly generated characters; column 0–9 are characteristic values, column 10 is the name. **Never mutated after generation.**
- `Data.car[10, 4]` (`.src/Data.class:8`) — `car[i, 0]` is the characteristic's display name; `car[i, 1..3]` are its option strings. Género (`car[0]`) has only 2 options (`car[0,3]` is unset).
- `Data.mock[2, 24]` (`.src/Data.class:6`) — per-player elimination board: `"-"` alive, `"*"` crossed out.
- `Data.vivos[2]` (`.src/Data.class:5`) — per-player count of alive characters, initialized to 24.
- `Data.oculto[2]` — index of each player's hidden character. `Data.jugador()` returns the current player (0/1), `Data.oponenteN()` the other one.
- In `.src/Plantilla.class`: `caracteristica` holds the characteristic index chosen from the first menu; `op` the option (1..3) chosen from the second menu. `chosen[2, 10, 3]` records which (player, characteristic, option) questions were already asked. `backup[24, 12]` (line 8) backs up state for rollback.

### Bug A — wrong group mapping for option 3 (`.src/Plantilla.class:228-247`)

```
    If Data.personajes[Data.oculto[Data.oponenteN()], caracteristica] = Data.car[caracteristica, op] Then
        Inc Data.vivos[Data.jugador()]
        
        Select Case op
            Case 1
                op1 = 2
                op2 = 3
            Case 2
                op1 = 1
                op2 = 3
            Case 3
                op1 = 2
                op2 = 3
        End
        
        opciones(op1)
        opciones(op2)
    Else 
        opciones(op)
    Endif
```

When the hidden character HAS the asked option, the other two option-groups must be eliminated. `Case 3` eliminates groups 2 and **3** — group 3 is the asked one (the hidden character's own group gets crossed out) and group 1 is never eliminated. Should be `1, 2`.

### Bug B — `vivos` corruption (`.src/Plantilla.class:229, 249, 274-287`)

1. The `Inc Data.vivos[Data.jugador()]` at line 229 (visible in the excerpt above) inflates the count by 1 on every correct guess. With the `Case 3` mapping fixed, the hidden character is never in an eliminated group, so no compensation is needed — the line is simply wrong.
2. Win check at line 249 compares the same value twice:

```
    If Data.vivos[Data.jugador()] = 1 Or Data.vivos[Data.jugador()] = 1 Then ' ganar por eliminación
```

3. `respaldar()` (lines 274–287) tries to restore `vivos` by diffing `Data.personajes` against the backup — but `personajes` is never mutated during play, so the diff never fires and `vivos` stays at 0 after a rollback while the board shows everyone alive:

```
Public Sub respaldar()
    Dim i As Byte
    Dim j As Byte
    
    For i = 0 To 23
        For j = 0 To 10
            If Data.personajes[i, j] <> backup[i, j] Then
                Inc Data.vivos[Data.jugador()]
            Endif
            Data.personajes[i, j] = backup[i, j]
        Next
        Data.mock[Data.jugador(), i] = backup[i, j]
    Next
End
```

(The `backup[i, j]` after the inner loop reads index `j = 11` — Gambas leaves the loop variable one past the end — which is where `adivinar()` stored the `mock` row; see `.src/Plantilla.class:221-226`.)

### Bug C — rejected question still scores and burns the slot (`.src/Plantilla.class:76-107`)

```
Public Sub MenuClicks2(i As Byte, j As Byte, k As Byte)
    op = i
    adivinar()
    
    Inc puntacion[Data.jugador()]
    Label3.Text = "Puntación: " & (puntacion[Data.jugador()] * 10)

    Data.restantes() ' consola
    
    If chosen[Data.jugador(), j, k] Or Data.vivos[Data.jugador()] = 0 Then
        If chosen[Data.jugador(), j, k] Then
            Message.Info("Esta característica ya fue seleccionada anteriormente. Vuelva a intentarlo.", "Ok")
        Else
            Message.Info("Todos los personajes comparten esta característica. Vuelva a intentarlo.", "Ok")
            respaldar()
        Endif

        MenuButton1.Text = "Características"
        MenuButton1.Menu = "Menu1"
        MenuButton1.Visible = True
        Button25.Enabled = True
        Button27.Visible = False
    Else
        MenuButton1.Visible = False
        MenuButton1.Text = "Características"
        Button25.Enabled = False
        Button27.Visible = True
    Endif
    
    tachar()
    chosen[Data.jugador(), j, k] = True
End
```

`adivinar()` and the score increment run **before** the validity checks; an already-asked question re-runs the elimination, and a rejected ("vuelva a intentarlo") question still scores a point and sets `chosen` so it can never be retried.

Here `j` is the characteristic index and `k = op - 1`; the click handlers (e.g. `car11_Click` → `MenuClicks2(1, 1, 0)`) hard-code these.

### Bug D — duplicate `"Cuadrados"` eye-shape option (`.src/Data.class:60-63`)

```
    ' forma de los ojos
    car[3, 0] = "Forma de los ojos"
    car[3, 1] = "Redondos"
    car[3, 2] = "Cuadrados"
    car[3, 3] = "Cuadrados"
```

Supporting facts, verified during planning: `img/` contains only `ojos1x.png` and `ojos2x.png` (no third eye shape), and the `op3` menu in `.src/Plantilla.form:77-85` has only two items (`car31`, `car32`). But `asignacion()` (`.src/Data.class:251-260`) assigns options 1..3 uniformly for all characteristics, so a third of characters get the *second* "Cuadrados". Consequence: a correct guess on "Cuadrados" (op=2) eliminates groups 1 and 3 — group 3 is the same string, so **every character on the board is eliminated**, which triggers the broken rollback of Bug B. The characteristic is really a 2-option one; `car[3, 3]` must go away and the assignment/elimination code must handle 2-option characteristics generically (género, `car[0]`, already has only 2 options — handled today by a special case at `.src/Data.class:229-232`).

The relevant assignment loop (`.src/Data.class:251-260`):

```
        ' resto de caracteristicas
        For j = 1 To 9
            Repeat
                r1 = Int(Rnd(1, 4))
            Until r1 <> 4
            personajes[i, j] = car[j, r1]
            
            If j = 5 Then
                Inc j
            Endif
        Next
```

(The `If j = 5 Then Inc j` skips `j = 6` — rasgos faciales — which is assigned earlier with a gender constraint at lines 241–248. The `Repeat/Until` guards are dead code — `Int(Rnd(1, 4))` already yields 1..3 — but leave the surrounding pattern intact except where this plan says otherwise.)

### Repo conventions

- Spanish identifiers and comments; 4-space indent; section banner comments like `'----------------------`. Match them.
- Conventional commit messages (see `git log --oneline`).

## Commands you will need

| Purpose | Command | Expected on success |
|---------|---------|---------------------|
| Compile (Linux + Gambas only) | `gbc3 -ag .` from repo root | exit 0, no `error:` lines |
| Compile (otherwise) | CI `build / compile` job from plan 001 on push | green |
| Locate code | `grep -n "<pattern>" .src/Plantilla.class` | line numbers as stated |

## Scope

**In scope** (the only files you should modify):
- `.src/Plantilla.class`
- `.src/Data.class`

**Out of scope** (do NOT touch, even though they look related):
- `.src/Plantilla.form` — the menu structure is already correct (op3 has 2 items).
- `.src/Ocultos.class` — its `imagen()` eye-drawing loop keeps working because option-2 matches before option-3 ever did; with `car[3,3]` empty it still matches options 1–2.
- The console `Print` statements in `Data.class` (`asignacion()` info dump, `restantes()`) — plan 003 handles those, and you may *use* their output to verify this plan.
- `tachar()`'s double `Data.oponenteL()` toggling (`Plantilla.class:305-307`) — behavior-preserving cleanup deferred to plan 006.

## Git workflow

- Branch: `advisor/002-fix-elimination-engine`
- One commit per step below; conventional messages, e.g. `fix(game): eliminate correct option groups on a hit`.
- Do NOT push or open a PR unless the operator instructed it.

## Steps

### Step 1: Make eye shape a true 2-option characteristic (`Data.class`)

1. Delete the line `car[3, 3] = "Cuadrados"` (`.src/Data.class:63`). Leave `car[3, 0..2]` as-is.
2. In the assignment loop at `.src/Data.class:251-260`, pick from 2 options when the characteristic has no third option. Replace the loop with:

```
        ' resto de caracteristicas
        For j = 1 To 9
            If j = 6 Then Continue ' rasgos faciales ya asignados arriba
            If car[j, 3] = "" Then
                r1 = Int(Rnd(1, 3)) ' característica de 2 opciones
            Else
                r1 = Int(Rnd(1, 4))
            Endif
            personajes[i, j] = car[j, r1]
        Next
```

   This also replaces the `If j = 5 Then Inc j` skip with an explicit `Continue` for `j = 6` — same effect (j=6 is assigned by the gender-constrained block above the loop), clearer. Do not touch the género block (`j = 0`, lines 228–232) or the rasgos block (lines 240–248).

**Verify**: `grep -c "Cuadrados" .src/Data.class` → `1`. `grep -n "Continue" .src/Data.class` → one match inside `asignacion()`. Compile gate passes (or is deferred to CI per plan 001 notes).

### Step 2: Fix the hit-branch elimination and remove the `vivos` inflation (`Plantilla.class`)

In `adivinar()` (`.src/Plantilla.class:214-257`):

1. Delete the line `Inc Data.vivos[Data.jugador()]` (line 229).
2. Replace the whole `Select Case op ... End` block plus the two `opciones(op1)` / `opciones(op2)` calls with a generic loop that eliminates every *other* non-empty option:

```
        For k = 1 To 3
            If k <> op And Data.car[caracteristica, k] <> "" Then opciones(k)
        Next
```

   Add `Dim k As Byte` to the declarations at the top of `adivinar()` and delete the now-unused `Dim op1 As Byte` and `Dim op2 As Byte`. This handles 2-option characteristics (género, forma de los ojos) and 3-option ones uniformly.
3. Fix the win check (line 249): replace

```
    If Data.vivos[Data.jugador()] = 1 Or Data.vivos[Data.jugador()] = 1 Then ' ganar por eliminación
```

   with

```
    If Data.vivos[Data.jugador()] = 1 Then ' ganar por eliminación
```

**Verify**: `grep -n "Select Case op" .src/Plantilla.class` → no match. `grep -n "Inc Data.vivos" .src/Plantilla.class` → no match. `grep -c "= 1 Or" .src/Plantilla.class` → 0.

### Step 3: Rewrite `respaldar()` and shrink the backup to what is actually backed up

`personajes` never changes during play, so backing it up is dead weight and the vivos-restore-by-diff can never work. Make the backup hold only the player's `mock` row and recount `vivos` after restoring:

1. Change the field declaration at `.src/Plantilla.class:8` from `backup[24, 12] As String` to `backup[24] As String`.
2. In `adivinar()`, replace the backup block (lines 220–226):

```
    ' respaldo
    For i = 0 To 23
        For j = 0 To 10
            backup[i, j] = Data.personajes[i, j]
        Next
        backup[i, j] = Data.mock[Data.jugador(), i]
    Next
```

   with:

```
    ' respaldo del tablero del jugador
    For i = 0 To 23
        backup[i] = Data.mock[Data.jugador(), i]
    Next
```

   Remove the now-unused `Dim j As Byte` from `adivinar()` if nothing else in the procedure uses `j` (after Step 2 it doesn't).
3. Replace the body of `respaldar()` (lines 274–287) with:

```
Public Sub respaldar()
    Dim i As Byte

    Data.vivos[Data.jugador()] = 0
    For i = 0 To 23
        Data.mock[Data.jugador(), i] = backup[i]
        If backup[i] <> "*" Then
            Inc Data.vivos[Data.jugador()]
        Endif
    Next
End
```

**Verify**: `grep -n "backup\[24\] As String" .src/Plantilla.class` → one match at the declaration. `grep -n "personajes\[i, j\] = backup" .src/Plantilla.class` → no match.

### Step 4: Reorder `MenuClicks2` so invalid questions neither score nor burn

Replace the entire `MenuClicks2` procedure (`.src/Plantilla.class:76-107`, excerpt in "Current state") with:

```
Public Sub MenuClicks2(i As Byte, j As Byte, k As Byte)
    op = i

    If chosen[Data.jugador(), j, k] Then
        Message.Info("Esta característica ya fue seleccionada anteriormente. Vuelva a intentarlo.", "Ok")
        reintentar()
        Return
    Endif

    adivinar()
    Data.restantes() ' consola

    If Data.vivos[Data.jugador()] = 0 Then
        Message.Info("Todos los personajes comparten esta característica. Vuelva a intentarlo.", "Ok")
        respaldar()
        reintentar()
    Else
        Inc puntacion[Data.jugador()]
        Label3.Text = "Puntación: " & (puntacion[Data.jugador()] * 10)
        chosen[Data.jugador(), j, k] = True

        MenuButton1.Visible = False
        MenuButton1.Text = "Características"
        Button25.Enabled = False
        Button27.Visible = True
    Endif

    tachar()
End

'----------------------------------------------
' restaurar la interfaz para repetir la pregunta
'----------------------------------------------
Private Sub reintentar()
    MenuButton1.Text = "Características"
    MenuButton1.Menu = "Menu1"
    MenuButton1.Visible = True
    Button25.Enabled = True
    Button27.Visible = False
End
```

Notes: the UI property assignments are copied verbatim from the original two branches — do not change which properties are set. The string `"Puntación: "` is intentionally kept as-is here (plan 004 fixes the typo repo-wide; keeping it identical avoids a conflict if 004 ran first — if the file already says `"Puntuación: "`, keep that spelling).

**Verify**: `grep -n "reintentar" .src/Plantilla.class` → 3 matches (two calls + one definition). `grep -n "chosen\[Data.jugador(), j, k\] = True" .src/Plantilla.class` → exactly one match, inside the `Else` branch.

### Step 5: Compile and manual play-through

Compile per plan 001 (locally if Linux, else via CI on operator push). Then, if a Linux GUI environment is available, run `gbx3 .` and execute this script (the console output from `Data.asignacion()` prints both hidden characters and their traits — use it as the oracle):

1. Note player 1's hidden character traits from the console.
2. Ask a question where the hidden character HAS the **third** option of a 3-option characteristic (e.g. accesorios = "Ninguno" if the console says so). Expected: characters with options 1 and 2 get crossed out; the hidden character stays.
3. Check the console output of `restantes()`: the printed remaining-count must equal the number of `-` cells in the printed matrix for that player. Repeat after several questions — the count must never exceed the number of uncrossed characters.
4. Ask "Forma de los ojos → Cuadrados" when the hidden character has square eyes. Expected: only round-eyed characters are eliminated; the board is NOT wiped.
5. Ask the same question twice. Expected: second attempt shows "ya fue seleccionada", the score does NOT increase, and the console matrix is unchanged.

If no GUI environment is available, record in your report which of these checks were not executed.

## Test plan

No test harness exists in this repo (Gambas; out of scope to introduce one). The manual play-through in Step 5 is the test plan; the console oracle (`asignacion()` dump + `restantes()` per turn) makes each check objective. Plan 003 will demote those prints to `Debug` — that is why 002 must execute before 003.

## Done criteria

Machine-checkable. ALL must hold:

- [ ] Compile gate passes (`gbc3 -ag .` exit 0 locally, or CI green after operator push).
- [ ] `grep -c "Cuadrados" .src/Data.class` → `1`.
- [ ] `grep -n "Select Case op\|Inc Data.vivos\|op1\|op2" .src/Plantilla.class` → no matches.
- [ ] `grep -n "backup\[24\] As String" .src/Plantilla.class` → one match.
- [ ] `git status` shows only `.src/Plantilla.class` and `.src/Data.class` modified.
- [ ] `plans/README.md` status row updated.

## STOP conditions

Stop and report back (do not improvise) if:

- Any "Current state" excerpt doesn't match the live code (beyond plan 001's `Sub` fix and plan 004's possible typo fixes).
- The compiler rejects `Continue` or the `<> ""` string comparison (would indicate an older Gambas dialect than assumed) — report the exact error, do not invent alternative syntax.
- You find any other writer to `Data.personajes` besides `asignacion()` (search: `grep -rn "personajes\[" .src/ | grep "="`). Step 3 assumes `personajes` is immutable during play; a second writer falsifies that.
- The manual play-through shows the board wiped on a "Cuadrados" hit after the fix.

## Maintenance notes

- The win-by-elimination check still lives inside `adivinar()` and closes all forms mid-flow while `MenuClicks2` continues to touch controls afterwards (pre-existing behavior, deliberately preserved). If a victory screen is ever added, move that check to the end of `MenuClicks2`.
- After this plan, the "Todos los personajes comparten esta característica" rollback should be unreachable in normal play (the hidden character always survives a correct elimination); it is kept as a defensive path. If it fires in testing, that is a bug — report it.
- Plan 005 (unique characters) and plan 006 (Group/Tag refactor) both edit `asignacion()` / `Plantilla.class` respectively; they were written against this plan's target state where noted in their drift checks.
- Reviewer should scrutinize: the generic elimination loop's `<> ""` guard (it is what protects 2-option characteristics), and that `reintentar()` reproduces the original reject-branch UI state exactly.
