# Handoff

## Last Session — 2026-07-17

Landed `issue-5-async-worker-function` — added `WorkerFunction.Async<T>` (CompletionStage return), changed `WorkerExecutor.execute()` to return `Uni<WorkerResult>`, replaced PolicyEnforcer with SmallRye FT Guard for fault tolerance. Unified sync/async pipeline in DefaultWorkerExecutor. Adversarial design review (3 rounds, 14 issues — 8 verified, 6 accepted). Key review-driven change: executor returns Uni instead of CompletionStage. Squashed to `60e185b`, pushed to fork and upstream. Garden entry: GE-20260718-052fbc (Guard SPI in unit tests).

## Immediate Next Step

Engine#726 is unblocked — the engine can now build an `AsyncWorkerFunctionHandler` to dispatch non-blocking workers through the SPI. Run `/work` on the engine repo.

## What's Left

- #4 WorkerContext — ambient execution state, changes WorkerFunction signature · M · Med
- #6 Timeout enforcement — re-scope after #8, remaining: async timeout + deadline tests · M · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| engine#726 | Engine Sync handler delegates to worker executor | M | Med | Unblocked — now also needs AsyncWorkerFunctionHandler |
| #4 | WorkerContext | M | Med | Blocked by engine#237, engine#419 |
| #6 | Timeout enforcement | M | Med | Re-scope after #8 |

## Cross-Module

**Linked:**
- engine#726 — Sync + Async handler delegates to worker executor (unblocked)
- engine#237 — long-lived workers with lifecycle scopes. Linked to worker#4 (WorkerContext)
- engine#419 — CaseContextProvider SPI. Linked to worker#4 (WorkerContext)

## References

- Garden: `GE-20260718-052fbc` — SmallRye FT Guard.create() fails in plain JUnit tests — needs standalone SPI
- Blog: `2026-07-17-mdp02-async-workers-and-policyenforcer-funeral.md`
- Spec: `docs/specs/2026-07-17-async-worker-function-design.md`
