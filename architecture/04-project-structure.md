# Project Repository Structure

This system uses a **monorepo layout** (`gym-chain/`) containing service implementations, local infrastructure deployment files, shared utility packages, and the Protobuf schema definitions.

---

## 1. Protobuf Directory Structure (`gym-proto/`)

The single source of truth for all API contracts and Kafka event schemas, located in the `gym-proto/` folder at the root of the repository.

```
gym-proto/
├── buf.yaml                        # Buf module config
├── buf.gen.yaml                    # Code generation template (Go & Java stubs)
└── proto/
    ├── common/v1/
    │   └── common.proto                # Shared types (Pagination, Money, Timestamps)
    ├── identity/v1/
    │   ├── identity.proto
    │   └── identity_http.yaml          # gRPC-Gateway HTTP annotations
    ├── member/v1/
    │   ├── member.proto
    │   └── member_http.yaml
    ├── payment/v1/
    │   ├── payment.proto
    │   └── payment_http.yaml
    ├── workout/v1/
    │   ├── workout.proto
    │   └── workout_http.yaml
    ├── trainer/v1/
    │   ├── trainer.proto
    │   └── trainer_http.yaml
    ├── checkin/v1/
    │   ├── checkin.proto
    │   └── checkin_http.yaml
    ├── notification/v1/
    │   └── notification.proto
    ├── analytics/v1/
    │   └── analytics.proto
    ├── promotion/v1/
    │   ├── promotion.proto
    │   └── promotion_http.yaml
    └── events/v1/                      # Kafka event schemas
        ├── identity_events.proto
        ├── membership_events.proto
        ├── payment_events.proto
        ├── workout_events.proto
        ├── booking_events.proto
        ├── checkin_events.proto
        └── promotion_events.proto
```

**Distribution Strategy:**
- `gym-proto` generates both languages for contract verification and publication.
- **Go**: Services consume tagged `github.com/pploc/proto-go`; generated Go stubs are not copied into service repositories.
- **Java**: Services consume tagged `com.gym.proto:gym-proto-java`; generated Java stubs are not copied into service repositories.
- Local `make gen-go`, `make gen-java`, or `make proto` commands preview generated artifacts inside the contract repository only.
- Existing `proto/*/v1/*_http.yaml` files are the HTTP mapping source of truth and are wired into `buf.gen.yaml` via `grpc_api_configuration`; unbound internal methods are not exposed.

---

## 2. Monorepo Structure (`github.com/gym-chain/backend`)

Contains Protobuf files, service implementations, local infrastructure deployment files, and global scripts.

```
gym-chain/
├── gym-proto/                          # Protobuf schemas & configs
│   ├── proto/                          # Protobuf definitions
│   ├── buf.yaml                        # Buf module config
│   └── buf.gen.yaml                    # Code generation template (Go & Java stubs)
├── common-go/                          # Shared Go library
│   ├── go.mod
│   ├── grpc/interceptor/
│   ├── kafka/
│   ├── config/
│   ├── errors/
│   ├── logging/
│   ├── health/
│   ├── pagination/
│   ├── crypto/
│   └── testutil/
│
├── common-java/                        # Shared Java/Gradle library
│   ├── build.gradle
│   ├── gradlew
│   └── src/main/java/com/gym/common/
│       ├── grpc/
│       ├── kafka/
│       ├── error/
│       ├── persistence/
│       ├── pagination/
│       └── config/
│
├── services/
│   ├── ms-gym-identifier/               # Go
│   │   ├── cmd/server/main.go
│   │   ├── internal/
│   │   │   ├── domain/
│   │   │   ├── usecase/
│   │   │   ├── adapter/
│   │   │   └── config/
│   │   ├── migrations/
│   │   ├── Dockerfile
│   │   ├── go.mod
│   │   └── go.sum
│   │
│   ├── ms-gym-member/                 # Java Spring Boot
│   │   ├── src/main/java/com/gym/member/
│   │   │   ├── domain/
│   │   │   ├── application/
│   │   │   ├── adapter/
│   │   │   └── config/
│   │   ├── src/main/resources/
│   │   │   ├── application.yml
│   │   │   └── db/migration/          # Flyway
│   │   ├── Dockerfile
│   │   ├── build.gradle
│   │   └── gradlew
│   │
│   ├── ms-gym-payment/                # Java Spring Boot
│   │   ├── src/main/java/com/gym/payment/
│   │   │   ├── domain/
│   │   │   ├── application/
│   │   │   ├── adapter/
│   │   │   └── config/
│   │   ├── Dockerfile
│   │   ├── build.gradle
│   │   └── gradlew
│   │
│   ├── ms-gym-workout/                # Go
│   │   ├── cmd/server/main.go
│   │   ├── internal/
│   │   ├── migrations/
│   │   ├── Dockerfile
│   │   └── go.mod
│   │
│   ├── ms-gym-trainer/                # Java Spring Boot
│   │   ├── src/main/java/com/gym/trainer/
│   │   ├── Dockerfile
│   │   ├── build.gradle
│   │   └── gradlew
│   │
│   ├── ms-gym-checkin/                # Go
│   │   ├── cmd/server/main.go
│   │   ├── internal/
│   │   ├── migrations/
│   │   ├── Dockerfile
│   │   └── go.mod
│   │
│   ├── ms-gym-notification/           # Go
│   │   ├── cmd/server/main.go
│   │   ├── internal/
│   │   ├── Dockerfile
│   │   └── go.mod
│   │
│   ├── ms-gym-analytics/              # Java Spring Boot
│   │   ├── src/main/java/com/gym/analytics/
│   │   ├── Dockerfile
│   │   ├── build.gradle
│   │   └── gradlew
│   │
│   └── ms-gym-promotion/              # Java Spring Boot
│       ├── src/main/java/com/gym/promotion/
│       ├── Dockerfile
│       ├── build.gradle
│       └── gradlew
│
├── infra/                              # Local infrastructure configs (Docker Compose)
│   ├── local/
│   │   └── init-dbs.sql
│   ├── haproxy/
│   │   └── haproxy.cfg
│   └── kong/
│       └── kong.yml
│
├── .github/
│   └── workflows/
│       ├── common-go-ci.yml            # Test + tag common-go
│       ├── common-java-ci.yml          # Test + publish common-java
│       ├── service-ci.yml              # Per-service build/test/deploy
│       └── infra-validate.yml          # Config validation
│
├── docker-compose.yml                  # Local development (all services + deps)
│   ├── docker-compose.deps.yml             # Dependencies only (DBs, Kafka, Redis)
│   ├── Makefile                            # Local development commands
│   └── docs/                               # Architecture Documentation
```

