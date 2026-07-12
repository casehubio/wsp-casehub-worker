---
layout: post
title: "The branch that nobody owned"
date: 2026-07-13
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-worker]
tags: [worker, cross-repo, exception-handling, branch-hygiene]
---

# The branch that nobody owned

I came in to land issue #4 (WorkerContext) and found something else waiting first.

A branch called `issue-203-context-bridge-protocol` had been sitting in the worker repo since July 10. One commit — parameterising `WorkerFunction<T>` with a type token and adding `Builder.fn().apply()` for typed input. Created by an engine session doing cross-repo work for engine#203. The engine session correctly didn't merge to worker main (the rule is clear: never merge to main in a peer repo outside of work-end). But it also didn't update worker's HANDOFF.md to say the branch existed. So subsequent worker sessions had no idea it was there.

Meanwhile, the engine's main branch was already compiling against the new parameterised API. The dependency was live, the artifact wasn't published, and nobody in the worker repo knew.

I ran the tests before landing. Three failures — all in `DefaultWorkerExecutor`'s exception handling, none related to the parameterisation itself. The engine session had also refactored the catch blocks while it was in the file: removed root cause unwrapping, removed the null message fallback to class name, removed `InterruptedPolicyException` propagation. All three were deliberate design decisions from issue #8 (exception-to-outcome conversion) a week earlier. The refactoring looked like cleanup in the context of the larger diff — shorter code, fewer catch blocks. The tests told a different story: "expected `permanent`, got `All 2 attempts failed`."

The fix was mechanical — restore the three behaviours. Tests green, squashed, pushed, CI published.

Then I checked blocks. Same pattern but wider: 14 issue branches, all stamped closed, all with their code already on main via squash. Four had old-format stamps (missing the landing SHA), one was open but identical to main. Updated the stamps, closed the orphan. The blocks repo is clean now — every branch verifiably landed.

The failure mode is specific and repeatable: session A creates a branch in repo B, correctly doesn't merge, but doesn't tell repo B's handoff. Session C picks up repo B with no knowledge the branch exists. The branch sits until something breaks or someone audits. The cross-repo handoff rule exists for exactly this case — it just wasn't followed.
