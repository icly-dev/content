---
title: Movement paths
---

# Movement paths

The [beam search](<../Engine/Beam Search>) ends with one winning [landing](<Landings>): a position, an orientation, and a spin label. The runner cannot act on that directly; it executes inputs. The path search is the last layer of the engine, and it runs once per decision: it turns the chosen landing into the string of inputs the runner performs.

Everything before this layer runs thousands of times per decision and must be cheap. This layer runs once and must be exact.

## Glossary

Shared engine vocabulary is defined in the [Glossary](<../Engine/Glossary>), and landing-specific terms in [Landings](<Landings>). This page adds:

| Term | Meaning |
|------|---------|
| Input path | The sequence of inputs the runner executes to carry the piece from where it is to the chosen landing. One character per input. |
| Frame | The unit of execution time the runner plays in. Every input costs some number of frames, so candidate paths can be compared by total time. |
| Held shift | One input that slides the piece several cells at once, as if the direction were held down; it costs one frame per cell moved. A single-cell shift costs the same as a rotation. |

## The contract

The path search takes the winning landing, the piece's current position and orientation (or, when the decision starts with a hold swap, the swapped-in piece's spawn), the board, and the movement options the runner declares. It returns the input path, or nothing.

It always has the full option set, regardless of the landing sweep's speed restriction: the landing exists on the board, so any real route to it is valid, and a shorter route through floating movement may be used even for a piece whose landings were enumerated without floating.

## Searching for the fastest inputs

The path search is a shortest-path search over movement states. A state is the piece's position, its true orientation, and the spin collected so far. From each state the possible moves are the runner's inputs:

- shift one cell left or right, or a held shift that slides until something blocks,
- rotate, clockwise, counterclockwise, and 180 degrees when allowed, with the ruleset's kick tables applied,
- soft drop one row, sonic drop to the resting spot, or hard drop.

Every input adds its frame cost, and the search prefers the cheapest total time, replacing a route whenever the same state is reached more cheaply. It stops when the landing is reached by a hard drop: the final input is always a hard drop, which locks the piece exactly where the landing promised. Because hard drop costs no frames and the search is deterministic in its tie-breaking, the same decision always produces the same path.

## Reproducing the promised spin

The spin label the beam scored is a promise, and the game will only honor it if the executed inputs actually end with the right rotation. So the spin is part of the movement state: after every rotation along the path it is re-derived from the board with the same corner and kick rules the [landing sweep](<Landings#spin-classification>) used, and a route that arrives with the wrong spin does not count as reaching the goal. The output path therefore ends in a rotation that produces the promised spin, or there is no path at all.

This is also what keeps the two layers consistent: both grade spins with the same rules, so a label that was scored is a label the runner can reproduce.

## Hold first

When the decision starts with a hold swap, the hold input is prepended and the movement part starts from the swapped-in piece's spawn, since that piece enters fresh at the standard spot. A decision can thus be "hold, then place", a pure hold swap, or a plain placement.

## When there is no path

If the winning landing cannot be reached by any input sequence, the answer degrades to no movement path rather than a wrong one, and the runner simply proceeds without inputs. Under normal operation this does not occur: the landing was found through real movement, and the path search has at least the options that found it.

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
