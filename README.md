# gitops-flux

Flux desired state for **platform** add-ons. Business applications are Argo CD's
job and live in `gitops-argocd`.

**Owner:** `@flux` · **Wave:** 3

## The boundary

```
Terraform (infra-aws)  →  provisions the cluster
Flux (this repo)       →  everything the platform runs inside it
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
```

Numeric prefixes encode the **dependency order**, which is real: cert-manager
must exist before Istio can have a gateway certificate, and Prometheus must exist
before Kiali has anything to query. Each directory is a `Kustomization` with
`dependsOn` pointing at the one below it.

## Adding an add-on

1. The configuration itself goes in `platform-addons`, owned by its workstream.
2. Here, add the `HelmRelease` and place it in the right numbered directory.
3. Set `dependsOn` explicitly. Do not rely on retry-until-it-works — a
   dependency that is only satisfied by luck will fail on a cold cluster rebuild.
4. Set `timeout` and `retries`. A release that hangs forever hides the failure.

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
[Terraform State Strategy — where Terraform stops](https://github.com/Ubuntu-25c-market-prep/ops-program/blob/main/docs/terraform-state-strategy.md)
