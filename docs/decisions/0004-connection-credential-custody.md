# ADR 0004: Connection administration and credential custody

- Status: Accepted for planning
- Date: 2026-08-12
- Governs: R4-R5, R10-R12, R22, R24, R27-R29; KTD11-KTD12, KTD18-KTD19; U1-U2, U4, U11A-U12, U15-U16
- Complements: [ADR 0003](0003-trusted-lane-access-boundary.md)

## Context

An organization Admin needs one console experience for connecting GitHub, Linear, and OpenAI while hosted and self-hosted installations use different secret stores. The product must support manual OpenAI API keys or Business/Enterprise Codex access tokens and, when qualified, allow an eligible user to authorize stock Codex with a ChatGPT subscription. Lane configuration and historical rollback need stable references, but neither the console database nor the untrusted worker may become credential authority.

The original phrase “the console never receives credentials” was too broad. A useful self-service console needs a narrowly authorized secret-entry request, yet ordinary application code must never gain a reusable read path. Managed ChatGPT authorization also has a distinct owner: stock Codex already implements its login, storage, refresh, notification, and logout protocols, so reproducing that lifecycle would create a proprietary token broker.

## Decision

### Connections are non-secret trust objects

The console stores organization-scoped Connection records containing provider, explicit authentication kind, expected and observed external account and workspace or installation identity, approved scopes, lifecycle status, active opaque credential-version and runtime-projection references, validation and rotation timestamps, dependent lanes, optimistic version, revocation fence, normalized health, and audit history. The implementation plan's normative lifecycle and health tables govern dispatch eligibility; only `active` with matching identity/scope/fence and `valid` or bounded `expiring` health is eligible.

Credential values, OAuth access or refresh tokens, device codes, authorization headers, browser cookies, Codex `auth.json`, and provider response bodies cannot enter PostgreSQL, accepted revisions, desired-state responses, KRM artifacts, Workflow payloads, logs, traces, errors, exports, ordinary backups, or ordinary application reads.

### Human authorization and agent boundaries

Only a current organization Admin may create, authorize, reauthorize, rotate, change kind or scope, disconnect, or revoke a Connection. The console derives organization and role from the Better Auth server session for every command. Members see only lane-relevant normalized status and remediation ownership. Time-bound hosted support sees only explicitly approved non-secret state. Coding agents receive narrow provider operations through the trusted orchestrator or Git broker and normalized failures; they receive neither Connection inventory nor credential-management operations.

OAuth consent, device-code completion, secret entry, kind changes, scope expansion, rotation, disconnect, and revocation remain human ceremonies. No agent tool can initiate or complete them.

### Manual credential path

A manual API key or access token crosses one Admin-only TLS Server Action into an installation-bound credential-store adapter. The adapter exposes only idempotent create-version, validate-through-owner, compare-and-swap promote, revoke, and non-secret status operations. It has no application-level value read, list, export, or runtime-mount method. The action returns only an opaque version reference and receipt, clears the submitted field, and never serializes the value into its result or an ancillary system.

Hosted installations bind this port to the operator's vault. Self-hosted installations bind the same port to a customer-local store or use operator-preprovisioned selectable references. Runtime projection is a separate consumer-side contract; Secrets Store CSI may implement projection and rotation, but it is not a credential writer or lifecycle authority.

### Stock-Codex-managed ChatGPT authorization

The exact pinned stock-Codex app-server is the sole owner of ChatGPT browser OAuth and device-code authorization. The console calls an authenticated, installation-bound control channel to start, cancel, observe, or log out. Stock Codex exchanges, stores, refreshes, and deletes tokens in an encrypted orchestrator-only auth store. The console stores only an opaque ceremony identifier, expiry, terminal status, and resulting non-secret account/workspace identity.

The launch model-authentication kinds are explicit:

1. A customer-owned OpenAI API key through stock Codex's documented API-key login shape.
2. A Business or Enterprise Codex access token through the documented personal-access-token input path. It is not represented as a fourth app-server login RPC.
3. Stock-Codex-managed `chatgpt` browser OAuth or `chatgptDeviceCode`; device code is preferred for hosted Pods.

Internal `chatgptAuthTokens`, imported browser cookies, imported `auth.json`, and automatic fallback across kind, account, workspace, or version are forbidden. Codex credential storage is configured explicitly; a plaintext storage fallback or unredacted `codex-login.log` fails release validation.

Account metadata such as `planType` and rate-limit observations is informational only. U2 enables a mode only after the exact pinned version completes a real run with the required account, workspace, seat, role, permission, containment, and acceptable hosted commercial-use evidence.

### Provider OAuth

GitHub and Linear OAuth callbacks remain narrow provider-protocol Route Handlers. A ceremony binds state, PKCE where supported, issuer, exact redirect URI, current Admin session, organization, installation, Connection, requested scope, nonce, and idempotency key. Token exchange writes directly to the selected custody adapter and retains only opaque references and non-secret provider identity, scope, expiry, and validation evidence.

Device authorization follows provider interval, `slow_down`, expiry, cancellation, denial, replay, and duplicate-completion rules. The UI displays the exact verification host, warns that the user code is secret and one-time, and persists no user or device code.

