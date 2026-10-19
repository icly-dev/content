---
title: Transposition table
---

# Transposition table

## Glossary

Shared engine vocabulary (board, roof, placement, depth, horizon, hold, status) is defined in the [Glossary](Glossary). This page only defines what is specific to caching:

| Term | Meaning |
|------|---------|
| Transposition | Two different move orders that reach the same board. The table's name comes from chess programming, where this idea originated. |
| Key | What a lookup matches on: the board, its roof, and the piece that will be in hold after the placement. |
| Hash | A 64-bit fingerprint of a key. Two identical keys always get the same hash; two different keys get the same hash only by rare accident. |
| Hit / miss | A lookup finds the key in the table (hit) or does not (miss). Only a miss triggers a new evaluation. |
| Bucket | A fixed group of table slots that a hash maps to. This table uses buckets of two slots. |
| Eviction | Discarding a stored evaluation to make room for a new one. Safe, since an evaluation can always be recomputed. |

## Overview

The engine caches the AI's evaluations in a transposition table. When the search reaches a position it has already judged, it reuses the stored evaluation instead of computing it again.

## What gets cached

For each candidate placement, the search builds the resulting board and looks it up under a key made from the board, its roof, and the piece that will be in hold after the placement:

- On a miss, the AI evaluates the position and the result is stored under that key.
- On a hit, the search reuses the stored evaluation.

In either case, the engine combines that evaluation with the placement's line clears, search depth, and carried game state to judge the candidate.

The table caches evaluations, not search subtrees. Different landing paths that reach the same key share an evaluation. This captures the largest share of the search's redundant work because many placements, including different piece orders reached through hold, end on the same board with the same piece in hold.

Here is the flow for one candidate in pseudocode:

```python
def score(placement, depth):
    key = (placement.board, placement.roof, placement.hold_after)
    hit, result = table_for_depth(depth).lookup(key)
    if not hit:
        result = evaluate(placement)
        table_for_depth(depth).store(key, result)
    return apply_context(result, placement)
```

The lookup and any needed evaluation are the table's only roles. Everything after that is candidate-specific work outside the table. In this sketch, `table_for_depth`, `evaluate`, and `apply_context` are placeholders for the mechanisms described here, not functions to call.

## Cached versus not cached

The table works because the AI's evaluation has two stages:

1. A **position evaluation** judges a board in the situation it was reached from. It may read the pending pieces and the hold slot, so it costs more than the second stage, but it can be cached because the key covers the board, its roof, and the piece that will be in hold.
2. A **context-dependent scoring step** is cheap, runs for every candidate, and does not use the table.

The second stage combines the cached evaluation with information the key does not carry:

- The line clears from this placement. Two placements can reach the same board while clearing different numbers of lines along the way.
- The candidate's search depth.
- Game state carried along the search path, such as the back-to-back chain and combo counter.

None of this is cached because it is not determined by the key. Two placements can end on the same board with the same piece in hold and differ in all of these details. Keeping the cached value free of them lets every candidate that reaches the same key use the same entry.

## Replacement policy

The table has a fixed capacity. A search can touch more distinct keys than it can hold, so every new entry must eventually replace an old one. Even when the table is not full, hashes that land in the same bucket compete for its two slots. At the default capacity, this is the more common case.

Eviction only costs work: an evicted evaluation is recomputed the next time that key comes up. The replacement policy therefore trades recomputation against hit rate. It can stay simple: at each depth, one move holds only about beam width times branching factor boards; entries are written once and read many times; and tables are rotated or cleared between moves. A minimal policy for each bucket captures almost all available reuse.

Each bucket uses a least-recently-used policy. A marker identifies the next slot to evict. A hit or store moves the marker to the other slot, protecting the entry just touched. If a lookup misses while both slots are occupied, the marked slot is replaced, so the more recently used entry survives.

## Hashing

The hash covers the board, its roof, and the piece that will be in hold after the placement. The hold slot belongs in the key because the evaluation may read it: two candidates that reach the same board with different pieces in hold can deserve different evaluations. A hold swap changes which piece sits in hold, so a swap arrival and a normal arrival on the same board key separately. The pending queue is not part of the key.

## One table per search depth

The engine keeps one table for each depth in the horizon.

This separation makes updates between moves cheaper. Rotating or clearing the cache touches one depth's table instead of the whole cache. With one shared table, each entry would need a depth tag, and refreshing a depth would require scanning every entry. The separation also keeps entries from different depths apart, so a lookup only matches an evaluation made at the same depth.

## Lifetime across moves

Between search calls, the engine decides which cached entries to keep:

- If the new board matches the prediction made after the previous chosen placement, the tables rotate by one depth. The table at depth *d + 1* becomes the table at depth *d*, because positions evaluated one move ahead are now at the current horizon. The deepest table is cleared.
- If the board does not match the prediction, all tables are cleared.
- When a game resets, the engine clears everything and forgets the prediction.

This rotation is what makes the table especially useful in a live game: most evaluations from the previous search remain valid for the next decision. Here is how it works across two moves:

```text
move N, search uses              move N+1, after choosing a
tables per depth                 placement that matches the prediction

depth 0  [ table A ]             depth 0  [ table B ]   was depth 1
depth 1  [ table B ]             depth 1  [ table C ]   was depth 2
depth 2  [ table C ]             depth 2  [ table D ]   was depth 3
depth 3  [ table D ]  chosen ->  depth 3  [ empty   ]   cleared
```

Every position evaluated at depth 1 or deeper on move N is still available, now one depth closer, on move N+1.

## Sizing

Table capacity is controlled by a compile-time build option (`TETRIS_TT_BITS`), set to 15 by default. Each additional bit doubles the number of entries and makes collisions less likely. Since the engine keeps one table per search depth, total memory use also grows with the horizon. The engine reports its exact memory use, including all tables.
