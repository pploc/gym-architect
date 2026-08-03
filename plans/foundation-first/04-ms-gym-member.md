# Phase 4 — Refactor `ms-gym-member`

## Objective

Fix independent correctness defects immediately, then use `ms-gym-member` as the controlled validator for the tagged `gym-proto` release and `common-java` RC. Reach the Member portion of G4 without introducing undeployed legacy Kafka migration complexity.

## Prerequisites

For urgent fixes:

- [Phase 0](00-contract-freeze.md) passed G0.

For transport refactor:

- [Phase 1](01-gym-proto.md) passed G1.
- [Phase 3](03-foundation-release.md) produced `common-java v2.0.0-rc.1` that passed G3.
- Read [roadmap adoption rules](README.md).

Rechecked baseline on 2026-08-03: `develop` at `342377d`, one local commit ahead of origin. The service still pins `common-java:1.0.6` and `gym-proto-java:1.0.6`.

## Preserve existing work

Before editing, inspect:

```bash
git -C /home/phucl/Workplace/gapi/ms-gym-member status --short --branch
git -C /home/phucl/Workplace/gapi/ms-gym-member diff -- build.gradle
```

Known user work at audit time:

- `build.gradle` changed `gym-proto-java` from `1.0.2` to `1.0.4`.
- `KAFKA_PROTOBUF_SCHEMA_REGISTRY_REFACTOR_PLAN.md` was untracked.

Reconcile the dependency line rather than overwriting the file. Do not delete or replace the untracked plan without explicit owner instruction.

## QR ownership boundary

`gym-proto v1.1.0` has already removed Member's `GetGymDailySecret` RPC and assigns QR device, root-key, payload issuance, validation, and rotation contracts to Check-in. Member remains authoritative only for gym locations and membership decisions through `GetGymLocation` and `ValidateMembership`.

The current Member source still contains the pre-v1.1 raw daily-secret implementation and compiles only because it pins `gym-proto-java:1.0.6`. Remove that implementation before or atomically with the v1.1 dependency pin. Phase 4 does not create `ms-gym-checkin`; that future service implements `checkin/v1.CheckInService` without moving key material back into Member.

## Explicit non-goals

- No adoption by other services.
- No implementation of the absent `ms-gym-checkin` repository.
- No Member storage, issuance, signed-payload validation, compatibility RPC, cache, or rotation for QR secrets; Member still validates membership status for Check-in.
- No Maven-local or untagged shared dependency.
- No JSON-topic, offset, dual-read, dual-write, or legacy-group migration.
- No replacement of the service-owned transactional outbox/idempotency persistence with shared transport code.
- No cross-service database foreign keys.

---

## Part A — Immediate independent correctness fixes

Implement these as separate, reviewable changes while shared foundations are being completed.

### A0. Remove obsolete Member QR ownership

This cleanup is required by the released G1 contract and may proceed before G3. Complete it before or in the same atomic change as the `gym-proto-java:1.1.0` pin; otherwise the removed generated symbols make Member fail compilation.

1. Delete the obsolete Member QR types:
   - `GymQRSecretEntity`;
   - `GymQRSecretJpaRepository`;
   - `GymQRSecretMapper`;
   - `GymDailySecretDto`;
   - `GymQRUseCase`;
   - `GymQRService`;
   - `GymQRRotationScheduler`.
2. Remove QR dependencies and imports from:
   - `GymLocationService`, including secret creation during gym creation;
   - `GymLocationGrpcDelegate`, including `getGymDailySecret`;
   - `MemberGrpcHandler`, including the removed RPC override;
   - `application.yml`, including `app.member.cron.qr-rotation`.
