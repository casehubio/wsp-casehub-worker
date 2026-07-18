---
layout: post
title: "Async workers and the PolicyEnforcer funeral"
date: 2026-07-17
type: phase-update
entry_type: note
subtype: diary
projects: [worker]
tags: [async, fault-tolerance, smallrye, mutiny]
series: issue-5-async-worker-function
---

*Part of a series on [#5 — async WorkerFunction](https://github.com/casehubio/casehub-worker/issues/5). Previous: [Removing the type bottleneck](2026-07-14-mdp01-removing-the-type-bottleneck.md).*

The original issue was straightforward: add a `WorkerFunction.Async<T>` variant so I/O-bound workers don't block the executor thread. The interesting part turned out to be everything around it.

## The async variant itself

`Async<T>` mirrors `Sync<T>` exactly — same `Class<T> inputType()`, same `fn()` accessor, different return type: `Function<T, CompletionStage<WorkerResult>>` instead of `Function<T, WorkerResult>`. CompletionStage stays in the API module because it's a JDK type — no framework coupling at the API tier.

The executor return type is a different question. `WorkerExecutor.execute()` now returns `Uni<WorkerResult>` — Mutiny's lazy reactive type. The runtime module already depends on Quarkus CDI and SmallRye FT, so Mutiny comes free. An adversarial design review pushed this change: the original spec had `CompletionStage` everywhere, but the reviewer was right that the runtime should speak the native reactive type. CompletionStage in the API, Uni in the runtime — each module uses what fits its dependency posture.

## PolicyEnforcer had to go

This is where the scope expanded in a way I'm glad it did. `PolicyEnforcer` was a hand-rolled retry/timeout implementation in `casehub-platform-governance` — `Thread.sleep()` between retries, `Future.get(timeout)` for deadline enforcement. It worked for sync, but it fundamentally can't compose with `CompletionStage` actions. The retry loop sleeps a thread. The timeout wraps a `Future`. Neither concept translates to non-blocking execution.

The question was whether to extend PolicyEnforcer with an async overload (cross-repo change), bypass it for async workers, or replace it entirely. SmallRye Fault Tolerance's `Guard` API made the answer obvious — it handles both sync and async actions natively, with retry, timeout, exponential backoff, and jitter built in. The replacement is about 15 lines of builder configuration.

The executor now has one code path: validate → schema check → lift to Uni → Guard → schema check output → exception mapping. Sync workers get wrapped in `Uni.createFrom().item()`, async workers in `Uni.createFrom().completionStage()`. Guard doesn't care which — it sees a `Uni` either way.

## OTel across the async boundary

The span lifecycle needed rethinking. The old try-finally pattern assumes the span and the execution live on the same thread. For async workers, the span must stay open until the `CompletionStage` completes — potentially on a different thread, potentially minutes later.

The fix separates two OTel concepts that the old code conflated: **Scope** (thread-local, opened and closed synchronously during dispatch) and **Span** (explicit lifecycle, closed in an `onTermination` callback on the Uni). For sync workers the callback fires immediately — same timing as try-finally. For async workers the span stays open and tracks actual execution duration.

## What surprised me

Guard handles sync function timeout correctly without virtual thread offload. The design review predicted that sync workers would need `runSubscriptionOn(virtualThreads)` so Guard's timeout race could fire — the theory being that a blocking sync supplier would prevent the timeout timer from interrupting it. The tests passed without it. Guard's internal implementation apparently handles this.

The `Guard.create()` SPI issue was less pleasant. It compiles against the API artifact but fails at runtime in plain JUnit tests with a misleading `NoClassDefFoundError`. The fix is a test-scope dependency on `smallrye-fault-tolerance-standalone` — not documented anywhere obvious.

With async execution in place, the engine can now build an `AsyncWorkerFunctionHandler` to dispatch non-blocking workers through the SPI. That's engine#726 territory — a separate concern, but this work unblocks it.
