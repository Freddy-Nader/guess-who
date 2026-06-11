# Plan 005: Guarantee unique names and distinguishable characters in board generation

> **Executor instructions**: Follow this plan step by step. Run every
> verification command and confirm the expected result before moving to the
> next step. If anything in the "STOP conditions" section occurs, stop and
> report — do not improvise. When done, update the status row for this plan
> in `plans/README.md` — unless a reviewer dispatched you and told you they
> maintain the index.
>
> **Drift check (run first)**: `git diff --stat f1df899..HEAD -- .src/Data.class`
> Changes from plans 001–003 are expected (this plan assumes plan 002's
> version of the assignment loop — see "Current state"). If `asignacion()`
> does not match either the original or the post-002 shape shown below,
> treat it as a STOP condition.

## Status

- **Priority**: P3 — **conditional: read plans/README.md first.** If plan 007's spike concluded in favor of a fixed character set, mark this plan REJECTED instead of executing it.
- **Effort**: M
- **Risk**: LOW
- **Depends on**: plans/002-fix-elimination-engine.md (rewrites the same loop)
- **Category**: bug
- **Planned at**: commit `f1df899`, 2026-06-11

## Why this matters

The game generates its 24 characters randomly with no uniqueness constraints. Names are drawn from a 48-name pool *with replacement*: with 24 draws, the probability of at least one duplicate name on the board is ≈99%, so nearly every game shows two characters with the same name. Worse, complete trait-vector duplicates are possible — two characters identical in all 10 characteristics — which makes the hidden character logically indistinguishable and the game unwinnable by deduction. The source name lists also contain literal duplicates ("Patricia" twice) and an accent-variant near-duplicate ("Mónica"/"Monica"). After this plan, every generated board has 24 distinct names and 24 pairwise-distinct trait vectors.

## Current state

Gambas `.class` files are **plain text source**; if a tool refuses them by extension, use `cat`/`sed`.

All changes are in `.src/Data.class`.

**Name pool defects** (inside `caracteristicas()`):

- `Data.class:161` — `nombres[1, 7] = "Patricia"` and `Data.class:197` — `nombres[1, 43] = "Patricia"` (duplicate)
- `Data.class:169` — `nombres[1, 15] = "Mónica"` and `Data.class:192` — `nombres[1, 38] = "Monica"` (accent variant of the same name)

**`asignacion()` as of the planned-at commit** (`.src/Data.class:220-269`, abridged to the parts this plan touches). Plan 002 replaces the `' resto de caracteristicas` inner loop with a `Continue`-based version; both shapes are shown so you can recognize either:

```
Public Sub asignacion()
    Dim i As Byte                       ' primera dimension
    Dim j As Byte                       ' segunda dimension
    Dim r1 As Byte                      ' para random
    Dim r2 As Byte                      ' para random

    ' asignacion de caracteristicas
    For i = 0 To 23
        ' genero
        Repeat
            r1 = Int(Rnd(1, 3))
        Until r1 <> 3
        personajes[i, 0] = car[0, r1]
        
        ' nombre
        Repeat
            r2 = Int(Rnd(0, 48))
        Until r2 <> 48
        personajes[i, 10] = nombres[r1 - 1, r2]
        
        ' rasgo facial
        If r1 = 1 Then
            Repeat
                r2 = Int(Rnd(1, 4))
            Until r2 <> 4
            personajes[i, 6] = car[6, r2]
        Else
            personajes[i, 6] = car[6, 3] ' para que no haya mujeres con barba o bigote
        Endif 
        
        ' resto de caracteristicas  (original shape)
        For j = 1 To 9
            Repeat
                r1 = Int(Rnd(1, 4))
            Until r1 <> 4
            personajes[i, j] = car[j, r1]
            If j = 5 Then
                Inc j
            Endif
        Next
    Next
    ...
```

Post-002 shape of the inner loop:

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

