# Phase 9 — Stable Identity, Kong gRPC-Gateway, and Generated OpenAPI 3.0

> **Status:** planned. Execute only after the active `gym-proto` generation is published and all three active services resolve the same contract generation.

## Objective

Reach G9 by exposing the public Member and Plans CRUD APIs as HTTPS/JSON at Kong while keeping Member and Plans as private mTLS gRPC services.

Phase 9 must:

- use Kong's bundled `grpc-gateway` plugin for unary HTTP/JSON-to-gRPC transcoding;
- generate canonical OpenAPI 3.0 directly from the Protobuf contract with pinned `protoc-gen-openapiv3`;
- remove Plans' native `/api/**` Spring MVC business endpoints;
- keep Plans Actuator health on `8080`;
- require Kong's mTLS SAN before Member or Plans trusts end-user claim metadata;
- keep workload-only RPCs absent from Kong and OpenAPI;
- migrate customer gym selection to explicit request resource context before generating routes;
- issue stable identity JWTs containing no `gym_id` or `membership_status`;
- remove the Identifier-to-Member selected-gym lookup and its dead internal RPC;
- preserve direct workload mTLS for Identifier-to-Plans, Member-to-Plans, Check-in-to-Member, and Notification-to-Member calls.

The gym admin dashboard is CRUD-only. Do not add a BFF in this phase.

## Architectural decision

### Target request path

```text
Browser admin dashboard
  HTTPS + JSON + Bearer JWT
          |
          v
Kong HTTPS route
  1. gym-jwt-claims validates JWT
  2. client-supplied x-user-* headers are removed
  3. verified identity and role claims are injected
  4. grpc-gateway maps HTTP/JSON to one unary gRPC request
          |
          v
mTLS grpcs://ms-gym-member:50051
or   grpcs://ms-gym-plans:50051
  peer certificate SAN = kong
          |
          v
Member / Plans gRPC interceptors
  1. verify Kong SAN for end-user RPC
  2. validate injected claims and role policy
  3. validate Protobuf request
  4. execute application service
```

Kong remains the public HTTPS endpoint. “Remove Plans HTTP” means remove Plans' native business HTTP adapter, not remove HTTPS from the public API and not remove Actuator.

### Why Kong's plugin is acceptable here

The active public methods are unary CRUD methods. Kong Gateway 3.8 includes `grpc-gateway` in the bundled plugin set. The plugin can map annotated HTTP paths, JSON request bodies, path fields, and query fields to unary gRPC requests.

Do not use this plugin for streaming APIs:

- client streaming is unsupported;
- server-streamed messages are concatenated without a stable JSON array, NDJSON, or SSE contract;
- compressed gRPC responses are not a tested public REST contract.

Future streaming APIs must use native gRPC, gRPC-Web, SSE/WebSocket projection, or a generated gateway with an explicit streaming contract.

### Important Kong limitation

Kong 3.8 `grpc-gateway` reads a `.proto` source file from its local filesystem and discovers routes from `google.api.http` method annotations. It does **not** consume:

- `proto/http.yaml`;
- `member_http.yaml` or `plans_http.yaml`;
- a binary descriptor set;
- gRPC reflection;
- OpenAPI 3.0.

Therefore the existing external HTTP YAML cannot remain the only route source for Member and Plans. Phase 9 must add `google.api.http` annotations to the public Member and Plans RPCs.

## Prerequisites

1. G8 remains historical evidence. Do not rewrite its evidence.
2. `gym-proto`, Member, and Plans use one released contract generation. Plans must not remain on `gym-proto-java:4.0.0` while Member and source use `4.1.0` or later.
3. Kong target image remains pinned and tested. Current local target is Kong `3.8`.
4. Member's existing Kong-SAN gate and method-specific workload allowlist pass.
5. Plans' native mTLS gRPC server and Identifier/Member workload allowlist pass.
6. No production client depends on Plans' direct `8080 /api/**` endpoints. If one does, run a bounded dual-route migration before deletion.

## Implementation file map

Use this as minimum change list. A lower-level implementation agent must inspect current names before editing and record any renamed equivalent in G9 evidence.

### `gym-proto`

| Path | Required change |
|---|---|
| `proto/identity/v1/identity.proto` | Remove `SelectGym`, its messages, and removed fields where applicable. Keep registration, login, verification, and refresh gym-neutral. Add inline `google.api.http` annotations to the 12 retained public Identity RPCs so the browser contract covers the complete active API; Identifier continues using its native generated HTTP gateway. |
| JWT/auth contract manifests | Remove `gym_id` and `membership_status`; retain stable identity and token-control claims only. |
| OpenAPI operation manifest | Freeze the exact 27-operation Identity/Member/Plans allowlist, operation IDs, verbs, paths, body selectors, security, and observed error policy without duplicating message schemas. |
| `proto/member/v1/member.proto` | Remove `GetMembershipStatusByUserId`; add explicit `gym_id` to gym-specific public requests; retain purchase inputs; import Google API annotations and annotate only the seven final public RPCs. |
| `proto/plans/v1/plans.proto` | Import Google API annotations and annotate only the eight public Plans RPCs. |
| `proto/http.yaml` | Remove Identity, Member, and Plans selectors after inline annotations become canonical; retain only mappings needed by deferred services. |
| `proto/member/v1/member_http.yaml` | Delete or retire from generation/verification. |
| `proto/plans/v1/plans_http.yaml` | Delete or retire from generation/verification. |
| `buf.gen.yaml` or new `buf.openapi.gen.yaml` | Add `github.com/protoc-gen/protoc-gen-openapiv3` pinned to tag `v0.7.7` and commit `58202f09d79fe7c5cd8870cb1374933ff421ceb4`; remove obsolete Member/Plans external mapping dependence. |
| `scripts/verify-http-config.py` | Verify inline public annotations, exact verb/path/body bindings, no internal bindings, and no duplicate external selectors. |
| `scripts/verify-generated-routes.sh` | Verify generated Go gateway routes and generated OpenAPI 3.0 operations. |
| `build.gradle` | Wire annotation/OpenAPI 3.0/runtime-bundle checks into `check` without hand-written schema generation. |
| `.github/workflows/publish-stubs.yml` | Generate, determinism-check, upload, checksum, and release OpenAPI 3.0 plus Kong runtime proto bundle. |
| `.gitignore` | Keep generated `gen/` and local `dist/` outputs ignored unless existing artifact policy requires a narrow exception. |

### `gym-infra`

