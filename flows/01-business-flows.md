# Business Flows

Detailed sequence diagrams for all major business flows in the system.

---

## 1. User Authentication (Normal Login & Token Refresh)

### Normal Login Flow (Email/Password)

```mermaid
sequenceDiagram
    actor U as User (Customer/Trainer/Admin)
    participant APP as Mobile App / Web Client
    participant K as Kong Gateway
    participant IS as Identity Service
    participant DB as PostgreSQL (identity_db)
    participant R as Redis

    U->>APP: Enter Email & Password
    APP->>K: POST /api/v1/auth/login {email, password}
    K->>IS: gRPC Login(LoginRequest)
    
    IS->>DB: SELECT * FROM users WHERE email = ?
    DB-->>IS: User details (role, password_hash, status)
    
    IS->>IS: Verify: status == ACTIVE
    IS->>IS: Verify password: Bcrypt hash check (cost=12)
    
    alt Credentials Valid
        IS->>IS: Generate JWT Access Token (15-min TTL, membership_status claim)
        IS->>IS: Generate Cryptographically Secure Random Refresh Token (7-day TTL)
        IS->>DB: INSERT INTO refresh_tokens (token_hash, user_id, expires_at, revoked=false)
        IS-->>APP: Return {access_token, refresh_token, expires_in=900}
    else Invalid
        IS-->>APP: Return error (401 Unauthorized / INVALID_ARGUMENT)
    end
```

### Stateless JWT Verification & Stateful Blacklist Check

```mermaid
sequenceDiagram
    participant APP as Client
    participant K as Kong Gateway
    participant R as Redis
    participant MS as Member Service

    APP->>K: GET /api/v1/members/me (Bearer JWT)
    K->>K: 1. Validate JWT signature & exp
    K->>R: 2. GET blacklist:{jwt_hash}
    alt Token Blacklisted (Logged out)
        R-->>K: Token exists
        K-->>APP: Return 401 Unauthorized
    else Token Valid
        R-->>K: Token not found
        K->>K: 3. Extract claims & inject as X-User-* headers
        K->>MS: Forward request to Service
        MS-->>APP: Return Response
    end
```

### Token Refresh Flow (with Sync Status Check)

```mermaid
sequenceDiagram
    participant APP as Client
    participant K as Kong Gateway
    participant IS as Identity Service
    participant DB as PostgreSQL (identity_db)
    participant MS as Member Service

    APP->>K: POST /api/v1/auth/refresh {refresh_token}
    K->>IS: gRPC RefreshToken(RefreshTokenRequest)
    
    IS->>DB: SELECT * FROM refresh_tokens WHERE token_hash = SHA256(refresh_token)
    DB-->>IS: RefreshToken record (revoked status, expiration)
    
    IS->>IS: Verify: revoked == false && expires_at > now
    
    alt Token Valid
        IS->>MS: gRPC GetMembershipStatusByUserId(user_id)<br/>(mTLS; verified Identifier peer)
        MS-->>IS: {membership_status: "NONE" | "ACTIVE" | "PAUSED" | "EXPIRED"}
        
        IS->>IS: Generate new JWT Access Token with latest membership_status
        IS->>IS: Generate new Refresh Token (Refresh Token Rotation)
        IS->>DB: Mark old refresh token as revoked = true
        IS->>DB: INSERT new refresh token
        
        IS-->>APP: Return {access_token, refresh_token}
    else Invalid
        IS-->>APP: Return error (401 Unauthorized)
    end
```

---

## 2. Customer Registration & Membership Purchase

