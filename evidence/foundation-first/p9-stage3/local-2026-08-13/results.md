# Phase 9 Stage 3 runtime compatibility results

## Historical direct Kong source-parser result — 2026-08-13

| Gate | Result | Evidence |
|---|---|---|
| Helm lint and Member/Plans NetworkPolicy assertions | PASS | `./kong/run-g9.sh` completed before fixture startup |
| Existing Kong plugin fixture | PASS | `go test -v ./kong/tests/...` completed before G9 startup |
| Candidate HTTP/OpenAPI and source-bundle verification | PASS | `verifyHttpConfig`, `verifyOpenApi`, `packageKongProto`, `verifyKongProto`, and bundle checksum completed |
| Kong 3.8 startup and source mounts | PASS | G9 Kong became healthy with additive `buf` and `google/api` mounts |
| Identifier HTTPS/JSON flow | PASS through super-admin login | Real G9 browser flow reached Plans request |
| Plans routed source-proto parse | FAIL | `POST /api/v1/gyms` returned HTTP `500` before upstream gRPC forwarding |
| Member routed source-proto parse | FAIL | Focused Kong request probe reproduced same source-parser failure |
| Root cause observed | FAIL | Kong bundled Lua parser rejects `buf/validate/validate.proto:535:9` with `field name expected` |
| Generated Go `grpc-gateway` fallback | SELECTED | Required JSON binding cannot use Kong 3.8 bundled source parser |

```text
POST /api/v1/gyms returned HTTP 500
[grpc-gateway] ... protoc.lua:230: buf/validate/validate.proto:535:9: field name expected
```

## Generated gateway fallback result — 2026-08-14

| Gate | Result | Evidence |
|---|---|---|
| Gateway focused Go tests | PASS | staged generated bindings; `go test ./...` |
| Member gateway SAN interceptor test | PASS | `./gradlew test --tests com.gym.member.unit.config.KongIdentityServerInterceptorUnitTest --no-daemon` |
| Plans gateway SAN interceptor test | PASS | `./gradlew test --tests com.gym.plans.unit.config.KongIdentityServerInterceptorUnitTest --no-daemon` |
| Plans live mTLS transport matrix | PASS | `./gradlew test --tests com.gym.plans.integration.PlansMtlsWorkloadIntegrationTest --no-daemon` |
| Helm lint and exact Member/Plans policy assertions | PASS | `./kong/run-g9.sh` |
| Existing Kong JWT/plugin fixture | PASS | `./kong/run-g9.sh` |
| HTTP/OpenAPI and candidate bundle verification | PASS | `./kong/run-g9.sh` |
| Generated gateway Compose build and health | PASS | `./kong/run-g9.sh` |
| Route ownership | PASS | 27 exact routes: Identity 12, Member 7, Plans 8 |
| Browser business flow | PASS | all 27 operations plus purchase/payment/subscription replacement |
| Gateway mTLS | PASS | Kong identity reaches gateway; wrong SAN and wrong CA identities fail; port `8443` is not host-published |
| Service public-peer trust | PASS | direct Kong certificate is denied on Member and Plans public RPCs; generated gateway certificate supplies public trust identity |
| Workload isolation | PASS | gateway certificate is denied from all Member/Plans workload RPCs; approved workload SANs retain exact method access |
| Header boundary | PASS | forged trusted headers and `Grpc-Metadata-X-User-Role` do not elevate browser requests |
| Upstream TLS target verification | PASS | swapped Member/Plans server names fail TLS validation |
| Exact route negatives | PASS | raw RPC, reflection, workload, wrong-method, retired, and collision paths are Kong `404` |
| Compatibility errors/CORS | PASS | `kong/g9-observed-errors.yaml` measured standard grpc-gateway bodies and browser-visible real `x-error-code` values |
| Deterministic 500/503 cases | SKIPPED | fixture inputs not provided; no behavior asserted |
| Release/publication | NOT RUN | no v6.0.0/Java 6.0.0/Go v1.6.0 publication, tag, consumer pin, or deployment artifact |
| Plans MVC removal | NOT RUN | `:8080` retained; Kong does not use it as business upstream |

Sanitized compatibility observations:

```text
400 plans validation: x-error-code=VALIDATION_FAILED
403 Member authorization: x-error-code=ACCESS_DENIED
404 Plans lookup: x-error-code=PLAN_NOT_FOUND
400 invalid JSON/query: no x-error-code
```

Gateway module lock SHA-256: `5eb5c31278b741b5bb2fa24200c94ed53416fd8ac52becaa4b5921f9037b1e75`.

The fallback supersedes only failed runtime transcoding. Historical direct Kong parser evidence remains valid. Stage 4 remains out of scope.
