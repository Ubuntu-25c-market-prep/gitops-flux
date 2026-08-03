# addons

Cluster add-ons: Helm values and Kustomize overlays. Delivered to the cluster by
Flux via `clusters/platform/`, never applied from here.

**Waves:** 2–6 · Absorbed from the former `platform-addons` repository (ADR 0010).

## Ownership

Six workstreams share this tree without merge contention because none of them
touch the same directories. `.github/CODEOWNERS` enforces it.

| Path | Owner | Contains |
|---|---|---|
| `core/` | `@infra` | vpc-cni, ebs-csi, coredns, kube-proxy, metrics-server |
| `scaling/` | `@scaling` | Karpenter NodePools, KEDA, HPA defaults |
| `utils/` | `@utils` | cert-manager, external-dns, sealed-secrets |
| `velero/` | `@velero` | Backup schedules, restore configuration |
| `rancher/` | `@rancher` | Rancher server, downstream cluster registration |
| `istio/` | `@istio` | istiod, gateways, Kiali, mesh configuration |

Need something in another workstream's directory? Open an issue against them.
Editing it and requesting review inverts ownership.

## How an add-on reaches the cluster

```
addons/            →  clusters/platform/  →  cluster
   values                HelmRelease          running
```

This tree holds **desired configuration**. `clusters/platform/` holds the Flux
objects that reference it. Nothing here applies itself, and there is no
`kubectl apply` in any workflow — if you find yourself reaching for one, the
add-on is not wired into Flux yet.

The separation survived the repository merge on purpose: values and delivery
objects still have different reviewers and different change cadence. What
changed is that they are now one pull request instead of two across repos.

## Adding an add-on

1. Create `<component>/` with `values.yaml` and per-environment overlays.
2. Add a `CODEOWNERS` entry for the path in the same pull request.
3. Wire the `HelmRelease` into the right numbered directory under
   `clusters/platform/infrastructure/`, with `dependsOn` set explicitly.
4. State the monthly cost delta. Every add-on consumes node capacity; several
   here consume a lot.

## Layout convention

```
<component>/
├── values.yaml            base, environment-agnostic
├── values-aws.yaml        AWS-only assumptions live HERE and nowhere else
├── values-dev.yaml
├── values-prod.yaml
└── README.md              what it does, what it depends on, what it costs
```

The `values-aws.yaml` split is not cosmetic. IRSA annotations, ALB annotations
and EBS storage classes must stay isolated so an add-on can move to the home k3s
cluster without rewriting its base configuration.

## Cost discipline

The add-on set here is the single largest line in the platform bill — larger
than the workloads it supports. Before adding anything:

- What does it request in CPU and memory, times replicas, times AZs?
- Does it need persistent volumes, and with what retention?
- Does it add cross-AZ traffic? Istio will, unless topology-aware routing is on.
