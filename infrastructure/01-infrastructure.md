# Infrastructure Architecture

> **Roadmap status:** G0–G10 are complete. Check-in locked clean-source E2E, protected CI, sanitized evidence, and owner acceptance are recorded in [`../evidence/foundation-first/g10-final/README.md`](../evidence/foundation-first/g10-final/README.md). Historical G9 lock and evidence remain unchanged.

## G9 baseline and in-progress G10 topology

```mermaid
flowchart TB
    Client[Browser clients] -->|HTTPS JSON + stable JWT| Kong[Kong 3.8]
    Kong -->|HTTP 8080| ID[Identifier]
    Kong -->|mTLS HTTPS 8443| GW[Generated Go grpc-gateway]
    GW -->|mTLS gRPC 50051| MB[Member]
    GW -->|mTLS gRPC 50051| PL[Plans]
    GW -->|G10 complete: mTLS gRPC 50051| CI[Check-in]

    ID -->|mTLS 50051: GetActiveGym| PL
    MB -->|mTLS 50051: ResolvePurchasablePlan| PL
    CI -.->|G10: ValidateMembership| MB
    CI -.->|G10: ValidateCheckInGym| PL
    MB -->|fixture protocol| FP[G8 Fake Payment]

    ID --> IDDB[(identity_db)]
    MB --> MBDB[(member_db)]
    PL --> PLDB[(plans_db)]
    CI -.-> CIDB[(checkin_db / YugabyteDB)]
    CI -.-> KMS[AWS KMS]
    ID --> Redis[(Redis)]
    ID --> Kafka[[Kafka]]
    MB --> Kafka
    CI -.->|checkin.recorded.v1| Kafka
    FP --> Kafka
    Kafka --> SR[Schema Registry]
```

Kong is only browser endpoint. It validates stable JWTs, removes client-supplied trusted headers, injects verified identity/role metadata, applies CORS, and selects exact routes. Generated gateway accepts trusted metadata only from Kong SAN, strips arbitrary inbound metadata, and performs generated HTTPS/JSON-to-gRPC binding.

Kong cannot directly reach Member or Plans `50051`. Gateway cannot call workload-only RPCs. There is no Identifier-to-Member runtime edge. Fake Payment remains test infrastructure, not production Payment.

## Database isolation

| Database | Owner | Target tables |
|---|---|---|
| `identity_db` | Identifier | users, refresh tokens, verification tokens |
| `member_db` | Member | members, subscriptions, pending purchases, outbox, processed events |
| `plans_db` | Plans | gym locations, membership plans |
| `checkin_db` | Check-in, G10 complete | base64 KMS ciphertext in `key_ciphertext`, resolved CMK ARN in `key_reference`, check-ins, transactional outbox |

Only Plans owns catalog tables. Member stores opaque IDs and frozen purchased terms. Check-in stores opaque `user_id`, canonical `member_id`, and `gym_id` plus Check-in-owned state. Identifier stores no gym assignment. No cross-service DB FK or join is allowed. G10 adds no display-device table.

## Kong routes

Identifier remains Kong's native HTTP upstream on `8080`. Member and Plans public unary APIs route to generated gateway over mTLS `8443`.

```yaml
services:
  - name: ms-gym-identifier-http
    url: http://ms-gym-identifier.gym-system.svc.cluster.local:8080
    routes:
      # Generate exact method + regex entries for 12 Identity operations.
      # Never proxy broad /api/v1/auth or /api/v1 paths.
      - name: identifier-register
        methods: [POST]
        paths: ["~/api/v1/auth/register$"]

  - name: ms-gym-api-gateway
    protocol: https
    host: ms-gym-api-gateway.gym-system.svc.cluster.local
    port: 8443
    client_certificate:
      id: ${KONG_CLIENT_CERTIFICATE_ID}
    tls_verify: true
    ca_certificates: [${GATEWAY_CA_ID}]
    routes:
      - name: member-and-plans-public-json
        protocols: [https]
        # Generated exact method + regex allowlist from active-operation manifest.
        # G10 adds JWT-authenticated Check-in routes, including SUPER_ADMIN display retrieval.
        # Kong does not run grpc-gateway and does not mount Protobuf source.
```

