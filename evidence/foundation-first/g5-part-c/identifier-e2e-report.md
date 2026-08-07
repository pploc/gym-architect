# Phase 5 Part C — Live Interoperability Evidence

> Current reproducible evidence: `gym-infra/kong/run-g5.sh` on 2026-08-07 passed Identifier-led business checks through real Kong, Identifier, Member mTLS, PostgreSQL, Redis, Kafka, and Schema Registry. Member external HTTP routes remain intentionally absent. The table below retains the earlier host-dependent run as historical detail.

Timestamp (UTC): see `versions.json`.

## Topology exercised

| Component | Where |
|---|---|
| Kong proxy | `gym-kong` `:8000` (declarative, DB-less) |
| Identifier HTTP/gRPC | host `:8082` / `:50052` binary `/tmp/ms-gym-identifier` |
| Member HTTP/gRPC | host `:8080` / `:50051` `bootRun` HEAD |
| identity PostgreSQL | `gym-identifier-postgres` host `:5433` / `identity_db` |
| member PostgreSQL | `gym-member-postgres` host `:5432` / `gym_member` |
| Kafka + Schema Registry | `gym-identifier-kafka` `:9092`, SR `:8081` |
| Redis blacklist | `gym-kong-redis` host `:6380` (shared by Identifier logout + Kong plugin) |
| Kong→Identifier reachability | WSL veth `192.168.143.2:8082` (not `host.docker.internal` on this host) |

Exact SHAs/versions: `versions.json`.

## Part C checklist

| # | Step | Result | Evidence |
|---|---|---|---|
| 1 | Register through Kong | **PASS** `200` `PENDING_VERIFICATION` | `results.jsonl` C1 |
| 2 | User + outbox commit atomic | **PASS** user row + outbox row same user key | C2 |
| 3 | `identity.user.registered.v1` published | **PASS** outbox → `PUBLISHED` | C3 |
| 4 | Member one shell; no duplicate profile | **PASS** one `members` row | C4 |
| 5 | Login through Kong (post-verify) gym-neutral JWT | **PASS** `membership_status=NONE`, no `gym_id` | C5 |
| 6 | Spoofed `x-user-*` headers replaced | **PASS** `/users/me` returns real subject, not attacker | C6 |
| 7 | Refresh rotates; still gym-neutral | **PASS** `200`, claims `NONE` | C7 |
| 8 | Activate membership (payment fixture) | **PASS** gym A subscription `ACTIVE` | C8 |
| 9 | SelectGym → JWT `ACTIVE` for selected gym; dual gym B independent | **PASS** A then B both `ACTIVE` | C9 / C9b |
| 10 | Previous signing key overlap | **PASS** Kong accepts `kid=previous` (upstream 500 only because synthetic sub missing) | C10 |
| 11 | Logout + Redis blacklist rejection | **PASS** logout `200`; me `401 Token has been revoked` | C11 |
| 12 | Suspend user + outbox + login denied | **PASS** suspend `200`; login `403 account is suspended`; suspended outbox `PUBLISHED` | C12 |
| 13 | Invalid / wrong iss/aud / expired / alg none | **PASS** all `401` | C13 |
| 14 | Kafka outage retains outbox; recovers | **PASS** register while Kafka stopped → `PUBLISHING` retained; after restart → `PUBLISHED`; user not lost | C14 |
| — | Internal membership RPC via Kong | **PASS** `404` (no external route) | C_internal_rpc |

Raw line log: `results.jsonl` (no passwords/tokens stored).

## Notes

- VerifyEmail still uses opaque user-id token (`ponytail` until real email sender).
- Payment service not in stack; activation used controlled `payment.completed.v1` publisher (`/tmp/paypub`) via common-go Confluent Protobuf framing.
- Identifier trusts Kong-injected headers for authenticated HTTP (direct JWT-only without headers returns `401 authentication is required`); production path is always behind Kong.
- After Part C, `kong.yml` restored to `mock-upstream` so gateway unit suite stays green.

## Gateway unit suite (post-restore)

```text
cd gym-infra/kong/tests && go test -count=1 ./...
# ok github.com/pploc/gym-infra/kong/tests
```
