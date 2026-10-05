---
title: Transposition table
---

# Transposition table

## Glossary

| Term | Meaning |
|------|---------|
| Board | The playfield: a grid of occupied and empty cells. Two boards are the same if every cell matches. In the ruleset this engine ships with, the grid is 10 cells wide. |
| Roof | The height of the tallest stack. Rows above the roof are always empty. |
| Placement | Dropping the current piece (or the held piece) at a specific position and orientation, including the line clears it causes. |
| Evaluation | A number (plus some auxiliary scores) the AI assigns to a board, measuring how good that board is. |
| Hash | A 64-bit fingerprint of a board. Two identical boards always get the same hash; two different boards get the same hash only by rare accident. |
| Depth | How many pieces into the future a position is. Depth 0 is the current move. |
| Horizon | The furthest depth the search looks ahead. The *real* horizon uses the actually queued pieces; the *fake* horizon extends beyond it with randomly sampled pieces. |
| Hold | The one-piece stash a player can swap the current piece into. |
| Hit / miss | A lookup finds the board's hash in the table (hit) or does not (miss). Only a miss triggers a new evaluation. |
| Beam search | The search strategy: at each depth it keeps only the most promising placements and expands those, discarding the rest. |

## Overview

The engine caches board evaluations in a transposition table so the same board position is never evaluated twice. When the search reaches a board shape it has already seen, it reuses the stored evaluation instead of recomputing it, which removes the single largest source of redundant work in the search.

The name comes from chess programming: a *transposition* is two different move orders that reach the same position. The same happens here, since many placements and piece orders converge on the same board shape.

## What gets cached

Every candidate placement considered by the search is reduced to its resulting board, and that board is hashed and looked up:

- On a miss, the AI evaluates the board once and the evaluation is stored under the board's hash.
- On a hit, the stored evaluation is reused directly.

Either way, the engine then combines the evaluation with the placement's own line clears, current depth, and game state to judge the candidate.

So the table caches *board evaluations*, not search subtrees. Two different landing paths that produce the same board share one evaluation. This is exactly the redundancy that dominates the search, since many placements (and many piece orders reached through hold) converge on the same board shape.

The flow for one candidate, in pseudocode:

```python
def score(placement, depth):
    h = board_hash(placement.board, placement.roof)
    hit, slot = table_for_depth(depth).lookup(h)
    if not hit:
        slot.result = evaluate(placement.board, placement.roof)
        slot.store(h)
    return apply_context(slot.result, placement)
```

The lookup and the potential evaluation are the whole interaction with the table; everything after that is per-candidate bookkeeping. `apply_context` here is just a placeholder for the second stage of the evaluation described in the next section; it is not a function the reader could call.

## Cached versus not cached

The table is the reason the AI's evaluation is split into two stages:

1. A **board-only evaluation**: a pure function of the board (and its roof). It is comparatively expensive but depends on nothing else, which is what makes it cacheable. This is what the table stores.
2. A **context-dependent scoring step**: cheap, runs for every candidate, and never touches the table.

The second stage takes the cached evaluation and folds in everything that a board alone cannot tell you:

- the line clears made by this particular placement (two placements can end on the same board but clear a different number of lines on the way),
- the search depth at which the candidate sits,
- the game state carried along the search path, for example the back-to-back chain and the combo counter in the shipped ruleset,
- the pending pieces and hold state of the node the candidate belongs to.

None of that is cached, because none of it is a function of the board. Two candidates with identical boards can sit at different depths, hold different combo counts, or have cleared different numbers of lines, and judging them identically would be wrong. Keeping the cached value board-only and applying context afterwards is what lets one stored entry serve every candidate that reaches the same board.

## Structure

The table is a fixed-size, open-addressing, two-way set-associative array:

- It holds a fixed number of entries set at compile time, 32768 by default.
- Entries are grouped in buckets of two slots each. A bucket is selected directly from the low bits of the hash, so a lookup touches at most two adjacent slots and never probes further.
- A hash value of zero is reserved as the empty-slot marker, so a freshly zeroed table is simply empty.

Each entry stores the 64-bit hash plus the cached evaluation. A default table occupies roughly 780 KiB in total.

## Replacement policy

Two facts make a replacement policy necessary. First, the table is fixed-size while the number of distinct boards a search touches is unbounded, so the table eventually fills and every new entry must displace an old one. Second, boards whose hashes land in the same bucket compete for the same two slots even when the table is nowhere near full, which is the more common case at default capacity.

