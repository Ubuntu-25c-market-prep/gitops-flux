# observability

Metrics, logs, traces and cost visibility. Delivered by Flux from
`clusters/platform/infrastructure/40-observability/`.

**Wave:** 5 · Absorbed from the former `platform-observability` repository (ADR 0010).

## Ownership

| Path | Owner | Stack |
|---|---|---|
| `monitoring/` | `@monitoring` | Prometheus, Grafana, alert rules, SLOs |
| `logging/` | `@logging` | Elasticsearch, Fluent Bit, Kibana |
| `tracing/` | `@tracing` | OpenTelemetry Collector, Jaeger |
| `finops/` | `@finops` | Kubecost, showback dashboards |

## The thing that will bite you

**Retention is the cost.** Not ingest — retention. Prometheus, Loki and
Elasticsearch on EBS grow without bound unless tiering is designed in before the
first byte lands. Elasticsearch alone is the heaviest single component in the
platform, roughly $200/month of node capacity.

Every component here ships with, in the same pull request that introduces it:

- A retention policy, stated in days
- A tiering path to S3 where the backend supports it
- An estimate of steady-state volume and its monthly cost

A dashboard with no retention policy is not done.

## Sampling

Trace ingest untuned is the second cost trap. Tail sampling is configured in
`tracing/` before any service is onboarded, not after the bill arrives.

## Self-hosted, deliberately

Prometheus and Grafana are self-hosted rather than AMP/AMG — roughly $200/month
cheaper at this scale. The trade is that upgrades, storage and availability are
ours. `@monitoring` owns that.

## Interfaces this tree publishes

Other workstreams depend on these; changing one is a pull request against the
documentation first, reviewed by consumers.

| Interface | Owner |
|---|---|
| Metric naming and required labels | `@monitoring` |
| Alert routing per workstream | `@monitoring` |
| SLO definition template | `@monitoring` |
| Instrumentation contract for service owners | `@tracing` |
| Log schema and required structured fields | `@logging` |
| Cost allocation tags for showback | `@finops` |

Showback depends on the tagging standard being enforced — `Org`, `Env`,
`Workstream`, `ManagedBy`, `Repo` on AWS resources, and
`u25c.io/workstream` / `u25c.io/owner` on Kubernetes workloads. Kyverno enforces
the Kubernetes half, from `../security/kyverno/`.
