# Graph RAG retrieval — full algorithm

Embedding-free retrieval over a folder of markdown notes. Validated in production (`connect-ai`'s `readGraphRagBrainContext`, TypeScript) against a real multi-hundred-note vault (`eery1677-wiki`). Reimplement the same shape in whatever language the target project uses — the logic is simple enough that a straight port is fine.

## Inputs

- `notes`: every `.md` file under the wiki root (skip raw-dump-only folders, caches, `.git`, anything clearly not a knowledge note). Cap how many files get scanned (e.g. 200) — that count-based cap is what actually keeps this fast as the vault grows into the hundreds of notes, since it bounds the number of parse operations regardless of any one file's size.

**Per-file size cap: don't copy an old "~80KB skip / ~12KB read" pair of numbers without checking the actual cost.** An earlier version of this reference recommended those two specific limits, reasoning "keep it fast." In production this broke silently: a legitimate single note (a supplier's full wholesale price list, pasted/converted into one note instead of being manually split) grew to 182KB, got skipped outright by the 80KB check, and vanished from retrieval with zero error anywhere — a customer-facing blog-writing feature could no longer see real, correct pricing it had access to. Measured the actual cost before trusting the old assumption: parsing that same ~154KB note with the regexes below took 2.46ms, and building the *entire* graph for a real 27-note vault took 5.2ms total. The "must stay fast" concern was real in principle but the specific numbers (80KB/12KB) were never actually benchmarked against real content — they were a guess, and the guess was wrong by an order of magnitude for anything resembling a reference table or price sheet. **Benchmark your own target content before picking a size cap** (regex parse time over a realistic large note, on the actual project's hardware) rather than reusing 80KB/12KB as received wisdom, and prefer one generous unified cap (e.g. 1MB) over two different small numbers — the count cap above is what protects overall latency; the per-file cap only needs to guard against a genuinely pathological file (something tens of MB, likely a mistake), not against a legitimately large but bounded business document.
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

Walk the ranked list, appending each note's title + a content excerpt to the output block until adding the next one would exceed `budget_chars`. Label each line:
- `🎯` if it was a seed (direct keyword match)
- `🔗` if it arrived via graph expansion, and name which seed pulled it in — e.g. "connected via `LM Studio Deployment`"

That labeling isn't optional polish: it's what lets the model (and a user reading the prompt) tell "this is directly relevant" apart from "this got pulled in because it's linked to something relevant" — useful signal for how much weight to give each item.

**What excerpt to show — seeds need more than the one-line insight.** A graph-neighbor (🔗) surfacing just the title + one-line insight is fine — it's context, not the main answer. But a seed (🎯, a direct match) is often the note the query is actually *about*, and a one-line insight can't answer a specific question ("what's the price of X") the way the note's own structured-knowledge body can. Give seeds a real excerpt of the body (a few hundred characters), not just the insight line.

**Don't take that excerpt as a blind prefix — center it on where the query's keywords actually appear.** `body[:400]` (or any fixed prefix length) silently fails whenever the specific thing being searched for isn't in the note's opening paragraph — which is often, since notes commonly open with general/introductory content (a company description, a category overview) before the specific details (a price table, a spec list) further down. Instead: find where the query's keywords occur in the body and take a window centered there, falling back to the plain prefix only if none of the keywords appear in the body at all.

**When multiple keywords match at different positions, prefer the *furthest* one, not the nearest.** This is the counter-intuitive part, confirmed against a real failure: a generic keyword (a brand name, a category term) tends to appear almost immediately in a note — frequently because it's literally in the title or the opening sentence — while the specific thing someone is actually looking for (a model number, an exact figure) tends to sit deeper in the structured content. Centering the window on the *earliest* keyword match keeps re-selecting that same generic opening paragraph and misses the specific detail every time; taking the position of whichever matched keyword occurs *latest* in the body reliably lands closer to the specific content instead.

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
