# AGENTS.md

Rules for writing documentation in this repo. Pages here are rendered by the SvelteKit site in `../site`; readers of the rendered site have access to the docs only, never to the source code.

## Placement and metadata

- Feature documentation for a project lives in a directory named after the project (e.g. `haipa/Engine/` for the haipa engine). A page `X.md` or `X/index.md` renders at `/docs/<path>`.
- Every page starts with front matter containing at least `title:`.
- Commit finished work here. Do not push; that is the user's job. The `content` symlink in consuming repos is gitignored there and never committed here.

## Audience rule: no code references

Readers cannot see the code, so pages must stand alone without it.

- Never reference identifiers the reader cannot look up: class, namespace, function, or member names, source file paths, or links into the source tree.
- Describe behavior, structure, and rationale in prose. Name mechanisms ("a per-bucket flag naming the slot to evict next"), not symbols (`replacement_`).
- Exception: user-facing knobs are fine because the reader can act on them. This includes compile-time build options (e.g. `TETRIS_TT_BITS`) and tool output the reader can observe (e.g. profiler report lines).
- Developer-only diagnostics behind compile-time defines do not get their own section. Fold the actionable interpretation (what a number means, what to do about it) into the section it supports, and mention the enabling condition honestly.

## Start with a glossary

- A feature page that relies on domain vocabulary opens with a short glossary table defining each term before first use (board, roof, horizon, hit/miss, and so on).
- Keep definitions ruleset-agnostic first; add concrete values as explicitly framed examples of the shipped ruleset, e.g. "In the ruleset this engine ships with, the grid is 10 cells wide." Never assert ruleset specifics as universal properties.
- If several pages share vocabulary, prefer a shared glossary page that others link to over repeating tables.

## Explain rationale, not just mechanics

- For every design decision, state why it exists, and lead with the original intent when known. Example: per-depth tables exist first because they make rotation and clearing cheap constant work per table, and only second because cross-depth reuse would be semantically wrong.
- State the safety or cost model where it matters: what eviction or a cache miss costs (a recomputation, never a wrong answer), and why a simple policy is sufficient (working set size, reuse pattern, lifetime between resets).
- Explain what is deliberately not covered by a cache or mechanism, with the reason (it depends on history or context, so caching it would give wrong answers). Concretes like back-to-back or combo counters are good examples, framed as ruleset examples.

## Aids: pseudocode and diagrams

- Use Python-style pseudocode for algorithms and plain `text` diagrams for layouts and flows. The site renderer does not support mermaid; it only does syntax highlighting (shiki), so mermaid blocks render as raw code.
- Link destinations that contain spaces must be wrapped in angle brackets: `[text](<Some Page>)`. A bare `](Some Page)` is not a link in CommonMark and renders as literal text.
- Mark invented placeholder names in pseudocode as placeholders (`apply_context` here is just a placeholder ... it is not a function the reader could call), and point to the section that explains the real behavior.
- Keep pseudocode aligned with the prose simplifications: if the text omits an edge case, the pseudocode should omit it too and the omission should be stated.

## Style

- No em dashes anywhere in page content; use commas, colons, or parentheses.
- Prefer short paragraphs, bullets, and tables over long prose.
- Write for a reader who has never seen the implementation and cannot ask the code questions.
