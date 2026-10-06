---
title: Overview
---

# Overview

This page follows one engine decision from input to output. Each stage has its own page, linked below. The AI supplies the judgment; see [AI interface](<../AI/AI interface>) for details.

## Glossary

| Term | Meaning |
|------|---------|
| Runner | The host program playing the game. It owns the game state and asks the engine for a decision. |
| Decision | One engine call: look at the current state, choose what to do with the active piece. |
| Movement path | The sequence of inputs that moves the active piece from its current position to the chosen landing. |
| Known preview | The piece queue the runner guarantees; see [Fake next and branching](<Fake next and branching>). |

## One decision, end to end

```text
game state -> prepare -> search -> decide -> command string
```

The runner normally calls the engine once per placed piece. Each call includes the full game state:

- the playfield, 10 cells wide and 22 rows tall, plus one row above it for incoming garbage (the engine's internal board is taller still; the rule layer declares it),
- the active piece with its position and rotation, in the runner's coordinates,
- the held piece and whether holding is currently allowed, plus whether 180-degree spins are allowed,
- the known preview, whose length sets how deep the search can look with certainty,
- the match state: the back-to-back flag, the combo counter, the incoming attack, and the runner's combo table,
- the search knobs: how deep into the known preview to look, and how much search work to spend,

Before searching, the engine prepares the current state:

- It loads the playfield and the active piece's position.
- It adds the match state (combo, back-to-back, and incoming attack) to the running status.
- It updates the caches. If the AI's parameters changed since the last call, it clears them. Otherwise, it reuses the transposition tables as described in [Transposition table](<Transposition Table>).

The [beam search](<Beam Search>) expands placements one depth at a time under a precomputed beam limit schedule. It scores candidates using cached board evaluations and stops early if the whole beam agrees on the first placement. When the known preview runs out, it branches over [sampled futures](<Fake next and branching>). The move generator ([movegen](<../Search/Movegen>)) supplies the reachable spots for each piece and identifies spin landings.

The decision is the best candidate from the deepest completed layer, traced back to its first placement. It specifies where the active piece should land, or whether to swap in the held piece first. The [path generator](<../Search/Pathgen>) turns that choice into a command string.

## What comes out

The engine returns one of three command strings:

- A **movement path** carries the active piece to the chosen landing, including any spin adjustments. The path may begin with a hold swap when the chosen placement uses the held piece. The runner executes the path, and the piece drops.
- The **hold command** is the hold input on its own. The engine answers this way only when the hold slot is empty, the known preview is empty, and holding is currently allowed: it swaps the current piece into hold, and the runner's next piece becomes active.
- A **hard drop** means the engine fell back to the simplest move: the search chose no placement, or the chosen landing had no input path. The piece drops straight down from where it is and locks.

Once the piece lands, the runner calls again with the updated state and the cycle repeats.

## What the engine remembers between calls

Each player has a separate engine instance, keyed by player ID, and calls for that player are handled one at a time. Between calls, the instance keeps:

- the transposition tables and predicted board, so previously evaluated positions can be reused when the prediction holds (see [Transposition table](<Transposition Table>)),
- the learned per-depth branching estimates that shape the beam limit schedule,
- the runner's combo table, cached the first time it is used,
- the AI's current parameters. The engine checks for changes on the next call; when the parameters changed, it clears the transposition tables and the branching estimates and forgets the prediction.

With the same game state and random sampling state, a decision is reproducible. The search budget is a count rather than a time limit, and ties are always broken the same way. The engine's random state can also be seeded for repeatable runs.
