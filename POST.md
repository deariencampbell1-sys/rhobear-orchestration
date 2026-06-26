# We built a self-driving swarm of coding agents. Here are the patterns — free.

For the last stretch we've been running something that, honestly, still feels like cheating: a fleet of AI coding agents that pick up work, write the code, review *each other*, merge what passes, fix what breaks, and write up what happened — continuously, overnight, with nobody watching.

We're not selling the system. We're **giving away the patterns** that make it actually work — the unglamorous parts nobody puts in the demo. They're in one repo: **[rhobear-orchestration](https://github.com/deariencampbell1-sys/rhobear-orchestration)**.

Here's the gist.

## The board

Everything is visible on one screen — a kanban of every wave of work moving Queued → Running → Review → Escalated → Done, plus owner kill-switches and a live token meter.

![The board, all states](screenshots/board-all-states.png)

That **Escalated** card with `auto-fix attempts: 3/3 · NEEDS OWNER`? That's the system admitting it's stuck instead of thrashing forever. The **Blocked** card next to it is a wave correctly refusing to start because its upstream dependency broke. Both states are the system being honest about reality — which is most of the battle.

## The five patterns

1. **A deterministic supervisor.** The loop that decides *"this phase is done, start the next one"* is plain code — **zero LLM calls.** Models write code and review it; *rules* move the line. That one decision is what makes the whole thing cheap, reproducible, and pausable.

2. **Capacity-aware model routing.** Two providers, two pools. When one is full, work routes to the other. When one runs out of tokens, *that pool holds* and the other keeps merging all night. One toggle pins everything to the cheap provider when you want.

3. **Auto-review by sweep.** Instead of spawning a reviewer per pull request (and getting a five-way stampede when lanes finish together), a sweeper batches *every* open PR and fires *one* reviewer that reviews, fixes, and merges on green. One at a time. No thundering herd.

4. **The async architect.** A human-facing agent that's an *auditor, not a gate*. Waves advance on their own; the architect just wakes on each state change, verifies, and writes the record — so the human stays oriented without being the bottleneck.

5. **Quota & zombie resilience.** The night a provider ran dry and the lesson that a "hung worker" was our own telemetry plugin, not the model. Per-pool holds, a circuit breaker on the auto-fixer, and observability kept *outside* the worker's critical path.

## Owner controls

You can stop the whole thing cold. Pause the supervisor and it stops dispatching; one button kills every worker.

![Owner controls — paused](screenshots/board-paused.png)

## "But does it actually do anything?"

Fair question — the failure mode of every "autonomous" system is the quiet lie. So we hold ours to a rule: **every state change leaves a record a human can read later.** Here's one real night of the architect waking on each merge and writing its own audit — including a line where it caught a duplicate notification and *refused to double-write*:

```
22:33  review_closed   — partial merge advanced (#5, #6)
23:14  wave_done       — #8/#9/#10 merged; ci=ok build=ok core=ok engine=ok fidelity=ok
23:15  wave_done       — DUPLICATE re-delivery; marked read, NO double audit
00:47  review_closed   — wave merged
01:44  review_closed   — wave merged
02:12  review_closed   — #19 merged; wave advanced
03:29  review_closed   — #20 merged/advanced
```

Eight autonomous wakes, five hours, nobody awake. The full write-up — diagrams, the contracts, every hard-won fix — is in the repo.

## Why free?

Because the ideas are worth more shared than hoarded, and because the parts that are actually hard to get right (what happens at 2am when a provider runs dry, how you stop a worker looping in the wrong file, how you *know* it merged something) are exactly the parts most write-ups skip. If these save you a week, we'd love a star and a link back.

**→ [github.com/deariencampbell1-sys/rhobear-orchestration](https://github.com/deariencampbell1-sys/rhobear-orchestration)**
