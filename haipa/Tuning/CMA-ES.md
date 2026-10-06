---
title: CMA-ES
---

# CMA-ES

The battles and the tournament say which candidates are better; something still has to decide which candidates to try next. That is the black box search, CMA-ES, the covariance matrix adaptation evolution strategy. The name is heavier than the idea: the search keeps one cloud over the parameter space, samples each generation's candidates from the cloud, and after every round pulls and reshapes the cloud toward the candidates that ranked best. It needs only rankings, never gradients, which is exactly what a tournament produces. The tournament itself is described in [Matchmaking](<Matchmaking>).

## Glossary

| Term | Meaning |
|------|---------|
| Search space | The theta, centered on the shipped defaults, with every dimension scaled to a comparable range. |
| Sigma | How far the cloud spreads when sampling. |
| Mean | The center of the cloud after the latest round, the current best guess of a good theta. |

## The space

The search starts at the shipped default theta and never loses sight of it: the cloud's center is expressed as offsets from the defaults, and each offset is scaled by its dimension's size, at least a tenth, so a feature with large numbers does not dominate a feature with small numbers purely through units. The spread starts at sigma 0.3, with a population of 30 candidates per generation by default.

## One round

Each round samples the population from the cloud, decodes the points into thetas, and hands them to the Elo tournament with the anchor. The returned ratings steer the cloud: it moves toward the better candidates and reshapes itself, learning which directions of the theta matter and how strongly they interact. Sampling, battles and the update together are one generation.

## The deliverable

There is no champion gate and no best-candidate pick: the deliverable is the cloud's center after each round, decoded back into a theta and written to a file every generation, with one history line appended per generation. Whether a candidate is worth adopting is settled by measurement first, see [Theta comparison](<Theta comparison>), not by its tournament rating alone.

## Stopping and resuming

The round count is a budget, and the search can also stop itself when its criteria say the cloud has converged. Every generation ends with the search's state written to disk, so a run can be interrupted and later resumed where it left off. The state records its configuration and refuses to resume if the tournament settings no longer match. The state file holds the distribution in pickle form, so load only state files you produced yourself.

## Running it

`python3 -m haipa.cmaes [threads] [generations] [state-file]`, printing one line per generation with the population's and the anchor's Elo, the spread of the cloud, and the elapsed time.
