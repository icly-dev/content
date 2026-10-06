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

The known preview rarely reaches as deep as the search should look. Every placement beyond the preview needs a piece assumption, so the engine invents one: it extends the horizon with fake pieces drawn from the same randomizer the game uses.

Trusting a single guess would bias the search: a continuation that happens to suit the position makes its first move look better than it is. Sampling several independent futures and letting them vote spreads that luck across guesses instead of concentrating it, the same way playing more games controls noise in match evaluation.

## How the fake pieces are sampled

Each branch draws its tail without replacement from a shuffled bag, refilling a fresh shuffled bag when one runs out, so a tail follows the same bag rule the game's real queue does. Different branches draw independently, so two branches rarely agree on the whole tail. Sampling uses up the engine's random state, which keeps every decision reproducible for a given seed.

The fake horizon length and the branch count are runner-side knobs: the search API takes both as parameters. With a fake horizon but a branch count of one, a single sampled future is still used, which is the cheapest configuration that still looks past the preview.

## How branches share work

Branches are independent only where they must be:

- The beam limit schedule is computed once for the combined horizon, known placements plus the fake tail, and every branch runs under the same caps. See [Beam search](<Beam Search>) for the schedule itself.
- The search runs once through all but the last of the placements covered by known pieces; with hold enabled and a held piece usable, that covered region is one placement deeper, since the swap is also a placement of a known piece. The frontier at that point is snapshotted, and every branch continues from it.
- Each branch then places the last known piece under its own expectations about the sampled tail, which can already change the paths the search prefers; from the following layer on, the placed pieces themselves differ. Branches run one at a time, in a fixed order.
- When nothing at all is known, no prefix is shared and every branch runs its own full search from the root.
- Every branch's expansions update the learned per-depth branching estimates that the schedule uses, so the fake region is learned like the rest.

## Voting

Each branch ends with one winning first move: the best candidate its own search found, traced back to its first placement. Votes are counted per first move, where a first move is identified by its landing position, piece and orientation included, and whether it was a hold swap; the spin grade is not part of the identity, so branches reporting the same landing with different grades count as one vote. Each count also remembers the best status any branch achieved with it.

The most-voted first move wins. Ties go to the move with the better status, then to the earlier-generated one. A branch can also end without a placement and with a hold decision instead; the engine remembers such requests, and if no branch produces a placement at all, it falls back to the hold.

## Interaction with the caches

The transposition table keeps a separate set of tables for the fake region, cleared on every move, since the sampled tail changes each decision. See [Transposition table](<Transposition Table>).
