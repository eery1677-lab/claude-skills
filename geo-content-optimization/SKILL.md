---
name: geo-content-optimization
description: How to write and structure public-facing content (blog posts, landing pages, product descriptions, video/upload descriptions, social posts, SEO copy) so it actually gets cited inside AI answers (ChatGPT, Perplexity, Google AI Mode/Overview) — not just ranked by traditional search. Use this whenever content is being written or reviewed for discoverability, whenever the user mentions SEO, GEO, "생성엔진최적화", getting found by AI, getting cited/recommended by AI, or wants more traffic/visibility for a site, product, or channel. This is evidence-backed (two real papers, not folklore) and specifically favors small/solo operators over big brands — treat it as the default lens for any "how do I get people to find this" question, not just an SEO checklist.
---

# GEO: Generative Engine Optimization

## Why this is different from old SEO advice

Most SEO instinct is stale for one big reason: **people increasingly ask an AI instead of scrolling ten blue links.** Getting ranked on a results page and getting *quoted inside an AI's answer* are related but distinct problems, and the second one runs on different rules — rules that have actually been measured, not guessed at. Two real studies anchor this skill:

- **Aggarwal et al., "GEO: Generative Engine Optimization," ACM SIGKDD 2024 (arXiv:2311.09735)** — tested 9 content techniques against 10,000 real queries, measuring how much and how early a source gets cited in an AI answer. Up to 40% visibility gain from the right techniques; keyword repetition did essentially nothing. **The standout finding: GEO helps small, lower-ranked sites proportionally more than big brands.** That's the opposite of traditional SEO, where budget and domain authority dominate — this is a rare case where a solo operator has a real structural edge, not just less-bad odds.
- **A 149,912-citation source analysis** — only **2.9%** of what an AI cites is a brand's own official site. The other ~97% is third-party (review blogs, comparison posts, ranking listicles). Of the third-party citations, **35.7%** came specifically from "Top N" ranking-list content.

Full technique details, domain-specific breakdowns, and the exact numbers are in `references/geo-research-findings.md` — read it when you need to justify a specific recommendation or want the precise figures. This file covers the actionable playbook.

## The core mental model: content lives in a vector space

An AI answer engine embeds the user's question and every candidate piece of content into the same vector space, then pulls whatever sits closest. This one idea explains almost every tactic below:

- A broad query ("photo app") is a **crowded neighborhood** — big brands and ad spend already sit there, so a small site is effectively far from it even if it's technically relevant.
- A hyper-specific query ("AI that resizes a selfie to passport-photo spec") is a **nearly empty neighborhood** — a handful of pieces of content aimed exactly at that phrasing become the closest match by default, at near-zero cost, because nothing else is competing for that spot.

**So: go extremely niche first, dominate that narrow space completely, then expand outward into adjacent niches.** Starting broad just means getting outspent immediately by whoever already owns the crowded neighborhood. This applies identically to a website, a YouTube channel, or any other embedding-indexed content — the same logic governs what an AI recommends from video platforms.

## The playbook

### 1. Do SEO and GEO together, not one instead of the other
SEO (keywords, backlinks, crawlability) is the foundation; GEO (getting quoted in an AI answer) sits on top of it. ~75% of search is still traditional, and Google's own AI Mode is still built on the Google index — SEO fundamentals still pay into GEO. Don't rip out SEO practices to chase GEO; add to them.

### 2. Lead with the answer, not the windup
AI extraction favors direct, specific, quotable statements over scene-setting or hedged prose.

- ❌ "저희 서비스는 다양한 장점이 있으며 여러모로 도움이 될 수 있습니다" (vague, hedged, nothing to quote)
- ✅ "이 서비스는 반려동물 프로필 사진을 30초에 만든다. 건당 2,000원, 무제한 재생성." (specific claim, specific numbers, upfront)

Put the core claim or value proposition in the first sentence. Save the elaboration, backstory, and caveats for after.

### 3. Build "evidence units" into every claim
This is the part that actually moved the needle in the research — not keyword density, which had ~zero effect. Four ingredients, and which one to lean on depends on the domain:

- **Cite sources** next to claims (a link or named reference, not just an assertion)
- **Include quotations** — expert, user, or research quotes carry weight AI models trust
- **Use concrete statistics** instead of vague quantifiers — "3,502명·4.8점" beats "많은 분들이 만족"
- **Write for mentions, not just links** — AI models weight your brand/product name *appearing in running text next to the problem it solves* more heavily than a hyperlink pointing at you (mention co-occurrence is roughly 3x more predictive of citation than backlinks in the cited research). Get named in the same sentence as the category, across many pieces of content — not just linked to.

