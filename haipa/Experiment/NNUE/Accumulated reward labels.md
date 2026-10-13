---
title: Accumulated reward labels
---

# Accumulated reward labels

**Status: refuted**

The first label recipe: train the net on the reward the teacher's search computed for the line it chose. This produced the first two nets, v1 and v2, and the failure that started the experiment series.

Shared vocabulary in [Glossary](<../../Engine/Glossary>); search terms in [Beam search](<../../Engine/Beam Search>). This page adds:

| Term | Meaning |
|------|---------|
| Teacher | The hand eval, acting as the label source during collection. |
| Label | The training target attached to a collected board. |
| Ante-peak spiral | Training on values that grow with the search horizon, teaching the net to chase reward behind the horizon. |
| Blocking survival | A recurring failure style: moves that keep the stack alive are skipped in favor of greedy building. |

## The hypothesis

The teacher's search already weighs attack chains and survival. Handing its line values to the net should transfer the full attitude in one step.

## What happened

The hand attitude adds reward per placement: attack and like-chain rewards accumulate along a line. A search value therefore grows with the horizon it was computed at, and a deeper teacher sees more reward. The nets learned to stack into danger to chase reward that only existed behind the horizon and died at the ante-peak.

The same mechanism explains a companion observation: a deeper teacher blocks survival more often than a shallower one, because it sees more of the reward behind the horizon.

## The verdict

**Refuted.** A label must be bounded against horizon growth. Every later recipe chases that constraint: single-board values, terminal values, sentinels.
