# Graph RAG retrieval — full algorithm

Embedding-free retrieval over a folder of markdown notes. Validated in production (`connect-ai`'s `readGraphRagBrainContext`, TypeScript) against a real multi-hundred-note vault (`eery1677-wiki`). Reimplement the same shape in whatever language the target project uses — the logic is simple enough that a straight port is fine.

## Inputs

- `notes`: every `.md` file under the wiki root (skip raw-dump-only folders, caches, `.git`, anything clearly not a knowledge note). Cap how many files get scanned (e.g. 200) and how large each one can be (e.g. skip files over ~80KB, read at most ~12KB of each) — this is a per-turn context-building step, not a batch job, so it needs to stay fast even as the vault grows into the hundreds of notes.
- `keywords`: terms describing what the current query/agent cares about — could be extracted from the user's message, from an agent's declared focus area, or both.
- `budget_chars`: how much of the final context window this retrieval is allowed to spend (see "Budget" below).

## Pass 1 — parse each note once

For each note file:
1. Read title (H1 or frontmatter title), compute a keyword-overlap **score** against `keywords` (simple term frequency is enough — this doesn't need to be sophisticated, it's a coarse filter, not the final ranking).
2. Extract every `[[wikilink]]` (strip an optional `|alias` suffix), lowercased, as the note's outgoing links.
3. Extract "anchor terms": the title itself, plus up to ~5 quoted or backtick-wrapped phrases (`` `like this` `` or `"like this"`) between 3 and 40 characters. This is a cheap proxy for "named entities this note is specifically about," without running an LLM extraction pass over every note.
4. Skip notes that score 0 *and* have no anchors worth indexing — no point carrying dead weight into pass 2.

Build a `title → node index` map as you go, so wikilinks can later resolve to actual nodes.

## Pass 2 — build the adjacency graph

Two edge types, both undirected:

- **Wikilink edges**: for every `[[link]]` in note A that resolves (via the title map) to note B, add an edge A↔B.
- **Anchor co-occurrence edges**: group notes by shared anchor term. If a given anchor term appears in **2 to 8** notes, connect all of them pairwise. Skip terms that appear in only 1 note (nothing to connect) or in more than 8 (too generic — it's noise, not a real conceptual link; a term shared across dozens of notes isn't telling you anything specific).

## Pass 3 — seed, expand, rank

1. Sort all scored notes descending, take the top **3** as seeds. If nothing scored above 0, retrieval returns empty — don't force irrelevant notes into context just to fill the budget.
2. Each seed keeps its full score.
3. For every neighbor of a seed (via the adjacency graph) that isn't itself a seed: assign it `seed_score * 0.5`. If a neighbor is reachable from multiple seeds, keep the *highest* resulting score and remember which seed it came from (for labeling).
4. Sort the combined seed+neighbor set by final score descending (tie-break by recency — more recently modified/reinforced notes first, so the vault self-prunes toward what's actually being used).

## Emit — budget-limited, labeled

Walk the ranked list, appending each note's title + one-line insight (not the full body — keep entries short) to the output block until adding the next one would exceed `budget_chars`. Label each line:
- `🎯` if it was a seed (direct keyword match)
- `🔗` if it arrived via graph expansion, and name which seed pulled it in — e.g. "connected via `LM Studio Deployment`"

That labeling isn't optional polish: it's what lets the model (and a user reading the prompt) tell "this is directly relevant" apart from "this got pulled in because it's linked to something relevant" — useful signal for how much weight to give each item.

## Budget guidance

Two tiers work well in practice:
- **Normal**: ~2000–2500 characters for this block.
- **Lean** (conversation already long / context getting tight): ~800–1000 characters. Don't drop the block entirely in lean mode — a shrunk version of the second brain is still worth more than none.

Cap every other context source (recent history, prior decisions, agent memory) similarly when running lean, rather than cutting brain retrieval specifically — the brain shouldn't be the first thing sacrificed when context is tight, since it's often the most differentiated part of what the assistant knows.

## Why this beats jumping straight to embeddings

- Zero external dependencies: no vector DB, no embedding API/model, no network call — it's regex + a dictionary walking the local filesystem.
- Fast enough to re-run on every single chat turn, even on modest local hardware, because it never touches the network and every note is capped in size.
- The graph structure (wikilinks + anchor co-occurrence) captures a meaningful chunk of what embeddings buy you — "these two things are related even without shared vocabulary" — without the infrastructure cost.
- It's legible: you can point at *why* a note got pulled in (seed match vs. graph hop), which is much harder to explain with a cosine-similarity score.

Only reach for embeddings if this approach demonstrably misses relevant notes that share no keywords, no links, and no anchor terms with the query — a genuinely different-vocabulary retrieval problem. That's a real failure mode eventually, just not the default one to design around.
