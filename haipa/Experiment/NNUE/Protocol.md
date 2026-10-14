---
title: NNUE experiment protocol
---

# NNUE experiment protocol

**Status: standing**

Conventions every arm in this directory follows, learned mostly by paying for their absence.

Shared vocabulary in [Glossary](<../../Engine/Glossary>); battle statistics in [Theta comparison](<../../Tuning/Theta comparison>); tuning terms in [CMA-ES](<../../Tuning/CMA-ES>).

## Readings

* Fix the search budget everywhere before comparing anything; the budget is part of the arm.
* One hundred games screens an arm, three hundred adopts one; a 100-game reading carries about five points of standard error.
* Head-to-head duels between arms follow the same seed discipline: more games over more distinct seeds, never mirror rematches.

## Fairness

* Never handicap an arm. An early handicap experiment, zeroing part of one seat's tuning vector, cost about thirty-nine points of measured strength before it was caught.
* Collect with no jitter. The deployment condition is the only condition that matters; jittered collection once flattered the net by weakening its opponent.

## Labels

* Labels always come from the production hand eval. The net's own recorded outputs are never targets; the previous net contributes state distribution only.
* Every new arm passes the same gauntlet: duel readings plus a host test for blocking survival before adoption.

## The ladder

| Net | 100 games | 300 games | What changed |
|-----|-----------|-----------|--------------|
| v4  | 18% | | first deployable: successor-static labels, jittered collection |
| v7  | 37% | | collection jitter removed |
| v7s | 25% | | board-static labels on mixed pools |
| v8s | 36% | | self-state pools added |
| v9s | 45% | 40.0% | one more self-state pool |
| v10s | 39% | 38.7% | one more pool |
| v11s | 44% | 36.3% | one more pool |
| v9sd | 46% | 35.3% | death labels on the v9s pools |
| v12c | 41% | | one more pool |
| v12t | 42% | | one more pool, freshest labeled line-end |

The 300-game column shows the pattern: fresher pools play cleaner but score no higher against production. The loop plateaued with the current recipe, which motivates [Blending](<Blending>).