| Path | Required change |
|---|---|
| `kong/kong.yml` and active G9 declarative overlay | Generate exact method-plus-regex routes for all 12 Identity, seven Member, and eight Plans operations. Keep Identity on native HTTP `8080`; replace Member native edge and Plans HTTP upstream with HTTPS transcode routes to `grpcs:50051`; configure real Kong client certificate and upstream CA verification. |
| `kong/g8-kong.yml` or new `kong/g9-kong.yml` | Preserve G8 file as historical input when evidence depends on it; prefer a new G9 overlay rather than silently changing historical fixtures. |
| `kong/g8-compose.yml` or new `kong/g9-compose.yml` | Mount released runtime proto bundle, rendered ignored Kong config, client identity, and CA files read-only. |
| `kong/g8-business-check.sh` or new `kong/g9-business-check.sh` | Send Member and Plans browser requests through Kong; remove direct `grpcurl` use of Kong certificate as proof of public traffic. Keep direct workload calls only for method/SAN tests. |
| `kong/run-g8.sh` or new `kong/run-g9.sh` | Run contract, TLS, browser, error, and negative exposure gates; write new G9 evidence. |
| `kong/plugins/gym-jwt-claims/handler.lua` | Preserve header stripping/injection and priority `1000`; change only if tests expose path-matching or response-header gaps. |
| `kong/plugins/gym-jwt-claims/schema.lua` | Keep HTTP protocol scope; update configuration validation only if required by new route patterns. |
| `helm/gym-service/examples/ms-gym-plans-values.yaml` | Permit Kong to Plans `50051`, remove Kong business ingress to `8080`, preserve probe/metrics access. |
| Member Helm values/overlays | Permit Kong and named workloads to Member `50051` with separate port-specific peers. |
| `helm/gym-service/templates/networkpolicy.yaml` | Render separate ingress rules per port and caller group; no union that grants every peer every port. |

### `ms-gym-plans`

| Path | Required change |
|---|---|
| `build.gradle` | Upgrade `gym-proto-java` to G9 release; remove `spring-boot-starter-webmvc-test` after MVC tests are gone; keep web/Actuator and `startEnv`/`stopEnv`. |
| `src/main/java/com/gym/plans/config/GrpcConfig.java` | Add Kong identity interceptor for end-user methods while preserving exact Identifier/Member internal method allowlist. |
| `src/main/java/com/gym/plans/config/` | Add Plans equivalent of Member `KongIdentityServerInterceptor` and peer certificate SAN parsing, reusing existing local/shared code where practical. |
| `src/main/java/com/gym/plans/adapter/in/http/controller/GymLocationHttpController.java` | Delete after Kong E2E passes. |
| `src/main/java/com/gym/plans/adapter/in/http/controller/MembershipPlanHttpController.java` | Delete after Kong E2E passes. |
| `src/main/java/com/gym/plans/adapter/in/http/filter/HttpTrustedClaimsFilter.java` | Delete after gRPC metadata path passes. |
| `src/main/java/com/gym/plans/adapter/in/http/filter/HttpRoleInterceptor.java` | Delete after gRPC authorization path passes. |
| `src/main/java/com/gym/plans/config/WebConfig.java` | Delete or reduce to non-business web configuration. |
| `src/main/java/com/gym/plans/adapter/in/http/exception/PlansHttpExceptionHandler.java` | Delete; gRPC exception mapping remains canonical. |
| `src/test/**/PlansHttpIntegrationTest.java` | Replace with direct-`8080` negative and Kong browser E2E coverage. |
| `src/test/**/PlansHttpControllerUnitTest.java` | Delete with controllers. |
| `src/test/**/HttpTrustedClaimsFilterUnitTest.java` | Delete with filter. |
| `src/test/**/HttpRoleInterceptorUnitTest.java` | Delete with interceptor. |
| `README.md` | Document `50051` business API, `8080` Actuator-only role, `startEnv`/`stopEnv`, and Kong public path. |

### `ms-gym-member`

| Path | Required change |
|---|---|
| `build.gradle` | Upgrade to exact G9 `gym-proto-java` release. Keep web starter only for Actuator or other proven non-business runtime need. |
| Public gRPC delegates/application services | Read gym context from validated request messages, enforce customer ownership, and stop reading `GrpcSecurityContext.getGymId()`. |
| Identifier lookup delegate/mapper | Delete `GetMembershipStatusByUserId` handling after its sole caller is removed. |
| `src/main/java/com/gym/member/config/GrpcConfig.java` | Preserve Kong-only end-user RPC gate and exact remaining workload method/SAN allowlist; remove Identifier's Member permission. |
| `src/main/java/com/gym/member/config/KongIdentityServerInterceptor.java` | Keep approved Kong DNS/SPIFFE SAN set aligned with deployed certificate. |
| `src/main/java/com/gym/member/config/PeerCertificateIdentity.java` | Reuse as reference for Plans; do not broaden accepted identities. |
| Member gRPC integration tests | Prove explicit gym validation, customer ownership, real Kong SAN metadata path, and rejection of swapped workload certificates. |
| `README.md` and `LOCAL_TESTING.md` | Replace any public direct-grpcurl/Kong-certificate shortcut with browser HTTPS/JSON through Kong; retain direct grpcurl only for named workloads. |

### `ms-gym-identifier`

| Path | Required change |
|---|---|
| Identity handlers and use cases | Delete `SelectGym`; registration, login, verification, Google login, and refresh issue stable identity tokens only. |
| JWT signer and claim tests | Remove `gym_id` and `membership_status`; retain `sub`, `role`, issuer, audience, time, JTI, key ID, and revocation behavior. |
| Member port/adapter/configuration | Delete client, startup wiring, mTLS settings, health requirements, and shutdown handling. |
| Plans port/adapter/configuration | Retain `GetActiveGym` for `CreateTrainer`; keep exact workload mTLS identity. |
| Trainer-account authorization | Require `SUPER_ADMIN` until authoritative staff-to-gym assignment exists. |
| README and deployment inputs | Remove `/api/v1/auth/gym`, Identifier-to-Member certificates/settings, and NetworkPolicy edge. |

### Documentation

Update architecture, Member, Plans, security, and infrastructure pages that still claim Plans business traffic terminates on `8080` or Member has no public JSON route. Add supersession notes to Phase 7 and Phase 8; do not alter their historical evidence.

## Contract ownership

### Canonical sources

`gym-active-api.openapi.yaml` covers every active browser API: 12 retained Identity operations plus seven Member and eight Plans operations. Identity remains a native HTTP upstream; inclusion in OpenAPI does not route Identity through Kong's `grpc-gateway` plugin. Deferred-service routes remain excluded until their services become active.

Use these ownership rules:

| Concern | Canonical source |
|---|---|
| Message fields, enums, validation | `.proto` files |
| Public HTTP verb/path/body mapping for Member and Plans | `google.api.http` annotations in `.proto` |
| Backend Java and Go message classes | generated from `.proto` |
| Canonical browser/API document | generated directly from `.proto` annotations with pinned `protoc-gen-openapiv3`; never handwritten |
| Frontend REST client and REST-facing types | generated from OpenAPI, if generation is wanted |
| Kong runtime transcoding schema | exported `.proto` source bundle mounted read-only |
| JWT and route protection policy | Kong declarative config plus `gym-jwt-claims` |
| Domain error code and gRPC status | service/common gRPC exception mapping |

Do not handwrite a OpenAPI document. A small OpenAPI generator configuration may supply title, version, server/base path, Bearer security metadata, tags, and documented response headers. It must not duplicate message schemas or route definitions.

### DTO and OpenAPI version decision

Backend DTO/message generation stays Protobuf-first.

