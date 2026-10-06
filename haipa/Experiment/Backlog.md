---
title: Experiment backlog
---

# Experiment backlog

**Status: proposed**

A technique-by-technique review of chess programming literature produced a list of candidate experiments for the engine. Chess and Tetris search overlap enough that the list is worth keeping, but several suggestions assume a chess-shaped search: a clock, recursive scoring, an opponent who moves between plies. This page records the backlog after checking each idea against how the [beam search](<../Engine/Beam Search>) actually works, so the promising ones are queued and the misfits are retired with their reasons.

Every experiment gets judged the same way: two thetas battle over fresh seed pairs as described in [Theta comparison](<../Tuning/Theta comparison>), with the change under test in one seat. A change the tuner should be able to price becomes a new tunable rather than a fixed constant.

## Glossary

Shared engine vocabulary (board, roof, placement, evaluation, depth, horizon, hold, status, search budget) is defined in the [Glossary](<../Engine/Glossary>); beam terms (frontier, survivor, candidate, first move, beam limit) in [Beam search](<../Engine/Beam Search>); caching terms in [Transposition table](<../Engine/Transposition Table>). This page adds the borrowed chess terms:

| Term | Meaning |
|------|---------|
| Futility pruning | Skipping a child whose best possible outcome cannot beat what the search already has. |
| Late-move reduction | Searching weak or late-ordered children at a shallower depth, and spending the saved work on the strong ones. |
| Quiescence search | In chess, extending the search through captures until the position settles. The analogue is searching past the horizon while the board is still mid-tactic. |
| Iterative deepening | Searching shallow first, then deeper, reusing earlier work between passes. |
| Killer and history moves | Remembering which placements scored well at a depth or across the game, and trying them early next time. |
| Lazy SMP | Running several searches in parallel on one shared cache and taking the best answer. |

## Verdicts at a glance

| Idea | Verdict |
|------|---------|
| Rank survivors by strength, not generation order | Ready: free today, unlocks the pruning ideas |
| Reserve beam space per first move | Ready: needs the split ratio as a tunable |
| Deepen undecided positions | Ready in a count-budget form |
| Late-move reduction and futility pruning | Ready after survivor ranking, margins need tuning |
| Tuck search pruning gate | Cheap side experiment |
| Forced downstack, opening book | Deferred: the policy is small, the book's value unproven |
| Search past the horizon (quiescence) | Reframed: the horizon ends at the visible preview |
| Transposition table with bounds | Reframed: the cache is correct as designed, only sizing applies |
| Garbage-aware search | Deferred: needs a ruleset hook, gains unproven |
| Killer and history ordering | Back of the queue: ordering only decides ties |
| Majority vote for the first move | Settled: tested and refuted |
| Parallel beams (Lazy SMP) | Not applicable: the budget is a count, not a clock |

## Ready to try

### Rank survivors by strength

Each layer's frontier is expanded in generation order. Every survivor is expanded, so the order changes nothing about the result today. But every pruning idea below wants a strength order to cut from. The change is to order survivors by status before expanding, keeping generation order as the tie-break. Since ties already resolve by generation order everywhere else, the search's answers should not move. Expected effect on its own: none. Value: it is the gate for the reduction idea.

### Reserve beam space per first move

Admission into the next layer is global: the strongest children everywhere compete for one pool. A first move whose lines score well early can fill the whole beam and starve every alternative, and once the survivors agree on that one first move, the search stops without the alternatives ever being compared. The fix reserves part of each layer's budget for each first move still alive, with the split ratio as a tunable. Reservations keep weak lines alive longer, which costs a little depth everywhere; the battles decide whether the robustness buys more than the depth costs.

### Deepen undecided positions

The search runs one pass at the full budget. The escalation form runs a narrow schedule first and reruns, deeper, only where the survivors still disagree. Work is not wasted between passes: evaluations persist within a move, and the tables rotate between moves, so the narrow pass's boards are already cached. What does not carry over from the chess version is time management. The budget is a count, not a clock, and each decision must do a fixed, reproducible amount of work. Escalation is triggered by disagreement between passes, never by elapsed time.

