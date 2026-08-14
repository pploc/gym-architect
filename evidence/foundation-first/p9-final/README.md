# Phase 9 final evidence

> **Status:** Not produced. Do not treat Stage 3 or Stage 4 local fixture output as final G9 proof.

Create this evidence only after all Phase 9 gates pass from immutable released inputs and clean committed source trees:

- `gym-proto v6.0.1`, Java `6.0.1`, Go `v1.6.1`, canonical merged OpenAPI, and release checksums are published;
- lock materializes every required repository at detached commit and uses digest-pinned generated gateway and Kong images;
- protected authenticated G9 CI passes, including TypeScript generation from released OpenAPI and `tsc --noEmit`;
- `run-g9.sh` passes from clean infrastructure checkout with no sibling repositories;
- all recorded product repositories are clean.

Commit only schema-controlled sanitized summaries. Include detached repository SHAs, artifact/image/route/template checksums, 27-route status summary, certificate subject/issuer/SAN/expiry/fingerprint, command labels and exit codes, direct Plans `404`, Actuator `200`, safe `500`/`503`, and CI/release URLs.

Never commit private keys, JWTs, Authorization values, tokens, PII, fixture identifiers, request values, raw logs, raw Protobuf payloads, or raw stack/transport details.
