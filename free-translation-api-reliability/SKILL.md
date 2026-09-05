---
name: free-translation-api-reliability
description: How to call free/unofficial translation wrappers (e.g. Python's `deep_translator` GoogleTranslator, or any library that scrapes a web translate endpoint rather than using an authenticated paid API) reliably at real volume — sequential batching with deliberate pacing instead of parallel threads, and explicit failure flagging instead of silent fallback. Use this whenever a feature calls Google Translate (or a similar free-tier scraping translator) for more than a handful of strings — subtitle translation, bulk content localization, multi-language metadata — especially if a `ThreadPoolExecutor` or any concurrency is involved. Two real, reproduced bugs are documented here, not theory.
---

# Free Translation API Reliability

## Why this exists

`deep_translator`'s `GoogleTranslator` (and similar libraries) don't call an authenticated Google Cloud API — they scrape Google Translate's web interface, no API key involved. That makes them free and easy to reach for, but it also means they inherit two failure modes that an official paid API wouldn't have, both reproduced live in a real project on the same day:

1. **Sharing one translator instance across threads corrupts results.** Calling `translator.translate()` on the *same* `GoogleTranslator` object from multiple threads concurrently doesn't raise an obvious error most of the time — it silently returns the *wrong* line's translation for some inputs (line 1 gets line 2's result) or occasionally raises `TranslationNotFound`. A 2-line parallel test reproduced this every time. Nothing in the stack trace points at concurrency; it just looks like a bad translation.
2. **Parallel *load* itself (even with a fresh instance per thread) triggers Google-side failures.** With the instance-sharing bug fixed, hammering the endpoint with `ThreadPoolExecutor(max_workers=8)` still produced a ~25% failure rate (`TranslationNotFound`) on a 4-line burst, and a heavier burst (dozens of calls within a couple of minutes, from live debugging) triggered what looks like an IP-level temporary block — every subsequent call, including single sequential ones, failed for several minutes afterward. This is exactly the kind of thing that's invisible in a quick manual test (a user clicking through one video, one language at a time, with natural pauses between clicks) and only shows up under real automated volume (a script translating 80-120 subtitle lines, or an agent stress-testing the endpoint while debugging).

## The proven fix: sequential batches with deliberate pacing, not parallel calls

A reference implementation in the same environment (a separate agent's `build_global_subtitles.py`, already used successfully on real production content at real volume — 16 songs' worth of lyrics × 5 languages) uses this shape, and re-doing it this way immediately fixed both failure modes:

```python
from deep_translator import GoogleTranslator
import time

def translate_batch_paced(texts: list[str], source: str, target: str) -> list[dict]:
    translator = GoogleTranslator(source=source, target=target)
    out = [None] * len(texts)
    chunk_size = 15  # not 1, not "as many as fit" — 15 is the size validated in production
    for i in range(0, len(texts), chunk_size):
        chunk = texts[i:i + chunk_size]
        idxs = list(range(i, i + len(chunk)))
        try:
            translated = translator.translate_batch(chunk)  # ONE call for the whole chunk
            if not translated or len(translated) != len(chunk):
                raise ValueError("batch size mismatch")
            for j, t in zip(idxs, translated):
                out[j] = {"text": (t or texts[j]).strip(), "failed": not bool(t and t.strip())}
        except Exception:
            # Batch failed — retry only this chunk, one line at a time, not the whole request.
            for j, text in zip(idxs, chunk):
                ok = False
                for _ in range(2):
                    try:
                        t = GoogleTranslator(source=source, target=target).translate(text)
                    except Exception:
                        t = None
                    if t and t.strip():
                        out[j] = {"text": t.strip(), "failed": False}
                        ok = True
                        break
                    time.sleep(0.2)
                if not ok:
                    out[j] = {"text": text, "failed": True}
        time.sleep(0.5)  # deliberate pause between chunks — this is what actually prevents blocking
    return out
```

Key decisions, each backed by the reproduced failures above:
- **No `ThreadPoolExecutor` at all.** Sequential, on purpose. The whole point is trading speed for not tripping the rate limiter — for an 80-120 line subtitle file this costs maybe 5-10 extra seconds total (a handful of `time.sleep(0.5)` calls), which is nothing next to the alternative of a multi-minute IP block that fails *everything*, including unrelated calls, for a while afterward.
- **`translate_batch()` over one call per line.** One HTTP round-trip per 15 lines instead of 15, which is both faster and gentler on the endpoint than either "1 huge request" or "15 individual requests."
- **A fresh `GoogleTranslator(...)` instance for every call** (batch or per-line retry) — never reuse one instance across what could be concurrent or even just many sequential calls if there's any chance something else touches it. Construction is cheap; sharing isn't safe.
- **Chunk-level fallback, not all-or-nothing.** If a 15-line batch fails, only that chunk drops to slower per-line retries — the other chunks that already succeeded aren't discarded or redone.

## Don't silently paper over a failed line

Even with the pacing above, occasional individual-line failures still happen after retries — this method is more reliable, not bulletproof. The tempting shortcut is `except: return original_text` so the pipeline "always succeeds." **Don't** — if the output is going into a different language a human reviewer can't read (translated subtitles, localized metadata), a silently-untranslated line sitting in the middle of a foreign-language block is invisible to exactly the person who's supposed to be catching problems before they ship. Return an explicit `failed: true`/`translation_failed: true` flag alongside the text and surface it in the UI (a warning icon, a highlighted row) instead of letting a failure look like a successful translation.

## When you hit failures that retries don't fix

If every call — including a single bare sequential one, isolated from any pipeline — starts failing, that's not a code bug to keep chasing; it's very likely a temporary IP-level block from the free endpoint, usually self-inflicted by recent heavy/rapid testing (including your own debugging). Retrying harder or lowering concurrency further won't fix an active block. Verify with one minimal, isolated call outside any batching/retry logic; if that alone fails, the fix is to wait (minutes to hours) rather than keep modifying the calling code. Don't mistake "the block is currently active" for "the fix didn't work" — re-test the exact same code again later before concluding it's broken.

## When this isn't enough

If a feature's real volume (many languages × many lines, run frequently) makes even paced sequential batching risk hitting blocks often enough to matter, the real fix is switching to an authenticated official API (e.g. Google Cloud Translation API) rather than continuing to tune the free scraper — that trades a small amount of setup (enabling the API + billing on an existing Google Cloud project, which is a separate product from an unrelated key like a YouTube Data API key even in the same project) for a rate limit that's actually documented and paid-tier reliable, with a generous free monthly quota that covers most single-project workloads at no cost.
