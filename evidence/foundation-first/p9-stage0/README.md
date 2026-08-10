# Phase 9 Stage 0 — stable identity-only JWT / explicit gym context

Separate evidence package for Stage 0 of
`docs/plans/foundation-first/09-kong-grpc-gateway-openapi.md`.

Historical G8 under `../g8/` is **not rewritten**.

## Status

| Item | Value |
|---|---|
| Stage | Phase 9 Stage 0 |
| Prepared | 2026-08-10 |
| Publication | **complete** — workflow `31404520393` success |
| Source tag | `v5.0.0` @ `59000df8efa312b50763c3b95694b46842d2f6df` |
| Java | `com.gym.proto:gym-proto-java:5.0.0` (GitHub Packages) |
| Go | `github.com/pploc/proto-go@v1.5.0` commit `d88537703e2c603a56e96c21f5631c5cf12c87da` |
| Release | https://github.com/pploc/gym-proto/releases/tag/v5.0.0 |
| Stage 1 work | none (no `google.api.http` annotations / OpenAPI / Kong transcoding) |

## Local run

- Path: `local-2026-08-10/`
- Timestamp zone: Asia/Ho_Chi_Minh (`+07:00`)

## Scope completed

- Identity-only JWT claims; `gym_id` / `membership_status` forbidden on access tokens.
- `SelectGym` and `GetMembershipStatusByUserId` retired from contracts and service code.
- Member public gym RPCs take explicit request `gym_id`; purchase nested body.
- Plans mutations `SUPER_ADMIN`-only; catalog reads authenticated.
- Workload allowlists: Identifier→Plans `GetActiveGym`; Member→Plans `ResolvePurchasablePlan`; Check-in→Member `ValidateMembership`; Notification→Member `ListMembersByStatus`.
- Identifier no longer depends on Member.
- Kong strips forged trusted headers; injects only `x-user-id` / `x-user-role`.
- Active Member NetworkPolicy drops Identifier→Member edge.
- Immutable contract release published; Identifier/Member/Plans adopt published artifacts.

## Artifacts in `local-2026-08-10/`

| File | Proof |
|---|---|
| `status.txt` | Gate summary, published SHAs, consumer pins |
| `commands.txt` | Commands executed and observed results |
| `results.md` | Failure-gate checklist with pass notes |
| `repo-shas.txt` | Pre-publish dirty HEADs + published artifact SHAs |
| `contract-files.sha256` | Checksums of key contract manifests |
| `generated.sha256` | Deterministic local `buf generate` file digests |
| `fixture.sha256` | Local fixture digest |
| `published-*.sha256` / `published-release-report.json` | Release asset digests from GH release |
| `blocked.txt` | Remaining optional items (not publication blockers) |

## Consumer pins after publication

- Identifier: `github.com/pploc/proto-go v1.5.0` (no local replace)
- Member/Plans: `com.gym.proto:gym-proto-java:5.0.0` from GitHub Packages
