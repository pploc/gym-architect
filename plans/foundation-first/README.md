# Foundation-First Platform Roadmap

## Purpose

Complete and release the shared platform foundations before ordinary services adopt them. The dependency order is:

```text
contract freeze
  -> gym-proto release
  -> common-java and common-go in parallel
  -> cross-language release candidates
  -> ms-gym-member Java validation
  -> stable common releases
  -> Kong fixture setup and ms-gym-identifier
  -> ordinary-service adoption
```

`gym-proto` comes first because it owns generated types and wire contracts. `common-java` and `common-go` can proceed in parallel only after those contracts are frozen.

## Current baseline

Recheck every SHA and working tree before execution.

| Repository | Rechecked baseline | Current classification |
|---|---|---|
| `gym-proto` | `develop` at `bc08215`; tag `v1.1.0` exists | G1 technical artifacts published; reconcile stale candidate wording while owner approval remains pending |
| `common-java` | `develop` at `3297115`; only tag `v1.0.1` | Frozen transport and legacy-envelope removal are implemented; live release validation, current metadata, and G2 evidence remain |
| `common-go` | `develop` at `82f7ab7`; no tags | Frozen Kafka producer/consumer/retry/DLQ source and live tests exist; reusable release validation and G2 evidence remain |
| `ms-gym-member` | `develop` at `342377d`; one local commit ahead of origin | Still pins `gym-proto-java:1.0.6` and retains obsolete Member QR ownership; Phase 4 refactor required |
| `gym-infra` | `develop` at `f24d64f`; one local commit ahead of origin | Shared Registry URL exists; deployable Registry/governance and Kong trust-boundary evidence remain |
| `ms-gym-identifier` | Repository absent | Create only after its contracts and stable dependencies are ready |

Rechecked on 2026-08-03. These SHAs are observations, not immutable gates; recheck them immediately before execution.

## Phase documents

1. [Phase 0 — Contract freeze](00-contract-freeze.md)
2. [Phase 1 — gym-proto](01-gym-proto.md)
3. [Phase 2 — Common libraries](02-common-libraries.md)
4. [Phase 3 — Foundation release](03-foundation-release.md)
5. [Phase 4 — ms-gym-member](04-ms-gym-member.md)
6. [Phase 5 — Kong and Identifier](05-kong-identifier.md)

Each phase is an implementation handoff. Execute it only after its prerequisites pass and preserve evidence for the next gate.

## Current gate status

- **G1:** technical release exists; reconcile contradictory publication wording without changing the immutable `v1.1.0` tag or fabricating owner approval.
- **G2:** technically passed for the exact source trees recorded in [G2 evidence](../../evidence/foundation-first/g2/3297115bf333a3781e4248535a7c965d14cb0388--82f7ab7fb7d6e26fa6db3f076f707883b87e7d39.md). Java and Go source/static/race gates and real Kafka/Registry failure-path proof passed; no common-library artifact was tagged or published.
- **G3:** required and not started. It needs immutable common-library RC artifacts and the bidirectional Java/Go matrix; separate G2 fixture suites do not replace it.
- **Phase 4:** starts RC transport adoption only after G3. Its independent correctness work may proceed earlier, including removal of obsolete Member QR ownership required by the already-released G1 contract.

## Governing rules

1. `gym-proto` owns API schemas, event schemas, auth vocabulary, error mappings, topic/subject rules, retry/DLQ contracts, and cross-language fixtures.
2. Common libraries pin tagged generated artifacts. Ordinary services pin stable common-library releases.
3. `ms-gym-member` is the designated Java release-candidate validator. Contract harnesses and Kong mock upstreams are not production adopters.
4. Do not copy unfinished claim parsing, Protobuf framing, retry, commit, or DLQ logic into services.
5. Kafka is at-least-once. Producer idempotence does not make DB and Kafka atomic. Services use idempotent handlers and transactional outboxes where needed.
6. Kong establishes the external JWT trust boundary. Common libraries consume trusted claims; normal downstream paths do not revalidate end-user JWT signatures.
7. `gym_id` is an opaque cross-service string, never a database foreign key to another service database.
8. User roles and workload identities are separate trust domains.
9. Releases are immutable and tag-derived. Branch pushes validate but do not publish or mutate source version files.
10. Technical evidence and human approval are separate. Never fabricate owner approval; leave the manifest pending when approval is absent.
11. Kafka is greenfield. No deployed JSON topics or offsets exist. Implement only the final Protobuf topics and groups; do not build dual-read, dual-write, offset migration, or JSON compatibility.

