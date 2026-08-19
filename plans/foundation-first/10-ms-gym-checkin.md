# Phase 10 — Implement `ms-gym-checkin`

> **Status:** In progress. Stage 2 `ms-gym-checkin` implementation is underway. Release/integration, infrastructure/gateway/Kong, locked E2E, protected CI/evidence, and owner acceptance remain pending. G10 opens only Check-in and does not activate Payment, Workout, Trainer, Promotion, Notification, or Analytics.

## Objective

Complete G10 by finishing the in-progress Go service against YugabyteDB and AWS KMS, publishing `checkin.recorded.v1` through a transactional outbox, adding the generated browser gateway/Kong surface, and proving the result from locked clean sources.

Check-in owns encrypted and versioned QR root keys, signed display payloads, scan validation, check-in records, and the Check-in event. A simple iPad app displays QR codes after the gym owner logs in. Plans remains authoritative for gym locations. Member remains authoritative for member identity and live membership state.

## Prerequisites

- [Phase 9](09-kong-grpc-gateway-openapi.md) passed G9 with immutable artifacts, locked clean-source execution, protected CI, and sanitized evidence.
- Read the [roadmap rules](README.md), especially immutable evidence and DB-per-service ownership.
- Record exact `develop` SHAs, tags, dependency versions, and clean-tree status before each stage.
- Preserve all G0–G9 evidence unchanged. Add supersession notes when G10 replaces a previously deferred boundary.
- Use disposable pre-production data. Do not add backfills, dual writes, or compatibility forwarding without a real data-migration requirement.

## In-scope repositories

- `docs`;
- `gym-proto`;
- `common-java`;
- `common-go`;
- `ms-gym-member`;
- `ms-gym-plans`;
- sibling repository `ms-gym-checkin`;
- `gym-infra`.

`common-java` is mandatory in Stage 1. The frozen Check-in topic must decode in Java and Go, and release evidence requires the existing bidirectional Java/Go foundation matrix. Identifier requires no Check-in runtime change.

## Frozen product and trust decisions

### Member identity

Stable JWT `sub` is the authoritative user identity for customer calls. Client input never selects the member identity used for a scan.

Change the exact Check-in-only Member workload boundary to:

```text
ValidateMembership(user_id, gym_id)
  -> member_id, valid, status
```

Member resolves its unique user reference and returns the canonical opaque `member_id`. The method remains unannotated, accepts only the verified `ms-gym-checkin` workload identity, and receives no end-user headers.

Remove `member_id` from `ProcessScanRequest` and reserve its field number and name. Split customer self-history from `SUPER_ADMIN` member-history so a customer request never chooses another member identity. Store both opaque `user_id` and canonical `member_id` on each Check-in record; publish only canonical `member_id` as the Kafka key.

### Plans gym validation

Add an exact internal Plans method such as:

```text
ValidateCheckInGym(gym_id) -> gym_id, status
```

The final RPC name and fields are frozen in Stage 0. It succeeds only for an existing active gym, accepts only `ms-gym-checkin`, and has no HTTP annotation, Kong route, or OpenAPI operation. Do not broaden Identifier's `GetActiveGym` permission or reuse Identifier's workload identity.

Check-in calls Plans when a `SUPER_ADMIN` requests display payloads or administers a root key, not during each scan. The display path validates an active gym and atomically loads or creates its current root-key version. The signed payload then binds scans to that gym.

### Scan idempotency

`ProcessScanRequest` requires `idempotency_key`. Check-in scopes uniqueness to `(user_id, idempotency_key)`:

- replay of the same canonical request returns the original successful result;
- reuse with a different request fingerprint returns conflict;
- distinct keys may create distinct check-ins, including within one QR slot.

Do not add an unrequested time-window or slot-level duplicate rule.

### Logged-in iPad display

The QR display is a simple iPad app used by a gym owner. G10 delivers and verifies its backend contract with a fixture client; building or distributing the app itself is outside this backend phase. It has no kiosk registration, installation secret, device credential, or independent device revocation lifecycle.

`GetDisplayQrPayload` uses the same stable JWT flow as other authenticated browser/mobile routes:

- the gym owner logs in through Identifier;
- Kong validates the JWT and forwards only verified `sub` and `role` through the generated gateway;
- the request supplies explicit `gym_id` resource context;
- G10 authorizes `SUPER_ADMIN` only because no owner/admin-to-gym assignment model exists;
- Check-in validates the requested active gym through its exact Plans workload RPC;
- credentials never appear in paths, query strings, logs, metrics, traces, errors, or evidence.

Remove and reserve obsolete `device_id` fields and remove `RegisterDevice` and `RevokeDevice` from the active G10 contract. Do not add an iPad/device table. The app stores login tokens through the platform's secure credential facility, not browser `localStorage`.

### Root-key protection and rotation

Use the official AWS SDK for Go v2 KMS client:

