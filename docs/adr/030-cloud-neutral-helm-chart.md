# ADR 030 — Cloud-neutral Helm chart with per-cloud profiles

- **Status:** Accepted
- **Date:** 2026-05-13
- **Re-curated from:** original ADR 0040 (archived)

## Context

Azure Kubernetes Service (AKS) is Kirimana's first production target. It was the right call for the launch phase — most early design partners run there. It also creates a real risk: the more we build, the more AKS-specific patterns can creep in unnoticed and turn into a corner we cannot escape.

The roadmap commits Kirimana to additional deployment targets: AWS (EKS) follows after Microsoft Fabric edition GA, on-premises Kubernetes is a near-term reality for healthcare and public-sector customers, and the OSS Edition must run on a developer's laptop with `k3d` or `kind`.

Without an explicit cloud-neutrality non-negotiable, three concrete dangers materialise:

1. **Hard dependencies on Azure-specific Custom Resources** — pod-managed identities, AzureWorkloadIdentity, AzureApplicationGateway, the Azure Key Vault CSI driver. Each solves a real AKS problem but has no equivalent on EKS or GKE.
2. **OSS contributors locked out** — a contributor cloning the repo cannot run the product if the local chart needs an Azure tenant, a Container Registry image, and managed-identity bindings.
3. **Marketplace listing complexity** — Databricks Apps run in their own runtime; the chart must produce something deployable there without Azure assumptions.

## Decision

One Helm chart, layered profiles. The base chart is cloud-neutral; clouds and edition variants are thin overlays.

```
infrastructure/helm/kirimana/
├── Chart.yaml
├── values.yaml                     # cloud-neutral defaults
├── values.profiles/
│   ├── azure-aks.yaml              # AzureWorkloadIdentity, Key Vault CSI
│   ├── aws-eks.yaml                # IRSA, Secrets Manager CSI
│   ├── on-prem.yaml                # cert-manager, no cloud-managed identity
│   └── local-dev.yaml              # k3d/kind, in-cluster Postgres, no TLS
└── templates/                      # cloud-neutral primitives only
```

### Three non-negotiables for the base chart

1. **No cloud-provider CRDs in templates.** Cloud-specific resources (managed identities, ingress controllers) live in overlays as additional template files merged at install time, never embedded in `templates/`.
2. **All secrets resolve through `VaultProvider`.** The chart never assumes a specific secret backend. Helm values declare which provider to use; the running service resolves at startup.
3. **The local-dev profile must work offline.** No image pulls from cloud registries, no managed services, no DNS dependencies. A contributor can run the whole stack on a laptop without an account anywhere.

## Consequences

- **Positive:** Adding AWS or on-prem support becomes an overlay, not a rewrite. The OSS contributor path is genuinely open. Future Databricks Apps packaging reuses the same chart with a thin profile.
- **Negative:** The discipline must be enforced — CI checks that templates don't import cloud-specific CRDs. The overlay model adds one indirection for operators learning the chart.
- **Neutral:** Per-cloud documentation grows in parallel; that's natural.

## Related

- See [arch-08-deployment.md](../architecture/arch-08-deployment.md)
- Related ADRs: 011 (platform-agnostic core), 029 (rate-limit store), 032 (CI/CD)
