# Phase 10 final evidence

> **Status:** Technical complete on 2026-08-23. Final proof uses immutable released artifacts, detached locked sources, protected CI, and sanitized evidence only. Accountable-owner acceptance is recorded separately below.

## Release and lock

| Item | Value |
|---|---|
| Contract source | `gym-proto v7.0.2` at `8da83a33411d10442a854fe1d05d34356fe85643` |
| Contract release | <https://github.com/pploc/gym-proto/releases/tag/v7.0.2> |
| Java package | `com.gym.proto:gym-proto-java:7.0.2` |
| Go module | `github.com/pploc/proto-go@v1.7.1` |
| Common libraries | `com.gym:common-java:3.0.0`, `github.com/pploc/common-go@v0.5.0` |
| Runtime lock tip | [`gym-infra@300cd1c14f628ceefd8df27b6e53f3e172083cbb`](https://github.com/pploc/gym-infra/commit/300cd1c14f628ceefd8df27b6e53f3e172083cbb) |
| Lock content SHA | `085f2c8fdc82348e31b4fafc0fec05ec5fd2e174` (closeout source restaged by the tip chore commit) |
| Lock file SHA-256 | `b06e64c0d9645204b1348298ca0b726fe0f5439b8c3f3bfed92fb68dc43ff552` |
| Generated gateway image | `ghcr.io/pploc/ms-gym-api-gateway@sha256:eae747494b85f5de9ba4a915209625999be09dae7ed45cfc898999644fa6b9f2` |
| Check-in image | `ghcr.io/pploc/ms-gym-checkin@sha256:6fbaf44bc9604e580341cf18b6388bcc913886cf6305fb2f0c13a4dad21b8efe` |
| Member image | `ghcr.io/pploc/ms-gym-member@sha256:cfa7e4cceaa1cb0fcda7b3c0b11aaaa399c7bb92584a679a0e105bb4657decbc` |
| Plans image | `ghcr.io/pploc/ms-gym-plans@sha256:5de9615e8f95685833677311150f8d37cd1c308b499720c58869261cdbcc77f1` |
| Identifier image | `ghcr.io/pploc/ms-gym-identifier@sha256:0d6466c143bc905bb12d5b8d4f24f2188e561bc798771e6dbe60ad98509a6de5` |
| Kong image | `kong:3.8-ubuntu@sha256:250cc9745fde8ce04be060bc8e4338dfc280f54986cb38ef24d03876cb6b6e2f` |
| Route inventory | 33 exact routes (identity 12, member 7, plans 8, checkin 6) |

The lock materializes detached repositories into a temporary mode-0700 workspace, verifies released assets and digest-pinned images, then runs only those images. Private sources, certificates, credentials, fixture state, and diagnostics are removed on exit.

## Protected gate and sanitized runtime evidence

| Gate | Result | Record |
|---|---|---|
| Contract release publication | PASS | [gym-proto run 32372731016](https://github.com/pploc/gym-proto/actions/runs/32372731016) |
| Check-in protected CI | PASS | [ms-gym-checkin run 32371263336](https://github.com/pploc/ms-gym-checkin/actions/runs/32371263336) |
| Member protected CI | PASS | [ms-gym-member run 32370532909](https://github.com/pploc/ms-gym-member/actions/runs/32370532909) |
| Plans protected CI | PASS | [ms-gym-plans run 32371259239](https://github.com/pploc/ms-gym-plans/actions/runs/32371259239) |
| Authenticated locked G10 | PASS | [gym-infra run 32651525475](https://github.com/pploc/gym-infra/actions/runs/32651525475) |
| PR required G10 gate | PASS | [gym-infra run 32652039056](https://github.com/pploc/gym-infra/actions/runs/32652039056) |
| Prior outbox publish proof | PASS | [gym-infra run 32651239509](https://github.com/pploc/gym-infra/actions/runs/32651239509) |
| Develop merge | PASS | [PR #1](https://github.com/pploc/gym-infra/pull/1) squash-merged as [`07f71c7f5002285663983e7594174a94a48e757a`](https://github.com/pploc/gym-infra/commit/07f71c7f5002285663983e7594174a94a48e757a) |
| Sanitized G10 artifact | PASS | artifact `g10-sanitized-789cdf0285cef652b57bebf079ebf89ba9aa8eb7`; SHA-256 `1d0ea85c9414e962e2218c68878efa4ca529c8873a5a0e996e2906e6b7a901b9` |
| Develop required check | PASS | `G10 locked fixture gate` required, `enforce_admins=true`, linear history, force push disabled |

Committed sanitized evidence: [`g10-sanitized-evidence.yaml`](g10-sanitized-evidence.yaml).

The protected gate passed lock validation, detached source materialization, certificate generation, Kong rendering, compose startup, runtime readiness, the business matrix, and evidence sanitization. Private failure diagnostics were skipped because no failure occurred.

Sanitized artifact schema validation confirms 23 named cases. All expected statuses matched. It records checksums and labels only: no request values, response bodies, JWTs, Authorization values, private keys, AWS credentials, KMS plaintext or ciphertext, PII, fixture IDs, raw QR/event payloads, raw logs, stack traces, or transport details.

## Browser and Check-in contract

Selected runtime topology:

```text
Browser HTTPS/JSON
  -> Kong JWT/CORS/exact routes
  -> mTLS generated Go grpc-gateway
  -> mTLS Check-in / Member / Plans gRPC :50051
```

Named locked-matrix cases that passed:

- route/auth negatives for health, unknown route, missing JWT, direct gRPC path, trailing slash
- empty self-history, SUPER_ADMIN display QR, customer display forbidden
- inactive membership scan fail-closed, positive scan, idempotent replay, idempotency conflict
- durable `checkin.recorded.v1` outbox `PUBLISHED` and single-publish after replay
- distinct idempotency key scan plus history/count updates
- admin scan forbidden, invalid QR rejected
- daily count and member history allow/deny
- QR root-key rotate customer forbidden, normal rotate, scan after rotate, emergency rotate

Service-level Check-in suites additionally cover KMS fail-closed readiness, QR grammar/expiry/noncanonical rejection, outbox retry/DLQ, and Yugabyte constraints. G10 fixture deliberately seeds ACTIVE membership by SQL because the locked compose has no fake-payment purchase path.

## Consumer immutable pins

| Repository | Recorded commit |
|---|---|
| gym-proto | [`8da83a33411d10442a854fe1d05d34356fe85643`](https://github.com/pploc/gym-proto/commit/8da83a33411d10442a854fe1d05d34356fe85643) |
| Identifier | [`def7936c9e54f489aad5aab246b9300697969d90`](https://github.com/pploc/ms-gym-identifier/commit/def7936c9e54f489aad5aab246b9300697969d90) |
| Member | [`989ee3284ea584f1c4876ab0e29143e8158f88c4`](https://github.com/pploc/ms-gym-member/commit/989ee3284ea584f1c4876ab0e29143e8158f88c4) |
| Plans | [`c0b9f16284c5605032267440bb478d04cebb30d8`](https://github.com/pploc/ms-gym-plans/commit/c0b9f16284c5605032267440bb478d04cebb30d8) |
| Check-in | [`e26e2089338808f53706e91ee575607646bf631f`](https://github.com/pploc/ms-gym-checkin/commit/e26e2089338808f53706e91ee575607646bf631f) |
| Infrastructure tip | [`300cd1c14f628ceefd8df27b6e53f3e172083cbb`](https://github.com/pploc/gym-infra/commit/300cd1c14f628ceefd8df27b6e53f3e172083cbb) |

## Owner acceptance

Technical gates passed on 2026-08-23. Accountable-owner acceptance: **approved** under the standing G10 pre-authorization after those gates.

## Final tracked-tree check

Recorded product commits used by the lock have no tracked modifications required for this closeout. Local ignored/untracked tooling such as `gym-infra/.claude/` is excluded from this claim.

G9 final evidence remains historical and unchanged under [`../p9-final/`](../p9-final/README.md).
