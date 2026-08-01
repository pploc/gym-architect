# Identity Service

> **Tech:** Go Gin | **DB:** PostgreSQL | **Port:** 50051 (gRPC) / 8080 (REST)

## Responsibilities

- User registration (email/password + Google OAuth2)
- JWT access token (15 min) + refresh token (7 days)
- Role management: `CUSTOMER`, `TRAINER`, `ADMIN`
- Password reset, email verification
- Token refresh, revocation, and Redis-backed logout blacklisting
- User suspension & cascade dispatching

---

## Data Model

```mermaid
erDiagram
    USERS {
        uuid id PK
        uuid gym_id FK
        varchar email UK
        varchar password_hash
        varchar auth_provider "LOCAL | GOOGLE"
        varchar provider_id "Google sub ID"
        varchar role "CUSTOMER | TRAINER | ADMIN"
        varchar status "ACTIVE | SUSPENDED | PENDING_VERIFICATION"
        varchar full_name
        varchar phone
        timestamp created_at
        timestamp updated_at
    }

    REFRESH_TOKENS {
        uuid id PK
        uuid user_id FK
        varchar token_hash
        timestamp expires_at
        boolean revoked
        varchar device_info
        inet ip_address
        timestamp created_at
    }

    USERS ||--o{ REFRESH_TOKENS : "has"
```

---

## Authentication and Registration Flows

### 1. User Registration Flow

```mermaid
sequenceDiagram
    participant C as Mobile App / Web Client
    participant K as Kong Gateway
    participant IS as Identity Service
    participant DB as PostgreSQL (identity_db)
    participant KF as Kafka
    participant MS as Member Service

    C->>K: POST /api/v1/auth/register {email, password, full_name, role, gym_id}
    K->>IS: gRPC Register(RegisterRequest)
    IS->>IS: Validate inputs (email format, strength, etc.)
    IS->>IS: Generate Bcrypt Hash of password (cost factor = 12)
    IS->>DB: INSERT INTO users (email, password_hash, role, status='PENDING_VERIFICATION', ...)
    IS->>KF: Publish event to 'identity.user.registered'
    IS-->>C: AuthResponse (status, verification details)
    
    Note over KF,MS: Async Member Shell Creation
    KF-->>MS: Consume 'identity.user.registered'
    MS->>MS: Create Member Profile shell (status = NONE)
```

### 2. Normal Login Flow (Email/Password)

```mermaid
sequenceDiagram
    participant C as Mobile App / Web Client
    participant K as Kong Gateway
    participant IS as Identity Service
    participant DB as PostgreSQL (identity_db)

    C->>K: POST /api/v1/auth/login {email, password}
    K->>IS: gRPC Login(LoginRequest)
    IS->>DB: SELECT * FROM users WHERE email = ?
    DB-->>IS: User details & password_hash
    IS->>IS: Verify password via Bcrypt comparison
    alt Password Matches & status != SUSPENDED
        IS->>IS: Generate JWT Access Token (15-min TTL)
        IS->>IS: Generate cryptographically secure random Refresh Token (7-day TTL)
        IS->>IS: Hash Refresh Token using SHA256
        IS->>DB: INSERT INTO refresh_tokens (user_id, token_hash, expires_at, revoked=false, ...)
        IS-->>C: AuthResponse {access_token, refresh_token}
    else Password Mismatch or User Suspended
        IS-->>C: Error (401 Unauthorized / INVALID_ARGUMENT)
    end
```

### 3. Token Refresh Flow (with Sync Membership Status Check)

```mermaid
sequenceDiagram
    participant C as Mobile App / Web Client
    participant K as Kong Gateway
    participant IS as Identity Service
    participant DB as PostgreSQL (identity_db)
    participant MS as Member Service (gRPC)

    C->>K: POST /api/v1/auth/refresh {refresh_token}
    K->>IS: gRPC RefreshToken(RefreshTokenRequest)
    IS->>IS: Compute SHA256(refresh_token)
    IS->>DB: SELECT * FROM refresh_tokens WHERE token_hash = ?
    DB-->>IS: RefreshToken record
    IS->>IS: Validate: expires_at > now AND revoked == false
    alt Token Valid
        IS->>MS: gRPC GetMembershipStatus(user_id)
        MS-->>IS: {membership_status: "ACTIVE" | "EXPIRED" | "PAUSED"}
        IS->>IS: Generate new JWT Access Token containing latest membership_status
        IS->>IS: Generate new Refresh Token (Refresh Token Rotation)
        IS->>DB: Mark old refresh token as revoked = true
        IS->>DB: INSERT new refresh token
        IS-->>C: AuthResponse {access_token, refresh_token}
    else Token Invalid / Revoked
        IS-->>C: Error (401 Unauthorized / Unauthenticated)
    end
```

---

## JWT Structure

```json
{
  "sub": "550e8400-e29b-41d4-a716-446655440000",
  "role": "CUSTOMER",
  "gym_id": "gym-uuid-here",
  "membership_status": "ACTIVE",
  "exp": 1700000000,
  "iat": 1699999100,
  "kid": "key-id-2024"
}
```

- `membership_status` is cached in the JWT for speed.
- **Force-Refresh Mechanism:** When a customer completes a membership purchase, the Mobile App receives the success screen. The app immediately calls `POST /api/v1/auth/refresh` using its stored refresh token. The Identity Service queries the Member Service via gRPC, fetches the new `ACTIVE` status, generates a new Access Token with `"membership_status": "ACTIVE"`, and returns it. This bypasses the 15-minute caching latency.

