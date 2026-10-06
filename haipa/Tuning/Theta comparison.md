---
title: Theta comparison
---

# Theta comparison

Sooner or later tuning produces a theta that claims to be better, and a claim is not adoption. The comparison tool settles the claim the slow, honest way: many battles, counted properly. Run it as `python3 -m haipa.compare <theta.bin|defaults> <theta.bin|defaults> [games] [seed]`, 128 games by default. The battles themselves are the [Match API](<Match API>).

## Glossary

| Term | Meaning |
|------|---------|
| Seat | The side that moves first in a battle. Moving first can be systematically good or bad: the seat advantage. |
| Seed pair | The two numbers that pin one battle's piece streams; one battle per seed pair. |
| Mean | The average of a batch's scores: 1.0 for a first-seat win, 0.5 for a tie, 0.0 for a loss. |
| Standard error | How much the mean itself wobbles: the sample spread divided by the square root of the game count. |
| z | The gap measured in standard errors. Around two is the usual bar for significance. |

## Why not first to seven

The obvious format, play until one side wins seven, fails three ways:

- too few games: seven to thirteen games cannot separate a real edge from seed luck in a game this noisy,
- no significance standard: seven to five is either an edge or a coin, and the format cannot say which,
- the seat advantage is ignored: a win may belong to the seat, not to the theta.

It is also an early-stopping rule: the number of games depends on how they go, and stopping early when things go well systematically inflates false positives.

## What the tool runs

Two batches, drawn from one shared run seed so a report can be reproduced later:

- the duel, the two thetas against each other over the requested number of seed pairs, with the first theta always in the first seat,
- the seat self-match, the first theta against itself over an independent batch of seed pairs, which measures how much the first seat alone is worth.

Each battle scores 1.0, 0.5 or 0.0 from the first seat, and both batches report a mean and a standard error:

```text
seed=... games=...
A.bin vs B.bin: <duel mean> (+- <duel standard error>) [<loss> loss, <tie> tie, <win> win]
seat self-match: <seat mean> (+- <seat standard error>)
verdict: <conclusion> (z=<z>, seat-adjusted z=<adjusted z>)
```

The verdict compares the duel mean against the coin: a mean two standard errors above one half names the first theta better, two below names the second, anything in between is no significant difference. The seat-adjusted figure subtracts the seat mean from the duel mean before measuring in standard errors, and is printed for reference only: the seat bias is measured from the first theta's self-match alone and assumed symmetric, so it does not decide the verdict.

## Why battles are never mirrored

Game tooling often mirrors a match: play it twice with sides swapped and merge the two results, hoping luck cancels. This tool never does:

- the luck would not cancel: piece sequences behave like noise but do not invert when the seeds swap, so a lucky theta can win both halves and one fake signal gets counted twice,
- the merge dilutes skill: two games folded into one record turn a real edge into a tie the moment the weaker side steals one game on luck,
- it doubles the cost for the same information.

Chess can mirror because the armies are identical and only the turn order differs. A Tetris battle is simultaneous, and the real asymmetry is that each side sees its own piece queue; swapping seeds cannot cancel that, only averaging over many independent seed pairs can.

## How many games

The standard error of a win rate shrinks with the square root of the game count. At thirteen games, a sixty percent win rate sits under one standard error from the coin, nowhere near significant. At one hundred twenty-eight, the same rate is just over two standard errors, enough to adopt. The price is roughly twenty times the machine time, which is why the practical flow is: screen candidates cheaply with small counts, then confirm with a hundred twenty-eight or more before adoption.
