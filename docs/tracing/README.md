# Tracing on dev-eks-us-east-1

Tracing shows the path a single request takes through your services, and how long each step
took. On `dev-eks-us-east-1`, apps send traces to an OpenTelemetry Collector, which forwards
them to Jaeger. You open the Jaeger UI to look at them.

Flux watches this repo and applies whatever is in it to the cluster. You don't run
`kubectl apply` yourself: to change the cluster, you change the files here and push.

## Terms used in this guide

- **Trace** - the full journey of one request. It is made of **spans**, one per step (an HTTP call, a database query), each with a start time and a duration.
- **OpenTelemetry (OTel)** - the open standard for producing traces, metrics and logs. Apps use an OTel SDK to create spans.
- **OTLP** - the protocol OTel uses to send data. Port `4317` is OTLP over gRPC, port `4318` is OTLP over HTTP.
- **Collector** - receives OTLP from apps, adds information to it, and forwards it to a backend. Apps only need to know the collector's address, never the backend's.
- **Jaeger** - stores traces and has a UI for searching and viewing them.

## How it works

```
your app ──OTLP──▶ opentelemetry-collector ──OTLP──▶ jaeger ──▶ Jaeger UI
          :4317 / :4318      (monitoring)            (monitoring)     :16686
```

1. Your app sends spans to the `opentelemetry-collector` Service in the `monitoring` namespace.
2. The collector adds the Kubernetes pod, namespace and node the spans came from, plus
   `k8s.cluster.name=dev-eks-us-east-1`, then batches them.
3. It sends them to Jaeger, and also prints a summary to its own log (the `debug` exporter),
   so you can confirm spans arrived without opening the UI.
4. Jaeger keeps them in memory and serves the UI on port `16686`.

The collector's own metrics (spans received, spans exported, export errors) go to the existing
Prometheus through a ServiceMonitor.

## The files

| File | What it does |
| --- | --- |
| `infrastructures/base/000-helm-repository/jaegertracing.yaml` | Where Flux downloads the Jaeger chart from. |
| `infrastructures/base/opentelemetry-collector/helmrelease.yaml` | Installs the collector. Switches off the chart's default receivers and its logs and metrics pipelines, so it only takes OTLP traces. Exports to `debug` only: which backend a cluster uses is decided in its overlay. |
| `infrastructures/dev/dev-eks-us-east-1/opentelemetry-collector/patch.yaml` | This cluster's changes: runs on the monitoring node pool, sets resources, turns on the ServiceMonitor, adds the cluster-name label, and sends traces to Jaeger. |
| `infrastructures/base/jaeger/helmrelease.yaml` | Installs Jaeger v2 with in-memory storage. |
| `infrastructures/dev/dev-eks-us-east-1/jaeger/patch.yaml` | This cluster's changes: runs on the monitoring node pool and sets resources. |

Each folder also has a `kustomization.yaml` that pulls in the base and applies the patch.
Flux's `infrastructure` Kustomization (`clusters/dev/dev-eks-us-east-1/infrastructures.yaml`)
deploys every folder under `infrastructures/dev/dev-eks-us-east-1/`.

## Before you start

You need:

- `kubectl` connected to `dev-eks-us-east-1`. Its API is private, so open the tunnel first:
  see `docs/access.md` in `terraform-infra-v2`. `kubectl --context dev-eks-us-east-1 get nodes` should work.
