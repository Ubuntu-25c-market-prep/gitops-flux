# gitops-flux

Everything Flux reconciles into the cluster: the platform add-ons themselves and
the Flux objects that deliver them. Business applications are Argo CD's job and
live in `gitops-argocd`.

**Owners:** `@flux` for `clusters/`, the owning workstream for every other path —
see `.github/CODEOWNERS` · **Waves:** 2–7

Absorbed `platform-addons`, `platform-observability` and `platform-security` per
[ADR 0010](https://github.com/Ubuntu-25c-market-prep/ops-program/blob/main/docs/adr/0010-one-repository-per-delivery-boundary.md).

## The boundary

```
Terraform (infra-aws)   →  provisions the cluster
Flux (this repo)        →  everything the platform runs inside it
Argo CD (gitops-argocd) →  everything the business runs inside it
```

Two GitOps controllers is a deliberate choice, not an accident of history.
Platform and product move at different cadences and have different reviewers.
Splitting them means an add-on upgrade cannot block an application release, and
an application rollback cannot touch the mesh.

**One owner per resource.** If Flux reconciles it, Terraform must not, and Argo
must not. A resource with two controllers is a fight, not redundancy.

## Layout

```
clusters/
└── platform/
    ├── flux-system/          bootstrap, do not hand-edit
    ├── sources/              HelmRepository, GitRepository
    └── infrastructure/
        ├── 00-core/          CNI, CSI, CoreDNS
        ├── 10-utils/         cert-manager, external-dns, sealed-secrets
        ├── 20-scaling/       Karpenter, KEDA
        ├── 30-mesh/          Istio, Kiali
        ├── 40-observability/ Prometheus, EFK, OTel, Kubecost
        └── 50-ops/           Velero, Rancher

addons/                       Helm values and overlays        → addons/README.md
observability/                Metrics, logs, traces, FinOps   → observability/README.md
security/                     Kyverno, Policy Reporter, mTLS  → security/README.md
```

Numeric prefixes under `infrastructure/` encode the **dependency order**, which
is real: cert-manager must exist before Istio can have a gateway certificate, and
Prometheus must exist before Kiali has anything to query. Each directory is a
`Kustomization` with `dependsOn` pointing at the one below it.

## Configuration and delivery are still separate

The merge did not collapse the two halves — it moved them into one repository.

```
addons/ · observability/ · security/   →  clusters/platform/  →  cluster
            values                          HelmRelease           running
```

Values are owned by the workstream that runs the component; the Flux objects that
reference them are owned by `@flux`. `CODEOWNERS` routes them to different
reviewers exactly as it did across repositories. What changed is that wiring a
new add-on is now **one pull request instead of two**, and a values change can
never be merged without the `HelmRelease` that consumes it.

## Adding an add-on

1. Create the component directory under `addons/`, `observability/` or
   `security/` with `values.yaml` and per-environment overlays.
2. Add a `CODEOWNERS` entry for the path in the same pull request.
3. Add the `HelmRelease` to the right numbered directory under
   `clusters/platform/infrastructure/`. Set `dependsOn` explicitly — do not rely
   on retry-until-it-works, because a dependency satisfied only by luck will fail
   on a cold cluster rebuild.
4. Set `timeout` and `retries`. A release that hangs forever hides the failure.
5. State the monthly cost delta.

## Secrets

Sealed-secrets only. A plaintext `Secret` in this repository is a credential in a
**public** git history — push protection blocks the obvious cases, but sealed
secrets are the control.

## Reconciliation

Flux polls; it does not need a webhook to be correct. To force a reconcile:

```bash
kubectl annotate --overwrite kustomization/<name> -n flux-system \
  reconcile.fluxcd.io/requestedAt="$(date +%s)"
```

If a `Kustomization` is not becoming Ready, read `dependsOn` first. Most stuck
reconciles are a dependency that never became Ready, not the thing you are
looking at.

## Standards

[CONVENTIONS](https://github.com/Ubuntu-25c-market-prep/ops-program/blob/main/CONVENTIONS.md) ·
[Engineering Handbook](https://github.com/Ubuntu-25c-market-prep/ops-program/blob/main/docs/engineering-handbook.md) ·
[Terraform State Strategy — where Terraform stops](https://github.com/Ubuntu-25c-market-prep/ops-program/blob/main/docs/terraform-state-strategy.md)
