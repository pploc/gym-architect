# Phase 9 final evidence

> **Status:** Complete on 2026-08-15. Final proof uses immutable released artifacts, detached locked sources, protected CI, and sanitized evidence only.

## Release and lock

| Item | Value |
|---|---|
| Contract source | `gym-proto v6.0.1` at `7f887b99503353b3c6a3066361ff5b36d44b6811` |
| Contract release | <https://github.com/pploc/gym-proto/releases/tag/v6.0.1> |
| Java package | `com.gym.proto:gym-proto-java:6.0.1` |
| Go module | `github.com/pploc/proto-go@v1.6.1`, commit `db0fb4090f426602eacc1cb61348f178d07f684c` |
| Canonical OpenAPI | `gym-active-api.openapi.yaml`, SHA-256 `e44d585c59b25b6962b86810083a3d0cc53e05a0c4618cdfee8f539fc4276fa3` |
| Runtime lock | [`gym-infra@4aa5f289f526b3fc2bb22790fe6e0dc2b49fe243`](https://github.com/pploc/gym-infra/commit/4aa5f289f526b3fc2bb22790fe6e0dc2b49fe243), `g9-release-lock.json` SHA-256 `dfd886e708e617f2122a603e2642704641a3c3311a537664591c41d18e8264b8` |
| Generated gateway image | `ghcr.io/pploc/ms-gym-api-gateway@sha256:b3cb6ff8be5b0d760b6b527ca267c90d1596eaa499a4c16ee147725943c8f21d` |
| Kong image | `kong:3.8-ubuntu@sha256:250cc9745fde8ce04be060bc8e4338dfc280f54986cb38ef24d03876cb6b6e2f` |
| Java generation template | `kong/g9-java.buf.gen.yaml`, SHA-256 `c96b65231ccc762d3d5ee25ea33cb0301c92ea13252554c1107e3181fcd4d5b6` |

The lock materializes detached repositories, verifies released assets and consumer versions, then runs only its pinned images. Generated Java sources exist only in a disposable Compose volume; locked source is never changed.

## Protected gate and sanitized runtime evidence

| Gate | Result | Record |
|---|---|---|
| Contract release publication | PASS | [gym-proto run 31824643941](https://github.com/pploc/gym-proto/actions/runs/31824643941) |
| Infrastructure static gate | PASS | [gym-infra run 31872354856](https://github.com/pploc/gym-infra/actions/runs/31872354856) |
| Authenticated locked G9 | PASS | [gym-infra run 31872359141](https://github.com/pploc/gym-infra/actions/runs/31872359141) |
| Sanitized G9 artifact validation | PASS | artifact `g9-sanitized-4aa5f289f526b3fc2bb22790fe6e0dc2b49fe243`; SHA-256 `c55717f747ae63e7239f4b1d0d9cb3685cb11253336c5c403f4fa70b09bf3baf` |

The protected gate passed locked fixture startup, Java-stub generation, schema seeding, all compatibility cases, and sanitized-evidence upload. Private failure diagnostics were skipped because no failure occurred.

Sanitized artifact schema validation confirms 13 named security/error/compatibility cases. All expected HTTP statuses matched actual statuses; browser-safe `500` and `503` passed; no case contains internal exception text. It records checksums only: no request values, response bodies, JWTs, Authorization values, private keys, PII, fixture IDs, raw logs, raw Protobuf payloads, stack traces, or transport details.

## Browser contract

Selected runtime topology:

```text
Browser HTTPS/JSON
  -> Kong JWT/CORS/exact routes
  -> mTLS generated Go grpc-gateway
  -> mTLS Member and Plans gRPC :50051
```

- 27 exact public operations: Identity 12, Member 7, Plans 8.
- Direct Plans business HTTP `:8080/api/v1/gyms` returns `404`; Actuator/probes remain available.
- Kong cannot directly reach Member or Plans public `:50051` methods.
- Deterministic safe responses passed: `500` has `Internal server error`; `503` has `Upstream service unavailable`.

## Consumer immutable workflow pins

| Repository | Recorded commit | CI |
|---|---|---|
| Identifier | [`def7936c9e54f489aad5aab246b9300697969d90`](https://github.com/pploc/ms-gym-identifier/commit/def7936c9e54f489aad5aab246b9300697969d90) | [run 31873144792](https://github.com/pploc/ms-gym-identifier/actions/runs/31873144792) PASS |
| Member | [`bd419bca04b0b0dc7da1baae94f15bb817c6ee3a`](https://github.com/pploc/ms-gym-member/commit/bd419bca04b0b0dc7da1baae94f15bb817c6ee3a) | [run 31872962844](https://github.com/pploc/ms-gym-member/actions/runs/31872962844) PASS |
| Plans | [`85ac3a9d8bf254bee390313be809bbe85574ff67`](https://github.com/pploc/ms-gym-plans/commit/85ac3a9d8bf254bee390313be809bbe85574ff67) | [run 31872966538](https://github.com/pploc/ms-gym-plans/actions/runs/31872966538) PASS |

Each reusable workflow reference pins `pploc/gym-infra` at `4aa5f289f526b3fc2bb22790fe6e0dc2b49fe243`; callers no longer pass reserved `github_token` reusable-workflow secrets. Identifier CI uses Go `1.26.6`, the upstream standard-library security patch required by `govulncheck`.

## Final tracked-tree check

Recorded product commits have no tracked modifications. Local ignored/untracked tooling is excluded from this claim: `gym-proto/scripts/__pycache__/` and `gym-infra/.claude/`.

Earlier Stage 3 and Stage 4 records remain historical. This document supersedes their final-proof status without altering their observations.