- Java and Go services use classes generated from `.proto`.
- Do not generate backend DTOs from OpenAPI and then map them back to Protobuf.
- Do not maintain parallel hand-written HTTP DTOs after Plans MVC removal.

For a browser frontend that calls HTTPS/JSON, OpenAPI generation is the better client boundary:

- OpenAPI contains HTTP paths, verbs, path/query/body placement, security, response status codes, and JSON schemas;
- Protobuf-generated TypeScript types contain message shapes but do not by themselves describe the REST call;
- generating both Protobuf TypeScript DTOs and OpenAPI DTOs for the same REST client creates duplicate types and conversion work.

Rule:

```text
Backend or native gRPC client: generate from .proto.
Browser REST client: generate one client from generated OpenAPI 3.0.
Browser using handwritten fetch: use generated OpenAPI as documentation and contract test input.
```

If the frontend later adopts native gRPC-Web instead of REST, generate its client from `.proto` and stop generating the OpenAPI client for that surface.

## Stage 0 — Replace selected-gym JWT state before gateway generation

Complete this contract and flow migration before adding HTTP annotations, generating OpenAPI, or configuring Kong. Otherwise Phase 9 would publish routes and clients for the obsolete selected-token contract and then break them again.

No production client or customer data exists. Treat Stage 0 as one coordinated `*.v1` pre-production contract break. Do not add dual JWTs, legacy route forwarding, request-to-JWT gym fallback, backfill, or database migration machinery.

### Stable identity contract

Identifier access tokens contain only stable identity and token-control claims:

```text
sub, role, iss, aud, iat, exp, jti, kid
```

They contain no `gym_id` and no `membership_status`. Registration, email verification, login, Google login, and refresh all issue the same identity-only shape. Live membership state remains Member-owned and must be queried from Member when a use case requires it.

Remove the obsolete Identity `SelectGym` RPC, its request/response message declarations, and `POST /api/v1/auth/gym`. Protobuf supports `reserved` only for field names/numbers inside retained messages and enum values/numbers, not service RPC names or top-level message names. Add field-level reservations only where fields are removed from a retained message, and add a compatibility check that retired RPC/message names are not reused. Remove `gym_id` and `membership_status` from the JWT profile and signer input. Keep Kong signature, issuer, audience, time, JTI, key-ID, and revocation validation unchanged.

### Frontend-selected gym contract

Gym selection remains explicit in browser URL/UI state:

```text
login with stable identity JWT
  -> GET /api/v1/gyms
  -> GET /api/v1/gyms/{gym_id}/plans
  -> user selects one gym and one plan
  -> POST /api/v1/gyms/{gym_id}/memberships/purchase
  -> GET /api/v1/gyms/{gym_id}/members/{member_id}/membership
```

A client-supplied `gym_id` expresses intent. It is never authoritative price, membership state, or administrative authorization. Member passes `(gym_id, plan_id)` to Plans `ResolvePurchasablePlan`; Plans proves the gym is active, the plan is active, and the plan belongs to that gym before returning canonical type, duration, and VND price.

### Final Member request shapes

Make every gym-specific public Member request carry validated gym resource context. Do not read gym scope from `GrpcSecurityContext.getGymId()`.

Use a nested purchase body so the path field is not duplicated in JSON and no existing purchase input is lost:

```protobuf
message PurchaseMembershipBody {
  string plan_id = 1;
  string provider = 2;
  string discount_code = 3;
  string idempotency_key = 4;
}

message PurchaseMembershipRequest {
  string gym_id = 1;
  PurchaseMembershipBody purchase = 2;
}
```

Apply the existing validation bounds to all moved fields. `discount_code` remains optional and nonblank values continue to fail until Promotion is an authoritative active service.

Add `gym_id` to pause, resume, and customer membership-status requests. `ListMembersRequest.gym_id` already exists and becomes explicit request scope rather than selected-token scope. Freeze these public mappings before Stage 1:

| RPC | HTTPS mapping | G9 policy |
|---|---|---|
| `GetMember` | `GET /api/v1/members/{member_id}` | customer self; `SUPER_ADMIN` override |
| `UpdateProfile` | `PUT /api/v1/members/{member_id}` | customer self; `SUPER_ADMIN` override |
| `ListMembers` | `GET /api/v1/gyms/{gym_id}/members` | `SUPER_ADMIN` only |
| `PurchaseMembership` | `POST /api/v1/gyms/{gym_id}/memberships/purchase`, body `purchase` | customer self from verified `sub` |
| `PauseMembership` | `POST /api/v1/gyms/{gym_id}/members/{member_id}/membership:pause` | customer self |
| `ResumeMembership` | `POST /api/v1/gyms/{gym_id}/members/{member_id}/membership:resume` | customer self |
| `GetMembershipStatus` | `GET /api/v1/gyms/{gym_id}/members/{member_id}/membership` | customer self; `SUPER_ADMIN` override |

Member must verify that the authenticated user owns the target member for customer operations and that the selected subscription belongs to the requested gym. Request `gym_id` alone grants nothing.

### Administrative gym authorization

No authoritative `ADMIN`-to-gym assignment model exists. Therefore G9 must not replace selected-gym JWT authorization with caller-chosen request scope.

Until a later contract defines assignment ownership, persistence, revocation, lookup, and tests:

- Plans gym and plan mutations are `SUPER_ADMIN`-only;
- Member gym-wide listing and management are `SUPER_ADMIN`-only;
- Identifier trainer-account creation with a gym argument is `SUPER_ADMIN`-only;
- catalog reads remain available to authenticated users.

Restoring gym-scoped `ADMIN` permissions is separate roadmap work, not a hidden Phase 9 shortcut.

Freeze these public Plans mappings before Stage 1:

| RPC | HTTPS mapping | G9 policy |
|---|---|---|
| `CreateGymLocation` | `POST /api/v1/gyms` | `SUPER_ADMIN` only |
| `UpdateGymLocation` | `PUT /api/v1/gyms/{id}` | `SUPER_ADMIN` only |
| `GetGymLocation` | `GET /api/v1/gyms/{id}` | authenticated |
| `ListGymLocations` | `GET /api/v1/gyms` | authenticated |
| `CreateMembershipPlan` | `POST /api/v1/gyms/{gym_id}/plans` | `SUPER_ADMIN` only |
| `UpdateMembershipPlan` | `PUT /api/v1/plans/{id}` | `SUPER_ADMIN` only |
| `GetMembershipPlan` | `GET /api/v1/plans/{id}` | authenticated |
| `ListMembershipPlans` | `GET /api/v1/gyms/{gym_id}/plans` | authenticated |

`GetActiveGym` and `ResolvePurchasablePlan` remain workload-only. `ValidateMembership` and `ListMembersByStatus` remain Member workload-only. None receives a `google.api.http` annotation or appears in Kong or OpenAPI 3.0.

### Exact active browser-operation allowlist

Canonical OpenAPI contains exactly these 27 operation IDs. The pinned generator derives each ID as `<Service>_<Rpc>`; verification must reject renames, duplicates, additions, and omissions.

