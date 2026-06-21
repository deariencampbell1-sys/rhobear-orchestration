# Proof: the architect actually wakes and writes

The most common failure mode of an "autonomous" agent system is the quiet lie — it *says* it's running, but nothing durable ever happens. We hold the system to a higher bar: **every state change has to leave a record a human can read later.**

Here is one real, unedited night. Each line was written **autonomously** by the architect agent the moment a wave's review closed and merged — no human prompted any of them. The agent is woken by a "courier" that injects a packet into its session; it wakes, reads what changed, verifies, and appends a line to its own wake-log.

```
22:33  review_closed (sweep ...222130)   — W2 partial merge advanced (#5 live-render, #6 override-serializer)
23:12  review_closed (sweep ...230140)   — w3 mode-b wire #8 MERGED (1/3 lanes)
23:14  wave_done w3-roundout             — #8/#9/#10 merged; verify: ci=ok build=ok core=ok engine=ok fidelity=ok
23:15  wave_done w3-roundout             — DUPLICATE re-delivery; already processed. Marked read; NO duplicate audit/flare.
00:47  review_closed gen-b-craft #15     — wave merged/advanced
01:44  review_closed gen-a-studios #16   — wave merged
02:12  review_closed (sweep ...014729)   — pattern-analysis #19 merged; wave advanced
03:29  review_closed (sweep ...032250)   — stash-expand #20 merged/advanced
```

## What to notice

1. **It runs unattended for hours.** Eight wakes across roughly five hours (22:33 → 03:29), each tied to a real merged pull request. No human was awake for most of it.

2. **It verifies before it claims.** The `23:14` line isn't "done!" — it's `ci=ok build=ok core=ok engine=ok fidelity=ok`. The agent's wake duty includes re-checking the work, not rubber-stamping a notification.

3. **It is idempotent — the `23:15` line is the important one.** The courier re-delivered the same `wave_done` packet a minute later (delivery is at-least-once). A naive agent would have written a second audit and fired a second alert. This one **recognized the duplicate, marked it read, and explicitly did *not* double-write.** That self-awareness is the difference between a demo and a system.

## Why this matters more than a screenshot

A dashboard animating is easy to fake and easy to misread. A growing, timestamped, merge-linked log that the agent maintains *about its own actions* is hard to fake and trivial to audit. When someone asks "is it actually working?", the answer is a `cat` of this file — not a vibe.

The mechanism behind these wakes is diagram 4 in [`architecture.md`](architecture.md): `architect-outbox → courier → claude -p --resume → the agent's thread`. The hard-won fixes that made it reliable (capturing the real session id, leaving a packet unread when a token limit ate the wake, refusing duplicate writes) are written up in [the async-architect pattern](04-async-architect.md).
