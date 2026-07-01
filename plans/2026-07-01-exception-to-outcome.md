# Exception-to-Outcome Conversion Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Close the `WorkerExecutor` result model — all worker-level exceptions become `WorkerOutcome` values; only infrastructure signals propagate.

**Architecture:** Two-repo change. First, add typed exception subclasses to `casehub-platform-governance` and fix `DefaultPolicyEnforcer` to use them. Then update `casehub-worker`'s `DefaultWorkerExecutor` to catch by type and map to outcomes. TDD throughout — tests first, then implementation.

**Tech Stack:** Java 21, Quarkus 3.32.2, JUnit 5, AssertJ, OpenTelemetry API

## Global Constraints

- **Spec:** `casehub-worker/docs/specs/2026-07-01-exception-to-outcome-design.md` — the authoritative design. All code must match the spec exactly.
- **Platform repo:** `/Users/mdproctor/claude/casehub/platform` — multi-module Maven, `governance/` submodule
- **Worker repo:** `/Users/mdproctor/claude/casehub/worker` — multi-module Maven, `api/`, `runtime/`, `testing/`
- **Build:** `mvn -f <repo>/pom.xml clean install` — platform first, then worker
- **Issue:** casehubio/casehub-worker#8, epic #3. All commits reference `Refs #8`.
- **Exception package:** `io.casehub.platform.governance` (same package as `PolicyEnforcementException`)
- **No backwards compatibility shims.** Breaking changes are intentional.

---

### Task 1: Typed exception subclasses in platform-governance

**Files:**
- Create: `platform/governance/src/main/java/io/casehub/platform/governance/TimeoutPolicyException.java`
- Create: `platform/governance/src/main/java/io/casehub/platform/governance/InterruptedPolicyException.java`
- Create: `platform/governance/src/main/java/io/casehub/platform/governance/RetryExhaustedException.java`
- Modify: `platform/governance/src/test/java/io/casehub/platform/governance/DefaultPolicyEnforcerTest.java`
- Modify: `platform/governance/src/main/java/io/casehub/platform/governance/DefaultPolicyEnforcer.java`

**Interfaces:**
- Produces: `TimeoutPolicyException extends PolicyEnforcementException` — single-arg `(String message)` constructor
- Produces: `InterruptedPolicyException extends PolicyEnforcementException` — two-arg `(String message, Throwable cause)` constructor
- Produces: `RetryExhaustedException extends PolicyEnforcementException` — two-arg `(String message, Throwable cause)` constructor

- [ ] **Step 1: Write failing tests for typed exceptions**

Add these tests to `DefaultPolicyEnforcerTest.java`. They will fail because `DefaultPolicyEnforcer` still throws the base `PolicyEnforcementException`.

Update the existing `execute_exhaustsRetries_throws` test to assert `RetryExhaustedException`:

```java
@Test
void execute_exhaustsRetries_throwsRetryExhaustedException() {
    ExecutionPolicy policy = new ExecutionPolicy(null, new RetryPolicy(2, 10));

    assertThatThrownBy(() -> enforcer.execute(policy, () -> {
        throw new RuntimeException("permanent");
    }))
        .isInstanceOf(RetryExhaustedException.class)
        .hasMessageContaining("2 attempts")
        .hasCauseInstanceOf(RuntimeException.class);
}
```

Update the existing `execute_timeout_failsIfExceeded` test — timeout should now be `TimeoutPolicyException` at the top level (no double-wrapping):

```java
@Test
void execute_timeout_throwsTimeoutPolicyException() {
    ExecutionPolicy policy = new ExecutionPolicy(50, new RetryPolicy(1, 0));

    assertThatThrownBy(() -> enforcer.execute(policy, () -> {
        try {
            Thread.sleep(200);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        return "late";
    }))
        .isInstanceOf(TimeoutPolicyException.class)
        .hasMessageContaining("timed out");
}
```

Add new test for interrupt during execution:

