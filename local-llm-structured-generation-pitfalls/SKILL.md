---
name: local-llm-structured-generation-pitfalls
description: A checklist of specific, hard-won bugs that recur when building any feature where a local/small LLM (via LM Studio, Ollama, or similar) generates structured content (JSON-schema-constrained output, few-shot-guided text, template-following prompts) with client-side validation and auto-retry. Use this whenever writing or debugging a prompt with a JSON schema response format, a "format example" or few-shot block, a validation+auto-retry loop, or any feature where a small local model needs to follow a strict structural template (fixed section counts, exact repeated text, precise metadata fields). Also use it when a generated result looks subtly wrong in a way that's hard to explain (extra text bleeding into output, validation flagging something that should be fine, a fix that silently didn't take effect). These are real bugs found and fixed in production, not theoretical advice.
---

# Local-LLM Structured Generation — Known Pitfalls

## Why this exists

Building features where a small local model must follow a strict structural template (exact section line-counts, JSON schema fields, verbatim-repeat sections) surfaces the same handful of failure modes over and over, across unrelated projects. None of these are exotic — they're all "obvious in hindsight, invisible while writing the code" — which is exactly why they're worth a checklist instead of re-discovering each one from scratch every time.

## 1. Few-shot format examples leak their own scaffolding into real output

If a prompt shows a format example using a suspicious repeated meta-pattern to demonstrate structure — e.g. numbering each example line `"(1번째 줄) ..."` to show "this section has 4 lines" — the model will sometimes copy the numbering pattern itself into the real generated content, not just absorb the line-count it was meant to demonstrate. It happened even with an explicit "don't copy this content" instruction, because the instruction addressed *content* but the leak was *format*.

**Fix**: make format examples read as genuinely plausible real content with no artificial markup (real-sounding sentences, not placeholder annotations), and add a second, separate instruction that names the specific leak pattern to avoid ("don't prefix lines with numbers or bracketed labels") — don't rely on a generic "don't copy this" instruction to cover a structural leak, since that instruction is aimed at content, not format.

## 2. Client-side validation and server-side prompt rules drift out of sync

When a project has a fixed validation rule ("this section must be exactly 4 lines") baked into client-side validation code, and *also* a prompt on the server describing that same rule to the model, the two are easy to update independently and get wrong in different ways: the prompt gets a new rule for a new content variant (e.g. a second document "genre" with a different line-count target) but the validator keeps using the old fixed rule everywhere — so a correctly-generated variant gets flagged as broken, and the auto-retry loop "fixes" something that was never wrong.

**Fix**: key both the prompt-side rule and the validator-side rule off the same variant identifier (e.g. a genre/category/type field), and when adding a new variant, grep for every place the *old* fixed rule was hardcoded — don't assume one prompt edit is sufficient. If feasible, derive the validator's expected values from the same source-of-truth structure the prompt is built from, rather than maintaining two independent copies of the same numbers.

## 3. A new safety-net (retry/validation loop) doesn't automatically cover every path that produces the same artifact

A validation+auto-retry loop added to the "generate this from scratch" code path is easy to forget adding to a separate "regenerate/edit this" code path that produces the *same kind* of artifact through different code. The result: the first-generation path is well-guarded, but a user-triggered "fix this" or "try again" action on existing content silently skips the safety net entirely, and a bug that was supposedly fixed keeps reappearing exactly there.

**Fix**: when adding a validation/retry safety-net, explicitly search for every function/endpoint that can produce or re-produce the artifact in question (grep for the shared response schema or shared content field), not just the one you were looking at when you noticed the bug.

## 4. Cached "prep"/prompt objects go stale across a long-lived UI session

