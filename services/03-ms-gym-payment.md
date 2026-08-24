# Payment Service

> **Roadmap:** G11 contracts only. `ms-gym-payment` implementation is deferred; the local tree is absent and `git@github.com:pploc/ms-gym-payment.git` is empty.
> **G11 provider:** SePay VietQR / bank-transfer webhook, `SEPAY` only. Momo, ZaloPay, and VNPay remain deferred.
> **Tech target:** Java 26 (Spring Boot 4) | **DB:** PostgreSQL | **Ports target:** 50051 (gRPC) / 8080 (REST)

## Responsibilities

- SePay VietQR payment initiation and webhook completion for `MEMBERSHIP`.
- Idempotent payment processing keyed by `(payment_type=MEMBERSHIP, reference_id=purchase_id)`.
- Publish `payment.completed.v1` after verified completion; G8 fake-payment remains the current fixture producer.
- Expose a native REST provider webhook without JWT.

Refunds, transaction history, discounts, `TRAINER_BOOKING`, and other providers remain deferred implementation scope. G11 freezes contracts only; it does not implement providers, persistence, an outbox, webhook code, Kong routes, or generated Payment HTTP exposure.

## Payment Flow

```mermaid
sequenceDiagram
    participant C as Mobile App
    participant K as Kong
    participant MS as Member Service
    participant PS as Payment Service
    participant PP as SePay VietQR
    participant KF as Kafka

    C->>K: Member-owned purchase route with provider: "SEPAY"
    K->>MS: gRPC PurchaseMembership
    MS->>PS: gRPC InitiatePayment(reference_id=purchase_id, amount_vnd)
    PS->>PS: Create or reuse PENDING intent by (MEMBERSHIP, purchase_id)
    PS->>PP: Create SePay VietQR payment session
    PP-->>PS: payment_url and provider_code
    PS-->>MS: payment_id and payment_url
    MS-->>C: payment_id and payment_url
    C->>PP: Complete bank transfer
    PP->>PS: POST /api/v1/payments/webhook/sepay
    PS->>PS: Verify raw-body signature, timestamp, direction, and amount
    PS->>PS: Complete once; duplicate callback is idempotent
    PS->>KF: Publish payment.completed.v1
    KF-->>MS: Activate membership from frozen terms
```

## Domain Contract

- Status: `PENDING`, `COMPLETED`, `FAILED`, `REFUNDED`.
- G11 producer type: `MEMBERSHIP`. `TRAINER_BOOKING` remains schema-present but inactive.
- Provider: `SEPAY` only.
- Currency: VND; `amount_vnd` is a non-negative `int64`.
- Cross-service IDs are opaque strings; no cross-service database foreign keys.
- Payment uses Member's frozen amount and never re-prices or re-resolves a plan during completion.

## Idempotency

Member always sends `reference_id = purchase_id`.

```text
key = (payment_type=MEMBERSHIP, reference_id=purchase_id)

PENDING or COMPLETED -> return the same payment_id and payment_url
FAILED -> retry requires a new Member purchase_id
missing -> create one PENDING intent
```

Payment must never mint a second charge for one membership `purchase_id`.

## SePay VietQR Contract

G11 freezes the provider boundary, not the provider SDK or network implementation. Clients use the existing opaque `payment_url`; raw QR fields are not added in G11.

```text
Create: SePay VietQR payment session
Return: payment_url and provider order/code reference
Webhook: POST /api/v1/payments/webhook/sepay
Headers: X-SEPAY-SIGNATURE, X-SEPAY-TIMESTAMP
Body: raw JSON; code, transferType, transferAmount, content, referenceCode
Verify: HMAC-SHA256 over vendor-defined raw-body signing input
Reject: invalid signature, invalid JSON, stale timestamp (> 5 minutes), transferType != in,
        or transfer amount below the frozen amount
Success: HTTP 200 {"success": true}; valid duplicates also return success
```

A verified callback maps the provider code/order reference to one Payment intent, transitions it once to `COMPLETED`, and publishes `payment.completed.v1` with `provider=SEPAY`. Exact field mapping and the vendor signing formula require confirmation during implementation.

Momo, ZaloPay, and VNPay are deferred providers; their endpoints are not part of G11.

## Kafka Events

### Published

