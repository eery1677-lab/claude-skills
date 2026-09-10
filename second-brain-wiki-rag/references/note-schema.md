# Wiki note schema ("P-Reinforce" format)

The exact frontmatter + section shape every structured note should follow, validated against real notes in a production vault. Keep this identical across projects — consistency here is what makes notes from different apps stay legible to each other, and to Obsidian or any plain markdown reader.

## Frontmatter

```yaml
---
id: lm-studio-local-deployment          # stable slug or uuid — used for dedup/updates
category: "[[10_Wiki/Skills]]"          # wikilink to the folder this note lives in
confidence_score: 0.9                   # see note below — don't leave this un-computed
tags: [lm-studio, local-llm, deployment]
last_reinforced: 2026-08-31             # date of last creation OR meaningful update
---
```

**On `confidence_score`**: in the reference implementation this is just a value the structuring prompt tells the model to write (usually defaults to `0.9`) — it is *not* computed from anything. Don't repeat that. If the verification gate (skill §5) runs, feed its result into this field instead of a hardcoded default — e.g. high confidence when grounded and clean, lower when the self-check flagged something that got kept anyway with a caveat. A confidence score nobody checked is worse than no confidence score, because it looks like a signal.

## Body sections, in this order

```markdown
# [[Note Title]]

## 📌 One-line insight
> The single most important takeaway, one sentence. Someone should be able to
> read only this line and know whether the full note is worth opening.

## 📖 Structured knowledge
The actual content — organized with subheadings, bullets, code blocks, whatever
the source material calls for. This is where the bulk of the useful detail lives.

## ⚠️ Contradictions & updates
Populated later, not at creation. When new information conflicts with something
already in this note, log the conflict here rather than silently overwriting the
old claim — e.g. "Assumed X worked this way based on the docs; testing on
2026-09-14 showed Y instead, corrected above." This is what keeps the note
honest over time instead of just reflecting whoever wrote it last.

## 🔗 Links (Graph)
- **Parent**: [[10_Wiki/Skills]]
- **Related**: [[Another Note Title]], [[Yet Another Note]]
- **Source**: [[00_Raw/2026-08-31/original_dump]] or an external URL
```

The `🔗 Links` section is not optional decoration — it's the literal graph edges that the retrieval algorithm (see `graph-rag-algorithm.md`) traverses. A note with no outgoing links is an island: it can only ever be found by direct keyword match, never surfaced as a graph neighbor of something else the user asked about.

## Worked example (real note, lightly trimmed)

```markdown
---
id: lm-studio-local-deployment
category: "[[10_Wiki/Skills]]"
confidence_score: 0.98
tags: [lm-studio, local-llm, deployment, gguf, cli, sdk, api]
last_reinforced: 2026-06-20
---

# [[LM Studio local LLM deployment guide]]

## 📌 One-line insight
> Download a fine-tuned GGUF model into LM Studio, then connect to it via the
> `lms` CLI or the OpenAI-compatible local endpoint for flexible integration.

## 📖 Structured knowledge

### 1. GUI and CLI setup
- Search for a model by Hugging Face ID, pick a quantization level that fits
  available VRAM (Q4_K_M is a reasonable default), load it.
- `lms server start --port 1234` boots a headless OpenAI-compatible server.

### 2. OpenAI-compatible local API
```python
from openai import OpenAI
client = OpenAI(base_url="http://localhost:1234/v1", api_key="lm-studio")
```

## ⚠️ Contradictions & updates
- Loading the full FP16 model instead of a quantized one caused a RAM
  bottleneck and made the local server unusably slow — Q4_K_M or Q8_0 is the
  practical tradeoff, not a fallback.

## 🔗 Links (Graph)
- **Parent**: [[10_Wiki/Skills]]
- **Related**: [[Fine-tuning and memory workflow]], [[connect-ai-lab extension]]
- **Source**: `https://lmstudio.ai/docs`
```

Notice the `⚠️ Contradictions & updates` section is doing real work here — it's not restating the main content, it's recording a specific thing that turned out to be wrong in practice and why the corrected guidance should be trusted over a naive first attempt.

**⚠️ Parser warning, confirmed as a real production bug**: notice this example's own `## 📖 Structured knowledge` section contains `### 1. GUI and CLI setup` and `### 2. OpenAI-compatible local API` as internal subheadings — this is normal and expected for any note with more than a couple paragraphs of real content. When writing the code that extracts this section's body (for retrieval — see `graph-rag-algorithm.md`), **do not find the section's end by looking for "the next heading-like line."** A regex like "capture until the next `##`" will stop at that note's own `### 1.` subheading (a bare `##\s` pattern matches inside `### ` too, so this breaks on `###` subheadings as well as `##` ones) and silently truncate everything after it — in a real production instance this reduced a 2,000+ character note down to 30 characters, with no error anywhere. Find the end of `📖 Structured knowledge` by matching specifically for the *next section's own fixed marker* — `## ⚠️` or `## 🔗` — never a generic "any heading" pattern, since the whole point of this section is that it's allowed to contain the source material's own heading structure.

## Folder placement

- `10_Wiki/Topics/` — conceptual/explanatory notes (how something works, a technique, a domain fact)
- `10_Wiki/Projects/` — notes tied to a specific project's ongoing work
- `10_Wiki/Skills/` — reusable how-to's, procedures, setup guides
- `10_Wiki/Decisions/` — a choice that was made and why, so it doesn't get silently re-litigated later

When the structuring LLM call picks a folder, it should be choosing one of these four (or the target project's equivalent taxonomy) — don't invent a new category per note; the value of the taxonomy comes from notes actually clustering into it.
