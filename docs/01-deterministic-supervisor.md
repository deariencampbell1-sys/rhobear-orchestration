# Pattern 1 — The deterministic supervisor

> Deciding *what runs next* is plain code. Models do the work; rules move the line.

## The problem

The instinct when building an "agentic" system is to put an LLM in charge of everything, including orchestration: *"You are the project manager. Look at the state and decide what to do next."* This feels powerful and is a trap. An LLM in the progression loop is:

- **Expensive** — you pay tokens on every tick just to decide nothing changed.
- **Non-deterministic** — the same state can produce different decisions, so bugs aren't reproducible.
- **Slow to trust** — you can never be sure it won't hallucinate a phase as "done."
- **Hard to pause** — there's no clean place to put a kill-switch.

## The pattern

Split the system into two kinds of work:

| Creative work (LLM) | Progression work (code) |
|---------------------|-------------------------|
| Writing the code in a lane | Deciding a phase is complete |
| Reviewing a PR | Dispatching the next phase |
| Unsticking a broken lane | Counting pool capacity |
| Auditing what happened | Advancing the wave |

The progression half is a **deterministic supervisor**: a stateless tick that runs on a fixed interval (ours is every 30 seconds), reads the current state from disk, applies rules, takes at most a few actions, and exits. Fresh process every tick. No memory, no model call, nothing to hallucinate.

```
tick:
  if control.supervisor_paused: return          # owner kill-switch, read every tick
  for plan in active_wave_plans():              # strict per-project order
    verdict = judge(plan)                        # pure function of files on disk
    match verdict:
      satisfied  -> advance_phase(plan); emit_packet()
      running    -> pass                          # do nothing, check next tick
      needs_neo  -> ensure_neo_dispatched(plan)
      failed     -> handle_break(plan)
```

## The load-bearing idea: `expect:` contracts

The hard question a progression loop must answer is *"is this lane done?"* — and "done" means different things: a PR was **merged**, a reviewer **accepted**, a command **exited 0**, an issue was **closed**. If you ask an LLM "is it done?" you get fuzzy, unreproducible answers.

Instead, every lane declares its completion contract **at plan time**, as data:

```yaml
lanes:
  - lane: g1-vendor-grapes
    expect: merged          # done when its PR is merged
    repo: org/repo
    branch: lane/g1-vendor-grapes
  - lane: smoke-test
    expect: success_returncode   # done when the command exits 0
```

Now `judge()` is a pure function: for `expect: merged` it asks the git host "is this branch's PR merged?" — a fact, not an opinion. Different completion semantics, zero fuzzy models. This single decision (`expect:` as declared data) is what lets the whole progression loop stay LLM-free.

## What you get

- **Reproducible.** Same files on disk → same decision, every time. Bugs are debuggable.
- **Cheap.** A tick that finds nothing to do costs a few milliseconds of Python, not a model call.
- **Pausable.** Because every tick re-reads a control file, a single boolean (`supervisor_paused`) cleanly stops the whole system — and resumes it — with no in-flight state to corrupt.
- **Restartable.** State lives on disk, not in a long-running process. Kill the supervisor mid-tick and the next tick picks up exactly where it should.

## The rule, stated plainly

**Never put an LLM in the loop that decides "phase N is done, dispatch phase N+1."** Reserve models for the two things only they can do: the creative work inside a lane, and unsticking something genuinely broken. Everything in between is rules.
