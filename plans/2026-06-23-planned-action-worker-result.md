# PlannedAction + Enriched WorkerResult Factories — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add PlannedAction record and enrich WorkerResult/WorkerOutcome with factories for planned actions and partial output (worker#2, prerequisite for engine#543).

**Architecture:** PlannedAction is a pure declaration record in the api module. It lives on `WorkerOutcome.Success` (not on `WorkerResult`) — the type system structurally enforces that only successful outcomes can carry an action. WorkerResult gains convenience factories for the new patterns.

**Tech Stack:** Java 21 records, sealed interfaces, JUnit 5, AssertJ

## Global Constraints

- Package: `io.casehub.worker.api`
- Module: `api/` only — no runtime or testing changes
- All commits reference `Refs #2`
- Build: `mvn -f /Users/mdproctor/claude/casehub/worker/pom.xml test`
- Single-module test: `mvn -f /Users/mdproctor/claude/casehub/worker/api/pom.xml test`

---

### Task 1: PlannedAction record

**Files:**
- Create: `api/src/test/java/io/casehub/worker/api/PlannedActionTest.java`
- Create: `api/src/main/java/io/casehub/worker/api/PlannedAction.java`

**Interfaces:**
- Consumes: nothing
- Produces: `PlannedAction(String description, String actionType, Map<String, Object> parameters)`, `PlannedAction.of(String, String)`, `PlannedAction.of(String, String, Map<String, Object>)`

- [ ] **Step 1: Write the failing tests**

Create `api/src/test/java/io/casehub/worker/api/PlannedActionTest.java`:

```java
package io.casehub.worker.api;

import org.junit.jupiter.api.Test;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class PlannedActionTest {

    @Test
    void construction_allFields() {
        PlannedAction pa = new PlannedAction("File SAR", "sar.file", Map.of("accountId", "ACC-123"));
        assertThat(pa.description()).isEqualTo("File SAR");
        assertThat(pa.actionType()).isEqualTo("sar.file");
        assertThat(pa.parameters()).containsEntry("accountId", "ACC-123");
    }

    @Test
    void construction_nullParameters_defaultsToEmptyMap() {
        PlannedAction pa = new PlannedAction("File SAR", "sar.file", null);
        assertThat(pa.parameters()).isEmpty();
    }

    @Test
    void construction_nullDescription_rejected() {
        assertThatThrownBy(() -> new PlannedAction(null, "sar.file", Map.of()))
            .isInstanceOf(NullPointerException.class);
    }

    @Test
    void construction_nullActionType_rejected() {
        assertThatThrownBy(() -> new PlannedAction("File SAR", null, Map.of()))
            .isInstanceOf(NullPointerException.class);
    }

    @Test
    void of_twoArg_defaultsParametersToEmpty() {
        PlannedAction pa = PlannedAction.of("Approve", "approval.grant");
        assertThat(pa.description()).isEqualTo("Approve");
        assertThat(pa.actionType()).isEqualTo("approval.grant");
        assertThat(pa.parameters()).isEmpty();
    }

    @Test
    void of_threeArg_passesAllFields() {
        Map<String, Object> params = Map.of("amount", 100);
        PlannedAction pa = PlannedAction.of("Transfer", "spend.transfer", params);
        assertThat(pa.description()).isEqualTo("Transfer");
        assertThat(pa.actionType()).isEqualTo("spend.transfer");
        assertThat(pa.parameters()).containsEntry("amount", 100);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn -f /Users/mdproctor/claude/casehub/worker/api/pom.xml test -pl . -Dtest=PlannedActionTest`
Expected: Compilation failure — `PlannedAction` does not exist.

- [ ] **Step 3: Write the implementation**

Create `api/src/main/java/io/casehub/worker/api/PlannedAction.java`:

```java
package io.casehub.worker.api;

import java.util.Map;
import java.util.Objects;

public record PlannedAction(String description, String actionType, Map<String, Object> parameters) {
    public PlannedAction {
        Objects.requireNonNull(description);
        Objects.requireNonNull(actionType);
        if (parameters == null) parameters = Map.of();
    }

    public static PlannedAction of(String description, String actionType) {
        return new PlannedAction(description, actionType, Map.of());
    }

    public static PlannedAction of(String description, String actionType, Map<String, Object> parameters) {
        return new PlannedAction(description, actionType, parameters);
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn -f /Users/mdproctor/claude/casehub/worker/api/pom.xml test -pl . -Dtest=PlannedActionTest`
Expected: All 6 tests PASS.

- [ ] **Step 5: Run full api test suite**

Run: `mvn -f /Users/mdproctor/claude/casehub/worker/api/pom.xml test`
Expected: All existing tests still PASS.

