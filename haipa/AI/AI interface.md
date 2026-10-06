---
title: AI interface
---

# AI interface

The engine manages each decision: it expands placements, maintains caches, and chooses a move. The AI is the pluggable part that supplies the judgment. For every call, the engine passes along the current game state, so the AI responds to the position and search depth at hand rather than to the game as a whole.

## Glossary

Shared engine vocabulary (board, roof, placement, depth, horizon, hold, status, search budget) is defined in the [Glossary](<../Engine/Glossary>). This page adds what is specific to the AI interface:

| Term | Meaning |
|------|---------|
| AI | The pluggable part of the engine that supplies judgment and answers ruleset questions through its rule layer. |
| Rule layer | The part of the AI that answers ruleset questions, including the absolute entry spot and the bag randomizer. |
| Suggested spawn | The absolute entry spot supplied by the rule layer before any adjustments. |
| Environment | What remains after a placement: the unplaced queue positions, the piece that will be held, and whether the placement used a hold swap. |

## Where the engine calls in

```text
engine setup:    give the AI its configuration
each decision:   ask where each piece enters
                 evaluate each new board
                 score each candidate
                 read the peak and spread for the beam schedule
```

The built-in AI supports these interactions; the behavior, not the method names, defines the contract. During setup, the engine passes the AI its configuration. For each decision, the beam schedule reads the configured peak and spread (see [Beam search](<../Engine/Beam Search>)).

## Where a piece enters

The AI helps choose the entry spot for every piece that enters during the search. The engine first asks the rule layer for a suggested spawn, then passes the AI:

- the piece that is entering,
- how many lines the placement before it cleared,
- the suggested spawn,
- whether the piece enters by hold swap,
- the board it enters on,
- the status carried to this node.

The AI returns the entry spot that the search uses.

The engine asks about each piece entry at every node. This includes the hold branch of the current decision, and the next piece at each child node (from either the queue or hold). Two paths can ask about the same piece at different depths and get different answers because each path has its own board and history. The runner can ask the engine for the same entry information when placing pieces itself. A hold-swap path starts from the entry spot returned by the AI, so the runner's replay matches the search.

Rulesets use that freedom. In tetr.io, for example, the entry spot moves up if the piece would collide with the stack at spawn or if the previous placement cleared lines, but only when the game's clutch option is on. An AI for that ruleset checks the board for a collision, reads the line-clear count, and knows from its configuration whether clutch is on. The engine provides those three pieces of information when the piece enters the search.

Gravity is another example. The suggested spawn is absolute, but by the time the engine finishes searching and the runner starts the move, the AI's answer determines the piece's actual y coordinate at entry.

The shipped AI returns the suggested spawn unchanged because its target ruleset makes no entry adjustments.

## The board-only evaluation

The engine evaluates each new board once. The evaluation receives:

- the resulting board, in the engine's row form (one value per row, a bit set for each occupied cell) or as the rule layer's board type, whichever the AI provides an evaluation for,
- the board's roof.

What the evaluation measures is up to the AI. Typical examples include board-shape features such as bumpiness and aggregate height. The engine caches each result under the board's hash (see [Transposition table](<../Engine/Transposition Table>)), so the result must depend only on the board. Candidates that land different pieces on the same board therefore share one evaluation.

## The context step

Context scoring runs for every candidate placement. It takes:

- the landing (see [Movegen](<../Search/Movegen>)),
- the cached evaluation,
- the line clears the placement made,
- the board and its roof,
- the depth the placement is made at,
- the status carried from the parent node,
- the environment.

It returns the child's status, a running assessment used to rank candidates and carry match state forward, including line clears, combo, and back-to-back. Rewards can reflect events from the placement, such as a back-to-back chain, a combo, a spin, or whether a piece like the I was used or saved for later. These examples show what the inputs make it possible to judge. Which signals a particular AI uses is up to that AI.

Splitting evaluation into these two stages makes board evaluations cacheable. The [Transposition table](<../Engine/Transposition Table>) page explains the split in detail.

When the AI declares a fixed list of priority pieces, the environment also carries, for each one, how many pieces away it sits in the known preview.

## The rule layer

The rule layer answers what belongs to the ruleset rather than to judgment: the absolute entry spot, the bag randomizer, and the board and ruleset types the rule layer shares with movegen. It has its own page: [Rule layer](<../Rule/Rule layer>).

## Optional hooks

Three parts of the interface are optional, and the engine works without each of them: a custom board hash instead of the built-in one, a conversion from the board to the engine's row form, and the priority pieces behind the piece distances in the environment.
