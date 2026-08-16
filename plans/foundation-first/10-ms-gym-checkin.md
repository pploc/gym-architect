# Phase 10 — Implement `ms-gym-checkin`

> **Status:** Planned; implementation has not started. G9 is complete. G10 opens only Check-in and does not activate Payment, Workout, Trainer, Promotion, Notification, or Analytics.

## Objective

Reach G10 by freezing the Check-in contracts, releasing immutable dependencies, implementing the Go service against YugabyteDB and Vault Transit, publishing `checkin.recorded.v1` through a transactional outbox, adding the generated browser gateway/Kong surface, and proving the result from locked clean sources.

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
- `common-go`;
- `ms-gym-member`;
- `ms-gym-plans`;
- new sibling repository `ms-gym-checkin`;
- `gym-infra`.

`common-java` changes are in scope only if the released Check-in Protobuf/Event fixture requires cross-language contract validation. Identifier requires no Check-in runtime change.

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

The QR display is a simple iPad app used by a gym owner. It has no kiosk registration, installation secret, device credential, or independent device revocation lifecycle.

`GetDisplayQrPayload` uses the same stable JWT flow as other authenticated browser/mobile routes:

- the gym owner logs in through Identifier;
- Kong validates the JWT and forwards only verified `sub` and `role` through the generated gateway;
- the request supplies explicit `gym_id` resource context;
- G10 authorizes `SUPER_ADMIN` only because no owner/admin-to-gym assignment model exists;
- Check-in validates the requested active gym through its exact Plans workload RPC;
- credentials never appear in paths, query strings, logs, metrics, traces, errors, or evidence.

Remove and reserve obsolete `device_id` fields and remove `RegisterDevice` and `RevokeDevice` from the active G10 contract. Do not add an iPad/device table. The app stores login tokens through the platform's secure credential facility, not browser `localStorage`.

### Root-key protection and rotation

Use Vault Transit through the official Vault Go client:

- generate each 32-byte QR root key with `crypto/rand`;
- store only Vault ciphertext and its key reference in `checkin_db`;
- authenticate deployed Check-in through Kubernetes workload identity;
- normal rotation accepts the prior key for exactly 120 seconds;
- emergency rotation retires the prior key immediately;
- every display/scan request checks Yugabyte key status;
- decrypted key bytes may exist only in an in-process bounded cache until their acceptance deadline;
- no Redis key cache;
- Vault outage marks readiness false, blocks key mutations and cache misses, and never makes a retired or unknown key acceptable.

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
  -> Vault Transit
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

### Immutable `gym-proto` release

1. Generate Java, Go, grpc-gateway, fixtures, contract source, and canonical OpenAPI from one approved source SHA.
2. Publish matching tag-derived Java and Go artifacts.
3. Record release asset checksums and canonical OpenAPI checksum.
4. Resolve both artifacts from clean external consumers with no `mavenLocal()`, `replace`, sibling checkout, or mutable branch.

Do not hardcode a total browser-operation count in this plan. Derive it from the frozen active-operation manifest.

### `common-go`

1. Upgrade to the released Check-in-capable Protobuf artifact.
2. Add `checkin.recorded.v1` to the frozen topic/type pair map.
3. Add the concrete generated message to Registry resolution and fixture tests.
4. Prove lookup-only Schema Registry behavior, canonical headers, framing, acknowledgement, retry, relay restart, and raw DLQ preservation against real Kafka/Registry.
5. Publish an immutable `common-go` release and prove clean external resolution.

Reuse existing Kafka interfaces and adapters. Do not add a Check-in-specific Kafka wrapper.

### Member

1. Implement user-based membership validation using Member-owned data.
2. Return canonical member identity and live gym-specific status.
3. Update exact method policy and Check-in SAN allowlist.
4. Add Given/When/Then tests for missing user, no gym membership, every status, wrong SAN, gateway identity, forged end-user metadata, and dependency-free execution.
5. Publish or pin an image that resolves only released contract artifacts.

### Plans

1. Implement exact active-gym validation using existing location ownership.
2. Update exact method policy and Check-in SAN allowlist without changing Identifier's permission.
3. Add Given/When/Then tests for active, closed, missing, wrong SAN, gateway identity, and sibling workload denial.
4. Keep all filtering/query composition on Spring Data JPA Specifications.
5. Publish or pin an image that resolves only released contract artifacts.

### Stage 1 verification

```bash
cd /home/phucl/Workplace/gapi/common-go
make verify
go test -tags=integration -race ./...

cd /home/phucl/Workplace/gapi/ms-gym-member
./gradlew startEnv
./gradlew clean check
./gradlew stopEnv

cd /home/phucl/Workplace/gapi/ms-gym-plans
./gradlew startEnv
./gradlew clean check
./gradlew stopEnv
```

### Stage 1 exit

- Immutable contract and common-go artifacts resolve externally.
- Member and Plans implement only the approved workload boundaries.
- Positive and negative mTLS/SAN tests pass.
- Workload methods remain absent from HTTP, canonical OpenAPI, generated gateway routes, and Kong.
- Pinned Member and Plans images resolve the same released contract generation.

---

## Stage 2 — Create and implement `ms-gym-checkin`

Create the sibling Git repository only after Stage 1 artifacts are available.

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
│       ├── vault/
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
- official Vault Go client;
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
- root-key ciphertext and Vault key reference only;
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

