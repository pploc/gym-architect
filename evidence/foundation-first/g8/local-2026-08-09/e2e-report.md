# G8 three-service E2E

Command:
  cd gym-infra && ./kong/run-g8.sh
Result: RUN_EXIT:0
Message: G8 three-service business checks passed. Member public HTTP remains intentionally unexposed.

## Positive path (g8-business-check.sh)
1. Register customer via Kong → PENDING_VERIFICATION
2. Capture email verification token from Kafka (schema-seed helper)
3. Verify email → gym-neutral access token (membership_status=NONE, no gym_id)
4. Login → gym-neutral access token
5. Promote SUPER_ADMIN; create ACTIVE gym via Plans HTTP POST /api/v1/gyms
6. Create ACTIVE plan via Plans HTTP POST /api/v1/gyms/{id}/plans (price_vnd=450000)
7. Create CLOSED gym for negative selection
8. Demote CUSTOMER; SelectGym ACTIVE gym → membership NONE
9. PurchaseMembership (Member gRPC, ROLE CUSTOMER claims) provider=MOMO → pending_purchases PENDING @ 450000
10. Fake Payment POST /complete → payment.completed.v1
11. Subscription ACTIVE; pending COMPLETED; snapshots MONTHLY/30/450000
12. SelectGym → membership ACTIVE
13. Fake Payment POST /replay → still one subscription, purchase COMPLETED
14. Mutate plan price to 999999 via Plans HTTP; Member snapshots remain 450000

## Negative path
- anonymous Plans POST /api/v1/gyms → 401
- SelectGym missing gym UUID → HTTP >=400
- SelectGym CLOSED gym → HTTP >=400
- Identifier peer ResolvePurchasablePlan → PermissionDenied
- Member peer GetActiveGym → PermissionDenied

## Fake Payment fixture
- Path: gym-infra/kong/fixtures/fake-payment
- gRPC InitiatePayment (plaintext in G8 only)
- HTTP /complete and /replay publish PaymentCompletedEvent with stable event-id
- Not production ms-gym-payment

## Kong
- Config: kong/g8-kong.yml
- Routes: Identifier auth + Plans gyms/plans; gym-jwt-claims protected_routes include Plans paths
- Member not routed