Eviction is always safe. A cached evaluation is a pure function of the board, so throwing one away never changes what the search computes; it only costs one recomputation of the evaluation the next time that board appears. The policy therefore only trades memory traffic against hit rate, never correctness. It also does not need to be clever: within a single move the working set is roughly the beam width times the branching factor per depth, entries are written once and read many times, and the tables are rotated or cleared between moves anyway. A minimal per-bucket policy captures almost all of the available reuse.

The policy itself: each bucket carries a small flag naming the slot to evict next. Both a hit and a store redirect that flag away from the entry just touched:

- On a hit, the flag moves to the other slot, protecting the entry just used.
- On a store, the flag moves to the other slot, protecting the entry just written.

When both slots are occupied on a miss, the flagged slot is the eviction target. This is a minimal per-bucket least-recently-used policy: within a bucket, the recently touched entry survives and the stale one is replaced. Clearing the table is a single pass that zeroes all entries and flags.

One bucket with both slots occupied, and the flag pointing at slot 1 as the next eviction target:

```text
bucket
+---------------------------+---------------------------+-----------+
| slot 0                    | slot 1                    | next-out  |
| hash A   result A         | hash B   result B         | slot 1    |
+---------------------------+---------------------------+-----------+
```

The three cases in pseudocode:

```python
def lookup(h):
    if slot0.hash == h:
        next_out = 1          # hit slot 0: protect it, mark slot 1
        return slot0
    if slot1.hash == h:
        next_out = 0          # hit slot 1: protect it, mark slot 0
        return slot1
    if slot0.hash == EMPTY:
        return slot0          # free slot, caller fills it
    if slot1.hash == EMPTY:
        return slot1          # free slot, caller fills it
    return slot[next_out]     # both full: caller overwrites the stale one

def store(slot, h):
    slot.hash = h             # caller wrote the evaluation first
    next_out = other(slot)    # protect the entry just written
```

The invariant to keep in mind: `next_out` always points at the entry that has gone untouched the longest, so the flagged slot is exactly the one a new store should replace.

## Hashing

Board hashes come from a Zobrist-style scheme: a fixed table of random 64-bit values, one per board row, generated once per process and shared by all engine instances. The hash of a board folds each occupied row up to the stack roof into a rolling value with rotates and multiplications.

Only rows below the roof participate. Rows above the roof are always empty, so boards that differ only in unreachable sky hash identically, which keeps the hash cheap and avoids spurious distinctions.

The hash covers the board and its roof only, not the piece being placed or the hold state. That is sound because the cached evaluation depends only on the board: candidates that land different pieces into the same board shape legitimately share an evaluation.

## One table per search depth

The engine does not keep a single table. It keeps one table per depth of the real piece horizon, plus a separate set of tables for the sampled fake-piece horizon used when playing out unknown future pieces.

This separation matters for two reasons.

The first is mechanical: it makes the between-moves update cheap. When the cache is rotated or cleared after a move (next section), a per-depth layout reduces the whole operation to moving table references and zeroing a single table. With one shared table, every entry would need a depth tag, and refreshing the cache would mean visiting every index to clear just one depth's entries, or clearing the whole table and throwing away all reuse. Splitting by depth turns both operations into constant work per table instead of a pass over the entire cache.

The second is semantic: the hash does not encode the pending piece sequence. Positions at different depths are incomparable states, since they differ in which pieces are still to come; giving each depth its own table prevents a position from shallow preview depth from answering a lookup at a deeper one. Within one depth, entries that collide differ only by their landed piece, which as shown above is benign.

## Lifetime across moves

Between search calls, the engine decides how much of the cache survives:

- If the new board matches what the engine predicted after its previous chosen placement, the real tables are rotated by one depth: the table for depth *d + 1* becomes the table for depth *d*, because everything the previous search evaluated one move ahead is exactly at the current horizon now. The vacated deepest table is cleared.
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

Table capacity is a compile-time build option (`TETRIS_TT_BITS`, 15 by default). Each extra bit doubles the entry count, and a larger table reduces collision pressure. Since there is one table per search depth, total memory scales with the search horizon as well; the engine reports its exact memory usage, which includes all tables.

The profiler can report the table's hit and replacement rates when built with statistics enabled. A hit rate near 80 percent or above means the search is revisiting boards heavily and the cache is doing its job; a high replacement-to-store ratio means the table is thrashing and a larger capacity is worth trying.
