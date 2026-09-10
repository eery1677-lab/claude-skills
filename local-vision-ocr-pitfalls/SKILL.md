---
name: local-vision-ocr-pitfalls
description: A checklist of specific, hard-won findings — including two plausible-sounding accuracy techniques that were tested and DISPROVEN — for improving accuracy when a local/small vision-language model (via LM Studio, Ollama, or similar) reads photos of receipts, invoices, or other structured documents and extracts fields into JSON. Use this whenever building, debugging, or trying to improve accuracy on any feature where a local vision model does OCR or structured field extraction from an image — receipt/invoice scanning, ID/document parsing, form digitization, or any "look at this photo and fill in these fields" pipeline. Load this BEFORE reaching for an intuitive-seeming fix like "upscale the image," "use a bigger model," or "split reading and structuring into two steps" — this skill shows real A/B-tested evidence that two of those three made things worse, not better, and explains what actually worked instead. Also load it when a local vision model via LM Studio seems inexplicably slow or times out, since that's frequently a context-length misconfiguration, not a hardware or model problem.
metadata:
  type: pitfalls-checklist
---

# Local Vision Model — Document/Receipt OCR Pitfalls

## Why this exists

"Make the local vision model read this document more accurately" invites a handful of
intuitive-seeming fixes — send a bigger image, use a bigger model, break the task into
steps. In one real project (a Korean vendor-receipt ledger app, LM Studio + Gemma-4 /
Qwen3-VL), each of these was tested head-to-head against real saved receipt photos with
known, human-corrected ground-truth values already in the app's own database — not
reasoned about, actually run and measured. Two of the three intuitive fixes made results
*worse*. This skill exists so the next project doesn't have to re-discover that the hard
way, and so the one prompt-level fix and one operational gotcha that *did* matter don't
get missed.

**The methodology matters as much as the findings** — see #6.

## 1. A labeled field can lose to a bigger, bolder distractor elsewhere on the page

