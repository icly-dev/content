---
title: Successor-static labels
---

# Successor-static labels

**Status: superseded**

The label is the hand eval of the board after the teacher's chosen move: a one-placement lookahead. This produced the first deployable net, v4.

Shared vocabulary in [Glossary](<../../Engine/Glossary>). This page adds:

| Term | Meaning |
|------|---------|
| Board-static label | The hand eval of the collected board itself. |
| Successor-static label | The hand eval of the board after the teacher's chosen move. |

## The hypothesis

One placement of lookahead makes the label bounded, a single board value, while keeping some of the teacher's selection knowledge: the label reflects the board the teacher chose to reach, not just the board it started from.

## What happened

The recipe was bounded and trained cleanly. The resulting net won 18% of games against production, the first deployable distillation. Over time it became clear the recipe is a close clone of the hand eval's ordering, which caps what the net can add.

## The verdict

**Superseded** by the [board-static label](<Board-static labels>), which trains on the recorded board itself and needs no teacher replay at label time. The clone ceiling it revealed is the subject of [Blending](<Blending>).