```text
IdentityService_Register
IdentityService_Login
IdentityService_LoginWithGoogle
IdentityService_RefreshToken
IdentityService_VerifyEmail
IdentityService_ResendEmailVerification
IdentityService_Logout
IdentityService_GetCurrentUser
IdentityService_ChangePassword
IdentityService_CreateTrainerAccount
IdentityService_SuspendUser
IdentityService_ListUsers

MemberService_GetMember
MemberService_UpdateProfile
MemberService_ListMembers
MemberService_PurchaseMembership
MemberService_PauseMembership
MemberService_ResumeMembership
MemberService_GetMembershipStatus

PlansService_CreateGymLocation
PlansService_UpdateGymLocation
PlansService_GetGymLocation
PlansService_ListGymLocations
PlansService_CreateMembershipPlan
PlansService_UpdateMembershipPlan
PlansService_GetMembershipPlan
PlansService_ListMembershipPlans
```

Identity retains its current verbs and paths except removed `POST /api/v1/auth/gym`. Add inline annotations for its 12 retained methods and remove their duplicate selectors from `proto/http.yaml`. Kong continues proxying those operations to Identifier HTTP `8080`; only Member and Plans use Kong transcoding. The OpenAPI verifier must compare operation ID, verb, path, body selector, and security against one checked-in semantic manifest. That manifest may repeat route metadata for verification, but it must not define request or response schemas.

### Identifier removal and retained dependency

Delete Identifier's `SelectGym` handler, use case, signing branch, tests, HTTP route, Member port/adapter, Member client configuration, startup wiring, certificate requirements, and NetworkPolicy edge.

After this deletion, `MemberService/GetMembershipStatusByUserId` has no production caller. Remove the RPC, messages, Member delegate/handler/mapper code, Identifier SAN allowlist entry, tests, and route manifests; reserve removed Protobuf field numbers and names where applicable.

Do not remove Identifier's Plans client. `CreateTrainer` still calls `PlansService/GetActiveGym`. Retain that exact mTLS method allowlist until trainer validation moves to an authoritative Trainer/staff-assignment boundary.

### Kong claim plugin migration

Update `gym-jwt-claims` so protected routes no longer require `membership_status` and no longer derive `x-gym-id` or `x-membership-status` from JWT. Continue removing all client-supplied trusted headers before proxying. Inject only verified identity/role metadata needed by public RPCs. Path-to-Protobuf binding supplies `gym_id`; it is not a trusted claim header.

### Stage 0 failure gates

Stop before Stage 1 unless all pass:

- access and refresh tokens contain no `gym_id` or `membership_status`;
- `/api/v1/auth/gym` is absent;
- Identifier has no Member runtime dependency;
- `GetMembershipStatusByUserId` is absent from generated stubs and Member method registry;
- every gym-specific public Member method validates explicit `gym_id`;
- purchase rejects missing gym, inactive gym, inactive plan, and plan/gym mismatch;
- caller-chosen gym cannot grant `ADMIN` access;
- `SUPER_ADMIN` behavior is explicit;
- Identifier-to-Plans trainer validation still passes;
- tests use `given_when_then` names.

## Stage 1 — Make Protobuf usable by Kong and generate contracts

### Make Protobuf usable by Kong

#### Add HTTP annotations

In `gym-proto/proto/identity/v1/identity.proto`, `gym-proto/proto/member/v1/member.proto`, and `gym-proto/proto/plans/v1/plans.proto`:

1. Import `google/api/annotations.proto`.
2. Add one `option (google.api.http)` block to every active public RPC in the 27-operation allowlist.
3. Add no HTTP option to workload-only RPCs.
4. Preserve existing Identity verb/path/body mappings except removed `SelectGym`.
5. Preserve the final Member and Plans route tables above, including `body: "purchase"` for `PurchaseMembership` and `body: "*"` only for mutations whose request body is the complete request message.

Example:

```protobuf
import "google/api/annotations.proto";

service PlansService {
  rpc CreateGymLocation(CreateGymLocationRequest) returns (CreateGymLocationResponse) {
    option (google.api.http) = {
      post: "/api/v1/gyms"
      body: "*"
    };
  }

  rpc GetGymLocation(GetGymLocationRequest) returns (GetGymLocationResponse) {
    option (google.api.http) = {
      get: "/api/v1/gyms/{id}"
    };
  }

  // Internal only: intentionally has no google.api.http annotation.
  rpc GetActiveGym(GetActiveGymRequest) returns (GetActiveGymResponse);
}
```

Keep field names in Protobuf snake_case. Protobuf JSON exposes lowerCamelCase by default. The frontend contract therefore uses names such as `memberId`, `gymId`, and `idempotencyKey`.

#### Remove duplicate active-service mapping sources

After annotations work:

- remove Identity, Member, and Plans selectors from `gym-proto/proto/http.yaml`;
- delete or stop validating `proto/member/v1/member_http.yaml` and `proto/plans/v1/plans_http.yaml`;
- keep external service configuration only for deferred, still-unannotated services;
- update `scripts/verify-http-config.py` so it validates annotations for all three active services rather than demanding duplicated YAML entries.

Do not keep the same selector in both an annotation and external YAML. One public method must have one route source.

The existing `protoc-gen-grpc-gateway` invocation may continue using `grpc_api_configuration=proto/http.yaml` for unannotated services. Its output must prove that annotated Identity/Member/Plans routes and externally configured deferred routes coexist without duplicates. If the pinned generator rejects that mixed mode, split generation into explicit Buf templates rather than restoring duplicate route definitions.

#### Export runtime Protobuf bundle

Kong parses `.proto` source at runtime. It must receive the root service file and all imports with their relative paths intact.

Add a deterministic packaging task in `gym-proto`, for example:

```bash
rm -rf dist/kong-proto
buf export proto --output dist/kong-proto
```

The exported bundle must contain at least:

```text
dist/kong-proto/member/v1/member.proto
dist/kong-proto/plans/v1/plans.proto
dist/kong-proto/common/v1/common.proto
dist/kong-proto/google/api/annotations.proto
dist/kong-proto/google/api/http.proto
dist/kong-proto/buf/validate/validate.proto
```

Package this directory as a versioned `kong-proto-<contract-version>.tar.gz` release artifact. Record SHA-256. Kong deployments must pin the artifact to the same contract version used by Member and Plans.

Do not copy source from a developer checkout into production Pods. Build an immutable Kong image layer or init-container artifact from the released bundle.

### Generate canonical OpenAPI 3.0 directly

#### Generator

Use `github.com/protoc-gen/protoc-gen-openapiv3` directly. Pin tag `v0.7.7` and commit `58202f09d79fe7c5cd8870cb1374933ff421ceb4`; never use an unpinned branch or `latest`. That release reads `google.api.http` annotations and emits `openapi: 3.0.0` YAML.

Source inspection already establishes generator limits that implementation must test, not assume away:

