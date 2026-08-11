# Symphony Platform

Symphony Platform packages the existing Symphony engine as one portable
product for hosted and self-hosted operation. This repository is currently the
implementation-planning authority; product code begins only after the selected
Effect platform validation gate passes.

## Documents

- [Implementation plan](docs/symphony-platform-implementation-plan.md) — the
  canonical product, architecture, sequencing, and acceptance contract.
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
