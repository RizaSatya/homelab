# OpenTelemetry Collector Gateway Design

## Context

This design bootstraps a shared OpenTelemetry Collector for the homelab platform.

The current observability stack already includes Prometheus and Grafana managed through Argo CD and Helm. `web-riza` is deployed as a handwritten application manifest in its own namespace and does not currently expose metrics or emit OpenTelemetry telemetry.

The goal of this design is to prepare the platform layer first so current and future apps can push OpenTelemetry metrics to a shared in-cluster gateway. Prometheus will continue to be the metrics backend, but it will scrape the Collector rather than scraping application pods directly.

## Goals

- deploy a shared OpenTelemetry Collector in the `observability` namespace
- manage the Collector through Argo CD and Helm, matching the current platform pattern
- expose a stable OTLP ingestion endpoint for apps
- expose a stable Prometheus scrape endpoint for Prometheus
- keep the first pipeline limited to metrics only
- make the design reusable for future apps beyond `web-riza`

## Non-Goals

- instrument `web-riza` in this phase
- add traces or logs in this phase
- introduce the OpenTelemetry Operator
- build a multi-tier agent plus gateway architecture
- optimize for high availability before the first signal path is proven

## Recommended Approach

Install the official `opentelemetry-collector` Helm chart as a shared gateway-style Collector in the `observability` namespace using Argo CD.

The Collector will run in `deployment` mode with one replica initially. Applications will push metrics over OTLP/HTTP to the Collector. The Collector will batch the incoming metrics and re-expose them on a Prometheus endpoint that the existing Prometheus instance scrapes through a `ServiceMonitor`.

This approach is recommended because it preserves the existing GitOps pattern, keeps platform and application concerns separate, and introduces a real OpenTelemetry architecture without adding the complexity of the OpenTelemetry Operator or multiple collector tiers.

## Architecture

The end-to-end path is:

`application -> OTLP metrics export -> OpenTelemetry Collector -> Prometheus scrape -> Grafana`

In the first intended use case:

`web-riza -> OTLP/HTTP -> shared Collector in observability -> Prometheus -> Grafana`

Component responsibilities:

- `web-riza` and future apps create metrics with the OpenTelemetry metrics SDK and export them over OTLP
- the shared Collector receives, batches, and translates the telemetry into a Prometheus-scrapable endpoint
- Prometheus scrapes that endpoint on its normal interval
- Grafana queries Prometheus as the user-facing visualization layer

## Platform Components

### 1. Argo CD Application

Add a new Argo CD `Application` manifest for the Collector under `argocd/`.

Responsibilities:

- reference the official OpenTelemetry Helm repository
- install the `opentelemetry-collector` chart
- target the `observability` namespace
- load values from the Git repo in the same style as the Prometheus stack application
- enable automated sync and namespace creation

Expected file:

- `argocd/opentelemetry-collector-application.yaml`

### 2. Collector Helm Values

Add a dedicated values file for the Collector under `platform/observability/opentelemetry-collector/`.

Responsibilities:

- set `mode: deployment`
- configure one replica initially
- define receiver, processor, and exporter pipeline
- expose only the ports needed for the first metrics-only rollout
- set explicit CPU and memory requests and limits sized for the current worker node

Expected file:

- `platform/observability/opentelemetry-collector/values.yaml`

### 3. ServiceMonitor

Add a handwritten `ServiceMonitor` manifest owned in Git rather than relying on chart-generated discovery behavior.

Responsibilities:

- select the Collector service in the `observability` namespace
- scrape the Collector Prometheus endpoint
- make the scrape contract explicit and reviewable in the repo

Expected file:

- `platform/observability/opentelemetry-collector-manifests/opentelemetry-collector-servicemonitor.yaml`

## Collector Configuration

The initial Collector configuration should be intentionally minimal.

### Receivers

- `otlp`

Protocol support for phase one:

- `http` enabled on port `4318`

OTLP/gRPC on `4317` is intentionally deferred until an application requires it.

### Processors

- `memory_limiter`
- `batch`

The `memory_limiter` processor should keep the Collector within a predictable memory envelope on the current 8 GiB / 2 core worker node. The `batch` processor improves efficiency and keeps the first pipeline simple to reason about.

### Exporters

- `prometheus`

The exporter should expose a scrape endpoint on port `8889`.

