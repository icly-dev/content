---
title: Welcome
---

# Welcome

This site is built from markdown files in a git repo, rendered by a small SvelteKit app. Pages live under `content/` and are mirrored to URLs, the directory tree on the left is generated from the repo layout, and every page accepts comments: sign in with GitHub or Discord to leave one.

The haipa documentation currently covers the engine, the movement layers beneath it, the AI it calls into, and the tuning loop around it:

- [Overview](haipa/Engine/Overview) for the input-to-decision flow,
- [Glossary](haipa/Engine/Glossary) for shared vocabulary,
- [Beam search](<haipa/Engine/Beam Search>),
- [Transposition table](<haipa/Engine/Transposition Table>),
- [Fake next and branching](<haipa/Engine/Fake next and branching>),
- [AI interface](<haipa/AI/AI interface>) for the calls the engine makes into the AI: where a piece enters, the board-only evaluation, and the context step,
- [Rule layer](<haipa/Rule/Rule layer>) for the ruleset side of the AI: the board, the mino types, the rotation system, and the two questions the engine asks,
- [Movegen](<haipa/Search/Movegen>) for how the engine finds every move a piece can make,
- [Pathgen](<haipa/Search/Pathgen>) for how the chosen move becomes the inputs the runner executes,
- [Tuning Overview](<haipa/Tuning/Overview>) for the loop that improves the AI's judgment,
- [Match API](<haipa/Tuning/Match API>) for the battle two thetas play and how a result becomes a score,
- [Matchmaking](<haipa/Tuning/Matchmaking>) for the Elo tournament that ranks a generation's candidates,
- [CMA-ES](<haipa/Tuning/CMA-ES>) for the black box search over the theta,
- [Theta comparison](<haipa/Tuning/Theta comparison>) for settling whether one theta really beats another.

## Feedback wanted

These docs and the engine behind them improve through reader feedback, and both kinds are welcome on any page:

- **Readability**: if a page is hard to follow, a definition arrives too late, a term is undefined, a diagram or example is missing, or the wording is just awkward, say so. Comments on readability directly shape how the docs are written and structured.
- **Methodology**: if you doubt an approach itself, whether the beam schedule, the sampling of unknown futures, the evaluation split, or the Elo tournament, question it. Methodology feedback feeds back into the implementation, not just the words.

You do not need to be polite or complete: a one-line "I got lost here" on the exact sentence is already useful. Wherever possible, point at the page and the part that tripped you, so improvements land where they matter.