- generate each 32-byte QR root key with `crypto/rand`;
- encrypt/decrypt the root key with AWS KMS; store base64 KMS ciphertext in `key_ciphertext` and the resolved CMK ARN in `key_reference` in `checkin_db`;
- production uses AWS SDK default credentials through EKS workload identity, with no static AWS credentials;
- IAM permits only `kms:Encrypt`, `kms:Decrypt`, and `kms:DescribeKey` on the CMK;
- `KMS_ENDPOINT_URL` is LocalStack-only and must not be set in production;
- normal rotation accepts the prior key for exactly 120 seconds;
- emergency rotation retires the prior key immediately;
- every display/scan request checks Yugabyte key status;
- decrypted key bytes may exist only in an in-process bounded cache until their acceptance deadline;
- no Redis key cache;
- KMS outage marks readiness false, blocks key mutations and cache misses, and never makes a retired or unknown key acceptable.

### Public topology

```text
Browser or display client
  -> Kong HTTPS exact route
  -> mTLS generated Go grpc-gateway :8443
  -> mTLS Check-in gRPC :50051

Check-in
  -> mTLS Member ValidateMembership
  -> mTLS Plans ValidateCheckInGym
  -> YugabyteDB checkin_db
  -> AWS KMS
  -> Kafka + Schema Registry through transactional outbox
```

Kong remains the only public endpoint. Check-in `8080` exposes health/readiness only. Use standard `net/http` for those endpoints; do not add Gin when there is no native HTTP business API.

## API and authorization target

Stage 0 freezes final names and paths. The target policy is:

| Capability | HTTP exposure | Authentication | Authorization |
|---|---|---|---|
| Process scan | Generated gateway | Stable JWT | `CUSTOMER`; identity from verified `sub` |
| Self history | Generated gateway | Stable JWT | `CUSTOMER`; identity from verified `sub` |
| Member history | Generated gateway | Stable JWT | `SUPER_ADMIN` |
| Daily gym count | Generated gateway | Stable JWT | `SUPER_ADMIN` |
| Display current/next QR | Generated gateway | Stable JWT | `SUPER_ADMIN`; explicit active `gym_id` |
| Rotate gym root key | Generated gateway | Stable JWT | `SUPER_ADMIN` |
| Validate membership | No HTTP | Workload mTLS | Check-in SAN only |
| Validate Check-in gym | No HTTP | Workload mTLS | Check-in SAN only |

No authoritative `ADMIN`-to-gym assignment exists. Do not grant gym-scoped `ADMIN` access from request context.

---

## Stage 0 — Freeze contracts and trust boundaries

### Check-in API

Revise `proto/checkin/v1/checkin.proto` as one coordinated semantic contract break:

1. Add inline `google.api.http` and OpenAPI annotations for approved public methods.
2. Remove and reserve `ProcessScanRequest.member_id` field number/name.
3. Add required, bounded `idempotency_key`.
4. Split self-history from administrator member-history.
5. Use typed Protobuf timestamps for response records and key activation times.
6. Define daily counts as UTC calendar dates with `[00:00:00Z, next 00:00:00Z)` bounds.
7. Define stable response/error semantics for invalid QR, invalid membership, idempotent replay, idempotency conflict, inactive gym, and dependency outages.
8. Remove `RegisterDevice` and `RevokeDevice`; remove and reserve obsolete display/record/event `device_id` fields.
9. Make display retrieval an authenticated `SUPER_ADMIN` operation with explicit `gym_id`.
10. Classify every generated RPC; unknown or unclassified methods fail closed.
11. Retire the deferred `google.api.Service` Check-in mapping so it cannot compete with inline active annotations.

Do not preserve the stale `Member.GetGymLocation` comment or client-authoritative member identity for source compatibility. Classify the change from actual source/JSON/domain semantics and release it under the required major contract generation.

### Member and Plans workload contracts

Revise Member `ValidateMembership` to accept `user_id` and `gym_id`, returning canonical `member_id`, `valid`, and status. Member implementation uses existing JPA repositories with Spring Data JPA `Specification` composition; do not add custom persistence queries.

Add the Check-in-only Plans gym-validation method. Plans remains location owner and returns only fields Check-in needs to validate an active display gym. Both workload methods:

- have Protovalidate constraints;
- have no `google.api.http` annotation;
- are absent from canonical OpenAPI and Kong;
- map exactly one caller SAN to one method;
- reject generated-gateway, Kong, Identifier, Member, Notification, and arbitrary workload identities as applicable.

### Kafka wire contract

Freeze:

| Topic | Key | Concrete value | Subject |
|---|---|---|---|
| `checkin.recorded.v1` | canonical `member_id` | `events.v1.CheckInRecordedEvent` | `checkin.recorded.v1-value` |

Use generated Protobuf, Confluent framing, `TopicNameStrategy`, canonical headers, `auto.register.schemas=false`, at-least-once relay semantics, and `checkin.recorded.v1.DLQ`. Do not add location, display-device, or root-key lifecycle topics.

Update the wire inventory, concrete fixture registration, clean Registry fixture, topic/type maps, route manifest, active-operation allowlist, generated-route assertions, OpenAPI generation, and semantic compatibility record together.

### Stage 0 verification

```bash
cd /home/phucl/Workplace/gapi/gym-proto
buf format -d --exit-code proto
buf lint proto
# Run repository-configured breaking check against the last stable tag.
python3 scripts/verify-http-config.py
scripts/verify-generated-routes.sh
make proto
./gradlew clean check
```

