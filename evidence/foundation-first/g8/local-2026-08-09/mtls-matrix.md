# G8 workload mTLS matrix

Proven by `gym-infra/kong/g8-business-check.sh` against `generate-g8-certs.sh` material (SPIFFE SANs + DNS SANs).

| Caller cert | Destination | Method | Expected | G8 E2E |
|---|---|---|---|---|
| ms-gym-identifier (SPIFFE sa/ms-gym-identifier) | Plans :50051 | GetActiveGym | ALLOW (SelectGym path) | PASS (SelectGym NONE then ACTIVE) |
| ms-gym-identifier | Plans :50051 | ResolvePurchasablePlan | PERMISSION_DENIED | PASS |
| ms-gym-member-client (SPIFFE sa/ms-gym-member) | Plans :50051 | ResolvePurchasablePlan | ALLOW (purchase path) | PASS (PurchaseMembership) |
| ms-gym-member-client | Plans :50051 | GetActiveGym | PERMISSION_DENIED | PASS |
| ms-gym-identifier | Member :50051 | GetMembershipStatusByUserId | ALLOW (SelectGym) | PASS |
| (test) identifier client cert + x-user-* claims | Member :50051 | PurchaseMembership | ALLOW ROLE_RESTRICTED CUSTOMER | PASS (grpcurl with claims) |

Public HTTP (Kong → service :8080 only):
- Identifier `/api/v1/auth/*` — public auth + SelectGym
- Plans `/api/v1/gyms`, `/api/v1/plans` — admin catalog; anonymous mutation 401
- Member — no Kong route (purchase proven via internal mTLS grpcurl only)

Internal Plans RPCs unmapped on HTTP (G7 unit + G8 route list).

Certificate generation: `kong/generate-g8-certs.sh`
- CA CN=gym-g8-test-ca (1-day disposable)
- plans serverAuth DNS:ms-gym-plans
- member serverAuth DNS:ms-gym-member
- identifier clientAuth DNS:ms-gym-identifier + URI spiffe://gym.cluster.local/ns/gym-system/sa/ms-gym-identifier
- member-client clientAuth DNS:ms-gym-member + URI spiffe://gym.cluster.local/ns/gym-system/sa/ms-gym-member