- operation IDs are `<Service>_<Rpc>`;
- the generator currently emits one document per generated input;
- request bodies reference the complete input message even when `google.api.http.body` names a nested field;
- generated responses are fixed to `200`, `400`, `401`, and `500` with generic JSON schemas;
- no built-in option documents observed Kong response headers or its full status matrix;
- its output does not prove Protovalidate constraints, enum/optional/map behavior, or Protobuf `int64` JSON safety.

First run the pinned generator against final Identity, Member, and Plans contracts and treat these as mandatory adoption gates:

- all final route verbs, paths, and path/body/query bindings are correct, including nested `body: "purchase"`;
- all request and response schemas are present;
- Protovalidate-required fields, enums, maps, optionals, and Protobuf JSON names are represented correctly;
- `int64` JSON behavior is safe for generated TypeScript clients;
- one deterministic merged active-API document can be produced.

The unmodified generator is acceptable only if every gate passes. Known nested-body and response limitations make a small pinned fork likely. Patch `protoc-gen-openapiv3` at the pinned commit, pin the fork commit, and add focused generator tests for only failed gates; do not build a second schema generator or hand-edit output. If the pinned upstream generator or its pinned reviewed fork cannot satisfy every gate, stop G9 publication. Do not switch generator families, add a 2.0 artifact, or add a conversion pipeline.

Recommended target template after the adoption spike confirms exact supported options:

```yaml
# buf.openapi.gen.yaml
version: v2
plugins:
  - local:
      - go
      - run
      - github.com/protoc-gen/protoc-gen-openapiv3@v0.7.7
    out: gen/openapi
    strategy: all
```

Record exact verified merge/output options in the template. If the generator emits one document per input and has no deterministic merge option, add the smallest deterministic OpenAPI 3.0 document merge step. OpenAPI 3.0 is the only generated API-document format.

Generate from active service inputs only. Do not publish deferred Payment, Workout, Trainer, Check-in, Notification, Analytics, or Promotion routes as if they are implemented.

Canonical artifact:

```text
gen/openapi/gym-active-api.openapi.yaml    # openapi: 3.0.0
```

Run generation twice in clean directories and require byte-identical output. Do not manually edit the generated file.

#### Deterministic metadata and observed error policy

Do not hand-edit generated YAML and do not maintain a path-by-path OpenAPI overlay. Add metadata and observed runtime errors through the pinned generator/fork plus checked-in semantic inputs:

1. `contracts/v1/http/active-operations.yaml` owns the 27-operation allowlist, expected operation IDs, verb/path/body selector, auth policy, and applicable gRPC status classes. It contains no message schemas.
2. `contracts/v1/http/kong-3.8-errors.yaml` records only behavior measured by the compatibility spike: HTTP status, body media type/schema policy, response headers, and CORS visibility. It contains no paths or Protobuf fields.
3. The generator/fork reads those inputs or receives their values through deterministic options and emits title, contract version, HTTPS server placeholder, Bearer security, operation responses, and `x-error-code` header references in one pass.
4. Generation fails when an allowlisted operation is absent, an extra active operation appears, an error class lacks observed behavior, or metadata conflicts with Protobuf annotations.

Use reusable `components.headers.XErrorCode` and only the observed body schema. If Kong returns an empty or non-contractual body, document no invented JSON body. Apply error responses only to operations whose service/domain mappings can produce them; route-level Kong `401`, `404`, and content-negotiation behavior remain separately described where applicable.

Inputs are metadata and verification policy, not another API schema. They must not duplicate Protobuf message fields. Paths appear only in the operation manifest because exact route verification requires them; Protobuf annotations remain runtime route source.

#### Publication

Update `gym-proto/.github/workflows/publish-stubs.yml`:

1. generate stubs, gateway code, canonical OpenAPI 3.0, and Kong proto bundle;
2. validate the public route set and generator adoption gates;
3. regenerate and compare checksums;
4. upload OpenAPI 3.0 and Kong bundle in validation artifacts;
5. attach both to the tagged `gym-proto` release;
6. include their SHA-256 checksums and pinned generator identity in release evidence.

The frontend consumes the released OpenAPI 3.0 artifact or an artifact copied into its repository by an explicit dependency update. Do not make frontend builds download a mutable `develop` artifact.

#### OpenAPI verification

Add a small script that parses canonical OpenAPI 3.0 and asserts:

- operation IDs equal the exact 27-operation Identity/Member/Plans allowlist;
- all operations have expected verbs, paths, body selectors, and security from `active-operations.yaml`;
- `SelectGym`, `GetMembershipStatusByUserId`, and four retained workload-only RPCs are absent;
- request/response schemas exist and nested purchase body exposes `PurchaseMembershipBody`, not the complete path-plus-body request;
- JWT security is declared for protected operations;
- every documented non-2xx status/header/body matches `kong-3.8-errors.yaml`;
- `x-error-code` is a reusable response header and is CORS-exposed where observed;
- `int64` fields retain string-safe JSON documentation where required by Protobuf JSON behavior;
- generated output contains no local filesystem paths;
- document declares exactly `openapi: 3.0.0` and is the frontend input;
- output is deterministic.

Do not verify generated OpenAPI with string grep alone. Parse YAML/JSON and compare structured operations.

## Stage 2 — Align service transport trust

Plans currently requires mTLS but authorizes only the two internal RPCs by exact workload SAN. Add Member-equivalent protection for public RPCs:

1. classify methods by the existing `GrpcMethodRegistry` policy;
2. allow `INTERNAL_WORKLOAD` methods only through the exact method-to-SAN map;
3. require Kong SAN for every claim-bearing end-user method;
4. reject missing TLS, unknown methods, CA-valid non-Kong clients, and forged `x-user-*` metadata;
5. keep reflection unavailable through Kong.

Preserve this final matrix:

| RPC group | Allowed peer identity |
|---|---|
| `PlansService/GetActiveGym` | Identifier only |
| `PlansService/ResolvePurchasablePlan` | Member only |
| Plans public CRUD RPCs | Kong only |
| Member public/profile/subscription RPCs | Kong only |
| `MemberService/ValidateMembership` | Check-in only |
| `MemberService/ListMembersByStatus` | Notification only |

Identifier has no Member permission after Stage 0. Reuse the established Member pattern. Do not add service-role metadata or `*_SERVICE` user roles.

A future common-java release may move peer-SAN extraction into a shared helper after both services prove identical behavior. Do not block Phase 9 on that refactor.

## Stage 3 — Configure Kong transcoding and upstream mTLS

### Route shape

For each backend, configure a `grpcs` Service and HTTP/HTTPS Routes. Attach `grpc-gateway` to the REST routes or Service.

Illustrative structure:

```yaml
services:
  - name: ms-gym-plans-grpc-json
    protocol: grpcs
    host: ms-gym-plans
    port: 50051
    client_certificate:
      id: 10000000-0000-0000-0000-000000000001
    tls_verify: true
    ca_certificates:
      - 20000000-0000-0000-0000-000000000001
    routes:
      - name: plans-public-json
        protocols: [https]
        paths:
          - /api/v1/gyms
          - /api/v1/plans
        strip_path: false
        plugins:
          - name: grpc-gateway
            config:
              proto: /usr/local/kong/proto/plans/v1/plans.proto
```

