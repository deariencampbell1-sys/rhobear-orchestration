# Architecture

Four Mermaid flowcharts describing the swarm. Paste any block into a Mermaid renderer (GitHub renders them natively, or [mermaid.live](https://mermaid.live)).

1. **Claude lane** — how a Claude worker is dispatched and run
2. **MiniMax lane** — how a MiniMax worker is dispatched and run
3. **The watcher** — per-turn token telemetry (reads what 1 & 2 write)
4. **The whole web** — everything wired together

Colour legend (consistent across all four):
🟪 owner / control · 🟧 supervisor core · 🟦 Claude path · 🟩 MiniMax path · 🟦‍🟩 telemetry · 🟥 escalation/courier

The whole thing runs on a single orchestrator host. Paths below are generic (`~/swarm`, `~/.claude`, etc.); names of hosts, accounts, and tokens are deliberately omitted.

---

## 1. Claude lane — `dispatch2.sh` → `run-worker-claude.sh`

```mermaid
flowchart TD
    subgraph SUP["Supervisor tick (deterministic, every 30s)"]
        A["wave_dispatcher.dispatch_lane(lane)"]
        B{"_dispatcher_for(model)<br/>model starts with 'claude'?"}
        CAP{"Claude pool slot free?<br/>POOL_CAPS.claude = 5"}
    end

    A --> B
    B -- "yes" --> CAP
    CAP -- "no — queue, retry next tick" --> A
    CAP -- "yes" --> D2["dispatch2.sh LANE WORKDIR BRIEF claude-model high"]

    subgraph LAUNCH["dispatch2.sh"]
        D2 --> CMD["write ~/swarm/dispatched/LANE.cmd"]
        CMD --> TMUX["tmux new-session -d -s LANE<br/>bash run-worker-claude.sh ..."]
    end

    TMUX --> RW["run-worker-claude.sh"]

    subgraph WORKER["run-worker-claude.sh (one shot, run to completion)"]
        RW --> GH["export GH_TOKEN (gh auth token)"]
        GH --> AUTH["source local env file<br/>(Claude OAuth token)"]
        AUTH --> START["write START line to ~/swarm/status/LANE.status<br/>fire 🔵 ops-bot START"]
        START --> CD{"cd WORKDIR<br/>(git worktree) exists?"}
        CD -- "NO" --> RC66["RC=66 'workdir missing'<br/>dies in ~1s, no log, no claude"]
        CD -- "no auth" --> RC67["RC=67 'no Claude auth'"]
        CD -- "yes" --> CLAUDE["claude -p '$(cat BRIEF)'<br/>--model claude-model<br/>--dangerously-skip-permissions  &gt; LOG"]
        CLAUDE --> DONE["write DONE rc=N to status<br/>parse PR URL · fire END notification"]
        RC66 --> DONE
        RC67 --> DONE
    end

    PRE["PREREQUISITE (per lane, before dispatch):<br/>git worktree add ~/swarm/work/LANE -b lane/LANE"]
    PRE -. "creates WORKDIR" .-> CD

    CLAUDE --> JSONL["~/.claude/projects/.../SESSION.jsonl<br/>usage{input,output,cache_read,cache_creation} per turn"]
    DONE --> MERGE["PR opened READY → Neo gate reviews → merges"]

    FM{"control.json force_minimax = ON?"}
    A -.-> FM
    FM -. "if ON, model rewritten → MiniMax (see diagram 2)" .-> A

    classDef owner fill:#241a3a,stroke:#7c5cff,color:#e9e9f0
    classDef sup fill:#2a1f10,stroke:#ffae42,color:#ffe9c7
    classDef claude fill:#10212a,stroke:#5cc8ff,color:#d7f0ff
    classDef tel fill:#0f2622,stroke:#3ad07a,color:#cffbe6
    classDef bad fill:#1c0f14,stroke:#ff5470,color:#ffd7df
    class A,B,CAP,FM sup
    class D2,CMD,TMUX,RW,GH,AUTH,START,CD,CLAUDE,DONE,PRE claude
    class RC66,RC67 bad
    class JSONL tel
    class MERGE sup
```

**Key facts**
- The Claude worker authenticates from a local env file holding an OAuth token tied to a Claude **subscription**, not a metered API key.
- `rc=66` = the worktree directory didn't exist → `claude` never even started. This is the single most common false "Claude is broken" — it's a missing `git worktree add`, not a quota/limit.
- Each turn's token usage is written to the session JSONL **during** the run (the telemetry source — diagram 3).

---

## 2. MiniMax lane — `dispatch.sh` → `run-worker.sh`

```mermaid
flowchart TD
    subgraph SUP["Supervisor tick (deterministic, every 30s)"]
        A["wave_dispatcher.dispatch_lane(lane)"]
        B{"_dispatcher_for(model)<br/>model starts with 'claude'?"}
        CAP{"MiniMax pool slot free?<br/>POOL_CAPS.minimax = 5"}
    end

    A --> B
    B -- "no (default)" --> CAP
    CAP -- "no — queue, FIFO topup next tick" --> A
    CAP -- "yes" --> D1["dispatch.sh LANE WORKDIR BRIEF MiniMax-M3 high"]

    subgraph LAUNCH["dispatch.sh"]
        D1 --> TMUX["tmux new-session -d -s LANE<br/>bash run-worker.sh ..."]
    end

    TMUX --> RW["run-worker.sh"]

    subgraph WORKER["run-worker.sh (one shot, run to completion)"]
        RW --> GH["export GH_TOKEN (gh auth token)"]
        GH --> START["write START line to ~/swarm/status/LANE.status<br/>fire 🟢 ops-bot START"]
        START --> CD{"cd WORKDIR<br/>(git worktree) exists?"}
        CD -- "NO" --> RC66["RC=66 'workdir missing'"]
        CD -- "yes" --> PI["pi -p '$(cat BRIEF)'<br/>--provider minimax --model MiniMax-M3<br/>--thinking high --approve  &gt; LOG"]
        PI --> DONE["write DONE rc=N · parse PR · fire END notification"]
        RC66 --> DONE
    end

    subgraph PICFG["pi runtime config"]
        AUTH["local auth file (MiniMax key)"]
        NOLF["langfuse plugin REMOVED from packages<br/>(was hanging workers on a 502 — the 'zombie')"]
    end
    AUTH -.-> PI
    NOLF -.-> PI

    PI --> JSONL["~/.pi/agent/sessions/.../SESSION.jsonl<br/>usage{input,output,cacheRead,cacheWrite,totalTokens,cost} per turn"]
    DONE --> MERGE["PR opened READY → Neo gate reviews → merges"]

    PRE["PREREQUISITE (per lane):<br/>git worktree add ~/swarm/work/LANE -b lane/LANE"]
    PRE -. "creates WORKDIR" .-> CD

    classDef sup fill:#2a1f10,stroke:#ffae42,color:#ffe9c7
    classDef mini fill:#0f2618,stroke:#3ad07a,color:#cffbe6
    classDef tel fill:#10212a,stroke:#5cc8ff,color:#d7f0ff
    classDef bad fill:#1c0f14,stroke:#ff5470,color:#ffd7df
    class A,B,CAP sup
    class D1,TMUX,RW,GH,START,CD,PI,DONE,PRE,AUTH,NOLF mini
    class RC66 bad
    class JSONL tel
    class MERGE sup
```

**Key facts**
- `pi` is hardcoded to `--provider minimax`; the model string only chooses the MiniMax variant. `dispatch.sh` can never reach Claude — provider choice is a *dispatcher* decision, not a worker one.
- **The zombie fix:** a telemetry plugin was removed from the agent config. When the self-hosted telemetry backend returned a 502, that plugin's exit-flush hung the worker at 0% CPU. It was misread as a "model API crash" for three debugging sessions. The agent now exits clean in ~3s. *Lesson: a hung worker is far more often your own observability sidecar than the model.*
- Same session-JSONL telemetry shape as Claude, but the MiniMax runtime also writes a per-turn **cost** field (Claude usage is priced by a local table — diagram 3).

---

## 3. The watcher — token-ledger telemetry

```mermaid
flowchart LR
    subgraph SOURCES["Usage sources (append-only, written per-turn DURING the run)"]
        PIJ["pi sessions<br/>~/.pi/agent/sessions/*/*.jsonl"]
        CLJ["claude sessions<br/>~/.claude/projects/*/*.jsonl"]
        EMIT["generic emitter sink<br/>~/telemetry/events/*.jsonl<br/>(apps, engine, paid AI)"]
    end

    subgraph LEDGER["token_ledger.py"]
        OFF["per-file byte-offset tracker<br/>(ingest_state table) — read only NEW lines<br/>re-read from 0 if file truncated/rotated"]
        AD{"adapter by source"}
        APING["_event_from_pi<br/>usage + cost direct"]
        ACL["_event_from_claude<br/>tokens → cost via PRICE table"]
        AEM["_event_from_emitter<br/>app/account/model schema"]
        DERIVE["derive lane / project / app<br/>from path + session-id"]
        DEDUP["INSERT OR IGNORE<br/>UNIQUE(file,line_no) — idempotent"]
    end

    PIJ --> OFF
    CLJ --> OFF
    EMIT --> OFF
    OFF --> AD
    AD --> APING --> DERIVE
    AD --> ACL --> DERIVE
    AD --> AEM --> DERIVE
    DERIVE --> DEDUP --> DB[("SQLite usage_events<br/>ts·source·app·project·lane·model·<br/>input·output·cache·total·cost_usd")]

    subgraph MODES["run modes"]
        BF["backfill — ingest everything on disk now"]
        WATCH["watch — poll every 3s, ingest new lines<br/>(off the worker path — cannot hang a worker)"]
        REP["report — aggregate by app/lane/model"]
    end
    BF --> DB
    WATCH --> DB
    DB --> REP
    DB --> API["dashboard /api/tokens → Tokens panel<br/>spend by lane / model / project / day"]

    classDef tel fill:#0f2622,stroke:#3ad07a,color:#cffbe6
    classDef store fill:#241a3a,stroke:#7c5cff,color:#e9e9f0
    classDef src fill:#10212a,stroke:#5cc8ff,color:#d7f0ff
    class PIJ,CLJ,EMIT src
    class OFF,AD,APING,ACL,AEM,DERIVE,DEDUP,BF,WATCH,REP tel
    class DB,API store
```

**Key facts**
- Usage is appended to the session JSONL **line-by-line, ~1–2s after each turn, while the process is still alive** — so a live meter works without waiting for a session to close.
- The watcher reads files **an arm's length from the workers**, so a telemetry backend going down can never hang a worker (the lesson from the zombie above).
- The same `usage_events` schema accepts a **generic emitter** sink — that's where a customer billing meter plugs in, keyed by `account_id`.

---

## 4. The whole web — everything wired together

```mermaid
flowchart TB
    OWNER(["Owner"])

    subgraph BOARD["Dashboard (localhost)"]
        KAN["Kanban: Queued/Running/Review/Escalated/Done"]
        CTRL["Owner controls → control.json<br/>supervisor_paused · autofix_enabled · force_minimax"]
        KILL["STOP everything (kill workers + pause)"]
        TOK["Tokens panel"]
        LIVE["Live workers list (neo/fixer/lane)"]
    end
    OWNER --> CTRL
    OWNER --> KILL

    subgraph CORE["Supervisor (deterministic tick, every 30s — ZERO LLM)"]
        TICK["main() tick"]
        PAUSE{"control.supervisor_paused?"}
        PLANS["list_active_wave_plans()<br/>smart sorter: strict per-project order"]
        JUDGE["wave_judge: per-lane expect<br/>satisfied / running / needs_neo / failed / escalate"]
        ADV["advance phase · emit architect packet"]
    end
    CTRL -. "read each tick" .-> PAUSE
    TICK --> PAUSE
    PAUSE -- "paused → skip" --> TICK
    PAUSE -- "running" --> PLANS --> JUDGE --> ADV

    JUDGE -- "needs Claude lane" --> CLA["dispatch2.sh → run-worker-claude.sh<br/>(diagram 1)"]
    JUDGE -- "needs MiniMax lane" --> MIN["dispatch.sh → run-worker.sh<br/>(diagram 2)"]
    CTRL -. "force_minimax reroutes Claude→MiniMax" .-> MIN

    CLA --> PRJ["PR opened READY"]
    MIN --> PRJ
    PRJ --> NEO["Neo gate: review → merge"]
    NEO --> ADV

    JUDGE -- "failed / escalate" --> BREAK["_handle_break"]
    CTRL -. "autofix_enabled?" .-> BREAK
    BREAK -- "autofix ON" --> FIX["supervisor-fixer (reads RUNBOOK)<br/>circuit breaker: 3 attempts → needs-owner"]
    BREAK -- "autofix OFF" --> WAIT["wave left escalated, waits for owner"]
    FIX -- "writes .resolved" --> ADV
    FIX -- "3x fail" --> NEEDS["needs-owner → card to ops bot"]

    ADV --> OUTBOX["architect-outbox/&lt;lane&gt;/*.md<br/>wave_started · phase_advanced · wave_done · chain_complete"]
    OUTBOX --> COURIER["hub-receiver courier (the human's machine)"]
    COURIER --> RESUME["claude -p --resume SESSION<br/>wraps packet → wakes the architect thread"]
    RESUME --> IRON(["Architect agent audits + commits"])

    CLA --> SJ["session JSONL (Claude)"]
    MIN --> SJ2["session JSONL (pi)"]
    SJ --> WATCHER["token_ledger watcher (diagram 3)"]
    SJ2 --> WATCHER
    WATCHER --> SQL[("SQLite usage_events")]
    SQL --> TOK
    PLANS --> KAN
    CLA --> LIVE
    MIN --> LIVE
    NEO --> LIVE

    APPS["Customer apps (paid AI)<br/>route AI calls server-side → meter per account"]
    APPS --> EMITSINK["~/telemetry/events/*.jsonl"]
    EMITSINK --> WATCHER
    SQL --> BILL["credits ledger → payments (future)"]

    classDef owner fill:#241a3a,stroke:#7c5cff,color:#e9e9f0
    classDef sup fill:#2a1f10,stroke:#ffae42,color:#ffe9c7
    classDef claude fill:#10212a,stroke:#5cc8ff,color:#d7f0ff
    classDef mini fill:#0f2618,stroke:#3ad07a,color:#cffbe6
    classDef tel fill:#0f2622,stroke:#3ad07a,color:#cffbe6
    classDef esc fill:#1c0f14,stroke:#ff5470,color:#ffd7df
    class OWNER,KAN,CTRL,KILL,TOK,LIVE,IRON owner
    class TICK,PAUSE,PLANS,JUDGE,ADV,NEO sup
    class CLA,SJ claude
    class MIN,SJ2 mini
    class WATCHER,SQL,APPS,EMITSINK,BILL tel
    class BREAK,FIX,WAIT,NEEDS,OUTBOX,COURIER,RESUME esc
```

**Reading it in one breath:** the owner sets toggles on the board → the deterministic supervisor reads them each tick, surfaces eligible waves, and dispatches workers down the Claude or MiniMax path (force-MiniMax can reroute). Workers open PRs that Neo reviews and merges; breakage is auto-fixed by a capped fixer (or handed to the owner). Every state change couriers a packet that wakes the architect agent to audit. Meanwhile every worker's token usage flows from its session JSONL into the ledger and back onto the board — and the same ledger is where customer billing plugs in.
