# Phase 5 Part D — Controlled Service Adoption

## Status

G5 passed for Identifier-led Kong/Member interoperability. Part D records the matrix and controlled-adoption gate.

- Compatibility matrix: `compatibility-manifest.json`
- Owner approvals: **pending** (no human sign-off supplied this session)
- Services already on stable pins: `ms-gym-identifier`, `ms-gym-member`
- Remaining services open **one at a time** under the policy in the manifest

## Gate (do not skip)

1. Pin only released foundation artifacts (no RC/branch/`replace`/`mavenLocal` in release).
2. Pass service-specific contract tests against frozen JWT (`iss=gym-identifier`, `aud=gym-api`), trusted headers, and `.v1` Protobuf events.
3. Reuse `common-go` / `common-java` transport/auth — no service-local forks.
4. Record SHAs in this evidence tree when a service adopts.

## Next adoption order (suggested)

1. `ms-gym-payment` (produces `payment.completed.v1` already consumed by Member)
2. `ms-gym-checkin` / `ms-gym-workout` (high throughput, JWT consumer path)
3. remaining Java/Go services

## Governance

Leave `ownerApproval: pending` until product/platform owners explicitly approve production cutover.
