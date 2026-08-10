# Phase 6 — Plans Contracts

> **Historical contract note:** G6 froze the selected-gym contract used by G8. [Phase 9 Stage 0](09-kong-grpc-gateway-openapi.md#stage-0--replace-selected-gym-jwt-state-before-gateway-generation) supersedes selected-gym JWT issuance, Identifier-to-Member membership lookup, and request-derived `ADMIN` gym scope. Preserve G6 evidence unchanged.

## Objective

Reach G6 by freezing and releasing the Plans API and revised Member boundary before either service adopts them.

## Prerequisites

- G5 remains completed historical evidence for the pre-split boundary.
- No customer or production data exists.
- Preserve existing G0–G5 evidence; add supersession notes instead of rewriting it.
- Recheck repository SHAs, tags, working trees, and package coordinates before recording evidence.

## Ownership freeze

| Boundary | Owner | Contract |
|---|---|---|
| Users, credentials, JWTs | Identifier | Identifier remains gym-neutral until `SelectGym`. |
| Selected-gym token | Identifier | Plans validates active gym, then Member resolves membership. |
| Member profile | Member | Gym and plan IDs are opaque references. |
| Subscription and lifecycle | Member | Member stores purchased terms and owns membership events. |
| Gym locations | Plans | Canonical location and status owner. |
| Membership-plan catalog | Plans | Canonical type, duration, availability, and VND list price owner. |
| Purchase orchestration | Member | Member resolves trusted terms before Payment initiation. |

Cross-service IDs are strings and never database foreign keys. Plans V1 has no Kafka producer, consumer, topic, outbox, cache, scheduler, or Payment integration.

## Plans V1 domain

- Gym status: `ACTIVE`, `CLOSED`.
- Plan type: `MONTHLY`, `YEARLY`, `LIFETIME`.
- Every plan belongs to exactly one gym.
- `price_vnd` is non-negative `int64`; currency is VND only.
- `MONTHLY` and `YEARLY` require positive `duration_days`.
- `LIFETIME` has no duration.
- Closed gyms and inactive plans are not purchasable.
- Catalog changes affect only purchases initiated after the change.
- No hard-delete API. Status and active flags handle removal from current use.

## Canonical Plans RPCs

One service owns the boundary: `plans.v1.PlansService`.

| RPC | Caller | HTTP | Policy |
|---|---|---:|---|
| `CreateGymLocation` | `SUPER_ADMIN` | `POST /api/v1/gyms` | Create chain location. |
| `UpdateGymLocation` | `ADMIN`, `SUPER_ADMIN` | `PUT /api/v1/gyms/{id}` | Admin is limited to selected gym. |
| `GetGymLocation` | authenticated user | `GET /api/v1/gyms/{id}` | Return canonical location. |
| `ListGymLocations` | authenticated user | `GET /api/v1/gyms` | Filter by chain, city, and status. |
| `CreateMembershipPlan` | `ADMIN`, `SUPER_ADMIN` | `POST /api/v1/gyms/{gym_id}/plans` | Admin is limited to selected gym. |
| `UpdateMembershipPlan` | `ADMIN`, `SUPER_ADMIN` | `PUT /api/v1/plans/{id}` | Admin is limited to plan gym. |
| `GetMembershipPlan` | authenticated user | `GET /api/v1/plans/{id}` | Return catalog terms. |
| `ListMembershipPlans` | authenticated user | `GET /api/v1/gyms/{gym_id}/plans` | Filter by type and active state. |
| `GetActiveGym` | Identifier workload | none | Return only an existing `ACTIVE` gym. |
| `ResolvePurchasablePlan` | Member workload | none | Validate gym match and availability; return trusted terms. |

Workload RPCs receive no trusted user headers and have no gRPC-Gateway or Kong mapping. Certificate identity authorizes the caller: Identifier may call only `GetActiveGym`; Member may call only `ResolvePurchasablePlan`.

`ResolvePurchasablePlan(plan_id, gym_id)` returns canonical `plan_id`, `gym_id`, `plan_type`, `duration_days`, and `price_vnd`. It fails for a missing or inactive plan, a closed gym, or a plan/gym mismatch.

## Revised Member boundary

Remove from `member.v1.MemberService`:

- `GetPlans`;
- `CreateGymLocation`;
- `UpdateGymLocation`;
- `ListGymLocations`;
- `GetGymLocation`;
- their location and catalog messages.

Member retains profile, purchase, pause/resume, status, `GetMembershipStatusByUserId`, `ValidateMembership`, and list-by-status methods.

Clients supply only `plan_id`, provider, and the existing optional `discount_code`. Client input never supplies authoritative gym, type, duration, or price. Until an authoritative discount owner exists, Member rejects nonblank discount codes.

## Purchase snapshot and correlation

1. Member reads member and selected gym from trusted context.
2. Member calls `ResolvePurchasablePlan(plan_id, selected_gym_id)` over mTLS.
3. Member stores a pending purchase with returned terms.
4. Member calls Payment with `reference_id=purchase_id`.
5. `payment.completed.v1.reference_id` identifies that pending purchase.
6. Member validates payment identity, user, gym, type, and amount against the frozen record.
7. Member activates or renews from frozen terms and marks the purchase completed in one transaction.

Member never rereads Plans during payment completion, pause/resume, expiry, renewal, or membership-event creation. Subscription rows retain `gym_id`, `plan_id`, `plan_type_snapshot`, `duration_days_snapshot`, and `price_vnd_snapshot`.

## Errors

Use canonical gRPC status and stable `x-error-code` through `common-java`:

| Condition | Status | Error code |
|---|---|---|
| Invalid field, price, duration, or filter | `INVALID_ARGUMENT` | `INVALID_ARGUMENT` or field-specific validation code |
| Missing gym | `NOT_FOUND` | `GYM_NOT_FOUND` |
| Missing plan | `NOT_FOUND` | `PLAN_NOT_FOUND` |
| Closed gym | `FAILED_PRECONDITION` | `GYM_INACTIVE` |
| Inactive plan | `FAILED_PRECONDITION` | `PLAN_INACTIVE` |
| Plan/gym mismatch | `FAILED_PRECONDITION` | `PLAN_GYM_MISMATCH` |
| Missing user authentication | `UNAUTHENTICATED` | `UNAUTHENTICATED` |
| Role, gym scope, or workload denial | `PERMISSION_DENIED` | `FORBIDDEN` |
| Plans dependency unavailable | `UNAVAILABLE` | `PLANS_UNAVAILABLE` |

Do not leak internal exception text.

## Release decision

The relocation removes generated Member symbols and is a major source break. Existing `v2.0.0` is immutable and must not be rewritten.

Target release:

- source tag: `gym-proto v3.0.0`;
- Java: `com.gym.proto:gym-proto-java:3.0.0`;
- Go: `github.com/pploc/proto-go/v3`;
- Go package options use the `/v3` module prefix.

Protobuf API packages remain `member.v1` and `plans.v1`; repository release major and API package version have different purposes.

## Explicit non-goals

- No compatibility forwarding or duplicate ownership in Member.
- No schema backfill, dual write, rollback service, or data migration.
- No new common-library release unless validation proves a direct incompatibility.
- No real Payment implementation or discount execution.
- No deferred-service implementation.

## Verification

1. Run Buf format and lint.
2. Generate Java, Go, Go gRPC, and gateway output from one clean source tree.
3. Run declared breaking comparison against `v2.0.0`; accept only moved Member symbols.
4. Verify all Plans public routes and exact selectors.
5. Verify `GetActiveGym` and `ResolvePurchasablePlan` have no HTTP route.
6. Verify all generated Go packages use `/v3` and generated Java uses the 3.0.0 artifact.
7. Build generated Java and Go output and verify deterministic checksums.
8. Record source SHA, artifact checksums, break report, route matrix, and unresolved owner approval.

## G6 exit criteria

- Ownership, schema, API, exposure, workload identity, errors, and purchase correlation are unambiguous.
- Expected break report contains no unrelated changes.
- Immutable Java and Go artifacts resolve from the same approved source tag.
- Technical evidence passes.
- Human owner approval and publication remain pending until actually supplied.
