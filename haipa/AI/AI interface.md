---
title: AI interface
---

# AI interface

The engine drives the whole decision: it expands placements, keeps the caches, and picks the move. The judgment lives in the AI, the pluggable half the engine calls into. Every call hands over the game state at the moment it happens, so the AI answers for the depth it is asked about, not for the game as a whole.

## Glossary

Shared engine vocabulary (board, roof, placement, depth, horizon, hold, status, search budget) is defined in the [Glossary](<../Engine/Glossary>). This page adds what is specific to the AI interface:

| Term | Meaning |
|------|---------|
| AI | The pluggable half of the engine: it owns the judgment and the ruleset answers the engine calls for. |
| Rule layer | The part of the AI that answers ruleset questions: the absolute entry spot and the bag randomizer. |
| Suggested spawn | The absolute entry spot the rule layer gives for a piece, before any adjustment. |
| Environment | What a placement leaves behind: the queue positions still unplaced, the piece that will sit in hold afterwards, and whether the placement was the swap. |

## Where the engine calls in

```text
engine setup:    init, handing the AI its configuration
each decision:   spawn, every piece entry
                 eval, every new board
                 get, every candidate
                 peak and spread, read for the beam schedule
```

The method names are the ones the built-in AI implements; the behavior, not the name, is the contract. When the engine is set up it hands the AI its configuration, and the beam schedule reads the peak and spread configured there (see [Beam search](<../Engine/Beam Search>)).

## Where a piece enters

Every piece that enters play during the search passes through the AI's `spawn`. The engine first asks the rule layer for the suggested spawn and then hands the AI:

- the piece that is entering,
- how many lines the placement before it cleared,
- the suggested spawn,
- whether the piece enters by hold swap,
- the board it enters on,
- the status carried to this node.

The AI returns the entry spot the search should use.

The call happens per node, once per piece entry: the hold branch of the current decision, the next piece of every child node, whether it comes from the queue or from hold, and every entry inside the sampled futures. Two paths can ask about the same piece at different depths and get different answers, because each sees its own board and history. The host can also ask the engine the same question when it places pieces itself. The path for a hold swap starts from the entry spot the AI returned, so the runner's replay matches what the search modeled.

Rulesets use that freedom. In tetr.io, for example, the entry spot moves up when the piece would collide with the stack at spawn (a top out), or when the placement right before it cleared lines, and only when the game's clutch option is on. An AI targeting that ruleset checks the board for the collision, reads the clear count, and knows from its own configuration whether clutch is on; the call hands over exactly those three things at the depth where the piece enters. Gravity is a quieter example: the suggested spawn is absolute, but by the time the engine has thought and the runner starts performing the move, the y coordinate a piece really enters at is the AI's to describe.

The shipped AI returns the suggested spawn unchanged: the ruleset it targets adjusts nothing at entry.

## The board-only evaluation

The engine calls `eval` once per board it has not seen yet. The evaluation receives:

- the resulting board, in the engine's row form or as the rule layer's board type, whichever the AI provides an evaluation for,
- the board's roof.

What it measures is the AI's own business; typical examples are board shape features such as bumpiness and aggregate height. Everything it returns is cached under the board's hash (see [Transposition table](<../Engine/Transposition Table>)), so it must depend on the board and nothing else: two candidates that land different pieces into the same board share one evaluation.

## The context step

The context step, `get`, runs for every candidate placement. It takes:

- the landing (see [Movegen](<../Search/Movegen>)),
- the cached evaluation,
- the line clears the placement made,
- the board and its roof,
- the depth the placement is made at,
- the status carried from the parent node,
- the environment.

It returns the child's status: the running assessment that ranks candidates and carries the match state, line clears, combo, and back-to-back, forward. The rewards it adds come from the placement's own events, for example a back-to-back chain, a combo, a spin, or how a specific piece such as the I was spent or held for later. The split between the two stages is what makes the evaluation cacheable; the [Transposition table](<../Engine/Transposition Table>) page explains it in full.

When the AI declares priority pieces, the environment also carries how far away each one sits in the known preview.

## The rule layer

The rule layer answers what belongs to the ruleset rather than to judgment: the absolute entry spot, the bag that fills the fake pieces, and the board and ruleset types the rule layer shares with movegen. It has its own page: [Rule layer](<../Rule/Rule layer>).

## Optional hooks

Three parts of the interface are optional, and the engine works without each of them: a custom board hash instead of the built-in one, a conversion from the board to the engine's row form, and the priority pieces behind the piece distances in the environment.
