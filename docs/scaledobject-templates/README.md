# ScaledObject templates

**Owner:** `@scaling` · **Issue:** gitops-flux#31 · **KEDA:** 2.20.2

Copyable starting points for autoscaling a workload with KEDA on `u25c-shared`,
plus the failure modes that cost time.

**How far each template has been proven** is stated in the table below and at the
top of each file, and the distinction is deliberate. `cron` has run end to end in
this cluster. The other four were accepted by KEDA's admission webhook, which
validates structure and scaler configuration but does not poll anything. Two of
them have never talked to the system they scale on. Every template here targets a Deployment
called `my-app` in your own namespace — change the name, the namespace and the
threshold, and nothing else.

Nothing in this directory is reconciled. It sits outside every Flux
`Kustomization` path on purpose: these are text to copy into *your* repository,
not objects to apply here. Business workloads live in `gitops-argocd` (ADR 0010,
ADR 0013), so that is where the copy lands.

The running example of all of this is `platform-scaling/scaling-reference`,
defined in `infrastructures/base/scaling-reference/`. It is a real Deployment on
a real ScaledObject, and it is where the measured numbers below come from.

## Which trigger

| Trigger | Use it for | Scale to zero | Status here |
|---|---|---|---|
| [`cron`](cron.yaml) | Known schedules: business hours, nightly batch | **Yes** | ✅ verified in-cluster |
| [`cpu`](cpu.yaml) | Request-driven services with no better signal | No | ✅ accepted by the webhook |
| [`memory`](memory.yaml) | Caches and buffers, rarely the right signal | No | ✅ accepted by the webhook |
| [`aws-sqs-queue`](aws-sqs-queue.yaml) | Queue workers — the reason KEDA is installed | **Yes** | ⚠️ webhook-valid, **never run against a real queue** |
| [`prometheus`](prometheus.yaml) | Anything with a metric but no native scaler | **Yes** | ⚠️ webhook-valid, **never run against a real query** |

**Start with the question "what does load actually look like for this workload?"**
If the honest answer is "requests arrive and CPU goes up", `cpu` is fine and you
do not need KEDA for it — see gitops-flux#32, which sets HPA defaults. KEDA earns
its place when the signal is *external* to the pod: a queue depth, a schedule, a
metric only Prometheus knows.

## The five things that will bite you

### 1. Do not commit a replica count

The single most common failure, and it does not look like a failure — it looks
like the autoscaler is fighting you, because it is.

KEDA drives replicas through an HPA. If your Deployment manifest also carries
`spec.replicas`, your GitOps controller reapplies that number on every reconcile
and the HPA immediately overrides it. The Deployment flaps forever.

- **Flux:** omit `spec.replicas` from the manifest entirely.
- **Argo CD:** omit it *and* add the ignore rule, because Argo reports the
  difference as `OutOfSync` and self-heal will act on it:

```yaml
spec:
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas
```

### 2. `kubectl get hpa` shows `MINPODS 1` even when you asked for zero

Not a bug and not your manifest being ignored. A Kubernetes HPA cannot target
zero replicas, so KEDA creates the HPA with `minReplicas: 1` and performs the
`1 -> 0` and `0 -> 1` transitions itself by scaling the Deployment directly. The
HPA only owns the range above 1.

Measured on the reference workload, outside its cron window:

```
NAME                          REFERENCE                      MINPODS   MAXPODS   REPLICAS
keda-hpa-scaling-reference    Deployment/scaling-reference    1         5         0
```

`MINPODS 1` and `REPLICAS 0` at the same time is the correct, healthy state.
Read `kubectl get scaledobject` for the real picture, not the HPA.

### 3. Scale-down is not immediate, and that is `cooldownPeriod`

`cooldownPeriod` (default **300s**) is how long KEDA waits after the trigger goes
inactive before scaling to zero. On top of that the HPA has its own scale-down
stabilisation window (default 300s) for the range above 1.

So after your queue drains or your cron window closes, expect **several minutes**
before replicas reach zero. Do not conclude the scale-down is broken until you
are past `cooldownPeriod`. If you need it faster, lower `cooldownPeriod`
deliberately and know that you are trading cost for churn.

### 4. `cpu` and `memory` cannot scale to zero

KEDA refuses `minReplicaCount: 0` on these two scalers, and the reason is not
arbitrary: both metrics are read *from the running pods*. At zero replicas there
is nothing to measure and no signal that would ever bring the workload back.

If you want scale-to-zero, the trigger has to be something observable from
outside the workload — a queue, a schedule, a Prometheus query.

### 5. `pollingInterval` and `cooldownPeriod` do nothing above `minReplicaCount: 1`

Both fields govern only the `0 <-> 1` transition, which is the part KEDA performs
itself. Above 1 replica the HPA is doing the scaling on its own 15-second cycle
and neither field is consulted.

KEDA's admission webhook says so when you apply one:

```
Warning: PollingInterval is configured but is not relevant. PollingInterval is
only relevant when minReplicaCount = 0 or idleReplicaCount = 0 [...]
```

Which means every `cpu` and `memory` ScaledObject that carries them is carrying
dead configuration - and someone will eventually tune those numbers trying to
change behaviour that is not controlled by them. The `cpu` and `memory`
templates here omit both on purpose.

### 6. A ResourceQuota that binds is invisible from the ScaledObject

If your namespace's `ResourceQuota` runs out mid-scale-up, **the ScaledObject and
the HPA both keep reporting that they want more replicas, and both look healthy.**
The ReplicaSet is what fails.

```bash
kubectl describe rs -n <ns> -l app.kubernetes.io/name=my-app | grep -i quota
kubectl get events -n <ns> --field-selector reason=FailedCreate
```

Size `maxReplicaCount` against your quota before you ship, not during an
incident. The reference workload's namespace sets `pods: "8"` against a
`maxReplicaCount` of 5 for exactly this reason.

## Where the pods land

Autoscaling a workload without saying where it runs is half a decision. Pods that
cannot be scheduled are just Pending, and the ScaledObject will still look fine.

See [`node-placement.md`](https://github.com/Ubuntu-25c-market-prep/ops-program/blob/main/docs/node-placement.md)
in `ops-program` for the pool contract. For autoscaled work specifically:

- **Bursty, retryable, scale-to-zero work belongs on `burst`** — spot-only, and
  nodes consolidate about a minute after the pods go away. That is what makes
  scale-to-zero actually save money rather than just reduce pod count.
- `burst` is **tainted**, so you need *both* the `nodeSelector` and the matching
  toleration. One without the other is the most common placement mistake.
- `burst` is **arm64 + amd64** and Karpenter buys whichever is cheaper, so your
  image must be multi-arch or pods die with `exec format error` on whichever
  architecture it lacks.
- Business workloads that are not retryable stay on `apps`, which is untainted
  and needs no selector at all.

## Verifying it works

```bash
kubectl get scaledobject -n <ns>                  # READY=True, ACTIVE=True in window
kubectl get hpa -n <ns>                           # MINPODS 1 is expected, see above
kubectl get pods -n <ns> -o wide                  # which node did they land on
kubectl logs -n keda -l app=keda-operator --tail=50 | grep -i <your-scaledobject>
kubectl describe scaledobject <name> -n <ns>      # trigger errors surface here
```

If `READY=False`, the message on the ScaledObject's conditions is specific and
worth reading before anything else — a `scaleTargetRef` naming a Deployment that
does not exist reports `error finding the scale target`, not a generic failure.