Create the corresponding Member service with:

```yaml
proto: /usr/local/kong/proto/member/v1/member.proto
```

Public proxy routes use `https` only. If port `80` is exposed, configure a separate HTTP-to-HTTPS redirect that never proxies upstream and never accepts Bearer credentials as an application route.

Generate declarative method-plus-regex routes from the exact 27-operation manifest: 12 Identity operations to native Identifier HTTP, seven Member operations to Member transcoding, and eight Plans operations to Plans transcoding. Broad `/api/v1`, `/api/v1/auth`, and `/api/v1/gyms` prefix routes are forbidden. Member and Plans share paths below `/api/v1/gyms`; membership paths must select Member and gym/catalog paths must select Plans. Give more specific regex routes higher priority where Kong route matching requires it. Unknown paths and wrong-method variants return route-level `404` and never cross into another backend. Tests, not comments, define the accepted route set.

### Plugin order

Current priorities are compatible:

```text
gym-jwt-claims priority 1000
grpc-gateway    priority 998
```

JWT validation and trusted-header injection therefore run before transcoding. Preserve this order and add an integration test proving injected identity/role metadata arrives as gRPC metadata. The plugin must never inject gym or membership state.

Update `gym-jwt-claims.config.protected_routes` from native gRPC method paths to exact HTTP-method plus path-template/regex entries for all protected Identity, Member, and Plans operations. Remove `/api/v1/auth/gym`. Handle CORS `OPTIONS` before JWT enforcement, but still strip forged `x-user-*` headers. Every proxied operation enforces JWT according to its frozen policy; unmatched methods/paths never reach an upstream.

### Kong client identity

Replace `client_certificate: null`. Create an actual Kong Certificate entity and reference it from both Member and Plans Services.

Requirements:

- certificate EKU permits client authentication;
- SAN contains `kong` or the approved Kong SPIFFE ID;
- Member and Plans trust the issuing client CA;
- Kong validates each upstream server certificate;
- upstream server SAN matches `ms-gym-member` or `ms-gym-plans`;
- private keys are never committed.

For local G9 only, generate a temporary ignored Kong declarative file that embeds disposable fixture certificate material. Mount it and the runtime proto bundle read-only. For production, use supported Kong Vault references or secret provisioning; do not render long-lived private keys into Git.

Kong's CA Certificate entity used for `tls_verify` is separate from the client Certificate entity used for `client_certificate`.

### CORS

Browser routes need an explicit Kong CORS policy. Configure only approved frontend origins. Expose error headers needed by the frontend, subject to the error-contract spike below.

Do not rely on the transcoder's permissive `OPTIONS` behavior as production CORS policy.

### Native gRPC edge route

G9 final public surface is HTTPS/JSON only. Remove existing public Member native gRPC route from Kong. This does not remove Member's private gRPC server; Kong transcoding and internal workloads still use it.

A later native client requires a separate roadmap decision, route, auth policy, and tests. Do not retain an unproven second public protocol surface in G9.

## Stage 4 — Remove Plans native business HTTP and prove G9

Delete Plans' business MVC adapter after Kong E2E passes:

- `GymLocationHttpController`;
- `MembershipPlanHttpController`;
- `HttpTrustedClaimsFilter`;
- `HttpRoleInterceptor`;
- `WebConfig` registrations used only by business HTTP;
- `PlansHttpExceptionHandler`;
- their HTTP unit and integration tests;
- `spring-boot-starter-webmvc-test` if no remaining test uses it.

Keep:

- `spring-boot-starter-web` while Actuator runs on the servlet server;
- `spring-boot-starter-actuator`;
- `server.port: 8080` for health/readiness only;
- `/actuator/health/**` and Prometheus configuration;
- Docker and Helm health checks;
- gRPC handlers, generated Protobuf messages, domain/application services, persistence, validation, and mTLS.

Required negative check:

```text
Direct request to Plans :8080/api/v1/gyms returns 404.
Kong HTTPS request to /api/v1/gyms reaches Plans gRPC :50051.
```

Do not call the second path “Plans HTTP.” It is Kong's public HTTP representation of Plans gRPC.

### NetworkPolicy and deployment

Update Plans and Member overlays so:

- Kong may reach gRPC `50051`;
- Identifier and Member may reach only Plans gRPC `50051` at the network layer, with method SAN checks as the second layer;
- Check-in and Notification may reach only their allowed Member workload methods on gRPC `50051`; Identifier has no Member network edge;
- Kong has no business need to reach Plans `8080`;
- no broad peer rule grants every caller every port;
- Actuator/metrics access remains separately scoped;
- runtime Protobuf bundle and Kong certificate material mount read-only.

The shared Kubernetes Service may retain HTTP `8080` for probes and metrics. Public ingress and Kong upstream configuration must not target that port.

Version-lock deployment inputs:

```text
Kong declarative config version
Kong image version
Kong runtime proto bundle version + checksum
Member gym-proto artifact version
Plans gym-proto artifact version
Generated OpenAPI version + checksum
```

Reject deployment when these contract versions differ.

## Error contract

### Runtime status mapping

Services continue returning gRPC statuses. Kong 3.8 maps them to HTTP:

| gRPC status | HTTP status |
|---|---:|
| `OK` | `200` |
| `CANCELLED` | `499` |
| `UNKNOWN` | `500` |
| `INVALID_ARGUMENT` | `400` |
| `DEADLINE_EXCEEDED` | `504` |
| `NOT_FOUND` | `404` |
| `ALREADY_EXISTS` | `409` |
| `PERMISSION_DENIED` | `403` |
| `RESOURCE_EXHAUSTED` | `429` |
| `FAILED_PRECONDITION` | `400` |
| `ABORTED` | `409` |
| `OUT_OF_RANGE` | `400` |
| `UNIMPLEMENTED` | `500` |
| `INTERNAL` | `500` |
| `UNAVAILABLE` | `503` |
| `DATA_LOSS` | `500` |
| `UNAUTHENTICATED` | `401` |

`406 Not Acceptable` is not a domain mapping. Kong may return it only for HTTP content negotiation behavior. Do not map a gRPC domain error to `406`.

### Structured error caveat

Kong's plugin translates status codes but does not create the canonical grpc-gateway `google.rpc.Status` JSON body. It also does not itself convert `grpc-message`, `grpc-status-details-bin`, or Member/Plans `x-error-code` trailers into a stable JSON error envelope.

Do not publish this fictional contract without implementing it:

```json
{
  "code": "PLAN_NOT_FOUND",
  "message": "Plan not found"
}
```

First add a Phase 9 compatibility spike that records, through real Kong 3.8:

- HTTP status;
- response body bytes;
- `Content-Type`;
- `grpc-status`;
- `grpc-message`;
- `x-error-code`;
- whether metadata arrives as headers or trailers;
- browser visibility under CORS.

Minimum acceptable V1 frontend contract:

```text
HTTP status is stable.
x-error-code is available as an exposed response header.
Internal exception text is absent.
```

Document the observed body exactly. Generated OpenAPI must describe observed behavior, not the desired behavior.

