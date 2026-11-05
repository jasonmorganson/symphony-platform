# ADR 0001: Effect platform foundation

- Status: Accepted
- Date: 2026-08-11
- Governs: KTD11, KTD17, U1, U11A, U11B, U12, U15, U16

## Context

The console needs one SQL-first TypeScript platform for authenticated human commands, private desired-state projection, durable product workflows, PostgreSQL persistence, and one independently operated web/worker image. The selected Effect composition was validated before U1 with beta dependencies explicitly allowed. This record fixes the accepted pins and the compatibility boundaries found by that gate.

## Decision

### Exact launch pins

| Component | Pin |
|---|---|
| Node.js | `26.7.0` |
| pnpm | `11.7.0` |
| Next.js | `16.3.0` |
| React / React DOM | `19.2.4` |
| TypeScript | `6.0.2` |
| Better Auth | `1.6.27` |
| Effect | `4.0.0-beta.107` |
| `@effect/platform-node` | `4.0.0-beta.107` |
| `@effect/sql-pg` | `4.0.0-beta.107` |
| direct `pg` dependency | `8.22.0` |
| PostgreSQL baseline | `18` |
| Atlas | `v1.3.0` |

Atlas uses the stable `v1.3.0` release. Its platform binary checksums are locked in `mise.lock`; changing Atlas or any Effect pin requires rerunning the owning migration or compatibility gate before adoption.

The validated container baseline is `node:26.7.0-bookworm-slim@sha256:cd565714d4da3e84bfd341e31448f81d47c6362198f152345297c9c1154e6341`. U1 may rebuild the console above this base; release images remain multi-platform and digest-addressed.

### Database and tenant topology

- Hosted installations initially use shared `app` tables with tenant identity on every tenant-owned row.
- Tenant tables enable and force PostgreSQL row-level security.
- Runtime roles are non-owner, `NOINHERIT`, non-superuser, and have no `BYPASSRLS` or schema DDL.
- The server derives actor, tenant, installation, trace, and execution context from authenticated state and sets it transaction-locally before tenant SQL.
- Every transaction resets context before a pooled connection is reused.
- Product SQL is reviewed and parameterized trusted application code. A SQL-capable compromised process can replace a custom GUC; RLS protects missing predicates and pooled-context leakage, not arbitrary SQL execution after process compromise.

Atlas executes production DDL for `app` and `auth`. Better Auth's supported `getMigrations` API produces reference SQL in disposable validation; changes are reviewed into Atlas migrations, replayed into a clean database, and compared bidirectionally with the expected Better Auth catalog. Better Auth does not migrate at startup and no private `dist` import is allowed.

Atlas creates the isolated `workflow` schema and roles, but pinned Effect Workflow owns runtime DDL inside that schema. The workflow role has no authority over `app`, `auth`, or `public`. Web, workflow-internal, workflow-activity, retention, and migration connections use distinct roles and credentials.

Restore procedures must recreate database roles, database-local search paths, and least-privilege grants before web or workflow processes start.

### Workflow enrollment and retention

`Workflow.execute` cannot enlist in the caller's Effect SQL transaction. Accepted revisions therefore use a transactional outbox:

1. Commit accepted state and the outbox row atomically.
2. Claim without holding a database lock across remote enrollment.
3. Enroll under a deterministic execution identity.
4. Acknowledge enrollment and later reconcile terminal evidence idempotently.

Drain means all three are zero: unacknowledged enrollment, enrolled nonterminal executions, and completed outbox rows missing immutable terminal evidence.

Effect Workflow retention is an explicit version-pinned compatibility adapter:

- Use separate least-privilege application and Workflow clients.
- Use exported unstable `MessageStorage.clearAddress`; product code never deletes `cluster_*` rows directly.
- Mirror the private main/clock address convention only inside the adapter and protect it with an exact-source sentinel.
- Hold the accepted-revision row lock while polling terminal state and atomically recording the tombstone and pending retention intent.
- After the intent exists, retries do not repoll state that may already have been cleared.
- Repeated main and clock clears are idempotent; receipt insertion and intent completion commit atomically.

