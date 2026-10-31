---
title: Shared queue and garbage seeds
---

# Shared queue and garbage seeds

**Status: closed** ✅ (question settled)

Every battle is pinned by a seed pair: it decides which pieces each side draws and when garbage arrives. Those draws are a large part of a battle's outcome, and the tournament allocates them in one of two ways. It can give every battle its own seed pair, so no two battles share a deal, or it can give every battle in a round one shared pair, so all the candidates in that round face the same pieces and the same garbage. The second arrangement is a common-random-numbers design: within a round, candidates are compared on the same deal rather than on luckier or unluckier ones.

The question was whether that shared deal drives what the tuner decides, and whether it produces a stronger theta. A first pass suggested a large gain, but it changed the pairing structure at the same time, so it could not separate seed sharing from schedule structure. The experiment was then repeated with the pairings held equal and with the seed variable isolated, and the answer changed.

Shared battle vocabulary (candidate, appearance, anchor, fitness, common random numbers) is defined in [Matchmaking](<../Tuning/Matchmaking>); seat, seed pair, mean, standard error, and z in [Theta comparison](<../Tuning/Theta comparison>); the search's mean and sigma in [CMA-ES](<../Tuning/CMA-ES>); board terms in the [Glossary](<../Engine/Glossary>). This page adds:

| Term | Meaning |
|------|---------|
| Independent seeds | Each battle draws its own seed pair from the generation's stream. |
| Shared seeds | Every battle in one round uses the same seed pair. |
| Balanced round | A round that pairs every candidate exactly once and gives every pairing the same seed pair, with seats alternating by round. |
| Matched queues | A comparison in which both players draw from the same queue and garbage seeds, so neither side gets the luckier deal. |

## What the two schedules do

The shipped tournament plays a complete round robin: every pair of candidates meets several times, and every battle draws a fresh seed pair and random seats. That is the independent design. The alternative gives each round one seed pair taken from the generation's stream and holds it for every pairing in the round, so a round is a set of battles on the same deal. Both designs can be given the same battle budget. With eight candidates, four meetings per pair in the round robin and 28 shared rounds give every candidate 28 candidate battles, and the same eight anchor battles are added on top. The experiment used that equality throughout, 28 candidate battles plus 8 anchor battles per candidate, so the only differences were seed allocation and, in the first pass, who met whom.

The tuning runs used the toj ruleset, eight candidates, five generations, the production theta as the search center and the only anchor, and the shipped search budget. The first generation is sampled from the run seed alone, so two arms launched with the same run seed start from exactly the same candidates; the experiments confirm this with identical first-generation anchor scores.

## Method

The experiment proceeded in three stages, each adding one control.

| Stage | What differed between the arms | Runs | Reading |
|---|---|---|---|
| First pass | Seed allocation and pairing structure | Two run seeds, five generations each | 500-battle duels between the final means |
| Endpoint, isolated | Seed allocation only, with balanced pairings in both arms | Two run seeds, five generations each | 500-battle duels between the final means |
| Decision | Seed allocation only, with identical candidates and identical pairings | One generation | Each candidate's mean score over its 28 candidate battles |

The duels always played 500 battles. In the first pass the first theta kept the first seat, so each reading was corrected with a seat self-match that measured the first-seat edge; in the isolated stage the seats alternated battle by battle, so no correction was needed. Every replicate used its own compare seed.

## Results

### The decision under identical everything

One generation, run seed 2, the same eight candidates in both arms, the same balanced pairings, the same anchors, the same 28 rounds. The only difference was where each round's seed pair came from: one shared pair for all four pairing battles, or a fresh pair for each battle.

| Candidate | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
|---|---|---|---|---|---|---|---|---|
| Shared seeds | 0.571 | 0.429 | 0.536 | 0.429 | 0.536 | 0.500 | 0.571 | 0.429 |
| Independent seeds | 0.393 | 0.500 | 0.571 | 0.571 | 0.643 | 0.321 | 0.393 | 0.607 |

The deals reshuffled the field. The mean absolute score change was 0.134 and the largest was 0.179, on the order of the sampling noise of two independent 28-battle estimates (a standard error of about 0.09 each). The rank correlation between the two readings was -0.37, which is inside the sampling error of a rank correlation over eight candidates (about 0.38). Candidate 0, tied for the top under shared seeds, fell to a tie for last under independent seeds, and candidate 4, the independent winner, placed third under shared seeds. The per-candidate anchor scores were identical in both arms, confirming the candidates were identical and the anchor battles used common deals.

A single generation cannot say which reading was closer to the truth. It does say that two estimates of the same eight candidates, built from the same pairings, agree no better than two independent draws would, so at 28 battles the seed allocation can outweigh the skill differences it is supposed to measure.

### The endpoint with pairings held equal

Two tuning runs per arm, five generations each, all four with balanced pairings, seats alternating by round, the same anchors, and the same battle counts. The independent arm drew a fresh seed pair for each pairing; the shared arm drew one per round. Each pair of runs started from the same run seed, so within a replicate the two arms began from identical candidates.

Duel scores are from the independent-seed mean's perspective, and a positive z means it was ahead. The standard error was 0.022 on each reading.