### Lifecycle and consistency

Connection activation is a staged saga rather than a claimed cross-system transaction:

1. Reserve inactive metadata and an idempotency key.
2. Create an immutable credential version or complete stock-Codex authorization.
3. Validate it through the owning trusted consumer using the least-authority operation.
4. Compare the allowlisted observed account/workspace or installation identity with the Admin's non-secret expectation. On mismatch, stop activation and require explicit acceptance plus revalidation or a new authorization; never overwrite silently. Active identity drift advances the fence, blocks dependents, and requires reauthorization or reassignment.
5. Compare-and-swap the active reference if the optimistic version and revocation fence remain current.
6. Retire the prior version and reconcile unreachable versions.

Rotation validates and promotes a replacement before retiring the prior version. Failure never triggers fallback. Cleanup failure produces a non-secret remediation record. Runtime consumers must demonstrate that they reload or remount the promoted version without exposing it to the console or worker.

Emergency revocation commits a monotonic local denial fence before remote provider revocation or stock-Codex logout. The fence immediately blocks new lane acceptance, leases, refresh, auth operations, and provider calls. A trusted auth supervisor terminates affected stock-Codex processes and broker streams or leases; local containment is complete only after required acknowledgements, otherwise it remains `action-required`. Remote cleanup is idempotent and its uncertainty is reported honestly. Disconnect shows dependent lanes and normally requires pause or reassignment; forced disconnect makes them `action-required`.

Desired-state rollback restores configuration, not authority. Every trusted use checks current Connection status and the revocation fence, so a historical revision cannot reactivate retired, revoked, disconnected, identity-changed, or scope-invalid authority. Restore applies fences and metadata before enabling work and proves deleted credential material does not reappear.

## Consequences

- The console offers self-service Connections without becoming a general secrets manager or token broker.
- PostgreSQL remains useful for durable lifecycle, audit, concurrency, and rollback references while credential material stays out of relational state.
- Hosted and self-hosted installations share product behavior but bind different writer, store, and projection adapters.
- Stock Codex remains responsible for its own managed ChatGPT token lifecycle and may be upgraded only after exact-version conformance.
- Connection mutations require crash, replay, race, cleanup, revocation, restore, and redaction tests beyond ordinary provider happy paths.
- A credential-store implementation without write-without-read application authority, or a stock-Codex mode without proven containment and commercial eligibility, is disabled rather than weakened.

## Rejected alternatives

- **Store encrypted credentials in PostgreSQL.** This gives the console and its backup path a reusable read authority and collapses metadata with custody.
- **Return a short-lived provider token to the worker.** Repository-controlled code could still extract and replay it; duration does not satisfy credential containment.
- **Build a ChatGPT token broker in the console.** This duplicates stock Codex authentication and assumes ownership of refresh and compatibility behavior.
- **Import browser cookies, `auth.json`, or internal token objects.** These bypass explicit provider and stock-Codex lifecycle contracts.
- **Use one secret-store API with broad CRUD.** A generic read method makes application compromise materially worse and is unnecessary for self-service entry.
- **Let rollback restore the credential version recorded by an old revision.** This would allow configuration history to override revocation and incident response.

## Verification obligations

U2 and later acceptance must prove:

- Admin-only, tenant-bound, current-session authorization for every Connection mutation.
- Write-without-read credential-store policy and absence of secret values from every application and recovery surface.
- Exact pinned stock-Codex API-key, managed browser OAuth, device-code, personal-access-token, storage, refresh, logout, notification, and containment behavior.
- OAuth state, PKCE, issuer, redirect, tenant, Connection, nonce, polling, expiry, denial, cancel, replay, and duplicate-callback handling.
- Idempotent activation, optimistic concurrency, validate-before-promote rotation, no fallback, cleanup remediation, revocation-fence ordering, disconnect dependencies, projection reload, and rollback/restore non-resurrection.
- Hosted-vault and customer-local-store behavioral parity.
- Expected-versus-observed identity mismatch, active identity drift, fail-closed dependent-lane handling, and auditable explicit resolution.
- Starter, Team, and Growth fixtures in which every managed-ChatGPT lane has an independent eligible authorization and encrypted store; bundle capacity never implies pooled subscription entitlement.

## Sources

- [Codex authentication](https://developers.openai.com/codex/auth)
- [Codex app-server authentication endpoints](https://learn.chatgpt.com/docs/app-server#auth-endpoints)
- [Codex access tokens](https://learn.chatgpt.com/docs/enterprise/access-tokens)
- [GitHub installation access tokens](https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/generating-an-installation-access-token-for-a-github-app)
- [Linear OAuth 2.0](https://linear.app/developers/oauth-2-0-authentication)
- [OAuth 2.0 Device Authorization Grant](https://www.rfc-editor.org/rfc/rfc8628)
- [OAuth 2.0 Security Best Current Practice](https://www.rfc-editor.org/rfc/rfc9700)
- [OAuth 2.0 Token Revocation](https://www.rfc-editor.org/rfc/rfc7009)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)
