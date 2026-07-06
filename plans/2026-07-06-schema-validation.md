# Schema Validation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #7 — feat: schema validation — enforce Capability inputSchema and outputSchema
**Issue group:** #7, #11

**Goal:** Validate worker inputs against `Capability.inputSchema()` before execution
(reject invalid) and outputs against `Capability.outputSchema()` after execution
(warn on invalid). Harden `MockWorkerExecutor` with Sync guard and null-capability test.

**Architecture:** A `SchemaValidator` CDI bean in the runtime module owns JSON Schema
parsing, caching, and validation. `DefaultWorkerExecutor` calls it at three points:
schema parse guard (programming error), input validation (before function), output
validation (after function, Success only). `MockWorkerExecutor` does not validate
schemas — it's a test double.

**Tech Stack:** Java 21, Quarkus 3.32.2, networknt json-schema-validator 1.0.83,
Jackson 2.x, OpenTelemetry API, JUnit 5, AssertJ

## Global Constraints

- `casehub-worker-api` is Tier 1 pure-Java — no Quarkus, no JPA, no new dependencies
- `casehub-worker-runtime` is Tier 3 — Quarkus, CDI, OTel
- `casehub-worker-testing` — `@DefaultBean` mock, test utilities
- Build: `mvn -f /Users/mdproctor/claude/casehub/worker/pom.xml clean install`
- Test: `mvn -f /Users/mdproctor/claude/casehub/worker/pom.xml test`
- Single module test: `mvn -f /Users/mdproctor/claude/casehub/worker/<module>/pom.xml test`
- Spec: `docs/specs/2026-07-06-schema-validation-design.md`
- networknt version 1.0.83 (Jackson 2 compatible — 3.x requires Jackson 3, incompatible with Quarkus)

---

### Task 1: MockWorkerExecutor hardening (#11)

**Files:**
- Modify: `testing/src/main/java/io/casehub/worker/testing/MockWorkerExecutor.java:33-34`
- Modify: `testing/src/test/java/io/casehub/worker/testing/MockWorkerExecutorTest.java`

**Interfaces:**
- Consumes: `WorkerFunction.Sync`, `WorkerFunction.NONE` from `casehub-worker-api`
- Produces: No new interfaces — hardening existing behavior to match `DefaultWorkerExecutor`

- [ ] **Step 1: Write the null-capability test**

Add to `testing/src/test/java/io/casehub/worker/testing/MockWorkerExecutorTest.java`:

```java
@Test
void execute_nullCapability_throwsNPE() {
    MockWorkerExecutor executor = new MockWorkerExecutor();
    Worker worker = TestWorkerBuilder.sync("w",
        input -> WorkerResult.of(Map.of()));

    assertThatThrownBy(() -> executor.execute(worker, null, Map.of()))
        .isInstanceOf(NullPointerException.class)
        .hasMessageContaining("capability");
}
```

- [ ] **Step 2: Write the non-Sync function test**

Add to `MockWorkerExecutorTest.java`:

```java
@Test
void execute_nonSyncFunction_throwsUnsupported() {
    MockWorkerExecutor executor = new MockWorkerExecutor();
    Worker worker = Worker.builder()
        .name("external").capabilityName("ext")
        .noFunction()
        .build();

    assertThatThrownBy(() -> executor.execute(worker, cap("ext"), Map.of()))
        .isInstanceOf(UnsupportedOperationException.class)
        .hasMessageContaining("Sync");
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn -f /Users/mdproctor/claude/casehub/worker/testing/pom.xml test`
Expected: 2 failures — null capability produces NPE (already works due to
`Objects.requireNonNull`), but non-Sync produces `ClassCastException` not
`UnsupportedOperationException`.

- [ ] **Step 4: Add Sync guard to MockWorkerExecutor**

In `testing/src/main/java/io/casehub/worker/testing/MockWorkerExecutor.java`,
replace the direct cast at lines 33-34:

```java
// Replace this:
try {
    return ((io.casehub.worker.api.WorkerFunction.Sync) worker.function()).fn().apply(input);

// With this:
if (!(worker.function() instanceof WorkerFunction.Sync sync)) {
    throw new UnsupportedOperationException(
        "MockWorkerExecutor supports Sync functions only, got: "
            + worker.function().getClass().getName());
}
try {
    return sync.fn().apply(input);
```

