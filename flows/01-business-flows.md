# Business Flows

> **Scope:** G0–G9 evidence remains historical. G10 is in progress with Stage 2 Check-in implementation underway with stable identity, a logged-in `SUPER_ADMIN` iPad QR display, live Member validation, and Plans-owned gym validation. Production Payment, Workout, Trainer, Notification, Analytics, and Promotion remain deferred.

## 1. Stable Identity Authentication

```mermaid
sequenceDiagram
    actor U as User
    participant C as Client
    participant K as Kong
    participant ID as Identifier
    participant DB as identity_db

    U->>C: Enter credentials
    C->>K: POST /api/v1/auth/login
    K->>ID: Login
    ID->>DB: Load active user
    ID->>ID: Verify password
    ID->>DB: Store hashed rotating refresh token
    ID-->>C: Access token + refresh token
    Note over ID,C: JWT contains identity/role; no gym_id or membership_status
```

Registration, email verification, Google login, and refresh use the same stable access-token shape. `identity.user.registered.v1` remains gym-neutral. Member consumes it to create a profile shell with `NONE` aggregate status.

The client does not call Identifier when choosing a gym. `POST /api/v1/auth/gym`, selected-gym token issuance, and Identifier-to-Member `GetMembershipStatusByUserId` are removed in Phase 9 Stage 0.

## 2. Browse, Select Gym, and Purchase

```mermaid
sequenceDiagram
    actor C as Customer
    participant APP as Client
    participant K as Kong
    participant ID as Identifier
    participant PL as Plans
    participant MB as Member
    participant DB as member_db
    participant FP as G8 Fake Payment
    participant KF as Kafka

    rect rgb(230,245,255)
        C->>APP: Register and verify email
        APP->>K: POST /api/v1/auth/login
        K->>ID: Login
        ID-->>APP: Stable identity JWT
    end

    rect rgb(245,240,255)
        APP->>K: GET /api/v1/gyms
        K->>PL: ListGymLocations
        PL-->>APP: Authenticated gym catalog
        C->>APP: Select gym
        Note over C,APP: gym_id stays in URL/UI state
        APP->>K: GET /api/v1/gyms/{gym_id}/plans
        K->>PL: ListMembershipPlans(gym_id)
        PL-->>APP: Plans for selected gym
    end

    rect rgb(255,245,230)
        C->>APP: Choose plan and provider
        APP->>K: POST /api/v1/gyms/{gym_id}/memberships/purchase
        K->>MB: PurchaseMembership(gym_id, purchase body) with verified sub/role
        MB->>MB: Resolve member from sub; reject nonblank discount_code
        MB->>PL: ResolvePurchasablePlan(gym_id, plan_id) over mTLS
        PL-->>MB: Canonical active gym, plan, type, duration, price_vnd
        MB->>DB: Create/load PENDING purchase by (user_id, idempotency_key)
        Note over MB,DB: Commit before Payment; stable purchase_id
        MB->>FP: InitiatePayment(reference_id=purchase_id)
        FP-->>MB: payment_id, payment_url
        MB->>DB: Attach payment_id
        MB-->>APP: payment_id, payment_url
    end

    rect rgb(230,255,230)
        FP->>KF: payment.completed.v1 reference_id=purchase_id
        KF-->>MB: Completion event
        MB->>DB: Claim event + lock purchase in one transaction
        MB->>DB: Activate from frozen terms and mark completed
        MB->>KF: membership.activated.v1 via outbox
    end
```

Rules:

- Client-selected `gym_id` is intent, not authority.
- Client never supplies authoritative plan type, duration, price, or membership state.
- Plans proves the plan belongs to the selected active gym and is active.
- Member derives user identity from Kong-verified metadata and checks ownership.
- Required `idempotency_key` reuses one purchase and Payment reference.
- Optional `discount_code` remains in the contract; nonblank values fail until Promotion exists.
- Completion validates purchase, payment, user, gym, type, provider, amount, and state.
- Completion uses frozen terms and never rereads Plans.
- Public traffic is HTTPS/JSON through Kong; Kong uses mTLS gRPC to Member and Plans.

## 3. Membership Status, Pause, and Resume

Gym-specific lifecycle operations use explicit resource paths and Member-owned subscription state:

```mermaid
sequenceDiagram
    actor C as Customer
    participant K as Kong
    participant MB as Member
    participant KF as Kafka

    C->>K: GET /api/v1/gyms/{gym_id}/members/{member_id}/membership
    K->>MB: GetMembershipStatus(gym_id, member_id), verified sub/role
    MB->>MB: Require customer owns member and subscription belongs to gym
    MB-->>C: Live gym-specific membership status

    C->>K: POST .../membership:pause
    K->>MB: PauseMembership(gym_id, member_id)
    MB->>MB: Require ACTIVE and snapshot type != LIFETIME
    MB->>MB: Save remaining days and PAUSED state
    MB->>KF: membership.paused.v1
    MB-->>C: PAUSED

    C->>K: POST .../membership:resume
    K->>MB: ResumeMembership(gym_id, member_id)
    MB->>MB: Require PAUSED subscription in requested gym
    MB->>MB: Restore end date from saved remaining days
    MB->>KF: membership.resumed.v1
    MB-->>C: ACTIVE
```

