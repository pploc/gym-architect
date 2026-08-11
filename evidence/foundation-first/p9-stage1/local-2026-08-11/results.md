# Phase 9 Stage 1 candidate results

| Gate | Result | Evidence |
|---|---|---|
| 27 active inline mappings | PASS | `verify-http-config.py` exact allowlist, no active YAML duplicate |
| Four workload RPCs unbound | PASS | `verify-http-config.py` |
| grpc-gateway routes | PASS | `verify-generated-routes.sh` |
| Gnostic named body | PASS | Purchase body references `member.v1.PurchaseMembershipBody` |
| Protobuf JSON shape | PASS | lowerCamelCase JSON fields; symbolic enums; `priceVnd` string |
| OpenAPI URI placeholders | PASS | direct Gnostic output uses lowerCamel Protobuf JSON names; runtime annotations remain snake_case |
| OpenAPI bearer documentation | PASS | native Gnostic annotations leave public operations open and require `BearerAuth` for protected operations |
| OpenAPI success examples | PASS | native response-schema examples use concrete enum values, never `*_UNSPECIFIED` |
| OpenAPI service split | PASS | direct tracked documents: Identity 12, Member 7, Plans 8; no cross-service operations |
| OpenAPI determinism | PASS | three byte-identical source-relative documents; hashes in `candidate-artifacts.sha256` |
| Kong source bundle determinism | PASS | `1983b106c2c6a82240371937d4833d970f2bf02a2a9fab38886a487cf1204fe6` |
| Kong 3.8 source parse | PASS | disposable Member and Plans source-root smoke |
| Kong error release gate | PASS | candidate gate passes; `--release` deliberately rejects pending evidence |
| `buf format`, lint, breaking against `v5.0.0` | PASS | local command suite |
| gym-proto `./gradlew check --no-daemon` | PASS | build successful |
| Stage 0 Kong gateway tests | PASS | isolated fixture `go test -v ./...` |
| Full G8 topology | BLOCKED, unrelated | Confluent package download failed in `schema-seed`; no Stage 1 code executed |
| v6/Go release conflicts | PASS | candidate target tags/releases absent |
| Publication | NOT RUN | blocked until Stage 2–3 measured Kong compatibility |

## Generator naming decision

Gnostic `v0.7.1` couples path-parameter formatting with Protobuf JSON naming. Direct `naming=json` output emits `{gymId}` from runtime annotation `{gym_id}` and preserves required lowerCamel JSON schemas. `naming=proto` would incorrectly convert JSON schemas to snake_case. Runtime behavior remains defined only by snake_case `google.api.http` annotations and deferred configuration; generated OpenAPI uses lowerCamel documentation placeholder labels. Native Gnostic annotations define Bearer documentation and response-schema examples. `output_mode=source_relative` emits one tracked document per service. `gen/` remains ignored for disposable language stubs; no post-generation normalizer, merger, or YAML rewrite exists.

No runtime error response or CORS claim is made here.
