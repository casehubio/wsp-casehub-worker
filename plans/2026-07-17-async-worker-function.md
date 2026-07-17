# Async WorkerFunction Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #5 — feat: async WorkerFunction — CompletionStage-based worker execution
**Issue group:** #5

**Goal:** Add an Async variant to WorkerFunction, change WorkerExecutor to return Uni\<WorkerResult\>, replace PolicyEnforcer with SmallRye FT Guard, and unify sync/async execution in a single pipeline.

**Architecture:** WorkerFunction gains an Async\<T\> record variant (CompletionStage return). The runtime module's WorkerExecutor returns Uni\<WorkerResult\>. DefaultWorkerExecutor lifts both Sync and Async into Uni, wraps with SmallRye FT Guard for fault tolerance, and manages OTel spans via callback-based lifecycle. PolicyEnforcer is removed.

**Tech Stack:** Java 21, Quarkus 3.32.2, SmallRye Mutiny (Uni), SmallRye Fault Tolerance (Guard), OpenTelemetry API, JUnit 5, AssertJ

## Global Constraints

- API module (`casehub-worker-api`) has zero framework dependencies — only `casehub-platform-api`. CompletionStage (JDK standard) in WorkerFunction.Async; never Uni.
- Runtime module uses Uni as the return type of WorkerExecutor — Mutiny is transitively available via quarkus-smallrye-fault-tolerance.
- Testing module depends on runtime — its MockWorkerExecutor must match the WorkerExecutor interface.
- ExecutionPolicy, RetryPolicy, BackoffStrategy stay in `casehub-platform-api` — they are domain config, not enforcement.
- Pre-release: breaking changes are free. Fix the design, never protect callers.

---

### Task 1: WorkerFunction.Async\<T\> Record

**Files:**
- Modify: `api/src/main/java/io/casehub/worker/api/WorkerFunction.java`
- Modify: `api/src/test/java/io/casehub/worker/api/WorkerFunctionTest.java`

**Interfaces:**
- Produces: `WorkerFunction.Async<T>` record with `inputType()` and `fn()` accessors. Same shape as `Sync<T>` but fn returns `CompletionStage<WorkerResult>`.

- [ ] **Step 1: Write failing tests for Async variant**

```java
@Test
void async_is_workerFunction() {
    WorkerFunction<?> fn = new WorkerFunction.Async<>(Map.class,
        input -> CompletableFuture.completedFuture(WorkerResult.of(Map.of())));
    assertThat(fn).isInstanceOf(WorkerFunction.class);
}

@Test
void async_fn_accessor_returns_function() {
    var async = new WorkerFunction.Async<>(Map.class,
        (Map<String, Object> input) -> CompletableFuture.completedFuture(
            WorkerResult.of(Map.of("key", "value"))));
    CompletionStage<WorkerResult> stage = async.fn().apply(Map.of());
    WorkerResult result = stage.toCompletableFuture().join();
    assertThat(result.output()).containsEntry("key", "value");
}

@Test
void asyncCarriesInputType() {
    var fn = new WorkerFunction.Async<>(String.class,
        s -> CompletableFuture.completedFuture(WorkerResult.of(Map.of("len", s.length()))));
    assertThat(fn.inputType()).isEqualTo(String.class);
}

@Test
void asyncRejectsNullInputType() {
    assertThatThrownBy(() -> new WorkerFunction.Async<>(null,
        input -> CompletableFuture.completedFuture(WorkerResult.of(Map.of()))))
        .isInstanceOf(NullPointerException.class);
}

@Test
void asyncRejectsNullFunction() {
    assertThatThrownBy(() -> new WorkerFunction.Async<>(String.class, null))
        .isInstanceOf(NullPointerException.class);
}

@Test
void async_is_not_sync() {
    WorkerFunction<?> fn = new WorkerFunction.Async<>(Map.class,
        input -> CompletableFuture.completedFuture(WorkerResult.of(Map.of())));
    assertThat(fn).isNotInstanceOf(WorkerFunction.Sync.class);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn -f /Users/mdproctor/claude/casehub/worker/api/pom.xml test -pl . -Dtest=WorkerFunctionTest`
Expected: Compilation failure — `WorkerFunction.Async` does not exist.