Only the fully isolated `beta.106 -> beta.107 -> beta.106` database and payload transition is approved. The beta.106 validation workspace must override transitive `@effect/platform-node-shared` to exact beta.106. Every future Effect version change fails closed until the source sentinel, concurrent dispatch/reconciliation, four-point retention crash matrix, restore, and old/new/old transition pass again.

### Effect HttpApi and private HTTP

The desired-state Effect Schema and `HttpApi` definition in `packages/contracts` are the sole wire authority. Exact pinned Effect beta.107 generates OpenAPI 3.1 through `OpenApi.fromApi`. The endpoint definition owns the method, path, request headers, authenticated installation context, 200 response with required `ETag`, 304 response, errors, and protocol bounds. TypeScript declarations are inferred from those schemas; the repository does not generate Effect source from OpenAPI.

A co-located Effect `OpenApi.Override` or `OpenApi.Transform` supplies document-only metadata that the standard generator cannot infer, including `x-kubeprojection-resource`. Generation fails if the named lane component or its projection metadata is absent or changes. The generated OpenAPI document is committed for the external controller, regenerated deterministically, and drift-checked; it is never edited as an independent contract.

The single Next.js desired-state Route Handler is a thin framework bridge to a handler built from the same `HttpApi`. Authentication middleware verifies controller workload identity and installation scope before application execution. A route-parity test prevents the filesystem route from changing the method, path, authentication order, statuses, headers, bounds, or body. No generated general HTTP server or client is required.

The selected wire capability limits are 8 MiB per canonical response and 2,048 lanes per installation response. They are protocol bounds, not a hosted scale promise.

### Tooling and release mechanics

The validation tool baseline is Skaffold `v2.24.0`, kind `0.32.0`, ORAS `1.3.3`, crane `0.21.9`, and Kubernetes/kubectl `1.36.1` for disposable cluster tests. CI must pin kubectl within supported server skew; it cannot inherit the validated host's older `1.33.9` client.

The initial external engine pins remain:

- Symphony revision `bd77c48d5800e9a849eeb2bf2aa29cf06abb121e`
- OpenAI base revision `f8e8b8a670c799f6e0ade7a8c25c4bf4a4a56ec7`
- orchestrator `ghcr.io/jasonmorganson/symphony-k8s-orchestrator@sha256:a102715003173717f17b17cacd74cfc41275b4ded7738f1bd1713b912d1d3254`
- worker `ghcr.io/jasonmorganson/symphony-k8s-worker@sha256:1a72392f97acd2acdfe1e520a39456b3d916d918b122f0011fff646887994ae8`

The product repository never builds these external images or contains their source.

## Later unit gates

Pre-U1 evidence proves stack mechanisms, not product acceptance. The following remain fail-closed:

- U2: real workload identity, ingress, provider OAuth/PKCE/redirect behavior, and external-component trust.
- U3: controller contract, digest agreement, conditions, and production RBAC.
- U4: production Kustomize/Flux package, storage and artifact service, and disconnected source topology.
- U11A: required API-key SQL-backed desired-state projection, console behavior and accessibility, private ingress, and product telemetry.
- U11B: optional Business/Enterprise access-token and managed-ChatGPT breadth after their independent conformance gates pass.
- U12: same signed release across hosted, self-hosted, and disconnected acceptance.
- U15: product export and deletion Workflow.
- U16: release-scale security, portability, rollback, deletion, and telemetry validation.

## Consequences

The stack is selected and comparisons are closed. The earlier authored-OpenAPI-to-Effect validation established useful schema constraints but is superseded as the contract direction; U1 must replace that reverse pipeline atomically with the authoritative `HttpApi` pipeline. The private Workflow retention seam is deliberately narrower than a supported stable public API, so exact pinning and revalidation are permanent release obligations. Shared-table RLS reduces common tenant-query failure modes but does not replace process isolation, least privilege, reviewed SQL, or hostile end-to-end tests.
