# Phase 1 — Harden and Release `gym-proto`

## Objective

Reach G1 by turning `gym-proto` into an enforced, immutable contract source and publishing matching Java/Go artifacts.

## Prerequisites

- [Phase 0](00-contract-freeze.md) passed G0.
- Read [shared roadmap rules](README.md).
- Recheck `gym-proto` branch, SHA, tags, and working tree.
- Audited baseline was `develop` at `0b44daca`, tagged `v1.0.6`.

## In-scope repositories

- `gym-proto`
- generated `proto-go` publication target
- platform docs only where generated/API changes require synchronized updates

## Explicit non-goals

- No `common-java`, `common-go`, Member, Kong, or Identifier implementation.
- No owner approval invented by an implementation agent.
- No backward support for undeployed JSON Kafka topics.

## 1. Add the internal membership RPC

Modify `proto/member/v1/member.proto`:

1. Add `GetMembershipStatusByUserId` and its request.
2. Keep `GetMembershipStatus(member_id)` unchanged.
3. Add comments declaring the new method internal and workload-authenticated.
4. Do not assign an HTTP mapping.
5. Use new field numbers; never reuse or rename existing fields.

Regenerate Java, Go, gRPC, and gateway outputs. Verify the new RPC appears in gRPC stubs but not in gateway routes.

## 2. Wire HTTP mappings into generation

Current `*_http.yaml` files are not referenced by `buf.gen.yaml`.

1. Choose the Phase 0 mapping source of truth.
2. Configure grpc-gateway generation with `grpc_api_configuration` or the equivalent supported option.
3. Retain only intentional external methods.
4. Remove `generate_unbound_methods=true` if it would expose internal RPCs.
5. Add a generated-route assertion for Identifier and Member paths.
6. Verify Kong’s future route matrix can target HTTP gateway port `8080`.

Critical files:

- `buf.gen.yaml`
- `proto/identity/v1/identity_http.yaml`
- `proto/member/v1/member_http.yaml`
- other `*_http.yaml` files

## 3. Update canonical contract artifacts

Update `contracts/v1` with the frozen:

- roles and membership statuses;
- trusted headers and tracing precedence;
- `.v1` topics and `<topic>-value` subjects;
- retry/DLQ behavior;
- internal workload identity rule;
- semantic compatibility rules;
- release/evidence structure.

Correct stale component versions in `contracts/v1/manifest.json`, but do not change pending owners to approved. Separate technical evidence status from human approval status.

## 4. Expand Confluent fixtures

Extend `contracts/v1/kafka/confluent-7.7.1-fixtures.json` to nine cases:

1. `UserRegisteredEvent` using `CUSTOMER`.
2. `UserSuspendedEvent`.
3. `UserRoleChangedEvent`.
4. `PaymentCompletedEvent`.
5. `MembershipActivatedEvent`.
6. `MembershipPausedEvent`.
7. `MembershipResumedEvent`.
8. `MembershipExpiringSoonEvent`.
9. `MembershipExpiredEvent`.

Each fixture includes:

- topic and subject;
- ordering key;
- descriptor full name;
- canonical headers;
- deterministic logical fields;
- schema ID/version from a clean disposable Registry;
- payload bytes;
- Protobuf message-index bytes;
- complete Confluent frame.

Update:

- `ConfluentFixtureGenerator.java`
- `ConfluentFixtureVerifier.java`
- `ConfluentCompatibilityVerifier.java`
- fixture support utilities and README

The generator may register against a disposable Registry. The verifier is read-only and deterministic.

## 5. Enforce compatibility and generation

Update `build.gradle` so `check` runs offline fixture verification, not only fixture compilation.

Update `.github/workflows/publish-stubs.yml` so PRs and tags run:

```bash
buf format -d --exit-code proto
buf lint proto
buf breaking proto --against '.git#tag=v1.0.6,subdir=proto'
buf generate
git diff --exit-code -- gen
./gradlew check verifyConfluentFixtures
```

Add a disposable Confluent 7.7.1 job that runs:

```bash
./gradlew verifyConfluentFixtures \
  -PschemaRegistryUrl=http://localhost:8081
./gradlew verifyConfluentBackwardCompatibility \
  -PschemaRegistryUrl=http://localhost:8081
```

Requirements:

- Compare breaking changes against the last stable tag, never a moving branch.
- Run both compatible-addition and incompatible-change cases.
- Fail on generated drift.
- Upload fixture checksums and reports.
- Validate that all expected subjects are `BACKWARD`.

## 6. Semantic compatibility policy

Document the `v1.0.1` incident where `emergency_contact` became `date_of_birth` at the same field number. Treat it as source/JSON semantic breakage even though the scalar wire encoding remained parseable.

Enforce:

- additive fields/RPCs: minor release;
- field removal/rename/number reuse/incompatible type changes: major contract change;
- incompatible topic, required-header, subject-strategy, retry, or DLQ changes: new contract generation and topics;
- optional headers only when existing consumers safely ignore them;
- Java/Go artifacts use the same source contract version.

Add an explicit semantic review checklist because Buf’s binary checks alone do not detect domain-meaning changes.

## 7. Make publication immutable and coordinated

Recommended target:

```text
gym-proto                         v1.1.0
com.gym.proto:gym-proto-java     1.1.0
github.com/pploc/proto-go        v1.1.0
contracts/v1 fixture set         v1.1.0
```

Release rules:

1. Branch and PR events validate only.
2. Protected `v*` tags publish.
3. Version comes only from the tag.
4. Refuse mismatched manifest release targets.
5. Refuse conflicting existing tags/packages.
6. Do not mutate source/version files during release.
7. Make generated Go publication atomic or explicitly recoverable; do not push `main` and then hope the tag succeeds.
8. Publish all contract artifacts needed by Go/Java tests, not only Kafka fixtures.
9. Attach exact source SHA, checksums, and reports to the release.
10. Protect tags from force updates.

## 8. External artifact smoke tests

Create clean temporary Java and Go consumers outside the source repository:

- Java resolves `gym-proto-java:1.1.0` from the package registry.
- Go resolves `github.com/pploc/proto-go@v1.1.0` without `replace`.
- Both compile representative Identity, Member, Payment, and Membership types.
- Go fixture files resolve from the published module.
- Checksums match release evidence.

## Verification commands

```bash
cd /home/phucl/Workplace/gapi/gym-proto
buf format -d --exit-code proto
buf lint proto
buf breaking proto --against '.git#tag=v1.0.6,subdir=proto'
buf generate
git diff --exit-code -- gen
./gradlew clean check verifyConfluentFixtures
```

Run live Registry checks through the repository’s disposable Compose/Testcontainers environment and clean it with the documented teardown command.

## Evidence produced

- Buf format/lint/breaking report.
- Generated-code no-drift report.
- Nine-case fixture report and checksums.
- Positive/negative Registry compatibility report.
- Java and Go external-resolution report.
- Release tag/package/checksum matrix.
- Manifest with accurate technical and approval status.

## G1 exit criteria

- All Phase 0 contracts are represented in source and generated artifacts.
- Internal RPC has no HTTP route.
- All nine fixtures pass offline and live.
- Buf and generated-diff gates are enforced in CI.
- Java and Go packages resolve externally at matching immutable versions.
- Publication is tag-only and reproducible.
- Manifest evidence is truthful; missing approvals remain pending.