- [ ] **Step 3: Implement WorkerFunction.Async record**

Add to WorkerFunction.java after the Sync record:

```java
record Async<T>(Class<T> inputType, Function<T, CompletionStage<WorkerResult>> fn)
        implements WorkerFunction<T> {
    public Async {
        Objects.requireNonNull(inputType, "inputType must not be null");
        Objects.requireNonNull(fn, "fn must not be null");
    }
}
```

Add import: `java.util.concurrent.CompletionStage`

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn -f /Users/mdproctor/claude/casehub/worker/api/pom.xml test -pl . -Dtest=WorkerFunctionTest`
Expected: All tests PASS.

- [ ] **Step 5: Commit**

```
feat: add WorkerFunction.Async<T> — CompletionStage-returning variant

Refs #5
```

---

### Task 2: Worker.Builder Async Convenience Methods

**Files:**
- Modify: `api/src/main/java/io/casehub/worker/api/Worker.java`
- Modify: `api/src/main/java/io/casehub/worker/api/TypedFunctionBuilder.java`
- Modify: `api/src/test/java/io/casehub/worker/api/WorkerFunctionTest.java` (add builder tests)

**Interfaces:**
- Consumes: `WorkerFunction.Async<T>` from Task 1
- Produces: `Worker.Builder.asyncFunction(Function<Map, CompletionStage<WorkerResult>>)` and `TypedFunctionBuilder.applyAsync(Function<T, CompletionStage<WorkerResult>>)`

- [ ] **Step 1: Write failing tests**

```java
@Test
void builder_asyncFunction_createsAsyncWorker() {
    Worker worker = Worker.builder()
        .name("async-w").capabilityName("cap")
        .asyncFunction(input -> CompletableFuture.completedFuture(
            WorkerResult.of(Map.of("done", true))))
        .build();
    assertThat(worker.function()).isInstanceOf(WorkerFunction.Async.class);
}

