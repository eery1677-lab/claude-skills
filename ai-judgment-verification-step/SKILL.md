---
name: ai-judgment-verification-step
description: How to add a lightweight self-verification (fact-check) step after any local/small-LLM analysis, diagnosis, or report output that cites real computed numbers or facts. Use this whenever building or reviewing a feature where a local model (LM Studio, Ollama, or similar) writes analysis/diagnosis/report text derived from real data — analytics summaries, health-check diagnoses, staged multi-step reports, channel/account audits, anything where the model is asked to "cite the evidence." Local and small models reliably fabricate or garble specific numbers when asked to justify a claim, even when the real numbers were right there in the prompt — this is a proven, already-shipped pattern (not theory) for catching that class of error cheaply, with a working reference implementation. Trigger proactively — don't wait for the user to ask for "verification" or "fact-checking" by name.
---

# AI Judgment Verification Step

## The problem this solves

A local/small model asked to write an analysis that cites real numbers ("**근거**: 조회수 5,300회") will sometimes invent a plausible-looking number instead of the real one — even when the real number was right there in its own prompt a few lines above. This gets worse as the prompt grows (more context, more stages already generated) and worse when the model is also juggling tone, structure, and strategic judgment at the same time as arithmetic. It's silent: the output reads fine, the number is just wrong.

The fix is not "make the prompt clearer" — it's to stop asking the same model call to do judgment *and* fact-checking at once. Split them into two calls with two different jobs.

## The pattern

1. **Code computes or fetches the real ground-truth data first** — never let the model be the source of truth for numbers it should be quoting.
2. **One model call writes the judgment/analysis/diagnosis**, citing whatever numbers it wants, at normal temperature.
3. **A second, separate model call — low temperature (~0.2), a narrow system prompt** — is given only the ground-truth data and the draft, and told to do exactly one job: check the draft's cited numbers/facts against the ground truth, correct any that are wrong or fabricated, and return the draft **completely unchanged** if everything already checks out. No commentary, no re-judging the strategy, no restructuring.
4. **A length safety net**: if the verification call's output comes back suspiciously short relative to the original (a good threshold is <40% of the draft's length), that's a sign the call broke or truncated — discard it and keep the original draft rather than trusting a corrupted "fix." Same fallback on any network/API failure.
5. **This is decided by code, not by the model** — the frontend/orchestration layer always calls the verify step for stages that cite ground-truth numbers, and always applies the length/failure fallback. The model is never asked "should you double check this."

Why the length safety net matters: a verification call that times out, gets rate-limited, or hits a local model that decides to summarize instead of return-verbatim will produce a plausible-looking but truncated or hallucinated "corrected" version. Silently swapping in a broken 200-character reply where a 2,000-character report used to be is worse than skipping verification entirely — always compare lengths before trusting the replacement.

## Where to apply it (and where not to)

Apply it to any stage that **cites specific numbers/facts from real data you already have**: analytics summaries, health-check diagnoses, "here's what's wrong and why" sections, any "**근거**:"-style evidence line.

