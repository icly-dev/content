---
title: Anchored versus unanchored tuning
---

# Anchored versus unanchored tuning

**Status: closed** ✅ (question settled)

A tournament scores each candidate against the rest of its generation and against fixed anchors. The candidate battles are zero-sum: one player's win is the other's loss, so the field's average score sits at one half whatever the thetas do. The anchor battles are the only battles that add to the total, which makes the anchor the only term that can rank one generation against another. That role also suggests the anchor steers the search: candidates that handle the anchor well rise in the ranking, so the next generation should bend toward the anchor's style and stay near it.

The experiment ran the tuner twice with identical settings except the anchor, one run with the production theta as its only anchor and one with no anchors at all, then compared where each landed and how strong its mean plays. It answers the question the anchor's stated purpose raises: does anchoring improve the result, or only change it?

Shared tournament vocabulary (candidate, appearance, anchor, fitness, generation) is defined in [Matchmaking](<../Tuning/Matchmaking>); seat, seed pair, standard error, and z in [Theta comparison](<../Tuning/Theta comparison>); the search's scale, sigma, and mean in [CMA-ES](<../Tuning/CMA-ES>). This page adds:

| Term | Meaning |
|------|---------|
| Scale unit | One unit of the search's scaled space: each parameter divided by the larger of its production value's magnitude and one tenth, so a difference means the same thing in every parameter. |
| Divergence | The distance between two runs' means, in scale units. |
| Anchor score | The generation's mean battle score against the anchors, reported separately from fitness. |

## The hypothesis

The anchor is the shipped production theta, a known working player rather than a member of the current field. If anchor battles carry real weight in the ranking, two things should follow.

- The anchored run should stay nearer the production theta. Candidates that drift away from that style should do worse against it and lose their place.
- The anchored mean should end up stronger. An unanchored run has only the field as a reference, and a field of related candidates rewards whatever beats the field, which is not necessarily what beats other players.

The opposite outcome is possible too: the anchor battles are a small share of fitness, so they might change which direction the search wanders without changing how strong the endpoint is.

## Method

The two runs differed in one setting. One had the production theta as its only anchor, with 15 anchor battles per candidate; the other had no anchors. Everything else matched: the production theta as the search center, the toj ruleset, one run seed, 16 candidates per generation, 128 candidate battles each, and the same fixed search budget per move. Both runs were served to remote helpers, so their different wall-clock times stand for nothing.

Both were planned for 100 generations and stopped early on available time. The anchored run reached generation 70; the unanchored run reached generation 50. The controlled comparison is generation 50 of each, and the anchored run also gives a longer-horizon reading at generation 70.

The two generation-50 means were settled by a direct duel of 512 battles with the same search budget, on one game seed, together with a seat self-match in which the same theta plays itself to measure the first-seat edge. Scores in the results are from the anchored mean's perspective, 0.5 being parity.

## Results

**Where the runs went.** Both means left the production theta by about the same distance, and in nearly unrelated directions.

| Reading | Anchored, gen 50 | Unanchored, gen 50 | Anchored, gen 70 |
|---|---|---|---|
| Distance from the production theta, scale units | 6.40 | 6.75 | 6.66 |
| Anchor score | 0.733 | no anchors | 0.713 |

The two generation-50 means sat 8.46 scale units apart (rms 1.31, largest single gap 4.53, on the T-spin triple slot weight). They landed on opposite sides of the production value on 11 of 42 parameters. The angle between their displacements from the production theta had a cosine of 0.17, close to perpendicular. The largest gaps sat in the spin slot weights, back-to-back, the well and bump terms, and the safety thresholds; the smallest were the attack and line clear rewards, which both runs moved the same way.

So the anchor did redirect the search: two means that started from the same point and ran the same schedule ended in almost unrelated directions. It did not act as a leash, though. The anchored mean is no closer to the production theta than the unanchored one, 6.40 against 6.75, a gap far inside the search's step size of about 0.27.

**How strong they play.** The duel found no difference.

| Reading | Mean | Standard error | Breakdown |
|---|---|---|---|
| Anchored mean in the first seat | 0.4980 | 0.0221 | 257 losses, 0 ties, 255 wins |
| Anchored mean against itself, first seat | 0.5352 | 0.0221 | first-seat edge |
| Duel minus seat expectation | 0.4628 | 0.0221 | z = -1.68 |

Raw, the duel is a dead heat (z = -0.09). Removing the first-seat edge leaves the unanchored mean ahead by 0.037, under two standard errors, so the experiment bounds an anchored advantage at roughly four points in either direction rather than proving equality. Attack per piece was 0.965 for the anchored mean and 0.958 for the unanchored one, also inside the noise.

**The longer horizon.** More generations did not change the picture in the anchored run. Its anchor score was lower at generation 70 (0.713) than at generation 50 (0.733), while its step size fell from 0.271 to 0.246 and its cloud grew more elongated. The anchored run before this experiment, 236 generations, ended the same way: late mean updates were uncorrelated (successive step cosine -0.005, 51.7% of steps positive), and the mean's total displacement matched a random walk (13.54 against 11.47). Past a point the cloud wanders around a plateau.

## Why anchoring changed the path but not the strength

The anchor's share of fitness is small. At the defaults an anchor adds 15 battles to a candidate's 128, so the anchor term is about a tenth of the score. The rest is zero-sum, which pins each candidate's mean near one half and builds the ranking only from how candidates did against each other. A tenth is enough to reorder candidates that are close, but not enough to do much more.

The anchor also sees a weak opponent. Candidates score about 0.73 against the production theta, so beating it separates the field less than a stronger opponent would. The anchor is fixed on purpose, with the same seed pairs for every candidate and every generation, so that differences come from play. That fixity is what makes it a stable reference, and also why a longer run tends to fit those 15 deals rather than learn to play better.

Finally, this parameter field has many equally good directions at this budget. The two runs ended far apart in theta space and equal in battle, which is what a broad, flat ridge looks like. An anchor with a tenth of the weight can change which direction the search takes without changing the height it reaches.

## What to do with this

Keep the anchor for what it does well: it is a stable reference that makes scores comparable between generations, and its battles use common seed pairs, so candidate differences in them come from play. Do not expect it to raise the endpoint.

If the question is whether anchoring protects a long run from drifting away from real play, the tournament cannot answer it, because the anchor is inside the fitness and the run learns to satisfy it. That question needs a held-out reading, a fixed opponent set or a refreshing anchor scored apart from the ranking, so the run cannot adapt to it. The tuner does not currently produce one.

To resolve a difference of a point or two in strength, add battles, not generations. At 512 battles the standard error is 0.022; at 2,000 it is 0.011, and a two-point edge needs about 5,000 battles for a confident call.

## Conclusion

The anchor changes the path and not the result. Two 50-generation runs, one anchored on the production theta and one unanchored, ended 8.46 scale units apart in nearly perpendicular directions, played a dead heat (0.4980 with a standard error of 0.0221, seat-adjusted z of -1.68), and the anchored mean was no closer to the production theta than the unanchored one. The anchor term is too small and its opponent too weak to redirect a 42-parameter search that has many equally good directions at this budget.

More generations is not the lever either. The anchored run was slightly worse against its own anchor at generation 70 than at 50, and the long run before this one diffused in its late generations. Sharpening the verdict is a battle-count question.

## What changed

Nothing in the tournament. The experiment settled the anchor's role, and the anchor stays because a stable cross-generation reference is what it is for. One thing did change in the tooling: a run can now have no anchors at all, which is what made this comparison possible. Before, asking for zero anchor battles still kept the production theta as an anchor.
