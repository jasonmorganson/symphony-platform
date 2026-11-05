# Effect platform pre-U1 gate receipt

- Date: 2026-08-11
- Verdict: READY for the selected Effect platform; historical OpenAPI direction superseded by KTD17
- Decision: [ADR 0001](../decisions/0001-effect-platform-foundation.md)

## Evidence set

The disposable source and raw evidence remain outside the repository by contract. These hashes identify the reviewed reports; durable U1 checks replace scratch behavior as implementation authority.

| Scope | Report SHA-256 | Disposition |
|---|---|---|
| Authored OpenAPI, Effect schema generation, private Next Route Handler | `51e4126a6cc668c2372807f664513f0f62fbbe0c3eae92bd97a304e3778ede25` | Historical pass; reverse direction superseded by KTD17 |
| Effect Schema, Effect SQL, RLS, context isolation | `f8d3119cc136e2f0b944acfc97dd030edcf84ffd04228153b1da5a3c9de9a393` | Pass with explicit arbitrary-SQL trust boundary |
| Selected `pg@8.22.0` SQL/RLS rerun | `a081f69a05d20a43c2e40483a88f963e9c7a9df580ae06a5c35c19d5c33694f0` | Pass with the same trust boundary |
| Better Auth, Atlas catalog, web/worker role composition | `ea7f3daee43e060adc3ea24b90168fc2ed234cd6f4a7eacccbb847bfd957121d` | Earlier platform gaps superseded by later receipts |
| Effect Workflow, outbox, retention, concurrency, beta.106/107 transition | `351e3e5099aa5cdb96b3146a0ad56b3d7989812fa938134195d7c8e76daecf35` | Pass with exact-pin retention adapter |
| Compiled Next Server Action, real Better Auth, stored authorization, Effect transaction | `653b52559d542cdbeb828c9782dc76d37dccb5de3d32512d66ba6870d8b5889f` | Pass |
| Kubernetes identities/PVCs/network, OCI transport, Node image, OTel Collector | `cffc4eee0d7202a4b2e021ba0c28aab96a48c561d83ed94baa0a01fd6f1eb305` | Pass for mechanism feasibility |
| PostgreSQL/file PVC backup, deletion, two restore paths, tombstone reconciliation, PV cleanup | `07d7d8a4bbbe25224d602b846189c52821c1940c90819d11b881ac8efac92981` | Pass |

The `pg@8.22.0` receipt hash above is the SHA-256 of `/private/tmp/symphony-effect-pre-u1.ozmFY2/VALIDATION-PG-8.22-RERUN.md` after the final rerun.

## Closed findings

- The authored-OpenAPI spike proved the selected schemas, strict decoding, canonicalization, bounds, private authentication order, and thin Next.js boundary. KTD17 now supersedes its reverse generation direction: authoritative Effect Schema and HttpApi source generates OpenAPI 3.1, and U1 must replace the old pipeline atomically. The historical hash remains evidence only for the constraints it actually exercised.
- The pre-U1 gate now distinguishes bounded mechanism feasibility from product integration and acceptance owned by U2-U16. Later-unit residuals are not waived or treated as passing.
- Direct `pg` is fixed at `8.22.0`; the core SQL/RLS suite was rerun on PostgreSQL 18.4.
- Atlas is accepted only at its exact checksummed pin.
- Shared-table RLS, transactional outbox, drain, Workflow retention, restore bootstrap, Effect version transition, OpenAPI hooks, protocol bounds, image/tool pins, and kubectl skew are fixed in ADR 0001.

## Entry disposition

No load-bearing selected-stack feasibility result failed. Exact pinned Effect beta.107 source inspection confirms `HttpApi`, 200/304 response schemas, request and response headers, `OpenApi.fromApi`, and co-located OpenAPI transform primitives needed by KTD17. That planning result is not a replacement implementation receipt: U1 starts with the authoritative contract migration and cannot complete until HttpApi execution, deterministic OpenAPI generation, required extensions, and Next bridge parity pass. Every later residual remains a blocking requirement of its owning implementation unit and the final release gate.