Add import if not present: `import io.casehub.worker.api.WorkerFunction;`

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn -f /Users/mdproctor/claude/casehub/worker/testing/pom.xml test`
Expected: All tests pass including the two new ones.

- [ ] **Step 6: Run full build**

Run: `mvn -f /Users/mdproctor/claude/casehub/worker/pom.xml clean install`
Expected: BUILD SUCCESS — no regressions.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/worker add testing/
git -C /Users/mdproctor/claude/casehub/worker commit -m "chore: MockWorkerExecutor hardening — Sync guard + null-capability test  Closes #11"
```

---

### Task 2: SchemaValidator

**Files:**
- Modify: `runtime/pom.xml`
- Create: `runtime/src/main/java/io/casehub/worker/runtime/SchemaValidator.java`
- Create: `runtime/src/test/java/io/casehub/worker/runtime/SchemaValidatorTest.java`
- Modify: `testing/src/main/java/io/casehub/worker/testing/TestWorkerBuilder.java`

**Interfaces:**
- Consumes: `Capability` from `casehub-worker-api`
- Produces:
  - `SchemaValidator.ensureSchemaParsed(String schema)` — throws `IllegalArgumentException` if malformed
  - `SchemaValidator.validateInput(Capability, Map<String, Object>)` → `Optional<String>`
  - `SchemaValidator.validateOutput(Capability, Map<String, Object>)` → `Optional<String>`
  - `TestWorkerBuilder.syncWithCapability(String name, String inputSchema, String outputSchema, Function)` → `WorkerWithCapability`

- [ ] **Step 1: Add networknt dependency to runtime/pom.xml**

Add to `runtime/pom.xml` in the `<dependencies>` section, after `opentelemetry-api`:

```xml
<dependency>
    <groupId>com.networknt</groupId>
    <artifactId>json-schema-validator</artifactId>
    <version>1.0.83</version>
</dependency>
```

Version pinned locally — not in casehub-parent BOM yet.

- [ ] **Step 2: Add schema-aware TestWorkerBuilder overload**

Add to `testing/src/main/java/io/casehub/worker/testing/TestWorkerBuilder.java`:

```java
public static WorkerWithCapability syncWithCapability(String name,
        String inputSchema, String outputSchema,
        Function<Map<String, Object>, WorkerResult> fn) {
    Worker worker = Worker.builder()
        .name(name)
        .capabilityName(name)
        .function(fn)
        .build();
    Capability capability = Capability.of(name, inputSchema, outputSchema);
    return new WorkerWithCapability(worker, capability);
}
```

- [ ] **Step 3: Write SchemaValidator tests**

Create `runtime/src/test/java/io/casehub/worker/runtime/SchemaValidatorTest.java`:

