# Foundation-First Platform Roadmap

## Purpose

Finish shared foundations, then establish three service boundaries in dependency order:

```text
contract freeze
  -> gym-proto release
  -> common-java and common-go
  -> cross-language validation
  -> Member foundation validation
  -> Kong and Identifier interoperability
  -> Plans contracts
  -> ms-gym-plans
  -> Identifier + Member + Plans integration
```

Only Identifier, Member, and Plans are active service scope. Other service documents are catalog-only until a later roadmap explicitly opens them.

## Current status

- G0–G4: historical foundation gates completed with evidence retained under `docs/evidence/foundation-first`.
- G5: passed Identifier-led Kong/Identifier/Member E2E through `gym-infra/kong/run-g5.sh`.
- G6: Plans/Member/Payment contracts published as `v3.0.0` / `gym-proto-java:3.0.0` / `proto-go/v3@v3.0.0`.
- G7: passed. Plans owns locations/catalog; evidence under `docs/evidence/foundation-first/g7/local-2026-08-09/` (reverified 2026-08-09 on current develop HEADs: plans `ecbccfac`, gym-infra `96dd2298`, docs `69847db4`; 70 tests, Helm/NetworkPolicy render pass).
- G8: passed. Identifier + Member + Plans clean-boundary E2E via `gym-infra/kong/run-g8.sh` (`RUN_EXIT:0`); evidence under `docs/evidence/foundation-first/g8/local-2026-08-09/`. Owner approval remains pending.

G5 evidence validates the Member boundary that existed during Phase 5. Phase 8 revised that boundary; G8 evidence supersedes G5 for location validation and catalog ownership. Member external HTTP remains intentionally unexposed.

No customer or production data exists. Phases 6–8 use a coordinated contract and schema reset. Do not add migration, backfill, compatibility forwarding, dual-write, or rollback machinery for disposable pre-production data.

## Phase documents

1. [Phase 0 — Contract freeze](00-contract-freeze.md)
2. [Phase 1 — `gym-proto`](01-gym-proto.md)
3. [Phase 2 — Common libraries](02-common-libraries.md)
4. [Phase 3 — Foundation release](03-foundation-release.md)
5. [Phase 4 — `ms-gym-member` foundation validator](04-ms-gym-member.md)
6. [Phase 5 — Kong and Identifier](05-kong-identifier.md)
7. [Phase 6 — Plans contracts](06-plans-contracts.md)
8. [Phase 7 — `ms-gym-plans`](07-ms-gym-plans.md)
9. [Phase 8 — Three-service integration](08-three-service-integration.md)

Each phase is an implementation handoff. Execute it only after prerequisites pass and preserve evidence for next gate.

## Target ownership

| Boundary | Identifier | Member | Plans |
|---|:---:|:---:|:---:|
| Users, credentials, JWTs | owner | — | — |
| Selected-gym token | owner | membership lookup | active-gym lookup |
| Member profile | — | owner | — |
| Subscription and lifecycle | — | owner | — |
| Gym location | opaque reference | opaque reference | owner |
| Plan catalog and price | — | purchased snapshot | owner |
| Membership validation | — | owner | — |

Every plan belongs to one gym; one gym can have many plans. V1 pricing is non-negative `int64 price_vnd` and VND only. Cross-service IDs are opaque strings and never cross-service database foreign keys.

## Governing rules

1. `gym-proto` owns API/event schemas, auth vocabulary, error mappings, topic rules, and generated artifacts.
2. Common libraries own shared trusted-claim, workload-auth, observability, Kafka framing, retry, commit, and DLQ behavior.
3. Kong establishes external JWT trust. User roles and workload identities remain separate trust domains.
4. Identifier owns identity, Member owns subscriptions, and Plans owns locations/catalog.
5. Member orchestrates membership purchase and obtains trusted terms from Plans. Client input never supplies authoritative price, type, or duration.
6. Member stores purchased terms on subscription. Later catalog edits affect only later purchases.
7. Plans V1 has no Kafka producer, consumer, topic, outbox, or cache.
8. Releases are immutable and tag-derived. Technical pass and owner approval remain separate.
9. No custom persistence queries for Plans filters; compose Spring Data JPA Specifications.
10. No work for deferred services unless needed to preserve an explicit boundary reference.

## Readiness gates

| Gate | Meaning | Required evidence |
|---|---|---|
| G0 — Contract frozen | Shared vocabulary and wire decisions agree | API/topic/JWT/auth matrices and generated diff |
| G1 — Proto released | Canonical contracts and generated artifacts are immutable | Tag, coordinates, checksums, Buf reports |
| G2 — Libraries complete | Common libraries implement frozen contracts | Unit/static/race and Kafka/Registry tests |
| G3 — Foundation RC | Cross-language matrix passes | Java-to-Go, Go-to-Java, retry/commit/DLQ reports |
| G4 — Stable foundation | Stable common tags and Member validation pass | Stable tags, import checks, Member report |
| G5 — Edge interoperability | Kong and Identifier enforce trust contract | Proxy capture and auth/member E2E |
| G6 — Plans contracts | Plans API and revised Member boundary are immutable | Buf checks, break report, artifacts, exposure/auth matrices |
| G7 — Plans service | Plans behavior, schema, security, and deployment pass | Tests, migration report, API fixtures, image/Helm evidence |
| G8 — Three-service integration | Identifier, Member, and Plans pass clean-boundary E2E | Three-service tests, schema inspection, mTLS and Kong evidence |

## Active-scope freeze

G8 closed Identifier/Member/Plans. Do not start implementation phases for Payment, Check-in, Workout, Trainer, Promotion, Notification, or Analytics until a later roadmap gate. Their docs may be corrected only to preserve Identifier/Member/Plans boundaries.

Allowed repositories for Phases 6–8 (historical scope list):

- `gym-proto`;
- `ms-gym-plans`;
- `ms-gym-member`;
- `ms-gym-identifier`;
- shared infrastructure needed to run those three services;
- architecture, service, flow, and evidence docs.

## Shared evidence requirements

Every report records exact git SHAs, artifact versions, checksums, infrastructure versions, commands, results, and timestamps. Owner approval remains explicitly pending until provided.

## Documentation maintenance

When a boundary changes, update this index, affected phase, `gym-proto` contracts, architecture, service, flow, and infrastructure docs together. Historical gate evidence remains unchanged; add supersession notes instead of rewriting what a completed gate proved.
