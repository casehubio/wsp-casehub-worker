# Capability-Aware Execution Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make `WorkerExecutor.execute()` capability-aware — the executor knows which capability is being invoked, validates the relationship, and traces at capability granularity.

**Architecture:** Change `execute(Worker, Map)` to `execute(Worker, Capability, Map)`. Add validation guards (null check, capability membership, Sync-only) before `PolicyEnforcer`. Add OTel span attribute for capability name. Update MockWorkerExecutor with same validation + capability tracking.

**Tech Stack:** Java 21, Quarkus 3.32.2, OpenTelemetry API, JUnit 5, AssertJ

## Global Constraints

- `casehub-worker-api` is Tier 1 pure-Java — no Quarkus, no JPA
- `casehub-worker-runtime` is Tier 3 — Quarkus, CDI, OTel
- `casehub-worker-testing` — `@DefaultBean` mock, test utilities
- Build: `mvn -f /Users/mdproctor/claude/casehub/worker/pom.xml clean install`
- Test: `mvn -f /Users/mdproctor/claude/casehub/worker/pom.xml test`
- Spec: `docs/specs/2026-07-02-capability-aware-execution-design.md`
- Issue: casehubio/casehub-worker#9

---

### Task 1: TestWorkerBuilder — syncWithCapability convenience

**Files:**
- Modify: `testing/src/main/java/io/casehub/worker/testing/TestWorkerBuilder.java`
- Create: `testing/src/test/java/io/casehub/worker/testing/TestWorkerBuilderTest.java`

**Interfaces:**
- Consumes: `Worker.builder()`, `Capability.of()` from `casehub-worker-api`
- Produces: `TestWorkerBuilder.WorkerWithCapability` record, `TestWorkerBuilder.syncWithCapability(String, Function)` — used by Task 2 and Task 3 tests

- [ ] **Step 1: Write the failing test**

Create `testing/src/test/java/io/casehub/worker/testing/TestWorkerBuilderTest.java`:

```java
package io.casehub.worker.testing;

import io.casehub.worker.api.WorkerOutcome;
import io.casehub.worker.api.WorkerResult;
import org.junit.jupiter.api.Test;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class TestWorkerBuilderTest {

    @Test
    void syncWithCapability_createsMatchingPair() {
        var wc = TestWorkerBuilder.syncWithCapability("greet",
            input -> WorkerResult.of(Map.of("greeting", "hello")));

        assertThat(wc.worker().name()).isEqualTo("greet");
        assertThat(wc.worker().capabilityNames()).containsExactly("greet");
        assertThat(wc.capability().name()).isEqualTo("greet");
        assertThat(wc.capability().inputSchema()).isEqualTo("{}");
        assertThat(wc.capability().outputSchema()).isEqualTo("{}");
    }

    @Test
    void syncWithCapability_functionExecutes() {
        var wc = TestWorkerBuilder.syncWithCapability("echo",
            input -> WorkerResult.of(input));

        var result = ((io.casehub.worker.api.WorkerFunction.Sync) wc.worker().function())
            .fn().apply(Map.of("key", "value"));
        assertThat(result.outcome()).isInstanceOf(WorkerOutcome.Success.class);
        assertThat(result.output()).containsEntry("key", "value");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn -f /Users/mdproctor/claude/casehub/worker/testing/pom.xml test -pl . -Dtest=TestWorkerBuilderTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: Compilation failure — `syncWithCapability` does not exist

- [ ] **Step 3: Implement WorkerWithCapability and syncWithCapability**

Modify `testing/src/main/java/io/casehub/worker/testing/TestWorkerBuilder.java`:

```java
package io.casehub.worker.testing;

import io.casehub.worker.api.Capability;
import io.casehub.worker.api.Worker;
import io.casehub.worker.api.WorkerResult;

import java.util.Map;
import java.util.function.Function;

public final class TestWorkerBuilder {
    private TestWorkerBuilder() {}