```mermaid
sequenceDiagram
    actor C as Customer
    participant APP as Mobile App
    participant K as Kong Gateway
    participant IS as Identity Service
    participant MS as Member Service
    participant PS as Payment Service
    participant PRS as Promotion Service
    participant PP as Momo / ZaloPay
    participant KF as Kafka
    participant NS as Notification Service

    rect rgb(230, 245, 255)
    Note over C,IS: Phase 1: Registration
    C->>APP: Sign up (email + password)
    APP->>K: POST /api/v1/auth/register
    K->>IS: gRPC Register(email, password, gym_id)
    IS->>IS: Hash password (bcrypt)
    IS->>IS: INSERT user (role=CUSTOMER)
    IS->>IS: Generate JWT access + refresh tokens
    IS->>KF: Publish identity.user.registered.v1
    IS-->>APP: {access_token, refresh_token}
    KF-->>MS: Consume identity.user.registered.v1
    MS->>MS: Create member shell (status=NONE)
    end

    rect rgb(255, 245, 230)
    Note over C,PS: Phase 2: Buy Membership
    C->>APP: Select Monthly Plan + enter code GYM20
    APP->>K: POST /api/v1/payments/initiate
    K->>PS: gRPC InitiatePayment

    PS->>PRS: gRPC ValidateAndReserve(code=GYM20, user_id, gym_id)
    PRS-->>PS: {valid: true, discount_percentage: 20, reservation_id}
    PS->>PS: price = 500000 VND - 20% = 400000 VND

    PS->>PS: Create PENDING payment (idempotency_key)
    PS->>PP: Create payment order
    PP-->>PS: {payment_url}
    PS-->>APP: {payment_url, payment_id}

    APP->>PP: Open Momo deeplink
    C->>PP: Confirm payment in Momo
    PP->>PS: Webhook callback (order_id, COMPLETED, signature)
    PS->>PS: Verify HMAC signature
    PS->>PS: Update payment COMPLETED
    PS->>PRS: gRPC ConfirmReservation(reservation_id, payment_id, discount_amount_vnd)
    PS->>KF: Publish payment.completed.v1
    end

    rect rgb(230, 255, 230)
    Note over MS,NS: Phase 3: Activation
    KF-->>MS: Consume payment.completed.v1 (type=MEMBERSHIP)
    MS->>MS: Create subscription status=ACTIVE end_date=today+30d
    MS->>KF: Publish membership.activated.v1

    KF-->>NS: Consume payment.completed.v1
    NS->>NS: Send payment receipt (email)

    KF-->>NS: Consume membership.activated.v1
    NS->>NS: Send welcome SMS
    end
```

---

## 3. Google OAuth Registration

```mermaid
sequenceDiagram
    actor C as Customer
    participant APP as Mobile App
    participant G as Google
    participant IS as Identity Service
    participant KF as Kafka
    participant MS as Member Service

    C->>APP: Tap Sign in with Google
    APP->>G: Google Sign-In SDK
    G-->>APP: Google ID Token
    APP->>IS: POST /api/v1/auth/oauth/google {id_token}
    IS->>G: Verify ID token (tokeninfo endpoint)
    G-->>IS: {sub, email, name, picture}

    alt New user
        IS->>IS: INSERT user (provider=GOOGLE, provider_id=sub)
        IS->>KF: Publish identity.user.registered.v1
        KF-->>MS: Create member shell
    else Existing user
        IS->>IS: Lookup by email found
    end

    IS->>IS: Generate JWT tokens
    IS-->>APP: {access_token, refresh_token}
```

---

## 4. QR Check-in at Gym

### Display provisioning and refresh

```mermaid
sequenceDiagram
    actor A as Admin
    participant K as Kong Gateway
    participant CS as Check-in Service
    participant MS as Member Service
    participant D as Display Kiosk

    A->>K: POST /api/v1/checkin/devices<br/>{gym_id, device_name} (Admin JWT)
    K->>CS: RegisterDevice
    CS->>MS: GetGymLocation(gym_id)<br/>(verified workload channel)
    MS-->>CS: Canonical ACTIVE gym location
    CS->>CS: Provision versioned gym root key and kiosk credential
    CS-->>A: {device_id, api_secret} (returned once)
    A->>D: Install device credential securely

    loop Before current and next payload expire
        D->>K: GET /api/v1/checkin/display/qr<br/>(device credential)
        K->>CS: GetDisplayQrPayload
        CS-->>D: Current + next 60-second signed payloads
        D->>D: Switch QR at the UTC slot boundary
    end
```

### Member scan

```mermaid
sequenceDiagram
    actor M as Member
    participant APP as Mobile App
    participant D as Display Kiosk
    participant K as Kong Gateway
    participant CS as Check-in Service
    participant MS as Member Service
    participant KF as Kafka
    participant AS as Analytics

    Note over D: Display signed QR for the current 60-second UTC slot
    M->>APP: Scan QR shown on kiosk
    APP->>K: POST /api/v1/checkin/scan<br/>{member_id, gym_id, qr_payload} (Bearer JWT)
    K->>K: Validate JWT and rate limit 5/sec per user
    K->>CS: ProcessScan with trusted claims

    CS->>CS: Parse v1 payload and require signed gym == request gym
    CS->>CS: Verify device, key version, current/previous slot,<br/>and HMAC-SHA256 in constant time
    CS->>MS: ValidateMembership(member_id, gym_id)<br/>(verified workload channel)
    MS-->>CS: {valid, status}

    alt Valid QR and active membership
        CS->>CS: INSERT check_in + outbox event atomically
        CS-->>K: {success: true, message: "Check-in thành công"}
        K-->>APP: {success: true, message: "Check-in thành công"}
        CS->>KF: Publish checkin.recorded.v1
        KF-->>AS: Update daily_attendance + member_activity
    else Invalid, expired, revoked, or unauthorized
        CS-->>K: {success: false, message: "Mã QR không hợp lệ"}
        K-->>APP: {success: false, message: "Không hợp lệ"}
    end
```

