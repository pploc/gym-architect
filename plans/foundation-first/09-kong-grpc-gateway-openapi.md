# Phase 9 — Stable Identity, Generated Gateway, and OpenAPI 3.0

> **Status:** Complete on 2026-08-15. Immutable v6.0.1 artifacts, locked clean-source G9, protected CI, sanitized final evidence, and recorded clean product trees passed. See [final evidence](../../evidence/foundation-first/p9-final/README.md).

## Selected architecture

```text
Browser HTTPS/JSON
  -> Kong JWT/CORS/exact routes
  -> mTLS generated Go grpc-gateway
  -> mTLS Member and Plans gRPC :50051
```

Kong remains the only browser endpoint. It proxies Identity to its native HTTP gateway on `8080`, validates JWTs, strips client-supplied trusted headers, injects verified identity/role metadata, and applies CORS and exact route policy.

Generated Go `grpc-gateway` is transport-only. It owns generated binding registration, Protobuf JSON conversion, mTLS, vetted trusted-metadata reconstruction, and grpc-gateway error handling. It owns no authorization decisions, business orchestration, persistence, DTOs, or workload routes.

Member and Plans accept end-user metadata only from generated gateway identity `ms-gym-api-gateway`. Kong may reach only gateway `8443`; it cannot directly reach Member or Plans public gRPC methods on `50051`.

## Historical rejected design

Kong 3.8 source-Protobuf `grpc-gateway` parsing was evaluated and rejected. It failed on a routed request with:

```text
buf/validate/validate.proto:535:9: field name expected
```

Old references to Kong parsing the released Protobuf bundle, mounting source `.proto` files for runtime transcode, or presenting Kong SAN to Member/Plans public RPCs are historical only. Do not restore this path, add a Lua parser/body rewrite, or treat the bundle as a generated-gateway runtime dependency.

## Public contract

`gym-proto v6.0.1` is the pending patch release. It keeps the active API unchanged and adds a canonical generated browser document:

```text
openapi/gym-active-api.openapi.yaml
```

Generation uses `github.com/google/gnostic/cmd/protoc-gen-openapi@v0.7.1` for Identity, Member, and Plans service documents, then `scripts/merge-openapi.py` performs deterministic collision-rejecting merge. It is not `protoc-gen-openapiv3`.

The canonical document has exactly 27 operations:

- Identity: 12 native HTTP operations;
- Member: 7 generated-gateway operations;
- Plans: 8 generated-gateway operations.

Workload-only RPCs remain unannotated and absent from Kong and OpenAPI:

- `MemberService.ValidateMembership`
- `MemberService.ListMembersByStatus`
- `PlansService.GetActiveGym`
- `PlansService.ResolvePurchasablePlan`

Backend and native gRPC clients consume generated Protobuf artifacts. Browser clients consume released canonical OpenAPI. Do not handwrite parallel REST schemas or backend DTOs from OpenAPI.

## Identity and authorization

JWTs contain only:

```text
sub, role, iss, aud, iat, exp, jti, kid
```

They never contain `gym_id` or `membership_status`. Browser gym choice is explicit request or path context, never authorization proof. Until staff-to-gym assignment has an owner, persistence model, revocation behavior, and tests, all gym-wide mutations remain `SUPER_ADMIN` only.

Kong injects only verified `x-user-id` and `x-user-role`. Generated gateway accepts them only from Kong SAN, strips arbitrary inbound metadata, and forwards only vetted values. Public services reject direct callers, forged headers, wrong SANs, and generated-gateway identity on workload RPCs.

## Transport and deployment boundaries

| Path | Allowed peer | Port |
|---|---|---:|
| Kong to generated gateway | Kong mTLS identity | `8443` |
| Generated gateway to Member public RPCs | `ms-gym-api-gateway` | `50051` |
| Generated gateway to Plans public RPCs | `ms-gym-api-gateway` | `50051` |
| Identifier to Plans `GetActiveGym` | Identifier identity | `50051` |
| Member to Plans `ResolvePurchasablePlan` | Member identity | `50051` |
| Plans Actuator/probes/metrics | approved health observers | `8080` / `9090` |

Plans has no native business HTTP adapter. `http://ms-gym-plans:8080/api/v1/gyms` must return `404`; Actuator health remains available on `8080`.

NetworkPolicies must model these paths separately. No broad peer rule may grant Kong direct service access or grant gateway workload methods.

## Error contract

Generated gateway must expose deterministic safe errors:

```json
{"code":13,"message":"Internal server error","details":[]}
```

for `500`, and:

```json
{"code":14,"message":"Upstream service unavailable","details":[]}
```

for `503`. No raw service transport details, exception messages, Protobuf payloads, tokens, or personal data may enter browser responses or committed evidence.

## Release lock and clean-source execution

`gym-infra/kong/g9-release-lock.json` is the authoritative runtime lock. Final form must pin:

- repository URL and detached commit for `gym-proto`, Identifier, Member, and Plans;
- v6.0.1 Java and Go contract artifacts, all release asset checksums, and canonical OpenAPI checksum;
- fake-payment contract pin and vendor identity;
- generated-gateway source and immutable image digest;
- Kong image digest, route manifest/template checksums, and redacted rendered-config checksum.

`run-g9.sh` uses `materialize-g9.py` to clone locked sources into a temporary `0700` workspace, detached-checkout each SHA, and pass only those paths to Compose. It must not read adjacent repositories or mutable branch heads. Temporary source, certificates, rendered config, credentials, and fixture data are removed on all exits.

The generated gateway must be built and published before its digest is added in later lock commit. G9 then runs the digest-pinned image rather than rebuilding local gateway source.

## Required gates

1. Publish immutable `gym-proto v6.0.1`, Java `6.0.1`, and Go `v1.6.1`, including canonical OpenAPI and checksums. Never mutate v6.0.0.
2. Update Identifier, Member, Plans, generated gateway, and fake-payment to released patch artifacts. Refresh fake-payment vendor tree because it builds with `-mod=vendor`.
3. Commit every runtime-affecting change, including Identifier Dockerfile secret handling. Prove no dirty source influenced G9.
4. Complete lock validation: detached sources, release assets, gateway identity/image digest, contract pins, route count, and redacted render checksum.
5. Run `run-g9.sh` from a fresh clean infrastructure checkout with no sibling repositories present.
6. Run protected authenticated CI with private repository/package read credentials. Build private Go dependencies only through BuildKit secrets; never use build arguments or persisted credentials.
7. Generate TypeScript types from released canonical OpenAPI and run `tsc --noEmit`.
8. Validate sanitized final evidence under `docs/evidence/foundation-first/p9-final/`. Reject keys, JWTs, authorization values, PII, fixture identifiers, raw logs, and raw stack/transport details.
9. Record exact repository SHAs, versions, checksums, certificate public metadata, command labels, exit codes, and CI/release URLs.
10. Verify every recorded repository has a clean tracked tree.

`make test-all` is not a G9 requirement: workspace root is not a repository and has no committed root `Makefile`. Use per-repository checks plus locked authenticated G9 CI.

## Completion rule

Do not mark Phase 9 complete because a local runtime fixture passes. Completion requires every required gate above, immutable released inputs, a passing isolated clean-source G9 run, protected CI, validated sanitized evidence, and clean committed source trees.

## Non-goals

- No BFF, public streaming API, direct browser service access, or public reflection.
- No custom Kong parser, Lua error rewrite, handwritten OpenAPI schema, or OpenAPI-derived backend DTO.
- No new staff-to-gym authorization model.
- No production key material in Git; use mounted secrets or secret manager.
