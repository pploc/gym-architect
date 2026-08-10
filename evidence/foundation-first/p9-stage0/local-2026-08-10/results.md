# Phase 9 Stage 0 failure gates

Plan gates from `09-kong-grpc-gateway-openapi.md` Stage 0.

| Gate | Result | Evidence |
|---|---|---|
| Access/refresh tokens contain no `gym_id` or `membership_status` | PASS (unit) | Identifier JWT tests; `contracts/v1/jwt-profile.json` forbids both |
| `/api/v1/auth/gym` absent from active routes | PASS (static) | `routes.json` / Identity protected set; historical G8 fixtures still mention route and are not rewritten |
| Identifier has no Member runtime dependency | PASS | Member client deleted; NetworkPolicy Identifier→Member edge removed from Member values |
| `GetMembershipStatusByUserId` absent from stubs and registry | PASS | `verify-retired-symbols.py`; generated digests; Member handler registry |
| Every gym-specific public Member method validates explicit `gym_id` | PASS | Member mapping + unit/integration tests via full `check` |
| Purchase rejects missing/inactive/mismatched gym/plan | PASS | Plans `ResolvePurchasablePlan` + Member purchase path tests |
| Caller-chosen gym grants no ADMIN power | PASS | Member/Plans SUPER_ADMIN-only mutations; no ADMIN self-bypass |
| SUPER_ADMIN behavior explicit | PASS | Trainer create, ListMembers, Plans mutations SUPER_ADMIN-only |
| Identifier→Plans trainer validation retained | PASS | Identifier Plans client retained; allowlist GetActiveGym |
| Tests use `given_when_then` names | PASS | New/changed tests follow convention |
| Kong injects identity/role only; strips forged gym/membership | PASS | Live Kong suite; `handler.lua` injects only `x-user-id`/`x-user-role` |
| No Stage 1 annotations/OpenAPI/transcoding | PASS | No `google.api.http` in proto sources |
| Buf format/lint | PASS | exit 0 |
| Declared break vs `v4.0.0` | PASS | SelectGym / GetMembershipStatusByUserId / PurchaseMembershipRequest changes present |
| Deterministic generation | PASS | regenerated stubs + aggregate sha256 recorded |
| Member `./gradlew spotlessCheck clean check` | PASS | against published `gym-proto-java:5.0.0` |
| Plans `./gradlew spotlessCheck check` | PASS | against published `gym-proto-java:5.0.0` |
| Identifier `go test` / `vet` / build | PASS | against published `proto-go@v1.5.0` (no replace) |
| Helm lint | PASS | gym-service chart |
| Confluent fixture regenerate + full gym-proto `check` | PASS | clean Registry; fixtures sha256 339824c9…; CI + local |
| Live Kong gateway tests | PASS | `go test ./...` under kong/tests |
| Live G5 | NOT RUN | optional; needs package tokens + full compose build |
| Immutable publication of `v5.0.0` / Java `5.0.0` / Go `v1.5.0` | PASS | workflow 31404520393; release + GH Packages + proto-go tag |

## Publication notes

- First tag attempt failed: non-executable retired-symbol script (exit 126). Fixed via `python3` invocation.
- Second tag attempt failed: fixture subject-count asserted total Registry subjects (import protos inflate count to 12). Fixed to per-case subject existence + BACKWARD.
- Final tag `v5.0.0` → `59000df8efa312b50763c3b95694b46842d2f6df`.

## Consumer pins after publication

- Identifier `go.mod`: `github.com/pploc/proto-go v1.5.0`, replace removed.
- Member/Plans: `implementation 'com.gym.proto:gym-proto-java:5.0.0'` resolves from GitHub Packages.
