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

The engine caches board evaluations in a transposition table. When the search reaches a board that is still in the table, it reuses the stored evaluation instead of computing it again.

## What gets cached

For each candidate placement, the search builds the resulting board, hashes it, and looks it up:

- On a miss, the AI evaluates the board and stores the result under its hash.
- On a hit, the search reuses the stored evaluation.

In either case, the engine combines that evaluation with the placement's line clears, search depth, and game state to judge the candidate.

The table caches *board evaluations*, not search subtrees. Different landing paths that produce the same board share an evaluation. This captures the largest share of the search's redundant work because many placements, including different piece orders reached through hold, end on the same board.

Here is the flow for one candidate in pseudocode:

```python
def score(placement, depth):
    hit, result = table_for_depth(depth).lookup(placement.board, placement.roof)
    if not hit:
        result = evaluate(placement.board, placement.roof)
        table_for_depth(depth).store(placement.board, placement.roof, result)
    return apply_context(result, placement)
```

The lookup and any needed evaluation are the table's only roles. Everything after that is candidate-specific work outside the table. In this sketch, `apply_context` is a placeholder for the second evaluation stage described next, not a function to call.

## Cached versus not cached

The table works because the AI's evaluation has two stages:

1. A **board-only evaluation** is a pure function of the board and its roof. It costs more than the second stage, but depends on nothing else, so the table can cache it.
2. A **context-dependent scoring step** is cheap, runs for every candidate, and does not use the table.

The second stage combines the cached evaluation with information the board alone cannot provide:

- The line clears from this placement. Two placements can reach the same board while clearing different numbers of lines along the way.
- The candidate's search depth.
- Game state carried along the search path, such as the back-to-back chain and combo counter in the shipped ruleset.
- The pending pieces and hold state at the candidate's node.

None of this is cached because it is not determined by the board. Two placements can end on the same board and differ in all of these details. Keeping the cached value board-only lets every candidate that reaches the same board use the same entry.

## Replacement policy

The table has a fixed capacity. A search can touch more distinct boards than it can hold, so every new entry must eventually replace an old one. Even when the table is not full, hashes that land in the same bucket compete for its two slots. At the default capacity, this is the more common case.

Eviction is always safe. Because an evaluation depends only on the board, removing it cannot change the search result. It only means the engine may need to compute that evaluation again if the board appears later. The replacement policy therefore trades recomputation against hit rate, never correctness. It can stay simple: at each depth, one move holds only about beam width times branching factor boards; entries are written once and read many times; and tables are rotated or cleared between moves. A minimal policy for each bucket captures almost all available reuse.

Each bucket uses a least-recently-used policy. A marker identifies the next slot to evict. A hit or store moves the marker to the other slot, protecting the entry just touched. If a lookup misses while both slots are occupied, the marked slot is replaced, so the more recently used entry survives.

## Hashing

The hash covers only the board and its roof, not the piece being placed or the hold state. That is safe because the cached evaluation depends only on the board. Candidates that land different pieces on the same board can share an evaluation.

Only occupied rows below the roof contribute to the hash. The search never creates a board with occupied cells above the roof, so omitting the other rows loses no information.

## One table per search depth

The engine keeps one table for each depth in the horizon.

This separation makes updates between moves cheaper. Rotating or clearing the cache touches one depth's table instead of the whole cache. With one shared table, each entry would need a depth tag, and refreshing a depth would require scanning every entry. Since cached values depend only on the board, sharing a table across depths would still give correct answers. The separation only affects which entries survive between moves. Within one depth, candidates that reach the same board share an entry regardless of the piece placed. Boards that merely share a bucket may cause an eviction, but never a wrong answer.

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
