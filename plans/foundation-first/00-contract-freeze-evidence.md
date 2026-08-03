# Phase 0 Contract Freeze Evidence

- **Prepared:** 2026-08-03
- **Gate:** G0 — Contract frozen
- **Approval status:** Pending accountable owner review
- **Generated artifacts changed:** No
- **Releases published:** No

## Repository snapshot

| Repository | Branch | SHA at phase start | Working tree at phase start |
|---|---|---|---|
| `gym-proto` | `develop` | `660f324740c823bbcd8e2702daa343afa933ea66` | Clean |
| `docs` | `develop` | `592310f6b17a0f4f3e907c3042b248e8e412dd15` | Existing staged plan/IDE files and nine modified documentation files |
| `common-go` | `develop` | `ba08285baa031eb61022c7ecf8b423840926cb7e` | Existing staged `.idea` files |
| `common-java` | `develop` | `f2863ef4baba3b088ca84b39116cea547aa35240` | Clean |

Existing worktree changes were preserved and not reset. Some in-scope docs already
contained user changes before this phase; the contract edits were applied on top.

## Identity vocabulary matrix

| Dimension | Frozen contract |
|---|---|
| End-user roles | `CUSTOMER`, `TRAINER`, `ADMIN`, `SUPER_ADMIN` |
| Public registration | Creates `CUSTOMER` only; requests for elevated roles are rejected |
| Elevated role provisioning | Protected administration or controlled out-of-band procedure |
| Workload identity | Separate from user role; never represented in `x-user-role` |
| Membership statuses | `NONE`, `ACTIVE`, `PAUSED`, `EXPIRED` |
| Default membership | New customers and non-customer roles use `NONE` |
| Ordinary authenticated method | Accepts any known status |
| Membership-gated method | Requires `ACTIVE` |
| Required-status failure | Missing, blank, malformed, conflicting, or unknown fails closed |

## Trusted-header and workload matrix

| Contract | Decision |
|---|---|
| Trusted headers | `x-user-id`, `x-user-role`, `x-gym-id`, `x-membership-status`, `x-trace-id` |
| Boundary behavior | Kong strips client copies before injecting validated claims |
| Downstream behavior | Trim/normalize; reject conflicting duplicates |
| Trace precedence | Valid `traceparent`/`tracestate` wins |
| Compatibility trace | `x-trace-id` is correlation fallback only and creates no OTel parent |
| Identifier → Member | mTLS; certificate SAN identifies `ms-gym-identifier` |
| Internal authorization | Member authorizes verified peer; metadata alone establishes no trust |
| Network boundary | NetworkPolicy permits intended callers on native gRPC `50051` |

## Kafka topic, subject, and header matrix

| Event | Topic | Subject | Ordering key |
|---|---|---|---|
| `UserRegisteredEvent` | `identity.user.registered.v1` | `identity.user.registered.v1-value` | `user_id` |
| `UserSuspendedEvent` | `identity.user.suspended.v1` | `identity.user.suspended.v1-value` | `user_id` |
| `UserRoleChangedEvent` | `identity.user.role-changed.v1` | `identity.user.role-changed.v1-value` | `user_id` |
| `PaymentCompletedEvent` | `payment.completed.v1` | `payment.completed.v1-value` | domain entity key (`user_id` in current schema) |
| `MembershipActivatedEvent` | `membership.activated.v1` | `membership.activated.v1-value` | `member_id` |
| `MembershipPausedEvent` | `membership.paused.v1` | `membership.paused.v1-value` | `member_id` |
| `MembershipResumedEvent` | `membership.resumed.v1` | `membership.resumed.v1-value` | `member_id` |
| `MembershipExpiringSoonEvent` | `membership.expiring-soon.v1` | `membership.expiring-soon.v1-value` | `member_id` |
| `MembershipExpiredEvent` | `membership.expired.v1` | `membership.expired.v1-value` | `member_id` |

| Property | Frozen contract |
|---|---|
| Value | Confluent-framed concrete Protobuf message; no envelope |
| Subject strategy | `TopicNameStrategy` |
| Compatibility | `BACKWARD` |
| Production schema registration | `auto.register.schemas=false` |
| Required headers | `event-type`, `source`, `timestamp`, `event-id`, `traceparent` |
| Optional header | `tracestate` |
| Compatibility fallback | `x-trace-id` read-only when W3C context is unavailable |
| Legacy `x-event-*` | New producers do not emit; no migration adapter because Kafka is greenfield |
| Delivery | At-least-once |
| DLQ | `{topic}.DLQ` |
| Initial Member group | `ms-gym-member-v1` |