3. Remove obsolete QR service, scheduler, mapper, handler, and location-creation test setup. Update the location test so gym creation persists only the location.
4. Add `V6__drop_gym_qr_secrets.sql` as a forward-only Flyway migration. Do not edit deployed `V1__init_member_schema.sql`; no data copy, compatibility view, or fallback API is required for predeployment QR data.
5. Preserve and test the valid Check-in-to-Member boundary:
   - `GetGymLocation` supplies canonical location data during Check-in provisioning;
   - `ValidateMembership(member_id, gym_id)` supplies the membership decision after Check-in validates its local signed payload.
6. After cleanup, require no production or test references to `GetGymDailySecret`, `GymDailySecret`, `GymQRSecret`, `GymQRService`, `GymQRUseCase`, `GymQRRotationScheduler`, or `qr-rotation`, except an intentional historical migration comment if needed.

Use explicit given/when/then structure and names such as:

- `givenValidGymDetails_whenCreateGymLocation_thenCreatesOnlyLocation`;
- `givenCurrentFlywayMigrations_whenSchemaIsCreated_thenGymQrSecretsTableIsAbsent`;
- `givenAuthorizedCheckinWorkload_whenGetGymLocation_thenReturnsCanonicalLocation`;
- `givenAuthorizedCheckinWorkload_whenValidateMembership_thenReturnsMembershipDecision`.

### A1. Fix role-enforcement bypass

Current problem: `GrpcMethodRegistry` scans the registered `BindableService` (`MemberGrpcHandler`), but role annotations are placed on delegates. The interceptor does not discover them.

Actions:

1. Put explicit method policies on every actual `MemberGrpcHandler` override.
2. Keep delegate-level self/gym resource checks where they enforce object scope.
3. Add startup/in-process tests asserting every registered RPC is classified.
4. Test:
   - missing claims gives `UNAUTHENTICATED`;
   - wrong role gives `PERMISSION_DENIED`;
   - correct role reaches the delegate;
   - public-by-omission is impossible.
5. Replace any service pseudo-role with workload identity when the fixed common library is adopted.

Critical files:

- `src/main/java/com/gym/member/member/adapter/in/grpc/MemberGrpcHandler.java`
- `MemberGrpcDelegate.java`
- `SubscriptionGrpcDelegate.java`
- `GymLocationGrpcDelegate.java`
- gRPC integration tests

### A2. Close user/gym scope gaps

Audit all RPCs and enforce:

- customer ownership for self reads/updates;
- admin scope limited to authenticated gym;
- only `SUPER_ADMIN` may intentionally span gyms;
- blank gym filters do not become cross-gym access;
- conflicting request and claim gym IDs fail;
- internal methods require workload identity and explicit gym scope.

Add cross-user, cross-gym, blank-filter, and super-admin tests.

### A3. Fix poisoned idempotency claims

Current problem: `IdempotencyService` uses `REQUIRES_NEW`, committing the processed-event claim before the business transaction. A failed handler then becomes a permanent false duplicate.

Actions:

1. Execute processed-event claim and business mutation in the same transaction (`REQUIRED`/`MANDATORY`).
2. Use a conflict-safe insert such as PostgreSQL `INSERT ... ON CONFLICT DO NOTHING` under the unique key.
3. Roll back claim, domain mutation, and associated outbox rows together on failure.
4. Return duplicate only after a prior successful transaction committed.
5. Test failure then retry, successful duplicate, concurrent duplicate, and outbox rollback.

Critical files:

- `MemberEventProcessingService.java`
- `IdempotencyService.java`
- `ProcessedEventJpaRepository.java`
- idempotency integration tests

### A4. Remove local gRPC error conflict

`GrpcErrorHandler` catches domain errors before the shared interceptor and maps forbidden failures incorrectly.

1. Allow errors to escape delegates to the shared interceptor.
2. Remove `GrpcErrorHandler` once all call sites are migrated.
3. Keep request validation and resource policy in the service.
4. Verify `x-error-code`, safe redaction, `NOT_FOUND`, `PERMISSION_DENIED`, and validation status through the registered server.

Complete deletion alongside the fixed common-java RC if the current legacy library cannot safely map every path.

### A5. Remove safe duplicates only