Skip it for stages that are pure creative/strategic judgment with nothing to fact-check against (e.g. a brainstorm of five new content ideas, a tone/style pass) — there's no ground truth to verify against, so a verify call there just burns a model call for nothing. In a real staged pipeline (e.g. a 4-stage SWOT → gap-analysis → pattern-insight → roadmap flow), it's normal and correct to verify stage 1 and stage 4 (which cite hard numbers) and skip stages 2–3 (which lean on the earlier stages' already-verified conclusions, or compare qualitatively rather than citing fresh figures).

## Shared system prompt (reusable across every pipeline in a project)

Don't write a bespoke fact-check prompt per feature — one shared prompt, parameterized only by which ground-truth text block gets passed in, works for a blog diagnosis, a channel analysis, an Instagram diagnosis, or anything else in the same project:

```python
def _fact_check_system_prompt() -> str:
    return """당신은 팩트체커입니다. AI가 작성한 리포트 초안이 [원본 데이터]에 나온 실제 숫자·사실과
일치하는지만 확인합니다. 리포트의 전략적 판단이나 문체에는 관여하지 않고, 오직 숫자·통계·사실 인용이
원본 데이터와 다른 경우만 찾아서 고칩니다.

## 작업 방식
1. 초안에서 숫자나 구체적 사실을 인용한 부분(퍼센트, 일수, 요일, 개수, 채널/영상명 등)을 하나씩
   원본 데이터와 대조하세요.
2. 원본 데이터에 없는 숫자를 지어냈거나, 원본과 다른 숫자를 썼거나, 존재하지 않는 사실을 언급했다면
   원본 데이터에 맞게 정정하세요.
3. 모두 일치한다면 초안을 단 한 글자도 바꾸지 말고 그대로 반환하세요.
4. 절대 "검증 결과:", "확인했습니다" 같은 코멘트를 앞뒤에 추가하지 마세요. 최종 텍스트만
   (수정본이든 원본 그대로든) 그대로 출력하세요. 구조(제목, 표 형식)는 원본 그대로 유지하세요."""
```

Each pipeline's `*/verify/prep` endpoint just builds the `user` message (ground-truth block + the draft) and returns `{"system": _fact_check_system_prompt(), "user": user}` for the frontend to stream — same shape as every other prep/stream call in the app, so no new plumbing is needed on the frontend beyond the one shared helper below.

## Shared frontend helper (call this after every stage that needs verification)

```javascript
// endpoint: the */verify/prep route. extraBody: whatever identifiers that route needs
// (account_key, blog_url, etc) besides draft_text. draftText: the stage's raw output.
async function verifyAgainstGroundTruth(endpoint, extraBody, draftText) {
  if (!draftText || !draftText.trim()) return draftText;
  try {
    const prepRes = await fetch(endpoint, {
      method: 'POST', headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ draft_text: draftText, ...extraBody }),
    });
    if (!prepRes.ok) return draftText;
    const prep = await prepRes.json();
    const verified = await streamStudioChat({
      messages: [{ role: 'system', content: prep.system }, { role: 'user', content: prep.user }],
      temperature: 0.2, maxTokens: 4096,
    });
    if (verified && verified.trim().length >= draftText.length * 0.4) return verified.trim();
    return draftText;
  } catch (e) {
    return draftText;
  }
}
```

Call it right after a stage's draft finishes generating, before rendering/saving it:

```javascript
stageState.step1 = await verifyAgainstGroundTruth(
  '/api/my-channels/analysis/step4/verify/prep',   // one shared endpoint can serve multiple stages
  { my_section: ctx.my_section, analytics_section: ctx.analytics_section },
  stageState.step1
);
```

Note the endpoint can be shared across multiple stages if they cite the same ground-truth data shape (in the reference implementation, the exact same `step4/verify/prep` route serves both stage 1 and stage 4 of the pipeline, since both only ever cite the channel's own data/analytics section — no need to build a separate verify endpoint per stage unless the ground-truth shape genuinely differs).

## What "ground truth" means for the verify prompt

The `user` message for the verify call should contain the *same real data block* that was already given to the original drafting call — not a re-fetch, not a summary of it, the literal same text. For a diagnosis feature this is usually a short, code-computed metrics block:

```python
ground_truth = f"""## 원본 데이터 (코드로 직접 집계한 스캔 결과, 최근 {a['post_count_scanned']}개 기준)
- 팔로워: {a['followers_count']}명 · 평균 좋아요 {a['avg_likes']}개
- 평균 게시 간격 {a['avg_posting_interval_days']}일"""
```

Keep it to numbers computed by code — never let this block itself be something the model wrote, or you've just made the fox guard the henhouse.

## Real track record

This exact pattern (shared system prompt + shared frontend helper + 40%-length fallback) was built and shipped in one project across three independent pipelines in a single session — a 4-stage channel-analysis report, a blog-health diagnosis, and an Instagram-account diagnosis — verified working via live end-to-end runs against a real local model each time (confirmed via server logs showing the verify endpoint being hit, and by inspecting the before/after text for consistency with the real underlying numbers). It's cheap (one extra short model call per verified stage, at a temperature tuned for precision not creativity) and it materially reduces the "AI cited a number that isn't in the data" failure mode that's otherwise very easy to miss on a casual read of the output.

## Theoretical grounding

This is a lightweight, single-project-scale instance of the broader "self-verification"/"self-refine" pattern from the agent literature — see Reflexion (Shinn et al.) and Chain-of-Verification / CoVe (Dhuliawala et al.) for the general idea of a separate verification pass catching errors a single generation pass misses. The value here isn't the theory, though — it's the concrete, already-proven implementation shape above; reach for the papers only if you need to justify or extend the approach beyond fact-checking numeric claims (e.g. into multi-step verification chains).
