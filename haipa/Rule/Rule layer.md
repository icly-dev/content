---
title: Rule layer
---

# Rule layer

The AI has two parts: judgment, which scores boards and candidates, and the rule layer, which answers questions about the ruleset. The rule layer is the source of truth. The engine and the judgment adapt to it, not the other way around. Each answer uses the game state at the time of the question, as described in [AI interface](<../AI/AI interface>).

## Glossary

| Term | Meaning |
|------|---------|
| Ruleset | The rules the game runs under: the mino types, the rotation system, the bag, and the entry spots. Each rule layer carries one. |
| Mino | One piece shape the ruleset offers; the tetromino set has seven, I, O, T, S, Z, J, and L. |
| Rotation system | How pieces turn, and which kicks apply when a turn is blocked. |

## The board

The rule layer declares the board type, which serves as the canonical definition. The engine uses it to get the board's width and height rather than setting its own. The AI's judgment and movegen must use the same board and ruleset types, and the engine checks this at compile time.

In the ruleset this engine ships with, the board is 10 cells wide and 24 rows tall.

## The ruleset

The ruleset declares its mino types and rotation system. The shipped ruleset is deliberately simplified: its tetromino set and SRS rotation system with kicks come from the movement library [fast-reachability](https://github.com/icly-dev/fast-reachability), rather than the rule layer. Later rulesets can define their own piece sets and rotation systems in the rule layer.

## What the engine asks

The engine asks two questions, each for a different purpose:

- It asks for the absolute entry spot for a piece. This spot is the same in every situation. In the shipped ruleset, it is unrotated and sits near the top, just left of center. The AI starts from this suggestion and adjusts it to match the real game's entry behavior.
- It asks for a shuffled draw of the full mino set. In the shipped ruleset, that means all seven tetrominoes. The engine uses the draw to refill fake pieces when the known preview runs out (see [Fake next and branching](<../Engine/Fake next and branching>)).
