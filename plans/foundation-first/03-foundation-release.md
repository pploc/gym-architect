# Phase 3 — Cross-Language RC and Stable Foundation Releases

> **Historical release note:** G3/G4 validated common releases against the original selected-gym trust profile. [Phase 9](09-kong-grpc-gateway-openapi.md) supersedes mutable gym/membership claims and the public transport boundary; retain this phase only as foundation-release evidence.

## Objective

Reach G3 with immutable release candidates that pass one cross-language matrix, then reach G4 after `ms-gym-member` validates the Java RC and stable common releases are published.

## Current status: G3 PASSED

The authoritative cross-language matrix run [30894691218](https://github.com/pploc/common-go/actions/runs/30894691218) passed all Java-to-Go, Go-to-Java, retry/DLQ, and Registry immutability gates. Release candidates `com.gym:common-java:2.0.0-rc.6` and `github.com/pploc/common-go@v0.3.0-rc.7` have been published.

## Prerequisites

- [Phase 1](01-gym-proto.md) passed G1.
- [Phase 2](02-common-libraries.md) passed G2 on exact source SHAs.
- Read [release/adoption rules](README.md).
- No ordinary-service adoption is allowed.

## In-scope repositories

- `gym-proto` or a dedicated foundation contract workflow location
- `common-java`
- `common-go`
- `ms-gym-member` only as the designated validator in Phase 4

## Explicit non-goals

- No Kong or Identifier live interoperability.
- No rollout to other services.
- No human approval fabrication.
- No JSON-topic migration tests.

## 1. Build one cross-language matrix workflow

The workflow checks out exact immutable refs for:

- `gym-proto` and generated `proto-go`;
- `common-java` RC;
- `common-go` RC.

It starts one clean Kafka and Confluent Schema Registry environment, preregisters all nine subjects with `BACKWARD`, and disables client auto-registration.

### Required matrix cases

1. Decode all canonical fixtures in Java and Go.
2. Publish every event from Java and consume in Go.
3. Publish every event from Go and consume in Java.
4. Compare topic, subject, ordering key, descriptor, headers, payload, and complete frame.
5. Verify W3C propagation and fallback behavior.
6. Inject retryable handler failures and capture attempt timestamps.
7. Inject permanent failures.
8. Inject malformed framing and unknown schema IDs.
9. Force DLQ publication failure.
10. Restart/rebalance consumers before commit and verify redelivery.
11. Prove no later offset is committed past an incomplete record.
12. Verify producer cancellation and acknowledgement timeout behavior.

## 2. Machine-readable evidence

Produce a matrix artifact similar to:

```json
{
  "gymProto": "v1.1.0",
  "protoGo": "v1.1.0",
  "commonJava": "v2.0.0-rc.1",
  "commonGo": "v0.3.0-rc.1",
  "kafka": "Confluent 7.7.1",
  "schemaRegistry": "Confluent 7.7.1",
  "fixtureCases": 9,
  "javaToGo": "pass",
  "goToJava": "pass",
  "retryCommitDlq": "pass",
  "rawFramePreservation": "pass"
}
```

Also retain:

- exact source SHAs and artifact checksums;
- Registry subject/version report;
- retry timestamps and offsets;
- source/DLQ key/value/header hashes;
- restart/redelivery transcript;
- command logs and CI run URLs.

Do not place secrets, payloads containing personal data, or credentials in evidence.

## 3. RC publication

Recommended immutable candidates:

```text
gym-proto / proto-go  v1.1.0
common-java           v2.0.0-rc.1
common-go             v0.3.0-rc.1
```

RC rules:

1. Publish only after the exact tagged source passes library verification.
2. Never rebuild an existing RC tag with different content.
3. Package metadata records source SHA and dependency versions.
4. External clean projects resolve RCs without local repositories or `replace` directives.
5. Release notes list scope and known non-technical pending approvals.
6. Owner approval remains a separate manifest field.

## 4. G3 acceptance

G3 passes when:

- every cross-language case passes on exact RC artifacts;
- malformed values survive DLQ byte-for-byte in both languages;
- commit-after-success/confirmed-DLQ and no-commit-on-DLQ-failure are demonstrated;
- Java typed consumer delivery is proven for multiple generated classes;
- Go race/integration tests pass;
- external dependency resolution succeeds;
- no unresolved high-severity technical blocker remains.

If G3 fails, fix the owning library or contract and issue a new RC. Do not edit/reuse the failed tag.

## 5. Designated Java validation

After G3, execute [Phase 4](04-ms-gym-member.md) against `common-java v2.0.0-rc.1` and the exact G1 proto release.

The validator must prove:

- actual registered gRPC method policy enforcement;
- shared error/tracing behavior;
- typed identity/payment consumers;
- transactional idempotency and duplicate handling;
- outbox UUID to Kafka `event-id` identity;
- all membership outputs;
- retry, commit, DLQ, and Registry behavior;
- clean initial deployment with only Protobuf topics.

Member is allowed to consume the RC only for validation. No other service may do so.

## 6. Stable promotion

After Member passes:

1. Tag/publish `common-java v2.0.0` from the exact validated RC content or an explicitly revalidated commit.
2. Tag/publish `common-go v0.3.0` after the same matrix and external-import checks pass.
3. Repin Member from Java RC to stable and rerun the complete suite.
4. Verify stable package checksums and source provenance.
5. Update the compatibility matrix and manifest technical evidence.
6. Preserve pending human approvals honestly.

`common-go` stable does not wait for Identifier service adoption. Identifier requires stable `common-go`; requiring Identifier first would recreate the dependency cycle. Record Go service adoption as pending until Phase 5.

## 7. Release/adoption matrix

| Stage | `gym-proto` / `proto-go` | `common-java` | `common-go` | Allowed adopters |
|---|---|---|---|---|
| Contract implementation | RC/develop | develop | develop | Contract harness only |
| G3 foundation RC | `v1.1.0` | `v2.0.0-rc.1` | `v0.3.0-rc.1` | Member validator only |
| Member validation | same | RC | RC remains test-only | Member only |
| G4 stable | same | `v2.0.0` | `v0.3.0` | Identifier may use stable Go |
| G5 edge proof | compatible patch allowed | stable | stable | Remaining controlled adoption opens |

## Verification

At minimum rerun:

```bash
# common-java
./gradlew clean check jacocoTestCoverageVerification kafkaContractIntegration

# common-go
make verify
go test -tags=integration -race ./...

# external consumers
# Resolve exact tags without mavenLocal or Go replace directives.
```

The matrix workflow is authoritative for cross-language compatibility. A unit-test-only pass cannot promote an RC.

## Evidence produced

- `cross-language-matrix.json`.
- `retry-dlq-transcript.json`.
- `raw-frame-checksums.json`.
- external import report.
- RC/stable provenance and checksums.
- Member adoption report after Phase 4.
- compatibility manifest with truthful approval state.

## G4 exit criteria

- G3 matrix passes immutable RCs.
- Member validates Java RC and repins to stable.
- Stable Java and Go artifacts resolve externally.
- Stable artifacts match tested source/dependencies.
- Release workflows are immutable and tag-only.
- Other services remain frozen until G5.
- Missing owner approvals remain pending, not fabricated.
