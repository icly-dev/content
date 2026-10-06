---
title: Fake next and branching
---

# Fake next and branching

## Glossary

Shared engine vocabulary (board, depth, horizon, hold, status, search budget) is defined in the [Glossary](Glossary). This page adds what is specific to unknown futures:

| Term | Meaning |
|------|---------|
| Known preview | The piece queue the runner guarantees for the next decisions. |
| Fake pieces | Randomly sampled pieces standing in for queue positions beyond the known preview. |
| Fake horizon | How many placements deep the search goes on fake pieces. |
| Branch | One independently sampled continuation of the search past the known preview. |
| Branch count | How many futures are sampled per decision. |
| Vote | The first move a branch's own search ends on. |

## Why fake pieces exist

The known preview rarely extends as far as the search needs to look. Beyond it, the engine has to assume which pieces come next, so it extends the horizon with fake pieces drawn from the game's randomizer.

Relying on one guess could bias the search. A future that happens to suit the position may make its first move look better than it is. Sampling several independent futures and letting them vote spreads that luck across guesses, much like playing more games helps control noise in match evaluation.

## How the fake pieces are sampled

Each branch draws its tail without replacement from a shuffled bag. When the bag runs out, it shuffles a new one, following the same rule as the game's real queue. Branches draw independently, so they rarely produce the same full tail. Sampling advances the engine's random state, keeping decisions reproducible for a given seed.

The runner sets the fake horizon length and branch count, and passes both to the search. If the fake horizon is nonzero but the branch count is one, the search still uses one sampled future. This is the cheapest way to look beyond the preview.

## How branches share work

Branches work independently only where needed:

- The search computes one beam limit schedule for the combined horizon, including known placements and the fake tail. Every branch uses the same caps. See [Beam search](<Beam Search>) for details.
- The search runs through all but the last placement covered by known pieces, then snapshots the frontier for each branch. If hold is enabled, a held piece is available, and either more than one queue piece is known or holding is currently allowed, this shared section extends one placement farther because the swap also places a known piece.
- Each branch places the last known piece using its own sampled tail. That can change the preferred paths even before the pieces themselves differ in the next layer. Branches run one at a time in a fixed order.
- If no pieces are known, there is no shared prefix, so each branch searches from the root.
- Each branch updates the learned branching estimate for every depth it expands. The fake region is learned like the rest of the search.

## Voting

Each branch votes for one first move: the best candidate found by that branch's search, traced back to its first placement. The engine counts votes by first move, identified by landing position, piece, orientation, and whether it uses a hold swap. It also records the best status among the branches that chose each move.

The move with the most votes wins. If votes are tied, the better status wins, followed by the move generated earlier. A branch can also finish with a hold decision and no placement. The engine remembers those requests and falls back to hold if no branch produces a placement.

## Interaction with the caches

The transposition table keeps a separate set of tables for the fake region, cleared on every move, since the sampled tail changes each decision. See [Transposition table](<Transposition Table>).
