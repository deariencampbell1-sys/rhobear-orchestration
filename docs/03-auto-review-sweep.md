# Pattern 3 — Auto-review by sweep

> One reviewer, fired in batches over every open PR — not one reviewer spawned per PR.

When workers open pull requests autonomously, you need a review gate that's also autonomous. The naive design spawns a reviewer the instant each PR opens. That falls apart fast: five lanes finishing within a minute spawn five reviewers that step on each other, burn five times the tokens, and race each other to merge into the same branch.

## The pattern: a sweeper, not a trigger

Instead of per-PR triggers, run a **sweeper** on an interval (ours: every 5 minutes):

```
sweep:
  if a review is already in flight: return        # one at a time
  prs = open_unreviewed_prs_from_done_lanes()     # gather the BATCH
  if not prs: return
  fire_one_reviewer(prs)                          # a single "Neo" over all of them
  record_pending(prs)
```

The sweeper gathers **all** currently-mergeable PRs and hands them to a *single* reviewer pass. This is strictly better than per-PR triggering:

- **No thundering herd.** Five PRs finishing together = one review pass, not five.
- **Cross-PR awareness.** The reviewer sees the whole batch and can catch conflicts between sibling PRs that a per-PR reviewer would miss.
- **Bounded cost.** Review spend is a function of how often you sweep, not how many PRs happen to land.
- **Backpressure for free.** "One review in flight at a time" means the gate can't be overwhelmed.

## What the reviewer ("Neo") actually does

The reviewer is the one place a model *should* be in the loop — judgement is exactly what it's for. For each PR in the batch it:

1. Reads the full diff and verifies the worker's claims itself (does the code do what the PR says?).
2. Checks the findings from any automated review bots already attached to the PR.
3. Returns a verdict: **ACCEPT** (merge it), **FIX-FORWARD** (small fixes, then merge), **BOUNCE** (send back to the lane), or **ESCALATE** (a human needs to look).
4. On green, **it merges.** Nothing lands without passing the gate.

Crucially, the reviewer is *separate from the worker that wrote the code*. An agent never reviews and merges its own work — the gate is adversarial by construction.

## Closing the loop: the review-close wake

The sweeper isn't just a merger; it's also a **notification source**. When a sweep's reviewer finishes, the sweeper detects the now-merged PRs and fires a *review-closed* event into the architect outbox (pattern 4). That's what wakes the human-facing agent to write the audit line you see in [the proof](proof.md).

This matters because a wave can complete in two ways: the supervisor advances it directly, **or** the sweeper merges its last PR out-of-band. Without the sweeper emitting its own wake, those sweep-merged waves would close silently. The fix was to make the sweeper a first-class wake source, routed to the right lane via a small `repo → lane` map.

## The rule, stated plainly

**Batch your autonomous reviews on a timer; don't trigger one per artifact.** A single reviewer sweeping all open work, one pass at a time, is cheaper, conflict-aware, and impossible to stampede — and if you make it emit its own completion events, it doubles as the signal that closes the loop back to the human.
