# ADR 0002: Stop U2 at the persistent-worker credential boundary

- Status: Superseded for planning by [ADR 0003](0003-trusted-lane-access-boundary.md); evidence retained
- Date: 2026-08-11
- Governs: R3-R5, R16, R21, AE4, U2, U6, U7
- Evidence: [U2 credential-containment receipt](../validation/u2-credential-containment-stop.md)
- Existing seams: [pinned-engine characterization](../validation/u2-existing-engine-seams.md)

## Context

U2 must prove that hostile repository code and the coding subprocess cannot read
or export reusable model, GitHub, or Linear credentials. The canonical plan also
requires that a lane worker cannot approve or merge a pull request. These are
explicit execution stop conditions, not release-hardening tasks that can be
deferred.

The U1 Kustomize skeleton pins the existing `symphony-k8s` worker image but does
not mount credentials because U2 owns that decision. The current image contract
cannot become operational without adding the credential bindings required by
its entrypoint.

## Finding

The existing persistent-worker topology does not satisfy the launch contract:

- `LINEAR_API_KEY` and `GITHUB_TOKEN` are injected into the worker container.
- The entrypoint writes the GitHub token to `.netrc` and the GitHub CLI
  configuration owned by the `symphony` Unix user.
- The entrypoint copies reusable Codex authentication to
  `/home/symphony/.codex/auth.json`, also owned by `symphony`.
- SSH agent commands run as that same `symphony` identity. File permissions
  protect credentials from other Unix users, but not from the untrusted coding
  process that the boundary is meant to constrain.
- A GitHub App installation token capable of pushing code needs repository
  `Contents: write`, which also authorizes GitHub's merge endpoint. Creating and
  updating pull requests needs `Pull requests: write`, which also authorizes the
  review endpoint whose accepted actions include `APPROVE`.

The pinned Symphony engine already keeps the selected tracker adapter and its
provider-native tool execution in the orchestrator, and unsets the adapter's
declared secret environment names before launching a local or remote Codex
child. That means Linear does not need to enter the worker and the current
`symphony-k8s` requirement for `LINEAR_API_KEY` can be removed without a new
product service. This existing seam narrows but does not resolve the stop:
authenticated clone and push still execute in remote hooks or agent commands,
and stock Codex app-server plus its authentication still execute on the remote
worker. See the characterization linked above.

A default-branch ruleset can deny effective default-branch updates to an app
without bypass permission, and review policy can keep an app review from
satisfying human approval. Those controls may make merge or approval ineffective
in a configured repository, but they do not meet the current literal R3 rule
that the lane worker cannot be granted those capabilities. They also do not
protect reusable credentials from repository-controlled code.

## Decision

> **Superseded planning disposition:** [ADR 0003](0003-trusted-lane-access-boundary.md) selects the trusted Git and stock-Codex execution boundary described by the first two alternatives below while preserving literal R3. This record remains the immutable reason the original image pair is not a qualified hosted-lane runtime. U2 resumes only to prove the selected boundary and stops again if any required seam fails.

Stop implementation before further U2, U3, or lane-runtime work. Do not mount
the three reusable credentials into the product worker, claim that file modes
provide containment, or weaken R3 through implementation convention.

Planning must choose and specify one of these materially different contracts:

1. Introduce a trusted, lane-scoped operation boundary that retains provider and
   model credentials and exposes only the narrow Git, GitHub, Linear, and model
   operations the coding process may perform. Repository rulesets remain an
   independent defense in depth.
2. Change the execution topology so trusted components perform credentialed
   operations while the SSH worker executes only uncredentialed workspace
   commands. Any Symphony or `symphony-k8s` change remains owned by its existing
   repository and image path.
3. Explicitly revise the product contract to accept worker-readable credentials
   and policy-based prevention of merge and approval. This is a product and
   security change, not an implementation assumption.

No option is selected by this ADR. The first two preserve the current product
contract; the third changes it. Resume requires a revised plan with an owning
implementation unit, interfaces, threat model, and executable positive and
negative acceptance cases.

## Consequences

- The current worker image remains pinned for provenance but is not a validated
  hosted-lane runtime.
- The incomplete U1 lane manifest must remain non-operational; adding raw secret
  mounts would knowingly cross the stop boundary.
- Self-hosted trusted environments may continue using the existing topology only
  under the plan's stated reduced-trust exception. That does not qualify it for
  hosted alpha or GA.
- Flux, ExternalArtifact, console, database, and product-release ownership
  decisions remain unchanged. This stop concerns the agent credential and
  provider-operation boundary.

## Primary references

- [`symphony-k8s` worker entrypoint at the credential-introduction commit](https://github.com/jasonmorganson/symphony-k8s/blob/14e7ed280779717e6f87bd23ebd050f9b2ceed5f/docker/worker/entrypoint.sh)
- [`symphony-k8s` worker StatefulSet at the same commit](https://github.com/jasonmorganson/symphony-k8s/blob/14e7ed280779717e6f87bd23ebd050f9b2ceed5f/k8s/base/worker-statefulset.yaml)
- [GitHub merge-pull-request permissions](https://docs.github.com/en/rest/pulls/pulls#merge-a-pull-request)
- [GitHub create-review permissions and review actions](https://docs.github.com/en/rest/pulls/reviews#create-a-review-for-a-pull-request)
- [GitHub ruleset update restrictions](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/available-rules-for-rulesets#restrict-updates)
