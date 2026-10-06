---
title: Glossary
---

# Glossary

Terms used across the engine documentation. Feature pages link here for shared vocabulary and define only what is specific to them.

| Term | Meaning |
|------|---------|
| Board | The playfield: a grid of occupied and empty cells. Two boards are the same if every cell matches. In the ruleset this engine ships with, the grid is 10 cells wide. |
| Roof | The height of the tallest stack. Rows above the roof are always empty. |
| Placement | Dropping the current piece (or the held piece) at a specific position and orientation, including the line clears it causes. |
| Evaluation | A number (plus some extra scores) the AI assigns to a board, measuring how good that board is. |
| Depth | How many pieces into the future a position is. Depth 0 is the current move. |
| Horizon | The furthest depth the search looks ahead. The *real* horizon uses the actually queued pieces; the *fake* horizon extends beyond it with randomly sampled pieces. |
| Hold | The one-piece stash a player can swap the current piece into. |
| Status | The AI's running assessment of a search path: the evaluation of the board plus context gathered along the way, such as line clears and chain counters. Statuses are ordered, so one path can be called better than another. |
| Search budget | How much work one search decision may do, counted as a number of iterations rather than time, which keeps searches reproducible. |
