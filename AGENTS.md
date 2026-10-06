# AGENTS.md

Rules for writing documentation in this repo. Pages here are rendered by the SvelteKit site in `../site`; readers of the rendered site have access to the docs only, never to the source code.

## Placement and metadata

- Feature documentation for a project lives in a directory named after the project (e.g. `haipa/Engine/` for the haipa engine). A page `X.md` or `X/index.md` renders at `/docs/<path>`.
- Every page starts with front matter containing at least `title:`.
- Commit finished work here. Do not push; that is the user's job. The `content` symlink in consuming repos is gitignored there and never committed here.

## Audience rule: no code references

Readers cannot see the code, so pages must stand alone without it.

- Never reference identifiers the reader cannot look up: class, namespace, function, or member names, source file paths, or links into the source tree.
- Name mechanisms in Tetris terms, not symbols (`replacement_`) and not generic paraphrases. "On a hash hit the search reuses the stored evaluation" says exactly what happens; "a stored value is reused" does not.
- Use the specialized vocabulary Tetris players and the Glossary already use (placement, hold, preview, roof, garbage, back-to-back). A paraphrase like "the stashed piece" instead of "hold" reads as imprecision, not simplicity.
- Exception: things the reader can act on are fine, such as compile-time build options (e.g. `TETRIS_TT_BITS`) and outputs the tooling produces (e.g. the comparison report's lines).
- The reader cannot run code either: usage commands, module paths, and environment variables stay out. Describe what tooling does and produces, not how to invoke it.
- Developer-only diagnostics behind compile-time defines do not get their own section; fold the actionable interpretation (what a number means, what to do about it) into the section it supports.

## Start with a glossary

- A feature page that relies on domain vocabulary opens with a short glossary table defining each term before first use (board, roof, placement, depth, horizon, hold, hit/miss).
- Keep definitions ruleset-agnostic first; add concrete values only as explicitly framed examples of the shipped ruleset, e.g. "In the ruleset this engine ships with, the grid is 10 cells wide."
- Shared vocabulary lives in one glossary page per project; feature pages link to it and define only what is specific to them.
- After a term is defined, use the term itself everywhere; restating the definition in prose is overexplaining.

## Explain the architecture, not the implementation

- Describe the system the way the reader meets it: the stages of a decision, what each takes in and produces, and the guarantees between them. A reader should be able to predict the engine's behavior from the page.
- Leave out internal detail: data structures, field packing, bit tricks, per-entry flags. "One table per search depth, rotated between moves" is architecture; how a slot packs its hash and result is implementation.
- Do not explain optional items the engine works without. If an item can vanish with no error and no wrong answer, like the runner's combo table, it gets at most a mention.
- State what the evaluation's inputs permit judging, and stop there: which features and rewards a specific AI computes is its own.
- The search budget's knob name and its mapping stay undocumented; describe the budget by its effect.
- Pseudocode and diagrams stay at that level too: data flow, not storage layout.

## Explain rationale, not just mechanics

- For every design decision, state why it exists, and lead with the original intent when known. Example: one transposition table per search depth makes the between-moves rotation touch one table instead of the whole cache.
- State the safety or cost model where it matters: what eviction costs (one recomputation, never a wrong answer), and why a simple policy suffices (within a move the working set is small, entries are written once and read many times).
- Explain what a mechanism deliberately does not cover, with the reason. The transposition table caches what the board alone determines; line clears, depth, and combo or back-to-back state are applied per candidate, because two placements can end on the same board and differ in all of them.

## Aids: pseudocode and diagrams

- Use Python-style pseudocode for algorithms and plain `text` diagrams for layouts and flows. The site renderer does not support mermaid; it only does syntax highlighting (shiki), so mermaid blocks render as raw code.
- Link destinations that contain spaces must be wrapped in angle brackets: `[text](<Some Page>)`. A bare `](Some Page)` is not a link in CommonMark and renders as literal text.
- Mark invented placeholder names in pseudocode as placeholders (`apply_context` here is just a placeholder ... it is not a function the reader could call), and point to the section that explains the real behavior.
- Keep pseudocode aligned with the prose simplifications: if the text omits an edge case, the pseudocode should omit it too and the omission should be stated.

## Style

- No em dashes anywhere in page content; use commas, colons, or parentheses.
- Prefer short paragraphs, bullets, and tables over long prose.
- Do not overexplain: state each behavior and each reason once. Cut restatements, edge-case inventories, and caveats the reader cannot act on.
- Prefer plain words: "go down" not "descend", "extra" not "auxiliary", "stop" not "halt". Specialized game terms stay; formality goes.
- Keep a natural tone: write the way one person explains something to another, not like a formal spec or marketing copy.
- Write for a reader who has never seen the implementation and cannot ask the code questions.
