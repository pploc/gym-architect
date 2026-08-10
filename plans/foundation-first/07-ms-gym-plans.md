# Phase 7 — Build `ms-gym-plans`

> **Historical transport note:** G7 proved native Spring MVC business HTTP on `8080` and selected-gym administration. [Phase 9](09-kong-grpc-gateway-openapi.md) supersedes those targets with stable identity, `SUPER_ADMIN` mutations until staff assignment exists, Kong gRPC-Gateway on `50051`, and Actuator-only `8080`. Preserve G7 evidence unchanged.

## Objective

Reach G7 by implementing the released Plans contract as the sole owner of gym locations, gym-specific membership plans, and VND list prices.

## Prerequisites

- [Phase 6](06-plans-contracts.md) passed G6.
- Immutable Plans-capable Java contract resolves from the approved package coordinate.
- Create or clone `pploc/ms-gym-plans` only with repository-owner authorization.

## Service baseline

Use the smallest established Java service stack:

- Java 26 and Spring Boot 4;
- Gradle wrapper;
- `common-java` stable release;
- released `gym-proto-java` Plans contract;
- Spring Data JPA and `JpaSpecificationExecutor`;
- PostgreSQL `plans_db` and Flyway;
- native gRPC on `50051` with required mTLS;
- in-process Spring MVC HTTP/JSON on `8080` for mapped public methods (Kong target; no Go grpc-gateway sidecar);
- validation and Actuator health;
- existing reusable Java CI, Docker, and Helm patterns.

Do not copy Member Kafka, outbox, processed-event, scheduler, lifecycle, cache, or Payment code.

## Schema

Create one clean initial migration.

### `gym_locations`

- `id` opaque string primary key;
- `chain_id`, `name`, `address`, `city`, `status`;
- created and updated timestamps;
- status constrained to `ACTIVE` or `CLOSED`.

### `membership_plans`

- `id` opaque string primary key;
- `gym_id` local foreign key to `gym_locations`;
- `name`, `plan_type`, `duration_days`, `price_vnd`, `description`, `active`;
- created and updated timestamps;
- type constrained to `MONTHLY`, `YEARLY`, or `LIFETIME`;
- `price_vnd >= 0`;
- positive duration for monthly/yearly and no duration for lifetime.

Add only indexes used by frozen filters: location chain/city/status and plan gym/type/active.

## Application behavior

- Public and admin adapters call the same application services.
- Validate request fields at HTTP and gRPC boundaries.
- Use DB constraints as final invariant enforcement.
- Use repository ID methods for direct lookups and writes.
- Compose Spring Data JPA Specifications for location and plan filters. Add no custom JPQL or native filtering query.
- `GetActiveGym` returns only an active location.
- `ResolvePurchasablePlan` verifies active gym, active plan, and exact plan/gym ownership before returning canonical terms.
- No request may set an authoritative value outside the Plans-owned record.

## Security

- Public HTTP accepts only Kong-established trusted claims.
- `SUPER_ADMIN` creates locations and may manage any location or plan.
- `ADMIN` updates only the selected gym and plans belonging to it.
- Authenticated users may read locations and plan catalog.
- Workload gRPC derives identity from verified certificate SAN and ignores end-user claim metadata.
- Identifier may call only `GetActiveGym`.
- Member may call only `ResolvePurchasablePlan`.
- Missing, wrong, or swapped workload identity fails closed.

## Tests

Use `given_when_then` names and explicit setup, action, and assertion sections.

Cover:

- create, update, get, and list locations;
- create, update, get, and list plans;
- composed chain/city/status and gym/type/active filters;
- zero and negative VND prices;
- monthly/yearly/lifetime duration rules;
- missing, inactive, closed, and plan/gym mismatch cases;
- unauthenticated reads and writes;
- admin selected-gym scope and super-admin access;
- Identifier and Member workload allowlists;
- missing/wrong certificates;
- internal methods absent from HTTP routes;
- empty-database Flyway startup.

## Delivery

- Add Dockerfile and health-compatible runtime. **Done.**
- Reuse `gym-infra` Java CI and Docker workflows. **Done** (`.github/workflows/ci.yml` → `java-ci` with `build`, `docker-build` for `ms-gym-plans`).
- Pin released dependencies; no `mavenLocal()` or local contract substitution. **Done** (`common-java:2.0.2`, `gym-proto-java:3.0.0`).
- Public HTTP binds generated protobuf messages as camelCase JSON via `common-java` `ProtobufJsonHttpMessageConverter`. **Done.**
- Add a Plans Helm values example with HTTP, gRPC, health, DB, and TLS settings. **Done** (`gym-infra/helm/gym-service/examples/ms-gym-plans-values.yaml`; evidence under `docs/evidence/foundation-first/g7/`).

## Verification

```bash
cd /home/phucl/Workplace/gapi/ms-gym-plans
./gradlew startEnv
./gradlew clean build
./gradlew stopEnv
```

Also:

1. Start with empty PostgreSQL and inspect constraints/indexes.
2. Start image and verify readiness/health.
3. Run HTTP fixtures and mTLS workload matrix.
4. Render the shared Helm chart with Plans values.
5. Inspect resolved dependencies for exact common and proto versions.
6. Search production configuration for Kafka, Schema Registry, topics, outbox, cache, and schedulers; require no matches beyond explicit non-goal documentation.

## Evidence produced

- unit and integration test report;
- empty-DB migration/schema report;
- API and validation fixture report;
- HTTP exposure and mTLS authorization matrix;
- dependency-resolution report;
- image metadata and health report;
- Helm render and NetworkPolicy report;
- no-messaging/no-cache static report.

## G7 exit criteria

- Plans is the sole tested owner of location and catalog data.
- Contract, DB constraints, HTTP, and gRPC behavior agree.
- Internal methods are mTLS-only and externally unmapped.
- Filters use JPA Specification composition.
- Service runs without Kafka, cache, outbox, scheduler, or Payment dependencies.
- Image and Helm deployment evidence pass.