---

## Token Blacklisting (Logout)

When a user logs out:
1. Access token is placed in Redis: `blacklist:{access_token_hash}` with TTL = remaining token lifetime.
2. Kong Gateway's custom Redis-lookup plugin checks every incoming request's token hash. If found in Redis, Kong rejects immediately with 401 Unauthorized (stateless validation with stateful revocation fallback).

---

## RBAC Matrix

| Route / RPC | Public | Customer | Trainer | Admin | Description |
|-------------|:---:|:---:|:---:|:---:|-------------|
| `Register` / `Login` | ✓ | | | | Auth endpoints |
| `GetCurrentUser` | | ✓ | ✓ | ✓ | Get logged in profile |
| `LogWorkout` / `GetMyQR` | | ✓ | | | Customer features (active membership checked) |
| `AcceptBooking` / `SetAvailability` | | | ✓ | | Trainer operations |
| `CreateTrainerAccount` | | | | ✓ | Admin registers new staff |
| `SuspendUser` | | | | ✓ | Suspend any account |
| `CreatePromotion` | | | | ✓ | Admin management |

---

## SuspendUser Cascade Flow

When an Admin suspends a user:
1. Identity Service marks user status as `SUSPENDED` in PostgreSQL.
2. Identity Service revokes all active `REFRESH_TOKENS` for that user.
3. Identity Service publishes `identity.user.suspended` event to Kafka.
4. **Member Service** consumes event → Sets member status to `SUSPENDED`, cancels any active subscriptions.
5. **Trainer Service** consumes event:
   - If user was a Trainer → Sets trainer status to `SUSPENDED`, cancels all future bookings, publishes `booking.cancelled` (triggering full refunds via Payment Service).
   - If user was a Customer → Cancels all future bookings for that customer, publishes `booking.cancelled` (refunds triggered).

---

## Google OAuth2 Flow

```mermaid
sequenceDiagram
    participant C as Mobile App
    participant G as Google
    participant IS as Identity Service

    C->>G: Google Sign-In SDK → get ID Token
    G-->>C: Google ID Token
    C->>IS: POST /api/v1/auth/oauth/google {id_token}
    IS->>G: Verify ID token (Google tokeninfo endpoint)
    G-->>IS: {sub, email, name, picture}
    IS->>IS: Find or create user (provider=GOOGLE, provider_id=sub)
    IS->>IS: Generate JWT access + refresh tokens
    IS-->>C: {access_token, refresh_token}
```

---

## Kafka Events Published

| Topic | Key | Payload | Consumed By |
|-------|-----|---------|-------------|
| `identity.user.registered` | `user_id` | `{user_id, email, full_name, role, gym_id}` | Member Service (create member profile) |
| `identity.user.role-changed` | `user_id` | `{user_id, old_role, new_role, gym_id}` | — |
| `identity.user.suspended` | `user_id` | `{user_id, role, gym_id}` | Member Service, Trainer Service |

---

## API (gRPC)

```protobuf
service IdentityService {
  // Public
  rpc Register(RegisterRequest) returns (AuthResponse);
  rpc Login(LoginRequest) returns (AuthResponse);
  rpc LoginWithGoogle(GoogleLoginRequest) returns (AuthResponse);
  rpc RefreshToken(RefreshTokenRequest) returns (AuthResponse);
  rpc Logout(LogoutRequest) returns (google.protobuf.Empty);

  // Authenticated
  rpc GetCurrentUser(google.protobuf.Empty) returns (UserResponse);
  rpc ChangePassword(ChangePasswordRequest) returns (google.protobuf.Empty);

  // Admin only
  rpc CreateTrainerAccount(CreateTrainerRequest) returns (UserResponse);
  rpc SuspendUser(SuspendUserRequest) returns (google.protobuf.Empty);
  rpc ListUsers(ListUsersRequest) returns (ListUsersResponse);
}
```

---

## Clean Architecture Layers

```
cmd/server/main.go                    ← bootstrap, wire dependencies
internal/
├── domain/
│   ├── user.go                       ← User entity, Role enum
│   ├── token.go                      ← RefreshToken entity
│   └── errors.go                     ← ErrInvalidCredentials, ErrUserExists
├── usecase/
│   ├── register.go                   ← RegisterUseCase
│   ├── login.go                      ← LoginUseCase
│   ├── refresh_token.go              ← RefreshTokenUseCase
│   └── port/
│       ├── user_repo.go              ← UserRepository interface
│       ├── token_repo.go             ← TokenRepository interface
│       ├── password_hasher.go        ← PasswordHasher interface
│       ├── token_generator.go        ← JWTGenerator interface
│       └── event_publisher.go        ← EventPublisher interface
├── adapter/
│   ├── grpc/
│   │   ├── handler.go                ← implements IdentityServiceServer
│   │   └── mapper.go                 ← proto <-> domain mapping
│   ├── repository/
│   │   ├── postgres_user.go          ← implements UserRepository
│   │   └── redis_token.go            ← implements TokenRepository
│   ├── security/
│   │   ├── bcrypt_hasher.go          ← implements PasswordHasher
│   │   └── jwt_generator.go          ← implements JWTGenerator
│   ├── kafka/
│   │   └── event_publisher.go        ← implements EventPublisher
│   └── oauth/
│       └── google_verifier.go        ← Google ID token verification
└── config/
    └── config.go                     ← env-based config loading
```
