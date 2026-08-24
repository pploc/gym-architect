# Gym Chain Architecture

Architecture and implementation documentation for multi-location gym backend.

## Active implementation scope

Only these services are actionable now:

1. [`ms-gym-identifier`](services/01-ms-gym-identifier.md) — identity, authentication, and stable identity tokens.
2. [`ms-gym-member`](services/02-ms-gym-member.md) — member profiles, subscriptions, membership lifecycle and validation.
3. [`ms-gym-plans`](services/10-ms-gym-plans.md) — canonical gym locations, gym-specific plans and VND pricing.
4. [`ms-gym-checkin`](services/06-ms-gym-checkin.md) — G10 complete; AWS KMS-protected QR keys, display payloads, scans, records, and `checkin.recorded.v1` are proven in [`evidence/foundation-first/g10-final/README.md`](evidence/foundation-first/g10-final/README.md).

5. [`ms-gym-payment`](services/03-ms-gym-payment.md) — G11 implementation in progress: membership-only SePay `InitiatePayment`, verified webhook, outbox, and locked gate. G8 fake-payment remains producer until the locked pass.

Workout, Trainer, Promotion, Notification, and Analytics remain architecture catalog entries. Their service documents describe target boundaries, not current implementation work.

## Authoritative roadmap

Use [Foundation-First Platform Roadmap](plans/foundation-first/README.md). G0–G5 are completed historical foundation work. Current work starts at:

- [Phase 6 — Plans contracts](plans/foundation-first/06-plans-contracts.md)
- [Phase 7 — `ms-gym-plans`](plans/foundation-first/07-ms-gym-plans.md)
- [Phase 8 — Three-service integration](plans/foundation-first/08-three-service-integration.md)
- [Phase 9 — Stable identity, generated gateway, and OpenAPI 3.0](plans/foundation-first/09-kong-grpc-gateway-openapi.md)
- [Phase 10 — `ms-gym-checkin`](plans/foundation-first/10-ms-gym-checkin.md) — complete; see [`evidence/foundation-first/g10-final/README.md`](evidence/foundation-first/g10-final/README.md).
- [Phase 11 — Implement `ms-gym-payment` and locked gate](plans/foundation-first/11-payment-contracts.md) — in progress; no completion claim until locked evidence passes.

[`PLAN.md`](PLAN.md) is a catalog and historical dependency sketch, not an execution plan.

## Documentation map

- [Architecture overview](architecture/00-overview.md)
- [Shared libraries](architecture/02-shared-libraries.md)
- [Kafka event catalog](architecture/03-kafka-events.md)
- [Repository structure](architecture/04-project-structure.md)
- [DevOps library](architecture/05-devops-library.md)
- [Business flows](flows/01-business-flows.md)
- [Infrastructure](infrastructure/01-infrastructure.md)
- [Service specifications](services/)
- [Foundation evidence](evidence/foundation-first/)

## Ownership summary

| Data or behavior | Owner |
|---|---|
| Users, credentials, tokens | Identifier |
| Member profile and subscriptions | Member |
| Membership lifecycle and validation | Member |
| Gym locations | Plans |
| Membership plan catalog and VND price | Plans |
| QR root keys, display payloads, check-ins, and Check-in event | Check-in |
| Membership payment intents, SePay webhooks, and completed event | Payment (G11 implementation in progress) |

Cross-service IDs are opaque strings. No service creates a database foreign key to another service database.
