# Handoff

## Last Session — 2026-07-03

Completed worker#9: capability-aware execution. `WorkerExecutor.execute()` now takes `(Worker, Capability, Map)` — validates capability membership, rejects null capability and non-Sync functions before PolicyEnforcer, adds `worker.capability` OTel span attribute. Design-reviewed (3 rounds, 13 issues, all resolved). Filed #10 (engine convergence), #11 (MockWorkerExecutor minor hardening), parent#339 (doc sync).

Also filed #9 and #10 on epic #3, updated the epic scope and suggested order.

## Immediate Next Step

Pick up #7 (schema validation) — now unblocked by #9. The executor has `Capability.inputSchema()` and `Capability.outputSchema()` available. Design decision needed: JSON Schema library choice (networknt, everit, or manual).

## What's Left

- #4 WorkerContext — ambient execution state. Changes WorkerFunction signature. Needs engine-side migration issue. · M · Med
- #5 Async WorkerFunction — CompletionStage execution. Needs engine-side executor changes. · M · Med
- #6 Timeout enforcement — re-scope after #8. Remaining: async timeout, deadline tests. · M · Med
- #7 Schema validation — enforce Capability inputSchema/outputSchema. Now unblocked by #9. · S · Med
- #10 Engine SyncAgentWorkerFunctionHandler delegates to this executor. · M · Med
- #11 MockWorkerExecutor minor hardening — non-Sync guard + null test. · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #7 | Schema validation | S | Med | Unblocked by #9 — proceed now |
| #11 | MockWorkerExecutor hardening | XS | Low | Independent — quick fix |
| #4 | WorkerContext | M | Med | Blocked by cross-repo tooling |
| #5 | Async WorkerFunction | M | Med | Blocked by cross-repo tooling |
| #10 | Engine convergence | M | Med | Cross-repo — needs engine-side work |

## Cross-Module

**Linked:**
- engine#237 — long-lived workers with lifecycle scopes. Commented re: worker#4 (WorkerContext)
- engine#419 — CaseContextProvider SPI. Commented re: worker#4 (WorkerContext)
- parent#339 — doc sync for capability-aware execution (deep-dive + PLATFORM.md)

## References

- Spec: `docs/specs/2026-07-02-capability-aware-execution-design.md`
- Plan: `plans/attic/issue-9-capability-aware-execution/2026-07-03-capability-aware-execution.md`
- Design review: `~/adr/casehub-worker/capability-aware-execution-*/`
- Blog: `blog/2026-07-01-mdp02-closing-the-result-model.md` (published, prior session)