Check-in owns the root key; no raw QR secret crosses the Member boundary. If the kiosk cannot refresh, it may use only its prefetched next payload and must show unavailable after that payload expires.

---

## 5. Workout Logging

```mermaid
sequenceDiagram
    actor M as Member
    participant APP as Mobile App
    participant K as Kong
    participant WS as Workout Service
    participant KF as Kafka
    participant AS as Analytics

    M->>APP: Start workout from template Push Day
    APP->>K: POST /api/v1/workouts/log
    K->>K: Validate JWT

    K->>WS: gRPC LogWorkout

    WS->>WS: gRPC Interceptor check membership_status == ACTIVE
    alt Not active
        WS-->>APP: PERMISSION_DENIED Active membership required
    end

    WS->>WS: Parse exercises + sets
    WS->>WS: INSERT workout_logs (Cassandra)

    WS->>WS: Check Personal Records
    loop For each exercise
        WS->>WS: best_volume = max(weight x reps)
        WS->>WS: Compare with personal_records table
        alt New PR
            WS->>WS: UPDATE personal_records
        end
    end

    WS->>KF: Publish workout.logged
    WS-->>APP: {workout_id, exercises, new_prs list}
    KF-->>AS: Update member_activity.last_workout_at
```

---

## 6. Trainer Booking Full Flow

```mermaid
sequenceDiagram
    actor C as Customer
    participant APP as Customer App
    participant TS as Trainer Service
    participant PS as Payment Service
    participant PP as Momo / ZaloPay
    participant KF as Kafka
    participant NS as Notification Service
    participant TAPP as Trainer App
    actor T as Trainer

    rect rgb(230, 245, 255)
    Note over C,TS: Search and Select
    C->>APP: Search trainers specialty strength
    APP->>TS: gRPC SearchTrainers(gym_id, specialty)
    TS-->>APP: list of trainers with rates

    C->>APP: View trainer check availability
    APP->>TS: gRPC GetAvailableSlots(trainer_id, date)
    TS-->>APP: [09:00, 10:00, 14:00, 16:00]

    C->>APP: Book 14:00 slot
    APP->>TS: gRPC CreateBooking(trainer_id, slot=14:00)
    TS->>TS: SELECT FOR UPDATE lock slot
    TS->>TS: INSERT booking status=PENDING_PAYMENT
    TS-->>APP: {booking_id, amount 300000 VND}
    end

    rect rgb(255, 245, 230)
    Note over C,PP: Payment
    APP->>PS: InitiatePayment(type=TRAINER_BOOKING, ref=booking_id)
    PS->>PP: Create order
    PP-->>PS: {payment_url}
    PS-->>APP: {payment_url}

    APP->>PP: Open Momo
    C->>PP: Pay
    PP->>PS: Webhook COMPLETED
    PS->>KF: payment.completed.v1 type=TRAINER_BOOKING
    end

    rect rgb(230, 255, 230)
    Note over TS,T: Trainer Approval
    KF-->>TS: Consume payment.completed.v1
    TS->>TS: Update booking REQUESTED

    TS->>KF: Publish booking.requested
    KF-->>NS: Send push to trainer
    NS->>TAPP: New booking request notification
    TAPP->>T: Push notification

    T->>TAPP: Accept booking
    TAPP->>TS: gRPC AcceptBooking(booking_id)
    TS->>TS: Update booking ACCEPTED
    TS->>KF: booking.accepted

    KF-->>NS: Send push to customer
    NS->>APP: Booking confirmed notification
    end

    rect rgb(245, 230, 255)
    Note over T,TS: After Session
    T->>TAPP: Mark session complete + notes
    TAPP->>TS: gRPC CompleteBooking(booking_id, notes)
    TS->>TS: Update booking COMPLETED
    TS->>KF: booking.completed
    end
```

---

## 7. Membership Pause and Resume

