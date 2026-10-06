---
title: Transposition table
---

# Transposition table

## Glossary

Shared engine vocabulary (board, roof, placement, depth, horizon, hold, status) is defined in the [Glossary](Glossary). This page only defines what is specific to caching:

| Term | Meaning |
|------|---------|
| Transposition | Two different move orders that reach the same board. The table's name comes from chess programming, where this idea originated. |
| Hash | A 64-bit fingerprint of a board. Two identical boards always get the same hash; two different boards get the same hash only by rare accident. |
| Hit / miss | A lookup finds the board's hash in the table (hit) or does not (miss). Only a miss triggers a new evaluation. |
| Bucket | A fixed group of table slots that a hash maps to. This table uses buckets of two slots. |
| Eviction | Discarding a stored evaluation to make room for a new one. Safe, since an evaluation can always be recomputed. |

## Overview

The engine caches board evaluations in a transposition table so the same board position is never evaluated twice: when the search reaches a board shape it has already seen, it reuses the stored evaluation instead of recomputing it.

## What gets cached

Every candidate placement considered by the search is reduced to its resulting board, and that board is hashed and looked up:

- On a miss, the AI evaluates the board once and the evaluation is stored under the board's hash.
- On a hit, the stored evaluation is reused directly.

Either way, the engine then combines the evaluation with the placement's own line clears, current depth, and game state to judge the candidate.

So the table caches *board evaluations*, not search subtrees. Two different landing paths that produce the same board share one evaluation. This is the largest share of the search's redundant work, since many placements (and many piece orders reached through hold) end at the same board shape.

The flow for one candidate, in pseudocode:

```python
def score(placement, depth):
    hit, result = table_for_depth(depth).lookup(placement.board, placement.roof)
    if not hit:
        result = evaluate(placement.board, placement.roof)
        table_for_depth(depth).store(placement.board, placement.roof, result)
    return apply_context(result, placement)
```

The lookup and the potential evaluation are the whole interaction with the table; everything after that is per-candidate work outside the table. `apply_context` here is just a placeholder for the second stage of the evaluation described in the next section; it is not a function the reader could call.

## Cached versus not cached

The table is the reason the AI's evaluation is split into two stages:

1. A **board-only evaluation**: a pure function of the board (and its roof). It costs more than the second stage but depends on nothing else, which is what makes it cacheable. This is what the table stores.
2. A **context-dependent scoring step**: cheap, runs for every candidate, and never touches the table.

The second stage takes the cached evaluation and folds in everything that a board alone cannot tell you:

- the line clears made by this particular placement (two placements can end on the same board but clear a different number of lines on the way),
- the search depth at which the candidate sits,
- the game state carried along the search path, for example the back-to-back chain and the combo counter in the shipped ruleset,
- the pending pieces and hold state of the node the candidate belongs to.

None of that is cached, because none of it is a function of the board; two placements can end on the same board and differ in all of it. Keeping the cached value board-only is what lets one stored entry serve every candidate that reaches the same board.

## Replacement policy

The table has a fixed capacity, and the number of distinct boards a search touches can outgrow it, so the table eventually fills and every new entry must replace an old one. Boards whose hashes land in the same bucket compete for the same two slots even when the table is nowhere near full, which is the more common case at default capacity.

Eviction is always safe. A cached evaluation is a pure function of the board, so throwing one away never changes what the search computes; it only costs one recomputation of the evaluation the next time that board appears. The policy therefore only trades recomputations against hit rate, never correctness. It also does not need to be clever: within a single move each depth holds only about beam width times branching factor boards, entries are written once and read many times, and the tables are rotated or cleared between moves anyway. A minimal per-bucket policy gets almost all of the available reuse.

The policy itself is least-recently-used within each bucket. Each bucket keeps a marker naming the slot to evict next, and both a hit and a store move that marker to the other slot, protecting the entry just touched. When a lookup misses and both slots are occupied, the marked slot is the eviction target: within the bucket, the recently touched entry survives and the stale one is replaced.

## Hashing

The hash covers the board and its roof only, not the piece being placed or the hold state. That is sound because the cached evaluation depends only on the board: candidates that land different pieces into the same board shape can share an evaluation.

Only occupied rows below the roof contribute. The search never produces a board with occupied cells above its roof, so nothing distinguishable is lost by leaving the rest of the rows out.

## One table per search depth

The engine does not keep a single table. It keeps one table per depth of the real piece horizon, plus one table per depth of the sampled fake-piece horizon used when playing out unknown future pieces.

This separation matters for two reasons.

The first is mechanical: it makes the between-moves update cheap. Rotating or clearing the cache after a move (next section) touches one depth's table instead of the whole cache; a single shared table would need a depth tag on every entry and a pass over all of it to refresh one depth.

The second is about lifetime: the fake tables are cleared on every move while the real ones rotate, and keeping them apart stops the fake-heavy work from evicting real evaluations. Since the cached value depends only on the board, sharing one table across depths would never return a wrong answer; what the separation decides is what survives a move, not what a lookup may return. Within one depth, two candidates that reach the same board share the entry whatever piece landed, and two boards that merely share a bucket cost an eviction, never a wrong answer.

## Lifetime across moves

Between search calls, the engine decides how much of the cache survives:

- If the new board matches what the engine predicted after its previous chosen placement, the real tables are rotated by one depth: the table for depth *d + 1* becomes the table for depth *d*, because everything the previous search evaluated one move ahead is exactly at the current horizon now. The deepest table is then cleared.
- If the board does not match the prediction, all real tables are cleared.
- The fake-piece tables are always cleared, since the sampled future pieces change every move.
- On a game reset, everything is cleared and the prediction is forgotten.

The rotation is the main reason the table pays off in a live game: most of the evaluation work done for the chosen move stays valid for the next decision. Pictured across two moves:

```text
move N, search uses              move N+1, after choosing a
tables per depth                 placement that matches the prediction

depth 0  [ table A ]             depth 0  [ table B ]   was depth 1
depth 1  [ table B ]             depth 1  [ table C ]   was depth 2
depth 2  [ table C ]             depth 2  [ table D ]   was depth 3
depth 3  [ table D ]  chosen ->  depth 3  [ empty   ]   cleared
```

Everything the search evaluated at depth 1 or deeper during move N is still reachable, just one level shallower.

## Sizing

Table capacity is a compile-time build option (`TETRIS_TT_BITS`, 15 by default). Each extra bit doubles the entry count, and a larger table makes collisions rarer. Since there is one table per search depth, total memory also grows with the search horizon; the engine reports its exact memory usage, which includes all tables.

The profiler can report the table's hit and replacement rates when built with statistics enabled. A hit rate near 80 percent or above means the search is revisiting boards heavily and the cache is doing its job; a high replacement-to-store ratio means the table keeps overwriting fresh entries, and a larger capacity is worth trying.