---

## Makefile (Local Dev Commands)

```makefile
.PHONY: test-go test-java docker-up docker-down docker-deps proto gen-go gen-java

# Protobuf compilation
proto: gen-go gen-java

gen-go:
	cd gym-proto && buf generate --template buf.gen.yaml --path proto/

gen-java:
	cd gym-proto && buf generate --template buf.gen.yaml --path proto/ -o ../common-java/

# Testing
test-go:
	@for svc in identifier workout checkin notification; do \
		echo "Testing ms-gym-$$svc..."; \
		cd services/ms-gym-$$svc && go test -race ./... && cd ../..; \
	done

test-java:
	@for svc in member payment trainer analytics promotion; do \
		echo "Testing ms-gym-$$svc..."; \
		cd services/ms-gym-$$svc && ./gradlew test && cd ../..; \
	done

test-all: test-go test-java

# Docker Compose
docker-deps:
	docker compose -f docker-compose.deps.yml up -d

docker-up:
	docker compose up -d --build

docker-down:
	docker compose down -v
```

---

## docker-compose.deps.yml (Local Dependencies)

```yaml
version: "3.9"

services:
  postgres:
    image: postgres:16
    ports: ["5432:5432"]
    environment:
      POSTGRES_PASSWORD: postgres
    volumes:
      - ./infra/local/init-dbs.sql:/docker-entrypoint-initdb.d/init.sql
      - pgdata:/var/lib/postgresql/data

  cassandra:
    image: cassandra:4.1
    ports: ["9042:9042"]
    volumes:
      - cassdata:/var/lib/cassandra
    environment:
      CASSANDRA_CLUSTER_NAME: gym-cluster

  yugabyte:
    image: yugabytedb/yugabyte:2.20-latest
    ports:
      - "5433:5433"    # YSQL
      - "9000:9000"    # Master UI
    command: >
      bin/yugabyted start
      --daemon=false
      --tserver_flags="ysql_enable_auth=false"

  kafka:
    image: confluentinc/cp-kafka:7.6.0
    ports: ["9092:9092"]
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:29093
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:29093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,CONTROLLER:PLAINTEXT
      CLUSTER_ID: gym-chain-local-001

  schema-registry:
    image: confluentinc/cp-schema-registry:7.6.0
    ports: ["8081:8081"]
    environment:
      SCHEMA_REGISTRY_HOST_NAME: schema-registry
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: kafka:9092
    depends_on: [kafka]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    ports: ["8090:8080"]
    environment:
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092
      KAFKA_CLUSTERS_0_SCHEMAREGISTRY: http://schema-registry:8081
    depends_on: [kafka, schema-registry]

volumes:
  pgdata:
  cassdata:
```