**Bug to be aware of while restructuring**: the original code reuses `r1` for both gender (1=Hombre, 2=Mujer) and the trait loop; the name lookup `nombres[r1 - 1, r2]` and the rasgo block read `r1` as gender, which only works because they run *before* the trait loop clobbers it. Preserve that ordering or use a dedicated variable (this plan's target code uses a dedicated `genero` variable to remove the trap).

**Trait space sanity check** (why uniqueness retry terminates): gender(2) × piel(3) × ropa(3) × forma-ojos(2, post-002) × color-ojos(3) × pecas(3) × tipo-cabello(3) × color-cabello(3) × accesorios(3) ≈ 8,748 male vectors (× rasgos 3) and 2,916 female — vastly more than 24; retry loops converge fast.

Repo conventions: Spanish identifiers/comments, 4-space indent, conventional commits. Local arrays in Gambas use the array-class form: `Dim usado As New Boolean[2, 48]`.

## Commands you will need

| Purpose | Command | Expected on success |
|---------|---------|---------------------|
| Compile (Linux + Gambas only) | `gbc3 -ag .` from repo root | exit 0 |
| Compile (otherwise) | CI `build / compile` job (plan 001) on push | green |

## Scope

**In scope** (the only file you should modify):
- `.src/Data.class`

**Out of scope** (do NOT touch):
- `.src/Plantilla.class`, forms — no UI involvement.
- The `car[]` option strings (logic compares them).
- The hidden-character selection loop (`oculto[]`) and the `mock` initialization at the end of `asignacion()`.

## Git workflow

- Branch: `advisor/005-unique-characters`
- Commits: `fix(data): remove duplicate names from the name pools` and `fix(data): generate unique names and distinguishable characters`.
- Do NOT push or open a PR unless the operator instructed it.

## Steps

### Step 1: Clean the name pools

In `caracteristicas()`:

1. `nombres[1, 43] = "Patricia"` → `nombres[1, 43] = "Gabriela"`
2. `nombres[1, 38] = "Monica"` → `nombres[1, 38] = "Daniela"`

(Replacements chosen to not collide with any existing entry — verify with the command below.)

**Verify**: `grep -o '"[A-ZÁÉÍÓÚÑ][a-záéíóúñü]*"' .src/Data.class | sort | uniq -d` → output contains no names from the `nombres` lists (characteristic option strings like `"Ninguno"` may legitimately repeat — check that whatever the command prints is not a person name; expected person-name duplicates: none).

### Step 2: Restructure the per-character assignment with uniqueness retries

Replace the `' asignacion de caracteristicas` outer loop (everything from `For i = 0 To 23` to its matching `Next`, i.e. the gender/name/rasgo blocks plus the `resto de caracteristicas` inner loop — NOT the `oculto`/`mock` sections below it) with:

```
    ' asignacion de caracteristicas
    Dim genero As Byte
    Dim duplicado As Boolean
    Dim p As Byte
    Dim usado As New Boolean[2, 48]

    For i = 0 To 23
        ' rasgos: reintentar hasta que el personaje sea distinto a los anteriores
        Repeat
            ' genero
            genero = Int(Rnd(1, 3))
            personajes[i, 0] = car[0, genero]

            ' rasgo facial
            If genero = 1 Then
                personajes[i, 6] = car[6, Int(Rnd(1, 4))]
            Else
                personajes[i, 6] = car[6, 3] ' para que no haya mujeres con barba o bigote
            Endif

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

            ' ¿identico a un personaje anterior?
            duplicado = False
            For p = 0 To i - 1
                duplicado = True
                For j = 0 To 9
                    If personajes[i, j] <> personajes[p, j] Then
                        duplicado = False
                        Break
                    Endif
                Next
                If duplicado Then Break
            Next
        Until Not duplicado

        ' nombre: sin repetir dentro del mismo genero
        Repeat
            r2 = Int(Rnd(0, 48))
        Until Not usado[genero - 1, r2]
        usado[genero - 1, r2] = True
        personajes[i, 10] = nombres[genero - 1, r2]
    Next
```

Notes:
- If executing against the **pre-002** code (drift check shows plan 002 not yet applied — that violates the dependency; STOP), do not improvise a hybrid.
- The name is assigned *after* the trait-retry loop so a retried character does not leak a consumed name.
- The `For p = 0 To i - 1` loop body never runs when `i = 0` (Gambas skips a `For` whose start exceeds its end), so the first character can't deadlock.
- Declarations: Gambas allows `Dim` at procedure top only; move the four new `Dim` lines up with the existing ones (`Dim i`, `Dim j`, `Dim r1`, `Dim r2` stay).

**Verify**: `grep -n "usado\[genero - 1, r2\] = True" .src/Data.class` → one match. `grep -n "Until r2 <> 48\|Until r1 <> 3\b\|Until r1 <> 4" .src/Data.class` → no matches (dead guards gone with the restructure).

### Step 3: Compile and statistical check

Compile per plan 001. If a Linux GUI is available, start the game 3 times; each time read the IDE Debug output (post-003) or the board buttons and confirm: 24 distinct names, and no two characters visually identical. If no GUI: record that the runtime check is deferred.

## Test plan

No automated harness exists. The deterministic checks are the greps above; the behavioral check is Step 3 (duplicates were near-certain before — ≈99% per game — so 3 clean runs is strong evidence). The trait-space sanity check in "Current state" is the termination argument for the retry loops.

## Done criteria

- [ ] Compile gate passes.
- [ ] Step 1 and Step 2 grep verifications hold.
- [ ] `git status` shows only `.src/Data.class` modified.
- [ ] `plans/README.md` status row updated.

## STOP conditions

Stop and report back (do not improvise) if:

- `plans/README.md` shows plan 007 concluded "adopt fixed character set" — mark this plan REJECTED in the index and stop.
- Plan 002 has not been executed (drift check shows the original `Select Case op` still in `Plantilla.class` or the original inner loop in `asignacion()`).
- The compiler rejects `Dim usado As New Boolean[2, 48]` — report the exact error; the fallback (class-level embedded array) changes scope and needs operator sign-off.
- Any writer to `personajes[i, 10]` exists outside `asignacion()` (`grep -rn "personajes\[" .src/ | grep "10\] ="`).

## Maintenance notes

- If plan 007's fixed-character-set direction is adopted *later*, this generation code gets deleted wholesale; the name-pool cleanup (Step 1) survives either way.
- The gender-balance is uncontrolled (each character is 50/50); if a designer ever wants guaranteed 12/12, this loop is the place.
- Reviewer should scrutinize: that the name `Repeat/Until` cannot exhaust (max 24 names per gender from pools of 48 — worst case all 24 same gender still terminates).
