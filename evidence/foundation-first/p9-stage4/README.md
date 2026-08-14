# Phase 9 Stage 4 — historical v6.0.0 contract and Plans MVC removal

## Supersession

This document records prior v6.0.0 local fixture results. It is not final G9 evidence because proof used mutable/dirty source and v6.0.0 lacks canonical merged OpenAPI. See [Phase 9 final evidence](../p9-final/README.md) for final clean-source requirements.

## Historical release lock

| Item | Value |
|---|---|
| Contract source | `gym-proto v6.0.0` at `80e41f2ffe074df13866458d04b63d2263d00c3c` |
| Java stubs | `com.gym.proto:gym-proto-java:6.0.0` |
| Go stubs | `github.com/pploc/proto-go v1.6.0` |
| Release | `https://github.com/pploc/gym-proto/releases/tag/v6.0.0` |
| Runtime lock | `gym-infra/kong/g9-release-lock.json` |
| Kong image | `kong:3.8-ubuntu@sha256:250cc9745fde8ce04be060bc8e4338dfc280f54986cb38ef24d03876cb6b6e2f` |
| Browser routes | 27 exact routes: Identity 12, Member 7, Plans 8 |

`run-g9.sh` verifies source SHA, release asset SHA-256 values, pinned consumer artifacts, route/template checksums, and Kong image digest before fixture startup. It downloads released OpenAPI and Kong bundle assets rather than using mutable artifacts.

## Runtime result

The release-lock G9 fixture passed before and after Plans MVC removal:

- browser HTTPS/JSON reached Plans through Kong, generated Go grpc-gateway, and Plans mTLS gRPC;
- all 27 public operations, JWT/trusted-header negatives, CORS, mTLS/SAN matrices, workload isolation, Helm, NetworkPolicy, and browser-safe deterministic `500`/`503` cases passed;
- exact deterministic browser-safe errors were `500 {"code":13, "message":"Internal server error", "details":[]}` with exposed `x-error-code: INTERNAL`, and `503 {"code":14, "message":"Upstream service unavailable", "details":[]}` with no `x-error-code`;
- Kong 3.8 source-Protobuf parser failure remains historical evidence; generated gateway is selected runtime.

## Plans MVC removal

Deleted native Plans business MVC controllers, trusted-header filter, role interceptor, HTTP exception advice, web configuration, and their HTTP-only tests. Removed `spring-boot-starter-webmvc-test`.

Retained:

- `PlansGrpcHandler`, mTLS/SAN policy, application/domain/JPA Specification behavior, and service gRPC tests;
- Spring web runtime, Actuator, Prometheus, `server.port: 8080`, Docker/Helm health checks, and `startEnv`/`stopEnv` Gradle tasks;
- generated gateway as only browser JSON representation of Plans business gRPC.

`g9-business-check.sh` now asserts `http://ms-gym-plans:8080/api/v1/gyms` returns `404`, while liveness and readiness probes return `200`, before executing Kong Plans CRUD checks.

## Verification

| Gate | Result |
|---|---|
| `ms-gym-plans ./gradlew clean check --no-daemon` after deletion | PASS |
| Released-lock `gym-infra/kong/run-g9.sh` after deletion | PASS |
| Identifier `go test -race ./...` | PASS |
| Generated gateway `go list -m` and `go test ./...` | PASS (`proto-go v1.6.0`) |
| Member `./gradlew clean check --no-daemon` | PASS |
| Released Plans OpenAPI `openapi-typescript` generation plus `tsc --noEmit` | PASS |
| Final root `make test-all` | Not available: no workspace-root `Makefile` |

No JWTs, private keys, fixture IDs, PII, or raw unsanitized logs are stored here. Pre-existing dirty paths remain disclosed separately until commit authorization: `ms-gym-identifier/Dockerfile`, `gym-infra/.claude/`, and `gym-infra/kong/__pycache__/`.
