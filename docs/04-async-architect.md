# Pattern 4 — The async architect

> A human-facing agent that wakes on state-change, audits, and writes the record. It gates nothing. It keeps the human's mental model in sync.

Most "human in the loop" designs make the human (or a human-facing agent) a **gate** — work stops at each step and waits for sign-off. That's safe and unbearably slow; the human becomes the bottleneck the whole system was built to remove.

The async architect flips it: the human-facing agent is an **auditor that runs after the fact**, never a gate between steps. Waves advance deterministically (pattern 1) the moment they're ready. *Separately*, every state change wakes the architect to record what happened — value-added, not blocking.

## Why "async, not gate" is the whole point

```
WRONG (architect as gate):
  wave N done → wait for architect → architect approves → wave N+1 fires
               └─ human is the bottleneck ─┘

RIGHT (architect as async auditor):
  wave N done → wave N+1 fires immediately (deterministic)
              ↘ courier wakes architect → it audits wave N (in parallel)
```

The architect waking up to write about wave N has **zero** effect on wave N+1, which already fired. If the architect is asleep, busy, or out of tokens, the swarm keeps merging. The records just fill in when it wakes.

## The mechanism: a courier that injects packets

The architect is a long-lived agent session (in our case a CLI agent thread). You can't "call" it like a function — so a small **courier** process bridges the gap:

1. The supervisor writes a packet to an outbox directory whenever state changes: `wave_started`, `phase_advanced`, `wave_done`, `review_closed`, `wave_escalated`, etc.
2. The courier polls the outbox, and for each packet, **injects it into the architect's session** by resuming the thread with the packet as a new turn (`claude -p --resume <session-id>`).
3. The packet is wrapped in a sentinel marker so the architect never mistakes it for a human message — it knows this is a system event and runs its audit duty, not a chat reply.
4. On success, the courier marks the packet read (moves it to a `read/` subdir).

## Three hard-won reliability fixes

Getting this to actually work — not "work in a demo" — took three fixes that are easy to miss:

1. **Capture the real session id.** Trying to *impose* a session id on the agent didn't persist. The fix: let the agent mint its own id, then capture it from the structured (`--output-format json`) output of the first run and store it for future resumes.

2. **Don't let a token limit eat a wake.** The resume command returns success (`rc=0`) even when the only thing it produced was *"you've hit your usage limit."* A naive courier marks that packet read — silently dropping the wake forever. The fix: after a successful run, scan the result for quota markers; if found, **leave the packet unread** so it retries after the limit resets.

3. **Be idempotent about delivery.** Delivery is at-least-once, so the same packet can arrive twice. The architect's duty includes recognizing a duplicate and **refusing to double-write** (see the `23:15` line in [the proof](proof.md)).

## The architect's duty on wake

When woken, the architect does a fixed sequence — and **proof-of-wake comes first**, because the system is headless and nothing animates a screen:

1. **Write proof immediately** — append a timestamped line to its own wake-log and fire a notification, *before* anything else. This is how a human knows it woke at all.
2. **Verify** — re-check the work that closed (CI, build, the actual artifact), don't trust the packet.
3. **Audit** — write a longer record of what merged and whether it's sound.
4. **Commit** — persist the records.

The architect produces no human-facing chat reply to a courier packet. Its output is the durable record, full stop.

## The rule, stated plainly

**Make your human-facing agent an asynchronous auditor, never a synchronous gate.** Let the deterministic core advance on its own; wake the architect after each state change to leave proof and keep the human oriented. And since the wake mechanism is headless, the architect's *first* act on waking must be to leave evidence it woke — the screen will never show you.