```mermaid
sequenceDiagram
    actor C as Customer
    participant APP as Mobile App
    participant MS as Member Service
    participant KF as Kafka
    participant NS as Notification Service

    rect rgb(255, 240, 240)
    Note over C,MS: Pause
    C->>APP: Request pause membership
    APP->>MS: gRPC PauseMembership()
    MS->>MS: Validate status==ACTIVE plan!=LIFETIME pause_count<2

    MS->>MS: remaining_days = end_date - today
    MS->>MS: Update status=PAUSED paused_at=now()
    MS->>KF: Publish membership.paused.v1

    KF-->>NS: Send confirmation SMS

    MS-->>APP: {status PAUSED, remaining_days 18}
    end

    Note over C,MS: ... days pass ...

    rect rgb(230, 255, 230)
    Note over C,MS: Resume
    C->>APP: Resume membership
    APP->>MS: gRPC ResumeMembership()
    MS->>MS: new_end_date = today + remaining_days
    MS->>MS: Update status=ACTIVE end_date=new_end_date paused_at=null

    MS->>KF: Publish membership.resumed.v1
    MS-->>APP: {status ACTIVE, end_date 2025-02-15}
    end
```

---

## 8. Promotion and Notification Fan-Out

```mermaid
sequenceDiagram
    actor A as Admin
    participant DASH as Admin Dashboard
    participant PRS as Promotion Service
    participant KF as Kafka
    participant NS as Notification Service
    participant MS as Member Service
    participant SMS as eSMS API
    participant EMAIL as SendGrid

    A->>DASH: Create promotion code=TET25 25% off gym Q1+Q7
    DASH->>PRS: gRPC CreatePromotion(...)
    PRS->>PRS: INSERT promotion status=DRAFT
    PRS-->>DASH: {promotion_id}

    A->>DASH: Click Publish and Notify
    DASH->>PRS: gRPC PublishPromotion(promotion_id)
    PRS->>PRS: Update status ACTIVE
    PRS->>KF: promotion.published

    KF-->>NS: Consume promotion.published
    NS->>MS: gRPC ListMembersByStatus(ACTIVE, gym_ids)
    MS-->>NS: 500 active members with contact info

    NS->>NS: Filter by notification_preferences
    NS->>NS: Render template for each channel

    par Bounded concurrency 50 goroutines
        NS->>SMS: Send to 350 members SMS opted-in
        NS->>EMAIL: Send to 450 members email opted-in
    end

    NS->>NS: Log results to Cassandra
```

---

## 9. Membership Expiry Warning

```mermaid
sequenceDiagram
    participant CRON as Scheduled Job (Member Service)
    participant MS as Member Service
    participant KF as Kafka
    participant NS as Notification Service
    participant SMS as eSMS
    participant PUSH as FCM

    Note over CRON: Runs daily at 9:00 AM

    CRON->>MS: Find subscriptions where end_date = today + 7 days AND status = ACTIVE

    loop For each expiring member
        MS->>KF: membership.expiring-soon.v1
    end

    KF-->>NS: Consume membership.expiring-soon.v1

    NS->>NS: Load notification template

    par Send via all channels
        NS->>SMS: SMS to member phone
        NS->>PUSH: Push notification to mobile
    end
```

---

## 10. Admin Creates Trainer Account

```mermaid
sequenceDiagram
    actor A as Admin
    participant DASH as Admin Dashboard
    participant IS as Identity Service
    participant KF as Kafka
    participant TS as Trainer Service

    A->>DASH: Create trainer (name, email, gym_id)
    DASH->>IS: gRPC CreateTrainerAccount(email, temp_password, gym_id)
    IS->>IS: INSERT user role=TRAINER
    IS->>KF: identity.user.registered.v1 role TRAINER
    IS-->>DASH: {user_id}

    DASH->>TS: gRPC CreateTrainer(user_id, gym_id, specialties, rate)
    TS->>TS: INSERT trainer profile
    TS->>KF: trainer.created
    TS-->>DASH: {trainer_id}

    Note over A: Trainer receives temp password via email<br/>logs in via Trainer App changes password<br/>uploads avatar + certifications
```

---

## 11. Spending and Coaching History

```mermaid
sequenceDiagram
    actor C as Customer
    participant APP as Customer App
    participant PS as Payment Service

    C->>APP: View Spending History
    APP->>PS: gRPC GetSpendingHistory(user_id, page)
    PS->>PS: SELECT FROM payments WHERE user_id=? AND status=COMPLETED ORDER BY created_at DESC

    PS-->>APP: list of transactions with date type amount description
```

```mermaid
sequenceDiagram
    actor T as Trainer
    participant TAPP as Trainer App
    participant TS as Trainer Service

    T->>TAPP: View Coaching History
    TAPP->>TS: gRPC GetCoachingHistory(trainer_id, page)
    TS->>TS: SELECT FROM bookings WHERE trainer_id=? AND status=COMPLETED ORDER BY scheduled_at DESC

    TS-->>TAPP: list of sessions with date customer duration notes
```
