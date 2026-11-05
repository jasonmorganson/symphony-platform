# Foundation architecture

The foundation is intentionally split by authority rather than by deployment
fashion:

- `apps/console` is the U1 application and image. U17 later adds
  `apps/git-broker` as the only second launch application, with separate issuer
  and lane-broker entrypoints.
- The console's web Deployment is the only initial console runtime. U15 later
  adds its Workflow Deployment with separate database credentials and entrypoint.
- `packages/backend` contains server-only Effect services and repositories. It
  is not a second deployed service; production Workflow definitions arrive only
  with U15.
- `packages/contracts` contains the authoritative private desired-state Effect
  HttpApi and schemas plus deterministically generated OpenAPI 3.1. The generated
  document is not edited as an independent wire authority.
- `apps/console/db/schema.sql` is the relational authority. Atlas owns reviewed
  migrations; Effect SQL owns runtime access, not DDL.
- `deploy/kustomize/base` is the complete self-hosted product source. The
  hosted overlay adds operator bindings and the local overlay is disposable.
- Skaffold builds only the console in U1 and adds the Git-boundary image in U17.
  Existing Symphony orchestrator and worker images are consumed by digest from
  the lane package.

KRM Source is the external dependency integrated in U3. This repository
statically installs the signed release's `DesiredStateSource`, `ExternalArtifact`,
root Flux `Kustomization`, and product-release `OCIRepository`; it does not
vendor or implement the controller. Flux remains the only writer of lane
resources.
