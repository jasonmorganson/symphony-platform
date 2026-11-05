# Foundation threat model

## Assets and authorities

| Asset | Authority | Foundation control |
|---|---|---|
| Human session | Better Auth catalog in `auth` | Server-side session validation; separate runtime role |
| Product revision | PostgreSQL `app` schema | FORCE RLS plus transaction-local tenant context |
| Connection metadata and revocation fences | PostgreSQL `app` schema | Non-secret rows, optimistic versions, opaque references, and monotonic denial before remote cleanup |
| Manual credential material | Hosted or customer-local credential store | Installation-scoped write/version/promote/revoke/status authority; console has no value read/list/export permission |
| Managed ChatGPT authorization | Encrypted stock-Codex auth store in the trusted orchestrator | Exact pinned stock Codex owns token exchange, storage, refresh, and logout; worker and console never receive tokens |
| Workflow state | PostgreSQL `workflow` schema | Separate role and exact version-pinned Effect Workflow adapter |
| Desired-state response | Effect Schema and HttpApi in `packages/contracts` | Installation-bound identity, strict schema validation, bounded canonical response, generated OpenAPI drift check |
| Lane Kubernetes state | Flux inventory | Console has no Kubernetes token; source adapter is status-only |
| Symphony work state | Symphony | No projection or duplicate scheduler in this repository |
| Linear and model credentials | Trusted Symphony orchestrator | Values do not enter the worker, accepted revisions, artifacts, or Kustomize substitutions |
| GitHub App credentials | Lane-scoped Git broker | Broker exposes only bound Git transport and pull-request capabilities; no credential leaves the trust domain |
| Agent-controlled Git refs | Dedicated PR-origin fork in the source repository's GitHub fork network | No Actions execution, secrets, environments, deploy keys, or deployment integration |

## Primary exploit paths and mitigations

