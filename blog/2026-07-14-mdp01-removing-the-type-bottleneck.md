---
layout: post
title: "Removing the Type Bottleneck"
date: 2026-07-14
type: phase-update
entry_type: note
subtype: diary
projects: [CaseHub Worker]
tags: [context-bridge, type-safety, api-design]
---

# Removing the Type Bottleneck

I came in to do engine convergence — making the engine's sync handler delegate
to the worker executor instead of reimplementing timeout, retry, and error
conversion. The issue had been sitting there since the capability-aware execution
work landed.

What I found was more interesting than what I expected.

## The audit that changed the scope

The worker executor's API takes `Map<String, Object>` input. Meanwhile, the
ContextBridge protocol — landed on the engine three days earlier — had widened
the entire engine pipeline to accept typed input via `WorkerFunction<T>`. POJOs,
JsonNodes, live-view contexts. The bridge resolves the type upstream, the
function declares its type via `inputType()`, and `Sync<T>.fn()` accepts `T`.
But the worker executor in the middle casts everything to
`Function<Map<String, Object>, WorkerResult>`, collapsing the type information.

The executor was the one place in the pipeline that still forced Map.

I traced the ContextBridge propagation across the engine codebase. Five open
issues (#689–#693) each extend the typed protocol to a different boundary —
WorkItem, SubCase, Signal, Connector, in-process composition. The worker
executor widening is the foundation layer: until the executor accepts typed
input, the engine can't delegate to it, and the typed pipeline has a hard
bottleneck at the worker boundary.

## Re-scoping

The original issue (#10) was filed as "engine convergence" — the engine delegates
to the worker executor. But the engine-side delegation depends on the worker
executor accepting typed input first. I re-scoped #10 to be the worker-side
widening and created engine#726 for the delegation work. The dependency chain:

```
Layer 1: worker#10 (this branch) — widen executor to Object
Layer 2: engine#726 — Sync handler delegates to worker executor
Layer 3: engine#693 — WorkerRuntime + sequence() typed composition
Layer 4: engine#689-692 — remaining boundaries (independent)
```

## The Map escape hatch that wasn't

The initial design had a backward-compatibility escape hatch: accept Map input
even for POJO-typed functions, since Java erasure means the function receives
it without a compile-time error. The adversarial review killed it.

The reviewer pointed out that lambda bridge methods include a `checkcast`
instruction. When javac compiles `Function<AmlTransaction, WorkerResult>`,
the generated bridge contains `checkcast AmlTransaction`. A Map passed to
that function throws `ClassCastException` *inside the lambda body* — not at
`apply()`. The call site gets a stack trace pointing at the lambda
implementation, not the incorrect caller. Strict `inputType().isInstance()`
validation at the call site produces a clear `IllegalArgumentException` with
both expected and actual types.

The escape hatch wasn't just unnecessary — it produced worse diagnostics than
no escape hatch at all.

## What landed

Four commits on `issue-10-engine-convergence`:

1. `SchemaValidator` widened from `Map<String, Object>` to `Object` —
   `valueToTree()` already handles any Jackson-serializable type
2. `WorkerExecutor.execute()` widened to `Object` input with strict
   `isInstance()` validation; `DefaultWorkerExecutor` uses raw-type
   `fn.apply(input)` instead of Map-casted invocation
3. `MockWorkerExecutor` mirrors the same validation and raw-type invocation
4. `DESIGN.md` updated with the full programming errors table

The worker executor is no longer a type bottleneck. The ContextBridge
pipeline can flow typed input end-to-end from bridge resolution through
to function invocation.

Engine#726 is unblocked — the Sync handler can now inject the worker executor
and delegate typed input to it.
