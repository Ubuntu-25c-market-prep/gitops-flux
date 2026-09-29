# Node pools on dev-eks-us-east-1

[Karpenter](https://karpenter.sh) adds a node when a pod has nowhere to run, and removes
the node when it is no longer needed. This guide shows how to put a workload on the right
pool.

Nothing moves by itself. A workload changes pool only when its owner adds the lines below.

## The pools

| Pool | For | Buys | Label | Taint |
|---|---|---|---|---|
| `dev-np-base-spot-us-east-1` | business apps, anything stateless | spot only | `capacity: spot` | none |
| `dev-np-logging-spot-od-us-east-1` | logging | spot, then on-demand | `workload: logging` | `dedicated=logging:NoSchedule` |
| `dev-np-istio-spot-od-us-east-1` | mesh | spot, then on-demand | `workload: istio` | `dedicated=istio:NoSchedule` |
| `dev-np-monitoring-spot-od-us-east-1` | monitoring | spot, then on-demand | `workload: monitoring` | `dedicated=monitoring:NoSchedule` |

The system node group is not a pool. It is managed by EKS, runs on-demand, and holds what
the cluster needs to start: Flux, CoreDNS, Karpenter, Sealed Secrets.

Nodes are amd64, with 2 vCPU and 4 or 8 GiB each. A pool has no nodes until a pod asks
for it, and usually runs one.

Each pool has a ceiling, so a faulty release cannot buy nodes without end:

| Pool | Ceiling |
|---|---|
| logging, istio, monitoring | 2 nodes each |
| base | 4 nodes |

Pods beyond the ceiling stay `Pending`. Raise it with a pull request on the pool's file.

## Where a pod lands

| The pod has | It runs on |
|---|---|
| nothing | a system node if one has room, otherwise a base spot node |
| a node selector and the matching toleration | that pool |
| only the node selector | nowhere: it stays `Pending` |
| only the toleration | anywhere it fits, the pool included |

The namespace plays no part. A pod in `monitoring` without the lines below is treated like
any other pod.

## Placing a workload

Add both parts to the pod spec. Replace `logging` with `istio` or `monitoring`.

```yaml
nodeSelector:
  workload: logging
tolerations:
  - key: dedicated
    value: logging
    effect: NoSchedule
```

For the base pool there is no taint. Add the selector only if the workload must never run
on a system node:

```yaml
nodeSelector:
  capacity: spot
```

Where these lines go depends on the chart:

| Component | Pool | Key in the HelmRelease values, or field in the resource |
|---|---|---|
| Elasticsearch, Kibana | logging | `spec.nodeSets[].podTemplate.spec` and `spec.podTemplate.spec` |
| ECK operator | logging | `nodeSelector`, `tolerations` |
| Prometheus | monitoring | `prometheus.prometheusSpec.nodeSelector`, `.tolerations` |
| Alertmanager | monitoring | `alertmanager.alertmanagerSpec.nodeSelector`, `.tolerations` |
| Grafana | monitoring | `grafana.nodeSelector`, `grafana.tolerations` |
| Prometheus operator | monitoring | `prometheusOperator.nodeSelector`, `.tolerations` |
| kube-state-metrics | monitoring | `kube-state-metrics.nodeSelector`, `.tolerations` |
| OpenTelemetry collector | monitoring | `nodeSelector`, `tolerations` |
| OpenCost | monitoring | `opencost.nodeSelector`, `opencost.tolerations` |
| istiod | istio | `nodeSelector`, `tolerations` |
| Kiali | istio | `deployment.node_selector`, `deployment.tolerations` |
| Your own Deployment | base | `spec.template.spec` |

For example, the OpenTelemetry collector's overlay `patch.yaml`:

```yaml
spec:
  values:
    nodeSelector:
      workload: monitoring
    tolerations:
      - key: dedicated
        value: monitoring
        effect: NoSchedule
```

Node agents (DaemonSets) need nothing. They already run on every node.

## Check where it landed

```
kubectl get pods -n <namespace> -o wide
kubectl get nodes -L karpenter.sh/nodepool,karpenter.sh/capacity-type,workload,capacity
```

A pod that stays `Pending` says why:

```
kubectl describe pod <pod> -n <namespace>
```

## Running on spot

AWS can take a spot node back with two minutes' notice. Karpenter then moves the pods to
another node. Your pod restarts.

- Set CPU and memory requests. Karpenter sizes nodes from them; a pod without requests
  counts as zero.
- Handle `SIGTERM` and finish within the pod's grace period.
- Do not keep data on the node's disk. Volumes are fine: they follow the pod within the
  same zone.
- To stay up during a move, run two replicas. With two or more, add a
  PodDisruptionBudget with `maxUnavailable: 1`.
- Never use `minAvailable: 1` with one replica. It blocks every move of that node.

logging and monitoring nodes are removed only when empty. base and istio nodes are also
replaced when a cheaper fit exists.

## Looking at a node

```
kubectl debug node/<node> -it --image=busybox     # through the API
ssh -J dev-bastion ec2-user@<node-private-ip>     # through the bastion
```

Laptop setup for both is in `docs/access.md` of `terraform-infra-v2`. Session Manager
works on these nodes too.

To keep a node while you look at it:

```
kubectl annotate node <node> karpenter.sh/do-not-disrupt=true
kubectl annotate node <node> karpenter.sh/do-not-disrupt-        # when done
```

This stops consolidation. It does not stop a spot interruption or the 30-day replacement.

## Changing a pool

Pools live in `clusters/dev/dev-eks-us-east-1/karpenter-nodepool/`, one file each. Change
them with a pull request. Flux puts back a manual edit within ten minutes.

| File | What it holds |
|---|---|
| `clusters/dev/dev-eks-us-east-1/karpenter.yaml`, `karpenter/` | the controller |
| `clusters/dev/dev-eks-us-east-1/karpenter-nodepool.yaml`, `karpenter-nodepool/` | the node class and the pools |

The AWS side (roles, queue, event rules) is in `terraform-infra-v2`; see its `eks` README.
