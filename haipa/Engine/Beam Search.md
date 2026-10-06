---
title: Beam search
---

# Beam search

## Glossary

Shared engine vocabulary (board, roof, placement, evaluation, depth, horizon, hold, status, search budget) is defined in the [Glossary](Glossary). This page adds what is specific to the search:

| Term | Meaning |
|------|---------|
| Breadth-first search (BFS) | Exploring a search tree one depth at a time: every position at depth *d* is expanded before any position at depth *d + 1*. |
| Beam search | A breadth-first search that cannot afford every position, so each depth keeps only a fixed number of survivors, the beam, and discards the rest. |
| Beam limit | The number of survivors allowed at one depth. Here it is a per-depth schedule computed before the search starts. |
| Search width | The beam size the iteration budget translates into; the first layer's cap and the deeper layers' budget scale from it. |
| Frontier | The set of survivors at the depth currently being expanded. |
| Candidate | One placement along one path: the board it produces, its status, and a link back to the first placement of its path. |
| Iterative widening | Gradually growing the beam, starting narrow and widening while budget remains. The upstream engine that haipa is based on uses this approach. |
| Branching factor | How many children an average kept position produces, counted as the placements scored from it. |
| Learned branching estimate | A running average of the observed branching factor at each depth, carried between moves and used to balance the schedule. |
| Tie-break | The rule that decides between equally good candidates. Here it is generation order, which makes the whole search deterministic. |

## Overview

