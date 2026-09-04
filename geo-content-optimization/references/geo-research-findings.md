# GEO research findings — full detail

Source: lecture material citing two academic papers, cross-checked directly against the papers' own abstracts/citing context before being trusted for this skill.

## Paper 1 — GEO: Generative Engine Optimization

**Aggarwal, Murahari, et al. — IIT Delhi, Princeton, Georgia Tech, Allen AI. ACM SIGKDD 2024 (a top-tier venue). arXiv:2311.09735.**

The first paper to academically define and measure "Generative Engine Optimization" as its own discipline, separate from SEO.

### Method
1. **GEO-bench**: ~10,000 real user questions collected from 9 sources (Google search logs, ELI-5, Perplexity queries, etc.) — not synthetic questions.
2. **9 content techniques**, tested as a controlled black-box experiment — the same underlying content rewritten with one technique added at a time: statistics, quotations, citing sources, authoritativeness, fluency/writing-quality, plain-language explanation, unique terminology, technical jargon, keyword repetition.
3. **Two measurements**: (a) how much of the AI's answer is drawn from a given source, and how early/prominently it's cited (position + volume), (b) an AI-assigned "influence score" for how much that citation shaped the answer.

### Headline results
- Up to **+40% visibility** improvement from the winning techniques (varies by domain — see below).
- **Keyword repetition: ~0% effect.** The single most load-bearing old-SEO tactic does essentially nothing for GEO.
- **Smaller / lower-ranked sites benefit disproportionately more than large, already-dominant brands.** This is the paper's most quoted finding and the reason this skill treats GEO as favorable terrain for solo operators rather than another arena where big budgets win by default.

### Domain-specific effectiveness
The paper found technique effectiveness is **not uniform across domains** — this matters because a generic "always add statistics" instruction will underperform a domain-matched one:
- **Data/fact-heavy or opinion-adjacent domains** (legal, public policy, comparative "which is better" content): statistics and cited sources are the strongest lever.
- **Authority-driven domains** (humanities, social commentary, history/context pieces): expert quotations outperform raw statistics.
- Match the evidence type offered to the domain the content sits in, rather than applying one formula everywhere.

### The FAQ/schema myth-bust
A common assumption carried over from old SEO practice is that FAQ-formatted content and schema markup are a GEO cheat code. The measured reality:
- **Basic schema markup**: a real but modest effect, roughly **+13%** — worth doing as hygiene, not worth treating as a strategy.
- **Q&A/FAQ-formatted pages specifically**: measured **-5.7%** — slightly *lower* citation rate than non-FAQ content covering the same material. Don't restructure content into FAQ format expecting a GEO win from the format itself.

## Paper 2 — AI citation source analysis (149,912 citations)

Referenced in the lecture as an analysis of what AI answer engines actually cite in practice, at scale (arXiv:2606.20065 per the lecture's citation).

### Headline results
- **Only 2.9%** of AI-answer citations point to a brand's own official website.
- The remaining **~97%** are third-party pages: review blogs, comparison/roundup articles, competitor-adjacent content, community discussion.
- Of those third-party citations, **35.7%** came specifically from ranking/listicle-format content ("Top 10 X", "Best Y for Z").

### Practical implication
Optimizing only your own official site — the classic SEO instinct — is optimizing the minority of the surface area that actually gets cited. A GEO strategy that stops at on-page changes is incomplete by construction; it has to include getting into (or creating) the third-party ranking content that AI models actually pull from.

## A worked "mention vs. backlink" data point

The lecture cites research finding that AI models weight **brand-mention co-occurrence** (the brand/product name appearing in running text next to the category or problem it solves) roughly **3x more predictive of citation** than a hyperlink pointing at the brand. This reframes what "getting your name out there" should optimize for: not backlink count, but how often your name shows up *written into sentences about the problem you solve*, across as many pieces of content (yours and others') as possible — link or no link.

## The diagnostic prompt pattern

The lecture provides a template prompt (translated/adapted, not copied verbatim) for auditing an existing page or service for both SEO and GEO gaps. The key instruction embedded in it — worth preserving in any adaptation — is telling the auditing LLM explicitly that **the majority of AI citations come from third-party pages**, so its output should include a third-party/listicle outreach plan alongside on-page fixes, not just on-page fixes. An audit that only proposes changes to the site itself is solving the smaller half of the problem.

## Cross-reference: Medical Graph RAG (arXiv:2408.04187)

A separate paper, verified directly (not from the lecture) while researching this skill's source material. Different domain (safe medical LLM answers) but the same underlying principle from the retrieval-architecture side: their "Triple Graph" construction connects generated content to trusted sources and standardized terminology, and that source-linkage is what the paper credits for outperforming ungrounded baselines on medical QA benchmarks. Useful as independent confirmation that "ground claims in traceable evidence" is the same winning move whether you're the one retrieving (a RAG system reading a knowledge base) or the one being retrieved (a public page an AI cites) — see `second-brain-wiki-rag` for the retrieval-side version of this same principle.