| Topic | Key | G11 status | Payload |
|---|---|---|---|
| `payment.completed.v1` | `user_id` | Frozen active; G8 fixture remains the current producer | `{payment_id, user_id, type, reference_id, amount_vnd, gym_id, provider}` |
| `payment.failed.v1` | `user_id` | Name freeze only; no producer, Registry subject, or Member consumer | Future contract |
| `payment.refunded.v1` | `user_id` | Name freeze only; no producer, Registry subject, or Member consumer | Future contract |

The historical unversioned names `payment.failed` and `payment.refunded` are superseded by the `.v1` names. `{topic}.DLQ` behavior remains owned by common libraries.

### Consumed later

| Topic | G11 status |
|---|---|
| `membership.paused.v1` | Refund behavior deferred |
| `booking.cancelled` | Trainer refund behavior deferred |
| `booking.auto-rejected` | Trainer refund behavior deferred |
| `trainer.suspended` | Trainer refund behavior deferred |

## API Contract

| RPC | Caller | HTTP in G11 | Policy |
|---|---|---|---|
| `InitiatePayment` | Member workload over mTLS | None | Active dependency; create or reuse an intent; return `payment_id` and `payment_url`. |
| `GetPaymentStatus` | Deferred public client | Deferred | No active route. |
| `GetSpendingHistory` | Deferred public client | Deferred | No active route. |
| `RefundPayment` | Deferred admin | None | Later implementation phase. |
| `GetRevenueReport` | Deferred admin | None | Later implementation phase. |
| `GetPaymentsByUser` | Deferred internal caller | None | Later implementation phase. |

Payment public methods remain in deferred `payment_http.yaml`; G11 does not promote them into inline `google.api.http`, generated OpenAPI, or Kong routes.

## Native Webhook Boundary

The SePay webhook is a plain Spring MVC REST controller, not a gRPC method. It must preserve raw request bytes for signature verification and has no JWT.

```java
@RestController
@RequestMapping("/api/v1/payments/webhook")
public class WebhookController {
    @PostMapping("/sepay")
    public ResponseEntity<Map<String, Object>> sepayCallback(
            HttpServletRequest request) throws IOException {
        byte[] rawBody = request.getInputStream().readAllBytes();
        // Verify X-SEPAY-SIGNATURE against rawBody before parsing JSON.
        // ...
        return ResponseEntity.ok(Map.of("success", true));
    }
}
```

Valid duplicate callbacks return HTTP 200 without a second completion event. Orphan persistence, alerting, and retry handling remain implementation-phase work.

## Errors

Use canonical gRPC statuses and stable `x-error-code` values through `common-java`:

| Condition | Status | Error code |
|---|---|---|
| Invalid field, amount, provider, or type | `INVALID_ARGUMENT` | `INVALID_ARGUMENT` |
| Missing payment reference | `NOT_FOUND` | `PAYMENT_NOT_FOUND` |
| Duplicate completed provider mismatch | `FAILED_PRECONDITION` | `PAYMENT_STATE_INVALID` |
| Missing workload authentication | `UNAUTHENTICATED` | `UNAUTHENTICATED` |
| Workload role denial | `PERMISSION_DENIED` | `FORBIDDEN` |
| Provider, Kafka, or Registry unavailable | `UNAVAILABLE` | `PAYMENT_UNAVAILABLE` |

Webhook authentication failures use HTTP `400` or `401`, not gRPC errors. Do not leak provider internal exception text.

## G11 Boundary

G11 includes documentation contract matrices only. It does not include:

- cloning or scaffolding `ms-gym-payment`;
- SePay SDK network calls;
- Payment Kong routes, generated OpenAPI, or active `google.api.http` annotations;
- failed/refunded producers, Registry subjects, or Member consumers;
- Promotion reservations, refunds, trainer payments, or raw QR fields;
- replacement of the G8 fake-payment fixture.

## Implementation-Phase Notes

The later implementation phase clones the empty remote, builds the Spring service/PostgreSQL schema and outbox, implements the SePay client and webhook controller, enforces Member-only mTLS SAN policy, replaces the fake fixture, and then opens gateway exposure and deferred event families.

Open risks: exact SePay `code` to Payment order mapping, exact HMAC string-to-sign, and whether clients eventually need raw VietQR content beyond `payment_url`.
