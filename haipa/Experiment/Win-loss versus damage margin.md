---
title: Win/loss versus damage margin
---

# Win/loss versus damage margin

**Status: refuted** ❌

A tuning tournament ranks candidates by their mean battle score, and that score is dominated by the battle's result: a win, a tie, or a loss, with attack per line adding a smaller secondary point. The result is a coarse reading. Two candidates can win the same number of battles while one sends much more attack, so the proposal was to add a normalized damage margin to the battle score. The hope was that the extra information would rank candidates more accurately at the same number of battles, or reach the same accuracy with fewer.

Shared tournament vocabulary (candidate, appearance, anchor, fitness, generation) is defined in [Matchmaking](<../Tuning/Matchmaking>); battle terms (seat, seed pair) in [Theta comparison](<../Tuning/Theta comparison>). This page adds:

| Term | Meaning |
|------|---------|
| Battle score | The value one battle contributes to fitness, from the first seat's perspective. |
| Attack per piece | The attack a player sent over the battle divided by the pieces that player placed. |
| Damage margin | The first player's attack per piece minus the second player's. |
| Damage share | The damage margin divided by the total attack both players sent. |
| Ranking noise | How much an order measured from a limited number of battles differs from the order measured from many more. |
| Top-pick hit | The share of resamples in which the candidate ranked first at a given budget is the one ranked first by the full dataset. |
| Rank correlation | How closely two orderings agree: 1 is identical, 0 is unrelated, -1 is reversed. |

## The hypothesis

A win/loss result throws away the size of the win. If one candidate wins by a wide attack margin and another wins narrowly, the score calls them equal, even though the first may be stronger. A normalized margin keeps that information: the attack each player sent per piece, differenced between the two players. Normalizing per piece is necessary because a longer battle gives both players more chances to attack; without it the margin would mostly measure the battle's length. With the normalization, the margin should reflect how much better one player traded damage, independent of how many pieces the battle lasted.

If the margin carries real strength information, adding it to the score should reduce ranking noise at a fixed battle count. The experiment measured two things: whether the ordering agrees better with a high-battle ground truth, and whether the top candidate is chosen more often.

## Method

Eight candidates were sampled around a tuning center, with the same ruleset, the same fixed search budget, and one run seed. Every pair of candidates played 64 battles, for 448 battles per candidate. Each battle recorded both players' attack, clears, pieces placed, and death, so the outcome and the margin could both be computed from the same battles.

The full dataset orders the candidates by mean battle score, and that order is the ground truth for this experiment. It is not exact: splitting the 64 battles per pair into two halves and ranking each half against the other gave a rank correlation of 0.93, so the closest places are not certain even at this count. The top two candidates scored 0.781 and 0.752, a 3-point gap; several lower pairs were closer.

The comparison re-read the same recorded battles at smaller budgets: 9, 18, 27, 36, and 54 battles per pair of candidates, which is 63 to 378 candidate appearances. At each budget the experiment drew 2000 random subsets of whole seed pairs, so every candidate kept the same number of battles and faced the same deals, and averaged the rank correlation against the full-dataset order. For the top-pick hit it counted how often the best candidate under the subset was the best candidate under the full dataset.

The estimators were:

- the plain win/loss score,
- the win/loss score plus a weight times the damage margin, with weights from 0.25 to 8,
- the damage margin alone,
- the damage share alone,
- the player's own attack per piece, without a win/loss term,
- small weights of a survival-share margin and an attack-per-line margin.

A weight of 1 adds the margin at its measured size against a win worth 1; larger weights let the margin dominate the result.

## Results

At 63 candidate appearances, close to the tournament's default quota of 64, the plain score ranks at 0.924. The margin alone is far weaker, and the mix does not improve on the plain score:

| Estimator at 63 appearances | Rank correlation |
|---|---|
| Win/loss score | 0.924 |
| Win/loss plus 0.25 damage margin | 0.920 |
| Win/loss plus 1 damage margin | 0.915 |
| Damage margin alone | 0.761 |
| Damage share alone | 0.773 |
| Attack per piece alone | 0.794 |

Every weight made the ordering worse, and the loss grew as the weight grew. The small weight looks harmless by rank correlation, but it is not harmless where it matters. The top-pick hit, the chance of identifying the best of the eight, falls from 0.707 to 0.636 at 63 appearances, and from 0.780 to 0.722 at 126 appearances. The broad order barely moves; the close top pair does.

Without the margin, the top-pick hit improves with battle count as expected:

| Candidate appearances | Top-pick hit |
|---|---|
| 63 | 0.707 |
| 126 | 0.780 |
| 189 | 0.842 |
| 252 | 0.916 |
| 378 | 0.995 |

A survival-share margin and an attack-per-line margin, tested at small weights, also failed to beat the plain score beyond resampling noise. No continuous margin beat the plain score, and every one lost ground as its weight grew.

## Why the margin does not help

The outcome is the event the tournament is trying to improve: who wins. The margin is a correlate of that event, and a noisy one. It measures how much attack each player sent per piece, but attack is shaped by the opponent and by the battle itself. A candidate facing an opponent who survives longer gets more chances to attack and to receive garbage, so the margin partly measures the battle's shape rather than the candidate's strength. Normalizing per piece removes the piece-count confound, but not the opponent's influence.

The measured strength of the signal is the direct evidence. Alone, the damage margin ranks candidates at 0.76 against the full dataset, while the plain score ranks them at 0.92 at the same budget. Combining a weak, noisy signal with a strong one adds variance; it can only help if the weak signal corrects cases the strong one gets wrong, and here it does not. The monotonic degradation with weight is the signature: the more the score trusts the margin, the worse the ranking.

The discrete outcome also has low variance by construction. It discards the size of the win, and the size is the noisy part. A 3-point true strength gap produces margins that overlap heavily across battles, so the average margin separates the candidates less reliably than the average result does.

The top-pick result shows why rank correlation alone can mislead. At the smallest weight the overall order is nearly unchanged, yet the best candidate is picked 7 points less often. The margin shuffles the candidates who are already close, which is exactly the group the search needs to resolve.

## What this means for the battle budget

The starting claim was that a reliable decision between close candidates needs hundreds of battles. The experiment confirms it, and gives the number for this field. For the 3-point top gap, about 300 candidate appearances reach a 95% top-pick hit, and about 450 to 500 reach near-certainty. At the default quota of 64 candidate appearances the hit is about 71%; at 128 it is about 78%. Narrower gaps need more.

The lever is the battle count, not the scoring rule. The cost is linear in battles, and battles spread across [remote helpers](<../Tuning/Remote helpers>), so extra appearances can be bought with machines rather than with a different signal. The anchor battles are not a substitute: they are fixed, common-seed opponents that keep the generation-to-generation reference stable, and they add only a small, low-noise component to the fitness. The close candidate-to-candidate comparison is where the noise lives.

The measurement is on the toj ruleset. The mechanism, a binary result being a lower-variance estimate than a continuous margin, is not ruleset-specific, but the exact budget curve is.

## Conclusion

The normalized damage margin does not reduce ranking noise at equal battle budget. It leaves the broad order roughly unchanged and makes the close top pair harder to resolve, so the tournament keeps its win/loss score. The experiment did not change the shipped scoring.

The answer to the original concern is a budget answer: a close field needs hundreds of appearances per candidate for a confident decision, and more battles is the only lever that worked. At 64 appearances the best candidate is missed about three times in ten, and the hit rate only approaches certainty around 450 to 500.
