# Pattern 5 — Quota & zombie resilience

> Running out of tokens must degrade the system, never break it. And a hung worker is more often your own telemetry sidecar than the model.

This page is two hard-won lessons from things that *actually went wrong* overnight, and the patterns that fixed them for good.

---

## Part A — The zombie that wasn't the model

**Symptom:** workers would hang at 0% CPU and never finish. For three debugging sessions this was blamed on the model provider — *"MiniMax is crashing."* The owner kept insisting it wasn't the model; he'd been calling that provider by hand all day with no trouble.

**Root cause:** a **telemetry plugin** in the agent's config. It flushed usage data to a self-hosted observability backend on exit. When that backend returned a `502`, the plugin's exit-flush hung — waiting on a network call that would never resolve — and took the whole worker down with it. The model had already finished. The agent was a zombie held open by its own logging.

**The fix and the pattern:**

1. **Remove the offending plugin** — the worker now exits clean in ~3 seconds.
2. **Read telemetry at arm's length.** Don't put your observability *inside* the worker's exit path. Workers append usage to a local session file as they go; a *separate* watcher process ingests those files on its own schedule (see [diagram 3](architecture.md)). A telemetry backend can now be down for a week and not hang a single worker.

> **Lesson:** when a worker hangs, suspect your own observability/sidecar code *before* you blame the model API. Read the whole stack trace. The owner was right; the API was innocent.

---

## Part B — Token exhaustion must hold, not crash

**Symptom:** one provider's subscription quota ran out overnight (*"you've hit your session limit · resets 10am"*). Every lane on that provider, **and** the reviewer gate, exited with an error all night. The supervisor, with no concept of "out of tokens," saw the dead reviewer plus a stale acknowledgement and concluded *"stale — re-fire"* — forever. It re-fired the reviewer every tick until morning, spamming notifications, while nothing advanced because all that provider's work was quota-dead. The board looked frozen. The owner shut the whole thing down.

**Root cause:** zero resilience to a pool running out of tokens. The progression loop couldn't tell *"this failed because it's broken"* from *"this failed because there are no tokens until 10am."*

**The fix — three composing rules:**

### 1. Per-pool holds
A small module detects quota failures from a worker's log (`session limit`, `429`, `rate limit`, `overloaded`, `quota`, `resets <time>`) and parses the reset time. On detection it places a **hold** on *that pool only*, stored on disk with an auto-expiry at the reset time.

- The router skips a held pool (pattern 2) — so the *other* provider keeps working at full speed.
- The hold auto-expires when read after the reset time — work resumes on its own, no human needed.
- The hold fires **one** notification when placed, not one per tick.

### 2. Don't escalate a token limit
When a lane breaks, the break handler checks *why* first. If it's a quota failure: hold the pool, clear the failed lane's status so it re-dispatches after reset, and **return without spawning the fixer.** A fixer can't fix a token limit — it would just burn more tokens trying.

### 3. A circuit breaker on the auto-fixer
For *non-quota* failures the system can try to self-heal — but bounded. The auto-fixer gets **3 attempts** on a given break; after that it stops and raises **needs-owner** rather than thrashing forever. (The owner also has a master `autofix_enabled` toggle to turn self-healing off entirely.)

**The edge case that bites:** a stale *"resets 10am"* log read *after* 10am will re-parse to *tomorrow* 10am and wrongly re-hold. So before turning the system back on after an outage, wipe stale worker logs/acks so fresh dispatches start clean. (Normal dispatches truncate their own log, so this is only a turn-on-after-outage concern.)

---

## The rule, stated plainly

**Distinguish "broken" from "out of capacity," and treat them completely differently.** Broken → bounded self-heal, then a human. Out of capacity → hold that pool, keep the rest running, auto-resume on reset. And keep your observability *outside* the worker's critical path, so the thing that's supposed to tell you about failures can never *become* the failure.