If `x-error-code` cannot be promoted reliably to a browser-visible response header, stop and choose the generated gateway alternative below. Do not build a large custom Kong body-rewrite plugin only to imitate grpc-gateway's standard error handler.

### Application error ownership

Keep service mappings in `common-java`/service gRPC interceptors:

```text
NotFoundException      -> NOT_FOUND         -> 404
ForbiddenException     -> PERMISSION_DENIED -> 403
ConflictException      -> ALREADY_EXISTS    -> 409
validation failure     -> INVALID_ARGUMENT  -> 400
unavailable dependency -> UNAVAILABLE       -> 503
unexpected exception   -> INTERNAL          -> 500 with redacted detail
```

Kong translates transport status. It does not decide domain error categories.

## Tests

Use `given_when_then` names and explicit setup, action, and assertion sections.

### `gym-proto`

- `SelectGym`, `GetMembershipStatusByUserId`, and their obsolete mappings are absent;
- JWT contract permits no `gym_id` or `membership_status`;
- all 27 active Identity, Member, and Plans methods have exact `google.api.http` bindings;
- workload-only methods have no binding;
- gateway and OpenAPI 3.0 generation are deterministic;
- generated OpenAPI 3.0 contains exactly the 27 allowlisted Identity, Member, and Plans operations;
- Kong proto bundle contains all transitive imports;
- Kong 3.8 can parse both mounted root protos;
- no selector has both annotation and external YAML mapping.

### Identifier

- registration, verification, login, Google login, and refresh issue identity-only tokens;
- `/api/v1/auth/gym` is absent;
- no Member client, configuration, certificate, health requirement, or NetworkPolicy edge remains;
- trainer creation remains `SUPER_ADMIN`-only and still validates the gym through Plans `GetActiveGym`;
- Identifier cannot call any Member method.

### Plans

- Kong SAN plus valid claims reaches each public RPC;
- missing Kong SAN fails;
- Identifier, Member, Check-in, or Notification certificate plus forged claims fails on public RPCs;
- Identifier calls only `GetActiveGym`;
- Member calls only `ResolvePurchasablePlan`;
- all mutations require `SUPER_ADMIN`;
- swapped SANs fail;
- direct `8080 /api/**` returns `404`;
- Actuator health remains available;
- all filter tests prove JPA Specification composition.

### Member

- Kong SAN plus valid identity/role metadata reaches each public RPC;
- browser JSON mapping preserves query, path, body, enum, optional, and `int64` behavior;
- every gym-specific public operation rejects missing or invalid explicit `gym_id`;
- purchase rejects inactive gym, inactive plan, and plan/gym mismatch;
- customer self-service verifies authenticated-user ownership and subscription gym;
- caller-chosen gym grants no `ADMIN` privilege;
- gym-wide administration is `SUPER_ADMIN`-only;
- Check-in calls only `ValidateMembership` and Notification calls only `ListMembersByStatus`;
- workload-only methods are absent from Kong and OpenAPI 3.0;
- wrong internal certificate fails even with forged claims.

### Kong/infrastructure

Use a G9 replacement fixture, not a claim that the removed selected-token flow itself still passes:

```text
Given a customer has an identity-only JWT and an active member profile,
when POST /api/v1/auth/gym is attempted,
then Kong returns route-level 404 and no upstream receives the request.

Given the same customer selects gym G and plan P in browser state,
when POST /api/v1/gyms/G/memberships/purchase sends P,
then Member passes (G, P) to Plans, freezes authoritative terms, and produces the same successful purchase/subscription outcome G8 proved.
```

- anonymous protected request returns `401` before upstream call;
- invalid, expired, or revoked JWT returns `401` before upstream call;
- forged `x-user-*` values never reach upstream;
- verified identity/role values arrive as gRPC metadata; gym and membership state do not;
- Kong presents its client certificate to Member and Plans;
- Kong validates upstream server CA and SAN;
- wrong Kong client certificate fails TLS or SAN authorization;
- all public JSON operations pass through Kong;
- all workload-only HTTP paths return `404`;
- internal RPC names are absent from Kong config;
- Identifier-to-Member connectivity is absent;
- error compatibility matrix matches the table above;
- CORS allows only configured frontend origins and exposes `x-error-code`;
- rendered NetworkPolicies are port-specific;
- the executable G9 replacement fixture proves removed `/api/v1/auth/gym` returns `404` and explicit gym request state preserves the successful G8 purchase/subscription outcome.

### Frontend contract fixture

Generate a temporary TypeScript client from released OpenAPI 3.0 and compile one CRUD smoke fixture. Do not commit generated frontend code into `gym-proto`.

The smoke fixture should prove:

- authorization header support;
- gym-scoped `ListMembers` path serialization;
- nested purchase body serialization without duplicate path `gym_id`;
- `UpdateProfile` JSON body names;
- Plans enum serialization;
- non-2xx errors remain inspectable by status and `x-error-code`.

## Delivery order

Execute in this order:

1. Complete Stage 0 contract changes in `gym-proto`: stable JWT profile, retired RPC/message checks, field-level reservations where valid, explicit gym-scoped Member requests, and final role policy.
2. Publish an immutable Stage 0 contract release containing Java/Go stubs and semantic manifests.
3. Upgrade Identifier and Member to that exact Stage 0 release; remove Identifier-to-Member runtime and network dependencies; prove stable login, explicit gym purchase, ownership, and trainer validation.
4. Add final Identity/Member/Plans annotations, direct OpenAPI 3.0 generation, Member/Plans runtime proto bundle, and contract tests.
5. Align Plans public trust with Member and prove the final exact method-to-SAN matrix.
6. Configure a G9 Kong candidate with HTTPS-only public proxy routes, identity/role-only metadata injection, CORS, real upstream mTLS, read-only proto bundle, and port-specific NetworkPolicies.
7. Run the real Kong 3.8 error/body/header compatibility spike while Plans MVC still exists. If `x-error-code` or browser behavior fails, stop and choose the generated-gateway alternative before publishing the browser contract.
8. Generate canonical OpenAPI 3.0 from the selected final runtime behavior, compile the frontend client fixture, and publish one immutable G9 contract release containing stubs, OpenAPI 3.0, Kong bundle, generator identity, versions, and checksums.
9. Upgrade Identifier, Member, Plans, and Kong deployment inputs to that exact G9 release; run positive and negative Kong E2E.
10. Remove Plans business MVC code and tests, remove Kong-to-Plans `8080` access, and run the complete G9 evidence suite.

Do not generate or publish the browser contract before steps 1–7 pass. Do not delete Plans MVC before step 9 passes.

## Verification commands

Exact commands may follow existing wrappers, but the completed phase must cover:

```bash
cd /home/phucl/Workplace/gapi/gym-proto
buf format -d --exit-code proto
buf lint proto
rm -rf gen dist/kong-proto
buf generate
buf export proto --output dist/kong-proto
./gradlew check

cd /home/phucl/Workplace/gapi/ms-gym-member
./gradlew clean check

cd /home/phucl/Workplace/gapi/ms-gym-plans
./gradlew startEnv
./gradlew clean check
./gradlew stopEnv

cd /home/phucl/Workplace/gapi/gym-infra
helm lint ./helm/gym-service
./kong/run-g9.sh
```

