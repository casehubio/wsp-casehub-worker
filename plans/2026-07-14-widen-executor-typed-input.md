# Widen WorkerExecutor to Object Input — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #10 — refactor: widen WorkerExecutor to Object input — honour WorkerFunction\<T\> type parameter
**Issue group:** #10

**Goal:** Make the worker executor accept typed input (`Object` instead of
`Map<String, Object>`) and pass it through to `WorkerFunction<T>.fn()` without
Map casting — removing the type bottleneck in the ContextBridge pipeline.

**Architecture:** The `WorkerExecutor` interface widens its input parameter.
`DefaultWorkerExecutor` validates input type via `inputType().isInstance()` (strict,
no Map escape hatch) and invokes `fn.apply(input)` with raw-type cast. `SchemaValidator`
widens its public methods from `Map<String, Object>` to `Object` — the internal
`valueToTree()` already handles any Jackson-serializable type. `MockWorkerExecutor`
mirrors the same input type validation.

**Tech Stack:** Java 21, Quarkus 3.32.2, Jackson 2.x, OpenTelemetry API,
networknt json-schema-validator 1.0.83, JUnit 5, AssertJ

## Global Constraints

- `casehub-worker-api` is Tier 1 pure-Java — no changes in this plan (WorkerFunction\<T\> already parameterised)
- `casehub-worker-runtime` is Tier 3 — Quarkus, CDI, OTel. Contains WorkerExecutor, DefaultWorkerExecutor, SchemaValidator
- `casehub-worker-testing` — `@DefaultBean` mock
- Build: `mvn -f /Users/mdproctor/claude/casehub/worker/pom.xml clean install`
- Test: `mvn -f /Users/mdproctor/claude/casehub/worker/pom.xml test`
- Single module: `mvn -f /Users/mdproctor/claude/casehub/worker/runtime/pom.xml test`
- Spec: `docs/specs/2026-07-14-widen-executor-typed-input-design.md`
- Pre-release — breaking changes are free. No backward-compat shims.

---

### Task 1: SchemaValidator — widen from Map to Object

**Files:**
- Modify: `runtime/src/main/java/io/casehub/worker/runtime/SchemaValidator.java:33-50`
- Modify: `runtime/src/test/java/io/casehub/worker/runtime/WorkerExecutorTest.java` (add POJO schema test)

**Interfaces:**
- Consumes: `Capability.inputSchema()`, `Capability.outputSchema()` — unchanged
- Produces: `validateInput(Capability, Object)`, `validateOutput(Capability, Object)`, `validate(String, Object)` — widened from `Map<String, Object>` to `Object`

- [ ] **Step 1: Write POJO schema validation test**

Add to `runtime/src/test/java/io/casehub/worker/runtime/WorkerExecutorTest.java`. This test will fail because `SchemaValidator.validateInput` takes `Map<String, Object>`, not `Object`.

First, define a test POJO as a static inner record at the top of the test class:

```java
record TestPojo(String name, int age) {}
```

Then add the test:

```java
@Test
void schemaValidator_validatesPojoInput() {
    SchemaValidator validator = new SchemaValidator();
    Capability cap = cap("test", REQUIRE_NAME_SCHEMA, "{}");
    validator.ensureSchemaParsed(cap.inputSchema());

    Optional<String> valid = validator.validateInput(cap, new TestPojo("alice", 30));
    assertThat(valid).isEmpty();

    Optional<String> invalid = validator.validateInput(cap, new TestPojo(null, 30));
    assertThat(invalid).isPresent();
}
```

- [ ] **Step 2: Run test — verify it fails**

Run: `mvn -f /Users/mdproctor/claude/casehub/worker/runtime/pom.xml test -Dtest=WorkerExecutorTest#schemaValidator_validatesPojoInput -pl .`
Expected: compilation failure — `validateInput(Capability, Map<String, Object>)` does not accept `TestPojo`

- [ ] **Step 3: Widen SchemaValidator**

Use `ide_edit_member` for each method:

