# Closing the Result Model

**Date:** 2026-07-01
**Branch:** issue-8-exception-to-failed
**Issue:** casehubio/casehub-worker#8

## What happened

The `WorkerExecutor` had two failure channels: `WorkerOutcome.Failed`/`Expired` returned as results, and uncaught exceptions propagated from the worker function or `PolicyEnforcer`. Callers had to handle both — check the result AND catch exceptions. The `Expired` outcome existed in the sealed hierarchy but nothing in the runtime produced it.

The root cause ran deeper than the worker repo. `DefaultPolicyEnforcer` in `casehub-platform-governance` used a single untyped `PolicyEnforcementException` for three semantically distinct failures: timeout, retry exhaustion, and thread interruption. Worse, the retry loop double-wrapped policy exceptions — a timeout became `PolicyEnforcementException("All 1 attempts failed", PolicyEnforcementException("timed out"))`, destroying type information at the top level.

## What we built

**Platform-governance (prerequisite):** Three typed exception subclasses — `TimeoutPolicyException`, `InterruptedPolicyException`, `RetryExhaustedException`. Fixed the retry loop to re-throw policy exceptions directly instead of wrapping them. Added uniform interrupt handling: retry loop breaks on interrupt, `sleep()` throws on interrupt (instead of silently continuing and burning through attempts in microseconds), and the no-timeout path detects interrupts via the thread flag.

**Worker:** `DefaultWorkerExecutor` now catches by type — `TimeoutPolicyException` maps to `Expired`, `RetryExhaustedException` and raw exceptions map to `Failed`, `InterruptedPolicyException` propagates. OTel semantics: timeout is an event not an error; all paths set the `worker.outcome` span attribute. `MockWorkerExecutor` also honours the contract.

The result model is closed: `execute()` always returns a `WorkerResult` for worker-level conditions. Only infrastructure collapse (interrupt, `Error`) propagates.

## Design review

Adversarial review ran 5 rounds ($13.65), raised 14 issues. All resolved — 13 verified, 1 accepted. Key additions from the review: uniform interrupt handling across all `DefaultPolicyEnforcer` code paths, null message fallback to class name (prevents `Failed(null)` NPE at call sites), OTel timeout event semantics, and `MockWorkerExecutor` contract alignment.

## What's next

Issue #6 (timeout enforcement) should be re-scoped — the sync timeout-to-Expired mapping is now delivered by this work. #6's remaining scope is async worker timeout (depends on #5) and workers-within-deadline testing.

## Garden entries

- GE-20260701-fdf192: Retry loop wrapping policy exceptions destroys type information
- GE-20260702-21be12: Thread.sleep catch-and-continue burns through retry attempts