```java
@Test
void execute_interrupted_throwsInterruptedPolicyException() {
    ExecutionPolicy policy = new ExecutionPolicy(5000, new RetryPolicy(1, 0));

    Thread.currentThread().interrupt();

    assertThatThrownBy(() -> enforcer.execute(policy, () -> "result"))
        .isInstanceOf(InterruptedPolicyException.class)
        .hasMessageContaining("Interrupted");

    assertThat(Thread.currentThread().isInterrupted()).isTrue();
    Thread.interrupted(); // clear flag for other tests
}
```

Add test for interrupt during backoff sleep:

```java
@Test
void execute_interruptDuringSleep_throwsInterruptedPolicyException() {
    AtomicInteger attempts = new AtomicInteger(0);
    ExecutionPolicy policy = new ExecutionPolicy(null, new RetryPolicy(3, 5000));

    Thread testThread = Thread.currentThread();
    Thread interruptor = new Thread(() -> {
        try { Thread.sleep(50); } catch (InterruptedException ignored) {}
        testThread.interrupt();
    });
    interruptor.start();

    assertThatThrownBy(() -> enforcer.execute(policy, () -> {
        attempts.incrementAndGet();
        throw new RuntimeException("fail");
    }))
        .isInstanceOf(InterruptedPolicyException.class)
        .hasMessageContaining("Interrupted during backoff");

    assertThat(attempts.get()).isEqualTo(1);
    Thread.interrupted(); // clear flag
}
```

Add test for interrupt in no-timeout path:

```java
@Test
void execute_interruptWithoutTimeout_throwsInterruptedPolicyException() {
    ExecutionPolicy policy = new ExecutionPolicy(null, new RetryPolicy(3, 10));

    Thread.currentThread().interrupt();

    assertThatThrownBy(() -> enforcer.execute(policy, () -> {
        throw new RuntimeException("fail while interrupted");
    }))
        .isInstanceOf(InterruptedPolicyException.class)
        .hasMessageContaining("Interrupted");

    Thread.interrupted(); // clear flag
}
```

Remove the old `execute_exhaustsRetries_throws` and `execute_timeout_failsIfExceeded` tests (replaced by the new ones above).

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn -f /Users/mdproctor/claude/casehub/platform/governance/pom.xml test`
Expected: compilation failures — `RetryExhaustedException`, `TimeoutPolicyException`, `InterruptedPolicyException` do not exist yet.

- [ ] **Step 3: Create the three exception subclasses**

`TimeoutPolicyException.java`:
```java
package io.casehub.platform.governance;

public class TimeoutPolicyException extends PolicyEnforcementException {
    public TimeoutPolicyException(String message) { super(message); }
}
```

`InterruptedPolicyException.java`:
```java
package io.casehub.platform.governance;

public class InterruptedPolicyException extends PolicyEnforcementException {
    public InterruptedPolicyException(String message, Throwable cause) { super(message, cause); }
}
```

`RetryExhaustedException.java`:
```java
package io.casehub.platform.governance;