haipa is based on [tetris_ai_runner](https://github.com/TetrisAI/tetris_ai_runner), but its search is the part that changed most. The upstream engine uses an anytime search controlled by a clock. It starts with a narrow beam, then *iteratively widens* it on each pass until the time limit expires. The final beam shape depends on how many widening steps fit in the available time, so machine load can affect the result.

haipa keeps the widening idea but removes the loop. Given an iteration budget *n*, it **precomputes the full beam limit schedule** for every depth, then runs one breadth-first pass under those limits. The schedule approximates the shape produced by iterative widening, while the search itself becomes one deterministic sweep. It does not measure time or re-expand positions it has already explored.

## One pass, layer by layer

The search explores the tree breadth-first, adding one placement per depth. For each candidate, it uses exactly the landings reported by [movegen](<../Search/Movegen>):

```python
def search(board, horizon, limits):
    frontier = keep(first_placements(board), limits[0])
    best = best_of(frontier)
    if all_agree_on_first_move(frontier):
        return frontier[0].first_move
    for depth in range(1, horizon):
        children = expand_and_score(frontier)
        if not children:
            break
        best = best_of(children)
        frontier = keep(children, limits[depth])
        if all_agree_on_first_move(frontier):
            break
    return best.first_move
```

Sampled-future branching happens outside this loop. The search runs once for each branch, continuing from a shared frontier (see [Fake next and branching](<Fake next and branching>)).

Each surviving candidate stores the *first* placement in its path, including whether it used a hold swap. No matter how deep the search goes, it can therefore return one concrete move for the current piece.

The search scores each child as soon as it is generated. The [transposition table](<Transposition Table>) caches the board evaluation, while the context-dependent part of the status is applied separately to each candidate.

## The beam limit schedule

The schedule is the heart of the design. Given the iteration budget *n* and the horizon, it decides how many survivors each depth may keep. Three ingredients shape it:

1. **A total budget.** The first layer is capped at twice the search width, which is determined by the iteration budget. The remaining budget for deeper layers scales with the same width times the horizon. The first layer is kept outside the profile intentionally: it contains every legal placement of the current piece, about 34 in the shipped ruleset. These placements are cheap to evaluate and directly relevant because one will be played. The cap is high enough that it never binds, so no legal first move is discarded before evaluation. The Gaussian profile only distributes the much larger remaining budget across deeper layers.
2. **A Gaussian depth profile.** The budget is not divided evenly. A Gaussian distributes it across depths, centered at a configurable fraction of the horizon and using a configurable spread. Each depth gets at least 5 percent of the peak. Depths near the center keep most survivors, while both ends get less. Early depths need less budget because they are cheap to revisit and their board evaluations remain in the transposition table. The final depths also get less because there is little opportunity to use their results.
3. **Branching-factor balancing.** The Gaussian profile shapes the *evaluation work* at each depth, not the number of survivors directly. The engine tracks a learned branching estimate, a running average of how many children each depth produced in previous searches. It divides each depth's budget share by that estimate. A depth with more branching gets a smaller survivor cap, so the number of children scored at each depth follows the profile. Survivor counts therefore move in the opposite direction from branching, and the caps are intentionally uneven even when the profile is smooth. The estimates reset when the AI's parameters change.

The animations below move the peak across the full horizon after the branching estimates have settled. They use a horizon of 7, an iteration budget of 200, and the measured per-depth averages of 68.0, 51.0, 68.7, 54.2, 33.1, 49.5, and 27.0 children per kept node at depths 1 through 7. Solid bars show what actually exists and enters the beam; outlines show the caps. The blue bar is the fixed first-layer cap. At this budget it is higher than the roughly 34 possible first placements, so it never binds. With a narrow spread, the peak covers one or two depths. With a wider spread, survivors cover most of the horizon. In both cases, survivor counts move opposite to branching:

![Beam limits with spread 0.5 while the peak sweeps across the horizon](../../asset/beam_limits_spread_0.5.gif)

![Beam limits with spread 1.5 while the peak sweeps across the horizon](../../asset/beam_limits_spread_1.5.gif)

Balancing by branching keeps the total number of candidates evaluated per decision almost unchanged as the peak and spread move. Across the full sweep above, the difference is within 0.1 percent.

Branching estimates start at 1, so early deep searches follow the unadjusted profile. The estimates settle within a few searches, as each search updates every depth it expands. During this warmup, the work is not yet balanced because there is no learned branching rate to divide by. The animation below shows the same sweep before the estimates have been learned:

![Beam limits during warmup, spread 1.5, before the branching estimates have been learned](../../asset/beam_limits_warmup_spread_1.5.gif)

The AI's configuration sets the two profile controls, peak position and spread. Changing the configuration changes where the search concentrates its work.

## Keeping the survivors

Children compete for space in a priority queue capped at the beam limit:

```python
def keep(child):
    if len(frontier) < limit:
        push(child)
    elif better(child, worst(frontier)):
        replace_worst(frontier, child)
    # otherwise the child is discarded
```

Candidates are ranked by status. Ties go to the candidate generated earlier. Together, these rules make the search deterministic, which is important for reproducible matches.

Beam search can make a mistake when it discards a candidate. A placement pruned at depth 2 is never reconsidered, even if it would have led to a better result. The schedule aims to make that loss unlikely where it matters.

## Memory

The schedule also puts a hard limit on search memory. A layer never holds more candidates than its beam limit, so the total number of live candidates cannot exceed the sum of the caps across the horizon. Each candidate stores its resulting board, status, remaining pieces, and a link to the first placement in its path. The search does not keep a tree. It drops pruned candidates immediately, and they can return only through another path.

![The search tree with the interior depths covered by an overlay reading unstored nodes](<../../asset/beam_unstored_nodes.png>)

The figure shows what this means across the tree. The first layer supplies the possible answers, and the last layer determines which answer wins. The interior layers are not stored. Only the current frontier remains as the search moves from layer to layer; the transposition table may retain evaluations from interior boards.

Each layer reuses the previous layer's storage, so memory use levels off after the initial layers instead of growing with the tree. Branches run one at a time and each copies the shared frontier, so they add only one extra frontier to peak memory, not one per branch.

Most memory is used outside this loop by the per-depth transposition tables, described in [Transposition table](<Transposition Table>), and the between-decision state listed in [Overview](<Overview>). The engine reports total memory use, including the tables.

## Stopping early

After setting up the first layer and after each later layer, the search checks whether all surviving candidates agree on the first placement. If the entire beam points to one move, deeper searches cannot change the answer, so the search stops and leaves the remaining budget unused. It also stops if a layer produces no children.

## Unknown futures

When the queue extends beyond the known preview, the search uses fake pieces to extend its horizon. One guessed future is not enough to trust, so the search explores several sampled futures and lets them vote on the first move. It computes one beam limit schedule for the combined horizon, covering known placements and the fake tail. Every branch follows the same caps and continues from a shared frontier through the known pieces.

See [Fake next and branching](<Fake next and branching>) for details.

## What pre-computing buys and costs

Precomputing the schedule instead of widening the beam during the search simplifies the engine in several ways:

- The budget is a count, not a clock, so each decision does the same amount of work and every game is reproducible.
- The search does not need to track partial exploration, revisit positions across widening passes, or adapt the beam shape to a time limit.
- The transposition table can rotate between moves after a completed search with a known horizon.

The trade-off is that haipa loses anytime behavior. The upstream engine can improve its answer if it gets more time during a decision; haipa commits to the schedule implied by *n* before seeing the board. The budget must therefore be chosen before the game starts.
