---
title: Matchmaking
---

# Matchmaking

Each search generation proposes candidates, but it has no quality score for them yet. The tournament creates those scores by having candidates play one another. This page explains who plays, how often, and how each battle affects the standings. [CMA-ES](<CMA-ES>) uses those standings, and the battles themselves run through the [Match API](<Match API>).

## Glossary

| Term | Meaning |
|------|---------|
| Participant | One candidate theta, plus the anchor, all playing the same tournament. |
| Elo | A rating that each battle's score pushes up or down. Higher means better within this tournament. |
| Match quota | The number of battles every participant is guaranteed, 64 by default. |
| Anchor | The shipped theta, playing unchanged, that pins what the ratings mean. |

## The field

Each tournament includes the generation's candidates and one fixed anchor, the shipped default theta. The anchor matters because Elo measures relative strength. A candidate's rating means little on its own, but its gap from the anchor shows whether the generation is stronger or weaker than the shipped AI. Ratings start equal at the beginning of each generation and reset at the end. The tournament ranks one generation at a time rather than tracking ratings across generations.

## Who plays whom

Every participant is guaranteed its quota of battles. Selection starts with the eligible participant that has played the fewest, which spreads matches evenly. The tournament ends only after everyone has played exactly their quota.

Opponents are chosen from an Elo neighborhood around the first player's rank. The window is based on the field size and extends about one sixteenth of the field in either direction. Similar-rated opponents make more informative matches; a battle between the strongest and weakest candidates is nearly decided in advance. If no eligible opponent is in the window, the tournament chooses from all eligible participants. Seats are assigned randomly, and seed pairs come from the generation's training stream.

## How a battle moves the ratings

Elo uses a 400-point logistic curve to calculate each player's expected score from the two ratings. It compares that expectation with the battle's actual score, then adjusts both ratings toward the result. The step size starts at 24 by default and drops to 4 as a participant approaches its quota. Early matches shape the standings quickly; later ones refine them.

Battles run on a thread pool. Selection counts matches already in progress, so every participant still finishes with the same quota. Candidate ratings become the generation's fitness, and the anchor's rating is reported for context.