public class RetryExhaustedException extends PolicyEnforcementException {
    public RetryExhaustedException(String message, Throwable cause) { super(message, cause); }
}
```

- [ ] **Step 4: Update DefaultPolicyEnforcer throw sites**

Replace the full `DefaultPolicyEnforcer` with the spec's code. Three changes:

**`executeWithTimeout()`** — typed throws + no-timeout interrupt detection:
```java
private <T> T executeWithTimeout(Integer timeoutMs, Supplier<T> action) {
    if (timeoutMs == null) {
        try {
            return action.get();
        } catch (Exception e) {
            if (Thread.currentThread().isInterrupted()) {
                throw new InterruptedPolicyException("Interrupted during execution", e);
            }
            throw e;
        }
    }
    Callable<T> callable = action::get;
    Future<T> future = timeoutExecutor.submit(callable);
    try {
        return future.get(timeoutMs, TimeUnit.MILLISECONDS);
    } catch (TimeoutException e) {
        future.cancel(true);
        throw new TimeoutPolicyException("Action timed out after " + timeoutMs + "ms");
    } catch (ExecutionException e) {
        Throwable cause = e.getCause();
        if (cause instanceof RuntimeException re) throw re;
        throw new PolicyEnforcementException("Action failed", cause);
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
        throw new InterruptedPolicyException("Interrupted during execution", e);
    }
}
```

**`sleep()`** — throw on interrupt:
```java
private void sleep(long ms) {
    try {
        Thread.sleep(ms);
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
        throw new InterruptedPolicyException("Interrupted during backoff", e);
    }
}
```

**`execute()` retry loop** — break on interrupt, re-throw policy exceptions directly:
```java
Exception lastException = null;
for (int attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
        return executeWithTimeout(policy.timeoutMs(), action);
    } catch (Exception e) {
        lastException = e;
        if (e instanceof InterruptedPolicyException) break;
        if (attempt < maxAttempts) {
            sleep(computeDelay(delayMs, backoff, attempt, maxDelayMs));
        }
    }
}
if (lastException instanceof PolicyEnforcementException pe) {
    throw pe;
}
throw new RetryExhaustedException(
    "All " + maxAttempts + " attempts failed", lastException);
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn -f /Users/mdproctor/claude/casehub/platform/governance/pom.xml test`
Expected: all tests pass.

- [ ] **Step 6: Run full platform build and install**

Run: `mvn -f /Users/mdproctor/claude/casehub/platform/pom.xml clean install`
Expected: BUILD SUCCESS. This installs the 0.2-SNAPSHOT with typed exceptions to the local Maven repo.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/platform add \
  governance/src/main/java/io/casehub/platform/governance/TimeoutPolicyException.java \
  governance/src/main/java/io/casehub/platform/governance/InterruptedPolicyException.java \
  governance/src/main/java/io/casehub/platform/governance/RetryExhaustedException.java \
  governance/src/main/java/io/casehub/platform/governance/DefaultPolicyEnforcer.java \
  governance/src/test/java/io/casehub/platform/governance/DefaultPolicyEnforcerTest.java
```

Commit message:
```
refactor: typed PolicyEnforcementException subclasses — TimeoutPolicyException, InterruptedPolicyException, RetryExhaustedException

Refs casehubio/casehub-worker#8
```

---

### Task 2: DefaultWorkerExecutor exception-to-outcome conversion

**Files:**
- Modify: `worker/runtime/src/main/java/io/casehub/worker/runtime/DefaultWorkerExecutor.java`
- Modify: `worker/runtime/src/test/java/io/casehub/worker/runtime/WorkerExecutorTest.java`

**Interfaces:**
- Consumes: `TimeoutPolicyException`, `InterruptedPolicyException`, `RetryExhaustedException` from Task 1
- Consumes: `WorkerResult.expired(String)`, `WorkerResult.failed(String)` from existing worker-api
- Produces: Updated `DefaultWorkerExecutor.execute()` that returns `WorkerResult` for all worker-level conditions

- [ ] **Step 1: Write failing tests**

Update the existing `execute_exhaustsRetries_throwsPolicyException` → now expects `Failed` outcome:

```java
@Test
void execute_exhaustsRetries_returnsFailed() {
    Worker worker = Worker.builder()
        .name("broken")
        .capabilityName("fail")
        .function(new WorkerFunction.Sync(input -> { throw new RuntimeException("permanent"); }))
        .executionPolicy(new ExecutionPolicy(null, new RetryPolicy(2, 10)))
        .build();

    WorkerResult result = executor.execute(worker, Map.of());
    assertThat(result.outcome()).isInstanceOf(WorkerOutcome.Failed.class);
    assertThat(((WorkerOutcome.Failed) result.outcome()).reason()).isEqualTo("permanent");
}
```

Add test for single exception without retry:

```java
@Test
void execute_workerThrows_returnsFailed() {
    Worker worker = Worker.builder()
        .name("throws")
        .capabilityName("boom")
        .function(new WorkerFunction.Sync(input -> { throw new IllegalStateException("bad state"); }))
        .build();

    WorkerResult result = executor.execute(worker, Map.of());
    assertThat(result.outcome()).isInstanceOf(WorkerOutcome.Failed.class);
    assertThat(((WorkerOutcome.Failed) result.outcome()).reason()).isEqualTo("bad state");
}
```

