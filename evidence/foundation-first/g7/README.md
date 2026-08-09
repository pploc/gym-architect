# Phase 7 — `ms-gym-plans` evidence

## Status

G7 verification package for Plans as sole owner of gym locations and membership catalog.

## Local run

- Path: `local-2026-08-09/`
- Timestamp zone: Asia/Ho_Chi_Minh (`+07:00`)
- SHAs recorded in `local-2026-08-09/dependency-resolution.txt`

## Artifacts

| File | Proof |
|------|--------|
| `test-report.txt` | 60 tests, 0 fail/err; JaCoCo LINE 94.8% |
| `helm-render.yaml` | Shared `gym-service` chart render with Plans values |
| `networkpolicy.yaml` | Kong/Istio → HTTP 8080; Identifier/Member → gRPC 50051 |
| `schema-*.txt`, `flyway-history.txt` | Empty Postgres Flyway V1 + constraints/indexes |
| `health.json`, `liveness.json`, `readiness.json` | Local jar Actuator UP |
| `image-metadata.txt`, `image-*.json`, `image-run.log` | Runtime image from boot jar; readiness/liveness UP |
| `dependency-resolution.txt` | `common-java:2.0.2`, `gym-proto-java:3.0.0` pins |
| `no-messaging-static.txt` | No Plans Kafka/Redis/outbox/cache/scheduler config |
| `mtls-matrix.md` | Live mTLS allow/deny matrix |

## Commands

```bash
cd /home/phucl/Workplace/gapi/ms-gym-plans
./gradlew startEnv
./gradlew clean build
./gradlew stopEnv

cd /home/phucl/Workplace/gapi/gym-infra
helm template ms-gym-plans ./helm/gym-service \
  -f ./helm/gym-service/examples/ms-gym-plans-values.yaml -n gym-system
```

## Notes

- Multi-stage `Dockerfile` needs `GITHUB_TOKEN` for GH Packages (CI supplies it). Local image evidence used prebuilt `ms-gym-plans-1.0.0.jar` on `eclipse-temurin:26-jre`.
- `common-java:2.0.2` still ships Kafka client libraries; Plans has no Kafka config, topics, consumers, or producers.