- [ ] **Step 6: Commit**

```bash
git add api/src/main/java/io/casehub/worker/api/PlannedAction.java api/src/test/java/io/casehub/worker/api/PlannedActionTest.java
git commit -m "feat(api): add PlannedAction record  Refs #2"
```

---

### Task 2: WorkerOutcome.Success gains PlannedAction

**Files:**
- Modify: `api/src/main/java/io/casehub/worker/api/WorkerOutcome.java`
- Modify: `api/src/test/java/io/casehub/worker/api/WorkerTest.java` (add tests)

**Interfaces:**
- Consumes: `PlannedAction` from Task 1
- Produces: `WorkerOutcome.Success(PlannedAction plannedAction)`, `WorkerOutcome.success()` (returns Success with null), `WorkerOutcome.success(PlannedAction)` (non-null required)

- [ ] **Step 1: Write the failing tests**

Add to `api/src/test/java/io/casehub/worker/api/WorkerTest.java` — append these test methods inside the class:

```java
@Test
void success_withoutAction_hasNullPlannedAction() {
    WorkerOutcome outcome = WorkerOutcome.success();
    assertThat(outcome).isInstanceOf(WorkerOutcome.Success.class);
    assertThat(((WorkerOutcome.Success) outcome).plannedAction()).isNull();
}

@Test
void success_withAction_carriesPlannedAction() {
    PlannedAction action = PlannedAction.of("File SAR", "sar.file");
    WorkerOutcome outcome = WorkerOutcome.success(action);
    assertThat(outcome).isInstanceOf(WorkerOutcome.Success.class);
    assertThat(((WorkerOutcome.Success) outcome).plannedAction()).isSameAs(action);
}

@Test
void success_withNullAction_rejected() {
    assertThatThrownBy(() -> WorkerOutcome.success(null))
        .isInstanceOf(NullPointerException.class);
}
```

Add the missing import at the top of WorkerTest.java:

```java
import static org.assertj.core.api.Assertions.assertThatThrownBy;
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn -f /Users/mdproctor/claude/casehub/worker/api/pom.xml test -pl . -Dtest=WorkerTest`
Expected: `success_withoutAction_hasNullPlannedAction` fails — `Success` has no `plannedAction()` accessor. `success_withAction_carriesPlannedAction` fails — no `success(PlannedAction)` overload. `success_withNullAction_rejected` fails — same.

- [ ] **Step 3: Implement the changes**

Replace the full content of `api/src/main/java/io/casehub/worker/api/WorkerOutcome.java`:

```java
package io.casehub.worker.api;

import java.util.Objects;

public sealed interface WorkerOutcome {
    static WorkerOutcome success() { return new Success(null); }
    static WorkerOutcome success(PlannedAction action) {
        Objects.requireNonNull(action);
        return new Success(action);
    }
    record Success(PlannedAction plannedAction) implements WorkerOutcome {}
    record Declined(String reason) implements WorkerOutcome {}
    record Failed(String reason) implements WorkerOutcome {}
    record Expired(String reason) implements WorkerOutcome {}
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn -f /Users/mdproctor/claude/casehub/worker/api/pom.xml test -pl . -Dtest=WorkerTest`
Expected: All tests PASS (new tests + existing tests).

- [ ] **Step 5: Run full api test suite**

Run: `mvn -f /Users/mdproctor/claude/casehub/worker/api/pom.xml test`
Expected: All tests PASS. Existing `isInstanceOf(WorkerOutcome.Success.class)` assertions are unaffected by the added component.

- [ ] **Step 6: Commit**

```bash
git add api/src/main/java/io/casehub/worker/api/WorkerOutcome.java api/src/test/java/io/casehub/worker/api/WorkerTest.java
git commit -m "feat(api): add PlannedAction to WorkerOutcome.Success  Refs #2"
```

---

### Task 3: WorkerResult factory additions

**Files:**
- Modify: `api/src/main/java/io/casehub/worker/api/WorkerResult.java`
- Modify: `api/src/test/java/io/casehub/worker/api/WorkerTest.java` (add tests)

**Interfaces:**
- Consumes: `PlannedAction` from Task 1, `WorkerOutcome.Success(PlannedAction)` from Task 2
- Produces: `WorkerResult.of(Map, PlannedAction)`, `WorkerResult.declined(String, Map)`, `WorkerResult.failed(String, Map)`, `WorkerResult.expired(String, Map)`

- [ ] **Step 1: Write the failing tests**

Add to `api/src/test/java/io/casehub/worker/api/WorkerTest.java` — append these test methods inside the class:

```java
@Test
void workerResult_ofWithAction_createsSuccessWithPlannedAction() {
    PlannedAction action = PlannedAction.of("File SAR", "sar.file", Map.of("accountId", "ACC-123"));
    WorkerResult result = WorkerResult.of(Map.of("key", "value"), action);
    assertThat(result.outcome()).isInstanceOf(WorkerOutcome.Success.class);
    WorkerOutcome.Success success = (WorkerOutcome.Success) result.outcome();
    assertThat(success.plannedAction()).isSameAs(action);
    assertThat(result.output()).containsEntry("key", "value");
}

@Test
void workerResult_ofWithNullAction_rejected() {
    assertThatThrownBy(() -> WorkerResult.of(Map.of(), (PlannedAction) null))
        .isInstanceOf(NullPointerException.class);
}

@Test
void workerResult_declinedWithPartialOutput() {
    Map<String, Object> partial = Map.of("progress", "50%");
    WorkerResult result = WorkerResult.declined("not my job", partial);
    assertThat(result.outcome()).isInstanceOf(WorkerOutcome.Declined.class);
    assertThat(((WorkerOutcome.Declined) result.outcome()).reason()).isEqualTo("not my job");
    assertThat(result.output()).isEqualTo(partial);
}

@Test
void workerResult_failedWithPartialOutput() {
    Map<String, Object> partial = Map.of("step", "validation");
    WorkerResult result = WorkerResult.failed("broken", partial);
    assertThat(result.outcome()).isInstanceOf(WorkerOutcome.Failed.class);
    assertThat(((WorkerOutcome.Failed) result.outcome()).reason()).isEqualTo("broken");
    assertThat(result.output()).isEqualTo(partial);
}

@Test
void workerResult_expiredWithPartialOutput() {
    Map<String, Object> partial = Map.of("elapsed", "30s");
    WorkerResult result = WorkerResult.expired("too slow", partial);
    assertThat(result.outcome()).isInstanceOf(WorkerOutcome.Expired.class);
    assertThat(((WorkerOutcome.Expired) result.outcome()).reason()).isEqualTo("too slow");
    assertThat(result.output()).isEqualTo(partial);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn -f /Users/mdproctor/claude/casehub/worker/api/pom.xml test -pl . -Dtest=WorkerTest`
Expected: Compilation failure — `of(Map, PlannedAction)`, `declined(String, Map)`, `failed(String, Map)`, `expired(String, Map)` do not exist on WorkerResult.

- [ ] **Step 3: Implement the changes**

Add these methods to `api/src/main/java/io/casehub/worker/api/WorkerResult.java`, after the existing `of(Map)` method:

```java
public static WorkerResult of(Map<String, Object> output, PlannedAction action) {
    Objects.requireNonNull(action);
    return new WorkerResult(output, new WorkerOutcome.Success(action));
}
```

Add the `Objects` import if not present:

```java
import java.util.Objects;
```

Add partial-output overloads after the existing `declined(String)`, `failed(String)`, `expired(String)` methods respectively:

After `declined(String)`:
```java
public static WorkerResult declined(String reason, Map<String, Object> partialOutput) {
    return new WorkerResult(partialOutput, new WorkerOutcome.Declined(reason));
}
```

After `failed(String)`:
```java
public static WorkerResult failed(String reason, Map<String, Object> partialOutput) {
    return new WorkerResult(partialOutput, new WorkerOutcome.Failed(reason));
}
```

After `expired(String)`:
```java
public static WorkerResult expired(String reason, Map<String, Object> partialOutput) {
    return new WorkerResult(partialOutput, new WorkerOutcome.Expired(reason));
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn -f /Users/mdproctor/claude/casehub/worker/api/pom.xml test -pl . -Dtest=WorkerTest`
Expected: All tests PASS (new + existing).

- [ ] **Step 5: Run full project test suite**

Run: `mvn -f /Users/mdproctor/claude/casehub/worker/pom.xml test`
Expected: All modules PASS — api, runtime, testing.

- [ ] **Step 6: Commit**

```bash
git add api/src/main/java/io/casehub/worker/api/WorkerResult.java api/src/test/java/io/casehub/worker/api/WorkerTest.java
git commit -m "feat(api): add WorkerResult factories for planned actions and partial output  Refs #2"
```

- [ ] **Step 7: Install to local Maven repo**

Run: `mvn -f /Users/mdproctor/claude/casehub/worker/pom.xml clean install`
Expected: BUILD SUCCESS. Updated `casehub-worker-api-0.2-SNAPSHOT.jar` in `~/.m2/repository/`. Engine#543 can now consume the changes.