Add test for timeout producing Expired:

```java
@Test
void execute_timeout_returnsExpired() {
    Worker worker = Worker.builder()
        .name("slow")
        .capabilityName("crawl")
        .function(new WorkerFunction.Sync(input -> {
            try { Thread.sleep(500); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
            return WorkerResult.of(Map.of());
        }))
        .executionPolicy(new ExecutionPolicy(50, new RetryPolicy(1, 0)))
        .build();

    WorkerResult result = executor.execute(worker, Map.of());
    assertThat(result.outcome()).isInstanceOf(WorkerOutcome.Expired.class);
    assertThat(((WorkerOutcome.Expired) result.outcome()).reason()).contains("timed out");
}
```

Add test for null exception message fallback:

```java
@Test
void execute_workerThrowsNullMessage_returnsFailedWithClassName() {
    Worker worker = Worker.builder()
        .name("npe")
        .capabilityName("null")
        .function(new WorkerFunction.Sync(input -> { throw new NullPointerException(); }))
        .build();

    WorkerResult result = executor.execute(worker, Map.of());
    assertThat(result.outcome()).isInstanceOf(WorkerOutcome.Failed.class);
    assertThat(((WorkerOutcome.Failed) result.outcome()).reason()).isEqualTo("java.lang.NullPointerException");
}
```

Remove the old `execute_exhaustsRetries_throwsPolicyException` test.

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn -f /Users/mdproctor/claude/casehub/worker/runtime/pom.xml test`
Expected: `execute_exhaustsRetries_returnsFailed` fails (still throws exception instead of returning Failed). Others fail similarly.

- [ ] **Step 3: Update DefaultWorkerExecutor catch block**

Replace the catch block in `execute()` with the spec's type-based dispatch:

```java
catch (TimeoutPolicyException e) {
    span.addEvent("worker.timeout", Attributes.of(
        AttributeKey.stringKey("timeout.message"), e.getMessage()));
    WorkerResult result = WorkerResult.expired(e.getMessage());
    span.setAttribute(AttributeKey.stringKey("worker.outcome"),
        result.outcome().getClass().getSimpleName());
    return result;
}
catch (InterruptedPolicyException e) {
    span.setStatus(StatusCode.ERROR, e.getMessage());
    span.recordException(e);
    throw e;
}
catch (Exception e) {
    span.setStatus(StatusCode.ERROR, e.getMessage());
    span.recordException(e);
    Throwable root = e.getCause() != null ? e.getCause() : e;
    String message = root.getMessage();
    if (message == null) message = root.getClass().getName();
    WorkerResult result = WorkerResult.failed(message);
    span.setAttribute(AttributeKey.stringKey("worker.outcome"),
        result.outcome().getClass().getSimpleName());
    return result;
}
```

Add the required imports:
```java
import io.casehub.platform.governance.TimeoutPolicyException;
import io.casehub.platform.governance.InterruptedPolicyException;
import io.opentelemetry.api.common.Attributes;
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn -f /Users/mdproctor/claude/casehub/worker/runtime/pom.xml test`
Expected: all tests pass.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/worker add \
  runtime/src/main/java/io/casehub/worker/runtime/DefaultWorkerExecutor.java \
  runtime/src/test/java/io/casehub/worker/runtime/WorkerExecutorTest.java
```

Commit message:
```
feat: exception-to-outcome conversion in DefaultWorkerExecutor

TimeoutPolicyException → Expired, RetryExhaustedException → Failed,
raw exceptions → Failed. InterruptedPolicyException propagates.

Refs #8
```

---

### Task 3: WorkerExecutor contract + MockWorkerExecutor alignment

