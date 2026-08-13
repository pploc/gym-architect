# Phase 9 Stage 3 — Kong runtime transcode compatibility

Evidence package for Stage 3 of `docs/plans/foundation-first/09-kong-grpc-gateway-openapi.md`.

Historical G8, Stage 0, Stage 1, and Stage 2 evidence remain unchanged.

## Status

| Item | Value |
|---|---|
| Stage | Phase 9 Stage 3 |
| Measured | 2026-08-14 |
| Browser path | Kong JWT/CORS/exact routes → mTLS generated Go `grpc-gateway` → mTLS Member/Plans gRPC |
| Kong candidate | `kong:3.8-ubuntu` — `kong@sha256:250cc9745fde8ce04be060bc8e4338dfc280f54986cb38ef24d03876cb6b6e2f` |
| Result | **PASS** — all 27 browser operations and fallback compatibility gates passed |
| Generated gateway | local G9 image `sha256:69dd7c1ee7b5d69dd2d0d8312d39c2973e7e88b046119341e9345e1608551f8a` |
| Publication | **not run** — no v6.0.0 tag, Java `6.0.0` package, Go `v1.6.0` tag, consumer pin, or deployment artifact |
| Plans MVC | retained on `:8080`; not Kong business upstream |

## Historical decision gate

Kong's bundled `grpc-gateway` source-Protobuf parser remains incompatible. It lazily fails a routed Member/Plans request before upstream forwarding:

```text
buf/validate/validate.proto:535:9: field name expected
```

That historical failure selected generated Go `grpc-gateway`. No source-Protobuf surrogate, downgrade, Lua parser patch, Kong body rewrite, or Kong error rewrite was added.

## Measured generated-gateway result

`./kong/run-g9.sh` passed:

- Helm lint and exact Member/Plans NetworkPolicy assertions;
- Kong JWT plugin fixture tests;
- generated HTTP/OpenAPI/bundle verification and checksum;
- generated gateway build plus focused Go tests;
- exact 27-route render: Identity 12, Member 7, Plans 8;
- all 27 HTTPS/JSON browser operations, purchase/payment/subscription replacement flow, JWT matrix, CORS, and exact-route negatives;
- Kong-to-gateway mTLS, gateway-to-service mTLS, direct Kong-to-public-service denial, gateway-to-workload-service denial, wrong peer/SAN TLS denial, and no host-published gateway port;
- forged trusted header and `Grpc-Metadata-*` rejection;
- real browser-visible `x-error-code` promotion with CORS exposure.

`kong/g9-observed-errors.yaml` contains sanitized measured status, content type, body byte count/checksum, merged header/trailer metadata, and CORS results. Observed service trailers promoted as HTTP headers were `VALIDATION_FAILED` (400), `ACCESS_DENIED` (403), and `PLAN_NOT_FOUND` (404). Invalid JSON and invalid query binding returned standard grpc-gateway 400 responses with no synthetic domain error code. Deterministic 500 and 503 fixture inputs were unavailable and remain explicitly skipped.

## Boundaries

- Generated gateway is transport-only: generated binding registration, default Protobuf JSON, mTLS, vetted metadata reconstruction, and default grpc-gateway errors.
- Kong remains JWT/CORS/exact-route owner.
- Gateway forwards only single nonblank `x-user-id`, `x-user-role`, `traceparent`, and `tracestate`; generic HTTP headers and `Grpc-Metadata-*` do not reach gRPC metadata.
- G8 remains unchanged.
- No private key, JWT, fixture identifier, release binary, or raw runtime log is retained as evidence.

See `local-2026-08-13/` for historical parser failure commands and result. Passing fallback measurement is recorded above and in the generated ignored fixture observation.
