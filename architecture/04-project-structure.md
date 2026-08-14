# Project Repository Structure

> **Roadmap status:** Sibling-repository workspace. G6–G8 are historical. Phase 9 is in progress: generated Go `grpc-gateway` is selected behind Kong, and final completion requires immutable v6.0.1 artifacts plus locked clean-source evidence.

## Workspace Model

The project is sibling Git repositories under one workspace, not one deployable monorepo:

```text
gapi/
├── docs/                 # Architecture, service specifications, roadmap, evidence
├── gym-proto/            # Protobuf, annotations, generated artifacts
├── common-go/            # Shared Go runtime packages
├── common-java/          # Shared Java runtime packages
├── gym-infra/            # Compose, Kong, Helm, reusable CI/CD
├── ms-gym-identifier/    # Go identity service
├── ms-gym-member/        # Java membership service
└── ms-gym-plans/         # Java catalog service
```

Payment, Workout, Trainer, Check-in, Notification, Analytics, and Promotion remain catalog entries.

## Contract Repository

`gym-proto` owns service/event schemas, validation, and final Member/Plans `google.api.http` annotations.

```text
gym-proto/
├── buf.yaml
├── buf.gen.yaml
├── buf.openapi.gen.yaml          # Gnostic OpenAPI generation
├── proto/
│   ├── identity/v1/
│   ├── member/v1/
│   ├── plans/v1/
│   ├── common/v1/
│   ├── events/v1/
│   └── google/api/
├── scripts/                      # Contract, route, OpenAPI, determinism checks
├── contracts/                    # JWT/route semantic manifests
├── openapi/
│   └── gym-active-api.openapi.yaml   # Canonical deterministic merged OpenAPI 3.0
└── dist/kong-proto-<version>.tar.gz  # Released contract source bundle
```

Phase 9 contract changes:

- remove Identity `SelectGym` and Member `GetMembershipStatusByUserId`; reserve only removed fields in retained messages and prevent retired symbol reuse with contract checks;
- remove `gym_id` and `membership_status` from JWT profile;
- add explicit gym context to gym-specific public Member requests;
- annotate public unary Member/Plans RPCs inline;
- keep `GetActiveGym`, `ResolvePurchasablePlan`, `ValidateMembership`, and `ListMembersByStatus` internal-only;
- generate Java/Go artifacts from Protobuf;
- generate exactly 12 Identity, seven Member, and eight Plans browser operations with Gnostic `protoc-gen-openapi@v0.7.1`, then deterministically merge them into canonical OpenAPI 3.0;
- publish contract source bundle for release verification; generated gateway compiles bindings and Kong does not parse Protobuf at runtime.

## Active Service Repositories

### `ms-gym-identifier`

```text
ms-gym-identifier/
├── cmd/server/
├── internal/
│   ├── domain/
│   ├── usecase/
│   │   └── port/
│   │       └── plans_client.go   # GetActiveGym for trainer validation
│   ├── adapter/
│   │   └── plans/
│   └── config/
├── migrations/
├── Dockerfile
└── go.mod
```

Phase 9 removes Member port/adapter/configuration and selected-gym use case. Identifier still uses PostgreSQL, Redis, Kafka, and Plans mTLS for trainer validation.

### `ms-gym-member`

```text
ms-gym-member/
├── src/main/java/com/gym/member/
│   ├── domain/
│   ├── application/
│   ├── adapter/
│   │   ├── in/grpc/
│   │   └── out/
│   │       ├── persistence/       # Profiles, subscriptions, purchases, outbox
│   │       ├── plans/             # ResolvePurchasablePlan client
│   │       ├── payment/
│   │       └── kafka/
│   └── config/
├── src/main/resources/db/migration/
├── Dockerfile
├── build.gradle
└── gradlew
```

Member owns no location/catalog package. Public business API remains gRPC internally and is represented as HTTPS/JSON by Kong.

### `ms-gym-plans`

```text
ms-gym-plans/
├── src/main/java/com/gym/plans/
│   ├── domain/
│   ├── application/
│   ├── adapter/
│   │   ├── in/grpc/              # Public + workload gRPC handlers
│   │   └── out/persistence/      # JPA repositories and Specifications
│   └── config/
├── src/main/resources/
│   ├── application.yml
│   └── db/migration/
├── Dockerfile
├── build.gradle
└── gradlew
```

After G9, Plans has no Spring MVC business adapter. Spring web remains only as needed for Actuator on `8080`; business gRPC uses `50051`. Filtering uses composed JPA `Specification` objects.

## Infrastructure Repository

```text
gym-infra/
├── kong/
│   ├── g5-*                       # Historical pre-split fixtures
│   ├── g8-*                       # Historical three-service fixtures
│   ├── g9-compose.yml             # Locked G9 fixture
│   ├── g9-release-lock.json
│   ├── materialize-g9.py
│   ├── generated-gateway/
│   ├── g9-business-check.sh
│   ├── run-g9.sh
│   └── fixtures/fake-payment/
├── helm/gym-service/
└── .github/workflows/
```

Create additive G9 fixtures rather than rewriting evidence inputs. G9 materializes locked detached sources, configures Kong-to-gateway and gateway-to-service mTLS, and renders port-specific NetworkPolicies.

## Development Commands

```bash
# Contracts
gym-proto$ make proto
gym-proto$ make gen-go
gym-proto$ make gen-java

# Go service
ms-gym-identifier$ go test -race ./...

# Java services
ms-gym-member$ ./gradlew startEnv
ms-gym-member$ ./gradlew test
ms-gym-member$ ./gradlew stopEnv

ms-gym-plans$ ./gradlew startEnv
ms-gym-plans$ ./gradlew clean check
ms-gym-plans$ ./gradlew stopEnv
```

## Dependency Rules

- Services consume tagged generated artifacts; do not copy generated stubs into service repositories.
- Backend/native clients generate from Protobuf.
- Browser REST clients generate from released OpenAPI 3.0.
- Generated gateway compiles released annotated Protobuf route bindings; Kong consumes neither Protobuf source nor OpenAPI at runtime.
- Shared behavior moves to common libraries only after more than one service proves identical need.
- Services own domain and migrations.
- Cross-service IDs are opaque strings and never database foreign keys.
- Public Member/Plans routes come from inline annotations; internal workload RPCs remain unmapped.
- Plans and Member filtering uses Spring Data JPA Specifications, not custom persistence queries.

See [Phase 9](../plans/foundation-first/09-kong-grpc-gateway-openapi.md) and [Plans service](../services/10-ms-gym-plans.md).