@Test
void builder_typedAsyncFunction_createsAsyncWorker() {
    Worker worker = Worker.builder()
        .name("typed-async").capabilityName("cap")
        .<String>fn().applyAsync(s -> CompletableFuture.completedFuture(
            WorkerResult.of(Map.of("len", s.length()))))
        .build();
    assertThat(worker.function()).isInstanceOf(WorkerFunction.Async.class);
    assertThat(worker.function().inputType()).isEqualTo(String.class);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn -f /Users/mdproctor/claude/casehub/worker/api/pom.xml test -pl . -Dtest=WorkerFunctionTest`
Expected: Compilation failure — methods don't exist.

- [ ] **Step 3: Implement Builder.asyncFunction and TypedFunctionBuilder.applyAsync**

In Worker.Builder, add:
```java
@SuppressWarnings("unchecked")
public Builder asyncFunction(
        java.util.function.Function<java.util.Map<String, Object>,
            java.util.concurrent.CompletionStage<WorkerResult>> fn) {
    this.function = new WorkerFunction.Async<>((Class) java.util.Map.class, fn);
    return this;
}
```

In TypedFunctionBuilder, add:
```java
@SuppressWarnings({"unchecked", "rawtypes"})
public Worker.Builder applyAsync(
        Function<T, java.util.concurrent.CompletionStage<WorkerResult>> fn) {
    parent.setFunction(new WorkerFunction.Async(runtimeType, fn));
    return parent;
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn -f /Users/mdproctor/claude/casehub/worker/api/pom.xml test -pl . -Dtest=WorkerFunctionTest`
Expected: All tests PASS.

- [ ] **Step 5: Commit**

```
feat: Worker.Builder async convenience — asyncFunction() and fn().applyAsync()

Refs #5
```

---

### Task 3: WorkerExecutor Returns Uni + Dependencies

**Files:**
- Modify: `runtime/pom.xml` — add quarkus-smallrye-fault-tolerance, remove casehub-platform-governance
- Modify: `runtime/src/main/java/io/casehub/worker/runtime/WorkerExecutor.java`
- Modify: `runtime/src/test/java/io/casehub/worker/runtime/WorkerExecutorCdiTest.java`
- Modify: `testing/pom.xml` — add smallrye-mutiny dependency if not transitive
- Modify: `testing/src/main/java/io/casehub/worker/testing/MockWorkerExecutor.java`
- Modify: `testing/src/test/java/io/casehub/worker/testing/MockWorkerExecutorTest.java`

**Interfaces:**
- Consumes: `WorkerFunction.Async<T>` from Task 1
- Produces: `WorkerExecutor.execute()` returning `Uni<WorkerResult>`. MockWorkerExecutor matching the new interface.

This task changes the interface and both implementors (Default + Mock) together — they must stay in sync for the build to compile.

- [ ] **Step 1: Update runtime/pom.xml dependencies**

Add `quarkus-smallrye-fault-tolerance`. Remove `casehub-platform-governance`.

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-smallrye-fault-tolerance</artifactId>
</dependency>
```

Remove:
```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-governance</artifactId>
</dependency>
```

- [ ] **Step 2: Change WorkerExecutor interface to return Uni**

```java
package io.casehub.worker.runtime;

import io.casehub.worker.api.Capability;
import io.casehub.worker.api.Worker;
import io.casehub.worker.api.WorkerResult;
import io.smallrye.mutiny.Uni;

public interface WorkerExecutor {
    Uni<WorkerResult> execute(Worker worker, Capability capability, Object input);
}
```

- [ ] **Step 3: Update MockWorkerExecutor to match new interface**

Return type changes to `Uni<WorkerResult>`. Handles Sync (wrap in Uni.createFrom().item()), Async (Uni.createFrom().completionStage()), and catches exceptions.

```java
@SuppressWarnings({"unchecked", "rawtypes"})
@Override
public Uni<WorkerResult> execute(Worker worker, Capability capability, Object input) {
    Objects.requireNonNull(capability, "capability");
    if (!worker.capabilityNames().contains(capability.name())) {
        throw new IllegalArgumentException(
                "Capability '" + capability.name() + "' not in worker '"
                + worker.name() + "' capabilities: " + worker.capabilityNames());
    }
    executionCount.incrementAndGet();
    lastWorkerName.set(worker.name());
    lastCapabilityName.set(capability.name());

    if (worker.function() instanceof WorkerFunction.Sync sync) {
        if (!sync.inputType().isInstance(input)) {
            throw new IllegalArgumentException(
                    "Input type mismatch: expected " + sync.inputType().getName()
                    + ", got " + (input == null ? "null" : input.getClass().getName()));
        }
        return Uni.createFrom().item(() -> {
            try {
                return (WorkerResult) ((Function) sync.fn()).apply(input);
            } catch (Exception e) {
                String message = e.getMessage();
                if (message == null) message = e.getClass().getName();
                return WorkerResult.failed(message);
            }
        });
    } else if (worker.function() instanceof WorkerFunction.Async async) {
        if (!async.inputType().isInstance(input)) {
            throw new IllegalArgumentException(
                    "Input type mismatch: expected " + async.inputType().getName()
                    + ", got " + (input == null ? "null" : input.getClass().getName()));
        }
        return Uni.createFrom().completionStage(
                () -> (CompletionStage<WorkerResult>) ((Function) async.fn()).apply(input))
            .onFailure().recoverWithItem(e -> {
                String message = e.getMessage();
                if (message == null) message = e.getClass().getName();
                return WorkerResult.failed(message);
            });
    } else {
        throw new UnsupportedOperationException(
                "MockWorkerExecutor supports Sync and Async functions only, got: "
                + worker.function().getClass().getName());
    }
}
```

- [ ] **Step 4: Stub DefaultWorkerExecutor to compile**

Temporarily change DefaultWorkerExecutor.execute() return type to `Uni<WorkerResult>` with a minimal implementation that just throws `UnsupportedOperationException("not yet implemented")`. This lets the build compile while we refactor the real implementation in Task 4. Remove PolicyEnforcer field and constructor param; add a no-arg constructor with just SchemaValidator.

- [ ] **Step 5: Update WorkerExecutorCdiTest**

```java
@Test
void injectedExecutor_resolvesToDefaultImpl() {
    assertThat(executor).isInstanceOf(DefaultWorkerExecutor.class);
}
```

This test should still pass — only the return type changed, CDI resolution is unchanged.

- [ ] **Step 6: Update MockWorkerExecutorTest to use Uni**

Every `executor.execute(...)` call now returns `Uni<WorkerResult>`. Add `.await().indefinitely()` to each call to get the result synchronously in tests.

- [ ] **Step 7: Run build to verify compilation**

Run: `mvn -f /Users/mdproctor/claude/casehub/worker/pom.xml compile -DskipTests`
Expected: BUILD SUCCESS.

- [ ] **Step 8: Run tests**

Run: `mvn -f /Users/mdproctor/claude/casehub/worker/pom.xml test`
Expected: MockWorkerExecutorTest and WorkerExecutorCdiTest pass. WorkerExecutorTest will fail (DefaultWorkerExecutor is stubbed). That's expected — Task 4 fixes it.

- [ ] **Step 9: Commit**

```
refactor: WorkerExecutor returns Uni<WorkerResult> — interface + MockWorkerExecutor

Replace casehub-platform-governance with quarkus-smallrye-fault-tolerance.
DefaultWorkerExecutor stubbed — next commit implements the unified pipeline.

Refs #5
```

---

### Task 4: DefaultWorkerExecutor — Unified Pipeline with Guard

**Files:**
- Modify: `runtime/src/main/java/io/casehub/worker/runtime/DefaultWorkerExecutor.java`
- Modify: `runtime/src/test/java/io/casehub/worker/runtime/WorkerExecutorTest.java`

**Interfaces:**
- Consumes: `WorkerExecutor` (Uni return) from Task 3, `WorkerFunction.Async<T>` from Task 1
- Produces: Fully functional DefaultWorkerExecutor with Guard-based fault tolerance, OTel tracing, schema validation. Single pipeline for Sync and Async.

- [ ] **Step 1: Write failing tests for async execution**

Add to WorkerExecutorTest (alongside existing sync tests that need updating to use `.await().indefinitely()`):

```java
@Test
void execute_asyncWorker_completesSuccessfully() {
    Worker worker = Worker.builder()
        .name("async").capabilityName("fetch")
        .asyncFunction(input -> CompletableFuture.completedFuture(
            WorkerResult.of(Map.of("fetched", true))))
        .build();

    WorkerResult result = executor.execute(worker, cap("fetch"), Map.of())
        .await().indefinitely();
    assertThat(result.outcome()).isInstanceOf(WorkerOutcome.Success.class);
    assertThat(result.output()).containsEntry("fetched", true);
}

@Test
void execute_asyncWorker_exceptionInStage_returnsFailed() {
    Worker worker = Worker.builder()
        .name("failing-async").capabilityName("fail")
        .asyncFunction(input -> CompletableFuture.failedFuture(
            new RuntimeException("async boom")))
        .build();

    WorkerResult result = executor.execute(worker, cap("fail"), Map.of())
        .await().indefinitely();
    assertThat(result.outcome()).isInstanceOf(WorkerOutcome.Failed.class);
    assertThat(((WorkerOutcome.Failed) result.outcome()).reason()).isEqualTo("async boom");
}

@Test
void execute_asyncWorker_throwsDuringDispatch_returnsFailed() {
    Worker worker = Worker.builder()
        .name("dispatch-fail").capabilityName("boom")
        .asyncFunction(input -> { throw new IllegalStateException("dispatch error"); })
        .build();

    WorkerResult result = executor.execute(worker, cap("boom"), Map.of())
        .await().indefinitely();
    assertThat(result.outcome()).isInstanceOf(WorkerOutcome.Failed.class);
    assertThat(((WorkerOutcome.Failed) result.outcome()).reason()).isEqualTo("dispatch error");
}

@Test
void execute_asyncTypedWorker_passesTypedInput() {
    Worker worker = Worker.builder()
        .name("typed-async").capabilityName("process")
        .<TestPojo>fn().applyAsync(pojo -> CompletableFuture.completedFuture(
            WorkerResult.of(Map.of("greeting", "hello " + pojo.name()))))
        .build();

    WorkerResult result = executor.execute(worker, cap("process"),
        new TestPojo("alice", 30)).await().indefinitely();
    assertThat(result.outcome()).isInstanceOf(WorkerOutcome.Success.class);
    assertThat(result.output()).containsEntry("greeting", "hello alice");
}

@Test
void execute_asyncWorker_inputTypeMismatch_throwsIAE() {
    Worker worker = Worker.builder()
        .name("typed-async").capabilityName("process")
        .<TestPojo>fn().applyAsync(pojo -> CompletableFuture.completedFuture(
            WorkerResult.of(Map.of())))
        .build();

    assertThatThrownBy(() -> executor.execute(worker, cap("process"), Map.of("name", "alice")))
        .isInstanceOf(IllegalArgumentException.class)
        .hasMessageContaining("TestPojo");
}

@Test
void execute_asyncWorker_schemaValidatesInput() {
    AtomicInteger callCount = new AtomicInteger(0);
    Worker worker = Worker.builder()
        .name("strict-async").capabilityName("validate")
        .asyncFunction(input -> {
            callCount.incrementAndGet();
            return CompletableFuture.completedFuture(WorkerResult.of(Map.of()));
        })
        .build();

    WorkerResult result = executor.execute(worker,
        cap("validate", REQUIRE_NAME_SCHEMA, "{}"),
        Map.of("age", 30)).await().indefinitely();

    assertThat(result.outcome()).isInstanceOf(WorkerOutcome.Failed.class);
    assertThat(((WorkerOutcome.Failed) result.outcome()).reason()).contains("name");
    assertThat(callCount.get()).isZero();
}
```

- [ ] **Step 2: Update all existing sync tests to use `.await().indefinitely()`**

Every `executor.execute(...)` call in WorkerExecutorTest returns `Uni<WorkerResult>` now. Add `.await().indefinitely()` to get WorkerResult. `assertThatThrownBy` tests for programming errors stay as-is (these throw before Uni is created).

- [ ] **Step 3: Implement DefaultWorkerExecutor unified pipeline**

Replace the entire execute() method and constructor. Key structure:

```java
@ApplicationScoped
public class DefaultWorkerExecutor implements WorkerExecutor {

    private static final String INSTRUMENTATION_NAME = "io.casehub.worker";
    private static final Logger LOG = Logger.getLogger(DefaultWorkerExecutor.class);

    private final SchemaValidator schemaValidator;
    private final ConcurrentHashMap<ExecutionPolicy, Guard> guardCache = new ConcurrentHashMap<>();

    @Inject
    public DefaultWorkerExecutor(SchemaValidator schemaValidator) {
        this.schemaValidator = schemaValidator;
    }

    @Override
    public Uni<WorkerResult> execute(Worker worker, Capability capability, Object input) {
        // 1. Validate (programming errors — throw immediately)
        Objects.requireNonNull(capability, "capability");
        // ... capability membership, input type check ...

        // 2. Schema validate input
        schemaValidator.ensureSchemaParsed(capability.inputSchema());
        schemaValidator.ensureSchemaParsed(capability.outputSchema());

        Span span = GlobalOpenTelemetry.getTracer(INSTRUMENTATION_NAME)
            .spanBuilder("worker.execute")
            .setAttribute("worker.name", worker.name())
            .setAttribute("worker.capability", capability.name())
            .startSpan();

        try (Scope ignored = span.makeCurrent()) {
            Optional<String> inputError = schemaValidator.validateInput(capability, input);
            if (inputError.isPresent()) {
                span.addEvent("worker.input.invalid", ...);
                WorkerResult result = WorkerResult.failed(inputError.get());
                span.setAttribute("worker.outcome", "Failed");
                span.end();
                return Uni.createFrom().item(result);
            }

            // 3. Lift to Uni
            Uni<WorkerResult> action = liftToUni(worker, input);

            // 4. Guard (fault tolerance)
            Guard guard = guardCache.computeIfAbsent(
                worker.executionPolicy(), this::buildGuard);
            Uni<WorkerResult> guarded = guard.call(() -> action, Uni.class);

            // 5-7. Schema validate output + OTel + exception mapping
            return guarded
                .map(result -> {
                    if (result.outcome() instanceof WorkerOutcome.Success) {
                        Optional<String> outputError = schemaValidator.validateOutput(
                            capability, result.output());
                        outputError.ifPresent(err -> {
                            span.addEvent("worker.output.invalid", ...);
                            LOG.warnf("Output schema violation ...");
                        });
                    }
                    return result;
                })
                .onFailure(TimeoutException.class).recoverWithItem(e -> {
                    span.addEvent("worker.timeout", ...);
                    return WorkerResult.expired(e.getMessage());
                })
                .onFailure().recoverWithItem(e -> {
                    span.setStatus(StatusCode.ERROR, e.getMessage());
                    span.recordException(e);
                    Throwable root = e.getCause() != null ? e.getCause() : e;
                    String message = root.getMessage();
                    if (message == null) message = root.getClass().getName();
                    return WorkerResult.failed(message);
                })
                .onTermination().invoke((result, failure, cancelled) -> {
                    if (result != null) {
                        span.setAttribute("worker.outcome",
                            result.outcome().getClass().getSimpleName());
                    }
                    span.end();
                });
        }
    }

    @SuppressWarnings({"unchecked", "rawtypes"})
    private Uni<WorkerResult> liftToUni(Worker worker, Object input) {
        if (worker.function() instanceof WorkerFunction.Sync sync) {
            return Uni.createFrom().item(() ->
                (WorkerResult) ((Function) sync.fn()).apply(input));
        } else if (worker.function() instanceof WorkerFunction.Async async) {
            return Uni.createFrom().completionStage(() ->
                (CompletionStage<WorkerResult>) ((Function) async.fn()).apply(input));
        }
        throw new UnsupportedOperationException(
            "DefaultWorkerExecutor supports Sync and Async functions only, got: "
            + worker.function().getClass().getName());
    }

    private Guard buildGuard(ExecutionPolicy policy) {
        var builder = Guard.create();
        if (policy.timeoutMs() != null) {
            builder.withTimeout()
                .duration(policy.timeoutMs(), ChronoUnit.MILLIS)
                .done();
        }
        RetryPolicy retry = policy.retries();
        if (retry != null && retry.maxAttempts() != null && retry.maxAttempts() > 1) {
            var rb = builder.withRetry()
                .maxRetries(retry.maxAttempts() - 1)
                .delay(retry.delayMs() != null ? retry.delayMs() : 0, ChronoUnit.MILLIS);
            if (retry.backoffStrategy() == BackoffStrategy.EXPONENTIAL) {
                rb.withExponentialBackoff().done();
            } else if (retry.backoffStrategy() == BackoffStrategy.EXPONENTIAL_WITH_JITTER) {
                rb.withExponentialBackoff().done()
                    .jitter(retry.delayMs() != null ? retry.delayMs() : 200, ChronoUnit.MILLIS);
            }
            if (retry.maxDelayMs() != null) {
                rb.maxDelay(retry.maxDelayMs(), ChronoUnit.MILLIS);
            }
            rb.done();
        }
        return builder.build();
    }
}
```

Note: The spec calls for `runSubscriptionOn(virtualThreads)` for sync workers so Guard's timeout can fire. This will be validated by the existing timeout test — if the test fails, add the virtual thread offload. If it passes without it (Guard may handle this internally), leave it out per YAGNI.

- [ ] **Step 4: Run all tests**

Run: `mvn -f /Users/mdproctor/claude/casehub/worker/pom.xml test`
Expected: All tests pass — existing sync behavior preserved, new async tests pass.

- [ ] **Step 5: If timeout test fails — add virtual thread offload for sync**

If `execute_timeout_returnsExpired` fails (sync blocks the subscriber thread, preventing Guard's timeout race), change the sync lift in `liftToUni`:
```java
return Uni.createFrom().item(() ->
    (WorkerResult) ((Function) sync.fn()).apply(input))
    .runSubscriptionOn(Infrastructure.getDefaultWorkerPool());
```

Rerun tests. Expected: timeout test passes.

- [ ] **Step 6: Commit**

```
feat: DefaultWorkerExecutor unified pipeline — Guard + OTel + Uni

Single code path for Sync and Async. SmallRye FT Guard replaces
PolicyEnforcer. OTel span closed in onTermination callback.

Refs #5
```

---

### Task 5: TestWorkerBuilder Async Factories

**Files:**
- Modify: `testing/src/main/java/io/casehub/worker/testing/TestWorkerBuilder.java`
- Modify: `testing/src/test/java/io/casehub/worker/testing/TestWorkerBuilderTest.java`

**Interfaces:**
- Consumes: `Worker.Builder.asyncFunction()` from Task 2
- Produces: `TestWorkerBuilder.async(name, fn)`, `asyncWithCapability(name, fn)`, `asyncWithCapability(name, inputSchema, outputSchema, fn)`

- [ ] **Step 1: Write failing tests**

```java
@Test
void async_createsWorkerWithAsyncFunction() {
    Worker worker = TestWorkerBuilder.async("fetcher",
        input -> CompletableFuture.completedFuture(WorkerResult.of(Map.of("ok", true))));
    assertThat(worker.name()).isEqualTo("fetcher");
    assertThat(worker.function()).isInstanceOf(WorkerFunction.Async.class);
}

@Test
void asyncWithCapability_createsWorkerAndCapability() {
    var wc = TestWorkerBuilder.asyncWithCapability("fetcher",
        input -> CompletableFuture.completedFuture(WorkerResult.of(Map.of("ok", true))));
    assertThat(wc.worker().function()).isInstanceOf(WorkerFunction.Async.class);
    assertThat(wc.capability().name()).isEqualTo("fetcher");
}

@Test
void asyncWithCapability_withSchemas_appliesSchemas() {
    var wc = TestWorkerBuilder.asyncWithCapability("fetcher",
        "{\"type\":\"object\"}", "{\"type\":\"object\"}",
        input -> CompletableFuture.completedFuture(WorkerResult.of(Map.of())));
    assertThat(wc.capability().inputSchema()).isEqualTo("{\"type\":\"object\"}");
    assertThat(wc.capability().outputSchema()).isEqualTo("{\"type\":\"object\"}");
}
```

- [ ] **Step 2: Run tests — expect compilation failure**

Run: `mvn -f /Users/mdproctor/claude/casehub/worker/testing/pom.xml test -Dtest=TestWorkerBuilderTest`

- [ ] **Step 3: Implement async factories**

```java
public static Worker async(String name,
        Function<Map<String, Object>, CompletionStage<WorkerResult>> fn) {
    return Worker.builder()
        .name(name).capabilityName(name)
        .asyncFunction(fn)
        .build();
}

public static WorkerWithCapability asyncWithCapability(String name,
        Function<Map<String, Object>, CompletionStage<WorkerResult>> fn) {
    Worker worker = Worker.builder()
        .name(name).capabilityName(name)
        .asyncFunction(fn)
        .build();
    Capability capability = Capability.of(name, "{}", "{}");
    return new WorkerWithCapability(worker, capability);
}

public static WorkerWithCapability asyncWithCapability(String name,
        String inputSchema, String outputSchema,
        Function<Map<String, Object>, CompletionStage<WorkerResult>> fn) {
    Worker worker = Worker.builder()
        .name(name).capabilityName(name)
        .asyncFunction(fn)
        .build();
    Capability capability = Capability.of(name, inputSchema, outputSchema);
    return new WorkerWithCapability(worker, capability);
}
```

- [ ] **Step 4: Run all tests**

Run: `mvn -f /Users/mdproctor/claude/casehub/worker/pom.xml test`
Expected: All pass.

- [ ] **Step 5: Commit**

```
feat: TestWorkerBuilder async factories — async(), asyncWithCapability()

Refs #5
```

---

### Task 6: Update docs/DESIGN.md + Full Build Verification

**Files:**
- Modify: `docs/DESIGN.md`

**Interfaces:**
- Consumes: All previous tasks — verifies the complete system works.

- [ ] **Step 1: Update DESIGN.md**

Add to the Execution Contract section:
- Document WorkerExecutor now returns `Uni<WorkerResult>`
- Document the Async variant in WorkerFunction
- Update the Programming errors table: replace "Non-Sync function" with "Non-Sync/Async function"
- Add Guard section explaining SmallRye FT replacement of PolicyEnforcer
- Update OTel documentation for callback-based span lifecycle

- [ ] **Step 2: Full build verification**

Run: `mvn -f /Users/mdproctor/claude/casehub/worker/pom.xml clean install`
Expected: BUILD SUCCESS with all tests passing.

- [ ] **Step 3: Commit**

```
docs: update DESIGN.md — async execution, Uni return, Guard fault tolerance

Refs #5
```

- [ ] **Step 4: Verify with ide_diagnostics**

Run `ide_diagnostics` on all modified files to confirm no warnings or errors.
