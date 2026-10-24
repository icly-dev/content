---
title: Match API
---

# Match API

The Match API runs battles between two thetas and reports the results. All tuning measurements use it, including the tournament's rating updates and the comparison tool's verdicts.

## Glossary

| Term | Meaning |
|------|---------|
| Battle | One game between two thetas, played move for move until one dies or the move cap is reached. |
| Seed pair | Two numbers, one per player. Every randomness source in a battle derives from the player's own seed. |
| APL | Attack per line: a player's total attack divided by total lines cleared. The skill tiebreak between two players. |

## What one battle is

The two players take turns, first one and then the other, up to the move cap for each player. On its turn, a player asks its engine for a decision and places the piece. The resulting attack first cancels that player's pending garbage, starting with the oldest packet. Any attack left over goes to the opponent. Garbage a player fails to cancel rises into its stack on the ruleset's own timing, and a rise that kills the player ends the battle for them. In the ruleset these docs describe, that timing is a placement which clears no lines.

Each player's randomness comes from its own seed:

- The piece stream draws full bags from one derived stream, giving each player a separate queue.
- Garbage hole columns come from another derived stream.

Those two streams are the whole of a battle's randomness: the engine makes no random choices of its own. With the same thetas, seed pair, and options, the battle replays move for move. Pinning the seeds keeps measurements consistent.

## The battle rules

Each AI fights with the attack conventions of its own ruleset, so the numbers below belong to the ruleset these docs describe:

```text
placement                    attack
single, no spin              0
double, no spin              1
triple, no spin              2
tetris                       4
single, mini spin            1
single, full spin            2
double or triple, spin       4 or 6
perfect clear                +6
```

A tetris or any spin qualifies for back-to-back. It adds the current chain depth, which is one line for most placements and two for a spin triple, and keeps the chain alive. A line-clearing placement that does not qualify breaks the chain, while a placement that clears nothing leaves it untouched. Every placement that clears lines also adds the combo table's value for the current combo count. The ruleset supplies that table.

On defense, the opponent's remaining attack is added to the player's pending garbage packets. The engine sees their combined total as the incoming attack. A ruleset can also declare a visible height (see [Rule layer](<../Rule/Rule layer>)); where it does, a player also dies if a piece would settle at or above it.

## Options

Three settings limit each battle. By default, the move cap is 3600 moves per player, the search budget is 100, and the known preview depth is 6. These settings pass directly to the engine, so they also determine how much work each decision can do. The search budget can also be set separately for the two seats, which lets a comparison give the sides different amounts of work.

## Results and scoring

For each player, a battle reports total attack, the same attack before any scoring multiplier, total lines cleared, moves played, and whether the player died. Scoring is separate from the battle, so the scoring measure can change without changing the games:

- A death gives the opponent two points.
- The player with the higher APL gets one more point.
- Equal scores count as a tie.

From the first player's perspective, a win scores 1.0, a tie scores 0.5, and a loss scores 0.0.

## Batch, threads and theta files

Battles can run one at a time or in batches with a specified thread count, since tuning may require thousands. A helper can also test a list of candidates against one opponent across a supplied list of seed pairs and return each candidate's mean score. A run can spread its battles across other machines with [remote helpers](<Remote helpers>), which play the same scheduled battles and report their scores back. Thetas are passed as lists of numbers or stored in files as raw doubles, one per parameter, with no header. A theta file must match the AI it is used with, since each AI has its own list. The shipped defaults can be queried, and a tuning run can start from them or from another theta.
