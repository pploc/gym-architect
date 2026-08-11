# Results

| Gate | Result |
|---|---|
| Plans-local peer SAN extraction | PASS — exact DNS type 2 / URI type 6 leaf SAN matching; missing TLS, certificate, SAN, or parse failure denies |
| Claim-bearing Plans gRPC peer | PASS — approved Kong DNS/SPIFFE SAN only |
| Kong SPIFFE form | PASS — `spiffe://gym.cluster.local/ns/gym-system/sa/kong` |
| Workload matrix | PASS — Identifier only for `GetActiveGym`; Member only for `ResolvePurchasablePlan` |
| Swapped workloads and Kong on workload RPCs | PASS — `PERMISSION_DENIED` |
| CA-valid forged claims from Identifier, Member, Check-in, Notification, Postman | PASS — `PERMISSION_DENIED` before handler |
| Missing/conflicting claims from Kong | PASS — shared auth returns `UNAUTHENTICATED` |
| Plaintext/no-client-certificate transport attempt | PASS — connection fails |
| Focused unit and live mTLS tests | PASS — `./gradlew test --tests 'com.gym.plans.unit.config.*' --tests 'com.gym.plans.integration.PlansMtlsWorkloadIntegrationTest' --no-daemon` |
| Environment lifecycle | PASS — `./gradlew startEnv`; PostgreSQL started |
| Full Plans quality gate | PASS — `./gradlew clean check` |
| Environment cleanup | PASS — `./gradlew stopEnv`; PostgreSQL stopped |
| Diff whitespace check | PASS — `git diff --check` |

The local certificate generator output is ignored and disposable. No certificate bodies, private keys, P12 files/passwords, or tokens are recorded.

Stage 3 remains required for real Kong 3.8 route/transcoding, upstream mTLS, reflection exposure, CORS, and HTTP error/header/body compatibility. Stage 4 remains required for Plans MVC removal and direct `:8080/api/**` rejection.