### Reduce and prune weak lines

Once survivors are strength-ordered, late weak children can be searched at reduced width, or dropped when even their best case cannot catch the leader. Two boundaries come from the existing design. The first layer never prunes: every legal placement of the current piece is scored, and one of them will be played. And pruning margins are exactly the kind of knob the tuner should price, so they enter as parameters rather than constants.

## Cheap side experiments

### The tuck search pruning gate

When no line has just been cleared and the current piece spawns above the roof, the pathfinder stops offering soft and sonic drops for pieces other than T, to keep searches fast. The price is tucked placements the search never sees. Whether the gate should open when roof margin allows is a plain question for the battle harness: cheap to flip, measurable either way.

### Downstack policy and a book

Two deferred leftovers: a forced clearing policy when the roof climbs past a threshold, restricting the beam to placements that clear lines, and a small opening book for the first placements on an empty board. The policy is a small gating change; the book is a data blob whose value is unproven. Both wait until the core experiments land.

## Needs a different shape

### Searching past the horizon

The chess suggestion was quiescence: extend the search when the position is volatile, with pending garbage, a half-built T-slot, or a combo in progress. The obstacle is information, not time. The horizon ends where the visible preview ends, so an extension past it means inventing pieces nobody can see. Two existing mechanisms already cover part of what an extension would buy: the hold swap extends the horizon by one layer, and the search tracks how soon each priority piece arrives in the preview. A true extension needs speculative piece generation, a design change rather than a tweak, so the volatile-position signals stay with the evaluation for now.

### A transposition table with bounds

The chess version stores search results with upper and lower bounds. That needs a recursive search that backs scores up the tree; the beam search instead keeps survivors layer by layer, so there are no bounds to store. The suggestion's premise, that different hold or combo states wrongly share a slot, misreads the design: the table caches only what the board alone determines, and hold, combo, and line clears are applied per candidate outside the cache. Two placements reaching the same board through different hold paths are supposed to share an entry. What remains actionable is sizing: more capacity through the compile-time build option, measured with the hit rates the cache statistics build reports.

### Garbage-aware search

Pending garbage is already priced by the evaluation, which raises the roof's cost and rewards canceling. The chess analogue would inject expected garbage rows into deeper boards during the search. The battle rules complicate this: garbage arrives only after a placement that clears nothing, and the hole column is random, so a deep board with guessed garbage is a guess about the future, not a position. The search core is also shared between rulesets, so ruleset-specific threat logic needs a hook rather than a change inside the core. Deferred until the core experiments land and the loss sources are measured: if deaths under pressure turn out rare, this stays unprofitable.

### Killer and history ordering

Remembering which landing shapes scored well, by column, rotation, spin type, or hold use, and trying them first next time. With admission already keeping the strongest children, generation order only decides ties and the last slots in a layer, so the expected effect is small. The idea stays queued behind the ones above.

## Settled

### Majority vote for the first move

Suggested as a fix for a count-based vote. The fuller version was already tested and refuted: the vote rule lost clearly at two search budgets, because popular is not the same as good. See [First-move selection](<First-move selection>). The shipped deepest-best rule stays.

### The 180 question

One suggestion was to test enabling 180 rotations in battles. The game has no 180 key, so battle search is correct to exclude them: the engine must only recommend inputs a player can press. The real inconsistency ran the other way. The profiler allowed 180 and so measured a superset of the game's moves; it now matches battle play. What remains from this item is the tuck search pruning gate above.

## Not applicable

### Parallel beams

Running several searches per decision with different widths on a shared cache buys strength only when strength per wall-clock second is the goal. Here the budget is a count of placements per decision, chosen before the game, and parallelism already goes to playing more games at once during tuning. Extra beams under a fixed per-decision budget change nothing, and under a wall-clock budget they would slow the tuning tournament. Dropped.

## Order of attack

Rank survivors first, then beam reservations, then escalation on disagreement, then reduction and pruning, with a battle verdict after each step. The cheap side experiments slot in anywhere; the reframed ideas wait for the core results to show whether their problem is real.
