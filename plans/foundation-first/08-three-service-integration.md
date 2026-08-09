# Phase 8 — Identifier, Member, and Plans Integration

> **Status:** G8 technical pass on prior generation. Evidence: `docs/evidence/foundation-first/g8/local-2026-08-09/` (`./kong/run-g8.sh` → `RUN_EXIT:0`). Active code now targets contract break (Java `4.0.0` / Go `github.com/pploc/proto-go`, enums, RPC-specific messages); G8 fixture/scripts updated. Fresh E2E re-run blocked until service images resolve published/staged pins. Owner approval pending.

## Objective

Reach G8 by removing location/catalog ownership from Member, splitting Identifier's downstream clients, and proving the clean three-service boundary through Kong, workload mTLS, isolated databases, and membership purchase E2E.

## Prerequisites

- [Phase 6](06-plans-contracts.md) passed G6 with immutable artifacts.
- [Phase 7](07-ms-gym-plans.md) passed G7.
- Preserve G5 topology and evidence as historical pre-split proof.
- Recreate disposable databases; do not build migration compatibility for pre-production data.

## Part A — Revise Member

### Remove Plans ownership

Delete Member location/catalog services, adapters, entities, repositories, moved RPC implementations, and tests. Member schema must contain no `gym_locations`, `membership_plans`, or foreign keys to Plans-owned IDs.

Retain opaque `gym_id` and `plan_id` on subscriptions and add:

- `plan_type_snapshot`;
- `duration_days_snapshot`;
- `price_vnd_snapshot`.

Member-local plan type is snapshot vocabulary, not catalog ownership.

### Pending purchase

Add Member-owned pending purchase persistence with:

- purchase, member, user, gym, and plan IDs;
- type, duration, and price snapshots;
- provider, payment ID, status, and timestamps.

Use Spring Data JPA Specification composition for lookup/filter behavior. Keep existing outbox and processed-event idempotency.

### Purchase flow

1. Resolve trusted user, member, and selected gym.
2. Reject nonblank discount code until authoritative discount pricing exists.
3. Call Plans `ResolvePurchasablePlan(plan_id, selected_gym_id)` over Member workload mTLS.
4. Persist pending purchase before calling Payment.
5. Call Payment with `reference_id=purchase_id`.
6. Store returned payment ID and return existing payment response.

### Payment completion

Within one Member transaction:

1. claim event through existing idempotency;
2. find and lock pending purchase by `reference_id`;
3. verify membership payment type, user, gym, payment ID, and amount;
4. activate or renew from frozen terms;
5. store snapshots on subscription;
6. mark purchase completed;
7. write membership outbox event.

No completion or lifecycle path rereads Plans. Pause, resume, expiry, warnings, renewals, and event creation use stored subscription snapshots. Completion replay creates no duplicate activation or renewal.

## Part B — Revise Identifier

- `MemberClient` retains only membership lookup.
- Add independent `PlansClient` for `GetActiveGym` with separate address and mTLS material.
- `SelectGym` calls Plans first, then Member. Any missing/inactive gym, unavailable dependency, or authorization failure prevents token issuance.
- Existing trainer gym validation moves from Member to Plans.
- Workload clients send no `x-user-*` trusted headers.
- Construct and close both clients independently.

Test `NONE`, `ACTIVE`, `PAUSED`, call order, independent outages, wrong server name/CA, and wrong client certificate.

## Part C — Additive G8 topology

Keep G5 files unchanged. Add separate G8 compose, runner, business check, certificates, and Kong configuration.

Topology:

- `identity_db`, `member_db`, and `plans_db` PostgreSQL databases;
- Redis;
- Kafka and Schema Registry for Identifier and Member only;
- Identifier, Member, Plans, Kong;
- Phase-8-only fake Payment fixture.

The fake implements only payment initiation, stable purchase-reference correlation, completion event publication, and replay. It is not a production Payment service.

### Ingress and mTLS

- Kong routes public Plans HTTP methods to Plans `8080`.
- Internal Plans methods remain unmapped.
- Identifier and Member call Plans native gRPC `50051` with distinct certificates.
- NetworkPolicy permits Kong to Plans HTTP, Identifier to Plans gRPC, and Member to Plans gRPC; no broad gRPC ingress.
- Method policy rejects swapped Identifier/Member identities and forged user metadata.

## E2E business flow

1. Start clean databases and dependencies.
2. Register and verify a customer through Kong.
3. Wait for Member profile projection.
4. Provision an authorized administrator.
5. Create active gym and plan through Plans API, never direct SQL.
6. Login as customer and select gym; membership is `NONE`.
7. Initiate Member membership purchase.
8. Trigger fake payment completion.
9. Wait for subscription activation and membership event.
10. Select gym again; selected-gym token reports `ACTIVE`.
11. Inspect frozen purchase/subscription terms.
12. Replay completion; state remains unchanged.

## Negative checks

- anonymous or wrong-scope Plans mutation;
- inactive or missing gym selection;
- inactive plan and plan/gym mismatch;
- client attempts to supply authoritative terms;
- Identifier calling plan resolution;
- Member calling active-gym lookup;
- missing/wrong mTLS peer;
- forged trusted user headers;
- Plans outage blocks selection and purchase;
- Member outage blocks selected-gym token after Plans succeeds;
- payment amount, user, gym, or payment-ID mismatch;
- catalog edit/deactivation after initiation does not alter frozen activation terms;
- completion replay is idempotent.

## Schema and dependency inspection

Prove:

- `plans_db` alone contains `gym_locations` and `membership_plans`;
- `member_db` contains opaque references and purchased snapshots, with no Plans FK;
- `identity_db` contains no Member or Plans tables;
- Plans receives no Kafka/Schema Registry configuration and starts without those connections.

## Documentation closeout

Update current architecture, Identifier, Member, Check-in boundary, business flow, Kafka, project structure, and infrastructure docs. Keep historical G4/G5 evidence intact and mark it superseded only where the boundary changed.

## Verification commands

```bash
cd /home/phucl/Workplace/gapi/ms-gym-member && ./gradlew clean check
cd /home/phucl/Workplace/gapi/ms-gym-identifier && go test -race ./...
cd /home/phucl/Workplace/gapi/ms-gym-plans && ./gradlew clean check
cd /home/phucl/Workplace/gapi/gym-infra && ./kong/run-g8.sh
```

Render Helm and inspect NetworkPolicy separately. Record all commands, exact SHAs, artifact versions, image digests, checksums, timestamps, and sanitized results.

## Evidence produced

- revised Member test and schema report;
- Identifier split-client and fail-closed report;
- Plans/Member workload mTLS matrix;
- Kong route/header capture;
- positive and negative three-service E2E report;
- purchase snapshot and replay-idempotency report;
- database ownership report;
- Helm/NetworkPolicy report.

## G8 exit criteria

- Identifier, Member, and Plans own only their frozen boundaries.
- No runtime path reads location/catalog data from Member.
- Purchase and activation trust Plans terms and preserve initiation-time snapshots.
- Selected-gym issuance fails closed on Plans or Member failure.
- Public and workload trust boundaries pass positive and negative checks.
- Database isolation and no cross-service FKs are proven.
- Historical evidence remains truthful and revised-boundary evidence is complete.
