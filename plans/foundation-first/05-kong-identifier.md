# Phase 5 — Kong and `ms-gym-identifier`

## Objective

Reach G5 by implementing an executable Kong trust boundary, building Identifier on stable foundations, and proving register/login/refresh/membership/event behavior end to end.

## Prerequisites

- [Phase 0](00-contract-freeze.md) passed G0.
- [Phase 1](01-gym-proto.md) passed G1.
- [Phase 3](03-foundation-release.md) passed G4 with stable common releases.
- [Phase 4](04-ms-gym-member.md) passed and Member exposes the internal user-ID membership RPC.
- Read [roadmap trust/adoption rules](README.md).

Kong fixture work may begin after G0 and overlap foundation implementation. Identifier core design may begin behind fakes after G0. Production adapters and release pin stable G4 artifacts.

## In-scope repositories

- `gym-infra`
- new `ms-gym-identifier`
- `ms-gym-member` only for live internal-RPC/E2E validation
- platform docs and evidence manifest

## Explicit non-goals

- No rollout to all remaining services before G5.
- No Kafka framing, retry, or auth parsing reimplemented locally.
- No user-role values for service identities.
- No cross-service DB foreign key for `gym_id`.
- No production private key committed.

---

## Part A — Executable Kong fixture environment

### A1. Add runnable configuration

Add a DB-less local topology under `gym-infra`, for example:

```text
kong/kong.yml
kong/docker-compose.yml
kong/plugins/gym-jwt-claims/
kong/tests/
kong/fixtures/
```

Include:

- pinned Kong image/version;
- fixture public key;
- test-only fixture signer/private key;
- mock HTTP upstream that records path/method/headers;
- health checks and deterministic startup/teardown;
- declarative configuration validation.

Do not use documentation snippets as evidence.

### A2. Route model

External routes target service-local HTTP gateway listeners:

```text
http://ms-gym-identifier:8080
http://ms-gym-member:8080
```

Keep any native `grpc://`/`grpcs://` routes separate and explicit. The internal `GetMembershipStatusByUserId` RPC has no Kong route.

Freeze an exact public/protected method/path matrix. Public routes still strip spoofed trusted headers and inject no user identity.

### A3. JWT and trusted-header policy

For protected requests:

1. Validate `RS256`, signature, issuer, audience, expiry, and known `kid`.
2. Reject algorithm confusion/downgrade.
3. Strip incoming:
   - `x-user-id`;
   - `x-user-role`;
   - `x-gym-id`;
   - `x-membership-status`;
   - any compatibility workload header.
4. Normalize and inject validated claims.
5. Reject unknown role/status.
6. Allow `NONE` on ordinary authenticated routes.
7. Reject `NONE`, missing, or malformed status for membership-gated routes.
8. Preserve valid `traceparent`/`tracestate`; use `x-trace-id` only as fallback.

### A4. Gateway-local tests

Before Identifier exists, prove with fixture tokens and mock upstream:

- valid token accepted;
- invalid signature, algorithm, issuer, audience, expiry, and `kid` rejected;
- current/previous keys both work during overlap;
- spoofed trusted headers removed/replaced;
- public routes carry no spoofed identity;
- `CUSTOMER` accepted and `MEMBER` rejected;
- `NONE` behavior matches route policy;
- exact HTTP path/method routing;
- internal membership RPC is unreachable externally;
- W3C precedence is preserved.

Store sanitized upstream captures as evidence.

## Part B — Create `ms-gym-identifier`

### B1. Repository baseline

Create the repository on `develop` with module path and release policy fixed before code generation. Pin exact stable:

- `github.com/pploc/common-go` G4 tag;
- `github.com/pploc/proto-go` G1 tag.

Release builds use no `replace` directive.

Recommended hexagonal layout:

```text
cmd/server/
internal/domain/
internal/usecase/
internal/usecase/port/
internal/adapter/grpc/
internal/adapter/gateway/
internal/adapter/postgres/
internal/adapter/redis/
internal/adapter/security/
internal/adapter/kafka/
internal/adapter/member/
internal/config/
migrations/
.github/workflows/
```

### B2. Ports and use cases

Define ports for:

- user repository;
- refresh-token repository;
- password hasher;
- access-token signer/key provider;
- Google identity verifier;
- Member membership-status client;
- transactional outbox repository/relay;
- Redis revocation/blacklist;
- clock and ID generator.

Implement:

- Register;
- Login;
- Google Login;
- Refresh;
- Logout;
- Get Current User;
- Change Password;
- Create Trainer;
- Suspend User;
- List Users.

Core can be tested with fakes before live Kong/Kafka, but production completion requires stable adapters.

### B3. Security rules

- Normalize email before uniqueness checks.
- Use bcrypt cost 12 unless a measured platform policy changes it.
- Return generic invalid-credential messages.
- Public registration creates only `CUSTOMER`.
- Generate cryptographically random refresh tokens.
- Store only SHA-256 refresh-token hashes.
- Rotate refresh tokens transactionally.
- Detect reused revoked tokens and revoke the token family.
- Issue only the frozen JWT claims/profile.
- Never log passwords, Google tokens, JWTs, refresh tokens, or hashes.
- Store `gym_id` as an opaque external identifier, not an FK.
- Define how registration validates an allowed gym through a service API/reference policy; do not join Member’s database.