If a frontend fetches a system prompt once (at page load, or at "generate" click time) and then reuses that cached object for every subsequent action in the same session (e.g. a "regenerate" button reusing the original generation's prompt), a server-side prompt fix made *after* the page was loaded never takes effect for that session — the user keeps hitting the exact bug that was supposedly already fixed, until they hard-reload.

**Fix**: for any action that re-invokes generation (regenerate, retry, edit), refetch the prep/prompt fresh at click time rather than reusing a cached reference from earlier in the session — the extra network round-trip is cheap; a user silently stuck on a stale, buggy prompt is not.

## 5. Don't ask the model to embed decorative formatting (emoji, brackets, fixed labels) inside a freeform string field

A schema field typed as a plain string array, with the prompt asking for entries like `"🎯 [Section Name]: the actual content"`, puts the model in charge of getting *punctuation* exactly right, not just content — and small/local models drop or mangle it more often than the content itself is wrong (missing emoji, missing brackets, an empty first entry). This showed up as a real bug: a "hook points" list where the first item silently rendered as an empty bullet and the emoji vanished from the others, on a small non-reasoning model — not a token-budget issue (usage was nowhere near the limit), purely a formatting-fidelity miss.

**Fix**: whenever a field has a small fixed set of possible labels (a title/section name/category that repeats across items), split it into two schema fields — a `label` constrained by a JSON-schema `enum` of the exact allowed values, and a `text`/`detail` field for the actual free-written content. Let the calling code deterministically attach the emoji/brackets/prefix for display, keyed off the label. This doesn't just fix formatting reliability — the `enum` constraint makes the wrong-label failure mode structurally impossible instead of just less likely. Apply the same move to any array where every item shares a constant static prefix (e.g. always "💬 "): drop the prefix from what the model writes entirely and prepend it in the rendering code, since a constant string never needed to come from the model in the first place.

## 6. Don't let the model generate metadata the calling code already knows

Fields like tenant/channel identity, a fixed category the UI button already determined, or other deterministic context should be passed straight through from the calling code into the saved/generated record — never put them in the LLM's JSON schema as a field for the model to infer or repeat back. An LLM asked to echo back a value it was already given can still drift (rephrase it, drop it, or literally get it wrong), and there's no reason to spend model attention or risk on a fact the surrounding code already has for free.

## 7. A narrow "cite the specific thing, or admit there isn't one" field needs an explicit abstention instruction with a negative example

Any schema field asking the model to identify one specific matching thing out of many candidates (a specific source citation, a specific matching category, a specific root cause) needs an explicit instruction to leave it empty/null when nothing genuinely matches — otherwise a model under JSON-schema pressure to fill every required field will confidently fabricate a plausible-sounding match rather than abstain. State the negative case explicitly ("if no specific X clearly applies, leave this field as an empty string — do not force a loose or approximate match") and test it against an input deliberately chosen to have no good match, not just an input engineered to have an obvious one.

## 8. Reasoning-style local models can silently burn most of the output budget on invisible "thinking" tokens

A "reasoning" local model (one that emits internal chain-of-thought before its final answer, exposed via a separate `reasoning_tokens` field in the API usage stats) can consume the majority of `max_tokens` on that invisible reasoning even for a short, simple task — measured at ~55-70% of a 2000-token budget in production, consistently across repeated runs. `finish_reason` still reports `"stop"` when it happens to fit, giving no warning sign — but the margin left for the actual visible content is thin enough that a slightly more complex prompt or longer input can push it over and truncate the *visible* JSON mid-structure, which reads as a formatting bug (a missing array item, a cut-off field) rather than what it is (a budget problem). A same non-reasoning small model (`reasoning_tokens: 0`) has none of this overhead and comfortably fits the same task in a fraction of the budget.

**Fix**: check the `reasoning_tokens` field (if the API surfaces it) when sizing `max_tokens` for any reasoning-capable local model — budget for reasoning overhead *plus* the desired visible content, not just the visible content alone. When two model tiers are both in scope (a larger reasoning model and a smaller non-reasoning one, e.g. for users on lighter hardware), size the shared budget for the reasoning model's overhead; the non-reasoning model will just finish well under it.

## 9. Python-specific: f-string nesting gotchas (applies directly if the stack is Python)

- A plain (non-f) string nested inside an f-string's `{}` expression does **not** get its own `{...}` placeholders interpolated — Python only evaluates the expression itself (which may *be* a plain string, so the whole thing degrades to that string's literal text, braces and all) rather than recursively treating it as another f-string. This fails silently: no exception, just literal `{var}` text leaking into the rendered output. If a large conditional block inside an f-string needs its own interpolation, prefix that inner string with `f` too, or (cleaner) build it as a separate variable beforehand and reference the variable.
- An f-string expression (the part inside `{}`) cannot contain a backslash in Python versions before 3.12 — this includes escaped quotes like `\"...\"` used to embed quoted text inside a conditional expression. Build the string with the escaped quotes as a plain variable *outside* the f-string first, then reference that variable inside `{}`.
- After any edit that adds or changes an f-string with escaped quotes or nested nesting, run the actual syntax check (`python -m py_compile` / `ast.parse`) before assuming the change is correct — both of the above fail in ways that are easy to miss on a visual read.

