---
title: NNUE experiment history
---

# NNUE experiment history

**Status: ongoing record**

The distillation loop replaces the hand eval inside the search's field term with a distilled net and retrains it from the engine's own play. This page records every arm the loop has tried, including the refuted ones, and the mechanism each verdict revealed. The current experiment is in [Blending](<Blending>).

Shared engine vocabulary is defined in the [Glossary](<../../Engine/Glossary>); search terms in [Beam search](<../../Engine/Beam Search>); battle statistics (seat, seed pair, standard error) in [Theta comparison](<../../Tuning/Theta comparison>). This page adds:

| Term | Meaning |
|------|---------|
| Label | The training target attached to a collected board. |
| Board-static label | The hand eval of the collected board itself. |
| Successor-static label | The hand eval of the board after the teacher's chosen move. |
| Line label | A statistic, mean or minimum, of hand eval values along the teacher's line. |
| Death label | A record whose teacher line reaches a dead board, tagged with a sentinel value below any real evaluation. |
| Collection jitter | Random perturbation of the hand seat's tuning during collection games. |
| Self-state | A board produced by the net's own play, as opposed to the hand's. |
| Field collapse | The net's output spread shrinking on the states it plays, leaving the reward layer's incentives unopposed. |
| Ante-peak spiral | Training on values that grow with the search horizon, teaching the net to chase reward behind the horizon. |
| Blocking survival | A recurring failure style: moves that keep the stack alive are skipped in favor of greedy building. |

## The arms

### Accumulated reward labels (refuted)

The first two nets trained on the reward the teacher's search computed for its chosen line. The hand attitude adds reward per placement, so these values grow with the search horizon: a deeper teacher sees more reward. The nets learned to stack into danger to chase reward that only existed behind the horizon and died at the ante-peak. A consistent companion observation: a deeper teacher blocks survival more often than a shallower one, for the same reason.

**Verdict: refuted.** A label must be bounded against horizon growth.

### Successor-static labels (adopted, then superseded)

The label is the hand eval of the board after the teacher's chosen move: a one-placement lookahead. Bounded, and it produced the first deployable net, at 18% of games against production. It later became clear this recipe is a close clone of the hand eval's ordering.

**Verdict: superseded** by the board-static recipe.

### Line labels, mean and minimum (refuted)

Two arms labeled each board with a statistic along the teacher's full line: the mean and the minimum of the hand eval values seen along it. The minimum anticorrelated with the hand eval; both nets lost heavily. Any quantity accumulated along a line re-imports horizon growth in a softer form.

**Verdict: refuted.** The terminal value of the line is a different quantity and is under test in [Blending](<Blending>).

### Board-static labels and the static clone (adopted, then diagnosed)

The label is the hand eval of the recorded board itself: the most faithful clone of the hand ordering. The clone played the most hand-like of all arms and also upstacked the most stubbornly, which is what isolated the real mechanism below.

**Verdict: adopted as the base recipe**, after the diagnosis explained its weakness.

### Field collapse (diagnosis, not an arm)

On the boards the net itself produces, the quantized net's output spread collapsed to a fraction of the hand eval's spread and tilted toward rating tall stacks kindly. The reward layer's threat-gated incentives then ran unopposed, so the net upstacked stubbornly, and the calmer the opponent the worse it got. Every earlier symptom, clone stubbornness and calm-opponent spirals alike, follows from this one mechanism. It stayed invisible until pools contained self-states, because the collapse only appears on the played distribution.

### Self-state pools (adopted, largest single gain)

Collection mixes two seats: one hand seat, one net seat, so training contains the boards the net itself reaches. The collapse vanished: output spread restored to parity, the height tilt gone. Strength jumped from 25% to 36% in one step.

**Verdict: adopted.** Every pool since is mixed-seat.

### Outcome blending (refuted)

The label blended the search label with a win or loss tag from the game the board came from. Per-game tags mark "positions I was losing in" as globally bad, but survival paths pass through losing-looking positions, so the tags taught the net to avoid the very states survival runs through. Strength neutral, blocking survival worse.

**Verdict: refuted.** An honest outcome signal needs per-board winrate aggregation, which needs far more games than the loop can afford; search-derived labels remain the practical survival signal.

### Death-label retention (neutral, parked)

Records whose teacher line dies are kept instead of dropped, tagged with a sentinel value far below any real evaluation. The net learns a clean separation: death boards land far below alive boards in its output, with no spillover onto alive boards. Strength cost about four to five points; play visibly more defensive.

**Verdict: parked as a style variant.** Retuning theta around it is the designated way to trade the extra defense back for strength.

## The ladder

A 100-game duel carries about five points of standard error, so only the 300-game column is comparable.

| Net | 100 games | 300 games | What changed |
|-----|-----------|-----------|--------------|
| v4  | 18% | | first deployable: successor-static labels, jittered collection |
| v7  | 37% | | collection jitter removed |
| v7s | 25% | | board-static labels on mixed pools |
| v8s | 36% | | self-state pools added |
| v9s | 45% | 40.0% | one more self-state pool |
| v10s | 39% | 38.7% | one more pool |
| v11s | 44% | 36.3% | one more pool |
| v9sd | 46% | 35.3% | death labels on the v9s pools |

The 300-game column shows the pattern: fresher pools play cleaner but score no higher against production. The loop has plateaued with the current label recipe; what comes next is in [Blending](<Blending>).

## Protocol lessons

* Fix the search budget everywhere before comparing anything; the budget is part of the arm.
* One hundred games screens an arm, three hundred adopts one; never adopt on a screening reading.
* Never handicap an arm. An early handicap experiment cost about thirty-nine points of measured strength before it was caught.
* Collect with no jitter. The deployment condition is the only condition that matters.
* Labels always come from the production hand eval. The net's own recorded outputs are never targets; the previous net contributes state distribution only.
* Every new arm passes the same gauntlet: duel readings plus a host test for blocking survival before adoption.
