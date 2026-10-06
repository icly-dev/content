---
title: Overview
---

# Overview

Tuning improves the AI's judgment by adjusting the theta, a list of numbers. Each feature and reward in the evaluation has a corresponding value, and changing those values changes what the engine prefers. The goal is to find a theta that wins more battles than the shipped version.

The loop has four parts, each with its own page:

```text
theta candidates -> battles -> Elo ranking -> search step -> better candidates
```

- [Match API](<Match API>) defines the battle two thetas play, the seeds that pin it, and how a result becomes a score.
- [Matchmaking](<Matchmaking>) ranks one generation's candidates with an Elo tournament.
- [CMA-ES](<CMA-ES>) is the black box search that proposes each generation's candidates from those rankings.
- [Theta comparison](<Theta comparison>) settles whether a candidate really beats another before anyone adopts it.

## Glossary

| Term | Meaning |
|------|---------|
| Theta | The list of numbers the AI's judgment is built on, 44 in the shipped AI. Tuning searches this list. |
| Generation | One round of the loop: sample candidates, battle, rank, update the search. |
| Anchor | The shipped theta, playing every generation's tournament unchanged. It pins what the rankings mean. |
| Black box | A search that sees only outcomes. Battles return wins, ties and losses, never the inside of the evaluation. |

## Why battles decide

The evaluation combines many features and rewards, and its quality is best judged by outcomes: which theta survives more battles. There is no formula that turns a theta into a quality score, so tuning measures thetas by having them play and gives those results to a black-box search. The comparison page checks the final choice, since luck and small samples can make a candidate look better than it is.
