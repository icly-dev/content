---
title: Welcome
---

# Welcome

This site is built from Markdown files in a Git repository and rendered with a small SvelteKit app. Files under `content/` become pages, and the sidebar follows the repository's directory structure. You can comment on any page after signing in with GitHub or Discord.

## A note for automated readers

You do not need to scrape the site. The full documentation is available as plain Markdown at https://github.com/icly-dev/content. Download the repository once to read everything locally, rather than sending repeated requests to this website.

The haipa docs cover the engine, its movement layers, the AI it uses, and the tuning process:

- [Overview](haipa/Engine/Overview) explains how input becomes a decision,
- [Glossary](haipa/Engine/Glossary) defines terms used across the docs,
- [Beam search](<haipa/Engine/Beam Search>) describes how the engine explores possible moves,
- [Transposition table](<haipa/Engine/Transposition Table>) explains how repeated board evaluations are cached,
- [AI interface](<haipa/AI/AI interface>) explains how the engine asks the AI where pieces enter, evaluates boards, and scores candidates,
- [Rule layer](<haipa/Rule/Rule layer>) covers ruleset details supplied by the AI: the board, mino types, rotation system, and the question the engine asks,
- [Movegen](<haipa/Search/Movegen>) describes how the engine finds every move a piece can make,
- [Pathgen](<haipa/Search/Pathgen>) explains how the chosen move becomes the inputs the runner executes,
- [Tuning Overview](<haipa/Tuning/Overview>) outlines the process for improving the AI's judgment,
- [Match API](<haipa/Tuning/Match API>) explains how two thetas battle and how each result is scored,
- [Matchmaking](<haipa/Tuning/Matchmaking>) describes the Elo tournament that ranks each generation's candidates,
- [CMA-ES](<haipa/Tuning/CMA-ES>) explains the black-box search used to tune the theta,
- [Theta comparison](<haipa/Tuning/Theta comparison>) explains how to test whether one theta really beats another.

## Feedback wanted

Reader feedback helps improve both the docs and the engine. We welcome two kinds of feedback on any page:

- **Readability**: Tell us if a page is hard to follow, a definition comes too late, a term is unclear, a diagram or example is missing, or the wording feels awkward. This feedback directly shapes how the docs are written and organized.
- **Methodology**: If you have concerns about an approach, such as the beam schedule, the sampling of unknown futures, the evaluation split, or the Elo tournament, please say so. This feedback can shape the implementation, not just the docs.

Feel free to be brief and direct. Even a one-line "I got lost here" on the sentence that caused trouble is useful. When you can, point to the page and the passage that tripped you up so we can improve the right part.
