---
title: Pathgen
---

# Pathgen

The [beam search](<../Engine/Beam Search>) ends with one winning [landing](<Movegen>): a position, an orientation, and the tag movegen attaches to it (the spin grade, in the shipped engine). The runner cannot act on that directly; it executes inputs. Turning the chosen landing into that string is path generation, usually shortened to pathgen, and it runs once per decision.

Everything before this layer runs thousands of times per decision and must be cheap. Pathgen runs once and must be exact.

## Glossary

Shared engine vocabulary is defined in the [Glossary](<../Engine/Glossary>), and landing-specific terms in [Movegen](<Movegen>). This page adds:

| Term | Meaning |
|------|---------|
| Input path | The sequence of inputs the runner executes to carry the piece to the chosen landing. One character per input. |
| Frame | The unit of execution time the runner plays in. Inputs carry frame costs, so paths can be compared by total time. |
| Held shift | One input that slides the piece several cells at once, as if the direction were held down. |

## The contract

Pathgen takes the winning landing, the piece's current position and orientation (or the swapped-in piece's spawn, when the decision starts with a hold swap), the board, and the runner's movement options. It returns the input path, or nothing.

It always has the full option set, regardless of movegen's speed restriction: the landing exists on the board, so any real route to it is valid, including a floating route shorter than the one that found it.

## Searching for the fastest inputs

Pathgen runs a shortest-path search over movement states. A state is the position, the orientation, and the spin carried so far. The moves are the runner's inputs:

- shift one cell left or right, or a held shift that slides until something blocks,
- rotate, clockwise, counterclockwise, and 180 degrees when allowed, with the ruleset's kick tables applied,
- soft drop one row, sonic drop to the resting spot, or hard drop.

Each input carries its cost, and the cheapest total time wins; a state reached again more cheaply replaces its route. The search stops at a hard drop that reaches the landing, so a path that places a piece always ends in a hard drop that locks it exactly where promised. Tie-breaking is deterministic: the same decision always yields the same path.

## Reproducing the promised spin

The scored spin grade is a promise: the game only honors it if the executed inputs end with the right rotation. So the spin is part of the movement state, recomputed after every rotation with the same corner and kick rules the [movegen grading pass](<Movegen#spin-classification>) used. A route arriving with the wrong spin does not reach the goal, so the output path ends in a rotation that produces the promised spin, or there is no path.

Both layers grade spins with the same rules, so a scored grade is a reproducible grade.

The movement queries underneath, shifts, rotations with kicks, moving down, come from the same public library, [fast-reachability](https://github.com/icly-dev/fast-reachability), that [movegen](<Movegen>) is built on.

## Hold first

When the decision starts with a hold swap, the hold input comes first, and the movement starts from the swapped-in piece's entry spot: what the AI's spawn call returned for it (see [AI interface](<../AI/AI interface>)). A decision can be "hold, then place", a pure hold swap, or a plain placement. A pure hold swap is the hold input alone, with nothing after it.

## When there is no path

If the winning landing cannot be reached by any input sequence, the answer is no path rather than a wrong one, and the runner proceeds without inputs. This should not occur in normal operation: the landing was found through real movement, and pathgen has at least the options that found it. The promised spin is the one requirement that can still cut every route, and then the answer is no path.

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
