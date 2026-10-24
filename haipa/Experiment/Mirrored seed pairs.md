---
title: Mirrored seed pairs
---

# Mirrored seed pairs

**Status: closed** ✅ (question settled)

An experiment testing what a mirrored rematch actually measures. A mirrored rematch replays a battle with the seats exchanged on the same seed pair, and the usual argument for it is that the two games cancel each other's luck. The tuning notes used to claim the opposite, that mirroring biases a comparison instead of reducing its noise. Both statements came from intuition about piece queues and had never been measured, so this experiment measured them.

Shared battle vocabulary (seat, seed pair, mean, standard error, z) is defined in [Theta comparison](<../Tuning/Theta comparison>); board and placement terms in the [Glossary](<../Engine/Glossary>). This page adds:

| Term | Meaning |
|------|---------|
| Mirrored pair | A battle and its replay with the seats exchanged on the same seed pair. |
| Independent pair | Two battles on different seed pairs, one with each theta in the first seat. |
| Complement | Two scores that add up to one: 1 and 0, or 0.5 and 0.5. |

## The two claims

- The bias claim: a lucky theta can win both battles of a mirrored pair, so folding the pair overstates that theta.
- The noise claim: a mirrored pair still contains two battles instead of one, so it is a cheap way to reduce variance.

Both depend on one assumption: that exchanging the seats on the same seed pair produces a different battle. If it produces the same battle seen from the other side, neither claim survives.

## Method

Two thetas were compared: the shipped production theta in the first seat, and the mean of an in-progress tuning run, a candidate that plays clearly stronger. For each of 200 seed pairs the experiment ran four battles:

- mirrored, production theta in the first seat, the pair in its original order
- mirrored, candidate theta in the first seat, the same pair swapped
- independent, production theta in the first seat, on a fresh seed pair
- independent, candidate theta in the first seat, on another fresh seed pair

That is 800 battles on one run seed, spread across the available machines. Both thetas played the same ruleset and the same fixed search budget throughout.

## Results

The complement check is exact, not a matter of noise: in all 200 pairs the two mirrored scores added up to one, with an average gap of 0.000000 and a largest gap of 0.000.

Scores are from the shipped theta's perspective, 0.5 being parity:

| Reading | Mean | Standard error | Breakdown |
|---|---|---|---|
| Mirrored, production theta in the first seat | 0.1250 | 0.0234 | 25 wins, 0 ties, 175 losses |
| Independent, averaged over the pair | 0.1600 | 0.0176 | 3 pairs swept, 58 split, 139 lost |

The two readings differ by -0.0350 with a standard error of 0.0293, z = -1.19. Mirroring showed no detectable bias against the independent estimate at this battle count.

The mirrored reading was also less precise: its standard error was 1.33 times the independent reading's, close to the 1.41 that halving the usable battles predicts. Matching the independent precision would take about 1.77 times the battles.

## Why the mirror is not a second battle

The battle simulation is exactly seat-symmetric. Both players start from the same position, each one draws its pieces from its own seed, and garbage is exchanged only after both have moved; no rule treats the seats differently. Exchanging the seats and the seeds together therefore replays the same two trajectories with the labels switched: the board the production theta built is rebuilt by the candidate, and the attacks it received are the attacks it now sends. The scores add up to one by construction.

Three consequences follow:

- **A lucky theta cannot win both battles.** In the run, the production theta won 25 first-seat battles and lost all 25 of their swapped partners. Its 25 lucky streams were handed to the opponent intact, not re-rolled.
- **Folding pairs cancels the signal, not the luck.** Every mirrored pair contributes exactly one half to the batch mean, so a batch of mirrored pairs averages one half no matter which theta is stronger.
- **The second battle is a duplicate.** The only usable reading is one battle per pair, which is why the mirrored standard error matches the independent one after accounting for half the battles.

The intuition that failed was about seeds: swapping the seats does not re-roll the randomizer, it gives each player the other one's piece stream. Mirroring is not biased, it is redundant.

## Conclusion

The no-mirror rule stays, with a corrected reason. A mirrored rematch does not tilt a comparison, it spends a battle to repeat one already played, and an independent pair is exactly as expensive and more precise. The [Theta comparison](<../Tuning/Theta comparison>) page now states this reason. The exact complement is a property of the battle simulation, so it would need re-measuring if the rules ever treat the seats differently.
