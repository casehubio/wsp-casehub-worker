---
layout: post
title: "CaseHub Worker — First Blood"
date: 2026-06-23
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-worker]
tags: [foundation-tier, type-design, structural-enforcement]
---

# CaseHub Worker — First Blood

**Date:** 2026-06-23
**Type:** phase-update

---

## What I was trying to achieve: PlannedAction on the foundation tier

casehub-worker exists to own the canonical worker vocabulary — Worker, Capability, WorkerFunction, WorkerResult, WorkerOutcome. It shipped as a skeleton in the bootstrap issue. This session adds the first real type: PlannedAction, a record that declares what consequential action a worker intends to take before the engine advances the case.

The work is a prerequisite for engine#543, which migrates nine files of parallel type definitions out of engine-api and onto their foundation-tier homes. PlannedAction is the first to move.

## What I believed going in: the issue had it figured out

The issue for this work was detailed — exact signatures, rationale for every decision, code samples. I expected transcription. The spec review proved otherwise.

## Extraction is not transcription

The engine's PlannedAction carries five fields: `workerId`, `caseId`, `description`, `actionType`, `context`. It has a `withIdentity()` method that enriches the record with worker and case identity before passing it to the risk classifier. The foundation-tier type strips all of that. PlannedAction in worker-api has three fields: `description`, `actionType`, `parameters`.

The identity fields (`workerId`, `caseId`) are engine routing concerns — a worker declaring an action doesn't know which case it belongs to. That knowledge lives in the engine's execution pipeline. Engine#543 handles this by introducing `ClassificationContext`, a separate record that carries identity alongside the action at the classifier call site. The `withIdentity()` enrichment step disappears entirely. No more two-state problem where the same type exists in both enriched and unenriched form.

The real catch was naming. The issue prescribed `action` as the first field — but `plannedAction.action()` is a tautology. The record IS the action. The engine's actual source code uses `description`, and every call site confirms it carries human-readable text: "File SAR report", "File resolution". Foundation-tier extraction is the one chance to get names right. Similarly, the engine's `context` field became `parameters` — the values are action arguments (`"accountId"`, `"amount"`), not ambient context.

## The structural enforcement

The key design decision: PlannedAction lives on `WorkerOutcome.Success`, not on `WorkerResult`. The engine currently enforces "no PlannedAction on non-success outcomes" with a runtime validation check in WorkerResult's compact constructor. The right design makes the invalid state unrepresentable:

```java
public sealed interface WorkerOutcome {
    record Success(PlannedAction plannedAction) implements WorkerOutcome {}
    record Declined(String reason) implements WorkerOutcome {}
    record Failed(String reason) implements WorkerOutcome {}
    record Expired(String reason) implements WorkerOutcome {}
}
```

PlannedAction can only exist inside a Success. The compiler enforces what the runtime check used to. The engine's `WorkerResultExpiredTest.expired_outcome_rejects_planned_action()` — a test that verifies the runtime throws on invalid combinations — becomes dead code. The scenario it tests is structurally impossible.

This also changes the access pattern. The engine currently does `workerResult.plannedAction()` — a direct accessor. After migration, every consumption site pattern-matches through the outcome: `outcome instanceof Success s ? s.plannedAction() : null`. More ceremony, but the ceremony is the point — you can't touch the action without first being in a success-handling branch.

## Where this leaves things

Seven commits, six files changed, 327 lines. The 0.2-SNAPSHOT is installed locally. Engine#543 can now consume it.

The worker module has its first real type, and the pattern is set: foundation-tier types are declarations, not execution context. They carry what the worker needs to say, stripped of what the orchestration tier needs to know. The separation that was implicit in the engine — PlannedAction created by the worker, enriched by the engine — is now structural. Two types in two tiers, each owning exactly the fields they should.
