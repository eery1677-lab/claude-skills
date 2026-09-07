---
name: self-improving-agent-strategy-memory
description: How to give an agent (especially one built on a local/small model) a persistent, growing memory of *which operational strategies worked and which failed* for specific kinds of tasks — so it stops re-discovering the same failure every run and starts each new attempt already knowing what to try first. Use this whenever the user asks for an agent that "learns from its mistakes," "gets better over time," "doesn't repeat the same error," or describes wanting reinforcement-learning-like self-improvement without an actual training loop. This is grounded in real published agent-memory research (Voyager, ExpeL, Agent Workflow Memory) and in a real hand-built instance of the same idea already working in one of the user's own projects (a YouTube-thumbnail "learning log") — not theory alone.
---

# Self-Improving Agent Strategy Memory

## The distinction this skill is about

There are two different kinds of memory an agent can have, and it's easy to build one and think you've built the other:

1. **Domain/content memory** — "what do I know about this business, this channel, this topic." [[second-brain-wiki-rag]] already covers this well.
2. **Operational strategy memory** — "when I try to do X, approach A fails for reason R, and approach B works instead." This is what this skill is about, and it's a different axis entirely — it's memory about *how to act*, not memory about *facts*.

Without #2, an agent (or a human re-invoking the same agent fresh each session) re-derives the same fix from scratch every time it hits the same wall — which is exactly what happened in this project on 2026-09-05: a parallel-call rate-limit bug in a free translation wrapper had to be rediscovered live, when a sibling agent (Luka) had already hit and solved the identical problem earlier, in a way that was sitting right there in its code as comments but wasn't in a form that got *consulted before acting*.

## What the research says (this is a real, active area — not folklore)

