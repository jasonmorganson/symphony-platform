# Symphony Platform documentation

## Status

This directory is a deferred reference design. It captures the product
decisions, architecture, security boundaries, implementation sequence, and
historical validation evidence developed during planning. It is not an active
roadmap commitment and does not prove that a production platform exists.

The documents are preserved so future work can resume from reviewed decisions
instead of reconstructing them from chat history. Before implementation, treat
all dependency versions, provider capabilities, commercial assumptions, and
security seams as potentially stale and revalidate them.

## Reading order and authority

1. [Implementation plan](symphony-platform-implementation-plan.md) is the
   canonical deferred product and implementation contract.
2. [Effect platform supplement](symphony-saas-console-effect-platform.md)
   defines the selected console implementation posture where it does not
   conflict with the implementation plan.
3. [Foundation architecture](architecture/foundation.md), the
   [ADRs](decisions/), and the [threat model](security/threat-model.md) record
   narrower decisions and security assumptions.
4. [Validation receipts](validation/) are historical evidence from disposable
   planning spikes. They are not current qualification results and must be
   rerun before relying on them.
5. [Historical architecture](symphony-platform-architecture.md) and the
   [historical console supplement](symphony-platform-console-implementation.md)
   are superseded context only.

When documents conflict, the implementation plan wins, followed by an accepted
ADR governing the narrower subject. Historical and superseded documents never
override current planning authority.

## Resume checklist

Before restarting implementation:

1. Confirm the current OpenAI Symphony specification and fork delta.
2. Requalify the exact Symphony, Codex, Flux, KRM Source, Kubernetes, Effect,
   Next.js, Better Auth, Atlas, PostgreSQL, and provider versions.
3. Re-run the credential-containment, remote-execution, Git-broker,
   desired-state, rollback, self-hosted, disconnected, and release-provenance
   gates.
4. Reconfirm the launch customer, pricing, provider terms, hosted operating
   target, and data-retention commitments.
5. Update the plan dates and status before merging implementation code.

Until those steps are complete, the repository should remain documentation-only.
