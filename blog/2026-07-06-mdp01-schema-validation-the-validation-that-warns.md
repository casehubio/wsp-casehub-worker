---
title: "Schema validation: the validation that warns"
date: 2026-07-06
author: mdp
tags: [worker, json-schema, validation, design]
---

Added JSON Schema validation to the worker executor — but with an asymmetry
that took some design thought: input validation rejects (returning a Failed
result before the function ever runs), while output validation only warns
(logging at WARN level but returning the result unchanged).

The rationale: input validation prevents wasting compute on garbage. If the
input doesn't match what the capability declared, there's no point running
the function. But output validation is observability, not enforcement — the
worker did its job, and a schema mismatch might be a documentation issue
rather than a bug. Silently dropping a valid result because the output schema
was slightly wrong would be worse than logging the mismatch.

The design review (7 rounds, 14 issues) surfaced a cross-repo naming
collision nobody had noticed: the engine uses `outputSchema` and
`inputSchema` as JQ projection expressions, while the worker API uses the
same names for JSON Schema validation documents. Today this is harmless
because all capabilities use `"{}"`, which is both valid JQ (creates an
empty object) and valid JSON Schema (validates everything). Once capabilities
carry real schemas, the engine's JQ fallback will interpret a JSON Schema
document as a JQ literal and produce wrong data. Filed as engine#677.

The `SchemaValidator` itself is straightforward — a CDI bean with a
`ConcurrentHashMap` cache that parses schemas once and reuses them. The
interesting design choice was where to check for malformed schemas: before
the OTel span (as a programming error, alongside null-capability and
capability-not-in-worker), not inside the try-catch where it would be
swallowed into a Failed result. A malformed schema is a build-time bug, not
a runtime condition.

Also bundled: MockWorkerExecutor hardening (#11). Two one-liners — an
`instanceof Sync` guard replacing a bare cast, and a null-capability test
for symmetry with DefaultWorkerExecutor.
