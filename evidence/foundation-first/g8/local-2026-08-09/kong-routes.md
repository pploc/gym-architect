# Kong G8 route capture (declarative)

Source: gym-infra/kong/g8-kong.yml

Services:
- ms-gym-identifier → http://ms-gym-identifier:8080 paths /api/v1/auth
- ms-gym-plans → http://ms-gym-plans:8080 paths /api/v1/gyms, /api/v1/plans
- mock-upstream fixture for membership-gated probe (historical G5 pattern)

Not routed:
- Member HTTP/gRPC
- Plans GetActiveGym / ResolvePurchasablePlan (native gRPC only)

Plugin gym-jwt-claims injects x-user-id, x-user-role, x-gym-id, x-membership-status after JWT validation on protected routes including Plans admin paths.
