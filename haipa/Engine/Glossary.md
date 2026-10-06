---
title: Glossary
---

# Glossary

This glossary defines terms used across the engine docs. Feature pages link here and define only the terms specific to their topic.

| Term | Meaning |
|------|---------|
| Board | The playfield, a grid of occupied and empty cells. Two boards are identical when every cell matches. In the shipped ruleset, the grid is 10 cells wide. |
| Roof | The height of the tallest stack. Every row above it is empty. |
| Placement | Dropping the current piece, or the held piece, at a specific position and orientation, along with any resulting line clears. |
| Evaluation | A number the AI assigns to a board to measure how good it is. |
| Depth | How many pieces into the future a position is. Depth 0 is the current move. |
| Horizon | How far ahead the search looks, using the pieces already in the queue. |
| Hold | The one-piece stash a player can swap the current piece into. |
| Status | The AI's running assessment of a search path. It combines the board evaluation with context gathered along the way, such as line clears, combo, and back-to-back, into one value for comparison. |
| Search budget | A count that sets the beam size for one search decision. It is not a time limit, which keeps searches reproducible. |
