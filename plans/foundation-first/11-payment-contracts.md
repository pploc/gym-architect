# Phase 11 — Payment Contracts (G11)

> **Mode:** docs + contract freezes only. No `ms-gym-payment` scaffold, service code, Helm/image, or active Kong Payment route.
> **Shape:** mirror [Phase 6](06-plans-contracts.md): ownership freeze, domain, RPCs/exposure, correlation, errors, release decision, non-goals, verification, and exit criteria.
> **Provider decision:** **SePay-first** (VietQR / bank-transfer webhook). `SEPAY` is the only G11 provider. Momo, ZaloPay, and VNPay remain deferred vocabulary, not active V1 producers.

## Context

G10 Check-in is complete. Production Payment remains catalog-only: local `ms-gym-payment/` is absent and `git@github.com:pploc/ms-gym-payment.git` is empty. Member already depends on `InitiatePayment` and `payment.completed.v1`; G8 proved that boundary with `gym-infra/kong/fixtures/fake-payment`.

G11 opens Payment contracts before service implementation. Preserve G0–G10 evidence and the existing Member/G8 wire behavior.

## Objective

Freeze Payment ownership, SePay-first provider policy, InitiatePayment, webhook, event, and Member compatibility contracts before cloning or implementing `ms-gym-payment`.

## Prerequisites

- G10 Check-in is complete.
- Current released artifacts remain `gym-proto` v7.0.2, `gym-proto-java` 7.0.2, `proto-go` v1.7.1, `common-java` 3.0.0, and `common-go` v0.5.0.
- G8 proved Member `PurchaseMembership` → `InitiatePayment` → completion → `payment.completed.v1` → Member activation.
- No customer or production data exists. Do not add migration, backfill, compatibility forwarding, or dual-write machinery.

## Ownership freeze

| Boundary | Owner | Contract |
|---|---|---|
| Users and JWTs | Identifier | Unchanged; Payment receives opaque `user_id`. |
| Purchase orchestration | Member | Resolve Plans terms; call Payment with trusted amount and `reference_id=purchase_id`. |
| Payment intent and provider session | Payment | Own `PENDING` → `COMPLETED` / `FAILED` / `REFUNDED`. |
| Provider webhook trust | Payment | Verify SePay signature over the raw body; no JWT. |
| Membership activation | Member | Consume `payment.completed.v1` only for `MEMBERSHIP`. |
| Catalog and price | Plans | Payment never prices or re-resolves membership plans. |
| Discount execution | Promotion | Deferred; Member rejects nonblank discount codes until this boundary opens. |
| Completed Payment event | Payment | Future production producer; G8 fake remains current fixture producer. |

Cross-service IDs are opaque strings. Payment creates no foreign key to Member, Plans, or Identifier databases.

## Payment V1 domain

- Status: `PENDING`, `COMPLETED`, `FAILED`, `REFUNDED`.
- G11 producer type: `MEMBERSHIP`. `TRAINER_BOOKING` remains schema-present but inactive.
- Provider: `SEPAY` only.
- Currency: VND; `amount_vnd` is non-negative `int64`.
- Membership idempotency key: `(payment_type=MEMBERSHIP, reference_id=purchase_id)`.
- Existing `PENDING` or `COMPLETED` intent returns the same `payment_id` and `payment_url`.
- A `FAILED` attempt requires a new Member `purchase_id`; Payment never creates a second charge for one membership purchase.
- Payment uses the Member-provided frozen amount and does not call Plans during completion.

## Canonical RPCs and exposure

| RPC | Caller | HTTP in G11 | Policy |
|---|---|---|---|
| `InitiatePayment` | Member workload over mTLS | None | Create or reuse intent; return `payment_id` and `payment_url`. |
| `GetPaymentStatus` | Deferred public client | Deferred | No active route. |
| `GetSpendingHistory` | Deferred public client | Deferred | No active route. |
| `RefundPayment` | Deferred admin | None | Later implementation phase. |
| `GetRevenueReport` | Deferred admin | None | Later implementation phase. |
| `GetPaymentsByUser` | Deferred internal caller | None | Later implementation phase. |

Only `InitiatePayment` is an active cross-service dependency for G11. Payment public methods remain in deferred `payment_http.yaml`; they are not promoted into active inline `google.api.http`, generated OpenAPI, or Kong routes.

### Compatibility invariants

Keep unchanged:

- `InitiatePaymentRequest`: `gym_id`, `payment_type`, `reference_id`, `provider`, `discount_code`, `user_id`, `amount_vnd`.
- Member sends `reference_id=purchase_id` and rejects nonblank `discount_code` until Promotion exists.
- `InitiatePaymentResponse.payment_id` and `.payment_url`; `payment_url` is an opaque SePay redirect or QR presentation URL.
- `PaymentCompletedEvent`, `payment.completed.v1`, its `user_id` key, subject, Confluent framing, and required headers.
- Member completion validation of payment ID, user, gym, type, provider, amount, and purchase state.
- G8 fake-payment `InitiatePayment`, `/complete`, replay, and idempotency semantics.

Do not add raw QR fields in G11. Additive QR fields require a later product decision and a new immutable contract release if needed.

## SePay webhook contract

This is a contracts-level boundary; no provider SDK or network call is part of G11.

