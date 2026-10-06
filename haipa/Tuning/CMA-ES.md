---
title: CMA-ES
---

# CMA-ES

Battles and the tournament show which candidates are better, but the search still has to choose what to try next. That is the job of the black-box search CMA-ES, the covariance matrix adaptation evolution strategy. The name sounds more complicated than the idea: it keeps a cloud over the parameter space, samples candidates from that cloud, then shifts and reshapes it toward the best-ranked candidates after each round. It needs rankings, not gradients, which is exactly what the tournament provides. See [Matchmaking](<Matchmaking>) for details about the tournament.

## Glossary

| Term | Meaning |
|------|---------|
| Search space | The theta, centered on the shipped defaults, with every dimension scaled to a comparable range. |
| Sigma | How far the cloud spreads when sampling. |
| Mean | The center of the cloud after the latest round, the current best guess of a good theta. |

## The space

The search starts from the shipped default theta. The cloud's center is represented as offsets from those defaults, with each offset scaled to its dimension's size and a minimum scale of one tenth. This keeps features with large values from dominating features with small values just because of their units. The spread starts at sigma 0.3, and the default population is 30 candidates per generation.

## One round

Each round samples candidates from the cloud, converts the points into thetas, and sends them to the Elo tournament with the anchor. The ratings guide the next update: the cloud moves toward stronger candidates and changes shape to learn which directions in the theta matter and how those directions interact. Sampling, battling, and updating together make up one generation.

## The deliverable

The search does not select a champion or pick the best candidate. Instead, it writes the cloud's center to a file after every round, decoded as a theta, and appends one line to the history. A candidate's tournament rating alone is not enough to justify adopting it. Test it first using [Theta comparison](<Theta comparison>).

## Stopping and resuming

The round count sets the search budget, and the search can also stop when its convergence criteria are met. At the end of each generation, it saves its state so you can interrupt a run and resume it later. The state includes the configuration and will not resume if the tournament settings have changed. It is stored in pickle format, so load only state files you created yourself.
