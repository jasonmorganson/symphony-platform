# U2 stock-Codex remote-environment seam: fail-closed validation receipt

Date: 2026-08-12
Result: **STOPPED — stock seam present; Symphony integration, bootstrap workflow source, and containment proof absent**

## Immutable inputs

- npm package: `@openai/codex@0.144.1`
- npm tarball integrity: `sha512-Xir1zqPfpenhdoAoshN53uonzbBXj18COyzRkFlVZpSNyEl5XtkuYu9oddELePFN7K/0sXUcSO34Ad5IeCXPbw==`
- npm tarball shasum: `b9861c8dbb5d81b02bbb4dd7abbfb38400bf096a`
- OpenAI Codex tag commit: `44918ea10c0f99151c6710411b4322c2f5c96bea`
- audited pinned `jasonmorganson/symphony` commit: `bd77c48d5800e9a849eeb2bf2aa29cf06abb121e`
- Node.js used by the fixture: `v26.7.0`

No real provider or model credential was used. The proposed runtime exercise used only a local mock
provider, but it was not accepted as credential-containment proof because the app-server and
exec-server would share this host's user, process namespace, and filesystem.

## Commands

```sh
npm view @openai/codex@0.144.1 dist --json
npm install --ignore-scripts --no-save @openai/codex@0.144.1
git clone --filter=blob:none --no-checkout https://github.com/openai/codex.git /private/tmp/openai-codex-u2
git -C /private/tmp/openai-codex-u2 checkout 44918ea10c0f99151c6710411b4322c2f5c96bea
/private/tmp/symphony-u2-codex-0.144.1/node_modules/.bin/codex --version
/private/tmp/symphony-u2-codex-0.144.1/node_modules/.bin/codex app-server generate-json-schema \
  --experimental --out /private/tmp/symphony-u2-codex-schema-0.144.1
node tests/conformance/model-auth/codex-remote-environment.mjs \
  --codex-bin /private/tmp/symphony-u2-codex-0.144.1/node_modules/.bin/codex \
  --codex-source /private/tmp/openai-codex-u2 \
  --symphony-source /Volumes/Home/jasonmorganson/.config/codex/worktrees/27c7/symphony
```

The final command exits `2` by design and emits a JSON blocker receipt.

## What the exact stock seam proves

The exact generated app-server schema includes experimental `environment/add` and
`environment/info` operations and explicit environment selection on `thread/start` and
`turn/start`. At the pinned source commit:

- `CODEX_EXEC_SERVER_URL=none` leaves no default or local execution environment.
- `environment/add` registers a named remote exec-server without making it the default.
- an unknown thread or turn environment is rejected as an invalid request.
- the stock suite contains runtime coverage for `exec_command` and `apply_patch` routing to the
  selected remote environment and asserts that the local marker is not returned.

This is sufficient to establish that a small Symphony integration could target a trusted local
app-server and an uncredentialed remote exec-server without changing Codex. It is not sufficient to
pass the product's credential-containment gate.

## Exact blocker

The audited Symphony client still opens `codex app-server` through `SSH.start_port` on the worker.
Its `thread/start` and `turn/start` requests do not call `environment/add` and do not include an
explicit `environments` selection. Therefore stock Codex's trusted-app-server/uncredentialed-worker
seam is not used by Symphony today.

The same pinned revision resolves `WORKFLOW.md` from a local path and loads it with `File.read/1`.
It has no broker-bound repository/revision/path workflow source that can be validated before issue
polling. U2 therefore cannot meet the selected bootstrap-safe workflow contract without a small
generic fork seam even if the Codex routing seam is added.

Additionally, a same-host fixture cannot prove that worker code is unable to read trusted process
metadata or trusted filesystem state. Nested macOS sandbox execution is unavailable in this test
environment (`Operation not permitted`), and a same-user mock would overstate the production trust
boundary. No claim is made that direct model network egress is denied; that remains a deployment
network-policy proof.

## Required proof before U2 can pass

1. Add the smallest generic fork change that starts Codex app-server in the trusted orchestrator,
   registers exactly one uncredentialed worker exec-server, and explicitly selects it for every
   thread and turn.
2. Add a generic workflow-source seam that validates the broker-bound repository, revision, and
   path before Symphony begins polling, without moving scheduling authority out of Symphony.
3. Start app-server with local/default execution disabled and prove wrong, empty, and omitted
   selections fail closed.
4. In the real two-workload topology, seed synthetic auth canaries and prove they are absent from
   worker environment, filesystem, process metadata, command output, prompts, tool results, logs,
   traces, and crash artifacts.
5. Prove worker direct model access fails under the lane egress policy while the trusted app-server
   can authenticate through stock Codex.

Until all five checks pass, R4's stock-Codex credential mode remains disabled and implementation
must return to planning rather than falling back to worker-local authentication.