| Reading | Independent-seed mean | Shared-seed mean | z |
|---|---|---|---|
| Run seed 1, independent queues | 0.524 | 0.476 | +1.07 |
| Run seed 2, independent queues | 0.476 | 0.524 | -1.07 |
| Run seed 1, matched queues | 0.490 | 0.510 | -0.45 |

The two replicates cancel: one put the independent-seed mean 0.024 ahead, the other put the shared-seed mean 0.024 ahead. Pooling the compare batches alone gives 0.000 with a standard error of 0.016; this treats the two tuning runs as fixed, so it understates run-to-run tuning variation. The seat self-matches were 0.482, 0.496, and 0.480, near parity as expected with alternating seats. The matched-queue reading, which removes any remaining difference in the deals each side draws, also found nothing: 0.490 against 0.510, z = -0.45.

At this precision the experiment bounds any shared-seed advantage at the endpoint to a point or two and shows no reason to expect one.

### The first pass, confounded

The first pass compared the shipped round robin against the balanced shared schedule, which differ in two ways: the round robin gives every pair four meetings and draws seats and seeds per battle, while the balanced schedule gives one meeting per candidate per round, pairs candidates differently, alternates seats by round, and shares the round's deal. Its two replicates disagreed.

z is from the independent-seed mean's perspective, and a positive z means it was ahead. The seat-adjusted column subtracts the measured first-seat edge:

| Run seed | Queues | Shared-seed mean | Independent-seed mean | z | Seat-adjusted z |
|---|---|---|---|---|---|
| 1 | independent | 0.578 | 0.422 | -3.53 | -2.35 |
| 1 | matched | 0.598 | 0.402 | -4.46 | -3.28 |
| 2 | independent | 0.478 | 0.522 | +0.98 | +2.68 |
| 2 | matched | 0.548 | 0.452 | -2.15 | -0.45 |

Run seed 1 made the shared schedule look much better; run seed 2 did not reproduce it, and its independent-queue reading even leaned the other way once the measured first-seat edge was removed. Because seed sharing and pairing structure moved together, no reading here can be assigned to seeds. Isolating the seed variable removed the effect.

## Why the deal moves the ranking

The ranking rests on 28 candidate battles. Around parity, the standard error of a mean over 28 battles is about 0.09. Within one generation the candidates are close relatives of the same center, so their true differences are small compared with that. Any shift in the deals of the size measured here therefore reorders them.

Sharing a seed pair per round does not add battles. It fixes which pieces and which garbage the candidates see in a round, but each candidate still plays its own 28 battles against its own sequence of opponents, and each outcome still depends on the interaction between the two boards. The shared deal removes one source of difference between the candidates; it does not make a 28-battle mean precise. The endpoint test is the arbiter, and it found the two schemes equally strong: changing which luck is common does not raise what the search reaches.

The lever that does shrink the noise is the battle count. At 28 battles the standard error is 0.09; at 112 it is 0.05, and candidate differences would begin to separate. The [anchored tuning experiment](<Anchored versus unanchored tuning>) reached the same conclusion for a different knob: changing the anchor redirects the path but does not raise the endpoint, and sharpening a verdict is a matter of more battles.

## A reporting bug found and fixed

While chasing why unrelated generations printed the same anchor score, the experiment found that the reported fitness divided by too large a count. The anchor battles already appear in the battle schedule, and the appearances tally added them a second time, so a candidate with 28 candidate battles and 8 anchor battles was divided by 44 instead of 36. Every candidate in the experiment was scaled by the same 36/44, and the search only consumes the order of the scores, so the ranking, the selection, and every comparison on this page were unaffected. Only the absolute printed level was about 18 percent below the true mean score over its 36 battles, so it did not read as a battle score. The tally was corrected after the experiment: each candidate is now divided by the battles it actually played, so the printed level is the mean it claims to be. A field with an equal number of battles per candidate, like this one, keeps its ranking; an odd field, where byes make the counts differ, could reorder.

## Conclusion

The queue and garbage seeds drive the tuner's decision. With candidates and pairings identical, changing only where each seed pair came from reshuffled the generation completely: mean absolute score shift 0.134, rank correlation -0.37, and a different winner. But shared seeds are not a strength lever. With the seed variable isolated, the two designs produced equally strong thetas, 0.024 one way in one replicate and 0.024 the other way in the second, pooled 0.000 with a standard error of 0.016, and matched queues agreed (0.490 against 0.510). What advantage the first pass showed came from the balanced pairing structure, not from sharing the deal.

So the tournament keeps independent seed pairs per battle, and the balanced schedule keeps its shared per-round deals for what they are, a fairer comparison inside a round, not a way to reach a stronger theta.

## What changed

Nothing in the tournament's design. No schedule was adopted or removed, and no tuning setting followed from the result. The shipped defaults already give each battle its own seed pair, and nothing here argues for changing that. The balanced schedule remains available for its fairer within-round comparisons, not as a strength lever. The reporting bug described above was fixed afterwards: the printed level now equals the mean battle score, and the ranking in an even field like this one is unchanged.