**Files:**
- Modify: `worker/runtime/src/main/java/io/casehub/worker/runtime/WorkerExecutor.java`
- Modify: `worker/testing/src/main/java/io/casehub/worker/testing/MockWorkerExecutor.java`
- Modify: `worker/testing/src/test/java/io/casehub/worker/testing/MockWorkerExecutorTest.java`

**Interfaces:**
- Consumes: `WorkerResult.failed(String)` from worker-api
- Produces: Updated `MockWorkerExecutor.execute()` that converts exceptions to `Failed` outcomes

- [ ] **Step 1: Write failing test for MockWorkerExecutor**

Add to `MockWorkerExecutorTest.java`:

```java
@Test
void execute_workerThrows_returnsFailed() {
    MockWorkerExecutor mockExecutor = new MockWorkerExecutor();
    Worker worker = TestWorkerBuilder.sync("throws",
        input -> { throw new RuntimeException("mock failure"); });

    WorkerResult result = mockExecutor.execute(worker, Map.of());
    assertThat(result.outcome()).isInstanceOf(WorkerOutcome.Failed.class);
    assertThat(((WorkerOutcome.Failed) result.outcome()).reason()).isEqualTo("mock failure");
    assertThat(mockExecutor.executionCount()).isEqualTo(1);
}
```

Add test for null message:

```java
@Test
void execute_workerThrowsNullMessage_returnsFailedWithClassName() {
    MockWorkerExecutor mockExecutor = new MockWorkerExecutor();
    Worker worker = TestWorkerBuilder.sync("npe",
        input -> { throw new NullPointerException(); });

    WorkerResult result = mockExecutor.execute(worker, Map.of());
    assertThat(result.outcome()).isInstanceOf(WorkerOutcome.Failed.class);
    assertThat(((WorkerOutcome.Failed) result.outcome()).reason())
        .isEqualTo("java.lang.NullPointerException");
}
```

Add necessary imports:
```java
import io.casehub.worker.api.WorkerOutcome;
import io.casehub.worker.api.WorkerResult;
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn -f /Users/mdproctor/claude/casehub/worker/testing/pom.xml test`
Expected: tests fail — `MockWorkerExecutor` currently throws instead of returning `Failed`.

- [ ] **Step 3: Add Javadoc to WorkerExecutor interface**

```java
package io.casehub.worker.runtime;

import io.casehub.worker.api.Worker;
import io.casehub.worker.api.WorkerResult;

import java.util.Map;

public interface WorkerExecutor {
    /**
     * Executes the given worker with policy enforcement.
     *
     * <p>All worker-level conditions (function exceptions, retry exhaustion,
     * timeout) are returned as {@link WorkerResult} outcomes. Only infrastructure
     * signals (thread interrupt, JVM errors) propagate as exceptions.
     */
    WorkerResult execute(Worker worker, Map<String, Object> input);
}
```

- [ ] **Step 4: Update MockWorkerExecutor with try-catch**

```java
@Override
public WorkerResult execute(Worker worker, Map<String, Object> input) {
    executionCount.incrementAndGet();
    lastWorkerName.set(worker.name());
    try {
        return ((io.casehub.worker.api.WorkerFunction.Sync) worker.function()).fn().apply(input);
    } catch (Exception e) {
        String message = e.getMessage();
        if (message == null) message = e.getClass().getName();
        return WorkerResult.failed(message);
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn -f /Users/mdproctor/claude/casehub/worker/testing/pom.xml test`
Expected: all tests pass.

- [ ] **Step 6: Run full worker build**

Run: `mvn -f /Users/mdproctor/claude/casehub/worker/pom.xml clean install`
Expected: BUILD SUCCESS across all modules.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/worker add \
  runtime/src/main/java/io/casehub/worker/runtime/WorkerExecutor.java \
  testing/src/main/java/io/casehub/worker/testing/MockWorkerExecutor.java \
  testing/src/test/java/io/casehub/worker/testing/MockWorkerExecutorTest.java
```

Commit message:
```
feat: WorkerExecutor contract + MockWorkerExecutor exception-to-outcome alignment

Refs #8
```
