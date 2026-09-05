---
name: second-brain-wiki-rag
description: How to design and build a persistent, self-improving "second brain" knowledge base for an AI assistant in any app — a markdown wiki that accumulates over time, retrieved cheaply without an embedding model via keyword+graph search, verified before it's trusted, auto-detects when to demand extra grounding instead of relying on a manual toggle, isolates knowledge across multiple tenants/channels/features sharing one vault, and synced to GitHub so it survives a dead laptop. Use this whenever the user wants an AI chatbot/agent to "remember" things across sessions beyond a single conversation, wants to add RAG or a knowledge base to a project, mentions "제2의 뇌" / "second brain" / "지식 저장소" / wiki-style memory, wants an Obsidian-like graph of notes, is extending an existing chatbot that currently only sees "the latest result" and needs it to accumulate knowledge over time instead, or the project has (or is about to have) multiple channels/tenants/audiences that must not see each other's notes. This is the user's own established architecture (validated against their real repos connect-ai and eery1677-wiki, and now also proven in a second production build) — treat it as the default pattern for this kind of feature, not one option among many.
---

# Second-Brain Wiki + Graph RAG

## Why this exists

A chatbot that re-reads "here's everything relevant right now" every turn only ever knows the most recent snapshot — anything from two weeks ago is gone unless someone happens to re-paste it. A second brain flips that: every worthwhile result becomes a permanent, searchable note, and the system gets more useful the longer it's used, without needing a vector database or an embedding model. That last part matters most for local-first, small-model setups — the retrieval method here is pure keyword scoring + link-graph traversal over the filesystem, so it stays fast and cheap even on a laptop running a 2B–12B local model.

This pattern was reverse-engineered from the user's own working systems (`connect-ai`, a VS Code extension, and `eery1677-wiki`, their real day-to-day knowledge vault) — it is proven, not theoretical. One real gap was found in that reference implementation during review: it never checks whether a newly-created note is actually faithful to its source before saving and pushing it. Section 5 below exists specifically to close that gap — don't skip it.

## The six pieces

Build these roughly in this order; each is independently useful, so a partial build still pays off.

### 1. Storage — two folders, not one

```
{brain_dir}/
├── 00_Raw/{date}/original.md      ← verbatim ingested source, untouched
└── 10_Wiki/
    ├── Topics/
    ├── Projects/
    ├── Skills/
    └── Decisions/
```

`00_Raw` is the paper trail — whatever came in (a pasted article, a generated report, a video transcript) gets saved exactly as-is, dated. `10_Wiki` holds the AI-restructured notes that actually get retrieved later. Keeping raw and structured separate means a bad restructuring pass never destroys the original, and gives the verification step (§5) something concrete to check against.

Every wiki note needs this frontmatter + section shape — see `references/note-schema.md` for the full spec and a worked example:

```yaml
---
id: <uuid>
category: "[[10_Wiki/Skills]]"
confidence_score: 0.9
tags: [tag1, tag2]
last_reinforced: 2026-08-31
---
## 📌 One-line insight
## 📖 Structured knowledge
## ⚠️ Contradictions & updates
## 🔗 Links (Parent / Related / Source)
```

The category and links use `[[wikilink]]` syntax — that's not decoration, it's the graph that §3 traverses.

### 2. Ingestion — raw in, structured note out, with a checkpoint

Something worth remembering shows up (user pastes a file, or a feature in the app produces output worth keeping) → save it verbatim to `00_Raw/{date}/`. Then **ask before structuring** — "want me to turn this into a knowledge note?" — rather than doing it silently. This isn't just politeness: it's the moment the user can say "no, that run was garbage, don't keep it," which is cheaper than fixing a bad note after the fact.

On confirmation, one LLM call reads the raw content and produces the structured note (frontmatter + sections above), choosing its own category/tags/wikilinks to existing notes. Use structured/JSON output if the target stack supports it (more reliable than parsing markdown back out of freeform text) — see §5 for why this call should also carry the verification check, not just the note content.

**Wire the save button into every stage whose reasoning matters, not just the final deliverable.** It's tempting to only add "save to brain" where a pipeline's *end product* appears (the finished article, the generated song, the final report) — but if an earlier stage is what actually explains *why* that output looks the way it does (a trend/competitive analysis that a later creative-generation stage is grounded in), skipping it leaves a real gap: the assistant can recall *what* was made but not *why*, because the reasoning never became a note. Caught in production: a chatbot could describe a generated song's lyrics and prompts in detail but had no way to answer "why did we pick this topic for this song" — the upstream channel analysis that the topic was chosen from was durably saved to its own history log (for trend computation), but was never wired into the wiki at all. Don't duplicate the full upstream analysis into every downstream note either — that bloats notes and duplicates content that's already saved elsewhere; wire a save button onto the upstream stage itself so it becomes its own note, and let §1's `related`/wikilink matching connect the two.