G9's 12 Identity, seven Member, and eight Plans operations remain historical. G10 derives its expanded exact method-plus-regex entries from the frozen active-operation manifest. Broad `/api/v1/auth`, `/api/v1`, `/api/v1/gyms`, and `/api/v1/checkin` prefix routes are forbidden. Membership, catalog, and Check-in paths must select only their owning backend. Unknown paths, wrong methods, and cross-backend matches return route-level `404`. Every Check-in browser route, including iPad display retrieval, uses Kong JWT validation. Public proxy routes are HTTPS-only; any port-80 route redirects and never proxies Bearer traffic. Internal RPCs and reflection have no Kong route.

Kong 3.8 source-Protobuf `grpc-gateway` parsing failed on `buf/validate/validate.proto:535:9: field name expected`. That parser, local source mounts, descriptor/reflection route discovery, and Kong runtime Protobuf bundles are historical rejected behavior.

## Stable JWT and trusted metadata

Identifier JWT contains token-control fields plus `sub` and `role`; no `gym_id` or `membership_status`.

Kong must:

1. validate signature, issuer, audience, time, JTI, key ID, and revocation;
2. strip client copies of trusted headers on every route;
3. inject only verified user identity and role metadata;
4. stop requiring or injecting `x-gym-id` and `x-membership-status`;
5. let path binding populate Protobuf `gym_id`.

Generated gateway accepts metadata only if Kong's client certificate SAN is valid. It rebuilds only vetted identity/role and tracing metadata for gRPC. Member and Plans accept public metadata only when peer SAN is `ms-gym-api-gateway`. Caller-selected path gym is request context, not trusted claim or authorization proof.

## Workload mTLS matrix

| Caller certificate | Destination | Allowed method | Public HTTP/OpenAPI |
|---|---|---|---|
| `ms-gym-identifier` | Plans | `GetActiveGym` | No |
| `ms-gym-member` | Plans | `ResolvePurchasablePlan` | No |
| `ms-gym-checkin` | Member | `ValidateMembership(user_id, gym_id)` | No; G10 complete |
| `ms-gym-checkin` | Plans | dedicated `ValidateCheckInGym`-style method | No; G10 complete |
| `ms-gym-notification` | Member | `ListMembersByStatus` | No; deferred caller |
| `ms-gym-api-gateway` | Member | declared public methods only | Yes |
| `ms-gym-api-gateway` | Plans | declared public methods only | Yes |

`GetMembershipStatusByUserId` and Identifier-to-Member mTLS are removed. Workload identity proves calling service, never end-user role.

## NetworkPolicy targets

Policies must separate peers by destination port. Do not union callers into broad rule.

- Kong reaches generated gateway `8443` only.
- Generated gateway reaches Member and Plans `50051` only.
- Kong has no direct Member or Plans `50051` rule.
- Identifier and Member retain only exact Plans workload paths at `50051`.
- Check-in reaches only its exact Member and Plans workload methods at `50051`.
- Generated gateway reaches Check-in public methods at `50051`; Kong has no direct Check-in rule.
- Check-in reaches Yugabyte YSQL, AWS KMS API, Kafka, Schema Registry lookup, DNS, and configured observability endpoints only.
- Health observers may reach Plans and Check-in `8080`, plus metrics ports where configured.

Method authorization remains exact SAN enforcement inside gRPC servers. NetworkPolicy alone is insufficient.

## Service configuration

Identifier retains only Plans downstream settings:

```text
Identifier:
  PLANS_GRPC_TARGET
  PLANS_GRPC_DEADLINE
  PLANS_GRPC_CA
  PLANS_GRPC_CERT
  PLANS_GRPC_KEY

Member:
  PLANS_GRPC_TARGET
  PLANS_GRPC_DEADLINE
  PLANS_GRPC_CA
  PLANS_GRPC_CERT
  PLANS_GRPC_KEY

Check-in (G10 complete):
  MEMBER_GRPC_TARGET / DEADLINE / CA / CERT / KEY
  PLANS_GRPC_TARGET / DEADLINE / CA / CERT / KEY
  YUGABYTE_DSN through secret mount
  KMS_KEY_ID
  KMS_ENDPOINT_URL (LocalStack only)
  KAFKA_BROKERS / SCHEMA_REGISTRY_URL and protected credentials

Production uses AWS SDK default credentials through EKS workload identity, never static AWS credentials. IAM grants only `kms:Encrypt`, `kms:Decrypt`, and `kms:DescribeKey` for the resolved CMK.
```

Gateway requires Kong client CA, gateway server certificate/key, Member/Plans client certificate/key, and their trusted CAs through secret mounts. Production private keys belong in Kubernetes Secrets or secret manager, never committed configuration.

## Plans deployment

Plans keeps:

- `50051` for business gRPC;
- `8080` for Actuator health/readiness only;
- `9090` for metrics if configured;
- no Kafka, Schema Registry, Redis, Payment, or scheduler settings.

Required negative proof:

```text
Direct Plans :8080/api/v1/gyms returns 404.
Kong HTTPS /api/v1/gyms reaches generated gateway, then Plans gRPC :50051.
Actuator health on :8080 remains available to approved observers.
```

## Generated contract deployment

`gym-proto v6.0.1` is the released G9 baseline. Gnostic `protoc-gen-openapi@v0.7.1` generated individual Identity, Member, and Plans documents. Deterministic collision-rejecting merge emitted canonical `openapi/gym-active-api.openapi.yaml` with 12 Identity, seven Member, and eight Plans browser operations. Workload RPCs remain absent.

G10 released `gym-proto` v7.0.2, `gym-proto-java` 7.0.2, and `proto-go` v1.7.1; `common-go` v0.5.0 is the current shared Go artifact. Release/integration proof remains pending. The generation adds only approved Check-in browser operations and keeps Member/Plans workload RPCs absent. Expected operation totals come from the frozen active-operation manifest, not a hardcoded count.

Frontend consumes released canonical OpenAPI. Backend/native clients consume Protobuf artifacts. Generated gateway compiles generated route bindings; Kong consumes neither Protobuf source nor OpenAPI at runtime.

The historical G9 release lock pins:

```text
Detached repository SHAs
v6.0.1 Java and Go artifacts plus all asset checksums
Canonical OpenAPI checksum
Generated-gateway source identity and image digest
Kong image digest
Route manifest/template and redacted rendered-config checksums
```

Reject execution when locked generations differ. `run-g9.sh` materializes detached locked sources in temporary workspace; it must never read mutable sibling source trees.

## G9 evidence and CI

Protected authenticated G9 CI uses package/repository read credentials and BuildKit secrets for private dependencies. It runs locked clean-source G9, all 27 route positives/negatives, mTLS/SAN matrix, direct Plans HTTP negative, Actuator positive, Helm/NetworkPolicy checks, safe `500`/`503` error checks, and TypeScript generation from released canonical OpenAPI with `tsc --noEmit`.

Commit only schema-controlled sanitized final evidence. It records SHAs, versions, checksums, certificate public metadata, command labels, exit codes, and CI/release URLs. It must reject private keys, JWTs, authorization values, PII, fixture IDs, raw logs, and raw exception/transport details.

## In-progress G10 Check-in infrastructure

Use existing generic chart plus official/runtime dependencies:

- Check-in service on mTLS gRPC `50051`, health/readiness-only `8080`, and configured metrics port;
- YugabyteDB YSQL `checkin_db` with least-privilege credentials and explicit migration job/path;
- AWS KMS using AWS SDK default credentials through EKS workload identity, with IAM limited to `kms:Encrypt`, `kms:Decrypt`, and `kms:DescribeKey`; `KMS_ENDPOINT_URL` is LocalStack-only;
- Kafka topic `checkin.recorded.v1`, pre-registered subject `checkin.recorded.v1-value`, and `.DLQ`;
- generated-gateway Check-in backend and exact Kong JWT routes;
- caller-specific mTLS certificates and port-specific NetworkPolicies.

Provision least-privilege DB credentials, Check-in service account, mTLS identity, KMS CMK/key reference and IAM workload policy, Kafka topic, Schema Registry subject, and migration job/path. Secret values are mounted or injected by the approved secret boundary, never committed.

The iPad app uses normal Identifier login and stable JWT through Kong. No HTTP Basic route, kiosk/device credential, device secret, device database, or device-revocation infrastructure is planned. Display requests require `SUPER_ADMIN` and explicit active `gym_id`. Selected gym and membership status remain absent from JWT.

## CI/CD repository model

- `gym-proto`: Buf checks, Gnostic OpenAPI generation/merge, route checks, and immutable artifact publication;
- `ms-gym-identifier`: Go race tests and secret-safe image build;
- `ms-gym-member`: Gradle checks and secret-safe image build;
- `ms-gym-plans`: Gradle checks, Flyway, Specification tests, image and Helm checks;
- `ms-gym-checkin`: G10 Go race/integration/security checks and secret-safe image build;
- `gym-infra`: historical G9 lock plus additive G10 lock validation, detached-source materialization, authenticated E2E, sanitizer, and Helm rendering.

Java service docs retain `./gradlew startEnv` and `./gradlew stopEnv` for local dependencies.