- the [`flux` CLI](https://fluxcd.io/flux/installation/#install-the-flux-cli)

## Send traces from your app

Point your app's OTel SDK at the collector. Most SDKs read these environment variables:

```
OTEL_SERVICE_NAME=my-app
OTEL_EXPORTER_OTLP_ENDPOINT=http://opentelemetry-collector.monitoring.svc.cluster.local:4318
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
```

For gRPC, use `opentelemetry-collector.monitoring.svc.cluster.local:4317` with
`OTEL_EXPORTER_OTLP_PROTOCOL=grpc`. `OTEL_SERVICE_NAME` is the name your app appears under in Jaeger.

## Send a test trace

You don't need an app to try it. `telemetrygen` is a small tool from the OTel project that
sends fake traces. This runs it once in the `default` namespace and deletes the pod when done:

```
kubectl --context dev-eks-us-east-1 run otel-demo -n default --rm -i --restart=Never \
  --image=ghcr.io/open-telemetry/opentelemetry-collector-contrib/telemetrygen:v0.162.0 -- \
  traces --otlp-endpoint=opentelemetry-collector.monitoring.svc.cluster.local:4317 \
  --otlp-insecure --traces=5 --child-spans=3 --service=otel-demo
```

That sends 5 traces, each with one parent span and 3 child spans.

## Check that it worked

The collector received the spans. Look for `Traces` lines from the `debug` exporter:

```
kubectl --context dev-eks-us-east-1 logs -n monitoring deploy/opentelemetry-collector --tail=20
```

Open the Jaeger UI. Leave this running in its own terminal:

```
kubectl --context dev-eks-us-east-1 port-forward -n monitoring svc/jaeger 16686:16686
```

Go to http://localhost:16686, pick `otel-demo` under **Service**, and click **Find Traces**.
Open a trace: under **Process** you should see `k8s.cluster.name`, `k8s.namespace.name` and
`k8s.pod.name`, added by the collector.

Both HelmReleases should show `Ready: True`:

```
flux --context dev-eks-us-east-1 get helmreleases -n flux-system | grep -E "jaeger|opentelemetry"
```

## Limits

- **Traces live in memory.** Jaeger keeps up to 20,000 and loses them all when its pod restarts,
  including when the spot node it runs on is reclaimed. Fine for dev and demos; anything that
  must keep traces needs persistent storage.
- **Traces only.** The collector's logs and metrics pipelines are switched off.
- **No sampling.** Every span sent is kept.
- **In-cluster only.** Both Services are `ClusterIP`; the UI is reached through `port-forward`.
  NetworkPolicies are not enforced on this cluster (the VPC CNI runs with
  `--enable-network-policy=false`), so a NetworkPolicy would not restrict access here.

## Changing the config

- **The collector chart merges your `config:` with its defaults.** To remove a default, set it
  to `null`, and do it in the base file. In a Kustomize patch, `null` means "delete this key",
  so Kustomize drops it, Helm never sees it, and the default quietly comes back.
- **A patch replaces lists, it doesn't add to them.** To add a processor or exporter in an
  overlay, write out the full list.
- **Merging is not deploying.** Flux checks Git every hour and re-applies infrastructure every
  20 hours. To deploy right after a merge:

```
flux --context dev-eks-us-east-1 reconcile kustomization infrastructure --with-source
```

## Troubleshooting

**Nothing under `otel-demo` in Jaeger, and the collector log has no `Traces` lines.** The test
pod couldn't reach the collector. Check it's running:
`kubectl --context dev-eks-us-east-1 get pods -n monitoring -l app.kubernetes.io/name=opentelemetry-collector`.

**The collector log has `Traces` lines but Jaeger shows nothing.** Look for export errors in the
collector log. `connection refused` means Jaeger is down:
`kubectl --context dev-eks-us-east-1 get pods -n monitoring -l app.kubernetes.io/name=jaeger`.
If Jaeger restarted, earlier traces are gone (see Limits); send the test trace again.

**A pod is stuck in `Pending`.** The monitoring node pool is tainted
`dedicated=monitoring:NoSchedule`. A pod needs both the `workload: monitoring` nodeSelector and
the matching toleration; one without the other stays Pending. See `docs/karpenter/README.md`.

**`kubectl` or `flux` says "connection refused" or times out.** The tunnel to the cluster isn't
open. See `docs/access.md` in `terraform-infra-v2`.
