---
title: Bag deduction
---

# Bag deduction

The search can only look as far as the pieces the runner has shown. At the end of that horizon the leaf is scored without knowing what comes next, which is the weakest point of the whole search. Often the next piece is not a guess at all: once a bag has given up all but one of its mino types, the piece that is left must be the next one drawn. The engine uses that fact to search one piece deeper.

Shared engine vocabulary (board, roof, placement, evaluation, depth, horizon, hold, status, bag) is defined in the [Glossary](Glossary). The search that consumes the extended horizon is [beam search](<Beam Search>).

## Glossary

| Term | Meaning |
|------|---------|
| Deduced piece | The one mino type of the current bag that has not been seen yet; the pieces already seen determine it completely. |
| Settling | The warm-up after the engine rebuilds its picture of the bag: the queue must advance as expected on three moves before a deduction is trusted. |

## Why the last piece is knowable

A bag contains one of each mino type, and the types leave it in a random order. Once every type but one has appeared, the remaining type has nowhere else to be: it is the next piece drawn from that bag, and it sits immediately after the visible queue.

The engine never guesses. It deduces a piece only when exactly one type of the current bag is unaccounted for. When two or more types are missing, their order is unknown, and the horizon stays at the visible queue.

## What the engine tracks

The engine counts the mino types it has seen since the current bag began: the active piece and every piece of the visible queue. It keeps that picture across calls, so a piece counted once is not counted again when it reappears in the next queue.

Between calls it rolls the picture forward by comparing the new queue with the previous one. When the queue advanced the way the picture predicts, the picture advances with it, and any piece that entered the queue at the far end is counted. When the queue does not match, or the active piece is not the one the picture expects, the engine rebuilds the picture from the current queue and settles again. A rebuilt picture must line up on three moves before a deduction is trusted, and a deduction that later turns out wrong triggers the same rebuild.

## What the extension changes

The deduced piece is appended to the visible queue, so the horizon grows by one and the deepest layer of the beam places that piece instead of leaving the position after the visible queue as a leaf.

That extra depth is worth spending because the leaf at the old horizon was already the weakest point of the search: it was scored without knowing what comes next. When the next piece is certain, the search prices its arrival exactly, and a position that dies to that piece stops being a survivor.

## What it does not cover

- Only the piece that completes the current bag is deduced. With two or more types still unseen, the search keeps its old horizon.
- The deduction is not statistical. The engine does not weight types by frequency or sample unseen queues; it uses a fact about the bag or nothing.
- The deduction is not part of every engine call. The runner that plays a live game turns it on, while the battle simulation used for tuning keeps its horizon at the visible queue, so a tuned configuration is measured without the deduced piece.