## Rejected circular or unnecessary gates

- Kong evidence does not block Kafka implementation or Kafka release.
- Identifier does not block base Kong fixture configuration.
- Owner approval does not block source implementation, testing, or an immutable technical RC; it gates promotion/production use.
- `ms-gym-member` must not migrate against an untagged or Maven-local shared build. It may consume an immutable RC solely as the designated validator.
- `common-go` stable must not require Identifier adoption before Identifier can consume it. Record service adoption separately.

## Readiness gates

| Gate | Meaning | Required evidence |
|---|---|---|
| G0 — Contract frozen | Cross-repository vocabulary and wire decisions are consistent | API/topic/JWT/auth matrices, reviewed generated diff |
| G1 — Proto released | Canonical contracts and Java/Go artifacts are immutable | Tag, package coordinates, checksums, Buf and fixture reports |
| G2 — Libraries complete | Both common libraries implement the frozen contract | Unit/static/race tests and real Kafka/Registry tests |
| G3 — Foundation RC | Exact tagged RCs pass one cross-language matrix | Java-to-Go, Go-to-Java, retry/commit/DLQ reports |
| G4 — Stable foundation | Stable common tags exist; Java RC passed Member validation | Stable tags, external import checks, Member adoption report |
| G5 — Edge interoperability | Kong and Identifier enforce the live trust contract | Proxy capture and register/login/refresh/event E2E report |

Definitions:

- **Implementation complete:** source and tests satisfy the frozen contract.
- **Release candidate:** immutable prerelease tag passes the contract matrix.
- **Stable:** immutable non-RC tag has no unresolved technical blocker.
- **Adoption proven:** a designated real service passes service-level integration and rollback tests.
- **Owner-approved:** governance status recorded only after accountable approval.

## Release and adoption matrix

Recommended targets; adjust only for a documented SemVer reason.

| Stage | `gym-proto` / `proto-go` | `common-java` | `common-go` | Allowed adopters |
|---|---|---|---|---|
| Contract implementation | RC/develop | develop | develop | Contract harness only |
| Foundation RC | `v1.1.0` | `v2.0.0-rc.1` | `v0.3.0-rc.1` | Member validator; disposable fixtures |
| Java validation | same | RC | RC | Member only |
| Stable foundations | same | `v2.0.0` | `v0.3.0` | Identifier may pin stable |
| G5 passed | compatible patch allowed | stable | stable | Controlled ordinary-service adoption opens |

## Ordinary-service adoption freeze

Until G4, Workout, Check-in, Payment, Trainer, Notification, Promotion, and Analytics must not:

- import foundation RCs;
- emit the new `.v1` records;
- implement local copies of shared framing, trusted-claim parsing, retry, or DLQ logic.

Allowed exceptions:

- `ms-gym-member` as the Java RC validator;
- foundation contract-test applications;
- disposable Kafka/Schema Registry environments;
- Kong mock upstream and fixture signer.

After G4, Identifier may consume stable `common-go`. Open remaining adoption only after G5, one service at a time.

## Shared evidence requirements

Every report records exact git SHAs, artifact versions, fixture checksums, infrastructure versions, command results, and timestamps. Expected artifacts:

- Buf format/lint/breaking report;
- generated-code drift report;
- fixture and release checksums;
- Schema Registry positive/negative compatibility report;
- cross-language matrix;
- retry/commit/DLQ transcript;
- raw source/DLQ frame checksums;
- Member adoption report;
- Kong upstream-header capture;
- Identifier E2E report;
- package provenance/SBOM where available.

A missing approval remains explicitly pending and does not become a technical pass.

## Documentation maintenance

When a contract decision changes, update this index, the affected phase file, `gym-proto/contracts`, relevant ADRs, and platform architecture/service documentation together. Replace obsolete implementation plans rather than leaving contradictory release or transport instructions active.