### Service Pipelines

One metrics pipeline only:

- `metrics: receivers [otlp] -> processors [batch] -> exporters [prometheus]`

No traces or logs pipelines should be configured in this phase.

## Network And Service Design

The Collector service should expose two distinct purposes:

- OTLP ingest for applications on port `4318`
- Prometheus scrape endpoint on port `8889`

This creates two clear contracts:

- application contract: send OTLP metrics to the Collector service
- platform contract: Prometheus scrapes the Collector service

The intended in-cluster OTLP endpoint for applications will be:

- `http://opentelemetry-collector.observability.svc.cluster.local:4318`

The Prometheus scrape target will be the Collector service metrics port selected through the `ServiceMonitor`.

## GitOps Layout

The new platform files should be organized as:

- `argocd/opentelemetry-collector-application.yaml`
- `platform/observability/opentelemetry-collector/values.yaml`
- `platform/observability/opentelemetry-collector-manifests/opentelemetry-collector-servicemonitor.yaml`

This layout follows the existing repository conventions:

- `argocd/` contains top-level application declarations
- `platform/observability/` contains platform-owned observability configuration

## Resource And Availability Strategy

Initial deployment characteristics:

- `mode: deployment`
- `replicaCount: 1`
- `resources.requests.cpu: 50m`
- `resources.requests.memory: 128Mi`
- `resources.limits.cpu: 200m`
- `resources.limits.memory: 256Mi`
- `memory_limiter.limit_mib: 192`
- `memory_limiter.spike_limit_mib: 64`
- no persistence required for the Collector

One replica is sufficient for the first rollout because the immediate goal is to prove the telemetry path, not to optimize for availability. If the Collector restarts, metrics in transit may be temporarily lost, which is acceptable for this learning phase.

Later improvements can include:

- multiple replicas
- readiness and liveness tuning
- PodDisruptionBudget
- broader receiver support

## Error Handling And Debugging Model

The expected debugging order is:

1. confirm the Argo CD application is healthy and synced
2. confirm the Collector pod is running and ready
3. confirm the Collector service exists and has endpoints
4. confirm the `ServiceMonitor` exists and is selected by Prometheus
5. confirm the Prometheus target for the Collector is `UP`
6. only after the platform is healthy, debug app-side OTLP export

Common failure modes:

- wrong chart values prevent the Collector from exposing the expected ports
- Prometheus does not discover the `ServiceMonitor`
- the Collector is healthy but no app sends OTLP traffic yet
- the app sends to the wrong service DNS name or protocol

## Security And Scope

This first phase is intentionally internal-only:

- the Collector is cluster-internal
- no internet exposure is required
- no authentication is added yet because traffic is limited to in-cluster workloads

If the Collector later becomes a shared gateway for many namespaces or for externally-originated telemetry, authentication and stronger network controls should be revisited.

## Testing And Verification

Platform definition of done:

- the Collector is deployed by Argo CD into `observability`
- the Collector exposes OTLP/HTTP on `4318`
- the Collector exposes a Prometheus scrape endpoint on `8889`
- the `ServiceMonitor` is present and selected
- Prometheus shows the Collector target as `UP`
- Grafana can query Collector-exposed metrics before any application-specific instrumentation is added
- the future OTLP endpoint for apps is documented and stable

Suggested verification sequence after implementation:

1. inspect Argo CD application health and sync status
2. inspect Collector pod logs for startup or pipeline errors
3. inspect Collector service and endpoints
4. inspect `ServiceMonitor` presence in the cluster
5. inspect Prometheus targets UI or target API for Collector scrape status
6. query a basic Collector metric in Grafana or Prometheus

## Future Expansion

This design intentionally leaves room for later phases:

- add OTLP/gRPC support on `4317`
- add traces pipeline and a trace backend such as Tempo
- add logs pipeline if needed
- reuse the same Collector service for additional applications
- introduce transformations, filtering, or routing rules once there is a concrete need

## Decision Summary

The platform will adopt a shared OpenTelemetry Collector gateway in `observability`, deployed with the official Helm chart through Argo CD. Applications will push OTLP metrics to the Collector. The Collector will expose Prometheus-format metrics for the existing Prometheus stack to scrape. This is the simplest platform design that is still recognizably real OpenTelemetry architecture and gives a clean path for future expansion.