Match technique to domain: **statistics and cited sources** work best for fact-heavy/opinion-adjacent domains (legal, policy, "which is better" comparisons); **expert quotations** work best for authority-driven domains (humanities, social commentary, history/context). One size doesn't fit every topic — see `references/geo-research-findings.md` for the domain breakdown.

### 4. Get into "best-of" lists — this is the single highest-leverage move
Since ~97% of AI citations are third-party pages and over a third of those are ranking-listicle format, optimizing only your own site is optimizing the 3% that matters least. Two-pronged tactic:

- **Author your own** honest "Best [category] tools/services" roundup that includes your product alongside real competitors — don't fake objectivity, just be genuinely useful and include yourself where you honestly belong.
- **Get into other people's lists** — proactively reach out to bloggers/YouTubers already publishing "best of X" content in your niche and offer your product for review/inclusion.

Don't stop at fixing your own landing page. The data says that's structurally where AI looks least.

### 5. Don't treat FAQ/schema as a silver bullet
Basic schema markup is real but modest hygiene (~+13% in the cited research) — worth doing, not worth over-investing in. Building an FAQ-heavy page expecting it to drive GEO wins is based on old SEO folklore that the actual measurement contradicts: Q&A-formatted pages scored slightly *lower* (-5.7%) on citation rate in the study. Spend the effort on evidence units (§3) and listicle presence (§4) instead.

### 6. Measure it like an experiment, because it behaves like one
AI answers are non-deterministic — a single before/after check is noise, not signal.

- Fix a set of **25–50 realistic benchmark questions** your actual target users would ask an AI assistant in your niche.
- Ask each one **3–5 times per check** (not once) and look at the distribution, not a single sample.
- Judge success by **multi-week trend**, never a single snapshot.
- Be honest with whoever you're reporting to: there is no "guarantee" — citation is probabilistic. Don't oversell a result from one lucky sample.

## The research-to-final-draft gap

A common half-implementation: a pipeline has a research/planning stage correctly instructed to gather concrete numbers and examples ("no vague generalities, cite specifics"), but the *later* final-writing stage never explicitly says to carry those specifics into the finished text. The model can and will produce a well-grounded research doc and then a vague final draft from it, because "use what you found" was only ever stated as a requirement of the research step, not the writing step. Symptom: the research output looks great, the published output reads as generic copy with the evidence quietly dropped.

Fix: add one explicit line to the *final-drafting* prompt, not just the research prompt — "carry at least 1–2 concrete numbers/examples from the research into the finished copy" — and verify by generating once and checking whether a specific figure from the research doc actually shows up (verbatim or close to it) in the output. Any pipeline with a separate research/evidence-gathering step and a separate final-copy step (blog outline → draft, thread research → thread post, brief → ad copy) should be checked for this gap specifically — it's easy to build the evidence requirement into only one of the two stages and assume it propagates to the other.

## Applying this to a content-generation system prompt

If a project has an LLM generate any public-facing content (blog posts, SEO articles, video/upload descriptions, social posts, landing copy), amend that generation prompt to require:

1. An **answer-first opening** — the core claim or value prop as a direct, specific, objective first sentence. No throat-clearing.
2. **At least one concrete number or named source per major claim** — never a vague quantifier standing alone.
3. Natural sentences where the **brand/product name co-occurs with the problem/category it solves**, instead of generic keyword-stuffing instructions — this is what actually predicts citation, not keyword density.
4. If the project already has any kind of real-time search or fact-checking capability, consider offering a lightweight **benchmark-tracking feature**: run a fixed question set through it periodically and log whether/how the tracked content gets cited, building a trend instead of a one-off check (see §6).

Don't add an FAQ-schema generation step expecting it to be a GEO win on its own (§5) — it's fine as basic hygiene, not worth building a whole content format around.

## Relationship to `second-brain-wiki-rag`

That skill is about grounding an AI *assistant's own answers* in a verified personal knowledge base — building content that's trustworthy for retrieval. This skill is about writing content designed to *be* a source that *other people's* AI systems retrieve and cite. Same underlying principle (evidence-grounded content beats ungrounded content, in either direction) applied to opposite ends of the retrieval relationship. When a project touches both — e.g. it both maintains a knowledge base and publishes public content — apply the evidence-unit discipline (§3 here, and the verification gate in that skill) consistently across both.

## What not to do

- Don't chase GEO by stripping out SEO fundamentals — they compound, they don't compete.
- Don't repeat keywords hoping for density gains — the research found this has no effect; spend that effort on evidence units instead.
- Don't optimize only the official site and stop — the citation data says that's the 3% that matters least; budget real effort toward third-party listicle presence.
- Don't build a big FAQ section expecting it to be the GEO win — it measured as a small negative, not a boost.
- Don't declare victory (or defeat) from a single check — treat every measurement as one noisy sample of a probabilistic process.
