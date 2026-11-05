# U2 pinned-engine credential-seam characterization

- Date: 2026-08-11
- Verdict: PARTIAL EXISTING SEAM; baseline retained for selected-boundary U2
- Symphony revision: `bd77c48d5800e9a849eeb2bf2aa29cf06abb121e`
- Governing stop: [ADR 0002](../decisions/0002-u2-credential-boundary-stop.md)
- Selected planning boundary: [ADR 0003](../decisions/0003-trusted-lane-access-boundary.md)

## Question

Can the current U2 failure be resolved through an existing public Symphony or
`symphony-k8s` boundary while keeping the pinned persistent SSH-worker topology,
stock Codex, and the current product contract?

## Exact-source findings

The selected Symphony revision was fetched by immutable commit and inspected
without changing the fork.

| Operation | Current owner and execution location | Credential result |
|---|---|---|
| Linear polling and normalization | Symphony orchestrator through the selected Linear adapter | Linear token can stay out of the worker |
| `linear_graphql` agent tool | Symphony orchestrator handles the Codex dynamic-tool request host-side | Linear token can stay out of the Codex child |
| Tracker-secret child launch | `Codex.AppServer` removes the selected adapter's declared secret environment names from local port environment and prepends `unset` to remote launch | Existing generic tracker seam passes source characterization |
| GitHub tracker `github_api` tool | Symphony orchestrator when GitHub is the selected tracker adapter | GitHub token can stay host-side for that adapter's REST calls |
| Launch combination of Linear tracker plus GitHub repository | Only the selected tracker adapter contributes dynamic tools | Existing binding does not provide both `linear_graphql` and `github_api` in one session |
| Workspace creation hook | Symphony sends the repository-owned hook over SSH and executes it on the worker | Private clone still needs a remote Git transport boundary |
| Branch commit and push | Agent shell and Git run in the remote workspace | Authenticated push still needs a remote Git transport boundary |
| Codex app-server | Symphony launches `codex.command` over SSH in the remote workspace | Stock Codex authentication must still be available to a process on the worker |

The existing tracker seam is generic and preserves Symphony's scheduling
authority. Reusing it does not create a second scheduler, tracker reader, or
agent loop.

## What this resolves

`LINEAR_API_KEY` is not a worker prerequisite of the pinned Symphony engine.
The `symphony-k8s` worker entrypoint currently requires it, but its removal is a
small packaging correction: the orchestrator retains Linear polling and tool
execution, and the remote Codex launch explicitly unsets the tracker secret.

This also shows the shape of an acceptable provider-operation boundary:
credentialed operations execute in the trusted orchestrator and Codex receives
only structured tool results. A future generic Symphony change could allow
multiple provider-tool bindings independently of the selected tracker, but that
would not by itself solve authenticated Git clone and push.

## What remains unresolved

### Git transport

Private clone and branch push are ordinary shell/Git operations on the SSH
worker. The existing image makes them work by storing `GITHUB_TOKEN` in `.netrc`
and GitHub CLI state owned by the agent identity. Host-side `github_api` cannot
replace Git's authenticated object transport. A compliant solution needs a
narrow remote Git credential or transport boundary, plus repository policy that
denies protected-branch updates.

### Model authentication

The SSH extension launches the Codex app-server itself on the worker. The
existing image copies reusable Codex authentication into the same Unix home
used by agent commands. Tracker-secret stripping does not cover Codex's own
authentication, and the selected Symphony source exposes no separate remote
model-auth operation. A compliant solution therefore needs an execution or
sandbox boundary that lets stock Codex authenticate while preventing its
repository-controlled command children from reading or exporting that material.
It cannot silently become a proprietary model gateway under KTD8.

### GitHub forbidden operations

Even a short-lived GitHub installation token with `Contents: write` and `Pull
requests: write` exposes the merge and approval endpoints at the token-permission
layer. Protected-branch rules can make the operations ineffective, but planning
must decide whether R3 remains a literal capability boundary or is revised to an
effective-policy boundary.

## Evidence limits

- This is exact pinned-source characterization, not a successful hosted-lane
  credential-canary run.
- The checked-in Symphony tests cover local tracker-secret stripping and the
  remote launch command's `unset` behavior, but they were not rerun for this
  receipt because the local detached checkout does not have its locked
  Erlang/Elixir toolchain installed.
- No raw credential was placed into a hostile repository fixture.
- No source, image, or runtime change was made in `jasonmorganson/symphony` or
  `symphony-k8s`.

## Disposition

The existing seam justifies removing Linear credentials from the worker. KTD18
and ADR 0003 now own Git transport, model-auth isolation, and literal R3
capability semantics together: Linear stays in the trusted orchestrator, Git
uses a lane-scoped broker and deployment-isolated PR origin, and stock Codex
app-server stays in the trusted orchestrator while an uncredentialed worker
exposes only the pinned remote exec-server environment. This receipt is source
characterization, not proof of that design. U2 may resume to run the selected
positive and negative conformance suite, but the stop fires again if the exact
Codex seam, broker, or no-deploy repository topology cannot pass.
