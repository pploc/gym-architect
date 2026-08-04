# Phase 5 Part A — Kong Fixture Evidence

Timestamp (UTC): `2026-08-04T15:58:58Z`

## Versions

| Component | Value |
|---|---|
| Kong image | `kong:3.8-ubuntu` |
| gym-infra baseline SHA | `f24d64fe571185d322e6c3d8c8cb26970a58866f` |
| gym-infra worktree | uncommitted `kong/` tree (this Part A artifact) |
| Plugin | `gym-jwt-claims` 1.0.0 |
| JWT profile | `gym-proto/contracts/v1/jwt-profile.json` (RS256, iss=`gym-identifier`, aud=`gym-api`) |
| Trusted headers | `gym-proto/contracts/v1/auth/trusted-headers.json` |

## Layout

```text
gym-infra/kong/
  kong.yml
  docker-compose.yml
  mock-upstream/{main.go,go.mod,Dockerfile}
  certs/fixture_rsa.{key,pub}
  certs/fixture_rsa_prev.{key,pub}
  plugins/gym-jwt-claims/{handler.lua,schema.lua}
  tests/{go.mod,go.sum,gateway_test.go}
  fixtures/kong-upstream-capture.json
```

## Validation commands

```bash
docker compose -f gym-infra/kong/docker-compose.yml config
docker run --rm \
  -e KONG_DATABASE=off \
  -e KONG_PLUGINS=bundled,gym-jwt-claims \
  -e 'KONG_LUA_PACKAGE_PATH=/opt/?.lua;/opt/?/init.lua;;' \
  -v "$PWD/gym-infra/kong/kong.yml:/kong.yml" \
  -v "$PWD/gym-infra/kong/plugins/gym-jwt-claims:/opt/kong/plugins/gym-jwt-claims" \
  kong:3.8-ubuntu kong config parse /kong.yml
# => parse successful

docker compose -f gym-infra/kong/docker-compose.yml up -d --build
cd gym-infra/kong/tests && go test -count=1 ./...
# ok github.com/pploc/gym-infra/kong/tests
```

## Route matrix (fixture)

| Path | Class | Notes |
|---|---|---|
| `/v1/auth/register` | public | strip trusted headers, no identity inject |
| `/v1/auth/login` | public | same |
| `/v1/auth/google` | public | same |
| `/v1/auth/refresh` | public | same |
| `/v1/auth/logout` | protected | JWT required |
| `/v1/auth/me` | protected | JWT required |
| `/v1/auth/change-password` | protected | JWT required |
| `/v1/admin/*` | protected | JWT required |
| `/v1/members/me` | protected | ordinary auth accepts `NONE` |
| `/v1/memberships/booking` | membership-gated | requires `ACTIVE` |
| `member.v1.MemberService/GetMembershipStatusByUserId` | internal only | no Kong route (404) |

## Policy proven by tests

- valid RS256 token accepted; claims injected as `x-user-id` / `x-user-role` / `x-gym-id` / `x-membership-status`
- invalid signature, issuer, audience, expiry, unknown `kid` rejected (401)
- algorithm `none` rejected (401)
- current + previous fixture keys accepted during overlap
- spoofed trusted headers stripped on public and protected routes
- public routes inject no user identity
- `CUSTOMER` accepted; `MEMBER` rejected (403)
- ordinary protected routes accept `membership_status=NONE`
- membership-gated routes reject `NONE`/`EXPIRED`, accept `ACTIVE`
- W3C `traceparent` preserved to upstream
- internal membership RPC unreachable externally

## Sanitized upstream capture

See `gym-infra/kong/fixtures/kong-upstream-capture.json`.

Notable headers after Kong processing of a spoofed request:

```text
X-User-Id: user-evidence          # from JWT sub, not spoofed
X-User-Role: CUSTOMER             # from JWT role, not SUPER_ADMIN spoof
X-Membership-Status: NONE         # from JWT, not ACTIVE spoof
Traceparent: 00-4bf92f...         # preserved
Authorization: Bearer <redacted>
User-Agent: Go-http-client/1.1
```

## Notes

- Fixture private keys are test-only and committed under `kong/certs/` as allowed by `jwt-profile.json` testing contract.
- Production private keys must never be committed.
- `deck` not installed in this environment; declarative validation used `kong config parse` instead.
- Helm lint/template commands from the phase doc exercise existing `gym-infra/helm` charts; they are independent of this Kong fixture tree.
