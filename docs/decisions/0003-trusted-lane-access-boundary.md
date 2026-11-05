# ADR 0003: Trusted lane access boundary

- Status: Accepted for planning; U2 conformance required
- Date: 2026-08-11
- Governs: R3-R5, R14-R18, R21, R26, AE4, KTD7-KTD9, KTD18, KTD20, KTD23-KTD26, U2, U4, U6, U7, U17
- Supersedes for planning: [ADR 0002](0002-u2-credential-boundary-stop.md)
- Baseline evidence: [credential stop](../validation/u2-credential-containment-stop.md) and [pinned-engine seams](../validation/u2-existing-engine-seams.md)

## Context

The original persistent worker stores GitHub, Linear, and stock-Codex credentials under the same Unix identity that executes repository-controlled code. GitHub App permissions broad enough for direct branch push and pull-request mutation also expose merge or review endpoints. File modes and repository rules reduce effective risk but do not satisfy literal R3: the lane worker must not receive those capabilities.

The selected design must preserve upstream Symphony as scheduler, tracker reader, workspace owner, retry authority, and stock-Codex app-server runner. It must not add a model gateway, tracker proxy, second agent loop, or product run controller. Symphony source changes remain in `jasonmorganson/symphony`; worker and orchestrator images remain built through `symphony-k8s`; no automation proposes changes to `openai/symphony`.

## Decision

Split the lane into four explicit trust domains:

1. **Trusted orchestrator.** Symphony retains the Linear credential, model credential, tracker adapter, and stock Codex app-server. It authenticates directly to OpenAI and conducts the model session. The pinned app-server remote-environment seam delegates every repository-controlled filesystem and subprocess operation to the worker's exec-server. Local execution and silent fallback are disabled.
2. **Untrusted execution worker.** The persistent worker owns only its lane workspace, SSH/exec-server identity, and restricted process environment. It receives no Linear, GitHub, or model credential and has no direct Linear, GitHub, or model endpoint egress. Repository code, tools, skills, hooks, and command children are all treated as hostile.
3. **Trusted installation issuer.** A separate entrypoint and Deployment from `apps/git-broker` alone receives sign-only GitHub App-key access. It validates a fresh installation-bound no-deploy attestation and mints distinct bounded tokens for distinct source and PR-origin installation identities; no token spans both repositories.
4. **Trusted lane Git broker.** A lane-scoped Deployment from the same minimal image receives no App key. It holds the separate source and PR-origin installation tokens only in memory and exposes only the lane-scoped Git and pull-request capability surface. It has no console database, human-session, tracker, model, prompt, scheduler, retry, workspace, or Symphony run-state authority.

Before polling, Symphony uses trusted lane identity to retrieve only the fixed `WORKFLOW.md` path at the configured immutable repository revision through a bootstrap-safe broker operation. The product stores only that reference and never stores or renders the file. Because the pinned Symphony build accepts a local workflow path, U2 must prove the generic loading seam and U6 must qualify the minimal fork-owned change and new external image digests.

For run operations, Symphony issues a short-lived security lease from trusted lane, issue, source-repository, verified PR-origin fork-network identity, onboarding-approved base ref, immutable PR head ref and SHA, and work-ref context already present in its lifecycle. The worker presents that opaque lease to the broker; repository input cannot select or expand any binding. The broker never returns a GitHub token or generic REST/GraphQL credential.

The issuer validates a fresh content-addressed no-deploy attestation before bootstrap authorization, run-lease issuance, or either source/PR-origin token mint. Relevant repository configuration changes invalidate it. The broker persists no lease; an isolated durable authority epoch fences issuance and use, and a trusted auth supervisor plus every broker instance must acknowledge live Codex, stream, and lease termination before local revocation is complete. All orchestrator, worker, broker, and issuer links use pinned-root mTLS with installation-and-lane-bound identities, and worker Git remotes use HTTPS.

The broker permits only:

- clone and fetch from the bound source repository;
- clone and fetch from the lane's dedicated PR-origin fork;
- creation or fast-forward update of the leased PR-origin work ref;
- idempotent read and creation of the corresponding pull request with the lease-bound head/base tuple;
- update of pull-request title and body only.