`validateInput` — change parameter from `Map<String, Object> input` to `Object input`:
```java
public Optional<String> validateInput(Capability capability, Object input) {
    return validate(capability.inputSchema(), input);
}
```

`validateOutput` — change parameter from `Map<String, Object> output` to `Object output`:
```java
public Optional<String> validateOutput(Capability capability, Object output) {
    return validate(capability.outputSchema(), output);
}
```

`validate` — change parameter from `Map<String, Object> data` to `Object data`:
```java
private Optional<String> validate(String schemaString, Object data) {
    if (EMPTY_SCHEMA.equals(schemaString)) return Optional.empty();
    JsonSchema schema = cache.computeIfAbsent(schemaString, this::parseSchema);
    JsonNode node = objectMapper.valueToTree(data);
    Set<ValidationMessage> errors = schema.validate(node);
    if (errors.isEmpty()) return Optional.empty();
    String message = errors.stream()
        .map(ValidationMessage::getMessage)
        .collect(Collectors.joining("\n"));
    return Optional.of(message);
}
```

- [ ] **Step 4: Run test — verify it passes**

Run: `mvn -f /Users/mdproctor/claude/casehub/worker/runtime/pom.xml test -Dtest=WorkerExecutorTest#schemaValidator_validatesPojoInput -pl .`
Expected: PASS

- [ ] **Step 5: Run all existing tests — verify no regressions**

Run: `mvn -f /Users/mdproctor/claude/casehub/worker/pom.xml test`
Expected: all tests pass (Map is Object — existing callers unaffected)

- [ ] **Step 6: Verify with ide_diagnostics**

