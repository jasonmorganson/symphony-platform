# U2 credential-containment stop receipt

- Date: 2026-08-11
- Verdict: STOPPED before credential injection or hosted lane execution
- Decision: [ADR 0002](../decisions/0002-u2-credential-boundary-stop.md)
- Selected planning boundary: [ADR 0003](../decisions/0003-trusted-lane-access-boundary.md)
- Existing seams: [pinned-engine characterization](u2-existing-engine-seams.md)
- Scope: existing persistent `symphony-k8s` worker topology and the product's
  pinned worker artifact

## Tested artifact and source record

The product foundation records:

| Item | Immutable value |
|---|---|
| Symphony fork revision | `bd77c48d5800e9a849eeb2bf2aa29cf06abb121e` |
| OpenAI base revision | `f8e8b8a670c799f6e0ade7a8c25c4bf4a4a56ec7` |
| Orchestrator image | `ghcr.io/jasonmorganson/symphony-k8s-orchestrator@sha256:a102715003173717f17b17cacd74cfc41275b4ded7738f1bd1713b912d1d3254` |
| Worker image | `ghcr.io/jasonmorganson/symphony-k8s-worker@sha256:1a72392f97acd2acdfe1e520a39456b3d916d918b122f0011fff646887994ae8` |
| Credential-introduction source commit | `14e7ed280779717e6f87bd23ebd050f9b2ceed5f` |
| `docker/worker/entrypoint.sh` SHA-256 | `b0b39b39dc50ffa2647c56cc46158f01d895b60eec33241a9912f163001f5e05` |
| `k8s/base/worker-statefulset.yaml` SHA-256 | `ae396cddb24223ea4d541734643ef5ffe509b142544ba31cd5f02a485fa098d0` |
| `docker/worker/Dockerfile` SHA-256 | `cf1e2f36844777775d6d2519791c4d417e019dea10a91cdb904abdbc2371f460` |
| `config/sshd_config.d/worker.conf` SHA-256 | `75ae63c5e0e18945c20f85df15aa0ab575ea498c5154c3cf004e644e9be3dd42` |

A registry config query against the exact worker digest reported Linux AMD64
with `Entrypoint: ["/usr/local/bin/worker-entrypoint.sh"]`. The source Dockerfile
copies `docker/worker/entrypoint.sh` to that path. This receipt does not claim a
layer-byte attestation that the registry did not publish.

## Source inspection results

| Boundary | Evidence | Result |
|---|---|---|
| GitHub credential placement | StatefulSet injects `GITHUB_TOKEN`; entrypoint writes `.netrc` and stores the same token through `gh` under `/home/symphony/.config/gh` | Fail |
| Model credential placement | StatefulSet mounts `codex-chatgpt-auth`; entrypoint copies `auth.json` into `/home/symphony/.codex` | Fail |
| Linear credential placement | StatefulSet injects `LINEAR_API_KEY`; entrypoint requires and exports it in the worker process | Fail |
| Unix trust separation | GitHub and Codex files are owned by `symphony`; SSH agent commands use `symphony` | Fail |
| GitHub merge capability | `Contents: write`, required for authenticated code push, authorizes the merge endpoint | Fail under literal R3 |
| GitHub approval capability | `Pull requests: write`, required for PR creation/update, authorizes a review action that includes `APPROVE` | Fail under literal R3 |

The first four failures are established by immutable source inspection and the
recorded artifact entrypoint. A hostile live export was intentionally not run:
placing real reusable credentials in the known-readable boundary would add risk
without changing the fail-closed result.

The Linear failure is packaging-specific rather than engine-required. The
pinned engine already runs Linear polling and `linear_graphql` host-side and
unsets the declared tracker credential before remote Codex launch. Removing
`LINEAR_API_KEY` from the worker is therefore a valid partial correction, but it
does not make the existing worker image ready because its current entrypoint
requires that variable and the GitHub and Codex failures remain.

## Stop-condition mapping

The following canonical stop conditions have fired:

- The persistent SSH-worker topology cannot prevent repository-controlled code
  and the coding subprocess from reading reusable model, GitHub, or Linear
  credentials.
- A reusable customer credential must be exposed to an untrusted worker for the
  current entrypoint to become ready.
- The current GitHub permission surface does not satisfy the literal prohibition
  on granting lane workers approval or merge capability.

Therefore U2 cannot produce passing credential or forbidden-capability evidence,
U6 cannot qualify the pinned pair for hosted use, and U7 cannot claim lane
credential containment.

## What remains valid

- U1 provenance, monorepo, database, contract, Kustomize, and console-image
  results are unaffected.
- Symphony remains the sole scheduler and live run authority.
- Flux remains the sole lane Kubernetes resource writer.
- Product releases still consume immutable external image digests and contain no
  Symphony or worker source.
- The ExternalArtifact and desired-state projection architecture is independent
  of this failure.

## Resume gate

The canonical plan now selects a trusted lane boundary without revising literal
R3. U2 may resume only to prove that exact design: stock Codex app-server and
model authentication in the trusted orchestrator, an uncredentialed remote
exec-server worker, a lane-scoped Git broker, and a deployment-isolated PR-origin
fork in the source repository's GitHub fork network. The revised U2 must include
credential canaries, hostile repository export attempts, `/proc`,
file-descriptor, environment, file, log and crash
inspection, local-execution and direct-model-egress denial, broker credential
extraction, encoded and wrong-repository ref attempts, protected-ref update,
delete and non-fast-forward denial, lease expiry and revocation, pull-request
create/update parity, merge/review/deploy API denial, Linear scope enforcement,
and base-workflow no-deploy analysis. Any failure reactivates this stop; worker
secrets, a token-returning helper, local Codex fallback, or repository policy
alone are not fallback modes.
