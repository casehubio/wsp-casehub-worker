# Idea Log

Undecided possibilities — things worth remembering but not yet decided.
Promote to an ADR when ready to decide; discard when no longer relevant.

## 2026-07-28 — Hybrid failure recovery: replay + context re-evaluation

Temporal uses event-sourced replay for single-process durability. CaseHub orchestrates whole systems, where replay alone isn't sufficient — the world may have changed across multiple independent components while a process was down.

**Hybrid approach:** offer both recovery strategies, selected per execution context:
- **Replay** for orchestrated flows (prescribed execution order) — walk the EventLog forward, skip activities with recorded outcomes. Requires idempotent/replayable worker marking and deterministic binding evaluation with frozen context snapshots.
- **Context re-evaluation** for choreographed flows — reload case state, re-evaluate bindings against current context. Handles cases where independent system components moved on during downtime.

The sealed `WorkerResult<R>` model enables replay — results are serializable, deterministic, and carry everything needed to skip re-execution. The existing EventLog/Ledger infrastructure already records the transitions.

**Quarkus Flow alignment:** if Quarkus Flow adopts replay-style execution, CaseHub could use that as the orchestration-mode backend while keeping choreography re-evaluation as-is.

**Key distinction:** Temporal optimises for one process surviving failure. CaseHub optimises for a whole system continuing to function. The right recovery strategy depends on what's deployed and running — not a global choice, but a per-flow decision.

**Boundary risk:** cases mixing choreographed and orchestrated stages need clear rules about which recovery mode applies where. Design problem, not feasibility problem.

**Scope:** cross-repo (engine, worker, platform). Depends on engine#237 (lifecycle scopes) landing first.
