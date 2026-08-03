# G2 foundation-library evidence

This directory is the durable, sanitized index for one G2 review. It is not a claim that G2 has passed.

## Status

- Technical status: **passed** for the exact working trees recorded in [`3297115bf333a3781e4248535a7c965d14cb0388--82f7ab7fb7d6e26fa6db3f076f707883b87e7d39.md`](3297115bf333a3781e4248535a7c965d14cb0388--82f7ab7fb7d6e26fa6db3f076f707883b87e7d39.md).
- Owner authorization: recorded for this technical G2 closeout; legal POM identity metadata remains deferred.
- Scope: independent `common-java` and `common-go` verification against published `gym-proto` / `proto-go v1.1.0`.
- Excluded: RC tags, common-library package publication, external common-library consumers, and the bidirectional G3 matrix.

## Record template

Create one reviewed record named `<common-java-sha>--<common-go-sha>.md` after both exact source SHAs pass. It must contain:

```text
common-java source SHA:
common-go source SHA:
gym-proto tag / SHA:
proto-go version:
fixture SHA-256:
Kafka / Schema Registry version:
run date and CI URLs:

Java commands and outcomes:
- ./gradlew clean check jacocoTestReport jacocoTestCoverageVerification --no-daemon
- ./gradlew kafkaContractIntegration --no-daemon
- published gym-proto dependency resolution without mavenLocal()
- dependency/security report outcome

Go commands and outcomes:
- make verify
- make integration
- go list -m -json github.com/pploc/proto-go (no Replace field)

Live evidence hashes and observations:
- all nine fixture cases
- Registry before/after snapshot hashes
- source/DLQ key, frame, and ordered-header SHA-256 values
- offsets and retry/commit/DLQ/redelivery observations
- coverage, race, static analysis, and vulnerability outcomes

Review decision: pending | pass | fail
Known exceptions and follow-up:
```

Link uploaded repository workflow artifacts instead of copying raw runtime material here. Never record credentials, Registry URLs with credentials, payload bytes, user data, tokens, or unredacted headers.
