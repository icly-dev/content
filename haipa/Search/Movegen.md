---
title: Movegen
---

# Movegen

The [beam search](<../Engine/Beam Search>) decides which placement wins, but it never looks for placements. Producing them is move generation, usually shortened to movegen, the lowest layer of the engine. Given a board and a piece at its spawn, it reports every spot the piece can come to rest on, and which of those spots are spins. The [pathgen](<Pathgen>) turns the winning spot into inputs.

```text
game state -> beam search -> movegen (every candidate) -> pathgen (winner only) -> command string
```

## Glossary

Shared engine vocabulary (board, roof, placement, depth, horizon, hold) is defined in the [Glossary](<../Engine/Glossary>). This page adds what is specific to movement:

| Term | Meaning |
|------|---------|
| Spawn | The fixed spot where a newly entering piece appears, near the top of the field. |
| Landing | A spot where the piece can come to rest: a position and orientation, before any lines clear. A [placement](<../Engine/Glossary>) is a landing plus the clears it causes. |
| Resting | A property of a landing: the piece cannot move further down from that spot. |
| Tag | An extra value attached to every landing by the search. Its meaning belongs to the search layer, not to the movement mechanics; in the shipped engine it grades T-spins. |
| Kick | The nudge a ruleset applies when a rotation is blocked; kicks can shift a piece sideways or upward while it rotates. |
| Floating movement | Inputs that let the piece descend under control: soft drop, one row at a time, and sonic drop, down to the resting spot without locking. Floating is what lets a piece slide under an overhang mid-descent. |
| Hard drop | Straight down to the resting spot, locking immediately. |
| Spin | A landing reached by rotating into a tight pocket rather than falling into it. The ruleset this engine ships with rewards these for one piece, the T, and grades them into a full spin and a weaker mini spin. |

## The contract

Movegen takes:

- the board,
- the piece and its spawn position and orientation,
- the movement options the runner declares (for example, whether 180-degree rotations are allowed),
- the candidate's context: the current stack roof, and whether the placement that produced this board cleared lines.

It reports every reachable landing: a position and orientation, plus an auxiliary tag. The tag is part of the search itself, not of the movement mechanics: the shipped engine grades T-spins with it (none, mini, or full); another search could carry something else. Reports arrive one at a time through a callback as movegen finds them, not as a list: the consumer places and scores each landing immediately. Movegen runs for every candidate at every depth, so this keeps the hot path free of per-candidate allocation; a return value would mean a growable result list on every call.

It never scores anything. Where the piece can go is mechanics; whether that is good is judgment, and judgment lives in the evaluation. Movegen must be fast and exhaustive over mechanics, and stay correct no matter how the evaluation changes.

## Built on fast-reachability

Movegen is a thin layer over a separate, public movement library, [fast-reachability](https://github.com/icly-dev/fast-reachability). One call per piece:

- in: the board, the spawn position and orientation, and the allowed options,
- out: the positions each orientation of the piece can reach, plus a movement checker built from that same call.

The results are exact: a position is reported only if the allowed inputs can reach it, and every reachable position is reported. The cells the piece cannot move down from are the landings, and movegen enumerates them and streams them out as described above.

The checker answers, for any spot: can the piece shift or descend from there, and where does a rotation with its kicks land. Spin grading and [pathgen](<Pathgen>) both ask that checker, so there is one definition of "can move".

haipa decides the inputs and interprets the outputs (landings, spin grades). The options are per-candidate: usually everything the runner allows, with one speed-driven exception, soft drop and sonic drop are disabled for non-spin pieces unless the stack reaches the spawn area or the previous placement cleared lines. The movement model stays in an independent, benchmarked library; the engine's code stays about policy, not geometry.

## Spin classification

Spin detection runs over the landings of the spin piece only, as a grading pass on top of the returned positions, using the checker. In the ruleset this engine ships with, that piece is the T, and the tag grades each landing none, mini, or full.

1. **Pinned.** The piece cannot move down from the landing; otherwise it has not spun.
2. **Corners.** The four diagonal cells around the piece's center are checked against the board. Enough occupied means the piece is wedged in a way only a rotation can produce.
3. **Front corners decide the grade.** Both corners on the side the piece points toward occupied means full.
4. **The kick test confirms the rest.** The corner rule alone can mislabel a slot the piece merely fell into, so the weaker cases must rotate out of the spot and back into exactly the same spot. The kick size on the way separates the strongest twist, which also counts as full, from a mini.

One placement can be delivered more than once: different kick sequences can reach the same spot with different strengths, so it is reported once per grade, for example no spin, mini, and full. The callback makes this natural: repeated delivery is just more calls, no list to grow, and the consumer evaluates each report as its own candidate. The ruleset scores the variants differently, which is what lets the search prefer a full spin over a mini at the same spot.

## What it costs and where it runs

Movegen runs for every candidate the beam expands, once per piece choice: the current piece, plus the held piece when holding is available. It is the engine's hot path: the profiler's timing summary reports it as its own stage, named `search`.

## What the beam search does with landings

Landings are the currency between the layers: a candidate is created from a landing, and each candidate remembers only the first landing of its path, which is how a deep search still answers with one concrete move. When the held piece equals the current one, the hold branch keeps only landings the current piece cannot reach itself: a hold swap that changes nothing is not worth an input.
