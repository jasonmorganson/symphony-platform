# Symphony Platform

> **Status: deferred reference design.** This repository preserves the reviewed
> product, architecture, security, and implementation planning for a possible
> future Symphony platform. No implementation is currently scheduled, and the
> documents do not describe a deployed or supported service.

Symphony Platform is a proposed portable product around the existing Symphony
engine for hosted and self-hosted operation. This repository intentionally
contains documentation only. Symphony source and runtime-image construction
remain in their existing repositories.

## Documents

- [Documentation index](docs/README.md) — status, reading order, and authority
  of the complete planning corpus.
- [Implementation plan](docs/symphony-platform-implementation-plan.md) — the
  canonical deferred product, architecture, sequencing, and acceptance
  contract.
- [Effect platform supplement](docs/symphony-saas-console-effect-platform.md) —
  the selected console application, persistence, workflow, and private-contract
  implementation posture.
- [Historical console supplement](docs/symphony-platform-console-implementation.md)
  — superseded Kysely-era implementation history.
- [Historical architecture](docs/symphony-platform-architecture.md) —
  superseded product context retained where the implementation plan does not
  replace it.

## Authority Boundaries

- `jasonmorganson/symphony` owns Symphony source changes.
- `symphony-k8s` owns the existing orchestrator and worker images consumed by
  digest.
- Symphony remains the sole work scheduler and runtime-state authority.
- Flux remains the sole writer, pruner, inventory owner, drift detector, and
  health reporter for lane Kubernetes resources.
- This repository owns the console, contracts, product packaging, hosted
  integrations, validation, and release configuration described by the plan.

## Repository scope

There is no application, controller, Kubernetes package, database migration,
or release pipeline on `main`. Paths such as `apps/`, `packages/`, `deploy/`,
and `tests/` in the implementation plan are proposed future paths, not existing
software.

If the project is resumed, begin by refreshing all external dependency and
provider assumptions, rerunning the recorded conformance gates, and reviewing
the implementation plan before creating production code.
