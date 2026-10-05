---
title: Overview
---

# Overview

This page walks through one decision of the engine: what goes in, what happens inside, and what comes out. The individual stages have their own pages, linked along the way.

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

The runner calls the engine once per decision, which is normally once per placed piece. The call carries the full game state:

- the playfield, 10 cells wide and 22 rows tall, plus one row above it for incoming garbage,
- the active piece with its position and rotation, in the runner's coordinates (converted internally, including the vertical axis direction),
- the held piece and whether holding is currently allowed, plus whether 180-degree spins are allowed,
- the known preview, whose length sets how deep the search can look with certainty,
- the match state: the back-to-back flag, the combo counter, the incoming attack, and the combo table,
- the search knobs: the preview depth and a level setting that picks the search budget on an exponential scale.

Preparation then sets up the search:

- the playfield is loaded into the engine's internal board representation and the piece position is translated,
- the match state (combo, back-to-back, incoming attack) is loaded into the running status,
- caches are reconciled: if the AI's parameters changed since the last call, everything cached is dropped; otherwise the transposition tables carry over as described in [Transposition table](<Transposition Table>).

The search itself is the [beam search](<Beam Search>): it expands placements depth by depth under a pre-computed beam limit schedule, scores candidates through the cached board evaluations, halts early when the whole beam agrees on the first placement, and branches over [sampled futures](<Fake next and branching>) when the known preview runs out.

The decision is the best candidate the deepest completed layer produced, traced back to its first placement: where the active piece should land, or whether a hold swap should happen first.

## What comes out

The engine answers with a command string for the runner, one of three things:

- a **movement path**: the inputs that carry the active piece to the chosen landing, including any spin adjustments; the runner executes it and the piece drops,
- the **hold command**, when the search concluded that swapping the held piece in is better than any placement this turn, and no placement was chosen,
- an **empty answer**, when there is nothing to do; the runner simply proceeds.

After the piece lands, the runner calls again with the new state, and the cycle repeats.

## What the engine remembers between calls

Each player gets a separate engine instance, keyed by the player id, and calls for one player are serialized. Between calls the instance keeps:

- the transposition tables and the predicted board, so evaluated positions survive across moves when the prediction holds,
- the learned per-depth branching estimates that shape the beam limit schedule, reset only when a game restarts,
- the combo table, cached on first use,
- the AI's current parameters; a change is detected on the next call and triggers the full cache reset.

Given the same game state and the same random state for sampling, a decision is reproducible: the search budget is a count rather than a clock, and every tie is broken deterministically.