### 3. Retrieval — Graph RAG without embeddings

This is the part worth stealing even if nothing else in this skill gets built. Full algorithm and scoring details are in `references/graph-rag-algorithm.md`; the shape of it:

1. Score every note by keyword overlap against the current query. Take the top ~3 as **seeds**.
2. Build a graph over all notes: an edge for every resolved `[[wikilink]]`, plus an edge between any two notes that share a distinctive quoted/backtick phrase (an "anchor term") — cheap proxy for "these are about the same specific thing" without running an LLM extraction pass over the whole vault.
3. Expand one hop from the seeds via BFS. Neighbors get pulled in at roughly half their connecting seed's score — this is the actual payoff: a note with zero keyword overlap can still surface because it's linked to something that matched.
4. Rank everything, greedily fill a character budget (roughly 2000–2500 chars normal, 800–1000 chars when the conversation is already long and context is tight), and inject it labeled by how it was found — 🎯 for a direct keyword match, 🔗 for a graph neighbor. That label isn't just for the model; showing it to the user makes the retrieval legible instead of a black box.

No vector store, no embedding calls, no network round-trip — it's regex and a keyword scorer walking the filesystem. That's what keeps it fast on modest local hardware, and it's why this approach beats reaching for an embedding pipeline by default: only add embeddings later if keyword+graph retrieval demonstrably misses things it shouldn't.

### 4. Self-RAG — a trust layer on top of retrieval, toggleable

This is a mode switch, not a permanent behavior — some questions genuinely don't need wiki grounding, and forcing it everywhere just makes casual answers slower and more hedged.

When **on**:
- Surface a running `verified.md` — previously self-grounded claims — with the *highest* priority in context, above the raw retrieval results from §3.
- Require every factual claim in the output to carry `[근거: source]` (grounded) or `[추측]` (guess). End the answer with a one-line self-audit: `자가검증: 사실 N개 / 추측 M개`.
- If guesses outnumber grounded claims, the model should refuse to answer substantively and say so — "정보 부족" — rather than produce a confident-sounding answer built mostly on guesses.
- After the response, scan it for `[근거:]`-tagged lines and append those claims to `verified.md`. Next time, those surface first. This is the self-reinforcing part: what got grounded once becomes the trusted baseline for next time, and it accumulates without any manual curation step.

When **off**: the model just answers from its own general knowledge, no tagging required, no wiki lookup forced. Good default for anything that isn't actually about the accumulated knowledge.

**Better default: auto-detect instead of a manual toggle.** A checkbox the user has to remember to flip before every fact-sensitive question adds friction and gets forgotten. Mirror whatever heuristic the project already uses to decide "does this need a live web search" (a keyword list — "정확히", "몇 개", "언제", "결정해야", decision/quantity/date-sensitive phrasing) and auto-enable Self-RAG on that same signal, keeping the manual toggle only as a "force it on for everything" override. Return *why* it turned on (auto vs. user-forced) alongside the response so the UI can show a small "자동으로 검증 모드 적용됨" indicator — the point is the user never has to think about it, but can still see when it fired.

### 7. Multi-context isolation — when one vault serves more than one domain

Skip this section if the assistant only ever serves one audience/context. Add it the moment a second one shows up: a second client, a second channel, a second product line, or even just a second distinct *feature* writing into the same vault (e.g. a chatbot's own notes vs. a separate content-generation feature's notes). Without it, retrieval eventually surfaces a customer-A specific detail while answering about customer B, or a marketing-team note while answering an engineering question — and it gets *worse* over time as more contexts pile into the same flat category structure, not better.

