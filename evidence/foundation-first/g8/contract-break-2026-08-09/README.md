# G8 contract-break staging (2026-08-09)

Status note for the in-place `*.v1` contract break after historical G8 (`local-2026-08-09/`).

## What changed

- **Artifacts (active targets):** Java `com.gym.proto:gym-proto-java:4.0.0`, Go module `github.com/pploc/proto-go` (no `/v4` path segment), `common-go` `0.4.0`, `common-java` `2.1.0`. Source packages remain `*.v1`.
- **RPC:** unique `<Rpc>Request` / `<Rpc>Response` per method; no shared `AuthResponse` / `MemberResponse` / `Empty`.
- **Enums:** closed vocabularies are prefixed proto enums on the wire; domain/JWT/DB keep short names.
- **Kafka:** topic names stay `.v1`; Schema Registry subjects overwrite in place (no dual `.v2` topics).
- **gym-infra:** `kong/fixtures/fake-payment` uses `proto-go` common/payment/events enums and publishes `payment.completed.v1` with `PaymentType`; `kong/g8-business-check.sh` expects wire enum JSON (`USER_STATUS_*`, `MEMBERSHIP_STATUS_*`, `PLAN_TYPE_*`, `GYM_LOCATION_STATUS_*`). JWT claim asserts remain short domain membership strings.

## Verification this cutover

| Check | Result |
|---|---|
| Member unit tests (`com.gym.member.unit.*`) | green (local composite) |
| Plans unit tests (`com.gym.plans.unit.*`) | green (local composite) |
| fake-payment `go build -mod=vendor` | green |
| Full `./kong/run-g8.sh` | **not re-run** — blocked until Identifier/Member/Plans images resolve published or staged v4/shared pins without permanent local replace/includeBuild |

## Preserved

- Historical G8 evidence under `../local-2026-08-09/` unchanged.
- G5 topology, runner, fixtures, and evidence untouched.