### B4. Database and outbox

Create migrations for:

- `users`;
- `refresh_tokens`;
- `outbox_events`.

Registration transaction:

1. Validate input.
2. Hash password.
3. Insert user.
4. Insert `identity.user.registered.v1` outbox record with immutable UUID.
5. Commit.

Suspension transaction:

1. Mark user suspended.
2. Revoke refresh tokens/families.
3. Insert `identity.user.suspended.v1` outbox record.
4. Commit.

Relay through stable `common-go`:

- concrete Protobuf value;
- outbox UUID as `event-id`;
- user ID as ordering key;
- canonical headers and `.v1` topic;
- mark published only after broker acknowledgement;
- retain rows on failure;
- no DB/Kafka exactly-once claim.

### B5. Membership-aware token issuance

For `CUSTOMER` login/refresh:

1. Call `GetMembershipStatusByUserId` over the verified workload channel.
2. Authorize as Identifier workload, not a user role.
3. Embed `NONE`, `ACTIVE`, `PAUSED`, or `EXPIRED`.
4. Fail availability rather than guess when Member is unavailable.

For non-customer roles, use `membership_status=NONE` unless the frozen contract says otherwise.

### B6. Servers and policies

Start:

- native gRPC on `50051`;
- gRPC-Gateway HTTP on `8080`;
- health/readiness endpoint;
- metrics endpoint according to platform convention.

Install the stable common-go chain and explicit policies:

- Register/Login/Google Login/Refresh: public;
- Logout/GetCurrentUser/ChangePassword: authenticated;
- CreateTrainer/SuspendUser/ListUsers: privileged role policy;
- no method public by omission.

## Part C — Live interoperability

Run through real Kong, Identifier, Member, PostgreSQL, Redis, Kafka, and Schema Registry:

1. Register through Kong.
2. Verify user and outbox rows commit atomically.
3. Verify `identity.user.registered.v1` is published with correct key/headers/frame.
4. Verify Member creates one shell and handles duplicate delivery.
5. Login through Kong and inspect sanitized upstream headers.
6. Send spoofed role/gym/status headers and verify replacement.
7. Refresh and verify internal membership lookup by user ID.
8. Activate membership through controlled fixture flow.
9. Refresh again and verify JWT/Kong status changes from `NONE` to `ACTIVE`.
10. Rotate signing key and verify overlap/retirement timing.
11. Logout/revoke and prove expected Kong/Identifier rejection behavior.
12. Suspend user, revoke refresh access, and publish/consume `identity.user.suspended.v1`.
13. Verify invalid, expired, revoked, wrong-issuer/audience/algorithm tokens fail.
14. Verify outages retain outbox rows and do not create partial registration loss.

## Part D — Open controlled service adoption

After G5:

1. Update the compatibility matrix with Identifier and Kong evidence.
2. Record real owner approvals if supplied; otherwise leave governance pending.
3. Open remaining service adoption one service at a time.
4. Require each service to pin stable artifacts and pass service-specific contract tests.
5. Do not allow service-local shared transport/auth forks.

## Verification commands

### Kong/infrastructure

```bash
cd /home/phucl/Workplace/gapi/gym-infra
helm lint helm/gym-service
helm lint helm/gym-infra
helm template test-service helm/gym-service
helm template test-infra helm/gym-infra
docker compose -f kong/docker-compose.yml config
deck file validate kong/kong.yml
```

Run the proxy fixture suite and sanitize captured headers before storing evidence.

### Identifier

```bash
cd /home/phucl/Workplace/gapi/ms-gym-identifier
gofmt -l .
go vet ./...
staticcheck ./...
go test -race ./...
go test -tags=integration -race ./...
govulncheck ./...
```

Also run migrations on disposable PostgreSQL and live outbox/Kong integration.

## Evidence produced

- Kong declarative validation and proxy test report.
- sanitized `kong-upstream-capture`.
- Identifier unit/race/static/vulnerability report.
- migration and repository integration report.
- outbox/Kafka/Registry report.
- membership refresh/workload-auth report.
- signing-key rotation report.
- full `identifier-e2e-report` with exact artifact versions/SHAs.

## G5 exit criteria

- Executable Kong configuration enforces the frozen JWT/header contract.
- Identifier pins stable G1/G4 foundations without local substitutions.
- Registration and suspension use transactional outboxes.
- Identifier and Member use the protected user-ID membership RPC.
- Register, login, refresh, membership transition, rotation, logout/revocation, and suspension pass end to end.
- Kafka events use only the final Protobuf `.v1` contract.
- Outages preserve data and retry safely.
- Evidence records exact versions and sanitized results.
- Remaining service adoption opens only through controlled stable-version migrations.
- Human approvals are recorded only when actually provided.
