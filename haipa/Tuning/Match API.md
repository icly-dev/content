---
title: Match API
---

# Match API

The match API plays battles between two thetas and reports what happened. Everything in tuning is measured with it: the tournament's rating updates and the comparison tool's verdicts both come from battles played here. The Python side exposes it as `haipa.match`; underneath sits the engine library, found through the `HAIPA_LIB` environment variable or the build directory.

## Glossary

| Term | Meaning |
|------|---------|
| Battle | One game between two thetas, played move for move until one dies or the move cap is reached. |
| Seed pair | Two numbers, one per player. Every randomness source in a battle derives from the player's own seed. |
| APL | Attack per line: a player's total attack divided by total lines cleared. The skill tiebreak between two survivors. |

## What one battle is

Two players take turns, the first player's move then the second player's, for up to the move cap each. On its turn a player asks its engine for one decision and places it. The placement's attack first cancels the player's own pending garbage, oldest packet first; whatever survives is sent to the opponent. A player that clears nothing lets its pending garbage rise into its stack, and dying to the rise ends the battle for that player.

Each player is built from its own seed:

- the piece stream draws full bags from one derived stream, so each player sees its own queue,
- the garbage hole columns come from another derived stream,
- the engine's sampling seed is derived once more and re-seeded every move with the move count mixed in.

Every randomness source in a battle is a function of the two seeds, so the same thetas with the same seed pair and the same options replay move for move. Measurements must not flicker, which is why battles are pinned this way.

## The battle rules

The shipped battle rules follow the runner's attack conventions:

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

A placement that qualifies for back-to-back, a tetris or any spin, adds the current chain depth: one line for most placements, two for a spin triple, and it keeps the chain alive; anything else breaks it. Every clearing placement adds the combo table's value for the running combo count, the same table the runner supplies.

Receiving works the other direction: the opponent's surviving attack joins the player's pending packets, and their total is what the engine sees as the incoming attack. A piece that would settle at or above the visible height dies as well.

## Options

Five knobs bound a battle, with defaults: the move cap at 3600 moves per player, the search budget at 100, the known preview depth at 6, the fake-piece horizon at 0, and the branch count at 1. They pass straight through to the engine call, so a battle is also a statement about how much work each decision may do.

## Results and scoring

A battle reports per player: total attack, total lines cleared, moves played, and whether the player died. Scoring is deliberately separate from the battle itself, so the measure can change without touching the games:

- a death hands the opponent two points,
- between two survivors, the higher APL takes the point,
- equal points tie.

The score reads 1.0 for the first player's win, 0.5 for a tie, 0.0 for the loss, always from the first seat.

## Batch, threads and theta files

Battles run one at a time or in a batch with a thread count, since tuning runs thousands of them. A helper also pits a list of candidates against one opponent over a list of seed pairs and returns the mean score per candidate. Thetas move in and out as plain number lists, and as theta files on disk: raw doubles, one per parameter, no header. The shipped defaults can be queried, which is where the search's starting point comes from.
