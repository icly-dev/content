---
title: First-move selection experiment
---

# First-move selection experiment

An experiment testing how the beam search should turn its final answer into a move. The hypothesis came from the observation that the search throws away most of what it learned: it plays the first move of one single best leaf and ignores the rest of the surviving tree.

Shared engine vocabulary (board, evaluation, status, depth, horizon, hold, search budget) is defined in the [Glossary](<../Engine/Glossary>); beam search terms (beam, survivor, frontier, first move) in [Beam search](<../Engine/Beam Search>); battle terms (seat, seed pair, APL, standard error, z) in [Theta comparison](<../Tuning/Theta comparison>). This page adds:

| Term | Meaning |
|------|---------|
| Vote | One deepest survivor counting toward the first move its line started with. |
| Vote rule | The selection rule being tested: play the first move with the most votes. |
| Deepest-best rule | The shipped selection rule: play the first move of the single best candidate at the deepest expanded depth. |
| Consensus move | A first move that many surviving lines share. |

## The hypothesis

The beam search expands depth after depth, keeping a bounded set of survivors. Whatever the horizon, the played move is always a first move: each surviving candidate stores the first placement of its path. The question is which first move to pick when the search ends.

The shipped rule takes the deepest-best candidate and plays its first move. One leaf decides, and every other surviving line is discarded.

The hypothesis says the opposite is better: let the deepest survivors vote. Count how many of them go back to the same first move, and play the first move with the most votes, breaking ties by the best descendant of each move. The intuition is robustness: if many independent lines, each surviving on its own merits, still lead back to the same first move, that move is likely sound, while the single best leaf may be an outlier the rest of the tree does not support.

## Method

The vote rule was added to the engine as a battle option, applied per seat, and left off by default so every existing measurement is unchanged. Everything else about a battle stayed identical: the same production theta on both sides, so the two players differ only in how they turn a finished search into a move.

- The duel ran at the default battle options (search budget 100) over 2048 battles, then at search budget 200 over 1024 battles.
- Each battle was played exactly once over fresh seed pairs. The vote rule played the first seat in half the battles and the second seat in the other half, so the seat advantage cannot fake a result.
- A preflight check confirmed the rules genuinely differ: on identical boards they choose different moves on about 30 percent of decisions.
- The vote tally turned out unambiguous. Every surviving position expanded at the final depth produced the same number of scored placements, so counting those placements or counting the surviving positions gives the same vote ranking.

## Results

Mean score from the vote rule's perspective, where 0.5 is parity:

| Search budget | Battles | Vote mean | Verdict |
|---|---|---|---|
| 100 (default) | 2048 | 0.3311 ± 0.0104 | vote loses, z = −16.2 |
| 200 | 1024 | 0.3369 ± 0.0148 | vote loses, z = −11.0 |

Per-battle averages:

| Search budget | Rule | Attack | Lines cleared | Died |
|---|---|---|---|---|
| 100 | vote | 256 | 188 | 33% |
| 100 | deepest-best | 272 | 177 | 17% |
| 200 | vote | 237 | 176 | 33% |
| 200 | deepest-best | 255 | 163 | 17% |

The vote rule's mean sits far below the coin-flip score of 0.5. The z number in the table is that gap measured in standard errors; [Theta comparison](<../Tuning/Theta comparison>) explains how it is computed and why around two is the usual bar for calling a difference real. Attack, lines cleared, and deaths are the per-player battle report described in the [Match API](<../Tuning/Match API>) page, which also defines the attack per line tiebreak used above.

## Why the vote rule loses

The results are consistent across both budgets and the mechanism is visible in the averages.

The vote rule plays the consensus move. Consensus means many surviving lines share the move, which rewards placements that keep many mediocre futures alive. That shows up as more lines cleared (188 versus 177 at the default budget) but with lower attack per line (1.36 versus 1.53) and twice the deaths (33 percent versus 17 percent). Popular is not the same as good: a first move that keeps one excellent line and several forgettable ones loses the vote to a first move with no excellent line but many average ones, and the player then dies to the attack the popular move failed to send.

The deepest-best rule instead plays the first move of the highest-evaluated leaf. This is also the rule the shipped theta was tuned under: the evaluation learned which leaves matter while the deepest-best rule did the picking. The vote rule moves every decision off the distribution the evaluation was optimized for, so part of the loss is a mismatch between the rule and the tuned theta, not purely a property of voting itself.

## Conclusion

The hypothesis is refuted at both tested search budgets, with margins far beyond noise. The deepest-best rule stays the shipped behavior.

The vote rule remains available as a battle option for future experiments. The natural remaining regime is a narrow search budget, where few lines survive and the votes are sparse; the experiments above covered budgets of 100 and 200, where the beam is wide enough that popularity and quality separate cleanly.
