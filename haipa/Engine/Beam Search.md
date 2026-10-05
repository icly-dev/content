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
| Frontier | The set of survivors at the depth currently being expanded. |
| Candidate | One placement along one path: the board it produces, its status, and a link back to the first placement of its path. |
| Iterative widening | Growing the beam gradually: start narrow, keep widening as long as budget remains, and stop when the budget runs out. The upstream engine this one derives from works this way. |
| Branching factor | How many children an average position produces, i.e. how many placements survive from one board. |
| Learned branching estimate | A running average of the observed branching factor at each depth, carried between moves and used to balance the schedule. |
| Tie-break | The rule that decides between equally good candidates. Here it is generation order, which makes the whole search deterministic. |

## Overview

The engine derives from [tetris_ai_runner](https://github.com/TetrisAI/tetris_ai_runner), and the search is the part that changed most. The upstream engine is an anytime search driven by wall-clock time: it starts with a narrow beam and *iteratively widens*, expanding a little more on every pass until the time limit expires. The result is a beam shape that emerges from however many widening steps fit in the budget, and a search whose outcome can vary with machine load.

haipa keeps the widening idea but removes the loop. Given an iteration budget *n*, it **pre-computes the entire beam limit schedule** for every depth, then runs a single breadth-first pass under those limits. The pre-computed schedule imitates the shape that iterative widening converges to, while the search itself becomes one deterministic sweep with no time measurement and no re-expansion of already-explored positions.

## One pass, layer by layer

The search expands the tree breadth-first, one placement per depth:

```python
def search(board, horizon, limits):
    frontier = admit(first_placements(board), limits[0])
    best = best_of(frontier)
    if all_agree_on_first_move(frontier):
        return frontier[0].first_move
    for depth in range(1, horizon):
        children = expand_and_score(frontier)
        if not children:
            break
        best = best_of(children)
        frontier = admit(children, limits[depth])
        if all_agree_on_first_move(frontier):
            break
    return best.first_move
```

Each surviving candidate remembers the *first* placement of its path (including whether it was a hold swap), so no matter how deep the search goes, the answer is always extractable as one concrete move for the current piece.

Every child is scored when it is generated. Scoring itself is where the [transposition table](<Transposition Table>) lives: the board evaluation is cached, and the context-dependent part of the status is applied per candidate.

## The beam limit schedule

The schedule is the heart of the design. Given the iteration budget *n* and the horizon, it decides how many survivors each depth may keep. Three ingredients shape it:

1. **A total budget.** The first layer of placements is capped at twice the width (`2 * max(2, n + 1)`); the remaining budget across the deeper layers scales with the same width times the horizon. The first layer sits outside the profile on purpose. It holds every legal placement of the current piece, about 34 in the shipped ruleset, all of them cheap to evaluate and all of them directly relevant, since one of them is the move the engine will actually play. The cap is set high enough to never bind, so no real move is discarded before it has been judged, and the Gaussian profile only decides how the much larger deeper layers share the rest of the budget.
2. **A Gaussian depth profile.** The budget is not spread evenly. Its distribution across depths follows a Gaussian centered at a configurable fraction of the horizon, with a configurable spread and a floor of 5 percent: depths near the center of the horizon keep most of the survivors, and both ends are starved on purpose. Early depths matter little because they are cheap to redo, and the last depths matter little because there is no time left to exploit their information.
3. **Branching-factor balancing.** The dome shapes the *evaluation work* of each depth, not the survivor count directly. The engine keeps an exponentially weighted average (weight 0.5 on the newest observation) of how many children each depth actually produced in past searches, and divides each depth's share by it: a depth that branches harder gets a smaller survivor cap, so that the number of scored children per depth follows the dome. Survivors therefore anti-correlate with branching, and the cap sequence is deliberately not smooth even when the dome is. The averages reset when a new game starts.

```python
def beam_limits(horizon, n, peak, spread, branch_estimate):
    width = max(2, n + 1)
    budget = 2 * width * (horizon - 1)
    mid = peak * (horizon - 1)
    weights = [0.05 + 0.95 * exp(-0.5 * ((d - mid) / max(abs(spread), 0.05)) ** 2)
               for d in range(1, horizon)]
    limits = [2 * width] * horizon
    for d in range(1, horizon):
        shape = weights[d - 1] / sum(weights)
        amount = budget * shape * (mean(branch_estimate) / branch_estimate[d])
        limits[d] = max(1, floor(amount))
    return limits
```

The animations below sweep the peak across the full horizon in the converged state (horizon 7, iteration budget 200, and the per-depth branching estimates converged to values measured from the engine: 68.0, 51.0, 68.7, 54.2, 33.1, 49.5, and 27.0 children per kept node at depths 1 through 6). Solid is what actually exists and enters the beam, the outline is the admission cap, and the blue bar is the fixed first-layer cap, which at this budget exceeds the roughly 34 possible first placements and never binds. With a narrow spread the bump is one or two depths wide; with a wider spread the survivors cover most of the horizon, while the anti-correlation with branching stays visible in both:

![Beam limits with spread 0.5 while the peak sweeps across the horizon](../../asset/beam_limits_spread_0.5.gif)

![Beam limits with spread 1.5 while the peak sweeps across the horizon](../../asset/beam_limits_spread_1.5.gif)

Dividing by branching has a property worth knowing: the total number of candidates evaluated per decision is essentially independent of the peak and the spread (within 0.1 percent across the full sweep on the measured profile above). That is what lets the tuner compare shape settings at equal work.

The branching estimates start at 1, so the first deep searches of every game run the pure dome shape before anything has been learned, and the estimates converge within a few searches (each search updates every depth it actually expands). During that warmup the equal-work property does not hold, since there is nothing to divide by; the animation below shows the same sweep before any estimate has been learned:

![Beam limits during warmup, spread 1.5, before the branching estimates have been learned](../../asset/beam_limits_warmup_spread_1.5.gif)

The two profile knobs (the peak position and the spread) come from the AI's configuration, so tuning the AI also retunes where the search concentrates its work.

## Admission and ranking

Children compete for the frontier slots through a bounded priority queue:

```python
def admit(child):
    if len(frontier) < limit:
        push(child)
    elif better(child, worst(frontier)):
        replace_worst(frontier, child)
    # otherwise the child is discarded
```

Ranking compares candidates by status, with ties broken by generation order: the earlier-generated candidate wins. Both rules together make the search fully deterministic, which matters because matches and tuning experiments must be reproducible.

Discarding a candidate is the beam search trade-off, and it is the one place where the search can be wrong: a placement pruned at depth 2 is never reconsidered, even if it would have led somewhere better. The schedule's job is to make that loss unlikely where it matters.

## Early halting

After the first layer is admitted, and again after every later layer, the search checks whether every surviving candidate agrees on the first placement. If the entire beam traces back to one move, expanding deeper cannot change the answer, so the search halts immediately and spends the remaining budget nowhere. The same applies if a layer produces no children at all.

## Unknown futures

When the piece queue beyond the known preview is unknown, the horizon extends into fake pieces, and a search over one guessed future cannot be trusted alone: the search branches over several sampled futures and lets them vote on the first move. The beam limit schedule is computed once for the combined horizon, known placements plus the fake tail, and every branch runs under the same caps, continuing from a shared frontier through the placements covered by known pieces.

That mechanism has its own page: [Fake next and branching](<Fake next and branching>).

## What pre-computing buys and costs

Pre-computing the schedule instead of widening iteratively is what makes the rest of the engine simpler:

- The budget is a count, not a clock, so a search decision takes exactly the same work every time and every game is reproducible.
- There is no bookkeeping for partial exploration, no re-expansion across widening passes, and no interaction between the time limit and the beam shape.
- The transposition table's across-move rotation works on a clean, completed search with a known horizon.

The cost is the loss of anytime behavior: the upstream engine can return a better answer if given more time mid-decision, while haipa commits to the schedule implied by *n* before it looks at the board. The budget must therefore be chosen offline, which is what the profiler and the tuner do.
