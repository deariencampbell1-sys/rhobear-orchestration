# AGENTS.md — rhobear-orchestration

<!-- lane-start -->
## Start here (RHOBEAR lanes, 2026-10-24)

This repo is a **source** for lane **L8a Blueprints**: the lane extracts what a kept product needs, then this repo is deleted. Do not start new work here. Before you change anything, read in `rhobear-north-star` (`docs/north-star/lanes/`): `README.md`, `GATE.md`, then `L8a-blueprints/BRIEF.md` and `L8a-blueprints/SOURCES.md`.

- What wins when sources disagree: the owner's newest words, then the final iteration (the newest board and the newest
  code), then the docs, this file included.
- No worker starts here until the lane architect's gate is signed and a worker brief names your files.
- Old UX never comes back: a screen, style, font, colour or mark that is not on the current board is deleted, not
  restyled. Fonts: `docs/north-star/FONTS.md` in `rhobear-north-star` (Nacelle is banned).
- Workers run only on Core, Peak, Summit or Guest. No keys in prompts, files or commits.
<!-- /lane-start -->

<!-- RHOBEAR-CANON:OPERATING-DISCIPLINE v1 — source of truth: rhobear-app/docs/canon/OPERATING-DISCIPLINE.md -->
## OPERATING DISCIPLINE — RHOBEAR CANON v1 (every agent, every spawn)
Binds you whether spun up as worker, crew agent, Pi step, or courier. Read first.

1. CODEGRAPH FIRST. Before grep/glob/find/read-to-locate, query CodeGraph
   (code-graph-mcp) — it has the symbol/dependency/route answer at a fraction of the
   tokens. Grep/glob are the FALLBACK. If CodeGraph is unreachable, say so in one
   line, then fall back.
2. ONE AGENT. ONE THREAD. ONE STEP. Never spawn subagents. No Agent/Task fan-out, no
   parallel instances in-thread. Work like Pi: smallest next step toward the goal,
   finish it, then the next. No speculative scaffolding. If the job is bigger than
   one agent should hold, STOP and hand back a note — don't recruit.
3. LEAN TOKENS. Let RTK-G (rtk) compress noisy tool output. Speak caveman-terse by
   default; keep every path/command/error string exact. (Exception: reporting to the
   owner in a planning thread — stay readable.)
4. KNOW WHERE YOU ARE. You are one of many agents across local + 3 VMs (GCP
   chat-brain-1, Hetzner rhobear-vps, Azure rhobear-bld-01). Consult docs/canon/
   before assuming where anything lives. One project = one repo.
5. STAY IN YOUR LANE. Do only the lane you were given. Out-of-lane → flag, don't absorb.
<!-- /RHOBEAR-CANON -->

This repo = self-driving agent-swarm orchestration patterns and documentation (one lane).