The broker rejects another repository, host, fork network, ref namespace, tag, delete, non-fast-forward update, protected ref, lease, installation, or pull request. It rejects pull-request head, base, state, review, merge-setting, deployment-field, and unknown-property mutation. It exposes no approve, review, merge, deployment, repository-administration, secret, environment, billing, deletion, or generic provider operation. Git smart-HTTP receive commands are parsed and authorized before upstream credential injection; unknown protocol capabilities or ambiguous framing fail closed. Concurrent streams, duration, idle time, request and pack bytes, per-lease rate, upstream cancellation, and lane resource consumption are bounded to the supported repository envelope.

Agent-controlled refs live in a dedicated PR-origin fork with no Actions execution, secrets, environments, deploy keys, or deployment integration. Onboarding verifies GitHub fork-network membership and private-fork organization/enterprise eligibility. It rejects any base-repository Action, GitHub App, webhook, environment, deployment integration, or automated rule reachable from pull-request creation or update that can execute or deploy agent-controlled content with privileged authority, and configuration drift revokes new operations. Repository rulesets remain independent defense in depth, not the capability boundary.

Linear remains Symphony-owned, but repository-controlled sessions receive only issue-scoped commands bound to Symphony's selected issue and configured review state; they never receive raw GraphQL, a token, or a generic tracker client. The worker has no direct package-registry or general internet egress and resolves dependencies only through installation-owned controlled HTTPS mirrors or authenticated egress proxies.

The product monorepo owns the broker because its narrow product capability contract is the reason for the separate trust domain. Symphony or packaging changes required to originate and propagate leases, host stock Codex app-server, and launch the uncredentialed exec-server remain in their existing repositories and are consumed here only as immutable image digests.

## Fail-closed conformance gate

U2 must prove the exact pinned stock-Codex remote-environment and disposable protocol contracts before U17 or a replacement image pair is accepted. U17 owns the first production issuer, broker, epoch, attestation, and supervisor implementation; U6 owns every Symphony integration and qualified image pair. Tests cover model and provider canaries; `/proc`, file-descriptor, environment, file, log, trace and crash-artifact access; direct model/provider egress; local execution fallback; issue-scoped Linear commands and raw-GraphQL denial; distinct source/fork identities and tokens; fresh attestation; durable epoch startup and restore; supervisor acknowledgements; wrong-lane and expired leases; broker credential extraction; Git protocol fuzzing; another repository or ref; protected-ref, delete and non-fast-forward pushes; pull-request action parity; merge, review and deployment API denial; GitHub App removal and rotation; controlled dependency mirrors; and PR-origin indirect-deployment paths.

The gate fails if the worker can obtain reusable or short-lived provider credentials, stock Codex cannot keep authentication outside its command environment, the broker cannot validate the complete Git transport command before credentialed forwarding, or the PR-origin no-deploy prerequisite cannot be enforced. Failure returns to planning; it cannot fall back to worker secrets, a token-returning credential helper, local Codex execution, a model gateway, or repository policy alone.

## Consequences

- Literal R3 is preserved as a capability boundary instead of weakened to effective repository policy.
- The persistent worker topology remains, but the existing worker and orchestrator digests are provenance inputs rather than qualified hosted-lane artifacts.
- `apps/git-broker` becomes the second launch application and image in the product monorepo because it has an independent security boundary.
- Linear credentials and normalization remain entirely inside Symphony; coding sessions receive only the generic issue-scoped commands qualified in U2 and adopted through U6.
- Stock Codex remains the model-session implementation; the product does not proxy prompts or model traffic.
- U17 owns the permanent broker and lease contract only after U2 proves the boundary. U6 adopts the resulting immutable external image pair, U4 packages the trust domains, and U7 proves hostile runtime containment.

## Primary references

- [Git smart HTTP protocol](https://git-scm.com/docs/http-protocol)
- [GitHub App repository permissions](https://docs.github.com/en/apps/creating-github-apps/registering-a-github-app/choosing-permissions-for-a-github-app)
- [GitHub App installation tokens](https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/generating-an-installation-access-token-for-a-github-app)
- [GitHub Actions security guidance](https://docs.github.com/en/actions/reference/security/securely-using-pull_request_target)
- [Codex app-server](https://github.com/openai/codex/blob/main/codex-rs/app-server/README.md)
- [Codex exec-server](https://github.com/openai/codex/blob/main/codex-rs/exec-server/README.md)
