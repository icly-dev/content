---
title: Pathgen
---

# Pathgen

The [beam search](<../Engine/Beam Search>) returns one winning [landing](<Movegen>): a position, an orientation, and the tag from movegen (the spin grade in the shipped engine). The runner cannot act on a landing directly; it needs inputs. Path generation, usually called pathgen, turns the chosen landing into an input string and runs once per decision.

Earlier stages run thousands of times per decision, so they must be cheap. Pathgen runs once and must be exact.

## Glossary

Shared engine vocabulary is defined in the [Glossary](<../Engine/Glossary>), and landing-specific terms in [Movegen](<Movegen>). This page adds:

| Term | Meaning |
|------|---------|
| Input path | The sequence of inputs the runner executes to carry the piece to the chosen landing. One character per input. |
| Frame | The unit of execution time the runner plays in. Inputs carry frame costs, so paths can be compared by total time. |
| Held shift | One input that slides the piece several cells at once, as if the direction were held down. |

## The contract

Pathgen takes the winning landing, the piece's current position and orientation (or the swapped-in piece's spawn position if the decision begins with a hold swap), the board, and the runner's movement options. It returns an input path or no path.

Pathgen always has the full set of movement options, even when movegen used a speed restriction. The landing exists on the board, so any valid route to it is acceptable, including a floating route that is shorter than the one used to find it.

## Searching for the fastest inputs

Pathgen searches for the shortest route through movement states. Each state records the position, orientation, and spin so far. Available moves are the runner's inputs:

- Shift one cell left or right, or hold a direction to slide until blocked.
- Rotate clockwise, counterclockwise, or 180 degrees when allowed, using the ruleset's kick tables.
- Soft drop one row, sonic drop to the resting spot, or hard drop.

Each input has a cost, and the route with the lowest total time wins. If pathgen reaches a state by a cheaper route, it replaces the previous route. The search ends when a hard drop reaches the chosen landing. Any path that places a piece therefore ends with a hard drop that locks it exactly where expected. Tie-breaking is deterministic, so the same decision always produces the same path.

## Reproducing the promised spin

The scored spin grade is a promise: the game honors it only if the executed inputs produce the right rotation. Pathgen therefore tracks spin as part of the movement state and recomputes it after every rotation, using the same corner and kick rules as the [movegen grading pass](<Movegen#spin-classification>). A route that reaches the landing with the wrong spin does not count as a solution. The output must end with a rotation that produces the promised grade, or pathgen returns no path.

Both layers use the same spin-grading rules, so the scored grade can be reproduced.

The movement queries for shifts, rotations with kicks, and downward movement come from [fast-reachability](https://github.com/icly-dev/fast-reachability), the same public library used by [movegen](<Movegen>).

## Hold first

If the decision begins with a hold swap, the hold input comes first. Movement then starts from the swapped-in piece's entry spot, as returned by the AI (see [AI interface](<../AI/AI interface>)). A decision can be "hold, then place," a hold swap on its own, or a plain placement. A hold swap on its own consists only of the hold input.

## When there is no path

If no input sequence can reach the winning landing, pathgen returns no path rather than an incorrect route, and the runner proceeds without inputs. This should not happen in normal operation: movegen found the landing through valid movement, and pathgen has at least the same options. The promised spin is the one requirement that could still rule out every route.

## The inputs the runner receives

The path is a string of single-character inputs, executed left to right:

| Input | Meaning |
|-------|---------|
| `l` / `r` | Shift one cell left / right. |
| `L` / `R` | Held shift: slide left / right until blocked. |
| `z` / `c` | Rotate counterclockwise / clockwise. |
| `x` | Rotate 180 degrees (only when the runner allows it). |
| `d` | Soft drop one row. |
| `D` | Sonic drop to the resting spot without locking. |
| `V` | Hard drop; locks the piece. |
| `v` | Hold swap; appears only as the first input of a path. |
