# Installed Operators and Platform Components

This document tracks what is installed and where it is defined.

Version numbers are intentionally omitted for maintainability unless a version is operationally significant. The source of truth for exact versions is the manifests in this repo.

## Find Exact Versions

```bash
# Show every HelmRelease chart and pinned version
rg -l "kind: HelmRelease" -g '*.yaml' | while read -r f; do
  yq e -N 'select(.kind == "HelmRelease") | [.metadata.namespace, .metadata.name, .spec.chart.spec.chart, .spec.chart.spec.version] | @tsv' "$f"
done
```

## FluxCD

- **Bootstrap manifests:** `clusters/willingham-k8s/flux-system/`
- **Core controllers:** `source-controller`, `kustomize-controller`, `helm-controller`, `notification-controller`

## Base Infrastructure (`base/`)

| Component | Namespace | Deployment | Notes |
|-----------|-----------|------------|-------|
| Cert-Manager | `cert-manager` | HelmRelease (`base/cert-manager.yaml`) | Let's Encrypt integration |
| External Secrets Operator | `external-secrets` | HelmRelease (`base/external-secrets.yaml`) | Includes `ClusterSecretStore/infisical` |
| Longhorn | `longhorn-system` | HelmRelease (`base/longhorn.yaml`) | Distributed storage, data engine configured in manifest |
| MetalLB | `metallb-system` | HelmRelease (`base/metallb.yaml`) | L2 mode, pool `192.168.4.2-192.168.4.9` |
| Metrics Server | `kube-system` | HelmRelease (`base/metrics-server.yaml`) | Cluster metrics API |
| NVIDIA Device Plugin | `nvidia-device-plugin` | HelmRelease (`base/nvidia-device-plugin.yaml`) | GPU resource discovery |
| Multus CNI | `kube-system` | Kustomize remote manifest (`base/multus/`) | Talos netns path + resource patches |
| SR-IOV Device Plugin | `kube-system` | DaemonSet (`base/sriov-device-plugin.yaml`) | Standalone Talos-compatible SR-IOV setup |
| OLM | `olm` | Kustomize remote manifest (`base/olm/`) | Required by OperatorHub subscriptions |

## Operators (`operators/`)

| Operator | Namespace | Deployment | Notes |
|----------|-----------|------------|-------|
| Actions Runner Controller | `arc-systems` | HelmRelease `arc-controller` | GitHub runner scale set controller |
| Apache Camel K | `camel-k` | HelmRelease `camel-k` | Integration runtime/operator |
| Kaoto Operator | `camel-k` | OLM Subscription (`channel: alpha`) | Designer operator |
| Dragonfly Operator | `dragonfly` | HelmRelease `dragonfly-operator` | Redis-compatible datastore operator |
| External-DNS | `external-dns` | HelmRelease `external-dns` | DNS record sync for service/gateway hostnames |
| KEDA | `keda` | HelmRelease `keda` | Event-driven autoscaling |
| Keycloak Operator | `keycloak` | OLM Subscription (`channel: fast`) | Keycloak CR management |
| kgateway | `kgateway-system` | HelmChart + HelmRelease (`operators/kgateway.yaml`) | Gateway API ingress plane |
| Knative Operator | `knative-operator` | HelmRelease `knative-operator` | Knative Serving/Eventing operator |
| OpenTelemetry Operator | `otel-system` | HelmRelease `opentelemetry-operator` | OTel collector/operator control plane |
| Percona Postgres Operator | `postgres` | HelmRelease `pg-operator` | PostgreSQL cluster operator |
| RabbitMQ Cluster Operator | `rabbitmq` | OLM Subscription (`channel: stable`) | RabbitMQ CR management |
| Strimzi Kafka Operator | `kafka` | HelmRelease `strimzi-cluster-operator` | Kafka operator |
| Tailscale Operator | `tailscale` | HelmRelease `tailscale` | Tailnet integration |
| HCP Terraform Operator | `tfc-operator-system` | HelmRelease `tfc-operator` | Terraform Cloud/HCP operator |
| Vertical Pod Autoscaler | `vpa` | HelmRelease `vpa` | VPA components and admission controller |

## App Platform Releases

These are workload/platform releases managed by Flux, not cluster operators:

- **Dev Platform (`dev-platform/`)**
  - `coder` + `coder-db`
  - `temporal` + `temporal-db`
  - `arc-runner-amd64` / `arc-runner-arm64`
  - `armada-operator`
  - `AgentPool` resources (`tfc-agent-amd64`, `tfc-agent-arm64`)

- **Messaging (`messaging/`)**
  - Kafka cluster (`Kafka` CR) on Strimzi
  - `kafbat-ui`
  - `nats`
  - `RabbitmqCluster/rabbitmq-cluster`

- **Observability (`observability/`)**
  - `vmks` (`victoria-metrics-k8s-stack`)
  - `dcgm-exporter` (NVIDIA GPU metrics via DCGM)
  - `headlamp`
  - `OpenTelemetryCollector/jaeger`

- **Serverless (`serverless/`)**
  - `KnativeServing/knative-serving`
  - `KnativeEventing/knative-eventing`
  - `eventing-kafka-broker` manifests (pinned in `serverless/eventing-kafka/`)
  - `net-gateway-api` manifests (pinned in `serverless/net-gateway-api/`)

## Related Docs

- [Dev Platform](./dev-platform.md)
- [Messaging](./messaging.md)
- [Serverless](./serverless.md)
- [Networking](./networking.md)
- [Multus & SR-IOV](./multus-sriov.md)
