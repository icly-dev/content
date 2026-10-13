---
title: Outcome blending
---

# Outcome blending

**Status: refuted**

The label blended the search label with a win or loss tag from the game the board came from.

Shared vocabulary in [Glossary](<../../Engine/Glossary>); battle terms in [Theta comparison](<../../Tuning/Theta comparison>). This page adds:

| Term | Meaning |
|------|---------|
| Outcome tag | A per-game win or loss constant attached to every board the game visited. |

## The hypothesis

Game outcomes carry the one signal search labels lack: whether the line actually survived. Mixing a scaled outcome tag into the label should inject the survival gradient without hand-designing badness.

## What happened

Strength neutral, blocking survival worse. Per-game tags mark "positions I was losing in" as globally bad, but survival paths pass through losing-looking positions, so the tags taught the net to avoid the very states survival runs through.

## The verdict

**Refuted.** An honest outcome signal needs per-board winrate aggregation, which needs roughly a hundred visits per board; measured board overlap between games makes that cost far more games than the loop can afford. Search-derived labels remain the practical survival signal. The variant that survived the gauntlet is [Death labels](<Death labels>).