Delete the five unused local membership event records after confirming no imports:

- `MembershipActivatedEvent.java`
- `MembershipPausedEvent.java`
- `MembershipResumedEvent.java`
- `MembershipExpiringSoonEvent.java`
- `MembershipExpiredEvent.java`

Production already constructs `com.gym.proto.events.v1` messages.

Remove redundant explicit scanning of `com.gym.common` only after a context test proves auto-configuration creates one bean set.

Keep:

- manual gRPC server lifecycle;
- transactional outbox entity/writer/relay/parser;
- processed-event persistence;
- `CommonMapperUtils`;
- service-specific access policy.

---

## Part B — Adopt tagged foundations

Begin only with exact G1 proto and G3 Java RC artifacts.

### B1. Dependencies

Begin only after A0 has removed all imports and overrides for Member QR symbols deleted from G1. Apply A0 and this pin atomically if intermediate commits must remain buildable.

Modify `build.gradle` carefully:

- pin `com.gym.proto:gym-proto-java:1.1.0` or the exact G1 version;
- pin `com.gym:common-java:2.0.0-rc.1`;
- add `https://packages.confluent.io/maven/` if dependencies are not mirrored;
- resolve without `mavenLocal()`;
- align gRPC dependency management with the common release;
- preserve unrelated user changes.

### B2. Configuration

Update `application.yml` and test configuration:

- remove envelope JSON serializer/deserializer classes;
- set Schema Registry URL;
- configure `TopicNameStrategy`;
- set `auto.register.schemas=false` for production;
- set publish timeout and `.DLQ` suffix;
- use shared retry property names and durations;
- require canonical `event-id`;
- configure only `.v1` topics and `ms-gym-member-v1`.

No legacy listener or migration feature flag is needed.

### B3. Typed inbound records

Refactor `EventConsumerAdapter.java` for:

- `UserRegisteredEvent` from `identity.user.registered.v1`;
- `UserSuspendedEvent` from `identity.user.suspended.v1`;
- `PaymentCompletedEvent` from `payment.completed.v1`.

For every record:

1. Receive the concrete generated payload plus preserved raw record metadata through the shared typed adapter.
2. Require canonical `event-id`; never use trace ID as business identity.
3. Read ordering key and canonical metadata.
4. Validate `event-type` equals the payload descriptor full name.
5. Invoke the transactional application method.
6. Acknowledge only after commit or confirmed duplicate.
7. Throw validation/business failures so shared retry/DLQ behavior runs.
8. Treat missing event ID/type as permanent v1 failures.

Remove all service references to `EventEnvelope`, envelope serializers/deserializers, Spring `__TypeId__`, and legacy `x-event-*` headers.

### B4. Correct outbox identity and payload metadata

Update:

- `OutboxEventWriter.java`
- `OutboxPayloadParser.java`
- `OutboxPublisherScheduler.java`

Required behavior:

1. Preserve the application-assigned outbox UUID.
2. Call `publish(topic, key, payload, eventId, headers)`.
3. Make `outbox_events.id == Kafka event-id` across retries/restarts.
4. Populate existing `payload_type` with `payload.getDescriptorForType().getFullName()`.
5. Retain simple `event_type` only where current scheduler/deduplication queries need it.
6. Register parser suppliers by descriptor name explicitly; descriptor names are not Java class names.
7. Keep Protobuf JSON as internal DB representation if useful; Kafka value is concrete Protobuf.
8. Treat local predeployment rows/data as disposable. Do not add legacy Kafka-envelope/topic parsers.

No migration is needed merely to populate the existing nullable `payload_type` column.

### B5. Implement internal membership lookup

Implement the G1 `GetMembershipStatusByUserId` RPC:

- lookup by external `user_id`;
- return canonical status, including `NONE`;
- authorize verified Identifier workload identity;
- expose no HTTP route;
- avoid user ID logging at INFO;
- test allowed/denied peers and unavailable persistence.

