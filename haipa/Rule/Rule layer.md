---
title: Rule layer
---

# Rule layer

The AI has two halves: the judgment, which the engine calls per board and per candidate, and the rule layer, which answers what belongs to the ruleset rather than to judgment. The rule layer is the source of truth for the ruleset: the engine and the judgment adapt to it, never the other way around. The engine asks, the rule layer answers, and the questions come with the game state of the moment, as described in [AI interface](<../AI/AI interface>).

## Glossary

| Term | Meaning |
|------|---------|
| Ruleset | The rules the game runs under: the mino types, the rotation system, the bag, and the entry spots. Each rule layer carries one. |
| Mino | One piece shape the ruleset offers; the tetromino set has seven, I, O, T, S, Z, J, and L. |
| Rotation system | How pieces turn, and which kicks apply when a turn is blocked. |

## The board

The rule layer declares the board type, and that type is the canonical one: the engine adopts it and reads the board's width and height from it instead of fixing its own. The AI's judgment and movegen must use the same board and ruleset types, and the engine checks that at compile time.

In the ruleset this engine ships with, the board is 10 cells wide and 24 rows tall.

## The ruleset

The ruleset declares the mino types and the rotation system. The shipped ruleset is deliberately simplified: its tetromino set and its SRS rotation system with kicks come from the movement library, [fast-reachability](https://github.com/icly-dev/fast-reachability), rather than from the rule layer. Later rulesets define their own set and system in the rule layer.

## What the engine asks

Two questions, each tied to a use case:

- `suggest_spawn`: the absolute entry spot for a piece, the same for every situation. In the shipped ruleset it sits just left of the middle, near the top, unrotated. The AI's spawn call starts from it and adjusts it to what the real game does at the entry.
- `make_bag`: one shuffled draw of the full mino set, all seven tetrominoes in the shipped ruleset. The engine uses it to refill the fake pieces when the known preview runs out (see [Fake next and branching](<../Engine/Fake next and branching>)).
