---
title: Rule layer
---

# Rule layer

The AI has two halves: the judgment, which the engine calls per board and per candidate, and the rule layer, which answers what belongs to the ruleset rather than to judgment. The engine asks, the rule layer answers, and the questions come with the game state of the moment, as described in [AI interface](<../AI/AI interface>).

## The absolute entry spot

`suggest_spawn` gives the absolute entry spot for a piece: the same spot for every situation, before any adjustment. It is the starting point the AI's spawn adjustment works from; the engine asks for it first and hands the result to the AI's spawn call.

## The bag

`make_bag` returns one draw of the randomizer's bag. The engine uses it to fill the fake pieces when the known preview runs out (see [Fake next and branching](<../Engine/Fake next and branching>)).

## Types

The rule layer also names the board and ruleset types the AI speaks; the engine checks that they match its own.