Run `ide_diagnostics` on `SchemaValidator.java` — expect no errors.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/worker add runtime/src/main/java/io/casehub/worker/runtime/SchemaValidator.java runtime/src/test/java/io/casehub/worker/runtime/WorkerExecutorTest.java
git -C /Users/mdproctor/claude/casehub/worker commit -m "refactor: widen SchemaValidator from Map to Object — valueToTree handles any type  Refs #10"
```

---

### Task 2: WorkerExecutor interface + DefaultWorkerExecutor — widen and honour T

**Files:**
- Modify: `runtime/src/main/java/io/casehub/worker/runtime/WorkerExecutor.java:9-26`
- Modify: `runtime/src/main/java/io/casehub/worker/runtime/DefaultWorkerExecutor.java:42-115`
- Modify: `runtime/src/test/java/io/casehub/worker/runtime/WorkerExecutorTest.java`

**Interfaces:**
- Consumes: `SchemaValidator.validateInput(Capability, Object)` — from Task 1
- Produces: `WorkerExecutor.execute(Worker, Capability, Object)` — widened interface

- [ ] **Step 1: Write typed POJO execution test**

Add `TestPojo` record to `WorkerExecutorTest` (if not already from Task 1) and add the test:

```java
@Test
void execute_typedPojoInput_passedToFunction() {
    Worker worker = Worker.builder()
        .name("typed").capabilityName("process")
        .<TestPojo>fn().apply(pojo -> WorkerResult.of(Map.of("greeting", "hello " + pojo.name())))
        .build();

    WorkerResult result = executor.execute(worker, cap("process"), new TestPojo("alice", 30));
    assertThat(result.outcome()).isInstanceOf(WorkerOutcome.Success.class);
    assertThat(result.output()).containsEntry("greeting", "hello alice");
}
```

- [ ] **Step 2: Write input type mismatch test**

```java
@Test
void execute_inputTypeMismatch_throwsIAE() {
    Worker worker = Worker.builder()
        .name("typed").capabilityName("process")
        .<TestPojo>fn().apply(pojo -> WorkerResult.of(Map.of()))
        .build();

    assertThatThrownBy(() -> executor.execute(worker, cap("process"), Map.of("name", "alice")))
        .isInstanceOf(IllegalArgumentException.class)
        .hasMessageContaining("TestPojo")
        .hasMessageContaining("Map");
}
```

- [ ] **Step 3: Write null input test**

```java
@Test
void execute_nullInput_throwsIAE() {
    Worker worker = Worker.builder()
        .name("typed").capabilityName("process")
        .<TestPojo>fn().apply(pojo -> WorkerResult.of(Map.of()))
        .build();

    assertThatThrownBy(() -> executor.execute(worker, cap("process"), null))
        .isInstanceOf(IllegalArgumentException.class)
        .hasMessageContaining("null");
}
```

- [ ] **Step 4: Run tests — verify they fail**

Run: `mvn -f /Users/mdproctor/claude/casehub/worker/runtime/pom.xml test -Dtest="WorkerExecutorTest#execute_typedPojoInput_passedToFunction+execute_inputTypeMismatch_throwsIAE+execute_nullInput_throwsIAE" -pl .`
Expected: compilation failure — `execute(Worker, Capability, Map<String, Object>)` does not accept `TestPojo` or `null` without ambiguity

- [ ] **Step 5: Widen WorkerExecutor interface**

Use `ide_edit_member` to replace the interface declaration. The full interface with updated Javadoc:

```java
public interface WorkerExecutor {
    /**
     * Executes the given worker for the specified capability with policy enforcement.
     *
     * <p>Three exception categories:
     * <ul>
     *   <li><b>Worker-level conditions</b> (function exceptions, retry exhaustion,
     *       timeout) — returned as {@link WorkerResult} outcomes. Never propagate.</li>
     *   <li><b>Programming errors</b> (null capability, capability not in worker,
     *       non-Sync function, input type mismatch) — propagate as exceptions
     *       ({@code NullPointerException}, {@code IllegalArgumentException},
     *       {@code UnsupportedOperationException}). These are call-site bugs,
     *       not execution outcomes.</li>
     *   <li><b>Infrastructure signals</b> (thread interrupt, JVM errors) — propagate
     *       as exceptions.</li>
     * </ul>
     */
    WorkerResult execute(Worker worker, Capability capability, Object input);
}
```

- [ ] **Step 6: Update DefaultWorkerExecutor.execute()**

Use `ide_edit_member` to replace the entire `execute` method. Key changes:
- Parameter `Map<String, Object> input` → `Object input`
- Add input type validation after Sync extraction
- Raw-type `fn.apply(input)` instead of Map-casted invocation

```java
@SuppressWarnings({"unchecked", "rawtypes"})
@Override
public WorkerResult execute(Worker worker, Capability capability, Object input) {
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
    if (!sync.inputType().isInstance(input)) {
        throw new IllegalArgumentException(
                "Input type mismatch: expected " + sync.inputType().getName()
                + ", got " + (input == null ? "null" : input.getClass().getName()));
    }

    schemaValidator.ensureSchemaParsed(capability.inputSchema());
    schemaValidator.ensureSchemaParsed(capability.outputSchema());

    Span span = GlobalOpenTelemetry.getTracer(INSTRUMENTATION_NAME)
                                   .spanBuilder("worker.execute")
                                   .setAttribute(AttributeKey.stringKey("worker.name"), worker.name())
                                   .setAttribute(AttributeKey.stringKey("worker.capability"), capability.name())
                                   .startSpan();
    try (Scope ignored = span.makeCurrent()) {
        Optional<String> inputError = schemaValidator.validateInput(capability, input);
        if (inputError.isPresent()) {
            span.addEvent("worker.input.invalid", Attributes.of(
                    AttributeKey.stringKey("validation.error"), inputError.get()));
            WorkerResult result = WorkerResult.failed(inputError.get());
            span.setAttribute(AttributeKey.stringKey("worker.outcome"),
                              result.outcome().getClass().getSimpleName());
            return result;
        }

        WorkerResult result = policyEnforcer.execute(
                worker.executionPolicy(),
                () -> ((java.util.function.Function) sync.fn()).apply(input));

        if (result.outcome() instanceof WorkerOutcome.Success) {
            Optional<String> outputError = schemaValidator.validateOutput(capability, result.output());
            if (outputError.isPresent()) {
                span.addEvent("worker.output.invalid", Attributes.of(
                        AttributeKey.stringKey("validation.error"), outputError.get()));
                LOG.warnf("Output schema violation for worker '%s' capability '%s': %s",
                          worker.name(), capability.name(), outputError.get());
            }
        }

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
```

- [ ] **Step 7: Run new tests — verify they pass**

Run: `mvn -f /Users/mdproctor/claude/casehub/worker/runtime/pom.xml test -Dtest="WorkerExecutorTest#execute_typedPojoInput_passedToFunction+execute_inputTypeMismatch_throwsIAE+execute_nullInput_throwsIAE" -pl .`
Expected: PASS

- [ ] **Step 8: Run all runtime tests — verify no regressions**

Run: `mvn -f /Users/mdproctor/claude/casehub/worker/runtime/pom.xml test`
Expected: all tests pass

- [ ] **Step 9: Verify with ide_diagnostics**

Run `ide_diagnostics` on `WorkerExecutor.java` and `DefaultWorkerExecutor.java` — expect no errors.

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/worker add runtime/src/main/java/io/casehub/worker/runtime/WorkerExecutor.java runtime/src/main/java/io/casehub/worker/runtime/DefaultWorkerExecutor.java runtime/src/test/java/io/casehub/worker/runtime/WorkerExecutorTest.java
git -C /Users/mdproctor/claude/casehub/worker commit -m "refactor: widen WorkerExecutor to Object input — honour WorkerFunction<T>  Refs #10"
```

---

### Task 3: MockWorkerExecutor — widen and add type validation

**Files:**
- Modify: `testing/src/main/java/io/casehub/worker/testing/MockWorkerExecutor.java:25-47`
- Modify: `testing/src/test/java/io/casehub/worker/testing/MockWorkerExecutorTest.java`

**Interfaces:**
- Consumes: `WorkerExecutor.execute(Worker, Capability, Object)` — from Task 2
- Produces: `MockWorkerExecutor.execute(Worker, Capability, Object)` — matches interface

- [ ] **Step 1: Write typed POJO execution test for mock**

Add `TestPojo` record to `MockWorkerExecutorTest`:
```java
record TestPojo(String name, int age) {}
```

Add test:
```java
@Test
void execute_typedPojoInput_passedToFunction() {
    MockWorkerExecutor executor = new MockWorkerExecutor();
    Worker worker = Worker.builder()
        .name("typed").capabilityName("process")
        .<TestPojo>fn().apply(pojo -> WorkerResult.of(Map.of("greeting", "hello " + pojo.name())))
        .build();

    WorkerResult result = executor.execute(worker, cap("process"), new TestPojo("alice", 30));
    assertThat(result.outcome()).isInstanceOf(WorkerOutcome.Success.class);
    assertThat(result.output()).containsEntry("greeting", "hello alice");
}
```

- [ ] **Step 2: Write type mismatch test for mock**

```java
@Test
void execute_inputTypeMismatch_throwsIAE() {
    MockWorkerExecutor executor = new MockWorkerExecutor();
    Worker worker = Worker.builder()
        .name("typed").capabilityName("process")
        .<TestPojo>fn().apply(pojo -> WorkerResult.of(Map.of()))
        .build();

    assertThatThrownBy(() -> executor.execute(worker, cap("process"), Map.of("name", "alice")))
        .isInstanceOf(IllegalArgumentException.class)
        .hasMessageContaining("TestPojo");
}
```

- [ ] **Step 3: Run tests — verify they fail**

Run: `mvn -f /Users/mdproctor/claude/casehub/worker/testing/pom.xml test -Dtest="MockWorkerExecutorTest#execute_typedPojoInput_passedToFunction+execute_inputTypeMismatch_throwsIAE" -pl .`
Expected: compilation failure — `execute(Worker, Capability, Map<String, Object>)` does not accept `TestPojo`

- [ ] **Step 4: Update MockWorkerExecutor**

Use `ide_edit_member` to replace the `execute` method:

```java
@SuppressWarnings({"unchecked", "rawtypes"})
@Override
public WorkerResult execute(Worker worker, Capability capability, Object input) {
    Objects.requireNonNull(capability, "capability");
    if (!worker.capabilityNames().contains(capability.name())) {
        throw new IllegalArgumentException(
                "Capability '" + capability.name() + "' not in worker '"
                + worker.name() + "' capabilities: " + worker.capabilityNames());
    }
    executionCount.incrementAndGet();
    lastWorkerName.set(worker.name());
    lastCapabilityName.set(capability.name());
    if (!(worker.function() instanceof WorkerFunction.Sync sync)) {
        throw new UnsupportedOperationException(
                "MockWorkerExecutor supports Sync functions only, got: "
                + worker.function().getClass().getName());
    }
    if (!sync.inputType().isInstance(input)) {
        throw new IllegalArgumentException(
                "Input type mismatch: expected " + sync.inputType().getName()
                + ", got " + (input == null ? "null" : input.getClass().getName()));
    }
    try {
        return ((java.util.function.Function) sync.fn()).apply(input);
    } catch (Exception e) {
        String message = e.getMessage();
        if (message == null) {message = e.getClass().getName();}
        return WorkerResult.failed(message);
    }
}
```

- [ ] **Step 5: Run new tests — verify they pass**

Run: `mvn -f /Users/mdproctor/claude/casehub/worker/testing/pom.xml test -Dtest="MockWorkerExecutorTest#execute_typedPojoInput_passedToFunction+execute_inputTypeMismatch_throwsIAE" -pl .`
Expected: PASS

- [ ] **Step 6: Run all testing module tests — verify no regressions**

Run: `mvn -f /Users/mdproctor/claude/casehub/worker/testing/pom.xml test`
Expected: all tests pass

- [ ] **Step 7: Verify with ide_diagnostics**

Run `ide_diagnostics` on `MockWorkerExecutor.java` — expect no errors.

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/worker add testing/src/main/java/io/casehub/worker/testing/MockWorkerExecutor.java testing/src/test/java/io/casehub/worker/testing/MockWorkerExecutorTest.java
git -C /Users/mdproctor/claude/casehub/worker commit -m "refactor: MockWorkerExecutor widens to Object input with type validation  Refs #10"
```

---

### Task 4: Documentation + full build verification

**Files:**
- Modify: `runtime/src/main/java/io/casehub/worker/runtime/WorkerExecutor.java` (Javadoc — already updated in Task 2)
- Modify: `docs/DESIGN.md`

**Interfaces:**
- Consumes: all changes from Tasks 1-3
- Produces: updated documentation

- [ ] **Step 1: Update DESIGN.md execution contract**

Use the Edit tool (markdown, not source code) to update `docs/DESIGN.md`.

Add "input type mismatch" to the execution contract section. Replace the paragraph:

```
`WorkerExecutor.execute()` always returns a `WorkerResult` for worker-level conditions. Only infrastructure signals (thread interrupt, JVM errors) propagate as exceptions.
```

With:

```
`WorkerExecutor.execute()` always returns a `WorkerResult` for worker-level conditions. Programming errors propagate as exceptions. Infrastructure signals (thread interrupt, JVM errors) propagate as exceptions.

### Programming errors

| Error | Exception | When |
|-------|-----------|------|
| Null capability | `NullPointerException` | `capability` is null |
| Capability not in worker | `IllegalArgumentException` | `capability.name()` not in `worker.capabilityNames()` |
| Non-Sync function | `UnsupportedOperationException` | `worker.function()` is not `WorkerFunction.Sync` |
| Input type mismatch | `IllegalArgumentException` | `input` is not an instance of `WorkerFunction.inputType()` |
| Null input | `IllegalArgumentException` | `input` is null (subcase of type mismatch — `isInstance(null)` is false) |
| Malformed schema | `IllegalArgumentException` | `Capability.inputSchema()` or `outputSchema()` is not valid JSON Schema |
```

- [ ] **Step 2: Run full build**

Run: `mvn -f /Users/mdproctor/claude/casehub/worker/pom.xml clean install`
Expected: BUILD SUCCESS — all modules compile and all tests pass

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/worker add docs/DESIGN.md
git -C /Users/mdproctor/claude/casehub/worker commit -m "docs: update execution contract — add input type mismatch to programming errors  Refs #10"
```
