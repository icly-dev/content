---
title: NNUE blending experiment
---

# NNUE blending experiment

**Status: planned**

An experiment testing whether a blend of two evaluation sources, tuned by CMA-ES, can push gameplay past both pure systems. It couples two changes: a blend weight between the distilled net and the hand eval, and a new label recipe (line-end) that gives the net evaluation knowledge the hand eval does not contain.

Shared engine vocabulary (board, evaluation, status, depth, horizon, hold, search budget) is defined in the [Glossary](<../Engine/Glossary>); search terms (beam, leaf, survivor) in [Beam search](<../Engine/Beam Search>); battle terms (seat, seed pair, standard error, z) in [Theta comparison](<../Tuning/Theta comparison>); tuning terms (candidate, Elo, generation, anchor) in [CMA-ES](<../Tuning/CMA-ES>); match terms (Options, seat iters) in [Match API](<../Tuning/Match API>). This page adds:

| Term | Meaning |
|------|---------|
| Net (NNUE) | A small neural network trained to reproduce the hand eval's values, quantized so evaluating a board is fast. Distilled, not self-taught: its only teacher is the hand eval. |
| Hand eval | The hand-written per-board evaluation. |
| Teacher | The hand eval, acting as the label source during collection. |
| Field | The per-board evaluation term the search adds to the rewards it accumulates per placement. Today it comes from one source: the net, or the hand eval before the net existed. |
| Blend weight | The share of the field taken from the net; the rest comes from the hand eval. |
| Line-end label | Training target: the hand eval value of the final board of the teacher's greedy line. |
| Residual-roof tilt | The correlation between the net's error and stack height. Positive values mean the net rates tall stacks too kindly, the signature of upstacking. |
| Control arm | The blend trained on today's per-board labels, predicting the blend weight drifts to zero (pure hand). |

## Background

The distillation loop has been iterated five times: play collection games between the hand seat and the current net seat, label sampled boards with the hand eval, retrain the net, quantize it, and duel the result against production. The nets are versioned v8s through v11s in the loop's own numbering. Gains came entirely from data fidelity, never from parameter tuning:

* Removing collection jitter from the hand seat: 18% to 37% of games won against production.
* Training on the net's own states, which fixed the field collapsing on the played distribution: 25% to 36%.
* Retaining death labels (boards where the teacher's line dies) made the net play more defensively at a small strength cost; parked as a style variant.

The loop has plateaued: at 300 games the two strongest nets score 40.0% and 38.7% against production, statistically tied (the 45% and 39% ladder readings at 100 games were noise). The reason is structural. Every label is a hand eval value, so the net is a pure approximation of the hand eval. Its deviations from the hand eval on live states are approximation error, not new knowledge. Iterating the loop re-weights where the net trains, but the value ordering it learns is always the teacher's. The reachable fixed point is the hand ordering fit tightly on the states the system itself generates, which sits near the current hand configuration.

Retuning theta alone cannot raise this ceiling: theta reshapes how the field is consumed, not what the field knows.

## The hypothesis

A blend of two evaluation sources beats either pure system once the sources differ in what they know:

`field = w * net + (1 - w) * hand`

With today's labels the net carries no extra knowledge, so the Elo-optimal blend weight should drift to zero and the system degenerates to "hand eval with retuned theta" (the control arm confirms this). With line-end labels the net becomes "hand eval plus lookahead depth", and the blend trades static consistency against depth. CMA-ES is the right tool to find the mix because it optimizes winrate directly.

## The line-end label

The collection tool already replays teacher lines for other label modes. Line-end labels the recorded state with the final board's hand eval value instead of any accumulated intermediate reward.

* Why not search scores as labels: the engine accumulates reward per placement (attack, like-chains), so search values grow with the search horizon. Training on them produced the ante-peak death spiral of the first two nets, which kept stacking to chase reward that only existed behind the horizon. A bounded label is one that cannot grow with horizon; the faithful bounded label for an accumulating evaluator is the terminal value of the line.
* Why not a stronger teacher instead: deeper teacher search changes which line the teacher picks, and stronger teachers were observed to block survival more often. Line-end labels keep the selection and the data distribution unchanged and change only the target value.
* Lineage warning: two earlier line-based label recipes failed, but they accumulated per-line reward; the terminal value is a different quantity and does not inherit their failure mode.
* Cost: one new label mode plus one labeling pass per pool.

## The blend

* The evaluation branch computes both values and returns the weighted mix. CMA-ES tunes the blend weight together with the theta parameters.
* Speed cost: the hand eval runs at every leaf whenever the blend weight is above zero, so the blended system gives up a planned lever (making net evaluation nearly free, allowing a higher search budget). Accepted while the search budget stays at its current size.

## Protocol

* Data: one fresh collect from the strongest current seat. Keep the raw collection until the line-end pass completes, then delete it; the collection disk is small.
* Arms, one variable at a time:
  1. Control: blend over the existing per-board-label net. Prediction: the blend weight drifts to zero and strength approaches the hand baseline.
  2. Treatment: blend over the line-end-label net. This is the real test.
* Readouts: quick duel at 100 games; any adoption verified at 300 games; head-to-head at 200 games against the incumbent champion; host test for blocking survival after any adoption.
* Tuning: bounded pass (few generations, fresh tuning state, the net pinned), gated: no adoption on 100-game noise, style invariants (spread ratio, residual-roof tilt) checked every generation, small theta steps so the dataset does not go stale between generations.
* Reference points at 300 games against production: v9s 40.0%, v10s 38.7%, v9sd (death labels) 35.3%.

## Success criteria

* Treatment beats the incumbent champion beyond noise at 300 games: 44% or better clears two standard errors over 38.7%.
* Blocking survival not worse on the host test.
* Blend weight stabilizes strictly between zero and one: the net is contributing, not decorative.

## Results

TBD.
