# Phase 11 — Implement `ms-gym-payment` and Locked Gate (G11)

> **Status: IN PROGRESS.** This phase supersedes the former contracts-only G11 scope on 2026-08-24. Its frozen decisions remain the baseline; preserve G0–G10 and G8 fake-payment evidence unchanged.
> **Gate:** no completion claim before locked clean-source E2E, protected CI, sanitized evidence, and owner acceptance pass.
> **Provider and scope:** `SEPAY` and `MEMBERSHIP` only. Momo, ZaloPay, VNPay, `TRAINER_BOOKING`, refunds, discounts, history, and revenue reporting remain out of scope.

## Objective

Build the sibling Java/Spring Payment service and prove the real membership-payment path in a locked gate:

```text
Member frozen purchase -> InitiatePayment -> opaque VietQR payment_url
SePay HTTPS webhook -> verify/persist/complete once -> outbox payment.completed.v1
Member -> validate frozen completion -> activate membership
```

No new Protobuf release is authorized. Keep `InitiatePayment`, `PaymentCompletedEvent`, `payment.completed.v1`, its `user_id` key, subject, framing, headers, and Member validation unchanged.

## Frozen decisions

| Decision | Locked choice |
|---|---|
| Scope | `MEMBERSHIP` only; Member is the only workload caller. |
| Provider | `SEPAY` only; emitted provider remains `SEPAY`. |
| Amount | Member supplies frozen `amount_vnd`; Payment never reads Plans or reprices. Overpayment completes and stores actual VND, but emits the frozen amount. Underpayment does not complete. |
| Intent idempotency | Unique `(payment_type=MEMBERSHIP, reference_id=purchase_id)`; pending/completed retries return the same `payment_id` and `payment_url`. A failed attempt needs a new Member `purchase_id`. |
| Completion | Match only exact configured payment code and `transferType=in`; complete once. SePay `id` is a stable integer, unique across retries/replays. |
| Event | Transactional outbox publishes the existing keyed-by-`user_id` `payment.completed.v1`; no new proto or Schema Registry subject. |
| Exposure | `InitiatePayment` is mTLS-only from Member. No public Payment RPC, inline `google.api.http`, generated OpenAPI, or Kong Payment route. `POST /api/v1/payments/webhook/sepay` is native Spring MVC, HTTPS-only, without JWT. |
| Deferred names | `payment.failed.v1` and `payment.refunded.v1` are names only: no producer, subject, consumer, refund behavior, or Member state transition. |
| Historical fixture | G8 fake-payment remains current producer until the G11 locked pass proves the real producer. |

Cross-service IDs are opaque strings. Payment creates no cross-service foreign key.

## Official SePay contract pin

Pinned from current official SePay documentation accessed **2026-08-24**:

- Authentication: <https://developer.sepay.vn/vi/sepay-webhooks/xac-thuc>
- Webhook payload and response: <https://developer.sepay.vn/vi/sepay-webhooks/tich-hop-webhook>
- Retry: <https://developer.sepay.vn/vi/sepay-webhooks/xu-ly-loi>
- Payment-code configuration: <https://developer.sepay.vn/vi/sepay-webhooks/cau-hinh-ma-thanh-toan>
- VietQR image/form: <https://developer.sepay.vn/vi/sepay-webhooks/tao-qr-va-form-thanh-toan>

Use the documented HMAC mode, not an API-key substitute. Verify `X-SePay-Signature` exactly as `sha256={hex_hash}` and `X-SePay-Timestamp` as Unix seconds. Compute HMAC-SHA256 with the configured webhook secret over:

```text
{timestamp}.{raw_body}
```

Preserve received raw bytes until verification completes, compare signatures in constant time, and reject timestamps outside ±5 minutes. Use HTTPS and a SePay IP allowlist where deployment controls permit it; HMAC remains mandatory.

Persist the raw webhook receipt, or a redacted/auditable digest where policy requires, its provider `id`, verification result, and completion outcome before acknowledging a valid first delivery. Never log the secret, signature, full request body, account number, or payment content.

### Required payload mapping

| SePay field | Required handling |
|---|---|
| `id` | Stable integer unique across retries/replays; unique receipt key. |
| `gateway` | Provider metadata. |
| `transactionDate` | Parse `YYYY-MM-DD HH:mm:ss` in Vietnam time. |
| `accountNumber`, `subAccount` | Match configured receiving account where applicable; protect in storage/logs. |
| `code` | Nullable parsed payment code; completion requires exact intent-code match. |
| `content`, `description`, `referenceCode` | Protected reconciliation metadata only as needed. |
| `transferType` | Complete only for `in`. |
| `transferAmount` | Positive integer VND; at least frozen amount; retain actual amount on overpay. |
| `accumulated` | Optional operational metadata; never completion authority. |

Configured codes use a 2–5 character uppercase prefix and a 1–30 character numeric/alphanumeric suffix. Match exact code only; no contains or normalization fallback.

Generate opaque `payment_url` with required, URL-encoded official VietQR parameters:

```text
https://vietqr.app/img?acc={account}&bank={bank}&amount={amount}&des={payment_code}
```

No raw QR fields are added. A valid completion, valid duplicate, or authenticated persisted orphan returns within 30 seconds with HTTP 200 or 201 and exact JSON `{"success":true}`. Invalid signature, malformed timestamp/body, or stale timestamp receives safe non-2xx without internal details.