- **[Voyager](https://arxiv.org/abs/2305.16291)** (2023) — an embodied Minecraft agent with three parts: an automatic curriculum, an **ever-growing skill library of executable, verified code**, and an iterative prompting loop that uses environment feedback + self-verification to refine a skill until it actually works before saving it. The skill library is retrieved from and added to across the agent's entire lifetime — this compounds capability and avoids "catastrophic forgetting" of things that used to work.
- **[ExpeL: LLM Agents Are Experiential Learners](https://arxiv.org/abs/2308.10144)** (2023) — extracts cross-task **natural-language insights** from past trajectories (including failed ones) without any fine-tuning, and retrieves the most similar past experience at inference time to inform the current attempt.
- **Agent Workflow Memory (AWM)** (2024) — instead of single-line "lessons," abstracts whole successful **task trajectories into reusable workflows** that later tasks can be matched against and reused wholesale.
- More recent 2026 work (e.g. "Experiential Reflective Learning," "Agentic Context Engineering") pushes the same idea further: build a pool of reusable heuristics that explicitly captures **both effective strategies and failure modes**, and evolve the agent's context/prompt over time rather than only its outputs.

The common shape across all of them: **retrieve similar past experience before acting → act → verify the outcome → write the (successful or failed) outcome back to memory** — a loop, not a one-shot log.

## Why the Voyager-style version needs adapting for a local/small model

Voyager leans on GPT-4's strong in-context reasoning to *improvise* a fix when something fails, live, unsupervised. A small local model is exactly the kind that [[ai-judgment-verification-step]] and [[local-llm-structured-generation-pitfalls]] already document as unreliable at that kind of live creative self-diagnosis — it can follow a known recipe well, but "cleverly notice why this failed and invent a working alternative on the spot" is a much higher bar, and expecting it silently degrades into the failure mode this whole project has been guarding against: confident-looking output that's actually wrong, with nobody around to catch it.

**The practical adaptation**: split "discovering a new strategy" from "using a known strategy" into two very different-cost operations:
- **Discovering a new strategy** (a genuinely novel failure, no matching playbook entry) is expensive and should route to a *stronger* model or a human — exactly what happened this session: a real bug surfaced, a larger reasoning process (not the local model) diagnosed it, found the working alternative (by reading a sibling agent's already-solved code), and the result got written down.
- **Using a known strategy** (this task matches a playbook entry seen before) is cheap and *is* something a local model — or even plain code — can do reliably: look up the entry, apply the documented fix, verify the mechanical/checkable part of the outcome (does it match the expected shape? did the known error re-appear?).

This means the local-friendly version of "self-improving" is less "the small model cleverly adapts on its own" and more **"the system accumulates a growing, code-checkable playbook that gets consulted automatically before every attempt of a task type it's seen before, and new entries get added by whoever/whatever actually solves a novel failure — local model, larger model, or a human."** That's a realistic, honest target that doesn't oversell what a small model can do unsupervised, while still delivering the actual thing the user wants: *not making the same mistake twice.*

## The concrete loop to build

1. **Before attempting a task**: query the strategy memory for entries matching this task's type/signature (reuse [[second-brain-wiki-rag]]'s keyword+graph retrieval — no need for a separate retrieval system). If a matching entry exists, inject it into the prompt/plan as a directive ("known issue: X fails this way, use Y instead") rather than letting the model rediscover it.
2. **Attempt the task.**
3. **Check the outcome with code where possible**, not just the model's self-report — a numeric check, a schema match, an HTTP status code, a "does the known error string appear in the log" grep. This is the same discipline as [[ai-judgment-verification-step]]: code decides pass/fail where a mechanical check exists; only fall back to model judgment where it genuinely can't be checked mechanically.
4. **Write the outcome back**, success or failure, as its own note: what was tried, under what conditions, what happened, and (if it failed) what fixed it once a fix was found. Use the same `00_Raw`/`10_Wiki` + `grounded`/`issues` self-check shape [[second-brain-wiki-rag]] already has — a strategy note is just a different *category* of note in the same vault, not a different system.
5. **Never re-attempt a documented dead end** without a human or larger-model override — if the playbook says "approach A fails for reason R," don't let a fresh agent instance try A again from scratch just because it doesn't remember trying it. This is the single biggest practical win: it's cheap to check "have I already ruled this out" and expensive to keep re-discovering the same wall.

## Real evidence this already works, hand-built, in the user's own project

Before formalizing this, check whether a cruder version already exists — it's cheaper to extend than to build from zero. In this case it did: `docs/thumbnail_learning_log.md` in a sibling agent's project (Luka, at `/Users/kambo/에이전트요원들/안티그래비티 요원들/직원들/루카에이전트/`) is a hand-authored instance of exactly this loop for one domain (YouTube thumbnail generation):
- A running "잘한 것 / 잘못한 것" (good/bad practices) log with concrete, specific past cases (e.g., "AI 엔진 기본 출력이 1024×1024인데 사전 보정 없이 올려서 유튜브에서 레터박스가 생김").
- A section literally titled "루카 AI 강화학습 액션 플랜" (reinforcement-learning action plan) that converts each lesson into a **mandatory rule** for all future runs ("이 규칙을 100% 의무 준수합니다").

This is ExpeL/Voyager's loop, done manually instead of automatically, and it works. **It was not applied everywhere, though** — checked directly (read-only) whether the same treatment existed for subtitle/translation work in the same agent folder, and it didn't: `build_global_subtitles.py` has good inline comments explaining *why* it batches with pacing, but there's no equivalent `subtitle_learning_log.md` turning that into an enforced, retrievable rule the way thumbnails got. That gap is exactly why the same rate-limit mistake had to be rediscovered from scratch in a different project rather than being looked up. **Lesson**: when you build this pattern for one domain of a project, audit whether it's missing from adjacent domains that hit similarly failure-prone external dependencies (rate limits, flaky APIs, format-sensitive outputs) — don't assume "we do this here" means "we do this everywhere."

## A second real case: decompose the generation task itself, not just the retry logic

A different domain, same underlying lesson, found 2026-09-07 in 나만의콘텐츠생성기's blog-writing pipeline. The task was "write a full blog post" as a single JSON response with a `body` field. A local small model (`google/gemma-4-e2b`) given that task wrote a 172-character summary — literally the marketing plan's own 3-line Zero-Click summary copy-pasted — and stopped, treating the JSON object as "complete" long before the article was actually written. Bumping `max_tokens` didn't matter (the model wasn't truncated, it voluntarily stopped short), and adding an explicit "write at least 1000 characters" instruction to the field description was only a partial, brittle fix.

The real fix was to stop asking for the whole article in one call and decompose the *generation task itself* into per-section calls: the outline stage now returns a JSON array of section objects (`heading`/`summary`/`alt_text`), and the draft stage loops over that array, calling the model once per section with a narrow, concrete instruction ("write only this section, 3-4 paragraphs") plus the text written so far (last ~1500 chars) so it doesn't repeat itself or lose the thread. A final small call reads the fully-assembled body and produces just `title_suggestions` + `hashtags` — a task small enough that the same short-completion failure mode doesn't reappear. Verified result: three sections of 448-582 characters each (evenly sized, not one section absorbing the whole budget), 1536 characters total, versus 172 before.

**The general lesson**: when a local/small model under-delivers on a "write X" task despite instructions and token budget, the first thing to try is not a stronger instruction — it's asking whether the task itself is actually several smaller tasks pretending to be one. A model that reliably writes 3 good paragraphs when asked for exactly that will often fail at "write 3000 good characters covering these 3 topics" as a single request, because length-as-a-target is a much fuzzier completion signal than "cover this one specific thing and stop." This is the same shape as the CapCut clip-per-color-variant lesson from the same session (one Flow AI video-generation call can't show 6 products because it's one continuous shot; the fix was N short per-product clips, not a cleverer single prompt) — **when a single generation call is asked to do multiple discrete things (multiple products, multiple sections, multiple languages), split it into one call per discrete thing before trying to prompt-engineer your way out.**

A secondary trap found in the same fix: a shared instruction block meant for the *whole* article ("body must be 1500+ characters total") was being injected verbatim into each *per-section* call too, creating a real risk that the model would try to hit the whole-article length target inside a single section. Whenever you decompose a task like this, audit every shared/reused instruction block for scope mismatches — an instruction correct at the old granularity can become actively wrong at the new, smaller granularity.

## What not to do

- Don't expect a local/small model to reliably invent a novel fix unsupervised and then judge for itself that the fix worked — that's asking it to do the exact thing it's least reliable at. Let code-checkable verification and/or a stronger reviewer gate whether something becomes a permanent playbook entry.
- Don't build a second, parallel memory system for this — extend the existing wiki vault with a `Strategies`/`Playbook` category (or reuse `Decisions`) rather than inventing new storage, retrieval, or sync machinery.
- Don't let "self-improving" become an excuse to skip human review on anything with real business risk — this pattern reduces *repeated* mistakes, it doesn't eliminate *novel* ones, and [[ai-judgment-verification-step]]'s "known limitation" caveat about verification not being a correctness guarantee applies here just as much.
