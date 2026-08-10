# Phase 2 — Complete `common-java` and `common-go`

> **Historical trust note:** G2 implemented the original selected-gym trusted-header profile. [Phase 9 Stage 0](09-kong-grpc-gateway-openapi.md#stage-0--replace-selected-gym-jwt-state-before-gateway-generation) removes trusted `gym_id` and `membership_status` claims; public services accept identity/role metadata only from the approved gateway SAN and read gym context from validated requests. Preserve G2 evidence unchanged.

## Objective

Reach G2 by implementing the frozen auth, error, tracing, Kafka, retry, commit, and DLQ contracts in both shared libraries. The two lanes run in parallel after G1.

## Prerequisites

- [Phase 0](00-contract-freeze.md) passed G0.
- [Phase 1](01-gym-proto.md) published matching G1 Java and Go artifacts.
- Read [roadmap rules and adoption freeze](README.md).
- Recheck repository SHAs, tags, and working trees.

Rechecked baselines on 2026-08-03:

- `common-java`: `develop` at `3297115`; only legacy tag `v1.0.1`; current source version is `2.0.0-SNAPSHOT`.
- `common-go`: `develop` at `82f7ab7`; no tag.

## Current status: G2 technically passed

The exact `common-java` and `common-go` working trees recorded in [the G2 evidence record](../../evidence/foundation-first/g2/3297115bf333a3781e4248535a7c965d14cb0388--82f7ab7fb7d6e26fa6db3f076f707883b87e7d39.md) passed source/static/race gates and real Kafka/Schema Registry 7.7.1 tests against published `gym-proto` / `proto-go v1.1.0`.

Closeout evidence confirms:

1. Java's JSON `EventEnvelope`, serializers, `EventConsumer`, and `RetryableConsumer` classes are absent and guarded against reintroduction.
2. Java and Go validation workflows provision Kafka and Schema Registry before running live contract suites.
3. Live G2 suites prove retry/commit ordering, failed-DLQ redelivery, cancellation, same-group replacement, and raw DLQ preservation. Exact 2/4/8-second waits remain deterministic unit-test assertions.
4. Java source version and both library READMEs describe the unreleased G2 line consistently; legal Maven POM identity metadata remains explicitly deferred.
5. The consolidated record binds exact source-tree checksums, published proto resolution, live transport outcomes, Registry stability, raw-material hashes, and analysis results.

Cross-language RC artifacts, external common-library resolution, and Java-to-Go / Go-to-Java transport validation remain Phase 3 work, not G2 prerequisites.

Do not reimplement working producer, consumer, framing, retry, or DLQ behavior merely because an earlier audit baseline described it as absent. Preserve the verified behavior and evidence.

## Explicit non-goals

- No ordinary-service adoption.
- No Kong prerequisite for Kafka work.
- No fabricated owner approval.
- No support for undeployed JSON topics or envelope records.
- No DB/Kafka exactly-once claim.

---

## Lane A — `common-java`

### A1. Dependency and release hygiene

Modify `build.gradle` and `.github/workflows/publish.yml`:

1. Pin the exact G1 `gym-proto-java:1.1.0` artifact.
2. Exclude `mavenLocal()` entirely.
3. Use `2.0.0-SNAPSHOT` for source builds and `RELEASE_VERSION` only after a validated immutable tag; do not keep a mutable `version.properties` counter.
4. Stop publishing from `develop`/`main` branch pushes.
5. Require `check`, coverage verification, the reusable provisioned contract workflow, dependency metadata, and available security-report evidence before publish.
6. Publish source/Javadoc artifacts and POM metadata containing the exact proto dependency plus verified repository identity/SCM facts.
7. Defer license, organization, developer, and other legal metadata until repository-owner input is recorded.

### A2. Authentication and method policy

Update:

- `AuthServerInterceptor.java`
- `GrpcMethodRegistry.java`
- `UserClaims.java`
- annotations/configuration representing method policy

Implement:

1. Parse `x-user-id`, `x-user-role`, `x-gym-id`, and `x-membership-status`.
2. Support `NONE`, `ACTIVE`, `PAUSED`, `EXPIRED`.
3. Trim and normalize role/status values to canonical uppercase.
4. Reject blank, malformed, unknown, and conflicting duplicate values.
5. Separate user and workload identities.
6. Classify every registered RPC explicitly as:
   - public;
   - authenticated;
   - role-restricted;
   - active-membership;
   - internal workload.
7. Fail closed or fail startup when an RPC is unclassified.
8. Resolve policy on the actual registered `BindableService` override; do not assume annotations on delegates are inherited.
9. Keep gym/resource authorization in services; shared policy handles method-level trust and roles.

Tests cover unary/stream, public/authenticated, every role/status, missing/conflicting metadata, and unclassified-service startup.

### A3. Error, logging, and tracing safety

Update `ExceptionInterceptor.java`, `LoggingInterceptor.java`, and tracing helpers:

- map categories to the frozen gRPC status contract;
- emit `x-error-code` consistently;
- redact all `INTERNAL` messages, including explicitly categorized domain errors;
- allow client-safe text only through an explicit safe contract;
- use valid W3C context first and `x-trace-id` only as fallback;
- avoid arbitrary `status.getDescription()` logging;
- never log payloads, tokens, passwords, raw headers, or user IDs by default;
- keep metrics labels low cardinality.

### A4. Deterministic Protobuf consumption

Replace the generic `ConsumerFactory<String,Object>` behavior in `KafkaAutoConfig.java` with deterministic generated-message delivery while preserving raw records.

Recommended design:

1. Kafka consumer receives raw key/value bytes.
2. A shared decoder validates the Confluent frame and resolves schema metadata.
3. `RawKafkaListenerAdapter` bridges a service-owned Spring raw listener to the existing `RawDeliveryCoordinator`.
4. Decode into the generated message resolved by the Confluent frame and validate canonical `event-type` against the descriptor.
5. Preserve raw key, frame, and headers beside the decoded message.
6. Treat malformed frame, unsupported type, missing required header, and descriptor mismatch as permanent failures.
7. Treat transient Registry/broker availability as retryable.

Do not rely on a generic `KafkaProtobufDeserializer` returning the class expected by listener signatures unless a live multi-type test proves that behavior and the configuration is explicit.

### A5. Retry, commit, and raw-byte DLQ

Implement:

- first handler attempt;
- retries after 2, 4, and 8 seconds;
- source offset commit after handler success;
- source offset commit after confirmed DLQ publication;
- no commit when DLQ publication fails;
- redelivery after restart/rebalance when unfinished.

Use separate producer paths:

- concrete Protobuf template for normal events;
- byte-array template for DLQ forwarding.

DLQ preserves byte-for-byte:

- original key;
- complete framed value;
- original headers.

Then add/replace only:

```text
x-original-topic
x-exception-message
x-failed-at
x-retry-count
```

Diagnostics are client-safe and bounded.

### A6. Producer contract

Retain the caller-owned event-ID overload:

```java
publish(topic, key, payload, eventId, headers)
```

Enforce:

- concrete Protobuf value;
- descriptor full name as `event-type`;
- immutable caller event ID where an outbox owns identity;
- reserved canonical headers cannot be overridden;
- W3C propagation before fallback trace ID;
- `TopicNameStrategy`, `.v1` topics, `auto.register.schemas=false`;
- `acks=all`, idempotence, bounded acknowledgement timeout.

### A7. Remove undeployed legacy APIs

Remove default/public legacy behavior from the new stable line:

- `EventEnvelope`;
- JSON envelope serializer/deserializer;
- envelope `EventConsumer`/`RetryableConsumer` abstractions;
- legacy `x-event-*` emission.

No migration adapter is required because no Kafka deployment exists. Breaking API removal justifies `common-java v2.0.0`.

### A8. Real integration tests

Retain and complete the existing `kafkaContractIntegration` suite using real Kafka and Confluent Schema Registry containers. Prove:

- all nine canonical frames become expected generated classes;
- production auto-registration is disabled and Registry state does not mutate;
- exact 2/4/8 retries are unit-tested with an injected sleeper;
- commits occur only after success/confirmed DLQ;
- failed DLQ, cancellation, and same-group replacement leave unfinished work redeliverable;
- malformed and unknown-schema frames survive DLQ exactly;
- W3C context round-trips;
- no sensitive payload logging.

Java-to-Go and Go-to-Java artifact interoperability is an immutable RC matrix requirement in Phase 3, not a G2 substitute.

### A9. Java verification

```bash
cd /home/phucl/Workplace/gapi/common-java
./gradlew clean check jacocoTestReport jacocoTestCoverageVerification
./gradlew kafkaContractIntegration
```

CI and release use the same tasks.

---

## Lane B — `common-go`

### B1. Complete gRPC/auth core

Update existing auth, error, logging, recovery, and observability packages:

1. Add `MembershipNone`.
2. Ensure HTTP and gRPC parsers trim, normalize, and reject conflicts/unknowns identically.
3. Preserve explicit public methods and fail closed elsewhere.
4. Wire approved claim/correlation enrichment into unary and streaming logger contexts.
5. Make panic recovery return safe `INTERNAL` and `x-error-code=INTERNAL`.
6. Confirm interceptor order observes final status and trailer behavior.
7. Expand tests for empty/missing/duplicate metadata, all statuses, HTTP headers, public methods, streams, panic, W3C precedence, and races.

Critical files:

- `auth/claims.go`, `auth/incoming.go`, `auth/http.go`, `auth/metadata.go`
- `grpc/interceptor/chain.go`, `logging.go`, `recovery.go`, `error.go`
- `observability/propagation.go`
- `logging/fields.go`

### B2. Complete and verify the Kafka producer without waiting for Kong

Retain the existing `franz-go` implementation behind project-owned public interfaces. Keep client-specific types private and close only verified contract gaps.

Producer requirements:

- validate `.v1` topic and non-nil concrete Protobuf payload;
- derive `event-type` from descriptor;
- use caller-supplied event ID or injectable generator;
- use injectable clock and UTC epoch-millisecond timestamp;
- extract W3C context and fallback trace correlation;
- protect canonical headers;
- build Confluent framing with cached schema metadata;
- use `TopicNameStrategy`, `BACKWARD`, and no production auto-registration;
- configure all acknowledgements/idempotent production;
- honor cancellation and bounded publish timeout;
- return only after broker acknowledgement;
- distinguish schema incompatibility, Registry outage, serialization failure, and broker failure;
- never log payloads.

Suggested packages:

```text
kafka/
internal/kafka/
```

### B3. Complete and verify consumer, retry, and DLQ behavior

Retain the existing manual-commit delivery path and verify these requirements:

1. Retain raw key, frame, and headers.
2. Disable auto-commit.
3. Process each partition sequentially by default.
4. Decode into a caller-provided concrete target/descriptor.
5. Validate required canonical metadata.
6. Reconstruct trace context.
7. Retry in place after 2, 4, and 8 seconds.
8. Support permanent-error classification.
9. Commit only after success or confirmed DLQ publication.
10. Never commit later offsets beyond an incomplete record.
11. Preserve exact raw frame and headers to `{topic}.DLQ`.
12. Leave source uncommitted when DLQ publication fails.
13. Handle cancellation, shutdown, and rebalances without committing unfinished work.
14. Expose injectable clock/sleeper for deterministic tests.

Service handlers retain idempotency responsibility. DB/Kafka atomicity remains a service outbox concern.

### B4. Go tests and release workflow

Extend `Makefile`/CI with an enforced coverage threshold, contract integration job, external import smoke test, and tag-only release workflow.

Run:

```bash
cd /home/phucl/Workplace/gapi/common-go
make verify
go test -tags=integration -race ./...
```

Integration coverage:

- all nine released fixtures;
- cancellation during handler/backoff;
- restart/rebalance redelivery;
- schema cache behavior;
- no goroutine leaks;
- exact DLQ frame preservation;
- published `proto-go` resolution without `replace`.

Go-to-Java/Java-to-Go artifact transport and clean external imports of the common-library RCs remain Phase 3 work after RC publication.

Suggested milestones:

- `v0.1.0`: core gRPC/auth only, optional and clearly scoped;
- `v0.2.0`: producer milestone, not complete foundation;
- `v0.3.0-rc.1`: complete producer/consumer/retry/DLQ candidate;
- `v0.3.0`: stable after Phase 3.

Ordinary services import none of the incomplete milestones.

## Shared evidence produced

Assemble one G2 bundle for the exact Java and Go source SHAs. CI artifact uploads from unrelated runs are not a substitute.

- Java and Go auth/error/trace conformance reports.
- Live Kafka/Registry reports using the pinned fixture environment.
- Retry/commit/DLQ transcript.
- Raw source/DLQ frame and header checksums.
- Coverage, race, static-analysis, and vulnerability results.
- Published Java/Go proto dependency-resolution reports without `mavenLocal()` or `replace`.
- Exact dependency versions, source SHAs, commands, timestamps, and CI run URLs.

## G2 exit criteria

- Both libraries implement the same frozen auth/error/trace semantics.
- Java delivers deterministic generated Protobuf types.
- Go producer and consumer are complete.
- Both preserve malformed frames through DLQ exactly.
- Retry and commit ordering is proven against real infrastructure.
- Legacy envelope transport is absent from the new Java stable line.
- Release workflows are tag-only and reproducible.
- No Kafka work depends on Kong evidence.