    public record WorkerWithCapability(Worker worker, Capability capability) {}

    public static Worker sync(String name, Function<Map<String, Object>, WorkerResult> fn) {
        return Worker.builder()
            .name(name)
            .capabilityName(name)
            .function(fn)
            .build();
    }

    public static WorkerWithCapability syncWithCapability(String name,
            Function<Map<String, Object>, WorkerResult> fn) {
        Worker worker = Worker.builder()
            .name(name)
            .capabilityName(name)
            .function(fn)
            .build();
        Capability capability = Capability.of(name, "{}", "{}");
        return new WorkerWithCapability(worker, capability);
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn -f /Users/mdproctor/claude/casehub/worker/testing/pom.xml test -pl . -Dtest=TestWorkerBuilderTest`
Expected: 2 tests PASS

- [ ] **Step 5: Commit**

```
feat(testing): add TestWorkerBuilder.syncWithCapability convenience

Produces a Worker + matching Capability pair for tests that need both.
Prerequisite for capability-aware executor (#9).

Refs #9
```

---

### Task 2: Capability-aware WorkerExecutor — interface + implementations + test migration

**Files:**
- Modify: `runtime/src/main/java/io/casehub/worker/runtime/WorkerExecutor.java`
- Modify: `runtime/src/main/java/io/casehub/worker/runtime/DefaultWorkerExecutor.java`
- Modify: `testing/src/main/java/io/casehub/worker/testing/MockWorkerExecutor.java`
- Modify: `runtime/src/test/java/io/casehub/worker/runtime/WorkerExecutorTest.java`
- Modify: `testing/src/test/java/io/casehub/worker/testing/MockWorkerExecutorTest.java`

**Interfaces:**
- Consumes: `Worker`, `Capability`, `WorkerFunction`, `WorkerResult` from `casehub-worker-api`; `PolicyEnforcer` from `casehub-platform-governance`; `TestWorkerBuilder.syncWithCapability()` from Task 1
- Produces: `WorkerExecutor.execute(Worker, Capability, Map)` — the new API surface

- [ ] **Step 1: Change the WorkerExecutor interface**

Replace `runtime/src/main/java/io/casehub/worker/runtime/WorkerExecutor.java` with:

```java
package io.casehub.worker.runtime;

import io.casehub.worker.api.Capability;
import io.casehub.worker.api.Worker;
import io.casehub.worker.api.WorkerResult;

import java.util.Map;

public interface WorkerExecutor {
    /**
     * Executes the given worker for the specified capability with policy enforcement.
     *
     * <p>Three exception categories:
     * <ul>
     *   <li><b>Worker-level conditions</b> (function exceptions, retry exhaustion,
     *       timeout) — returned as {@link WorkerResult} outcomes. Never propagate.</li>
     *   <li><b>Programming errors</b> (null capability, capability not in worker,
     *       non-Sync function) — propagate as exceptions ({@code NullPointerException},
     *       {@code IllegalArgumentException}, {@code UnsupportedOperationException}).
     *       These are call-site bugs, not execution outcomes.</li>
     *   <li><b>Infrastructure signals</b> (thread interrupt, JVM errors) — propagate
     *       as exceptions.</li>
     * </ul>
     */
    WorkerResult execute(Worker worker, Capability capability, Map<String, Object> input);
}
```

- [ ] **Step 2: Update DefaultWorkerExecutor**

Replace `runtime/src/main/java/io/casehub/worker/runtime/DefaultWorkerExecutor.java` with:

```java
package io.casehub.worker.runtime;

import io.casehub.platform.governance.PolicyEnforcer;
import io.casehub.platform.governance.TimeoutPolicyException;
import io.casehub.platform.governance.InterruptedPolicyException;
import io.casehub.worker.api.Capability;
import io.casehub.worker.api.Worker;
import io.casehub.worker.api.WorkerFunction;
import io.casehub.worker.api.WorkerResult;
import io.opentelemetry.api.GlobalOpenTelemetry;
import io.opentelemetry.api.common.AttributeKey;
import io.opentelemetry.api.common.Attributes;
import io.opentelemetry.api.trace.Span;
import io.opentelemetry.api.trace.StatusCode;
import io.opentelemetry.context.Scope;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.util.Map;
import java.util.Objects;

@ApplicationScoped
public class DefaultWorkerExecutor implements WorkerExecutor {

    private static final String INSTRUMENTATION_NAME = "io.casehub.worker";

    private final PolicyEnforcer policyEnforcer;

    @Inject
    public DefaultWorkerExecutor(PolicyEnforcer policyEnforcer) {
        this.policyEnforcer = policyEnforcer;
    }

    @Override
    public WorkerResult execute(Worker worker, Capability capability, Map<String, Object> input) {
        Objects.requireNonNull(capability, "capability");
        if (!worker.capabilityNames().contains(capability.name())) {
            throw new IllegalArgumentException(
                "Capability '" + capability.name() + "' not in worker '"
                    + worker.name() + "' capabilities: " + worker.capabilityNames());
        }
        if (!(worker.function() instanceof WorkerFunction.Sync sync)) {
            throw new UnsupportedOperationException(
                "DefaultWorkerExecutor supports Sync functions only, got: "
                    + worker.function().getClass().getName());
        }

        Span span = GlobalOpenTelemetry.getTracer(INSTRUMENTATION_NAME)
            .spanBuilder("worker.execute")
            .setAttribute(AttributeKey.stringKey("worker.name"), worker.name())
            .setAttribute(AttributeKey.stringKey("worker.capability"), capability.name())
            .startSpan();
        try (Scope ignored = span.makeCurrent()) {
            WorkerResult result = policyEnforcer.execute(
                worker.executionPolicy(),
                () -> sync.fn().apply(input));
            span.setAttribute(AttributeKey.stringKey("worker.outcome"),
                result.outcome().getClass().getSimpleName());
            return result;
        } catch (TimeoutPolicyException e) {
            span.addEvent("worker.timeout", Attributes.of(
                AttributeKey.stringKey("timeout.message"), e.getMessage()));
            WorkerResult result = WorkerResult.expired(e.getMessage());
            span.setAttribute(AttributeKey.stringKey("worker.outcome"),
                result.outcome().getClass().getSimpleName());
            return result;
        } catch (InterruptedPolicyException e) {
            span.setStatus(StatusCode.ERROR, e.getMessage());
            span.recordException(e);
            throw e;
        } catch (Exception e) {
            span.setStatus(StatusCode.ERROR, e.getMessage());
            span.recordException(e);
            Throwable root = e.getCause() != null ? e.getCause() : e;
            String message = root.getMessage();
            if (message == null) message = root.getClass().getName();
            WorkerResult result = WorkerResult.failed(message);
            span.setAttribute(AttributeKey.stringKey("worker.outcome"),
                result.outcome().getClass().getSimpleName());
            return result;
        } finally {
            span.end();
        }
    }
}
```

- [ ] **Step 3: Update MockWorkerExecutor**

Replace `testing/src/main/java/io/casehub/worker/testing/MockWorkerExecutor.java` with:

```java
package io.casehub.worker.testing;

import io.casehub.worker.api.Capability;
import io.casehub.worker.api.Worker;
import io.casehub.worker.api.WorkerResult;
import io.casehub.worker.runtime.WorkerExecutor;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;

import java.util.Map;
import java.util.Objects;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.atomic.AtomicReference;

@DefaultBean
@ApplicationScoped
public class MockWorkerExecutor implements WorkerExecutor {
    private final AtomicInteger executionCount = new AtomicInteger(0);
    private final AtomicReference<String> lastWorkerName = new AtomicReference<>();
    private final AtomicReference<String> lastCapabilityName = new AtomicReference<>();

    @Override
    public WorkerResult execute(Worker worker, Capability capability, Map<String, Object> input) {
        Objects.requireNonNull(capability, "capability");
        if (!worker.capabilityNames().contains(capability.name())) {
            throw new IllegalArgumentException(
                "Capability '" + capability.name() + "' not in worker '"
                    + worker.name() + "' capabilities: " + worker.capabilityNames());
        }
        executionCount.incrementAndGet();
        lastWorkerName.set(worker.name());
        lastCapabilityName.set(capability.name());
        try {
            return ((io.casehub.worker.api.WorkerFunction.Sync) worker.function()).fn().apply(input);
        } catch (Exception e) {
            String message = e.getMessage();
            if (message == null) message = e.getClass().getName();
            return WorkerResult.failed(message);
        }
    }

    public int executionCount() { return executionCount.get(); }
    public String lastWorkerName() { return lastWorkerName.get(); }
    public String lastCapabilityName() { return lastCapabilityName.get(); }
    public void reset() {
        executionCount.set(0);
        lastWorkerName.set(null);
        lastCapabilityName.set(null);
    }
}
```

- [ ] **Step 4: Migrate existing WorkerExecutorTest**

Replace `runtime/src/test/java/io/casehub/worker/runtime/WorkerExecutorTest.java` with:

```java
package io.casehub.worker.runtime;

import io.casehub.platform.api.governance.ExecutionPolicy;
import io.casehub.platform.api.governance.RetryPolicy;
import io.casehub.platform.governance.DefaultPolicyEnforcer;

import io.casehub.worker.api.Capability;
import io.casehub.worker.api.Worker;
import io.casehub.worker.api.WorkerFunction;
import io.casehub.worker.api.WorkerOutcome;
import io.casehub.worker.api.WorkerResult;
import org.junit.jupiter.api.Test;

import java.util.Map;
import java.util.concurrent.atomic.AtomicInteger;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class WorkerExecutorTest {

    private final DefaultWorkerExecutor executor = new DefaultWorkerExecutor(new DefaultPolicyEnforcer());

    private static Capability cap(String name) {
        return Capability.of(name, "{}", "{}");
    }

    @Test
    void execute_successfulWorker() {
        Worker worker = Worker.builder()
            .name("greet").capabilityName("greet")
            .function(new WorkerFunction.Sync(input -> WorkerResult.of(Map.of("greeting", "hello " + input.get("name")))))
            .build();

        WorkerResult result = executor.execute(worker, cap("greet"), Map.of("name", "world"));
        assertThat(result.outcome()).isInstanceOf(WorkerOutcome.Success.class);
        assertThat(result.output()).containsEntry("greeting", "hello world");
    }

    @Test
    void execute_retriesTransientFailures() {
        AtomicInteger attempts = new AtomicInteger(0);
        Worker worker = Worker.builder()
            .name("flaky").capabilityName("process")
            .function(new WorkerFunction.Sync(input -> {
                if (attempts.incrementAndGet() < 3) {
                    throw new RuntimeException("transient");
                }
                return WorkerResult.of(Map.of("recovered", true));
            }))
            .executionPolicy(new ExecutionPolicy(null, new RetryPolicy(3, 10)))
            .build();

        WorkerResult result = executor.execute(worker, cap("process"), Map.of());
        assertThat(result.outcome()).isInstanceOf(WorkerOutcome.Success.class);
        assertThat(attempts.get()).isEqualTo(3);
    }

    @Test
    void execute_exhaustsRetries_returnsFailed() {
        Worker worker = Worker.builder()
            .name("broken").capabilityName("fail")
            .function(new WorkerFunction.Sync(input -> { throw new RuntimeException("permanent"); }))
            .executionPolicy(new ExecutionPolicy(null, new RetryPolicy(2, 10)))
            .build();

        WorkerResult result = executor.execute(worker, cap("fail"), Map.of());
        assertThat(result.outcome()).isInstanceOf(WorkerOutcome.Failed.class);
        assertThat(((WorkerOutcome.Failed) result.outcome()).reason()).isEqualTo("permanent");
    }

    @Test
    void execute_workerThrows_returnsFailed() {
        Worker worker = Worker.builder()
            .name("throws").capabilityName("boom")
            .function(new WorkerFunction.Sync(input -> { throw new IllegalStateException("bad state"); }))
            .build();

        WorkerResult result = executor.execute(worker, cap("boom"), Map.of());
        assertThat(result.outcome()).isInstanceOf(WorkerOutcome.Failed.class);
        assertThat(((WorkerOutcome.Failed) result.outcome()).reason()).isEqualTo("bad state");
    }

    @Test
    void execute_timeout_returnsExpired() {
        Worker worker = Worker.builder()
            .name("slow").capabilityName("crawl")
            .function(new WorkerFunction.Sync(input -> {
                try { Thread.sleep(500); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
                return WorkerResult.of(Map.of());
            }))
            .executionPolicy(new ExecutionPolicy(50, new RetryPolicy(1, 0)))
            .build();

        WorkerResult result = executor.execute(worker, cap("crawl"), Map.of());
        assertThat(result.outcome()).isInstanceOf(WorkerOutcome.Expired.class);
        assertThat(((WorkerOutcome.Expired) result.outcome()).reason()).contains("timed out");
    }

    @Test
    void execute_workerThrowsNullMessage_returnsFailedWithClassName() {
        Worker worker = Worker.builder()
            .name("npe").capabilityName("null")
            .function(new WorkerFunction.Sync(input -> { throw new NullPointerException(); }))
            .build();

        WorkerResult result = executor.execute(worker, cap("null"), Map.of());
        assertThat(result.outcome()).isInstanceOf(WorkerOutcome.Failed.class);
        assertThat(((WorkerOutcome.Failed) result.outcome()).reason()).isEqualTo("java.lang.NullPointerException");
    }

    // --- Validation tests ---

    @Test
    void execute_nullCapability_throwsNPE() {
        Worker worker = Worker.builder()
            .name("w").capabilityName("c")
            .function(new WorkerFunction.Sync(input -> WorkerResult.of(Map.of())))
            .build();

        assertThatThrownBy(() -> executor.execute(worker, null, Map.of()))
            .isInstanceOf(NullPointerException.class)
            .hasMessageContaining("capability");
    }

    @Test
    void execute_capabilityNotInWorker_throwsIAE() {
        Worker worker = Worker.builder()
            .name("w").capabilityName("supported")
            .function(new WorkerFunction.Sync(input -> WorkerResult.of(Map.of())))
            .build();

        assertThatThrownBy(() -> executor.execute(worker, cap("unsupported"), Map.of()))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("unsupported")
            .hasMessageContaining("w");
    }

    @Test
    void execute_nonSyncFunction_throwsUnsupported() {
        Worker worker = Worker.builder()
            .name("external").capabilityName("ext")
            .noFunction()
            .build();

        assertThatThrownBy(() -> executor.execute(worker, cap("ext"), Map.of()))
            .isInstanceOf(UnsupportedOperationException.class)
            .hasMessageContaining("Sync");
    }
}
```

- [ ] **Step 5: Migrate existing MockWorkerExecutorTest**

Replace `testing/src/test/java/io/casehub/worker/testing/MockWorkerExecutorTest.java` with:

```java
package io.casehub.worker.testing;

import io.casehub.worker.api.Capability;
import io.casehub.worker.api.WorkerOutcome;
import io.casehub.worker.api.WorkerResult;
import io.casehub.worker.api.Worker;
import org.junit.jupiter.api.Test;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class MockWorkerExecutorTest {

    private static Capability cap(String name) {
        return Capability.of(name, "{}", "{}");
    }

    @Test
    void execute_bypassesPolicyEnforcement() {
        MockWorkerExecutor executor = new MockWorkerExecutor();
        var wc = TestWorkerBuilder.syncWithCapability("test",
            input -> WorkerResult.of(Map.of("ok", true)));

        WorkerResult result = executor.execute(wc.worker(), wc.capability(), Map.of());
        assertThat(result.outcome()).isInstanceOf(WorkerOutcome.Success.class);
        assertThat(executor.executionCount()).isEqualTo(1);
        assertThat(executor.lastWorkerName()).isEqualTo("test");
        assertThat(executor.lastCapabilityName()).isEqualTo("test");
    }

    @Test
    void execute_workerThrows_returnsFailed() {
        MockWorkerExecutor executor = new MockWorkerExecutor();
        Worker worker = TestWorkerBuilder.sync("throws",
            input -> { throw new RuntimeException("mock failure"); });

        WorkerResult result = executor.execute(worker, cap("throws"), Map.of());
        assertThat(result.outcome()).isInstanceOf(WorkerOutcome.Failed.class);
        assertThat(((WorkerOutcome.Failed) result.outcome()).reason()).isEqualTo("mock failure");
        assertThat(executor.executionCount()).isEqualTo(1);
    }

    @Test
    void execute_workerThrowsNullMessage_returnsFailedWithClassName() {
        MockWorkerExecutor executor = new MockWorkerExecutor();
        Worker worker = TestWorkerBuilder.sync("npe",
            input -> { throw new NullPointerException(); });

        WorkerResult result = executor.execute(worker, cap("npe"), Map.of());
        assertThat(result.outcome()).isInstanceOf(WorkerOutcome.Failed.class);
        assertThat(((WorkerOutcome.Failed) result.outcome()).reason())
            .isEqualTo("java.lang.NullPointerException");
    }

    @Test
    void execute_tracksCapabilityName() {
        MockWorkerExecutor executor = new MockWorkerExecutor();
        var wc = TestWorkerBuilder.syncWithCapability("worker",
            input -> WorkerResult.of(Map.of()));

        executor.execute(wc.worker(), wc.capability(), Map.of());
        assertThat(executor.lastCapabilityName()).isEqualTo("worker");
    }

    @Test
    void reset_clearsCapabilityName() {
        MockWorkerExecutor executor = new MockWorkerExecutor();
        var wc = TestWorkerBuilder.syncWithCapability("worker",
            input -> WorkerResult.of(Map.of()));

        executor.execute(wc.worker(), wc.capability(), Map.of());
        assertThat(executor.lastCapabilityName()).isEqualTo("worker");

        executor.reset();
        assertThat(executor.lastCapabilityName()).isNull();
        assertThat(executor.lastWorkerName()).isNull();
        assertThat(executor.executionCount()).isZero();
    }

    @Test
    void execute_capabilityNotInWorker_throwsIAE() {
        MockWorkerExecutor executor = new MockWorkerExecutor();
        Worker worker = TestWorkerBuilder.sync("w",
            input -> WorkerResult.of(Map.of()));

        assertThatThrownBy(() -> executor.execute(worker, cap("wrong"), Map.of()))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("wrong");
    }
}
```

- [ ] **Step 6: Run all tests**

Run: `mvn -f /Users/mdproctor/claude/casehub/worker/pom.xml test`
Expected: ALL tests pass across all modules

- [ ] **Step 7: Commit**

```
feat: capability-aware execution — WorkerExecutor knows which capability is invoked

WorkerExecutor.execute() now takes (Worker, Capability, Map) instead of
(Worker, Map). Validates capability membership, rejects null capability and
non-Sync functions before PolicyEnforcer (programming errors are not retried).
Adds worker.capability OTel span attribute.

MockWorkerExecutor applies the same validation contract and tracks
lastCapabilityName for test assertions.

Closes #9
```