```java
package io.casehub.worker.runtime;

import io.casehub.worker.api.Capability;
import org.junit.jupiter.api.Test;

import java.util.Map;
import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class SchemaValidatorTest {

    private final SchemaValidator validator = new SchemaValidator();

    private static final String REQUIRE_NAME_SCHEMA = """
        {
          "type": "object",
          "properties": {
            "name": { "type": "string" }
          },
          "required": ["name"]
        }""";

    private static final String REQUIRE_RESULT_SCHEMA = """
        {
          "type": "object",
          "properties": {
            "result": { "type": "number" }
          },
          "required": ["result"]
        }""";

    @Test
    void validateInput_validData_returnsEmpty() {
        Capability cap = Capability.of("c", REQUIRE_NAME_SCHEMA, "{}");
        Optional<String> result = validator.validateInput(cap, Map.of("name", "Alice"));
        assertThat(result).isEmpty();
    }

    @Test
    void validateInput_missingRequiredField_returnsError() {
        Capability cap = Capability.of("c", REQUIRE_NAME_SCHEMA, "{}");
        Optional<String> result = validator.validateInput(cap, Map.of("age", 30));
        assertThat(result).isPresent();
        assertThat(result.get()).contains("name");
    }

    @Test
    void validateInput_wrongType_returnsError() {
        Capability cap = Capability.of("c", REQUIRE_NAME_SCHEMA, "{}");
        Optional<String> result = validator.validateInput(cap, Map.of("name", 42));
        assertThat(result).isPresent();
        assertThat(result.get()).containsIgnoringCase("string");
    }

    @Test
    void validateOutput_validData_returnsEmpty() {
        Capability cap = Capability.of("c", "{}", REQUIRE_RESULT_SCHEMA);
        Optional<String> result = validator.validateOutput(cap, Map.of("result", 42));
        assertThat(result).isEmpty();
    }

    @Test
    void validateOutput_invalidData_returnsError() {
        Capability cap = Capability.of("c", "{}", REQUIRE_RESULT_SCHEMA);
        Optional<String> result = validator.validateOutput(cap, Map.of("result", "not-a-number"));
        assertThat(result).isPresent();
    }

    @Test
    void validateInput_emptySchema_skipsValidation() {
        Capability cap = Capability.of("c", "{}", "{}");
        Optional<String> result = validator.validateInput(cap, Map.of("anything", "goes"));
        assertThat(result).isEmpty();
    }

    @Test
    void validateOutput_emptySchema_skipsValidation() {
        Capability cap = Capability.of("c", "{}", "{}");
        Optional<String> result = validator.validateOutput(cap, Map.of("anything", "goes"));
        assertThat(result).isEmpty();
    }

    @Test
    void ensureSchemaParsed_validSchema_noException() {
        validator.ensureSchemaParsed(REQUIRE_NAME_SCHEMA);
    }

    @Test
    void ensureSchemaParsed_emptySchema_noException() {
        validator.ensureSchemaParsed("{}");
    }

    @Test
    void ensureSchemaParsed_malformedSchema_throwsIAE() {
        assertThatThrownBy(() -> validator.ensureSchemaParsed("not valid json"))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("Malformed JSON Schema");
    }

    @Test
    void ensureSchemaParsed_cachesSchema() {
        validator.ensureSchemaParsed(REQUIRE_NAME_SCHEMA);
        Capability cap = Capability.of("c", REQUIRE_NAME_SCHEMA, "{}");
        Optional<String> result = validator.validateInput(cap, Map.of("name", "Alice"));
        assertThat(result).isEmpty();
    }
}
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `mvn -f /Users/mdproctor/claude/casehub/worker/runtime/pom.xml test -pl . -Dtest=SchemaValidatorTest`
Expected: Compilation failure — `SchemaValidator` class does not exist.

- [ ] **Step 5: Create SchemaValidator**

Create `runtime/src/main/java/io/casehub/worker/runtime/SchemaValidator.java`:

```java
package io.casehub.worker.runtime;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.networknt.schema.JsonSchema;
import com.networknt.schema.JsonSchemaFactory;
import com.networknt.schema.SpecVersion;
import com.networknt.schema.ValidationMessage;
import io.casehub.worker.api.Capability;
import jakarta.enterprise.context.ApplicationScoped;

import java.util.Map;
import java.util.Optional;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;
import java.util.stream.Collectors;

@ApplicationScoped
public class SchemaValidator {

    private static final String EMPTY_SCHEMA = "{}";

    private final ObjectMapper objectMapper = new ObjectMapper();
    private final JsonSchemaFactory schemaFactory =
        JsonSchemaFactory.getInstance(SpecVersion.VersionFlag.V202012);
    private final ConcurrentHashMap<String, JsonSchema> cache = new ConcurrentHashMap<>();

    public void ensureSchemaParsed(String schema) {
        if (EMPTY_SCHEMA.equals(schema)) return;
        cache.computeIfAbsent(schema, this::parseSchema);
    }

    public Optional<String> validateInput(Capability capability, Map<String, Object> input) {
        return validate(capability.inputSchema(), input);
    }

    public Optional<String> validateOutput(Capability capability, Map<String, Object> output) {
        return validate(capability.outputSchema(), output);
    }

