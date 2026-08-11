# Phase 9 Stage 2 — Plans gRPC transport trust

Evidence package for Stage 2 of `docs/plans/foundation-first/09-kong-grpc-gateway-openapi.md`.

## Status

| Item | Value |
|---|---|
| Stage | Phase 9 Stage 2 |
| Prepared | 2026-08-12 |
| Plans base | `54bc02c2bd153145b8fe116bb948593b50639ccc` |
| Plans dependencies | `com.gym:common-java:2.1.1`; `com.gym.proto:gym-proto-java:5.0.0` |
| Contract candidate | `gym-proto` Stage 1 candidate remains unreleased |
| Publication | **not run** — no tag, package, consumer pin, or release created |
| Stage 3 readiness | **blocked** pending real Kong 3.8 upstream mTLS, error/header/body, and CORS measurements |

## Transport matrix

| Plans RPC group | Allowed immediate mTLS peer |
|---|---|
| Eight claim-bearing catalog CRUD RPCs | Kong DNS `kong` or approved Kong SPIFFE URI SAN |
| `PlansService/GetActiveGym` | Identifier DNS/default SPIFFE/gym-system SPIFFE SAN only |
| `PlansService/ResolvePurchasablePlan` | Member DNS/default SPIFFE/gym-system SPIFFE SAN only |
| Unknown or unclassified method | Shared `GrpcMethodRegistry` / `AuthServerInterceptor` rejects |

Plans now accepts `x-user-id` and `x-user-role` on claim-bearing gRPC RPCs only after the immediate peer passes the Plans-local Kong SAN gate. Shared `AuthServerInterceptor` still owns claim validation, role authorization, conflicting-header rejection, and security context installation. Workload identity remains separate and is proven only by the exact RPC-to-SAN allowlist.

## Local certificates

The ignored local generator creates disposable DNS/SPIFFE client identities for Kong, Identifier, Member, Check-in, Notification, and Postman. Evidence records no private key, P12 password, certificate body, token, or production credential.

## Verification

See `local-2026-08-12/` for reference revisions, executed commands, and outcomes.

## Explicit boundaries

- No `common-java` refactor or release.
- No `gym-proto` v6 publication or consumer pin.
- No `gym-infra` Kong route, transcoding, CORS, upstream certificate, Helm, or NetworkPolicy change.
- No measured Kong HTTP error/header/body behavior claim.
- No Plans MVC removal or `:8080` business-route behavior change.
- No private key, release binary, or generated certificate committed.
- G8, Stage 0, and Stage 1 evidence remain unchanged.