The fix is cheap and doesn't require redesigning storage:
- Add two more frontmatter fields to every note: a **context tag** (which tenant/channel/audience this belongs to, empty string if it's genuinely context-agnostic — general skills/decisions usually are) and a **source tag** (which feature/subsystem produced it, for observability more than filtering).
- These are **caller-supplied, not AI-inferred** — the code path that triggers note creation already knows which tenant/context it's running in, so just pass that value straight through into the note. Don't ask the LLM to guess it; that's a fact the surrounding system already has for free.
- Filter §3's retrieval by the context tag: a note whose context tag is set and doesn't match the current context is excluded from being a seed *or* a graph neighbor — not down-weighted, excluded, because a wrong-context match is worse than no match. A note with an **empty** context tag (genuinely general knowledge) stays eligible regardless of current context — don't over-filter and lose the shared knowledge that's supposed to be shared.
- The source tag can start as metadata-only (tagged at write time, not yet filtered at read time) — decide whether it needs to become a hard retrieval filter later based on whether cross-feature bleed actually turns out to be a problem in practice, rather than guessing upfront. Channel/tenant bleed is usually worth filtering immediately (wrong-customer data is a real incident); cross-feature bleed inside one tenant is often fine or even useful (one feature's grounded knowledge legitimately informing another) and a hard filter there can remove signal you actually wanted.

### 5. The verification gate at *creation* time — the gap this skill fixes

Self-RAG (§4) verifies claims when the model **answers using** stored knowledge. It says nothing about whether the knowledge was accurate **when it was stored**. In the reference implementation, note creation is: AI restructures the raw content → file gets written → git push, gated only on whether the file write succeeded — never on whether the note is actually faithful to the source. A `confidence_score` field exists in the frontmatter, but it's a value the AI is told to write from a template, not something checked against anything.

Left alone, that's a real problem: a note that quietly drifted from its source doesn't just cause one bad answer — §3's graph traversal means it keeps getting pulled into unrelated future context, and §4 can even auto-promote a claim built on it into `verified.md`, where it then outranks everything else. One ungrounded note early on can quietly poison a lot of answers downstream, and it'll look *more* trustworthy over time, not less, since nothing is checking it against the source it supposedly came from.

The fix costs almost nothing if done right: **have the same LLM call from §2 also emit a grounding self-check** — a structured field alongside the note content, e.g. `{ "note": {...}, "grounded": true/false, "issues": ["claim X isn't actually in the source"] }` — rather than firing off a second, separate verification call. One call, one extra field, checked before the note gets saved or pushed; if `grounded` is false, surface the issues and don't commit. This is a few-second, occasional-event cost (only happens when someone deliberately saves to the brain, not on every chat turn), and it's the one piece of this whole design that's non-negotiable if the surrounding project holds itself to a "never let fabricated content pass as verified fact" standard — don't build the rest of this pattern without it.

**Don't stop at "surface the issues and don't commit" — feed them back for one corrective retry.** A `grounded: false` result already tells you exactly what's wrong (the `issues` array). Throwing that away and either giving up or making the user click save again (which re-triggers the identical prompt with zero memory of what failed) wastes the one piece of information you just got for free. Instead: call the same structuring endpoint again, this time passing the previous attempt's content and its `issues` back in as extra context, with an instruction to fix only the flagged parts using content actually present in the source (or drop the unsupported claim entirely rather than keep guessing). Cap it at one retry — if it's still `grounded: false` after that, stop and surface the block; a retry loop just hides the failure instead of showing it. See [[ai-judgment-verification-step]]'s "self-critique-via-schema + one corrective retry" section for the shipped version of this.

### 6. Sync + visualization — durability and legibility

**Git sync**: after any note is created or edited, `git add . && git commit && git pull (prefer local on conflict) && git push` against a dedicated repo for the vault — separate from the app's own code repo. This is what makes the brain survive a dead laptop; it also means the vault is just plain markdown files that can be opened directly in Obsidian or any editor, not something locked inside the app.

**Visualization** (optional, do it last): render notes as nodes and wikilinks as edges with a force-directed graph library (e.g. `force-graph`, a small canvas/WebGL JS library with no server dependency) — an Obsidian-graph-view equivalent. Nice for the user to see what the system actually knows; not load-bearing for the retrieval to work.

## Adapting to the target stack

None of the above is tied to any particular language or framework — the reference implementation is a TypeScript VS Code extension, but the pattern has been applied to a Python/FastAPI + vanilla-JS app just as well. Concretely translate:
- "00_Raw / 10_Wiki folders" → wherever the target app keeps its local data (e.g. `data/brain/...`)
- "one LLM call with structured output" → whatever structured-output mechanism the target stack's LLM client supports (JSON schema, function calling, or careful prompting + parsing if neither is available)
- "git sync" → shell out to `git`, same commands, any OS
- Keep the note schema (frontmatter fields, section headers) identical across projects even when the surrounding code differs — that consistency is what lets notes from different projects stay legible to each other and to Obsidian.

## What not to do

- Don't skip §5 to save a round-trip. It's designed to *not* cost a round-trip — fold it into the structuring call.
- Don't reach for an embedding model/vector DB before trying §3's keyword+graph approach. It's simpler, faster on local hardware, and the reference system runs on it in production.
- Don't force Self-RAG's grounded/guess tagging on every single message — reserve it for answers that are actually drawing on the wiki.
- Don't auto-structure without the confirm step in §2 — a quick "save this?" is cheap; retroactively cleaning bad notes out of a graph that's already been traversed and promoted into `verified.md` is not.
- Don't wait until a second tenant/channel actually causes a visible mix-up to add §7's context tagging — retrofitting it means re-tagging every existing note; adding the field from day one costs nothing since it can just stay empty until it's needed.
- Don't make Self-RAG a checkbox-only feature if the project already has a "should I search live" heuristic elsewhere — reuse that pattern for auto-detection rather than shipping one more toggle for the user to manage.
