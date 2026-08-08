# Project Repository Structure

> **Roadmap status:** Sibling-repository workspace. G6 contracts published (`v3.0.0`). `ms-gym-plans` exists locally with Spring HTTP + gRPC; G8 integration and remaining G7 delivery evidence (CI/Helm) still open. Historical Phase 0–5 evidence remains unchanged.

## Workspace Model

The project is a collection of sibling Git repositories under one local workspace, not a single deployable monorepo:

```text
gapi/
├── docs/                 # Architecture, service specifications, roadmap, evidence
├── gym-proto/            # Protobuf, HTTP mappings, generated artifact publication
├── common-go/            # Shared Go runtime packages
├── common-java/          # Shared Java runtime packages
├── gym-infra/            # Compose, Kong, Helm, reusable CI/CD
├── ms-gym-identifier/    # Existing Go service
├── ms-gym-member/        # Existing Java service
└── ms-gym-plans/         # G7 Java service (Spring HTTP 8080 + gRPC 50051)
```

Payment, Workout, Trainer, Check-in, Notification, Analytics, and Promotion remain service-catalog entries. Their contracts and design docs do not imply that service repositories or deployments are active through G8.

## Contract Repository

`gym-proto` is the source of truth for service and Kafka contracts. Existing packages keep API package names such as `identity.v1` and `member.v1`. G6 plans a breaking repository release because location and plan catalog RPCs move out of `MemberService`.

```text
gym-proto/
├── buf.yaml
├── buf.gen.yaml
├── proto/
│   ├── identity/v1/
│   │   ├── identity.proto
│   │   └── identity_http.yaml
│   ├── member/v1/
│   │   ├── member.proto
│   │   └── member_http.yaml
│   ├── plans/v1/                 # Planned G6 addition
│   │   ├── plans.proto
│   │   └── plans_http.yaml
│   ├── payment/v1/               # Deferred service catalog
│   ├── workout/v1/               # Deferred service catalog
│   ├── trainer/v1/               # Deferred service catalog
│   ├── checkin/v1/               # Deferred service catalog
│   ├── notification/v1/          # Deferred service catalog
│   ├── analytics/v1/             # Deferred service catalog
│   ├── promotion/v1/             # Deferred service catalog
│   └── events/v1/
├── scripts/
└── contracts/
```

G6 target changes:

- Add `plans.v1.PlansService` and public Plans HTTP mappings.
- Remove plan listing and gym-location RPCs/messages from `member.v1.MemberService`.
- Keep Plans `GetActiveGym` and `ResolvePurchasablePlan` workload-only and absent from HTTP mappings.
- Prepare immutable release targets: Java `com.gym.proto:gym-proto-java:3.0.0` and Go `github.com/pploc/proto-go/v3`.

Those release coordinates remain proposals until G6 validation and explicit publication approval. Existing v2 artifacts are never rewritten.

## Active Service Repositories

### `ms-gym-identifier`

```text
ms-gym-identifier/
├── cmd/server/
├── internal/
│   ├── domain/
│   ├── usecase/
│   │   └── port/            # Separate Member and planned Plans ports
│   ├── adapter/
│   │   ├── member/          # Membership-status workload client
│   │   └── plans/           # Planned active-gym workload client
│   └── config/
├── migrations/
├── Dockerfile
└── go.mod
```

Identifier uses PostgreSQL `identity_db`, Redis for token revocation, and Kafka for identity events. It owns no Plans or Member persistence.

### `ms-gym-member`

```text
ms-gym-member/
├── src/main/java/com/gym/member/
│   ├── domain/
│   ├── application/
│   ├── adapter/
│   │   ├── in/grpc/
│   │   └── out/
│   │       ├── persistence/       # Profiles, subscriptions, pending purchases
│   │       ├── plans/             # Planned ResolvePurchasablePlan client
│   │       ├── payment/
│   │       └── kafka/
│   └── config/
├── src/main/resources/
│   └── db/migration/
├── Dockerfile
├── build.gradle
└── gradlew
```

The G8 target contains no Member-owned location or plan-catalog package. Member retains membership lifecycle, validation, events, and purchased-term snapshots.

### `ms-gym-plans`

```text
ms-gym-plans/
├── src/main/java/com/gym/plans/
│   ├── domain/
│   ├── application/
│   ├── adapter/
│   │   ├── in/
│   │   │   ├── http/        # Spring MVC public catalog routes (Kong target)
│   │   │   └── grpc/        # Native gRPC public + internal workload RPCs
│   │   └── out/persistence/ # JPA entities, repositories, Specifications
│   └── config/
├── src/main/resources/
│   ├── application.yml
│   └── db/migration/
├── Dockerfile
├── build.gradle
└── gradlew
```

Stack: Java 26, Spring Boot 4, Flyway, Spring Data JPA, PostgreSQL `plans_db`, Spring HTTP `8080`, native gRPC `50051`. Public HTTP is in-process Spring MVC (not a Go grpc-gateway sidecar). Query filtering uses composed JPA `Specification` objects. Plans V1 has no Kafka, Schema Registry, outbox, cache, scheduler, or Payment packages.

## Infrastructure Repository

`gym-infra` is separate from service repositories:

```text
gym-infra/
├── kong/
│   ├── g5-compose.yml             # Historical pre-split fixture
│   ├── g5-business-check.sh       # Historical evidence helper
│   └── ...planned G8 additions...
├── helm/
│   └── gym-service/
└── .github/workflows/
```

G5 files remain unchanged and truthful about the pre-split topology. G8 infrastructure is additive: a separate Plans database/service, caller-specific certificates, Plans public routes, and a phase-only fake Payment fixture.

## Development Commands

Commands run in the repository they target:

```bash
# Contracts
gym-proto$ make proto
gym-proto$ make gen-go
gym-proto$ make gen-java

# Existing active services
ms-gym-identifier$ go test -race ./...
ms-gym-member$ ./gradlew test

ms-gym-plans$ ./gradlew clean check
```

Workspace-level convenience targets may coordinate repositories, but they do not change repository ownership or imply that deferred service repositories exist.

## Dependency Rules

- Services consume tagged generated artifacts; generated stubs are not copied into service repositories.
- Shared runtime behavior belongs in `common-go` or `common-java` only when more than one service needs it.
- Service repositories own their domain and database migrations.
- Cross-service IDs are opaque strings and never database foreign keys.
- Public HTTP routes come from per-service HTTP configuration; internal workload RPCs remain unmapped.

See [Phase 6 contracts](../plans/foundation-first/06-plans-contracts.md), [Phase 7 Plans](../plans/foundation-first/07-ms-gym-plans.md), and the [Plans service specification](../services/10-ms-gym-plans.md).