Lifecycle uses subscription snapshots and does not call Plans. JWT membership state is never consulted. Aggregate member status remains derived across subscriptions; gym-specific reads use the selected subscription.

## 4. Administrative Gym Scope

No authoritative `ADMIN`-to-gym assignment model exists. Request `gym_id` cannot replace that model.

During G9:

- Plans mutations are `SUPER_ADMIN`-only;
- Member gym-wide listing and administration are `SUPER_ADMIN`-only;
- Identifier trainer-account creation with gym context is `SUPER_ADMIN`-only;
- authenticated users may browse gyms and plans;
- customer membership operations remain self/ownership scoped.

A future staff-assignment boundary must define ownership, persistence, revocation, lookup, and tests before gym-scoped `ADMIN` access returns.

## 5. Planned G10 QR Check-in

G10 is in progress and Stage 2 Check-in implementation is underway. Plans owns locations; Member owns member identity and membership decisions. The QR display is a simple iPad app used after the gym owner logs in. G10 has no kiosk registration, device secret, HTTP Basic flow, `device_id`, or independent display revocation.

### Display flow

```mermaid
sequenceDiagram
    actor O as Gym owner / Super admin
    participant IP as iPad app
    participant K as Kong
    participant GW as Generated gateway
    participant CS as Check-in
    participant PL as Plans
    participant KMS as AWS KMS
    participant DB as checkin_db

    O->>IP: Log in and select gym
    IP->>K: GetDisplayQrPayload(gym_id) + stable JWT
    K->>GW: Validate JWT; forward verified sub/role
    GW->>CS: Display request with explicit gym_id
    CS->>CS: Require SUPER_ADMIN
    CS->>PL: ValidateCheckInGym(gym_id) over Check-in mTLS
    PL-->>CS: Active canonical gym
    CS->>DB: Load or create current encrypted key version
    CS->>KMS: Encrypt new 32-byte key or decrypt on bounded-cache miss
    CS-->>IP: Current and next 60-second signed QR payloads
    IP-->>O: Full-screen rotating QR
```

`gym_id` is explicit resource context, not authorization proof. G10 uses `SUPER_ADMIN` because no authoritative owner/admin-to-gym assignment exists.

### Scan and outbox flow

```mermaid
sequenceDiagram
    actor M as Member
    participant APP as Mobile app
    participant IP as iPad display
    participant K as Kong
    participant GW as Generated gateway
    participant CS as Check-in
    participant DB as checkin_db
    participant MB as Member
    participant KF as Kafka

    IP-->>APP: Signed 60-second gym/key QR
    M->>APP: Scan
    APP->>K: ProcessScan(gym_id, qr_payload, idempotency_key) + JWT
    K->>GW: Validate JWT; forward verified sub/role
    GW->>CS: Generated gRPC request
    CS->>CS: Derive user_id from sub; verify signed gym, key, slot, and HMAC
    CS->>DB: Resolve (user_id, idempotency_key)
    CS->>MB: ValidateMembership(user_id, signed_gym_id) over Check-in mTLS
    MB-->>CS: canonical member_id, valid, live status
    alt Active membership
        CS->>DB: Insert check-in, idempotency result, and outbox atomically
        CS-->>APP: Stored success
        CS->>KF: Relay checkin.recorded.v1 at least once
    else Invalid or conflicting request
        CS-->>APP: Safe categorized failure
    end
```

Check-in never trusts JWT `membership_status`, never calls Member for location data, never lets the client choose canonical `member_id`, and never treats request `gym_id` alone as authority. Exact canonical replay returns the original success; changed input under the same key conflicts; distinct keys may record distinct check-ins.

## 6. Deferred Catalog Flows

These remain designs, not G10 implementation commitments:

- Workout logging and explicit Member membership validation
- Trainer search, availability, booking, payment, and approval
- Promotion publication and notification fan-out
- Membership expiry notifications
- Production Payment provider webhooks, refunds, and history
- Analytics projections and dashboards

## 7. Admin Creates Trainer Account

Only Identifier credential creation is active. Trainer profile and staff-to-gym assignment remain deferred.

```mermaid
sequenceDiagram
    actor A as Super admin
    participant ID as Identifier
    participant PL as Plans
    participant TS as Future Trainer

    A->>ID: CreateTrainerAccount(email, temp_password, gym_id)
    ID->>PL: GetActiveGym(gym_id) over Identifier mTLS
    PL-->>ID: Active gym
    ID->>ID: Create user with TRAINER role
    ID-->>A: user_id
    Note over A,TS: Gym assignment/profile creation is deferred
```

Identifier retains Plans `GetActiveGym` for this validation, so Identifier-to-Plans is not removed with the selected-gym customer flow. The requested gym is not stored in the stable JWT or identity event.
