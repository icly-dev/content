---
title: Field collapse
---

# Field collapse

**Status: closed**

A diagnosis, not an arm. It explains why faithful static clones upstack stubbornly and why the failure worsened the calmer the opponent.

Shared vocabulary in [Glossary](<../../Engine/Glossary>). This page adds:

| Term | Meaning |
|------|---------|
| Field | The per-board evaluation term the search adds to the rewards it accumulates per placement. |
| Field collapse | The net's output spread shrinking on the states it plays, leaving the reward layer's incentives unopposed. |
| Residual-roof tilt | The correlation between the net's error and stack height; positive values rate tall stacks too kindly. |

## The mechanism

On the boards the net itself produces, the quantized net's output spread collapsed to a fraction of the hand eval's spread and tilted toward rating tall stacks kindly. The reward layer's threat-gated incentives then ran unopposed, so the net upstacked stubbornly, and the calmer the opponent the worse it got: with no incoming garbage there is nothing to check the upstacking.

The diagnosis stayed invisible until pools contained self-states, because the collapse only appears on the played distribution. Earlier probes measured the net on states it never reached.

## The resolution

Mixed-seat collection, the [self-state pools](<Self-state pools>) recipe, restored the spread to parity and removed the tilt. It remains the largest single gain of the loop.