- strict component count, bounds, canonical IDs, decimal integers without leading zeroes, and unpadded base64url;
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
7. Decrypt on cache miss through Vault and verify HMAC/time.
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
- config validation, readiness, Vault outage/cache miss, and bounded shutdown;
- real interceptor chain through `bufconn`;
- Yugabyte migrations/transactions/rollback/concurrency;
- Member and Plans mTLS clients;
- Vault Transit encrypt/decrypt/denial/outage;
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
- Vault protects all durable root-key material and fails closed.
- Record/outbox atomicity and Kafka wire contract pass against real dependencies.
- Released dependencies resolve without local replacements.
- CI and image build pass; repository tree is clean.

---

## Stage 3 — Infrastructure, generated gateway, OpenAPI, and Kong

### Platform dependencies

Use existing official/runtime facilities before adding code:

- YugabyteDB official image/chart and YSQL for `checkin_db`;
- Vault official image/chart with Transit and Kubernetes auth;
- Kafka and Schema Registry existing platform contracts;
- existing generic `gym-service` chart;
- existing generated Go grpc-gateway.

Provision least-privilege DB credentials, Check-in service account, mTLS identity, Vault role/policy/key reference, Kafka topic, Schema Registry subject, and migration job/path. Secret values are mounted or injected by the approved secret boundary, never committed.

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
| Check-in | Vault | Transit/auth API only |
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

1. Start pinned YugabyteDB, Vault, Kafka, Schema Registry, Kong, generated gateway, Member, Plans, and Check-in.
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
- Vault denied/unavailable with readiness and fail-closed behavior;
- any secret/PII/raw-payload match in committed evidence.

### Release lock

Pin:

- exact SHAs for `gym-proto`, `common-go`, Member, Plans, Check-in, and `gym-infra`;
- contract and common-go tags, artifact coordinates, and checksums;
- Member, Plans, Check-in, generated-gateway, Kong, YugabyteDB, Vault, Kafka, and Schema Registry image digests;
- canonical OpenAPI, route manifest/template, rendered config, migration, and fixture checksums;
- certificate public metadata;
- protected CI and release URLs.

The runner materializes detached sources into a temporary `0700` workspace, rejects dirty/mismatched inputs, uses digest-pinned images, and deletes sources, certificates, credentials, fixture data, and private diagnostics on every exit. Private package access uses protected credentials and BuildKit secrets, never build arguments or persisted Git credentials.

### Sanitized evidence

Evidence may contain exact commands/labels, timestamps, exit codes, SHAs, versions, checksums, image digests, safe aggregate results, certificate issuer/subject/SAN/fingerprint/validity, and CI/release URLs.

Evidence must reject private keys, JWTs, `Authorization` values, refresh tokens, Vault plaintext or ciphertext, DB credentials, PII/fixture IDs, raw QR/event payloads, request/response bodies, raw logs, stack traces, and internal transport exceptions.

### Stage 4 exit

- Locked local G10 passes from clean detached sources.
- Protected authenticated G10 CI passes.
- Positive/negative matrix and evidence sanitizer pass.
- Every artifact/image resolves by immutable version/digest.
- Every pinned product tree is clean.
- Technical status and accountable-owner acceptance status are recorded separately.

## Evidence produced

- Stage 0 contract, semantic-compatibility, generated-output, OpenAPI, route, and fixture reports;
- immutable `gym-proto` and `common-go` release matrices;
- Member/Plans workload mTLS allow/deny report;
- Check-in unit/race/static/coverage/vulnerability report;
- Yugabyte migration, constraint, concurrency, transaction, idempotency, and outbox report;
- Vault policy, encryption, rotation, denial, outage, and secret-leak report;
- Kafka/Registry frame, key, headers, acknowledgement, retry, restart, and DLQ report;
- Helm, NetworkPolicy, certificates, generated gateway, Kong, OpenAPI, and TypeScript report;
- locked local/protected-CI G10 report and sanitized final evidence manifest.

Every report records exact SHAs, versions, checksums, commands, results, and timestamps. Do not infer owner acceptance from technical success.

## G10 exit criteria

- All Stage 0 contracts are frozen and represented in immutable released artifacts.
- Member and Plans expose only the exact approved Check-in workload methods.
- `ms-gym-checkin` owns only QR-key, scan, record, and Check-in event state; it owns no display-device lifecycle.
- Stable JWT identity, live Member validation, active Plans gym validation, and exact workload SANs pass positive and negative checks.
- YugabyteDB isolation, no cross-service foreign keys, idempotency, record/outbox atomicity, and concurrency are proven.
- Vault Transit protects durable key material and rotation/outage behavior fails closed.
- `checkin.recorded.v1` matches the frozen Protobuf/Schema Registry contract and relay semantics.
- Generated gateway, Kong, OpenAPI, Helm, NetworkPolicy, and native-HTTP denial checks pass.
- Locked clean-source local run, protected CI, sanitized evidence, and clean pinned trees pass.

Do not mark G10 complete from a local fixture alone.

## Explicit non-goals

- No production Payment, Workout, Trainer, Promotion, Notification, or Analytics implementation.
- No Analytics or Notification Check-in consumer.
- No staff-to-gym assignment model.
- No Redis without measured need.
- No copied Plans location or Member membership authority.
- No handwritten REST DTOs, parallel gateway, public workload RPC, or native Check-in business HTTP.
- No generic repository, gRPC client, crypto, KMS, or event framework for one service.
- No location, display-device, or QR-key lifecycle Kafka topic.
- No rewrite of historical G0–G9 evidence.