When SePay retry is enabled, connection errors and non-2xx responses receive initial delivery plus seven Fibonacci retries at 1, 1, 2, 3, 5, 8, and 13 minutes, about 33 minutes. Valid replay handling must therefore be safe.

## Service and schema target

Create sibling `ms-gym-payment/` with Java 26, Spring Boot 4, PostgreSQL `payment_db`, gRPC `50051`, actuator health/readiness `8080`, and `./gradlew startEnv`/`./gradlew stopEnv` wrapping local Docker Compose dependencies.

```text
ms-gym-payment/
├── src/main/java/com/gym/payment/
│   ├── domain/                 # intent, receipt, state transitions
│   ├── application/            # initiate and complete commands
│   ├── adapter/in/{grpc,http}/ # Member gRPC; native SePay webhook
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

| Table | Required responsibility |
|---|---|
| `payment_intents` | Payment ID, opaque Member references/user/gym, `MEMBERSHIP`, `SEPAY`, frozen amount, actual received amount, exact payment code, URL, state, and unique membership-reference key. |
| `payment_webhook_receipts` | Unique SePay `id`, verified receipt metadata, processing state, and linked intent when resolved. |
| `outbox_events` | Existing `payment.completed.v1` payload/key/headers and relay state, committed with completion. |

Database uniqueness and transactions enforce idempotency. A completion transaction records receipt, transitions a pending intent once, stores actual received amount, and inserts the outbox row. Duplicates create no new event. `FAILED` and `REFUNDED` remain persistence vocabulary only if needed; no event behavior opens here.

## Executable stages

| Stage | Work | Required proof |
|---|---|---|
| 0 — Entry lock | Record the SePay pin; preserve zero-wire contract; make a G11 source/evidence lock manifest. | URL/date/content checksum; `buf format -d --exit-code`, `buf lint`, and `buf breaking --against '.git#tag=v7.0.2'` show zero wire delta. |
| 1 — Service base | Create sibling service, Gradle environment tasks, PostgreSQL migrations, health/readiness, Member-only mTLS `InitiatePayment`, intent idempotency. | Clean `startEnv`, migration, tests, `stopEnv`; Member fixture proves stable ID/URL reuse. |
| 2 — QR and webhook | Build official encoded VietQR URL; raw-body HMAC/timestamp verification, exact code/account/direction/amount checks, receipt storage, duplicates. | Fixtures cover signature, ±5-minute boundary, exact code, duplicate `id`, underpay, overpay. |
| 3 — Event path | Add transactional outbox relay; replace fake only in G11 integration fixture. | Real service emits one unchanged framed event; Member activates once; replay harmless. |
| 4 — Deployment security | Image/Helm values, DB secrets, webhook HTTPS/IP allowlist, Kafka/Registry access, mTLS SAN/NetworkPolicy, CI. | Rendered manifests prove exact peers/ports; secrets and non-webhook public routes absent. |
| 5 — Locked gate | Detached locked sources and pinned images/configuration; protected CI; sanitizer; clean-tree checks. | Sanitized evidence, SHAs/checksums, command exits, route negatives, security/replay/overpay E2E, owner acceptance. |

## Tests and acceptance checks

Use given/when/then names or explicit sections. Prove:

- same membership `purchase_id` reuses one intent; only Member mTLS SAN can initiate;
- VietQR URL encodes all four required parameters;
- HMAC signs `{timestamp}.{raw_body}` without JSON reserialization; absent/malformed/wrong `sha256=` signatures and timestamps outside ±5 minutes fail;
- exact configured uppercase code, receiving account, `transferType=in`, and positive VND amount are enforced;
- first verified callback makes one completion/outbox row; same SePay `id` or retried callback returns exact success without another event;
- underpay remains incomplete; overpay stores actual VND while emitted event keeps frozen amount;
- valid orphan is retained/alerted and safely acknowledged; malformed or unauthenticated requests are safe non-2xx;
- Member consumes real event once, validates existing fields, and activates frozen terms; fake remains selectable only for historical G8 tests;
- migration, relay/retry/DLQ, Kafka framing/headers, health, mTLS/SAN, NetworkPolicy, HTTPS webhook, and no-Kong/no-public-Payment-RPC negatives pass.

## Infrastructure and locked evidence

Payment needs least-privilege `payment_db` credentials, migration path/job, `SEPAY_WEBHOOK_SECRET` plus receiving-account/bank/code configuration via approved secret boundary, Member↔Payment mTLS certificates, Kafka/Schema Registry lookup credentials, and HTTPS webhook ingress with optional deployment-managed SePay IP allowlist. Never commit secret values, callback captures, account numbers, payment content, JWTs, or private keys.

The G11 lock pins detached source SHAs, released contract/library versions, service/dependency image digests, migration checksum, rendered Helm/NetworkPolicy checksum, SePay URL/date/content checksum, and sanitized fixture checksums. Add produced proof under `docs/evidence/foundation-first/g11/`; never rewrite G8 or earlier evidence.

## Exit criteria

G11 is complete only when all stages pass in the locked gate. Until then, `ms-gym-payment` is **implementation in progress**, G8 fake-payment remains current producer, and `payment.failed.v1`/`payment.refunded.v1` remain names only.
