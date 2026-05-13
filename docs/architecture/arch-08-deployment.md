# arch-08: Deployment

Kirimana deploys into the customer's own cloud, not a vendor-hosted SaaS. The runtime is a small set of stateless services plus a Postgres for catalog and audit; everything else runs against the customer's existing data platform. This document describes the cloud-neutral Helm chart, multi-environment CI/CD, project-and-target isolation, the forward-only promotion model, and the supported runtime topologies for v0.9 preview and beyond.

## Cloud-neutral Helm chart

The deployment artefact is a single Helm chart with thin per-cloud overlays. The base chart targets stock Kubernetes and depends on three widely-available primitives: cert-manager for TLS, External Secrets Operator for credential resolution, and an OIDC-issuing identity provider for workload identity. The per-cloud overlays — azure, aws, gcp, onprem — map those primitives to the cloud-native equivalents (Azure Workload Identity, AWS IRSA, GCP Workload Identity Federation).

There is one chart; there is no Azure-specific fork. A customer who installs the azure overlay gets the same services and the same configuration shape as a customer who installs the aws overlay. The differences are confined to the auth and secret-resolution paths.

Components in the chart:

- **dca-core API** (FastAPI) — the library functions exposed over HTTP for the web app and MCP server
- **MCP server** — separate process, same library underneath (arch-04)
- **catalog Postgres** — the catalog backend; runs in the customer's subscription/account, never in a Kirimana-managed plane
- **AI Gateway** — stateless, scales horizontally; rate-limit state in Postgres for multi-replica correctness
- **web app** — Next.js front end and Python BFF

CLI deployments (no web app, no Postgres) are also supported for teams who want only the contract toolchain in CI.

## Project / target isolation

A Kirimana **project** is a governance boundary (arch-05). A **target** is a physical deployment of a project against a specific platform environment (e.g. the `prod` target on a specific Databricks workspace, or the `test` target on a Trino cluster). The mapping is one project to many targets, one target per environment per platform.

Each project owns one catalog namespace, marked at first apply by a `_dca_meta._dca_project_manifest` row in the target. The manifest prevents accidental cross-project writes: a second project cannot apply against a target that already carries another project's manifest without explicit override. This is the lowest-level safety net under the higher-level RBAC.

RBAC roles (viewer, author, approver, platform-admin) are scoped by environment and domain. An approver in `dev/sales` is not an approver in `prod/sales` or in `dev/finance` unless explicitly granted.

## Forward-only promotion

Kirimana promotes by moving environment cursors forward along the main branch. There is one Kirimana installation per customer, one main branch, and three cursors (dev, test, prod) that advance independently. Each cursor records the commit SHA it points at and the release manifest that produced the active state.

Promotion is forward-only by strict ancestry: prod's cursor must be an ancestor of test's cursor, which must be an ancestor of dev's. A "rollback" is a forward commit that reverts the change, not a cursor regression. Re-deriving a prior state through history is always possible because plans are deterministic (arch-02) and the release manifest captures the inputs.

The release manifest is a small YAML file emitted by `kiri release` listing the commit SHA, the contracts in scope, the adapter conformance versions, and the actor who triggered the release. It is the single document an audit asks for to reconstruct what shipped.

## Runtime topologies in v0.9

Three production-targeted topologies are supported in v0.9 preview:

- **Helm on Azure (today)** — the default Starter and Databricks-Edition deployment. AKS, External Secrets pointed at Azure Key Vault, Postgres Flexible Server, Workload Identity. This is the topology with the most operational mileage.
- **Helm on AWS / GCP / on-prem** — beta. Same chart, different overlays. Production-quality code path; thinner field experience than Azure.
- **Databricks App (GA target)** — Kirimana packaged as a Databricks App, deployable into a customer's Databricks workspace without a separate Kubernetes cluster. Beta in v0.9; targeted for GA. Same library underneath; the App is a packaging concern, not a code fork.

A direct AWS-native managed-service option (not Helm) is on the longer-term roadmap, after the Microsoft Fabric integration matures. AWS Databricks parity is on the roadmap; the OSS website (`kirimana.io`) carries the current target window. Until then, AWS customers run the Helm chart.

CI/CD across environments is conventional: GitHub Actions or equivalent runs `kiri validate` on PR, `kiri plan` on merge to main, and `kiri apply` against the dev target. Promotion to test and prod is a manual trigger that runs `kiri apply` against the next cursor.

## Related ADRs

010, 028, 029, 030, 031, 032
