---
title: Self-state pools
---

# Self-state pools

**Status: adopted**

Collection mixes two seats: one hand seat, one net seat, so training contains the boards the net itself reaches.

Shared vocabulary in [Glossary](<../../Engine/Glossary>). This page adds:

| Term | Meaning |
|------|---------|
| Self-state | A board produced by the net's own play, as opposed to the hand's. |

## The hypothesis

A distilled net is only as good as its accuracy on the states it is asked to evaluate, and those are the states its own play generates. Training only on hand-generated states leaves exactly those boards out of distribution.

## What happened

Output spread restored to parity and the height tilt disappeared on self-states. Strength jumped from 25% to 36% in one step. The recipe also closed the gap between label variants: with self-states in the pool, static-label and lookahead-label nets converged.

## The verdict

**Adopted.** Every pool after it was mixed-seat. The base label is the [board-static](<Board-static labels>) recipe.