## 10. An optional/missing sub-field is not a reason to skip the whole required output

When one generated item in a per-item pipeline (one scene of many, one row of many) has an optional detail that legitimately comes back empty — a fact-based label the model correctly declined to fabricate because nothing applied, a citation left blank per pitfall #7 — it's tempting to gate the *entire* item's downstream artifact on that detail being present ("only render this scene's output box if it has a label"). That silently drops the item's mandatory output whenever the optional detail is absent, and if the model is well-behaved and abstains often (exactly the behavior pitfall #7 asks for), a large fraction of items can end up with no output at all — which reads to the user as "half my scenes just didn't generate" rather than "the model correctly found nothing to cite here." This is a case of two independent instructions accidentally being wired together at the render layer, not by the model or the schema.

**Fix**: keep "does this item produce its required artifact" and "does this item have the optional detail" as two separate conditions. The mandatory output (built from content that always exists — e.g. the scene description) should render unconditionally; the optional detail should only affect what's *inside* that output (an empty array/field instead of a fabricated one), never *whether* the output appears.

## 11. A JSON blob that parses cleanly and matches your own schema can still violate the spec you were handed

Passing `JSON.parse()` and having the right top-level keys only proves internal self-consistency with the schema *you* designed — it says nothing about whether that design actually satisfies the constraint the user (or the reference doc) originally gave. A concrete case: a spec said "at most 1 piece of text per generated item, to avoid text-mangling downstream." The implementation split "text" into two schema fields — an annotation label and a separate caption — reasoning each field was legitimate on its own. Both fields were populated, the JSON was well-formed, and automated parsing tests all passed. But two populated text fields is two pieces of text, which is exactly what "at most 1" ruled out — the violation was invisible to any check that only asked "is this valid JSON matching my schema," and only surfaced when the user looked at real rendered images and it visually reads as "the same style everywhere," a full generation loop and human read the actual output against the *original* constraint.

**Fix**: after building a structured-output schema from a spec, re-read the spec's own constraints (counts, "at most N", mutual exclusivity) as a checklist against the schema you just designed — not just against the final generated values. Self-consistency checks (parses, has the right keys, is internally coherent) catch a different class of bug than spec-compliance checks (does it violate a rule from the source document); both are needed, and passing the first is not evidence for the second. When the downstream consumer of the JSON is an external system, a full-loop test through that actual consumer (not just the JSON validator) is what catches this class of gap — this is what caught both #10 and #11 in practice, not the parsing test.

## What not to do

- Don't ship a format example with any kind of artificial per-line markup (numbering, bracketed meta-labels) even when explicitly telling the model not to copy the *content* — the leak risk is in the markup pattern itself, not the content.
- Don't maintain the same structural rule in two places (prompt text and validator code) without a shared key to keep them in sync when a new variant is added.
- Don't assume a retry/validation safety-net automatically covers a "regenerate" or "edit" button just because it covers the original "generate" button — check both explicitly.
- Don't reuse a cached prep/prompt object for actions that happen later in a long UI session — refetch when the action fires.
- Don't add an LLM schema field for something the calling code already knows deterministically.
- Don't ask a model to reproduce exact decorative punctuation (emoji, brackets) as part of freeform text — split label from content and add the decoration in code.
- Don't size `max_tokens` for a reasoning-capable model off the visible content alone — check `reasoning_tokens` in the usage stats and budget for both.
- Don't skip a syntax check after editing an f-string with escaped quotes or a nested string literal — this class of bug compiles fine visually and fails silently or throws a confusing error at request time instead of at edit time.
- Don't gate a required per-item output on an optional sub-field being present — render the mandatory artifact unconditionally and let the optional detail vary only what's inside it.
- Don't treat "parses as valid JSON matching my schema" as proof the output satisfies the original spec — re-check the spec's own constraints (counts, exclusivity) against the schema design, and test through the real downstream consumer when one exists.
