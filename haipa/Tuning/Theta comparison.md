---
title: Theta comparison
---

# Theta comparison

Tuning will eventually produce a theta that looks better, but that alone is not enough to adopt it. The comparison tool tests the claim with a large, carefully counted batch of battles, 128 by default. The battles run through the [Match API](<Match API>).

## Glossary

| Term | Meaning |
|------|---------|
| Seat | The side that moves first in a battle. Moving first can be systematically good or bad: the seat advantage. |
| Seed pair | The two numbers that pin one battle's piece streams; one battle per seed pair. |
| Mean | The average of a batch's scores: 1.0 for a first-seat win, 0.5 for a tie, 0.0 for a loss. |
| Standard error | How much the mean itself wobbles: the sample spread divided by the square root of the game count. |
| z | The gap measured in standard errors. Around two is the usual bar for significance. |

## Why not first to seven

The obvious format is to play until one side wins seven games, but it has three problems:

- **Too few games:** Seven to thirteen games cannot separate a real advantage from seed luck in a game this noisy.
- **No significance standard:** A seven-to-five result might show a real edge or might be a coin flip, and this format cannot tell the difference.
- **It ignores seat advantage:** A win may be due to the seat, not the theta.

It also stops early, so the game count depends on the results. Stopping early when a candidate is doing well systematically inflates false positives.

## What the tool runs

The tool runs two batches using one shared run seed, so you can reproduce a report later:

- **The duel:** The two thetas play over the requested number of seed pairs, with the first theta always in the first seat.
- **The seat self-match:** The first theta plays against itself over an independent batch of seed pairs. This measures the advantage of going first.

Each battle scores 1.0, 0.5, or 0.0 from the first seat's perspective. Both batches report a mean and standard error:

```text
platform=... seed=... games=...
A.bin vs B.bin: <duel mean> (+- <duel standard error>) [<loss> loss, <tie> tie, <win> win]
seat self-match: <seat mean> (+- <seat standard error>)
attack per piece (no multiplier): <duel value> vs <seat value> [<attack>/<pieces> vs <attack>/<pieces>]
verdict: <conclusion> (z=<z>, seat-adjusted z=<adjusted z>)
```

The first line records which AI was compared, the run seed, and the game count, so a report can be reproduced later. The attack line reports the attack each side sent per piece placed, before any scoring multiplier is applied, along with the raw totals behind those averages. It is context for how the two thetas differ, not part of the verdict.

The verdict compares the duel mean with a coin-flip score of one half. If the mean is two standard errors above one half, the first theta is declared better. If it is two standard errors below, the second theta is better. Anything between those thresholds is not a significant difference.

The report also shows a seat-adjusted figure, which subtracts the seat mean from the duel mean before measuring the difference in standard errors. This is for reference only. It assumes the seat bias measured from the first theta's self-match is symmetric, so it does not determine the verdict.

## Why battles are never mirrored

Mirroring a battle means replaying it with the seats exchanged on the same seed pair and folding the two scores together, on the assumption that the pair cancels luck. This tool never does it, but the reason is not bias.

The battle simulation is exactly seat-symmetric: both players start from the same position, each draws its pieces from its own seed, and garbage is exchanged only after both have moved. Swapping the seats and the seeds together replays the same two trajectories with the labels switched, so the two scores always add up to one. A [measured check](<../Experiment/Mirrored seed pairs>) found this exact complement in every one of 200 pairs, and no difference from an independent batch beyond the noise of the two readings.

Two consequences follow:

- **Folding pairs cancels the difference, not the luck.** Every mirrored pair averages exactly one half whatever the thetas, so a mirrored batch says nothing about which is stronger.
- **The second battle is a duplicate.** One battle per pair is usable, half of an independent batch, and the mirrored reading's standard error was 1.33 times the independent one in the same check.

Chess can mirror because the armies are identical and only turn order changes, so the second game is a genuine, differently played game. In a Tetris battle the seats have no differing rules: exchanging the seeds hands each player the other player's piece stream rather than re-rolling it. Only averaging over many independent seed pairs reduces the noise.

## How many games

The standard error of a win rate shrinks with the square root of the number of games. With 13 games, a 60 percent win rate is less than one standard error from a coin flip, far from significant. With 128 games, the same rate is just over two standard errors away, enough to support adoption. This takes roughly 20 times as much machine time, so the practical approach is to screen candidates with small batches, then confirm the strongest ones with at least 128 games.
