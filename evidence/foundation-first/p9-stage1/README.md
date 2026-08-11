# Phase 9 Stage 1 — inline HTTP mappings candidate

Evidence package for Stage 1 of `docs/plans/foundation-first/09-kong-grpc-gateway-openapi.md`.

Historical G8 and Stage 0 evidence are unchanged.

## Status

| Item | Value |
|---|---|
| Stage | Phase 9 Stage 1 |
| Prepared | 2026-08-11 |
| Source | `gym-proto` develop `59000df8efa312b50763c3b95694b46842d2f6df` with candidate working tree |
| Candidate target | gym-proto `v6.0.0`; Java `6.0.0`; Go `github.com/pploc/proto-go@v1.6.0` |
| Publication | **blocked** — no tag, Java package, Go tag, consumer pin, or release created |
| OpenAPI generator | `github.com/google/gnostic/cmd/protoc-gen-openapi@v0.7.1` commit `39aedcfe9f00c4336420c8ed927fb0f803f921f2` |
| OpenAPI candidates | `openapi/identity/v1/identity.openapi.yaml` (12); `openapi/member/v1/member.openapi.yaml` (7); `openapi/plans/v1/plans.openapi.yaml` (8) |
| Kong source bundle | `dist/kong-proto-6.0.0.tar.gz` |
| Release readiness | **false** — Kong 3.8 error/header/CORS measurements are pending Stage 3 |

## Candidate scope

- 27 public Identity, Member, and Plans RPCs own exact inline `google.api.http` bindings.
- Four workload RPCs remain unbound: `ValidateMembership`, `ListMembersByStatus`, `GetActiveGym`, and `ResolvePurchasablePlan`.
- Active service-local HTTP YAML mirrors are deleted; deferred services remain in `proto/http.yaml`.
- Gnostic produces direct lowerCamelCase Protobuf JSON schemas, query names, and URI placeholders, correct named `PurchaseMembershipBody`, string-safe `int64`, and symbolic enums.
- Exact upstream Gnostic `v0.7.1` OpenAPI annotation sources are vendored under `proto/openapiv3/`. Native annotations define documentation-only `BearerAuth`, protected-operation security, and safe success-schema examples with concrete enum values.
- Runtime `google.api.http` templates and grpc-gateway routes remain snake_case. Placeholder labels differ only in generated OpenAPI documentation; URI segment matching is unchanged.
- Gnostic emits three service-local documents directly with `output_mode=source_relative`; no post-generation normalizer, merger, or YAML rewrite exists.
- Public OpenAPI contracts live under tracked `openapi/`. `gen/` remains ignored because it contains disposable Go, Java, and grpc-gateway generated stubs.
- Kong source bundle is deterministic and Kong 3.8 parses Member and Plans roots.

## Explicitly out of scope

- No Stage 2 Plans SAN work.
- No Stage 3 G9 Kong route, mTLS, CORS, or runtime transcoding implementation.
- No Stage 4 Plans MVC removal.
- No invented Kong error body, status, header, trailer, or CORS behavior.
- No `v6.0.0` publication or consumer upgrade.

## Local artifacts

See `local-2026-08-11/` for commands, results, hashes, and status.