- Endpoint: `POST /api/v1/payments/webhook/sepay`.
- Transport: native Spring MVC REST, not gRPC-Gateway.
- Authentication: verify SePay signature headers over the raw request body and reject stale timestamps. Webhooks have no JWT.
- Match only inbound transfers (`transferType=in`) to the Payment intent using the provider order/code mapping; require the received amount to be at least the frozen payment amount.
- A valid duplicate returns HTTP 200 `{"success":true}` without a second completion event or charge.
- A valid first completion transitions the intent to `COMPLETED` and publishes `payment.completed.v1` with `provider=SEPAY`.
- Orphan webhook persistence and alerting remain implementation-phase work; the later service must return HTTP 200 after valid authentication to avoid provider retry storms.

The exact SePay field mapping (`code`, `transferType`, `transferAmount`, `content`, `referenceCode`) and canonical HMAC string-to-sign require vendor verification during implementation.

## Kafka contracts

| Topic | G11 status |
|---|---|
| `payment.completed.v1` | **Frozen active**; already present in wire-format and fixtures. |
| `payment.failed.v1` | Name freeze only; no wire-format activation or Member consumer. |
| `payment.refunded.v1` | Name freeze only; no wire-format activation or Member consumer. |
| `{topic}.DLQ` | Existing common-library retry/DLQ rule remains unchanged. |

The catalog's historical unversioned `payment.failed` and `payment.refunded` names are superseded by the `.v1` names above. No producer, Registry subject, or consumer is activated for them in G11.

`payment.completed.v1` remains keyed by `user_id` and carries the existing `PaymentCompletedEvent` fields. For membership completion, `reference_id` resolves a pending Member-owned `purchase_id`; Member activates from frozen terms and never rereads Plans.

## Errors

Use canonical gRPC status and stable `x-error-code` through `common-java`:

| Condition | Status | Error code |
|---|---|---|
| Invalid field, amount, provider, or type | `INVALID_ARGUMENT` | `INVALID_ARGUMENT` |
| Missing payment or reference | `NOT_FOUND` | `PAYMENT_NOT_FOUND` |
| Duplicate completed or provider mismatch | `FAILED_PRECONDITION` | `PAYMENT_STATE_INVALID` |
| Missing workload authentication | `UNAUTHENTICATED` | `UNAUTHENTICATED` |
| Workload or role denial | `PERMISSION_DENIED` | `FORBIDDEN` |
| Provider, Kafka, or Registry unavailable | `UNAVAILABLE` | `PAYMENT_UNAVAILABLE` |

Webhook authentication failures use HTTP `400` or `401`, not gRPC errors. Do not leak provider or internal exception text.

## Release decision

Default G11 is docs + contract matrices only. Do not publish a new `gym-proto` tag when wire contracts remain unchanged.

If a later decision adds QR fields or changes wire bytes, publish one new immutable source/artifact release through the protected contract flow. Preserve the existing v7.0.2 artifacts and G8 evidence.

## Explicit non-goals

- Clone or scaffold `ms-gym-payment`.
- Make SePay network calls or add a provider SDK.
- Promote Payment routes into Kong, generated OpenAPI, or active `google.api.http` annotations.
- Add Member consumers for failed or refunded events.
- Activate Momo, ZaloPay, VNPay, `TRAINER_BOOKING`, Promotion reservations, refunds, spending history, or revenue reporting.
- Replace the G8 fake-payment fixture.
- Add raw QR payload fields without a product decision.

## Verification

```bash
cd /home/phucl/Workplace/gapi/gym-proto
buf format -d --exit-code
buf lint
buf breaking --against '.git#tag=v7.0.2'

cd /home/phucl/Workplace/gapi
grep -R -n -E 'payment\.failed|payment\.refunded|SEPAY' \
  docs/services/03-ms-gym-payment.md \
  docs/architecture/03-kafka-events.md \
  docs/plans/foundation-first/11-payment-contracts.md

grep -n 'payment\.completed\.v1' gym-proto/contracts/v1/kafka/wire-format.json
grep -R -n -E 'InitiatePayment|payment\.completed' ms-gym-member/src/main/java/com/gym/member --include='*.java'
```

G11 verification is documentation and contract consistency. Existing Member tests and the G8 fake-payment fixture remain the compatibility smoke check; no new service E2E is required.

## G11 exit criteria

- This document freezes ownership, SePay-first policy, RPC exposure, webhook trust, idempotency, and Kafka naming.
- The roadmap lists Phase 11 / G11 and opens Payment contracts only.
- The Payment catalog no longer presents Momo, ZaloPay, or VNPay as active V1 providers.
- `payment.completed.v1` and Member/G8 behavior remain unchanged.
- `payment.failed.v1` and `payment.refunded.v1` appear only as documented future names.
- Buf format, lint, and breaking checks pass for the chosen zero-wire delta.
- No `ms-gym-payment` implementation tree is required.

## Deferred to the implementation phase

Clone the empty remote; build Spring service, PostgreSQL schema, outbox, SePay client, webhook controller, mTLS workload auth, and tests; swap the fake-payment fixture; then open Kong/OpenAPI, failed/refunded producers, refunds, discounts, trainer payments, and any QR fields.

## Open risks

1. Exact SePay `code` ↔ Payment order mapping.
2. VietQR clients may need raw QR content beyond `payment_url`.
3. Payment public HTTP remains deferred YAML, not gateway-ready.
4. Exact HMAC string-to-sign must be pinned against SePay documentation.
5. InitiatePayment mTLS SAN allowlist is policy in G11 and enforcement is deferred.
