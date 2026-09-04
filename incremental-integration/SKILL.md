---
name: incremental-integration
description: How to sequence work when a project has several features that need to talk to each other — build-and-connect-immediately vs. build-everything-separately-then-merge-at-the-end. Use this whenever planning multi-feature development, deciding project architecture/order, or when the user asks how to structure a build with several interconnected pieces (e.g. "이 순서로 개발하면 될까", "여러 기능을 어떻게 연결하지", "따로 만들고 나중에 합칠까"). Also useful when reviewing why an integration between two existing features silently broke.
---

# Incremental Integration

## The core call

Default to **connecting each piece to the rest of the system as soon as it's built**, not building every piece in isolation and wiring them together in one late "integration phase." This is true for almost all multi-feature software work — the exception is narrow and covered below.

## Why — not just a preference

- **Bugs are cheap when found early, expensive when found late.** Right after wiring feature B into the system, if something's broken, the blame surface is B and its one new connection. If ten features get built in isolation and connected on the same day, a bug could be in any of them, or in how two of them interact — the search space is combinatorial.
- **The same bug class repeats invisibly.** If a connection pattern is subtly wrong (a scoping mistake, a missing field, an assumption that doesn't hold), building the next five features the same way copies the mistake five times before anyone notices. Catching it after feature one means it never gets copied.
- **"Integration day" is a myth that always runs long.** A separate, late integration phase relies on every isolated piece being correct on first contact with the real system — which is never true in practice. Continuous integration turns one scary unknown-sized task into many small known-sized ones.
- **You always have something real to show and test.** At every step there's a working, connected system to click through, not a pile of parts with a promise they'll fit.

## The LEGO metaphor, done right

"Build pieces independently, snap together later" sounds like LEGO but isn't how LEGO actually works. Real bricks fit because the **stud-and-tube connector spec was fixed before any brick was molded** — every piece is designed against that one shared, stable interface from day one. The LEGO-like move in software is the same: define the shared connection point first (a registry, an event bus, a plugin-registration function, a well-known API contract, a shared schema), then have every new feature plug into *that fixed interface* the moment it's built, and verify the connection immediately. That's integrating early — not integrating late.

Concrete shape of that pattern: a project keeps a small registry object at the top level (`registerX(key, providerFn)` / `collectAllX()`); when the chatbot, the dashboard, or any consumer feature is built, it just calls `collectAllX()` and does not need to know what producers exist. Every new producer feature registers itself and is instantly visible everywhere else — no later "now let's go wire feature 7 into feature 2" step ever needs to happen, because the wiring already happened as feature 7 was built.

## Decision rule: when to integrate immediately vs. when it's fine to defer

**Integrate immediately (as you build) when:**
- The new piece reads or writes state that another feature also needs.
- Its correctness can only really be judged by seeing it work *inside* the live system (not just in isolation).
- Other already-built features would benefit from knowing about it right away.

**Deferring integration (building it standalone first, connecting later) is fine when:**
- The piece is genuinely self-contained — a pure utility, an isolated asset generator, a script with one narrow and already-stable interface to the rest of the system.
- The eventual connection point is a single well-understood call, not a design question — connecting it later costs the same as connecting it now, so there's no real advantage to doing it early.

If a piece doesn't clearly meet the "defer" bar, treat immediate integration as the default, not something to schedule for later.

## Practical checklist

For any new feature that touches shared state or other features:
1. Build the minimum working version.
2. Wire it into the existing shared interface (or create that interface now if this is the first piece that needs it) — don't invent a bespoke one-off connection.
3. Run one real end-to-end check *through that interface* (not just a unit-level check of the new piece alone) before moving on.
4. Only then start the next independent unit.

## Anti-pattern to watch for

If a plan includes a step literally called "integration phase," "connect everything," or "wire it all up at the end" as a separate late milestone, that's the warning sign. The fix is to make "connected and verified with the rest of the system" part of each feature's own definition of done, not a project phase that comes after all features are "complete."
