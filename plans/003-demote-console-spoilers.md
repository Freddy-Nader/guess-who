# Plan 003: Demote console output that reveals the hidden characters to debug-only

> **Executor instructions**: Follow this plan step by step. Run every
> verification command and confirm the expected result before moving to the
> next step. If anything in the "STOP conditions" section occurs, stop and
> report — do not improvise. When done, update the status row for this plan
> in `plans/README.md` — unless a reviewer dispatched you and told you they
> maintain the index.
>
> **Drift check (run first)**: `git diff --stat f1df899..HEAD -- .src/Data.class`
> Changes from plans 001 (`Sub` keyword at line 39) and 002 (`asignacion()`
> option handling) are expected. If the `Print` blocks excerpted below are
> missing or altered, treat it as a STOP condition.

## Status

- **Priority**: P2
- **Effort**: S
- **Risk**: LOW
- **Depends on**: plans/002-fix-elimination-engine.md (002 uses these prints as its manual-test oracle; run 002 first)
- **Category**: bug (game integrity)
- **Planned at**: commit `f1df899`, 2026-06-11

## Why this matters

This Gambas Guess Who game is a hotseat two-player game, but at game start it prints **both players' secret hidden characters with all ten traits** to the console, and after every question it prints both players' live boards. Anyone who can see the terminal has a cheat sheet, defeating the game's premise. The output is genuinely useful for debugging, so the fix is to demote it from `Print` to Gambas's `Debug` instruction, which only emits output when the project runs inside the IDE/debugger — release runs stay silent.

## Current state

Gambas `.class` files are **plain text source**; if a tool refuses them by extension, use `cat`/`sed`.

Two blocks in `.src/Data.class`:

**Block 1 — hidden-character dump at the end of `asignacion()`** (`.src/Data.class:271-278` at the planned-at commit; line numbers may have shifted slightly after plan 002):

```
    ' info. de personajes ocultos
    For i = 0 To 1
        Print "El personaje oculto del jugador " & (i + 1) & " es " & (personajes[oculto[i], 10]) & " y tiene las siguientes características:"
        For j = 0 To 9
            Print (car[j, 0]) & ": " & (personajes[oculto[i], j])
        Next
        Print Null
    Next
```

**Block 2 — the whole `restantes()` procedure** (`.src/Data.class:292-313`), called once per question from `Plantilla.MenuClicks2`:

```
Public Sub restantes()
    Dim i As Byte                       ' primera dimension
    Dim j As Byte                       ' segunda dimension

    ' numero
    For i = 0 To 1
        Print "El jugador " & (i + 1) & " tiene " & (vivos[i]) & " jugadores restantes."
    Next

    ' matriz
    Print "\nJugador 1\t\tJugador 2"
    For i = 0 To 3
        For j = 0 To 5
            Print mock[0, 6 * i + j];
        Next
        Print "\t\t",
        For j = 0 To 5
            Print mock[1, 6 * i + j];
        Next
        Print Null
    Next
End
```

Gambas facts the executor needs:
- `Debug <expr>` prints `ClassName.Procedure.Line: <expr>` to stderr **only when the executable was compiled/run in debugging mode** (IDE F5 / `gbx3` on a `-g`-compiled project run from the IDE). In a normal run it is a no-op. It does not support the `Print`-style `;` (no newline) or `,` (tab) continuation syntax — each `Debug` emits one full line, so the matrix rows must be accumulated into a string first.
- String concatenation/append: `&` and `&=`. Escapes like `"\t"`/`"\n"` work in Gambas string literals (already used in this file).

Repo conventions: Spanish identifiers/comments, 4-space indent, conventional commit messages.

## Commands you will need

| Purpose | Command | Expected on success |
|---------|---------|---------------------|
| Compile (Linux + Gambas only) | `gbc3 -ag .` from repo root | exit 0 |
| Compile (otherwise) | CI `build / compile` job (plan 001) on push | green |
| Residual check | `grep -n "Print" .src/*.class` | no matches |

## Scope

**In scope** (the only file you should modify):
- `.src/Data.class`

**Out of scope** (do NOT touch):
- `.src/Plantilla.class` — keep the `Data.restantes() ' consola` call site; with `Debug` inside, it is a no-op in release and still useful in the IDE.
- Any UI `Message.Info` texts.

## Git workflow

- Branch: `advisor/003-demote-console-spoilers`
- Single commit, e.g. `fix(data): hide hidden-character console dumps behind Debug`.
- Do NOT push or open a PR unless the operator instructed it.

## Steps

### Step 1: Convert the hidden-character dump to Debug

In `asignacion()`, replace Block 1 with:

```
    ' info. de personajes ocultos (solo en modo depuración)
    For i = 0 To 1
        Debug "El personaje oculto del jugador " & (i + 1) & " es " & (personajes[oculto[i], 10]) & " y tiene las siguientes características:"
        For j = 0 To 9
            Debug (car[j, 0]) & ": " & (personajes[oculto[i], j])
        Next
    Next
```

(The `Print Null` blank-line separator is dropped — `Debug` lines are already prefixed and self-separating.)

**Verify**: `grep -n "Print" .src/Data.class` → only matches inside `restantes()` (Block 2) remain.

### Step 2: Convert `restantes()` to Debug

Replace the body of `restantes()` with:

```
Public Sub restantes()
    Dim i As Byte                       ' primera dimension
    Dim j As Byte                       ' segunda dimension
    Dim linea As String

    ' numero (solo en modo depuración)
    For i = 0 To 1
        Debug "El jugador " & (i + 1) & " tiene " & (vivos[i]) & " jugadores restantes."
    Next

    ' matriz
    Debug "Jugador 1\t\tJugador 2"
    For i = 0 To 3
        linea = ""
        For j = 0 To 5
            linea &= mock[0, 6 * i + j]
        Next
        linea &= "\t\t"
        For j = 0 To 5
            linea &= mock[1, 6 * i + j]
        Next
        Debug linea
    Next
End
```

**Verify**: `grep -cn "Print" .src/Data.class` → 0 matches. `grep -rn "Print " .src/*.class` → no matches in any class file.

### Step 3: Compile

Compile per plan 001 (locally on Linux, else CI on operator push). If a Linux GUI is available, run `gbx3 .` from a plain terminal (not the IDE) and confirm the terminal shows **no** character information at game start or after a question.

## Test plan

No automated harness exists (see plan 001). Manual check is Step 3's silent-terminal confirmation, plus (optional, IDE available): run from the Gambas IDE and confirm the same information now appears as `Debug` output in the IDE console — debuggability preserved.

## Done criteria

- [ ] `grep -rn "Print" .src/*.class` → no matches.
- [ ] Compile gate passes.
- [ ] `git status` shows only `.src/Data.class` modified.
- [ ] `plans/README.md` status row updated.

## STOP conditions

Stop and report back (do not improvise) if:

- The compiler rejects `Debug` or `&=` (older dialect than assumed) — report the exact error.
- You find additional `Print` statements outside `Data.class` (search first: `grep -rn "Print" .src/`); at planning time there were none, so any new ones mean drift — list them, convert only the ones in `Data.class`.

## Maintenance notes

- Plan 002's manual play-through relies on this output as its oracle — that is why 002 runs first (or, if 003 already landed, 002's tester must run from the IDE to see the Debug stream).
- If a proper logging facility is ever introduced, these `Debug` lines are the candidates to migrate.
