---
title: Overview
---

# Overview

Tuning improves the AI's judgment by adjusting the theta, a list of numbers. Each feature and reward in the evaluation has a corresponding value, and changing those values changes what the engine prefers. The goal is to find a theta that wins more battles than the shipped version.

The loop has four parts, each with its own page:

```text
theta candidates -> tournament and anchors -> scores -> search step -> better candidates
```

- [Match API](<Match API>) defines the battle two thetas play, the seeds that pin it, and how a result becomes a score.
- [Matchmaking](<Matchmaking>) scores one generation's candidates in a round robin, with fixed anchors for a stable reference.
- [CMA-ES](<CMA-ES>) is the black box search that proposes each generation's candidates from those scores.
- [Theta comparison](<Theta comparison>) settles whether a candidate really beats another before anyone adopts it.

Battles can also run on other machines through [remote helpers](<Remote helpers>), which changes nothing about the loop.

## Glossary

| Term | Meaning |
|------|---------|
| Theta | The list of numbers the AI's judgment is built on. Each AI has its own theta, and tuning searches one at a time. |
| Generation | One round of the loop: sample candidates, battle, score, update the search. |
| Candidate | One theta sampled by the search for a generation. |
| Anchor | A fixed theta outside the generation that every candidate plays, so scores stay comparable across generations. |
| Black box | A search that sees only outcomes. Battles return wins, ties and losses, never the inside of the evaluation. |

## Why battles decide

The evaluation combines many features and rewards, and its quality is best judged by outcomes: which theta survives more battles. There is no formula that turns a theta into a quality score, so tuning measures thetas by having them play and gives those results to a black-box search. The comparison page checks the final choice, since luck and small samples can make a candidate look better than it is.