Asking a vision model to extract a field from a specific labeled box (e.g. "vendor name
from the 공급자/supplier box") isn't enough on its own — if the document also has a large,
bold logo or brand name printed elsewhere (a product brand, not the company name), a small
model will often grab the visually prominent text instead of the correctly labeled field,
even when the prompt already says the label takes priority "if present." A generic
label-first instruction reads to the model as a soft preference, not a hard rule, and a
big bold logo is a strong competing signal.

**Fix**: Spell out the priority as an explicit ordered list, not a single sentence — (1)
labeled box wins, always, when present; (2) name the *specific kind* of distractor to
ignore (e.g. "a large logo or brand name — that's a product brand, not the company name,
ignore it when a labeled field exists"); (3) only fall back to unlabeled prominent text
when no labeled field exists at all. Include one concrete negative example naming the
actual confusion pattern ("don't confuse the shop's own brand logo for the supplier's
name") rather than a generic "don't get distracted" instruction — the fix that worked was
specific enough to name the failure mode itself, not just assert a priority order.

## 2. Upscaling the input image: tested, DISPROVEN — don't reach for this by default

Intuition says a low-resolution photo with small text should get *more* legible after
upscaling before sending it to the model. Tested directly: took a real receipt the model
had partially misread (right box, one character wrong), upscaled it 1.5x with a good
resampling filter, re-ran the identical prompt against the identical model. Result: the
model did *worse* — it reverted to grabbing the wrong logo entirely, and misread a number
it had previously gotten right at the original resolution.

**Why this probably happens**: many local/edge-oriented vision-language models resize
input images to a small fixed internal resolution before their vision encoder ever sees
them, regardless of how large the input is — so upscaling doesn't hand the model more real
information, it just changes how the image looks after the model's own downsampling, and
an interpolated/softened version of small text can read worse to the encoder than a
naturally lower-res but sharp original.

**Fix**: Don't assume higher input resolution helps a fixed-resolution local vision
model. If pursuing this, A/B test it against the exact case that's currently failing
before shipping it — don't ship it on intuition alone. If it needs to help, a model with a
genuinely dynamic/tiled vision encoder (the model card will say so) is a more promising
lever than a bigger input image sent to the same model.

## 3. A bigger model in the same family: tested, DISPROVEN — measure, don't assume

Swapping a 7.5B "efficient slice" model for the 12B full model in the *same family*
(same training recipe, just bigger) was expected to be a straightforward accuracy win.
Measured result: 2-3x slower, and *worse* on some numeric fields (misread a subtotal the
smaller model had read correctly), no improvement on the failure mode it was meant to
fix. Scaling up within a family is not a reliable accuracy lever for this class of
task — it has to be tested against the actual failing cases, not assumed from parameter
count.

**Fix**: Before recommending or shipping a same-family model upgrade for accuracy,
run it against the specific cases that are currently failing and compare real outputs,
not benchmarks from elsewhere. If it's not clearly better on the actual failing cases, the
switch cost (VRAM, latency) isn't worth paying.

## 4. Splitting "read + structure" into two stages: tested, DISPROVEN for this shape — but a narrower decomposition DID work

The generally-true principle that small models do better with decomposed tasks does
**not** automatically apply to every way a task could be split. Tested: splitting a
single "look at this photo and fill in this JSON" call into stage 1 ("transcribe
everything you see, no structure yet") → stage 2 ("now structure that transcript into
JSON, looking at the photo again"). Result: *worse* on both test cases — stage 1
sometimes skipped an entire labeled section of the document in its transcript, and stage
2, working from that incomplete transcript, then fabricated a vendor name that didn't
appear anywhere in the source rather than reporting it missing. Also ~2.5x slower than
the single-shot call.

This is not a blanket argument against decomposition — in the *same* project, a
different two-stage split worked well: stage 1 = full extraction, stage 2 = "look at the
same photo again and re-check only these specific numeric fields, don't touch anything
else." That narrow, closed-scope re-verification pass genuinely caught digit-level
misreads the first pass made.

**The pattern**: decomposition helps when the second stage has a **narrow, well-defined
job with something concrete to check against** (the same image, a specific field list).
It can actively hurt when the *first* stage is itself an open-ended perception task,
because an incomplete or lossy first stage doesn't just lose information — it actively
misleads the second stage into confidently filling the gap with a fabrication instead of
correctly reporting "not found." An incomplete transcript looks to stage 2 like ground
truth, not like a hint to look harder.

**Fix**: Before splitting a vision task into stages, ask what the second stage is being
asked to do. "Re-verify this specific narrow thing" is a safe decomposition. "Read the
document" (again, differently) is not — if stage 1 can silently drop information, stage 2
will build on the gap rather than notice it. If open-ended reading needs to improve,
prompt-level fixes (see #1) or model choice are more promising levers than adding a
second read-the-whole-thing pass.

## 5. LM Studio: an oversized auto context-length silently makes a model "unusably slow"

A model loaded through LM Studio without an explicit context-length setting can get
assigned a huge default (e.g. 262,144 tokens) that overflows available VRAM and forces
slow CPU fallback — every request then takes minutes instead of seconds, sometimes
timing out entirely. The symptom ("this model doesn't work, it's way too slow" or "it
just times out") reads like a broken model, a bad prompt, or a hardware limit, and is
easy to misdiagnose as any of those.

**Fix**: Run `lms ps` and check the `CONTEXT` column whenever a locally-loaded model is
inexplicably slow — an absurdly large number relative to what the task actually needs
(a single-image analysis rarely needs more than a few thousand tokens) is the signature
of this bug. Reload with an explicit, task-appropriate context length and full GPU
offload: `lms load <model> --context-length 8192 --gpu max -y`. In one real case this
took a model from timing out past 180 seconds to completing in under a minute, using
6.66 GiB of a 12 GiB GPU instead of spilling to system RAM.

## 6. Entity-resolution correction: fuzzy-match against known-good records, always confirm before applying

When a vision model extracts an entity name (vendor, company, person) that's supposed to
correspond to one of a small set of already-known, previously-verified records the
application already stores, don't just trust the fresh OCR read at face value — cross-check
it against the known set and offer a correction.

**Priority order**: (1) an exact match on a strong structured identifier if the document
has one (e.g. a business registration number, an account number) — this is far more
reliable than name text, since a model can misread a company name while still reading a
printed ID number correctly; (2) edit-distance (Levenshtein) name similarity as a
fallback. **Calibrate the threshold using absolute edit distance for short strings, not
just a similarity ratio** — a single-character difference in a 3-character name (e.g.
"엄티컴" vs "옵티컴") drops a ratio-based similarity score below most reasonable
thresholds (1 - 1/3 ≈ 0.67), even though it's obviously the same entity with one
misread character. Accept a match if `distance <= 1` regardless of ratio (for names long
enough that this doesn't produce nonsense matches on very short strings), OR the ratio
clears a normal threshold (~0.7) for longer names.

**Never auto-apply the correction silently.** Surface it as a suggestion the human
confirms with one click ("AI read 'X', which is very close to your registered vendor
'Y' — use that instead?") rather than swapping it in automatically. A wrong auto-merge of
two genuinely different entities is a real data-integrity mistake that's expensive to
untangle later; asking costs one click and is never wrong to ask.

## 7. Methodology: test against real ground truth, don't reason about what "should" work

Every finding above — including the two disproven ones — came from running the actual
candidate change against real saved photos that already had known, human-verified correct
answers on file (a receipt-ledger app's own transaction history, in this case), and
diffing the model's output field-by-field against that ground truth. Two out of three
plausible, common-sense accuracy techniques (upscale the image, use a bigger model) failed
that test outright. If real examples with known-correct answers exist anywhere in the
project already (a database of previously-reviewed records, a labeled test set), use them
to A/B test an accuracy change before shipping it — intuition about what "should" help a
vision model is unreliable enough in practice that it isn't a substitute for measuring
against 2-3 real hard cases.

## What not to do

- Don't rely on a single soft "prefer the labeled field" sentence when a document also has
  a prominent logo/brand competing for attention — name the specific distractor pattern
  and give a negative example.
- Don't upscale images to "help" a local vision model without testing it against a case
  that's actually currently failing — it regressed real results in this project.
- Don't assume a bigger model in the same family is automatically more accurate — the
  12B version was slower *and* less accurate on real cases here.
- Don't split an open-ended "read this document" task into a transcribe-then-structure
  pipeline expecting it to help by default — an incomplete first pass teaches the second
  pass to fabricate, not to admit uncertainty. Reserve decomposition for narrow,
  closed-scope re-verification against something concrete.
- Don't trust a local vision model's read of an entity name at face value when the app
  already has a small set of verified records to check it against — but don't silently
  auto-correct either; always ask first.
- Don't debug "the local model is too slow" by assuming it's the model or the hardware
  before checking `lms ps` for an oversized context length.
- Don't ship an accuracy fix on intuition when real examples with known-correct answers
  are available to test against instead.
