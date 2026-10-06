---
title: Matchmaking
---

# Matchmaking

A search generation proposes candidates and needs one quality number per candidate. Such numbers do not exist yet; the tournament manufactures them by making the candidates play each other. This page is that tournament: who plays whom, how often, and how a battle moves the standings. The search that reads the standings is [CMA-ES](<CMA-ES>); the battles themselves are the [Match API](<Match API>).

## Glossary

| Term | Meaning |
|------|---------|
| Participant | One candidate theta, plus the anchor, all playing the same tournament. |
| Elo | A rating that each battle's score pushes up or down. Higher means better within this tournament. |
| Match quota | The number of battles every participant is guaranteed, 64 by default. |
| Anchor | The shipped theta, playing unchanged, that pins what the ratings mean. |

## The field

Participants are the generation's candidates plus one fixed anchor, the shipped default theta. The anchor matters because Elo only measures relative strength: a candidate's rating says nothing by itself, but the gap to the anchor says whether this generation lives above or below the shipped AI. Ratings start equal at the beginning of every generation and are forgotten at its end; the tournament is a ranking device for one generation, not a ledger across them.

## Who plays whom

Every participant is owed its quota of battles. Selection always starts from the least-played eligible participant, so the quota spreads evenly, and the tournament ends only when every participant has played exactly its quota. The opponent comes from the Elo neighborhood: participants ranked within a window around the first player's rank, sized from the field, about one sixteenth of it in each direction. Comparable opponents make battles informative; a match between the strongest and the weakest candidate is nearly a foregone conclusion. If the window offers nobody eligible, the opponent comes from anywhere eligible. Seats are drawn at random, and the seed pair comes from the generation's training stream.

## How a battle moves the ratings

Elo compares each player's expected score, from the 400-point logistic on the two ratings, with the battle's actual score, and moves both ratings toward what happened. The step size starts large, 24 by default, and decays to 4 as a participant approaches its quota: early battles set the shape of the standings quickly, later battles refine them. Battles run on a thread pool; selection counts battles already in flight, so the equal-quota guarantee survives threading.

The candidate ratings leave the tournament as the generation's fitness, and the anchor's rating is reported alongside for context.
