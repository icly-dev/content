---
title: Fake next and branching
---

# Fake next and branching

## Glossary

Shared engine vocabulary (board, depth, horizon, hold, status, search budget) is defined in the [Glossary](Glossary). This page adds what is specific to unknown futures:

| Term | Meaning |
|------|---------|
| Known preview | The piece queue the host guarantees for the next decisions. |
| Fake pieces | Randomly sampled pieces standing in for queue positions beyond the known preview. |
| Fake horizon | How many placements deep the search goes on fake pieces. |
| Branch | One independently sampled continuation of the search past the known preview. |
| Branch count | How many futures are sampled per decision. |
| Vote | A branch's endorsement of one first move for the current piece. |

## Why fake pieces exist

The known preview rarely reaches as deep as the search should look. Every placement beyond the preview needs a piece assumption, so the engine invents one: it extends the horizon with fake pieces drawn from the same randomizer the game uses.

Trusting a single guess would bias the search: a continuation that happens to suit the position makes its first move look better than it is. Sampling several independent futures and letting them vote spreads that luck across guesses instead of concentrating it, the same way playing more games controls noise in match evaluation.

## How the fake pieces are sampled

Each branch draws its tail without replacement from a shuffled bag, refilling a fresh shuffled bag when one runs out, so a tail respects the same bag invariant the game's real queue does. Different branches draw independently, so two branches rarely agree on the whole tail. Sampling consumes the engine's random state, which keeps every decision reproducible for a given seed.

The fake horizon length and the branch count are host-side knobs: the profiler exposes them as flags, and the search API takes both as parameters. With a fake horizon but a branch count of one, a single sampled future is still used, which is the cheapest configuration that still looks past the preview.

## How branches share work

Branches are independent only where they must be:

- The beam limit schedule is computed once for the combined horizon, known placements plus the fake tail, and every branch runs under the same caps. See [Beam search](<Beam Search>) for the schedule itself.
- The search runs once through the prefix covered by known pieces, one placement deeper when hold is enabled and a held piece is usable, since the swap is also a placement of a known piece. The frontier at that point is snapshotted, and every branch continues from it.
- The branches place the last known piece with different expectations about the sampled tail, which can already change the paths the search prefers; from the following layer on, the placed pieces themselves differ.
- When nothing at all is known, no prefix is shared and every branch runs its own full search from the root.
- Every branch's expansions update the learned per-depth branching estimates that the schedule uses, so the fake region is learned like the rest.

## Voting

Each branch ends with one winning first move: the best candidate its own search found, traced back to its first placement. Votes are tallied per first move, where a first move is identified by its landing position and whether it was a hold swap, and each tally also remembers the best status any branch achieved with it.

The most-voted first move wins. Ties go to the move with the better status, then to the earlier-generated one. If no branch produces a move, the engine falls back to requesting a hold when the branches asked for one.

## Interaction with the caches

The transposition table keeps a separate set of tables for the fake region, cleared on every move, since the sampled tail changes each decision. See [Transposition table](<Transposition Table>).
