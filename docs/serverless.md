# Serverless Platform

Components in the `serverless` Kustomization that provide Knative Serving and Eventing on top of kgateway and Kafka.

## Overview

- **Flux Kustomization:** `clusters/willingham-k8s/serverless.yaml`
- **Depends on:** `namespaces`, `operators` (includes `knative-operator` and `kgateway`)
- **Manifest source:** `serverless/`

## Components

### Knative Serving

- **Resource:** `KnativeServing/knative-serving`
- **Namespace:** `knative-serving`
- **File:** `serverless/knative-serving.yaml`
- **Ingress class:** `gateway-api.ingress.networking.knative.dev`
- **Domain:** `kn.wcloud.sh`
- **URL template:** `{service}-{namespace}.kn.wcloud.sh`

### Knative Eventing

- **Resource:** `KnativeEventing/knative-eventing`
- **Namespace:** `knative-eventing`
- **File:** `serverless/knative-eventing.yaml`
- **Default broker class:** `Kafka`
- **Kafka bootstrap servers:** `kafka-cluster-kafka-bootstrap.kafka.svc.cluster.local:9092`
- **Default topic settings:** `partitions=3`, `replication.factor=3`

### eventing-kafka-broker

- **Purpose:** Kafka data/control plane for Knative brokers
- **Source:** `serverless/eventing-kafka/kustomization.yaml`
- **Release ref:** managed in `serverless/eventing-kafka/kustomization.yaml` (GitHub release URL refs)
- **Installed manifests:**
  - `eventing-kafka-controller.yaml`
  - `eventing-kafka-broker.yaml`

### net-gateway-api

- **Purpose:** Bridges Knative ingress to Kubernetes Gateway API
- **Source:** `serverless/net-gateway-api/kustomization.yaml`
- **Release ref:** managed in `serverless/net-gateway-api/kustomization.yaml` (GitHub release URL refs)
- **Patched ConfigMap:** `config-gateway` in `knative-serving`
- **Configured gateways:**
  - External: `external-gateway` (`kgateway-system`)
  - Local: `knative-local-gateway` (`kgateway-system`)

### Knative Local Gateway

- **Resource:** `Gateway/knative-local-gateway`
- **Namespace:** `kgateway-system`
- **File:** `serverless/knative-local-gateway.yaml`
- **Service type:** `ClusterIP`
- **Purpose:** Internal routing for Knative local traffic

## Routing and TLS

- Wildcard hostnames are handled by the external gateway listeners defined in `network/gateway.yaml`:
  - `*.wcloud.sh`
  - `*.kn.wcloud.sh`
- Knative service routes are served under `*.kn.wcloud.sh`.

## Quick Verification

```bash
# Flux status
flux get kustomizations serverless

# Knative CRs
kubectl get knativeserving -n knative-serving
kubectl get knativeeventing -n knative-eventing

# Gateway resources
kubectl get gateways -n kgateway-system

# Kafka broker plane
kubectl get pods -n knative-eventing | grep kafka
```
