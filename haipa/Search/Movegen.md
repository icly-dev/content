---
title: Movegen
---

# Movegen

The [beam search](<../Engine/Beam Search>) chooses the winning placement, but it does not find possible placements itself. Move generation, usually shortened to movegen, does that. Given a board and a piece at its spawn, it reports every reachable resting spot and identifies the spin landings. [Pathgen](<Pathgen>) then turns the winning spot into inputs.

```text
game state -> beam search -> movegen (every candidate) -> pathgen (winner only) -> command string
```

## Glossary

Shared engine vocabulary (board, roof, placement, depth, horizon, hold) is defined in the [Glossary](<../Engine/Glossary>). This page adds what is specific to movement:

| Term | Meaning |
|------|---------|
| Spawn | The position where a newly entering piece appears. The AI's answer determines it (see [AI interface](<../Engine/AI interface>)). |
| Landing | A spot where the piece can come to rest: a position and orientation, before any lines clear. A [placement](<../Engine/Glossary>) is a landing plus the clears it causes. |
| Resting | A property of a landing: the piece cannot move further down from that spot. |
| Tag | An additional value the search attaches to every landing. Its meaning belongs to the search layer, not the movement mechanics. In the shipped engine, it records the T-spin grade. |
| Kick | The nudge a ruleset applies when a rotation is blocked; kicks can shift a piece sideways or upward while it rotates. |
| Floating movement | Inputs that let the piece move down under control: soft drop, one row at a time, and sonic drop, down to the resting spot without locking. Floating is what lets a piece slide under an overhang while it falls. |
| Hard drop | Straight down to the resting spot, locking immediately. |
| Spin | A landing reached by rotating into a tight pocket rather than falling into it. The ruleset this engine ships with rewards these for one piece, the T, and grades them into a full spin and a weaker mini spin. |

## The contract

Movegen takes:

- the board,
- the piece, including its spawn position and orientation,
- the runner's movement options, such as whether 180-degree rotations are allowed,
- the candidate's context: the current stack roof and whether the previous placement cleared lines.

It reports every reachable landing as a position and orientation, along with a tag. The tag belongs to the search, not the movement mechanics. In the shipped engine, it records whether a T-spin is none, mini, or full; another search could use it for something else. Movegen reports landings one at a time through a callback, and the search places and scores each one as it arrives. It runs for every candidate at every depth.

Movegen does not score placements. It determines where a piece can go; the evaluation decides whether a landing is good. Movegen must be fast, report every reachable landing, and remain correct regardless of how the evaluation changes.

## Built on fast-reachability

Movegen is a thin layer over the separate, public movement library [fast-reachability](https://github.com/icly-dev/fast-reachability). For each piece, one call takes the board, spawn position and orientation, and allowed movement options. It returns the positions each orientation can reach, along with a movement checker built from the same call.

The results are exact: a position appears only if the allowed inputs can reach it, and every reachable position is included. A reachable position is a landing when the piece cannot move farther down from it. Movegen reports each such landing as described above.

The checker answers movement questions for any position: whether the piece can shift or move down, and where a rotation with its kicks would take it. Spin grading and [pathgen](<Pathgen>) use the same checker, so there is one definition of "can move."

haipa chooses the inputs and interprets the outputs, including landings and spin grades. Options are set for each candidate and usually include everything the runner allows. For speed, soft drop and sonic drop are disabled for pieces other than the spin piece, unless the piece would enter within the stack's rows or the previous placement cleared lines. Floating movement mostly increases the number of spots to report and score; above the roof, it adds none. Keeping the movement model in a separate, benchmarked library lets the engine focus on policy rather than geometry.

## Spin classification

Spin detection checks only the landings of the spin piece. It uses the movement checker to grade the returned positions. In the shipped ruleset, the spin piece is the T, and the tag records a grade of none, mini, or full.

1. **Pinned.** The piece must be unable to move farther down from the landing. Otherwise, it has not spun.
2. **Corners.** The four diagonal cells around the piece's center are checked against the board. If enough are occupied, the piece is wedged in a way that only a rotation could produce.
3. **Front corners set the grade.** If both corners on the side the piece faces are occupied, the spin is full.
4. **The kick test confirms other grades.** The corner rule alone can mistake a spot the piece simply fell into for a spin. In weaker cases, the piece must rotate out and return to exactly the same spot. The kick size distinguishes a mini from the strongest twist, which also counts as full.

A placement can be reported more than once. Different kick sequences may reach the same spot with different strengths, so movegen reports it once for each grade, such as none, mini, and full. The search scores each report as a separate candidate, and the ruleset scores the variants differently. This lets it prefer a full spin over a mini at the same spot.

## What it costs and where it runs

Movegen runs for every candidate expanded by the beam, once for each piece choice: the current piece and, when holding is allowed, the held piece.

## What the beam search does with landings

Landings connect the movement and search layers. Each candidate starts from a landing and keeps only the first landing in its path, which lets a deep search return one concrete move. If the held piece is the same as the current piece, the hold branch keeps only landings the current piece cannot reach on its own. A hold swap that changes nothing is not worth an input.
