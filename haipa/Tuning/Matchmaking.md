---
title: Matchmaking
---

# Matchmaking

Each search generation proposes candidates, but it has no quality score for them yet. The tournament creates those scores by having candidates play one another and a fixed set of anchors. This page explains who plays, how often, and how each battle becomes a score. [CMA-ES](<CMA-ES>) uses those scores, and the battles themselves run through the [Match API](<Match API>).

## Glossary

| Term | Meaning |
|------|---------|
| Candidate | One theta proposed by the search for the current generation. |
| Appearance | One battle a candidate plays. Every candidate in a generation plays the same number. |
| Anchor | A fixed reference theta, outside the generation, that every candidate plays in the same way. |
| Common random numbers | Reusing the same seed pair for the same anchor slot, so every candidate faces the same deal and differences come from play, not luck. |
| Fitness | A candidate's mean battle score over all of its appearances. |

## Why anchors

A tournament of only the generation's candidates produces relative fitness. A candidate's score says how it did against this field, not how close it is to good. The field itself changes every generation, so a rising mean can mean better candidates or a weaker field, and the search cannot tell the difference. A battle between two weak candidates produces a winner just as a battle between two strong ones does.

Anchors fix the reference point. They are fixed thetas that stand outside the generation, are never sampled, and never change between generations. Every candidate plays them in every generation, so fitness becomes comparable across generations and the search has a stable target: a candidate scores well by beating the anchors, not by outlasting a weak field. The anchor list always starts with the shipped production theta, and extra anchors can be added to cover different styles.

Anchors also cut noise, through common random numbers. For a given anchor and repeat, the seed pair comes from the run seed and is identical for every candidate and every generation. Every candidate faces the same anchor, in the same seat, on the same deal, so a difference in the result comes from the candidate.

A [measured check](<../Experiment/Anchored versus unanchored tuning>) ran the tournament with and without the anchor: the anchor changed which direction the search took, but not how strong the result played.

## The field

A generation has 16 candidates by default, the population the search samples, plus the anchors. Only the candidates are scored for the search; the anchors are opponents and never play one another. Extra anchors are kept few, because each one adds battles to every candidate's generation: at the defaults, one anchor and an appearance quota of 64, a candidate plays 64 candidate battles plus 8 anchor battles.

## Who plays whom

The candidate battles form a round robin. At the defaults, 16 candidates meeting every pair four times covers 60 battles per candidate, and the remaining four appearances per candidate are paired at random, so everyone plays exactly 64. Seats are assigned at random and every battle draws a fresh seed pair from the generation's training stream.

The quota is chosen as a multiple of the opponent count so the round robin comes out even; any remainder is topped up with random pairings rather than left as an uneven field.

## The anchor battles

Every candidate plays every anchor eight times by default. Repeats use different seed pairs, and the seats alternate: the candidate goes first on even repeats and second on odd ones, so the seat advantage cancels over the batch without replaying a battle. Both seats draw from their own seed, so the two players never share a piece queue; the seed pair is held constant across candidates, not across the two sides of one battle.

## Why no mirrored rematch

A mirrored rematch replays a battle with the seats exchanged and folds the two scores, on the assumption that the pair cancels luck. Instead it cancels the difference: the replay's score is always one minus the original, so every pair averages one half whatever the thetas. The tournament spends those battles on fresh seed pairs. [Theta comparison](<Theta comparison>) has the measured result.

## How a battle becomes fitness

Each battle scores from the first seat's perspective: a death gives the opponent two points, and the player with the higher attack per line gets one more, so a win scores 1.0, a tie 0.5, and a loss 0.0.

Fitness is the candidate's mean score over its candidate battles and its anchor battles together. The generation also reports its mean score against the anchors, which shows whether the field as a whole is moving closer to them or further away.

The battles run on a thread pool, and a run can spread them to [remote helpers](<Remote helpers>) on other machines. Where a battle runs does not change its result, since a battle depends only on the theta pair, its seeds, and the options.

## Choosing the quota

More battles per candidate lower the noise in fitness, at a linear cost in time. The appearance quota and the anchor count are the two levers. On a candidate that looks promising, confirm it with a full [Theta comparison](<Theta comparison>) rather than trusting its tournament score.