    private Optional<String> validate(String schemaString, Map<String, Object> data) {
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

    private JsonSchema parseSchema(String schemaString) {
        try {
            JsonNode schemaNode = objectMapper.readTree(schemaString);
            return schemaFactory.getSchema(schemaNode);
        } catch (Exception e) {
            throw new IllegalArgumentException(
                "Malformed JSON Schema: " + e.getMessage(), e);
        }
    }
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `mvn -f /Users/mdproctor/claude/casehub/worker/runtime/pom.xml test -Dtest=SchemaValidatorTest`
Expected: All 11 tests pass.

- [ ] **Step 7: Run full build**

Run: `mvn -f /Users/mdproctor/claude/casehub/worker/pom.xml clean install`
Expected: BUILD SUCCESS — all existing tests still pass (they use `"{}"` schemas
which skip validation).

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/worker add runtime/pom.xml runtime/src/ testing/src/main/
git -C /Users/mdproctor/claude/casehub/worker commit -m "feat: SchemaValidator — JSON Schema validation with caching  Refs #7"
```

---

### Task 3: DefaultWorkerExecutor integration

**Files:**
- Modify: `runtime/src/main/java/io/casehub/worker/runtime/DefaultWorkerExecutor.java`
- Modify: `runtime/src/test/java/io/casehub/worker/runtime/WorkerExecutorTest.java`

**Interfaces:**
- Consumes: `SchemaValidator.ensureSchemaParsed(String)`, `SchemaValidator.validateInput(Capability, Map)`, `SchemaValidator.validateOutput(Capability, Map)` from Task 2
- Produces: No new interfaces — extends existing `WorkerExecutor.execute()` behavior

- [ ] **Step 1: Write the invalid-input test**

Add to `runtime/src/test/java/io/casehub/worker/runtime/WorkerExecutorTest.java`:

```java
private static final String REQUIRE_NAME_SCHEMA = """
    {
      "type": "object",
      "properties": {
        "name": { "type": "string" }
      },
      "required": ["name"]
    }""";

private static Capability cap(String name, String inputSchema, String outputSchema) {
    return Capability.of(name, inputSchema, outputSchema);
}
```

Then add the test:

```java
@Test
void execute_invalidInput_returnsFailed_functionNeverCalled() {
    AtomicInteger callCount = new AtomicInteger(0);
    Worker worker = Worker.builder()
        .name("strict").capabilityName("validate")
        .function(new WorkerFunction.Sync(input -> {
            callCount.incrementAndGet();
            return WorkerResult.of(Map.of());
        }))
        .build();

    WorkerResult result = executor.execute(worker,
        cap("validate", REQUIRE_NAME_SCHEMA, "{}"),
        Map.of("age", 30));

    assertThat(result.outcome()).isInstanceOf(WorkerOutcome.Failed.class);
    assertThat(((WorkerOutcome.Failed) result.outcome()).reason()).contains("name");
    assertThat(callCount.get()).isZero();
}
```

- [ ] **Step 2: Write the valid-input-invalid-output test**

```java
private static final String REQUIRE_RESULT_SCHEMA = """
    {
      "type": "object",
      "properties": {
        "result": { "type": "number" }
      },
      "required": ["result"]
    }""";

@Test
void execute_validInput_invalidOutput_returnsSuccessWithWarning() {
    Worker worker = Worker.builder()
        .name("bad-output").capabilityName("compute")
        .function(new WorkerFunction.Sync(input ->
            WorkerResult.of(Map.of("result", "not-a-number"))))
        .build();

    WorkerResult result = executor.execute(worker,
        cap("compute", "{}", REQUIRE_RESULT_SCHEMA),
        Map.of());

    assertThat(result.outcome()).isInstanceOf(WorkerOutcome.Success.class);
    assertThat(result.output()).containsEntry("result", "not-a-number");
}
```

- [ ] **Step 3: Write the malformed-schema test**

```java
@Test
void execute_malformedSchema_throwsIAE() {
    Worker worker = Worker.builder()
        .name("broken").capabilityName("bad")
        .function(new WorkerFunction.Sync(input -> WorkerResult.of(Map.of())))
        .build();

    assertThatThrownBy(() -> executor.execute(worker,
            cap("bad", "not valid json", "{}"), Map.of()))
        .isInstanceOf(IllegalArgumentException.class);
}
```

- [ ] **Step 4: Write the Declined-partial-output test**

```java
@Test
void execute_declinedWithPartialOutput_noOutputValidation() {
    Worker worker = Worker.builder()
        .name("decliner").capabilityName("dec")
        .function(new WorkerFunction.Sync(input ->
            WorkerResult.declined("nope", Map.of("partial", "data"))))
        .build();

    WorkerResult result = executor.execute(worker,
        cap("dec", "{}", REQUIRE_RESULT_SCHEMA),
        Map.of());

    assertThat(result.outcome()).isInstanceOf(WorkerOutcome.Declined.class);
    assertThat(result.output()).containsEntry("partial", "data");
}
```

- [ ] **Step 5: Write the empty-schema backward-compatibility test**

```java
@Test
void execute_emptySchemas_noValidation() {
    Worker worker = Worker.builder()
        .name("legacy").capabilityName("old")
        .function(new WorkerFunction.Sync(input ->
            WorkerResult.of(Map.of("anything", "goes"))))
        .build();

    WorkerResult result = executor.execute(worker, cap("old"), Map.of("random", 42));
    assertThat(result.outcome()).isInstanceOf(WorkerOutcome.Success.class);
}
```

- [ ] **Step 6: Run tests to verify they fail**

Run: `mvn -f /Users/mdproctor/claude/casehub/worker/runtime/pom.xml test -Dtest=WorkerExecutorTest`
Expected: Compilation failure — `DefaultWorkerExecutor` constructor requires only
`PolicyEnforcer`, not `SchemaValidator`. And the new test methods reference `cap()`
overload that doesn't exist yet.

- [ ] **Step 7: Update DefaultWorkerExecutor constructor**

In `runtime/src/main/java/io/casehub/worker/runtime/DefaultWorkerExecutor.java`:

Add field and update constructor:

```java
private final PolicyEnforcer policyEnforcer;
private final SchemaValidator schemaValidator;

@Inject
public DefaultWorkerExecutor(PolicyEnforcer policyEnforcer, SchemaValidator schemaValidator) {
    this.policyEnforcer = policyEnforcer;
    this.schemaValidator = schemaValidator;
}
```

Add logger:

```java
private static final org.jboss.logging.Logger LOG =
    org.jboss.logging.Logger.getLogger(DefaultWorkerExecutor.class);
```

- [ ] **Step 8: Add schema parse guard**

After the `instanceof Sync` check and before the span creation, add:

```java
schemaValidator.ensureSchemaParsed(capability.inputSchema());
schemaValidator.ensureSchemaParsed(capability.outputSchema());
```

- [ ] **Step 9: Add input validation**

Inside the try block, before `policyEnforcer.execute()`, add:

```java
Optional<String> inputError = schemaValidator.validateInput(capability, input);
if (inputError.isPresent()) {
    span.addEvent("worker.input.invalid", Attributes.of(
        AttributeKey.stringKey("validation.error"), inputError.get()));
    WorkerResult result = WorkerResult.failed(inputError.get());
    span.setAttribute(AttributeKey.stringKey("worker.outcome"),
        result.outcome().getClass().getSimpleName());
    return result;
}
```

Add import: `import java.util.Optional;`

- [ ] **Step 10: Add output validation**

After `policyEnforcer.execute()` returns the result, before the existing
`span.setAttribute(...)` line, add:

```java
if (result.outcome() instanceof WorkerOutcome.Success) {
    Optional<String> outputError = schemaValidator.validateOutput(capability, result.output());
    if (outputError.isPresent()) {
        span.addEvent("worker.output.invalid", Attributes.of(
            AttributeKey.stringKey("validation.error"), outputError.get()));
        LOG.warnf("Output schema violation for worker '%s' capability '%s': %s",
            worker.name(), capability.name(), outputError.get());
    }
}
```

- [ ] **Step 11: Update test constructor**

In `WorkerExecutorTest.java`, update the executor field:

```java
private final DefaultWorkerExecutor executor =
    new DefaultWorkerExecutor(new DefaultPolicyEnforcer(), new SchemaValidator());
```

Add the two-parameter `cap()` overload alongside the existing one:

```java
private static Capability cap(String name, String inputSchema, String outputSchema) {
    return Capability.of(name, inputSchema, outputSchema);
}
```

- [ ] **Step 12: Update WorkerExecutorCdiTest if needed**

Check `runtime/src/test/java/io/casehub/worker/runtime/WorkerExecutorCdiTest.java`
— if it instantiates `DefaultWorkerExecutor` directly, update the constructor call
to include `SchemaValidator`. If it uses CDI injection, no change needed.

- [ ] **Step 13: Run tests to verify they pass**

Run: `mvn -f /Users/mdproctor/claude/casehub/worker/runtime/pom.xml test`
Expected: All tests pass — existing tests use `cap("name")` which returns
`Capability.of(name, "{}", "{}")`, so schema validation is skipped.

- [ ] **Step 14: Run full build**

Run: `mvn -f /Users/mdproctor/claude/casehub/worker/pom.xml clean install`
Expected: BUILD SUCCESS — all modules pass.

- [ ] **Step 15: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/worker add runtime/
git -C /Users/mdproctor/claude/casehub/worker commit -m "feat: schema validation in DefaultWorkerExecutor — input rejects, output warns  Closes #7"
```
