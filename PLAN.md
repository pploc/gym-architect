# Service Catalog and Roadmap Pointer

## Authoritative execution plan

Use [`plans/foundation-first/README.md`](plans/foundation-first/README.md). G0–G9 foundation for Identifier, Member, and Plans is technically complete; owner approval remains separate. G9 uses Kong in front of generated Go `grpc-gateway` for Member and Plans browser APIs. Final immutable-release, locked clean-source, protected-CI, and sanitized-evidence proof is recorded in [`evidence/foundation-first/p9-final/README.md`](evidence/foundation-first/p9-final/README.md). Deferred services stay catalog-only until a later roadmap gate.

## Active services

| Service | Stack | Database | Ownership |
|---|---|---|---|
| `ms-gym-identifier` | Go + Gin | PostgreSQL `identity_db` | Identity, credentials, and stable identity tokens |
| `ms-gym-member` | Java 26 + Spring Boot 4 | PostgreSQL `member_db` | Profiles, subscriptions, membership lifecycle and validation |
| `ms-gym-plans` | Java 26 + Spring Boot 4 | PostgreSQL `plans_db` | Gym locations, gym-specific plans, VND pricing |

## Deferred catalog

These services remain documented for architecture continuity but have no actionable implementation phase yet:

| Service | Intended stack | Intended database | Future role |
|---|---|---|---|
| `ms-gym-payment` | Java 26 + Spring Boot 4 | PostgreSQL | Provider payments, refunds, transaction history |
| `ms-gym-checkin` | Go + Gin | YugabyteDB | QR validation and check-in records |
| `ms-gym-workout` | Go + Gin | Cassandra | Workout logging |
| `ms-gym-trainer` | Java 26 + Spring Boot 4 | PostgreSQL | Trainer schedules and bookings |
| `ms-gym-promotion` | Java 26 + Spring Boot 4 | PostgreSQL | Discount campaigns and coupon reservations |
| `ms-gym-notification` | Go + Gin | Cassandra | Notification fan-out and delivery history |
| `ms-gym-analytics` | Java 26 + Spring Boot 4 | YugabyteDB | Materialized reports and trends |

Do not treat deferred service documents as implementation instructions until roadmap adds a gate for them.

## Current dependency order

```mermaid
flowchart LR
    F[G0-G5<br/>Foundation complete] --> C[Phase 6<br/>Plans contracts]
    C --> P[Phase 7<br/>Build ms-gym-plans]
    P --> I[Phase 8<br/>Identifier + Member + Plans integration]
    I --> S[Phase 9 Stage 0<br/>Stable identity + explicit gym context]
    S --> G[Phase 9 Stages 1-4<br/>Kong gRPC-Gateway + generated OpenAPI 3.0]
```

## Target service interaction

```mermaid
flowchart LR
    C[Browser] -->|browse gyms and plans through Kong| PL[Plans]
    C -->|gym-scoped membership request through Kong| MB[Member]
    ID[Identifier] -->|GetActiveGym for trainer validation only| PL
    MB -->|ResolvePurchasablePlan gym + plan| PL
```

Member keeps `PurchaseMembership`; production Payment remains deferred. G8 proved the outbound Payment port with `gym-infra/kong/fixtures/fake-payment`.

## Change index

| Change | Update together |
|---|---|
| Plans API or fields | Phase 6, `gym-proto`, Plans service spec, Member purchase boundary |
| Gym ownership | Architecture overview, Identifier, Member, Plans, Check-in boundary, infrastructure routes |
| Subscription terms | Member spec, Plans spec, purchase flow, future Payment boundary |
| External route | Protobuf `google.api.http` annotation, generated OpenAPI 3.0, Kong route documentation, NetworkPolicy documentation |
| Kafka event | Kafka catalog, canonical Protobuf event, producer/consumer ownership |
