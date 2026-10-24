---
title: CMA-ES
---

# CMA-ES

Battles and the tournament show which candidates are better, but the search still has to choose what to try next. That is the job of the black-box search CMA-ES, the covariance matrix adaptation evolution strategy. The name sounds more complicated than the idea: it keeps a cloud over the parameter space, samples candidates from that cloud, then shifts and reshapes it toward the better-scoring candidates after each round. It needs scores, not gradients, which is exactly what the tournament provides. See [Matchmaking](<Matchmaking>) for details about the tournament.

## Glossary

| Term | Meaning |
|------|---------|
| Search space | The theta, measured as offsets from a chosen center, with every dimension scaled to a comparable range. |
| Sigma | How far the cloud spreads when sampling. |
| Mean | The center of the cloud after the latest round, the current best guess of a good theta. |
| Candidate | One theta the search samples for a generation. |
| Generation | One round: sample the candidates, battle them, update the cloud. |

## The space

The search starts from a center theta: the shipped defaults unless a different starting point is chosen, such as all zeros or a theta read from a file. The cloud's center is represented as offsets from that center, with each offset scaled to its dimension's size and a minimum scale of one tenth. This keeps features with large values from dominating features with small ones just because of their units. The spread starts at sigma 0.3, and the default population is 16 candidates per generation. That is close to the size the method recommends for this many parameters, so a generation stays affordable while still giving the ranking enough candidates to work with.

## One round

Each round samples candidates from the cloud, converts the points into thetas, and sends them to the tournament. The tournament's scores guide the update: the cloud moves toward the stronger candidates and changes shape to learn which directions in the theta matter and how those directions interact. The anchor battles in that score keep the target stable from one generation to the next. Sampling, battling, and updating together make up one generation.

## The deliverable

The search does not select a champion or pick the best candidate. Instead it writes the cloud's center to a file after every round, decoded as a theta, and appends one line to the history. A candidate's tournament score alone is not enough to justify adopting it. Test it first using [Theta comparison](<Theta comparison>).

## Stopping and resuming

The round count sets the search budget, and the search can also stop when its own convergence criteria are met. After every battle it saves its state, so an interrupted run resumes at the next battle it has not played instead of replaying the generation. The state records the cloud, the random stream that built the schedule, the sampled candidates, the battle schedule, and the scores already collected. It also records the configuration and the center the run started from, and it refuses to resume when those differ. Load only state files you created yourself.
