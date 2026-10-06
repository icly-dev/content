---
title: Overview
---

# Overview

Tuning is the loop that improves the AI's judgment. The judgment is a list of numbers, the theta: every feature and reward the evaluation weighs carries one, and changing the theta changes what the engine prefers. Tuning searches for a theta that wins more battles than the shipped one.

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

The evaluation is a long chain of features and rewards, and its quality shows up only in outcomes: which theta survives more battles. No formula maps a theta to a quality score, so tuning measures thetas by playing them and lets a black box search read those measurements. The comparison page then guards the last step, because a win streak is easy to fake with luck and small samples.