| Threat | Mitigation | Owning verification |
|---|---|---|
| A client supplies another tenant or role | Derive tenant, membership, and role from the live server session; validate again inside every domain command | U1 action boundary; U11A-U11B permission suite |
| A missing SQL predicate crosses tenants | FORCE RLS with transaction-scoped context and catalog/runtime conformance | U1 database gate |
| Arbitrary SQL reassigns the transaction GUC | Treat the shared runtime role as a defense against omissions, not arbitrary SQL execution; deny raw query capability to user input and monitor injection paths | U1 ADR; U16 injection suite |
| A browser reaches the controller contract | Exclude `/internal/v1/` from public ingress; verify installation-bound workload identity before application execution | U1 handler seam; U2 identity and NetworkPolicy |
| A controller mutates bindings or lanes | Installation owns specifications; controller RBAC permits status-only writes and read-only condition observation | U3 controller contract; U4 live RBAC negatives |
| A secret enters desired state | Authoritative Effect schemas contain names and references only; strict excess-property rejection, generated-OpenAPI parity, and response scanning | U1 contracts; U11A-U11B end-to-end fixtures |
| A secret leaks during Admin submission | Stream one validated Admin-only TLS action into a write-only adapter; never echo or persist the field; disable request-body capture; scan browser state, SQL, caches, outbox, Workflow, logs, traces, errors, crashes, exports, and backups with canaries | [ADR 0004](../decisions/0004-connection-credential-custody.md); U2 writer conformance; U11A-U11B console suite; U16 sink suite |
| Application compromise reads the credential store | Give the console identity create/version/promote/revoke/status only, no value read/list/export or runtime mount; keep trusted-consumer projection under separate identities and paths | U2 policy conformance; U4 live identity/RBAC/network negatives |
| OAuth callback or device ceremony is mixed across tenant, issuer, or session | Bind state, PKCE, issuer, redirect, Admin session, organization, installation, Connection, nonce, expiry, and requested scope; honor device polling and one-time-code rules; reject replay and duplicates | U2 provider/Codex protocol suite; U11B browser suite for enabled optional modes |
| Internal or imported ChatGPT tokens bypass stock Codex lifecycle | Permit only pinned stock-Codex `chatgpt` and `chatgptDeviceCode`; reject `chatgptAuthTokens`, browser cookies, and imported `auth.json`; configure encrypted storage and suppress or redact login logs | ADR 0004; U2 exact-version schema/runtime gate |
| Rotation or rollback restores stale authority | Validate replacement before compare-and-swap promotion, never fall back, commit a monotonic fence before remote cleanup, and recheck current status at every trusted use; historical revisions contain no authority | U2 race/crash suite; U11A-U11B lifecycle suite; U12 restore suite |
| A mutable or unproven Symphony artifact enters a release | External images are digest-pinned and tied to the recorded fork/base revisions; no Symphony source exists here | U1 provenance; U6 black-box compatibility |
| Repository-controlled code reads reusable provider or model credentials | Keep Linear and stock-Codex authentication in the trusted orchestrator, run every repository-controlled operation only through the uncredentialed remote exec-server worker, keep GitHub authentication in the broker, and deny direct provider/model egress and local execution fallback | [ADR 0003](../decisions/0003-trusted-lane-access-boundary.md); U2 credential-canary suite; U7 hostile-runtime suite |
| A worker invokes approval, merge, deploy, or generic GitHub APIs | Broker exposes only leased source/PR-origin Git transport and PR read/create/update; no token or generic API reaches the worker; repository rulesets remain defense in depth | U2 action-parity negatives; U17 broker suite |
| Git protocol ambiguity expands the leased ref capability | Parse receive-pack commands before upstream credential injection; reject tags, deletes, force pushes, protected refs, unknown capabilities, hosts, repositories, and ambiguous framing | U2 protocol spike; U17 fuzz and parity suite |
| A generic pull-request update broadens authority | Bind creation to the verified fork-network identity, immutable head ref/SHA, and approved base ref; allow updates to title and body only; deny base, head, state, reviews, merge settings, deployment fields, and unknown properties | U2 action-parity contract; U17 field-level negatives |
| Agent-controlled code reaches deployment indirectly through CI or integrations | Isolate work refs in a PR-origin fork with no execution or deployment capability and reject base-repository Actions, Apps, webhooks, environments, deployment integrations, or automated rules that can execute or deploy agent-controlled PR content | U2 onboarding analysis; U11A repository-change revalidation; U16 hostile integration fixtures |
| A hostile worker exhausts the credentialed Git broker | Enforce bounded concurrent streams, duration, idle timeout, backpressure, pack/request bytes, per-lease rates, cancellation, and lane-scoped resource quotas; exceeding a bound cannot leave a partial ref | U17 resource-exhaustion and interrupted-stream suite |
| Stock Codex falls back from remote execution to the credentialed orchestrator | Pin Codex and generated app-server schema, require the selected remote environment for every filesystem/process action, deny local execution, and fail closed if the environment is unavailable | U2 exact-version conformance; U6 black-box compatibility |
| Workflow bootstrap exposes a repository credential or mutable file | Symphony requests only the fixed `WORKFLOW.md` path at the bound immutable revision through the broker using trusted lane identity before polling; the product stores no contents and wrong repository/revision/path fails closed | U2 workflow-source gate; U6 black-box loading suite |
| The broker or trusted orchestrator is compromised | Keep them as separate least-privilege identities and network zones; broker has no model/tracker/console state, orchestrator has no GitHub credential, and both lack lane-resource write authority | U4 secret/RBAC/network tests; U16 compromise-path suite |
| Web and workflow compromise spreads laterally | Separate database roles, credentials, health endpoints, and entrypoints; no Kubernetes API token | U1 topology; U12 deployment isolation |

## Explicit trust boundary

PostgreSQL RLS protects application queries that omit tenant predicates. A role
that can execute arbitrary SQL can call `set_config`, so RLS is not claimed as
an arbitrary-SQL sandbox. Preventing attacker-controlled SQL and maintaining
the least-privilege role split are independent required controls.

The lane worker is explicitly untrusted even though it is namespace- and
storage-isolated. The trusted orchestrator, Git broker, console, credential
writer, secret store, source controller, Flux reconcilers, and lane applier are
separate authorities; trust in one does not imply credentials or permissions
from another. The console is trusted to authorize one credential write but is
not trusted to retrieve the resulting value. Neither file permissions nor a
short-lived credential returned to the worker qualifies as credential
containment.
