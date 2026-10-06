---
title: Landings
---

# Landings

The [beam search](<../Engine/Beam Search>) decides which placement wins, but it never looks for placements: every candidate is handed the list of placements its piece can make. Producing that list is the landing search, the lowest layer of the engine. Given a board and a piece at its spawn, it reports every spot where the piece can come to rest, and marks which of those spots are spins. The [path search](<Movement paths>) then turns whichever spot wins into inputs.

```text
game state -> beam search -> landing sweep (every candidate) -> path search (winner only) -> command string
```

## Glossary

Shared engine vocabulary (board, roof, placement, depth, horizon, hold) is defined in the [Glossary](<../Engine/Glossary>). This page adds what is specific to movement:

| Term | Meaning |
|------|---------|
| Spawn | The fixed spot where a newly entering piece appears, near the top of the field. |
| Landing | A spot where the piece can come to rest: a position and orientation on the board, before any lines clear. A [placement](<../Engine/Glossary>) is a landing plus the clears it causes. |
| Resting | A property of a landing: the piece cannot move further down from that spot. |
| Kick | The nudge a ruleset applies when a rotation is blocked; kicks can shift a piece sideways or upward while it rotates. |
| Floating movement | Inputs that let the piece descend under control: soft drop, one row at a time, and sonic drop, straight down to the resting spot without locking. Floating is what lets a piece slide sideways under an overhang partway through its descent. |
| Hard drop | Straight down to the resting spot, locking immediately. |
| Bitboard | A board stored as raw bits, one per cell, so a whole row of cells is examined or moved in a single operation. |
| Spin | A landing reached by rotating into a tight pocket rather than falling into it. The ruleset this engine ships with rewards these for one piece, the T, and grades them into a full spin and a weaker mini spin. |

## The contract

The landing search takes:

- the board,
- the piece and its spawn position and orientation,
- the movement options the runner declares (for example, whether 180-degree rotations are allowed),
- the candidate's context: the current stack roof, and whether the placement that produced this board cleared lines.

It reports every reachable landing, each labeled with a spin type: none, mini, or full. The reports arrive one at a time through a callback as the sweep finds them, not as a collected list: the consumer places and scores each landing immediately and discards what it does not want. Since the sweep runs for every candidate at every depth, this streaming form keeps the hot path free of per-candidate allocation, and memory stays flat no matter how many landings a piece has; returning the same information would require a growable result list on every call.

It never scores anything. Where the piece can go is mechanics; whether going there is good is judgment, and judgment lives entirely in the AI's evaluation. This split is deliberate: the landing search must be fast and exhaustive over mechanics, and it must stay correct no matter how the evaluation changes.

## One sweep per piece

The sweep does not walk positions one by one. For each orientation of the piece it keeps one bitboard: every cell where that orientation currently fits. One operation advances the whole set at once:

- shift the entire set one cell left or right, minus wherever that would overlap,
- rotate the entire set into the neighboring orientation, applying the ruleset's kick tables cell by cell,
- when floating is allowed, descend the entire set one row.

Repeating this until nothing new appears yields, for every orientation, all cells the piece can occupy by any sequence of shifts, rotations, and (when allowed) controlled descent. The landing spots are the cells in those sets from which the piece cannot move down. Enumerating them one by one produces the landings.

Two properties matter architecturally:

- **Exactness.** A landing is reported only if the movement inputs can genuinely reach it, and every reachable landing is reported, subject to one deliberate restriction described below. The evaluation never sees a placement the piece could not actually make, and never misses one it could.
- **Early exit.** The sweep stops as soon as every resting spot on the board is covered by the set it has built. On open boards this ends the sweep long before convergence.

When floating is allowed, the sweep also credits the piece for inputs it holds while falling: the positions reachable by sliding along unobstructed rows during the descent from the spawn are computed in one direct step, which cell-by-cell expansion alone would undercount.

## When floating is allowed

By default the sweep may use every option the runner allows. One restriction is applied per candidate: when all of the following hold, soft drop and sonic drop are switched off for that sweep, so the piece may only shift and rotate near the top of the board and then hard drop:

- the piece is not the spin piece,
- the placement that produced this board cleared no lines,
- the piece spawns entirely above the stack, which is the normal case (its lowest cell sits above the [roof](<../Engine/Glossary>)).

The rationale is speed. Floating movement multiplies the reachable positions enormously, most floating placements for a non-spin piece are ones the evaluation would discard anyway, and this sweep runs for every candidate at every depth, making it the largest single cost of a decision. Cutting floating where it rarely matters buys a large reduction where it always runs.

The exceptions keep floating exactly where it earns its keep: the spin piece always keeps it, because spins are made by descending partway and rotating into a pocket; and it returns whenever the stack has grown up into the spawn area, or the previous placement cleared lines.

Two consequences are worth stating plainly:

- The movement options depend on the candidate's history (the clear count travels with it), not only on the board. Two candidates on similar boards can face different movement rules.
- The restriction shapes only the landing sweep. The [path search](<Movement paths>) always has the full option set, so whatever the beam chose can be executed.

## Spin classification

Spin detection runs over the landings of the spin piece only, as a labeling pass on top of the sweep. In the ruleset this engine ships with, that piece is the T, and the labels are none, mini, and full.

The classification proceeds in steps:

1. **Pinned.** A landing can only be a spin if the piece cannot move down from it; a piece that can still fall has not spun.
2. **Corners.** The four diagonal cells around the piece's center are checked against the board. With enough of them occupied, the piece is wedged in a way only a rotation can produce.
3. **Front corners decide the grade.** The two corners on the side the piece points toward separate a full spin from a weaker one: both occupied means full.
4. **The kick test confirms the rest.** The corner rule alone can mislabel a slot the piece merely fell into, so in the weaker cases the sweep verifies that the piece can rotate out of the spot and back into exactly the same spot. Only then is a spin label applied, and the size of the kick used on the way distinguishes the strongest twist, which also counts as full, from a mini.

One placement can be delivered more than once: different kick sequences can reach the same resting spot with different strengths, and the spot is then reported once per applicable label, for example once with no spin, once as a mini, and once as a full. The callback form is what makes this natural: repeated delivery of one spot is just more calls, with no list to grow and no special case at the consumer, which evaluates each report as its own candidate. The ruleset scores the variants differently, so treating them separately is what lets the search prefer a full spin over a mini at the same spot.

## What it costs and where it runs

The sweep runs for every candidate the beam expands, once per piece choice (the current piece, and the held piece when holding is available). It is the engine's hot path: the profiler's timing summary reports it as its own stage, named `search`, so a profile says directly how much of a decision's time went into movement.

## What the beam search does with landings

Landings are the currency between the layers: a candidate is created from a landing, and each candidate remembers only the first landing of its path, which is how a deep search still answers with one concrete move. When the held piece is the same piece as the current one, the hold branch keeps only landings the current piece cannot reach itself: a hold swap that changes nothing is not worth an input.
