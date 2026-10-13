---
title: Board-static labels
---

# Board-static labels

**Status: adopted**

The label is the hand eval of the collected board itself: no teacher replay at label time, no lookahead. This is the base recipe of the current loop.

Shared vocabulary in [Glossary](<../../Engine/Glossary>). This page adds:

| Term | Meaning |
|------|---------|
| Board-static label | The hand eval of the collected board itself. |
| Static clone | A net trained on board-static labels, tracking the hand eval's ordering almost perfectly. |

## The hypothesis

If the goal is a fast, quantized stand-in for the hand eval inside the field term, the most faithful target is the hand eval value of the very board being evaluated.

## What happened

The clone, v7s, played the most hand-like of all arms and won 25% against production on the pools of its day. It also upstacked the most stubbornly, which is what exposed the real mechanism: see [Field collapse](<Field collapse>).

## The verdict

**Adopted as the base recipe.** Its weakness is not the label but the distribution: train it on the states the system actually plays and the collapse disappears, which is the [self-state pools](<Self-state pools>) fix.
