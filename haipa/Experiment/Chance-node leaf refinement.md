---
title: Chance-node leaf refinement experiment
---

# Chance-node leaf refinement experiment

**Status: refuted** ❌

An experiment testing whether a one-ply expectation over the unseen pieces can fix the horizon effect: the beam search ends at the visible queue, and its deepest survivors are evaluated without knowing what comes after the horizon, so a leaf that looks best now may be a trap once any of the possible next pieces arrives.

Shared engine vocabulary (board, evaluation, status, depth, horizon) is defined in the [Glossary](<../Engine/Glossary>); beam search terms (beam, survivor, frontier, leaf) in [Beam search](<../Engine/Beam Search>); battle terms (seat, seed pair, z) in [Theta comparison](<../Tuning/Theta comparison>). This page adds:

| Term | Meaning |
|------|---------|
| Chance node | A hypothetical position reached from a leaf by dealing one possible next piece. |
| Chance value | A leaf's raw evaluation blended with an expectation over its chance nodes. |
| Worst blend | The chance value using a weighted mix of the per-piece mean and the per-piece worst outcome. |

## The hypothesis

The beam admits survivors depth by depth using raw evaluation. At the final depth the winner is chosen among the surviving leaves, and each leaf is scored only on what is visible: the board after its last placement plus the remaining visible queue. Two leaves that tie under that score can differ sharply in what they allow next: one keeps a good placement available for every piece the randomizer might deal, the other dies or decays on some of them.

The hypothesis says that re-scoring the top leaves as an expectation over the seven possible next pieces picks the robust leaf over the fragile one, and that this shows up as a match win rate above parity.

## Method

The refinement was implemented in the engine, gated by a new 45th theta parameter (`chance`, default 0) so production behavior was untouched until a weight was adopted:

- At the final beam depth the surviving children were screened into a top-k set, then each leaf was re-scored. For each of the seven piece types the leaf was copied as a parent, a normal spawn was dealt, and the best child placement was evaluated. The chance value was the leaf value plus a weight times the marginal over pieces, with the leaf that has no legal continuation scored minus infinity.
- Two bookkeeping variants were tested: skipping dead chance outcomes, and including them with their encoded death value. The second removes a survivorship bias in principle.
- A third variant blended the per-piece mean with the per-piece worst outcome, `w * worst + (1 - w) * mean`, so weight 1 is the classic paranoid defense against fragile leaves.
- The refinement uses only public randomizer knowledge, the seven piece types, never the unseen queue, and chance children that still have visible queue left are evaluated with the true environment.
- Cost was measured with the profiler at default settings: about 18 percent more searches and 6 percent more wall time than the unrefined search.
- A trace diff confirmed the rules genuinely differ: the refined search chooses a different move on essentially every decision.

## Results

Duels of 100 battles (seed 1), production theta versus the same theta with a chance weight, from the refined side's perspective where 0.5 is parity. The seat self-match scored 0.47, and both raw and seat-adjusted z are reported as [Theta comparison](<../Tuning/Theta comparison>) prints them.

| Variant | Weight | Mean | Verdict |
|---|---|---|---|
| mean, skip dead | 0.5 | 0.4000 | refined loses, z = −2.03 (seat-adjusted −2.64) |
| mean, skip dead | 1.0 | 0.5100 | no significant difference, z = 0.20 (seat-adjusted 0.80) |
| mean, skip dead | 2.0 | 0.4000 | refined loses, z = −2.03 (seat-adjusted −2.64) |
| mean, include dead | 0.5 / 1.0 / 2.0 | identical battle counts | deaths are rare at leaf depth, so both variants coincide |
| worst blend | 0.25 | 0.5600 | no significant difference, z = 1.20 (seat-adjusted 1.80) |
| worst blend | 0.5 | 0.4500 | no significant difference, z = −1.00 (seat-adjusted −0.40) |
| worst blend | 1.0 | 0.4600 | no significant difference, z = −0.80 (seat-adjusted −0.20) |

No tested form or weight reaches a significant win, and two weights lose significantly.

## Why the refinement does not help

The leaf evaluation already sees the true next queue: it prices the immediate continuation, including which placements keep the player safe from the piece actually coming. The chance expectation therefore mostly re-measures quality the evaluation already encodes, and what it adds is a near-constant one-piece marginal cost that cancels out when comparing leaves. What remains is a reordering among near-equal survivors, and the survivors were already selected by myopic admission at earlier depths, so a line the beam pruned before the final layer cannot be recovered by re-scoring the leaves that beat it.

The worst blend shifts the balance toward paranoia but lands within noise in both directions: avoiding fragile leaves neither wins nor loses, because at these depths fragile leaves are rare and the evaluation's danger terms already avoid them.

## Conclusion

The hypothesis is refuted at every tested weight within 100-battle noise, with no trend that justifies a longer tuning run. The prototype was discarded and the engine keeps its shipped behavior: final-depth leaves are ranked by raw evaluation alone.

The negative result also localizes the horizon effect: if it exists in this engine, it lives in the admission decisions before the final layer, not in the choice among the leaves those decisions left alive.