## Public, protected, and internal route matrix

| Surface | Methods | Listener / trust |
|---|---|---|
| Public Identity | Register, Login, Google Login, Refresh | Kong → HTTP/JSON `8080`; no end-user JWT required |
| Protected Identity | Logout and all remaining methods unless explicitly public | Kong validates JWT then targets HTTP/JSON `8080` |
| Public Member HTTP mappings | Only methods present in `member_http.yaml` | Kong → HTTP/JSON `8080` |
| Internal membership lookup | `GetMembershipStatusByUserId` | Native gRPC `50051`, no HTTP mapping, verified Identifier mTLS identity |
| Other internal Member RPCs | `ValidateMembership`, `ListMembersByStatus` | Native gRPC `50051`, no HTTP mapping, authorized workload channel |

The existing `GetMembershipStatus(member_id)` contract remains unchanged. The new
lookup accepts an opaque `user_id`, returns `NONE` for a known user without a
subscription, and returns availability failure when Member cannot answer.
Identifier does not guess. `user_id`, `member_id`, and `gym_id` remain distinct.

## JWT profile and rotation rules

| Property | Contract |
|---|---|
| Algorithm | `RS256` |
| Issuer | `gym-identifier` |
| Audience | `gym-api` |
| Required claims | `sub`, `iss`, `aud`, `iat`, `exp`, `jti`, `kid` |
| Application claims | `role`, `gym_id`, `membership_status` |
| Public methods | Register, Login, Google Login, Refresh |
| Protected methods | Logout and all others unless explicitly public |
| Gateway checks | Algorithm, signature, issuer, audience, expiry, key ID |
| Rotation | Current and previous public keys overlap for at least max access-token TTL |
| Tests | Committed fixture public key and test-only signer/private key |
| Production | Secret-manager supplied keys; no committed production private key |

## Protobuf/API design diff reviewed

Phase 0 freezes this additive Phase 1 source change without modifying generated
artifacts:

```diff
 service MemberService {
   rpc GetMembershipStatus(GetMembershipStatusRequest) returns (MembershipResponse);
+  // Internal, workload-authenticated, and intentionally unmapped from HTTP.
+  rpc GetMembershipStatusByUserId(GetMembershipStatusByUserIdRequest)
+      returns (MembershipResponse);
 }
+
+message GetMembershipStatusByUserIdRequest {
+  string user_id = 1;
+}
```

HTTP mapping source of truth is the existing `proto/*/v1/*_http.yaml` Google API
service configuration, to be wired into `buf.gen.yaml` via
`grpc_api_configuration` in Phase 1. `generate_unbound_methods` must be removed
so internal RPCs remain unmapped. Generation and release remain Phase 1 work.

## Canonical machine-readable artifacts

- `gym-proto/contracts/v1/auth/trusted-headers.json`
- `gym-proto/contracts/v1/compatibility.json`
- `gym-proto/contracts/v1/jwt-profile.json`
- `gym-proto/contracts/v1/routes.json`
- `gym-proto/contracts/v1/kafka/wire-format.json`
- `gym-proto/contracts/v1/dlq/retry-and-dlq.json`
- `gym-proto/contracts/v1/manifest.json`

## Verification results

| Check | Result |
|---|---|
| Contract JSON syntax | Pass (`python3 -m json.tool` for all changed/new contract JSON) |
| Diff whitespace | Pass (`git diff --check` in all four repositories) |
| Unversioned frozen topics in active docs | Pass; no matches outside superseded/historical plan context |
| Invalid `GetMembershipStatus(user_id)` refresh lookup | Pass; replaced by `GetMembershipStatusByUserId(user_id)` in active docs |
| `MEMBER` / JSON-envelope contradiction scan | Remaining matches explicitly describe removal, rejection, or verification criteria |
| Buf lint | Pass when run from `gym-proto` after the format preview |
| Buf format preview | Not clean: existing files `analytics.proto`, `common.proto`, `payment.proto`, and `promotion.proto` need comment-spacing formatting; Phase 0 did not alter them |
| Generated diff | Design diff reviewed only; generation intentionally deferred to Phase 1 |

## Unresolved human approvals

The following approvals remain **pending** and have not been inferred from source
changes or test results:

- Go owner
- Java owner
- Protobuf owner
- Kafka / Schema Registry owner
- Gateway owner

Technical implementation and review can proceed, but owner-approved status and
production promotion cannot be claimed until accountable owners record approval.
