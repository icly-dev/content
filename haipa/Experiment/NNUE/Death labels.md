---
title: Death labels
---

# Death labels

**Status: parked**

Records whose teacher line dies are kept instead of dropped, tagged with a sentinel value far below any real evaluation.

Shared vocabulary in [Glossary](<../../Engine/Glossary>). This page adds:

| Term | Meaning |
|------|---------|
| Death label | A record whose teacher line reaches a dead board, tagged with a sentinel below any real evaluation. |
| Blocking survival | A recurring failure style: moves that keep the stack alive are skipped in favor of greedy building. |

## The hypothesis

Death is a per-board fact, not a per-game judgment, so unlike [outcome blending](<Outcome blending>) it cannot poison the survival manifold. Boards whose line dies should sit far below boards whose line survives, giving the net a danger gradient in the board term.

## What happened

The separation came out clean: death boards landed far below alive boards in the net's output, with no spillover onto alive boards. Strength cost about four to five points against production; play visibly more defensive.

## The verdict

**Parked as a style variant.** The defense is a dial, not a defect: retuning theta around the death-label net is the designated way to trade the extra defense back for strength. See [Protocol](<Protocol>) for the tuning pass shape.