### B6. Local infrastructure

Update `docker-compose.yml` or Testcontainers setup:

- add pinned Schema Registry 7.7.1;
- add health checks;
- configure Kafka UI with Registry URL;
- use correct host/container URLs;
- provision only `.v1`/`.v1.DLQ` topics and v1 subjects;
- verify broker compatibility with the pinned fixture environment.

## Part C — Validation and initial deployment

### C1. Rewrite tests

Replace mocked-template/envelope tests with real Kafka/Registry coverage:

- all three inbound generated events;
- all five outbound generated membership events;
- exact canonical headers and subjects;
- `auto.register.schemas=false` behavior;
- outbox UUID equals `event-id`;
- duplicate delivery;
- rollback-safe idempotency;
- 2/4/8 retry behavior;
- source commit ordering;
- malformed/unknown-schema byte-preserving DLQ;
- no commit when DLQ publication fails;
- process restart/redelivery;
- internal membership RPC authorization;
- gym creation without Member-owned QR persistence;
- current Flyway schema contains no `gym_qr_secrets` table;
- generated Member service has no `GetGymDailySecret` RPC;
- authorized Check-in workload access to `GetGymLocation` and `ValidateMembership`.

Rewrite test seeders to publish concrete Protobuf with canonical headers. Remove JSON strings and `__TypeId__`.

### C2. CI

Update `.github/workflows/ci.yml` to run:

```bash
./gradlew clean check
./gradlew kafkaContractIntegration
```

Include coverage verification and live Kafka/Registry tests. A job running only `test` is insufficient.

### C3. Greenfield initial deployment

1. Start with clean development/test Kafka and database state.
2. Provision only `.v1` and `.v1.DLQ` topics.
3. Set subjects to `BACKWARD` and preregister G1 schemas.
4. Confirm runtime credentials cannot register schemas.
5. Deploy consumers and verify readiness.
6. Enable `.v1` producers.
7. Monitor lag, retries, errors, and DLQ.

Rollback stops affected producers/consumers while retaining topics, subjects, offsets, and pending outbox rows for diagnosis. Do not add legacy consumers, dual writes, offset resets, or JSON-topic fallback.

## Verification commands

```bash
cd /home/phucl/Workplace/gapi/ms-gym-member
./gradlew clean check
./gradlew kafkaContractIntegration
```

Also:

- run dependency resolution without `mavenLocal()` and verify exact RC/proto coordinates in the report;
- search production and test sources for `GetGymDailySecret|GymDailySecret|GymQRSecret|GymQRService|GymQRUseCase|GymQRRotationScheduler|qr-rotation` and require no stale matches;
- inspect the migrated schema and generated Member descriptor to prove the QR table and RPC are absent.

## Evidence produced

- gRPC policy coverage report.
- cross-user/cross-gym negative tests.
- transactional idempotency report.
- QR ownership cleanup report proving QR-free gym creation, absent `gym_qr_secrets`, and absent Member QR RPC/symbols.
- retained `GetGymLocation` and `ValidateMembership` Check-in boundary report.
- dependency resolution/version report.
- Member Kafka/Registry integration report.
- outbox/event-ID invariant report.
- raw-frame DLQ checksums.
- initial deployment/rollback test report.

## Exit criteria

- Independent auth, scope, idempotency, and error defects are fixed.
- Member stores or exposes no QR secret, daily token, QR scheduler, or QR RPC; the forward migration removes `gym_qr_secrets`.
- `GetGymLocation` and `ValidateMembership` remain protected and tested for Check-in integration.
- Member pins exact G1 proto and G3 Java RC artifacts.
- No envelope/JSON Kafka transport remains.
- All inbound/outbound events pass live transport tests.
- Internal membership lookup is implemented and protected.
- Greenfield deployment requires no migration compatibility.
- Member adoption report is sufficient for Phase 3 stable promotion.
- After `common-java v2.0.0` is published, Member repins to stable and passes again.