Also prove deterministic generation, exact public operation diff, absence of workload routes, clean Schema Registry fixture generation, and matching Java/Go outputs from one source SHA.

### Stage 0 exit

- Check-in, Member, Plans, route, event, auth, and HTTP contracts agree.
- Removed fields are reserved and semantic breaking changes are documented.
- Every RPC has one exposure and authorization classification.
- Exact SAN-to-method positive and denial matrices are frozen.
- Generated artifacts and canonical OpenAPI contain only approved public operations.
- Kafka topic, key, type, subject, framing, headers, retry, commit, and DLQ behavior are unambiguous.
- Accountable-owner acceptance status is recorded separately from technical checks.

---

## Stage 1 — Release dependencies and prepare Member/Plans

Stage 1 has four ordered gates. Do not update a downstream repository from a local sibling checkout or an unpublished artifact. Source commits and pushes do not authorize tags, packages, GitHub releases, release-environment changes, or container images; stop at each publication boundary for explicit approval.

### Gate 1 — Repair and release `gym-proto`

The Stage 0 source commit `80950af313306491029b8eff464698678fdda9ba` passed Actions run [`31930988141`](https://github.com/pploc/gym-proto/actions/runs/31930988141), but that run is not release evidence:

- `.github/workflows/publish-stubs.yml` wrote an empty `validation-report.json` because its report-producing `jq` command omitted `-n`;
- `develop` has no branch protection or ruleset, the `release` environment has no required reviewer or deployment-branch policy, and `proto-go/main` is unprotected;
- the run validated a branch commit and correctly skipped publication.

Do not describe this run as protected CI and do not tag `80950af`. Repair and release in this order:

1. Add `jq -n` to the validation-report producer and add a regression assertion that the report is non-empty JSON with the candidate source SHA, release versions, fixture checksum, and generated checksums.
2. Correct `contracts/v1/release-evidence.md` to distinguish the passed unprotected Actions run from protected release evidence. Existing G9 generated-gateway/Kong evidence satisfies the current Kong release gate; Check-in-specific runtime route, header, and error evidence remains a Stage 3 obligation.
3. Configure and verify the approved repository/tag rules and release-environment protections through GitHub, or leave release blocked and record the missing protection truthfully. Protection changes require explicit authorization.
4. Rerun the complete Stage 0 suite on the new exact source SHA. Download the Actions artifact and verify a non-empty report plus these candidate outputs: all eleven fixture cases, deterministic Java/Go/OpenAPI/Kong generation, exact 24 approved Buf diagnostics, and manifest-derived route counts.
5. Before publication, prove absence of source tag `v7.0.2`, Go tag `v1.7.1`, GitHub release `v7.0.2`, and Java package `com.gym.proto:gym-proto-java:7.0.2`. Stop for explicit tag/package/release authorization.
6. Publish Java `7.0.2` and Go `v1.7.1` only from the approved source SHA. The workflow creates the annotated Go tag locally, publishes Java, and only then pushes the Go tag; failure after Java publication is a partial publication: stop, preserve the immutable Java package, diagnose, and resume only a same-SHA recovery path. Never blindly retry, replace a remote tag, or publish different bytes under an existing version.
7. From clean consumers, prove the annotated source tag peels to the approved SHA; `github.com/pploc/proto-go@v1.7.1` resolves without `replace`; Java `7.0.2` resolves from GitHub Packages without `mavenLocal()`; representative Identity, Member, Plans, Check-in, and event types compile on Go 1.26 and Java 26; and release assets bind the source SHA, fixture checksum, canonical OpenAPI checksums, Kong bundle checksum, and generated-output checksums.

Do not hardcode a browser-operation total in release logic. Derive operation and route inventories from the frozen manifests. Schema ID `6` is fixture-local and must never appear as a runtime constant.

### Gate 2 — Release a coordinated `common-java` / `common-go` pair

`common-java` is required. Java and Go must each publish and consume every frozen frame, so releasing one stable library before testing the other candidate is insufficient.

#### Reconcile provenance and freeze versions

1. Record current immutable baselines: `common-go v0.5.0` and `common-java v2.1.0`.
2. Reconcile the existing `com.gym:common-java:2.1.1` package with source history and publication logs. It is used by Member and Plans but has no matching Git tag or GitHub release and still declares `gym-proto-java:4.1.0`; do not treat it as complete release evidence.
3. Freeze the exact next semantic versions only after deciding the disposition of `2.1.1` and recording source/package provenance. Do not guess versions in this plan.
4. Repair candidate/stable workflows only after those versions are frozen. Replace obsolete `common-java 2.0.0`, `common-go v0.3.0`, and Protobuf v1.1.0 assumptions with exact approved candidate refs, released `gym-proto v7.0.2`, `proto-go v1.7.1`, source SHAs, and fixture checksum `e79341b996d5052c0e0e4a0fe2621fd5edab6ad3b2d3a8304678664cbb46b4ab`.

#### Prepare `common-java`

1. Upgrade the public dependency to `com.gym.proto:gym-proto-java:7.0.2`; resolve it externally without `mavenLocal()`.
2. Add `checkin.recorded.v1` / `events.v1.CheckInRecordedEvent` to `KafkaContract.TOPIC_TYPES`. Change the ten-pair `Map.of(...)` to `Map.ofEntries(...)`; add no new registry abstraction.
3. Keep the existing fixture-driven concrete descriptor, Confluent-frame, canonical-header, lookup-only publication, retry, acknowledgement, redelivery, and DLQ tests. Rename generation-specific test wording such as `givenSeededV110Registry...` without weakening assertions.
4. Replace the hardcoded Registry subject total with checks that each fixture subject exists and reports `BACKWARD`. Imported `buf/validate/validate.proto` and `common/v1/common.proto` subjects make total Registry counts invalid.
5. Correct README artifact/provenance and fixture-generation text.

#### Prepare `common-go`

1. Upgrade `github.com/pploc/proto-go` to released `v1.7.1` with `GOWORK=off`; regenerate `go.sum` through Go tooling.
2. Add the Check-in topic/type pair to `kafka/schema.go` and add both missing concrete Registry messages: `EmailVerificationRequestedEvent` and `CheckInRecordedEvent`.
3. Remove the whole-test pre-v4 fixture skip in `proto_fixture_test.go`. Every one of the eleven committed cases must construct its concrete message and verify payload/frame bytes.
4. Replace hardcoded nine-case assertions and workflow metadata with inventory derived from the committed fixture/topic contract.
5. Preserve `auto.register.schemas=false`, `UseLatestVersion`, `TopicNameStrategy`, Registry before/after immutability checks, canonical headers, retry/commit behavior, and raw DLQ preservation. Reuse `Event`, `FrameEncoder`, `ConfluentProtobufRegistry`, `FranzProducer`, and `FranzConsumer`; add no Check-in-specific wrapper.
6. Correct README release and fixture-generation text.

#### Candidate matrix and publication

1. Run local unit and real Kafka/Schema Registry integration suites in both repositories from clean trees.
2. Stop for explicit authorization before creating candidate refs or publishing candidate packages.
3. Create immutable Java and Go candidate refs bound to exact source SHAs and publish the authorized candidate packages. Resolve those immutable candidates from clean consumers, then run the existing foundation matrix in both directions across all eleven fixtures: Java candidate publishes and the externally resolved Go candidate consumes every frame; Go candidate publishes and the externally resolved Java candidate consumes every frame. Also prove Registry immutability, canonical headers, acknowledgement, retry/commit, redelivery, and DLQ behavior.
4. Record matrix evidence against both candidate refs, package coordinates, resolved checksums, and source SHAs. Any failed, skipped, truncated, source-checkout-only, or single-direction matrix blocks both stable releases.
5. Stop again for explicit stable tag/package/GitHub-release authorization. Publish both stable libraries from the exact proven candidate source SHAs, resolve them from clean external consumers with no sibling checkout, mutable ref, `replace`, or `mavenLocal()`, and rerun the bidirectional all-fixture matrix against the externally resolved stable packages before Member and Plans consume them.

### Gate 3 — Prepare Member

1. Upgrade to released `gym-proto-java:7.0.2` and the coordinated released `common-java` version. Remove `mavenLocal()` and prove clean cache-independent dependency resolution.
2. Keep the gRPC handler/delegate thin. Resolve `request.user_id` to the canonical Member-owned row with `MemberSpecifications.hasUserId`, then compose `SubscriptionSpecifications.hasMemberId(...).and(hasGymId(...))`. Add no repository method and no custom `@Query`.
3. Evaluate all gym subscriptions against an injected UTC `Clock`. Return canonical `member_id` and one effective result:
   - missing member: gRPC `NOT_FOUND`; Check-in maps it to its frozen invalid-membership response and writes no Check-in/outbox state;
   - no subscription for the gym: `valid=false`, `NONE`;
   - effective `ACTIVE`: `valid=true`, `ACTIVE`;
   - `PAUSED`: `valid=false`, `PAUSED`;
   - `EXPIRED`: `valid=false`, `EXPIRED`.
4. For historical rows, evaluate dates before applying priority `ACTIVE > PAUSED > EXPIRED > NONE`. A persisted `ACTIVE` row with `end_date < today` is effectively expired; `end_date == today` and a lifetime row with null `end_date` remain active. Do not depend on the expiry scheduler having already rewritten stale rows.
5. Preserve `@RequirePolicy(INTERNAL_WORKLOAD)` and the exact `ValidateMembership -> ms-gym-checkin` DNS/SPIFFE allowlist. Preserve Notification access only to its separate method.
6. Add Given/When/Then tests for canonical user/member resolution, missing member, no gym row, all statuses, multiple historical rows, date boundary, stale active row, and canonical response ID. Add real TLS-handshake integration proof for allowed Check-in DNS/SPIFFE identities and denial of generated gateway, Kong, Identifier, Plans, Member, Notification on the wrong method, arbitrary CA-valid clients, missing certificates, plaintext, swapped methods, and forged end-user metadata.
7. Prove `ValidateMembership` remains absent from HTTP annotations, canonical OpenAPI, generated routes, and Kong. Keep `./gradlew startEnv` / `stopEnv` documented and passing.

### Gate 4 — Prepare Plans

1. Upgrade to released `gym-proto-java:7.0.2` and the coordinated released `common-java` version. Remove `mavenLocal()` and prove clean cache-independent dependency resolution.
2. Add one thin `@RequirePolicy(INTERNAL_WORKLOAD)` `validateCheckInGym` handler. Reuse `GymLocationService.getActive`; return its canonical persisted gym ID and active status. Add no service, DTO, mapper abstraction, repository query, Specification, or migration.
3. Add only `ValidateCheckInGym -> ms-gym-checkin` to the exact method allowlist. Preserve `GetActiveGym -> ms-gym-identifier` and `ResolvePurchasablePlan -> ms-gym-member` unchanged.
4. Add Given/When/Then tests for active canonical response, closed gym, missing gym, all approved Check-in DNS/SPIFFE forms, and denial of generated gateway, Kong, Identifier, Member, Notification, arbitrary CA-valid clients, missing certificates, plaintext, and swapped methods. Add a real TLS-handshake workload integration test; mocked `SSLSession` unit checks alone are insufficient.
5. Prove `ValidateCheckInGym` remains absent from HTTP annotations, canonical OpenAPI, generated routes, and Kong. Document the workload method and keep `./gradlew startEnv` / `stopEnv` passing.

Member and Plans `develop` pushes currently invoke image publication. Before pushing runtime changes, stop for explicit image-publication authorization. When authorized, publish only source-SHA tags, record registry digests, and prove the images resolve the released `gym-proto` and coordinated `common-java` artifacts. Mutable branch or `latest` tags are not evidence. Package credentials remain BuildKit secrets and never become build arguments.

### Stage 1 verification

```bash
cd /home/phucl/Workplace/gapi/common-java
./gradlew clean check --no-daemon
# Run the repository Kafka/Schema Registry integration workflow against the frozen candidate pair.

cd /home/phucl/Workplace/gapi/common-go
GOWORK=off make verify
GOWORK=off go test -tags=integration -race ./...
# Run the bidirectional all-fixture foundation matrix against common-java.

cd /home/phucl/Workplace/gapi/ms-gym-member
./gradlew startEnv
./gradlew clean check --no-daemon
member_rc=$?
./gradlew stopEnv
exit "$member_rc"

cd /home/phucl/Workplace/gapi/ms-gym-plans
./gradlew startEnv
./gradlew clean check --no-daemon
plans_rc=$?
./gradlew stopEnv
exit "$plans_rc"
```

Run the two service command blocks independently so the first `exit` does not skip Plans. Also run clean dependency-resolution checks, real mTLS integration suites, manifest-derived route/OpenAPI absence checks, and external package/image resolution from disposable consumers.

### Stage 1 exit

- `gym-proto v7.0.2`, `gym-proto-java:7.0.2`, and `proto-go v1.7.1` resolve externally and bind one approved, actually protected source SHA with non-empty release evidence.
- Repository/tag/release-environment protection is verified through authoritative configuration; any absent protection remains a release blocker and is not relabeled as protected CI.
- Exact `common-java` and `common-go` versions have reconciled provenance, resolve externally, and bind the same eleven-case fixture/checksum and released Protobuf generation.
- The bidirectional Java-to-Go and Go-to-Java all-fixture matrix passes without skipped cases or Schema Registry mutation.
- Member and Plans implement only the approved workload boundaries, resolve dependencies without `mavenLocal()`, and pass effective-state/domain tests plus real mTLS allow/deny tests.
- Both workload methods remain absent from HTTP, canonical OpenAPI, generated gateway routes, and Kong.
- Explicit image-publication authorization is recorded before Member/Plans publication; authorized source-SHA image tags and immutable registry digests resolve the approved dependency versions.
- Technical evidence and accountable-owner acceptance are recorded separately. No package, release, tag, protection change, or image is inferred from a source commit/push.

---

## Stage 2 — Implement `ms-gym-checkin`

The sibling Git repository exists and Stage 2 implementation is underway. Finish only the scoped Check-in service work below.

### Minimum repository structure

```text
ms-gym-checkin/
├── cmd/server/
├── internal/
│   ├── config/
│   ├── domain/
│   ├── usecase/
│   │   └── port/
│   └── adapter/
│       ├── grpc/
│       ├── yugabyte/
│       ├── member/
│       ├── plans/
│       ├── kms/
│       └── kafka/
├── migrations/
├── test/integration/
├── .github/workflows/
├── Dockerfile
├── docker-compose.yml
├── Makefile
├── README.md
├── go.mod
└── go.sum
```

Create only ports required by use cases. Do not add generic repositories, factories, a shared crypto package, or an abstraction for one implementation.

### Composition and runtime

Follow `ms-gym-identifier` for composition-root placement and `common-go` for platform behavior:

- typed fail-fast configuration;
- canonical interceptor chain and Protovalidate;
- exact method registry completeness check;
- categorized safe errors and `x-error-code` trailers;
- W3C tracing, bounded-cardinality metrics, and secret-safe logging;
- concrete mTLS Member and Plans clients with deadlines;
- official AWS SDK for Go v2 KMS client;
- `database/sql` through Yugabyte YSQL/PostgreSQL wire protocol;
- explicit migrations, never hidden pod-startup migrations;
- bounded readiness checks, HTTP shutdown, gRPC drain with force-stop fallback, background relay cancellation, and normal returns so deferred cleanup runs.

Check-in `8080` serves `/healthz` and `/readyz` only. Do not expose native business HTTP or placeholder metrics presented as production telemetry.

### Persistence

Use append-only migrations for:

```text
gym_qr_root_keys
check_ins
outbox_events
```

Required invariants include:

- `(gym_id, key_version)` primary key;
- one active/current root-key state per gym as frozen by Stage 0;
- concurrency-safe first-display root-key creation for an active Plans gym;
- base64 KMS ciphertext in `key_ciphertext` and resolved CMK ARN in `key_reference` only;
- opaque `user_id`, `member_id`, and `gym_id` with no cross-service foreign keys;
- unique `(user_id, idempotency_key)`;
- canonical request fingerprint stored with the idempotency result;
- check-in record and outbox insert in one transaction;
- outbox claim with safe concurrent relay behavior and retry state.

Use database constraints for final invariants. Test concurrent identical requests and conflicting idempotency-key reuse against YugabyteDB.

### QR and display behavior

Implement exactly:

```text
v1.<gym_id>.<key_version>.<utc_slot>.<mac_base64url>

utc_slot = floor(unix_seconds / 60)
message  = "checkin-qr:v1|<gym_id>|<key_version>|<utc_slot>"
mac      = HMAC-SHA256(root_key, message)
```

Requirements:

- strict component count, bounds, decimal integers without leading zeroes, and unpadded base64url;
- `gym_id` is the exact canonical lowercase UUID string emitted by Plans (`UUID.toString()` form); reject noncanonical, uppercase, encoded, or dot-containing forms before signing or verification so the five-component grammar is unambiguous;
- constant-time MAC comparison;
- current and immediately previous slot only;
- current and next payload returned to a logged-in `SUPER_ADMIN` iPad app for an explicit active gym;
- `crypto/rand` for root keys;
- every display/scan verifies gym, key version, status, and retirement deadline from Yugabyte;
- normal prior key accepted no longer than 120 seconds; emergency prior key rejected immediately;
- zero key bytes after use where practical and bound any in-process cache by acceptance deadline;
- no root key, JWT, raw QR, or request body in logs, metrics, traces, errors, events, or evidence.

### Scan workflow

1. Derive `user_id` from validated gateway claims.
2. Validate request and compute a canonical request fingerprint.
3. Resolve existing idempotency result or reject conflicting key reuse.
4. Strictly parse QR and derive signed gym/key/slot.
5. Validate request gym consistency if Stage 0 retains explicit gym input.
6. Load acceptable key status.
7. Decrypt on cache miss through AWS KMS and verify HMAC/time.
8. Call Member `ValidateMembership(user_id, signed_gym_id)` over Check-in mTLS, forwarding no user headers.
9. Require `valid` and active status; use returned canonical `member_id`.
10. Insert Check-in, idempotency result, and outbox event atomically.
11. Return the stored result on exact replay.

Plans is not called during scan. It is called when a `SUPER_ADMIN` requests display payloads or administers gym root keys.

### Tests and quality gates

Use `given_when_then` test names and lightweight fakes. Cover:

- QR parsing, canonicalization, HMAC, slot boundaries, future/expired values, and constant-time verification path;
- normal/emergency rotation and retired-key behavior;
- logged-in display authorization, active-gym validation, concurrency-safe first-display key creation, and denial for non-`SUPER_ADMIN` roles;
- user-derived identity, role rules, exact method registry, and all unauthorized paths;
- idempotent replay, changed fingerprint conflict, distinct-key behavior, and concurrency;
- config validation, readiness, KMS outage/cache miss, and bounded shutdown;
- real interceptor chain through `bufconn`;
- Yugabyte migrations/transactions/rollback/concurrency;
- Member and Plans mTLS clients;
- AWS KMS encrypt/decrypt/denial/outage;
- Kafka/Registry outbox publish, restart, retry, acknowledgement, and DLQ;
- secret-leak scanning of API output, logs, metrics, and evidence;
- reproducible scan latency in the locked fixture, targeting p95 below 100 ms without claiming a production SLO.

Run:

```bash
cd /home/phucl/Workplace/gapi/ms-gym-checkin
gofmt -d .
go vet ./...
go test -race ./...
go test -tags=integration -race ./test/integration/...
```

CI also runs configured lint, coverage, static analysis, `govulncheck`, clean dependency verification, and a non-root secret-safe image build using BuildKit secrets for private modules.

### Stage 2 exit

- Every business branch and trust boundary has runnable positive and negative checks.
- Yugabyte constraints and transactions enforce local invariants.
- AWS KMS protects all durable root-key material and fails closed.
- Record/outbox atomicity and Kafka wire contract pass against real dependencies.
- Released dependencies resolve without local replacements.
- CI and image build pass; repository tree is clean.

---

## Stage 3 — Infrastructure, generated gateway, OpenAPI, and Kong

### Platform dependencies

Use existing official/runtime facilities before adding code:

- YugabyteDB official image/chart and YSQL for `checkin_db`;
- AWS KMS with EKS workload identity; LocalStack endpoint override only for local tests;
- Kafka and Schema Registry existing platform contracts;
- existing generic `gym-service` chart;
- existing generated Go grpc-gateway.

Provision least-privilege DB credentials, Check-in service account, mTLS identity, KMS CMK/key reference and IAM workload policy, Kafka topic, Schema Registry subject, and migration job/path. Secret values are mounted or injected by the approved secret boundary, never committed.

### Network and method policy

Add separate port/peer rules:

| Caller | Destination | Port/method |
|---|---|---|
| Kong | Generated gateway | `8443`, exact Check-in HTTP routes |
| Generated gateway | Check-in | `50051`, declared public methods only |
| Check-in | Member | `50051`, `ValidateMembership` only |
| Check-in | Plans | `50051`, Check-in gym validation only |
| Check-in | YugabyteDB | YSQL only |
| Check-in | Kafka / Schema Registry | required producer/lookup ports only |
| Check-in | AWS KMS | KMS API only |
| Health observers | Check-in | `8080`/configured metrics only |

Deny Kong direct Check-in gRPC, gateway workload methods, sibling reuse of Check-in permissions, broad namespace ingress, and unrelated egress. NetworkPolicy does not replace server-side SAN/method authorization.

### Generated gateway and Kong

1. Add Check-in backend config and generated handler registration to the existing gateway.
2. Continue rejecting arbitrary inbound headers.
3. Forward verified user/role/tracing metadata on JWT routes.
4. Forward verified user/role/tracing metadata to the display RPC exactly as for other JWT-authenticated Check-in routes.
5. Configure exact Kong method/path routes and apply JWT to scan, history, administration, and display routes; all routes remain TLS/mTLS protected.
6. Keep safe deterministic `500` and `503` bodies and avoid credential reflection.
7. Generate Check-in OpenAPI from Protobuf, merge deterministically, reject collisions, and derive expected operation/route counts from the active manifest.
8. Keep all workload methods and Check-in `8080` business paths absent.

### Stage 3 verification

```bash
cd /home/phucl/Workplace/gapi/gym-infra
helm lint helm/gym-service
# Run repository Helm render and NetworkPolicy tests.
# Run generated gateway and Kong route tests.
```

Also run TypeScript generation from the released canonical OpenAPI with `tsc --noEmit`, route ownership negatives, forged/duplicate metadata tests, display-role authorization tests, certificate SAN checks, deterministic render/checksum checks, and direct native-HTTP `404` checks.

### Stage 3 exit

- Check-in deployment, probes, resources, service account, secrets, ports, and migration path render correctly.
- Dependency bootstrap uses supported official images/charts and least privilege.
- NetworkPolicy and server method policy pass full allow/deny matrix.
- Generated gateway reaches only public Check-in methods.
- Kong route/plugin behavior matches the frozen auth matrix.
- Canonical OpenAPI and generated client compile; workload methods remain absent.
- Rendered configuration checksum and immutable image digests are recorded.

---

## Stage 4 — Locked G10 end-to-end proof and closeout

Add G10-specific files rather than modifying G9 lock/evidence:

```text
gym-infra/kong/g10-release-lock.json
gym-infra/kong/materialize-g10.py
gym-infra/kong/g10-compose.yml
gym-infra/kong/g10-business-check.sh
gym-infra/kong/run-g10.sh
gym-infra/.github/workflows/g10-checkin.yml
docs/evidence/foundation-first/g10-final/
```

Final names may follow repository conventions, but G10 must have an independent lock, runner, sanitizer, and evidence bundle.

### Locked positive flow

From clean detached sources with no readable sibling checkout:

1. Start pinned YugabyteDB, local KMS emulator only where required, Kafka, Schema Registry, Kong, generated gateway, Member, Plans, and Check-in.
2. Create/select an active gym through the approved API fixture.
3. Log in as `SUPER_ADMIN` from the iPad-app fixture and obtain current/next signed payloads for the active gym.
4. Prove non-`SUPER_ADMIN` users cannot obtain display payloads.
5. Scan with a stable customer JWT and required idempotency key.
6. Prove Check-in resolved canonical member identity/live membership through the exact Member workload method.
7. Replay the same request and receive the original result without another record/event.
8. Reuse the key with changed input and receive conflict; use a distinct key and receive a distinct allowed record.
9. Read customer self-history and `SUPER_ADMIN` member history/daily count.
10. Prove atomic Check-in/outbox state, framed event type, canonical key, required headers, and pre-registered subject.
11. Prove 120-second normal overlap and immediate emergency retirement.

### Locked negative flow

Prove failure for:

- malformed, noncanonical, tampered, expired, or future QR;
- signed/request gym mismatch;
- unknown or retired key version;
- missing JWT, non-`SUPER_ADMIN` display request, or forged role;
- missing, malformed, or closed gym during display/key administration;
- missing member, inactive membership, or client identity conflict;
- changed request under an existing idempotency key;
- wrong SAN, sibling workload, gateway call to workload RPC, Kong direct service attempt, forged trusted headers, or duplicate trusted metadata;
- missing/mismatched Registry subject while auto-registration remains disabled;
- Yugabyte transaction rollback and relay restart;
- AWS KMS denied/unavailable with readiness and fail-closed behavior;
- any secret/PII/raw-payload match in committed evidence.

### Release lock

Pin:

- exact SHAs for `gym-proto`, `common-java`, `common-go`, Member, Plans, Check-in, and `gym-infra`;
- contract, common-java, and common-go tags, artifact coordinates, provenance, bidirectional-matrix evidence, and checksums;
- Member, Plans, Check-in, generated-gateway, Kong, YugabyteDB, Kafka, and Schema Registry image digests, plus KMS configuration identity;
- canonical OpenAPI, route manifest/template, rendered config, migration, and fixture checksums;
- certificate public metadata;
- protected CI and release URLs.

The runner materializes detached sources into a temporary `0700` workspace, rejects dirty/mismatched inputs, uses digest-pinned images, and deletes sources, certificates, credentials, fixture data, and private diagnostics on every exit. Private package access uses protected credentials and BuildKit secrets, never build arguments or persisted Git credentials.

### Sanitized evidence

Evidence may contain exact commands/labels, timestamps, exit codes, SHAs, versions, checksums, image digests, safe aggregate results, certificate issuer/subject/SAN/fingerprint/validity, and CI/release URLs.

Evidence must reject private keys, JWTs, `Authorization` values, refresh tokens, KMS plaintext or ciphertext, DB credentials, PII/fixture IDs, raw QR/event payloads, request/response bodies, raw logs, stack traces, and internal transport exceptions.

### Stage 4 exit

- Locked local G10 passes from clean detached sources.
- Protected authenticated G10 CI passes.
- Positive/negative matrix and evidence sanitizer pass.
- Every artifact/image resolves by immutable version/digest.
- Every pinned product tree is clean.
- Technical status and accountable-owner acceptance status are recorded separately.

## Evidence produced

- Stage 0 contract, semantic-compatibility, generated-output, OpenAPI, route, and fixture reports;
- immutable `gym-proto` release evidence and coordinated `common-java`/`common-go` provenance, external-resolution, and bidirectional all-fixture matrix reports;
- Member/Plans clean dependency and workload mTLS allow/deny reports, including source-SHA image digests when image publication is authorized;
- Check-in unit/race/static/coverage/vulnerability report;
- Yugabyte migration, constraint, concurrency, transaction, idempotency, and outbox report;
- AWS KMS IAM, encryption, rotation, denial, outage, and secret-leak report;
- Kafka/Registry frame, key, headers, acknowledgement, retry, restart, and DLQ report;
- Helm, NetworkPolicy, certificates, generated gateway, Kong, OpenAPI, and TypeScript report;
- locked local/protected-CI G10 report and sanitized final evidence manifest.

Every report records exact SHAs, versions, checksums, commands, results, and timestamps. Do not infer owner acceptance from technical success.

## G10 exit criteria

- All Stage 0 contracts are frozen and represented in immutable released Java and Go artifacts from one approved, protected source SHA.
- Coordinated `common-java` and `common-go` releases have reconciled provenance, resolve externally, and pass the bidirectional all-fixture matrix without Registry mutation or skipped cases.
- Member and Plans expose only the exact approved Check-in workload methods and resolve the coordinated released dependencies without local repositories or replacements.
- Required repository/tag/release-environment protections are verified; missing protection blocks completion and is never reported as protected CI.
- Member and Plans image publication has explicit authorization, and their source-SHA tags and immutable registry digests are recorded.
- `ms-gym-checkin` owns only QR-key, scan, record, and Check-in event state; it owns no display-device lifecycle.
- Stable JWT identity, live Member validation, active Plans gym validation, and exact workload SANs pass positive and negative checks.
- YugabyteDB isolation, no cross-service foreign keys, idempotency, record/outbox atomicity, and concurrency are proven.
- AWS KMS protects durable key material and rotation/outage behavior fails closed.
- `checkin.recorded.v1` matches the frozen Protobuf/Schema Registry contract and relay semantics.
- Generated gateway, Kong, OpenAPI, Helm, NetworkPolicy, and native-HTTP denial checks pass.
- Locked clean-source local run, protected CI, sanitized evidence, and clean pinned trees pass.

Do not mark G10 complete from a local fixture alone.

## Explicit non-goals

- No production Payment, Workout, Trainer, Promotion, Notification, or Analytics implementation.
- No iPad/mobile app implementation or distribution; G10 proves the display API with a fixture client.
- No Analytics or Notification Check-in consumer.
- No staff-to-gym assignment model.
- No Redis without measured need.
- No copied Plans location or Member membership authority.
- No handwritten REST DTOs, parallel gateway, public workload RPC, or native Check-in business HTTP.
- No generic repository, gRPC client, crypto, KMS, or event framework for one service.
- No location, display-device, or QR-key lifecycle Kafka topic.
- No rewrite of historical G0–G9 evidence.
