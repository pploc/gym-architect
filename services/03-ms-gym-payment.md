# Payment Service

> **Roadmap:** G11 complete on 2026-08-24. This supersedes the former contracts-only status; historical G8 fake-payment evidence remains unchanged. Locked evidence and user-directed accountable owner acceptance are recorded in [Phase 11](../plans/foundation-first/11-payment-contracts.md).
> **Provider and scope:** SePay VietQR / bank-transfer webhook, `SEPAY` and `MEMBERSHIP` only. Momo, ZaloPay, VNPay, `TRAINER_BOOKING`, refunds, history, discounts, and reports remain deferred.
> **Target:** Java 26 + Spring Boot 4 | PostgreSQL `payment_db` | gRPC `50051` | actuator health/readiness `8080`

## Responsibilities

- Member-only mTLS `InitiatePayment`, idempotent by `(MEMBERSHIP, purchase_id)`.
- Generate an opaque official VietQR `payment_url`.
- Verify and durably process the native SePay webhook.
- Complete once and relay unchanged `payment.completed.v1` through a transactional outbox.
- Store actual overpayment while emitting the Member-frozen amount.

Payment transactional outbox is the active `payment.completed.v1` producer. The G8 fake remains a historical compatibility fixture.

## Frozen boundary

| Concern | Rule |
|---|---|
| Amount | Member freezes `amount_vnd`; Payment does not call Plans or reprice. `transferAmount >= amount_vnd` completes; overpay actual is retained, event amount stays frozen. |
| Payment code | Exact configured code maps an inbound transfer to one intent. Prefix is 2–5 uppercase characters; suffix is 1–30 numeric/alphanumeric characters. |
| Event | Existing `PaymentCompletedEvent`, key `user_id`, `payment.completed.v1` subject/framing/headers only. No new proto. |
| Duplicate | Same SePay `id` or valid replay returns success without another completion/event. |
| Exposure | No public Payment RPC, inline HTTP annotation, OpenAPI, or Kong route. Native webhook only: `POST /api/v1/payments/webhook/sepay`, HTTPS, no JWT. |
| Future names | `payment.failed.v1` and `payment.refunded.v1` are names only. |

## Official SePay webhook pin

Official sources accessed 2026-08-24:

- [Authentication](https://developer.sepay.vn/vi/sepay-webhooks/xac-thuc)
- [Webhook payload and response](https://developer.sepay.vn/vi/sepay-webhooks/tich-hop-webhook)
- [Retry](https://developer.sepay.vn/vi/sepay-webhooks/xu-ly-loi)
- [Payment code](https://developer.sepay.vn/vi/sepay-webhooks/cau-hinh-ma-thanh-toan)
- [VietQR image/form](https://developer.sepay.vn/vi/sepay-webhooks/tao-qr-va-form-thanh-toan)

Use recommended HMAC, not an API-key substitute:

```text
X-SePay-Signature: sha256={hex_hash}
X-SePay-Timestamp: Unix seconds
HMAC-SHA256 secret input: {timestamp}.{raw_body}
reject timestamp: outside ±5 minutes
```

Preserve raw bytes until HMAC verification completes and compare in constant time. HTTPS is mandatory; use a SePay IP allowlist where infrastructure supports it. Do not log secret/signature/full callback/account/payment content.

| Field | Handling |
|---|---|
| `id` | Stable integer, unique across retry/replay; receipt uniqueness key. |
| `gateway` | Provider metadata. |
| `transactionDate` | `YYYY-MM-DD HH:mm:ss` in Vietnam time. |
| `accountNumber`, `subAccount` | Verify configured receiving account where applicable; protect. |
| `code` | Nullable; exact intent-code match required. |
| `content`, `description`, `referenceCode` | Protected reconciliation metadata only as needed. |
| `transferType` | Only `in` may complete. |
| `transferAmount` | Positive integer VND; underpay does not complete; retain actual overpay. |
| `accumulated` | Never completion authority. |

Valid completion, duplicate, or authenticated persisted orphan responds within 30 seconds with HTTP 200 or 201 and exact `{"success":true}`. Invalid signature, malformed timestamp/body, and stale timestamps return safe non-2xx. SePay retries connection errors/non-2xx with initial plus seven Fibonacci retries: 1, 1, 2, 3, 5, 8, 13 minutes, about 33 minutes.

The payment URL is opaque and generated from URL-encoded required parameters:

```text
https://vietqr.app/img?acc={account}&bank={bank}&amount={amount}&des={payment_code}
```

## Flow

```mermaid
sequenceDiagram
    participant MB as Member
    participant PM as Payment
    participant DB as payment_db
    participant SP as SePay
    participant KF as Kafka

    MB->>PM: mTLS InitiatePayment(purchase_id, frozen amount)
    PM->>DB: Create/reuse PENDING intent and exact code
    PM-->>MB: payment_id, opaque VietQR payment_url
    SP->>PM: HTTPS webhook, raw body + HMAC headers
    PM->>PM: Verify HMAC/timestamp before parsing
    PM->>DB: Record receipt; match inbound exact code; complete once + outbox
    PM-->>SP: 200/201 {"success":true}
    PM->>KF: payment.completed.v1, frozen amount
    KF-->>MB: Existing completion consumer activates frozen terms
```

## Service target

```text
ms-gym-payment/
├── src/main/java/com/gym/payment/
│   ├── domain/
│   ├── application/
│   ├── adapter/in/{grpc,http}/
│   ├── adapter/out/{persistence,kafka}/
│   └── config/
├── src/main/resources/db/migration/
├── src/test/
├── docker-compose.yml
├── Dockerfile
├── build.gradle
├── gradlew
└── README.md
```

`payment_intents` owns opaque Member refs, provider/type, frozen/actual amounts, code, URL, state, and unique membership reference. `payment_webhook_receipts` uniquely owns SePay `id` and verification/processing state. `outbox_events` is committed in the same completion transaction. Database constraints plus transaction boundaries enforce idempotency.

The repository must supply `./gradlew startEnv` and `./gradlew stopEnv` for local dependencies.

## G11 completion evidence

- given/when/then tests for intent reuse, Member-only mTLS, encoded QR URL, raw HMAC/timestamp boundaries, exact code/direction/account, duplicate `id`, underpay, overpay, orphan, and safe errors;
- PostgreSQL migration, outbox relay/DLQ, unchanged Kafka framing/headers, Member one-time activation, health, HTTPS ingress, mTLS SAN, NetworkPolicy, and no-public-Payment-route negatives;
- least-privilege `payment_db`, secret-mounted `SEPAY_WEBHOOK_SECRET` and account/bank/code settings, Kafka/Registry credentials, and optional deployment-managed SePay IP allowlist;
- detached source/image/config lock, protected CI, sanitized evidence, clean trees, and owner acceptance.

This proof passed on 2026-08-24. Payment is complete for the frozen membership-only SePay scope; G8 fake-payment remains a historical compatibility fixture.