Run root `make test-all` after all active repositories resolve the released contract.

## Evidence produced

Create new G9 evidence; do not edit G8 evidence:

- exact repository SHAs and clean-tree state;
- contract/artifact versions;
- canonical generated OpenAPI 3.0 and checksum;
- Kong proto bundle and checksum;
- exact 27-operation Identity/Member/Plans route and exposure matrix;
- Kong plugin and image version;
- Kong upstream certificate subject/SAN and expiry, excluding private key;
- positive and negative JWT tests;
- positive and negative mTLS SAN matrix;
- full HTTP-to-gRPC request captures with secrets removed;
- full error status/header/body compatibility matrix;
- direct Plans `8080 /api/**` negative proof;
- Actuator health proof;
- Helm render and NetworkPolicy report;
- generated TypeScript client compile report;
- executable G9 replacement proof: `/api/v1/auth/gym` route-level `404` plus successful explicit-gym purchase/subscription outcome.

## Alternative if Kong's plugin fails the error or compatibility gate

Use one generated Go `grpc-gateway` runtime as a thin, non-BFF transport adapter behind Kong:

```text
Browser HTTPS/JSON
  -> Kong JWT/CORS/rate limiting
  -> mTLS HTTPS generated gateway
  -> mTLS gRPC Member/Plans
```

It contains generated route registration, Protobuf JSON marshaling, and a standard grpc-gateway error handler only. It owns no business orchestration, persistence, domain DTOs, or authorization scope, so it is not a BFF.

Trust model:

- Browser reaches only Kong; NetworkPolicy denies direct browser/ingress access to gateway and service ports.
- Kong strips every client-supplied trusted header, validates JWT, and injects verified identity/role.
- Kong authenticates the gateway over TLS with its existing `kong` client identity and validates gateway server SAN `ms-gym-api-gateway`.
- Gateway accepts injected identity/role only when its immediate peer SAN is `kong`; direct callers and CA-valid non-Kong callers fail before routing.
- Gateway strips inbound gRPC metadata names before constructing each upstream request, then reinjects only verified identity/role from its trusted request context.
- Gateway calls Member and Plans with distinct client SAN `ms-gym-api-gateway`; it validates each upstream CA and server SAN.
- Member and Plans replace `kong` with `ms-gym-api-gateway` only for public RPCs. Workload allowlists remain unchanged: Identifier to `GetActiveGym`, Member to `ResolvePurchasablePlan`, Check-in to `ValidateMembership`, Notification to `ListMembersByStatus`.
- Kong cannot directly reach Member/Plans `50051` in fallback mode; gateway cannot reach workload-only HTTP paths because generated registration excludes them, and service method authorization rejects its SAN on workload RPCs.
- NetworkPolicies permit Kong to gateway HTTPS only, gateway to Member/Plans `50051` only, and existing exact workloads to their destination `50051`; no union grants gateway access to service `8080`.
- Production gateway/Kong private keys come from Secrets or a secret manager and are never committed.

Fallback tests must prove wrong immediate-peer SAN, forged trusted headers, direct gateway access, gateway SAN on workload RPCs, direct Kong-to-service access, swapped upstream SANs, and absent internal routes all fail. Update route/SAN matrices, OpenAPI error evidence, Helm values, and release version lock before cutover.

Choose this alternative when the Kong spike cannot expose stable `x-error-code`, cannot match published body behavior, or fails required JSON binding. It is also preferable for canonical structured JSON errors, custom error handlers, richer JSON marshaling, streaming-specific behavior, or when runtime Protobuf source mounting is unacceptable. Prefer it over growing custom Kong Lua plugins.

For current unary CRUD scope, try Kong's bundled plugin first. The structured-error compatibility spike is the decision gate.

## Explicit non-goals

- No BFF.
- No frontend framework selection.
- No implementation of deferred services.
- No streaming transcode contract.
- No hand-written OpenAPI schemas.
- No backend DTO generation from OpenAPI.
- No direct browser access to service ports.
- No public exposure of reflection or workload RPCs.
- No custom Kong error-body plugin unless separately approved after the compatibility spike.

## Final consistency matrix

| Concern | G9 final state |
|---|---|
| JWT fields | `sub`, `role`, `iss`, `aud`, `iat`, `exp`, `jti`, `kid`; no `gym_id` or `membership_status` |
| Gym source | Explicit path/request resource context; never authorization proof |
| Customer authorization | Verified `sub` plus Member-owned profile/subscription ownership |
| Gym-wide administration | `SUPER_ADMIN` only until authoritative staff-to-gym assignment exists |
| Member purchase path | `POST /api/v1/gyms/{gym_id}/memberships/purchase` with body field `purchase` |
| Authoritative purchase data | Plans `ResolvePurchasablePlan(gym_id, plan_id)` over Member mTLS identity |
| Retained Identifier dependency | Plans `GetActiveGym` for `CreateTrainer` only |
| Removed Identifier dependency | No Identifier-to-Member client, method permission, certificate, or NetworkPolicy edge |
| Public transport | Browser HTTPS/JSON to Kong; Kong `grpcs:50051` to Member and Plans |
| Public metadata trust | Kong certificate SAN plus verified identity/role metadata only |
| Internal authorization | Exact RPC-to-workload-SAN allowlist |
| Plans `8080` | Actuator, probes, and metrics only; `/api/**` returns `404` |
| Backend/native DTOs | Generated from Protobuf |
| REST source | Inline `google.api.http` annotations |
| Browser contract | Canonical OpenAPI 3.0 with exactly 12 Identity, seven Member, and eight Plans operations, generated directly from Protobuf |
| Kong schema | Released annotated Protobuf source bundle with transitive imports |

## G9 exit criteria

- Stable login and refresh require no gym or Member/Plans lookup and issue no mutable gym/membership claims.
- Browser browses gyms/plans and submits explicit gym context through Kong HTTPS/JSON; no BFF exists.
- Member validates customer ownership and uses Plans for authoritative `(gym_id, plan_id)` purchase terms.
- Request gym grants no `ADMIN` authorization; G9 gym-management policy is `SUPER_ADMIN`-only.
- Identifier has no Member dependency and retains only Plans trainer-gym validation.
- Kong transcodes to Member and Plans `grpcs:50051` with a verified Kong client identity.
- Member and Plans trust end-user metadata only from Kong SAN.
- Workload-only RPCs remain direct mTLS-only and absent from OpenAPI 3.0 and Kong.
- Plans has no native `/api/**` business endpoints on `8080`; Actuator remains healthy.
- OpenAPI 3.0 is generated directly from Protobuf, deterministic, versioned, and published.
- Backend DTOs remain generated from Protobuf; optional frontend REST client is generated from OpenAPI 3.0.
- Observed error status/header/body behavior matches the published frontend contract.
- NetworkPolicies and E2E evidence prove the intended path without direct-cert test shortcuts.
