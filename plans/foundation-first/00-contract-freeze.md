# Phase 0 — Freeze Cross-Repository Contracts

> **Historical contract note:** This phase froze the original selected-gym JWT and Identifier-to-Member boundary. [Phase 9 Stage 0](09-kong-grpc-gateway-openapi.md#stage-0--replace-selected-gym-jwt-state-before-gateway-generation) supersedes those targets with stable identity, explicit gym request context, no Identifier-to-Member edge, and Member/Plans public gRPC behind Kong. Preserve G0 evidence unchanged.

## Objective

Reach G0 by making the API, identity, Kafka, ingress, and JWT contracts unambiguous before shared transport implementations change.

## Prerequisites

- Read [roadmap rules and gates](README.md).
- Recheck repository SHAs and working trees.
- Do not alter generated artifacts or publish releases in this phase.
- Do not mark owner approval unless accountable owners provide it.

## In-scope repositories

- `gym-proto`
- `docs`
- Relevant ADRs in `common-go` and `common-java`

## Explicit non-goals

- No Kafka producer/consumer implementation.
- No Kong installation.
- No Identifier repository creation.
- No service dependency upgrades.
- No JSON-topic migration design; Kafka has not been deployed.

## 1. Canonical identity vocabulary

Record these exact end-user roles:

```text
CUSTOMER
TRAINER
ADMIN
SUPER_ADMIN
```

Actions:

1. Replace `MEMBER` in fixtures, shared-library tests, and docs with `CUSTOMER`.
2. Define public registration as `CUSTOMER` only.
3. Reject client requests that attempt to create `TRAINER`, `ADMIN`, or `SUPER_ADMIN` through public registration.
4. Provision elevated roles only through protected administration or controlled out-of-band procedures.
5. Keep workload identities out of `x-user-role`.

Record these membership statuses:

```text
NONE
ACTIVE
PAUSED
EXPIRED
```

Policy:

- Ordinary authenticated methods accept any known status.
- Membership-gated methods require `ACTIVE`.
- Missing, blank, malformed, conflicting, or unknown status fails closed when required.
- New customers and non-customer roles use `NONE`.

Update representative contract and documentation files:

- `gym-proto/contracts/v1/auth/trusted-headers.json`
- `gym-proto/contracts/v1/compatibility.json`
- `common-go/docs/adr/0002-auth-and-tracing-contract.md`
- `docs/services/01-ms-gym-identifier.md`
- `docs/services/02-ms-gym-member.md`
- `docs/architecture/02-shared-libraries.md`

## 2. Trusted headers and tracing

Canonical trusted headers:

```text
x-user-id
x-user-role
x-gym-id
x-membership-status
x-trace-id       # compatibility correlation only
```

Rules:

1. Kong strips client-supplied copies before injecting validated claims.
2. Downstream libraries trim/normalize and reject conflicting duplicates.
3. Valid W3C `traceparent`/`tracestate` takes precedence.
4. `x-trace-id` is fallback correlation only and does not fabricate an OpenTelemetry parent.
5. Claims do not establish trust unless the request passed through the authenticated boundary or verified workload channel.

## 3. Workload identity

Use a separate internal identity mechanism for Identifier-to-Member calls.

Recommended initial contract:

- mTLS between workloads;
- certificate identity/SAN identifies `ms-gym-identifier`;
- the internal RPC authorizes that verified peer;
- NetworkPolicy permits only intended callers;
- metadata such as `x-service-id`, if retained for observability, is never accepted without binding to the verified peer.

Document that internal workload identity is not a user role and is not injected by public clients.

## 4. Kafka contract

Define the first and only deployed topic generation:

| Event | Topic | Subject |
|---|---|---|
| `UserRegisteredEvent` | `identity.user.registered.v1` | `identity.user.registered.v1-value` |
| `UserSuspendedEvent` | `identity.user.suspended.v1` | `identity.user.suspended.v1-value` |
| `UserRoleChangedEvent` | `identity.user.role-changed.v1` | `identity.user.role-changed.v1-value` |
| `PaymentCompletedEvent` | `payment.completed.v1` | `payment.completed.v1-value` |
| `MembershipActivatedEvent` | `membership.activated.v1` | `membership.activated.v1-value` |
| `MembershipPausedEvent` | `membership.paused.v1` | `membership.paused.v1-value` |
| `MembershipResumedEvent` | `membership.resumed.v1` | `membership.resumed.v1-value` |
| `MembershipExpiringSoonEvent` | `membership.expiring-soon.v1` | `membership.expiring-soon.v1-value` |
| `MembershipExpiredEvent` | `membership.expired.v1` | `membership.expired.v1-value` |

Additional rules:

- Subject naming: `TopicNameStrategy`.
- Compatibility: `BACKWARD`.
- Production: `auto.register.schemas=false`.
- DLQ: `{topic}.DLQ`.
- Initial Member group: `ms-gym-member-v1`.
- Ordering key is the domain entity key.
- Value is a Confluent-framed concrete Protobuf message.
- No envelope wrapper.
- Kafka delivery is at-least-once.

Canonical headers:

```text
event-type
source
timestamp
event-id
traceparent
tracestate (optional)
```

Preserve `x-trace-id` only as a compatibility fallback. New producers emit no legacy `x-event-*` headers.

Because no cluster was deployed, remove legacy JSON/envelope behavior rather than adding migration adapters, dual topics, or offset conversion.

## 5. Identifier-to-Member API

Add a new internal API design:

```protobuf
rpc GetMembershipStatusByUserId(GetMembershipStatusByUserIdRequest)
    returns (MembershipResponse);

message GetMembershipStatusByUserIdRequest {
  string user_id = 1;
}
```

Contract rules:

1. Keep existing `GetMembershipStatus(member_id)` unchanged.
2. Do not expose the new RPC over gRPC-Gateway or Kong.
3. Require Identifier workload identity.
4. Return `NONE` when a known user has no subscription.
5. Return availability failure when Member cannot answer; Identifier must not guess.
6. Keep `user_id`, `member_id`, and `gym_id` as separate opaque identifiers.

## 6. Ingress model

Freeze two listeners per externally exposed gRPC service:

- native internal gRPC: `50051`;
- service-local gRPC-Gateway HTTP/JSON: `8080`.

Kong external routes target HTTP `8080`. Native gRPC routes, if introduced, are separate explicit routes. Do not assume Kong path routing to `grpc://...:50051` performs REST transcoding.

Decide one source of truth for HTTP mappings. Wire existing `*_http.yaml` files into `buf.gen.yaml` through `grpc_api_configuration`, or consolidate them into one referenced service config. Internal RPCs stay unmapped.

## 7. JWT contract

Freeze:

| Property | Contract |
|---|---|
| Algorithm | `RS256` |
| Issuer | `gym-identifier` |
| Audience | `gym-api` |
| Required claims | `sub`, `iss`, `aud`, `iat`, `exp`, `jti`, `kid` |
| Application claims | `role`, `gym_id`, `membership_status` |
| Public methods | Register, Login, Google Login, Refresh |
| Protected methods | Logout and all other methods unless explicitly public |
| Rotation | Current and previous public keys overlap for at least max access-token TTL |
| Tests | Committed fixture public key and test-only signer/private key |
| Production keys | Secret-manager supplied; never committed |

Kong validates algorithm, signature, issuer, audience, expiry, and key ID before claim injection.

## 8. Resolve documentation contradictions

Update or supersede instructions that still describe:

- unversioned topics;
- JSON envelopes;
- Viper/local JWT validation in common libraries;
- generated Go stubs copied into services;
- HTTP paths routed directly to raw gRPC without explicit transcoding;
- Kong only after all business services;
- `MEMBER` role;
- Identifier calling a member-ID RPC with a user ID.

At minimum inspect:

- `docs/PLAN.md`
- `docs/architecture/00-overview.md`
- `docs/architecture/02-shared-libraries.md`
- `docs/architecture/03-kafka-events.md`
- `docs/architecture/04-project-structure.md`
- `docs/infrastructure/01-infrastructure.md`
- `docs/services/01-ms-gym-identifier.md`
- `docs/services/02-ms-gym-member.md`
- current shared-library plans and ADRs

## Verification

1. Search all tracked contracts/docs for `MEMBER`, unversioned target topics, `x-event-`, JSON envelope defaults, and the invalid refresh lookup.
2. Confirm all remaining matches are historical incident notes or explicitly deprecated context.
3. Generate a contract diff and have it reviewed across Go, Java, Protobuf, Schema Registry, and gateway scopes.
4. Run Buf format/lint as a preview; generation/release occurs in Phase 1.

## Evidence produced

See [Phase 0 contract-freeze evidence](00-contract-freeze-evidence.md) for:

- `identity-vocabulary` matrix;
- Kafka topic/subject/header matrix;
- public/protected/internal route matrix;
- JWT profile and rotation rules;
- reviewed Protobuf/API design diff;
- repository snapshot and verification results;
- list of unresolved human approvals.

## G0 exit criteria

- One canonical role and membership vocabulary exists.
- All nine initial Kafka topics/subjects are fixed.
- Internal membership lookup and workload-auth mechanism are fixed.
- Ingress and JWT behavior are unambiguous.
- Contradictory active documentation is removed or explicitly superseded.
- No implementation depends on an unresolved choice.
- Approval status remains truthful.
