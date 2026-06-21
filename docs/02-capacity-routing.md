# Pattern 2 — Capacity-aware model routing

> When one provider is full, the work doesn't wait — it routes to the other. When one is out of tokens, that pool holds; the other keeps running.

This is the pattern that makes a multi-agent system feel *alive* instead of brittle. You watch the board and see a lane that wanted MiniMax quietly come up on Claude instead, because MiniMax was at capacity — and nothing stalled.

## The setup: two pools, counted separately

Each provider gets a **pool** with a fixed concurrency cap (ours is 5 each). A worker occupies a slot from the moment it's dispatched until it writes its terminal status. The supervisor counts live workers per pool every tick by reading status files — not by trusting an in-memory counter that drifts.

```
MiniMax pool: ███░░  3/5
Claude  pool: ████░  4/5
```

A subtle but important detail: **only workers in a START state count against a pool.** Idle shells, relay processes, and finished sessions that happen to be lying around must not be miscounted — otherwise you get phantom occupancy and the router refuses to dispatch into a pool that's actually free.

## Routing decision (pure code, no model)

When a lane is ready to dispatch, the dispatcher picks a path:

```
def dispatch_lane(lane):
    model = lane.model
    if control.force_minimax:          # owner override toggle
        model = "MiniMax-M3"

    pool = "claude" if model.startswith("claude") else "minimax"

    if is_pool_held(pool):             # token-exhaustion hold (see pattern 5)
        return  # try again next tick; the OTHER pool is unaffected

    if pool_slots_free(pool) == 0:
        return  # full — queue, retry next tick (FIFO topup)

    launch(pool, lane)
```

Two things are happening here that matter:

1. **Capacity is a gate, not an error.** A full pool isn't a failure — the lane simply isn't dispatched this tick and is retried on the next one. Slots free up as workers finish; the queue drains FIFO. No lane is ever dropped for being briefly unlucky.

2. **A held pool is skipped, not fatal.** If a provider is out of tokens, its pool is *held* (pattern 5) and the router transparently skips it. Lanes targeting the healthy provider keep dispatching at full speed. The system degrades to half-throughput, not zero.

## The "it just worked" moment

The combination of (a) per-pool capacity counting and (b) a force-toggle and (c) per-pool holds is what produces the behaviour that looks magical from the board:

- MiniMax fills up with cheap grunt work → new lanes that *could* run on either provider land on Claude instead → throughput stays high.
- MiniMax runs out of tokens overnight → its pool holds until reset → Claude lanes keep merging all night → in the morning MiniMax auto-resumes when the hold expires.
- The owner flips **Force MiniMax** for a cost-sensitive batch → every lane, even Claude-tagged ones, reroutes to MiniMax → one toggle, whole-fleet effect.

None of that requires a human watching. It's three small deterministic rules composing.

## Picking the right model per lane

Routing also encodes a cost/quality policy. The default assignment we use:

- **MiniMax** — the workhorse. Bulk implementation lanes, mechanical transforms, anything where throughput and cost matter more than finesse.
- **Claude** — design, review (the Neo gate), and the hardest lanes where getting it right the first time saves a re-dispatch.

The model is just a string in the lane's plan, so the policy is **data you can see and change**, not logic buried in code. A lane that keeps bouncing on MiniMax gets promoted to Claude by editing one field.

## The rule, stated plainly

**Treat providers as interchangeable capacity behind a router, not as hardcoded dependencies.** Count each pool honestly, make "full" a retry rather than a failure, make "out of tokens" a per-pool hold rather than a system outage, and give the human one override toggle. Half your fleet should never be able to take down the other half.
