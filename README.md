<div align="center">

# 🏠 willingham-k8s

**Home Kubernetes cluster — my learning lab for NVIDIA GPU workloads on k8s,
building custom operators, Temporal workflows, and whatever else catches my interest.**

3 nodes · 448 GB RAM · ~3.4 PFLOPS AI compute · GitOps with FluxCD

![Kubernetes](https://img.shields.io/badge/kubernetes-cluster-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![Talos Linux](https://img.shields.io/badge/talos_linux-OS-FF7300?style=for-the-badge&logo=linux&logoColor=white)
![FluxCD](https://img.shields.io/badge/FluxCD-GitOps-5468FF?style=for-the-badge&logo=flux&logoColor=white)
![Cilium](https://img.shields.io/badge/cilium-CNI-F8C517?style=for-the-badge&logo=cilium&logoColor=black)

</div>

---

## 📑 Table of Contents

- [🔗 Service Dashboard](#service-dashboard)
- [🧱 The Stack](#the-stack)
- [🖥️ The Hardware](#the-hardware)
- [📂 Repo Layout](#repo-layout)
- [🔄 GitOps Workflow](#gitops-workflow)
- [🔑 Access](#access)
- [🛠️ Useful Commands](#useful-commands)
- [📚 Docs](#docs)

---

## 🔗 Service Dashboard

> **Note:** All `*.wcloud.sh` services are only accessible from my Tailscale network. These links won't resolve for anyone outside the tailnet. The dashboard also lists a few external/out-of-band services (e.g. Omni) that are not on `wcloud.sh`.

Most internal cluster UIs are exposed under `*.wcloud.sh`; selected internal services use Keycloak/OIDC SSO as configured per app. External links may have different auth or be publicly accessible.

| Service | URL | What it does |
|---------|-----|--------------|
| 🎛️ **Headlamp** | [k8s.wcloud.sh](https://k8s.wcloud.sh) | Kubernetes dashboard — see your pods, logs, everything |
| 📊 **Grafana** | [grafana.wcloud.sh](https://grafana.wcloud.sh) | Metrics, dashboards, pretty graphs |
| 📈 **Victoria Metrics** | [metrics.wcloud.sh](https://metrics.wcloud.sh) | Raw metrics backend |
| 🚨 **AlertManager** | [alerts.wcloud.sh](https://alerts.wcloud.sh) | Alert routing & silencing |
| 🚀 **ArgoCD** | [argo-cd.wcloud.sh](https://argo-cd.wcloud.sh) | GitOps deployments for apps |
| ⚡ **Argo Workflows** | [argo-workflows.wcloud.sh](https://argo-workflows.wcloud.sh) | Workflow engine |
| ⏱️ **Temporal** | [temporal.wcloud.sh](https://temporal.wcloud.sh) | Durable execution platform |
| 🐰 **RabbitMQ** | [rabbitmq.wcloud.sh](https://rabbitmq.wcloud.sh) | Message broker management |
| 📨 **Kafbat UI** | [kafka.wcloud.sh](https://kafka.wcloud.sh) | Kafka cluster management |
| 💾 **Longhorn** | [longhorn.wcloud.sh](https://longhorn.wcloud.sh) | Distributed storage UI |
| 🔎 **Jaeger** | [jaeger.wcloud.sh](https://jaeger.wcloud.sh) | Distributed tracing |
| 🔭 **Hubble** | [hubble.wcloud.sh](https://hubble.wcloud.sh) | Cilium network observability |
| 💻 **Coder** | [coder.wcloud.sh](https://coder.wcloud.sh) | Cloud dev environments |
| 🔗 **Kaoto** | [camel.wcloud.sh](https://camel.wcloud.sh) | Apache Camel K visual integration designer |
| 🔐 **Keycloak** | [keycloak.wcloud.sh](https://keycloak.wcloud.sh) | Identity & SSO provider |
| 🌐 **Omni** | [Omni Dashboard](https://willingham.na-west-1.omni.siderolabs.io) | Talos cluster management |

---

## 🧱 The Stack

| Layer | Tech | Badge |
|-------|------|-------|
| 🐧 OS | [Talos Linux](https://www.talos.dev) via [Omni](https://willingham.na-west-1.omni.siderolabs.io) | ![Talos](https://img.shields.io/badge/talos-immutable_OS-FF7300?logo=linux&logoColor=white) |
| ☸️ Orchestration | Kubernetes | ![K8s](https://img.shields.io/badge/k8s-cluster-326CE5?logo=kubernetes&logoColor=white) |
| 🔄 GitOps | FluxCD | ![Flux](https://img.shields.io/badge/flux-gitops-5468FF) |
| 🌐 CNI | Cilium | ![Cilium](https://img.shields.io/badge/cilium-eBPF-F8C517?logo=cilium&logoColor=black) |
| 💾 Storage | Longhorn (data engine) | ![Longhorn](https://img.shields.io/badge/longhorn-distributed-0078D4) |
| ⚖️ Load Balancer | MetalLB L2 (`192.168.4.2-9`) | ![MetalLB](https://img.shields.io/badge/metallb-L2-important) |
| 🚪 Ingress | kgateway (Gateway API / Envoy) | ![Gateway API](https://img.shields.io/badge/gateway_API-envoy-6C3FC5) |
| 🔐 Auth | Keycloak SSO (OIDC) | ![Keycloak](https://img.shields.io/badge/keycloak-SSO-00B8E2) |
| 🔑 Secrets | External Secrets + Infisical | ![ESO](https://img.shields.io/badge/ESO-infisical-green) |
| 📊 Monitoring | Victoria Metrics + Grafana | ![VictoriaMetrics](https://img.shields.io/badge/victoria_metrics-monitoring-621773) |
| 📬 Messaging | Kafka · NATS · RabbitMQ | ![Messaging](https://img.shields.io/badge/messaging-3_brokers-orange) |
| ⚡ Serverless | Knative Serving + Eventing (Kafka broker) | ![Knative](https://img.shields.io/badge/knative-serving%2Feventing-blue) |

---

## 🖥️ The Hardware

| Node | Hardware | CPU | RAM | Notes |
|------|----------|-----|-----|-------|
| `talos-76w-3r0` | 🖥️ Custom PC | AMD Ryzen 9 9950X3D (16c/32t) | 192 GB DDR5 | 3D V-Cache · RTX 5070 Ti (~1.4 PFLOP FP4) |
| `talos-7aj-lwl` | 🟢 DGX Spark | NVIDIA GB10 Grace Blackwell (20c ARM) | 128 GB | 1 PFLOP FP4 |
| `talos-ysi-4k0` | 🟢 DGX Spark | NVIDIA GB10 Grace Blackwell (20c ARM) | 128 GB | 1 PFLOP FP4 |

**Total:** 56 cores / 72 threads · ~448 GB RAM · ~3.4 PFLOPS FP4 · All-Blackwell GPUs · All 3 nodes are control-plane

The two DGX Sparks are connected via a 200Gbps QSFP DAC link (ConnectX-7, SR-IOV via [Multus + standalone SR-IOV device plugin](./docs/multus-sriov.md)).

---

## 📂 Repo Layout

```
📁 clusters/willingham-k8s/   ← FluxCD entry points
📁 namespaces/                ← Namespace definitions
📁 base/                      ← Foundational operators (storage, certs, LB, secrets)
📁 operators/                 ← Helm charts for infrastructure
📁 crds/                      ← Custom Resource Definitions
📁 network/                   ← MetalLB + Gateway API configs
📁 messaging/                 ← Kafka, NATS, RabbitMQ
📁 security/                  ← Keycloak
📁 observability/             ← Metrics, tracing, cluster UI (VMKS, Jaeger, Headlamp)
📁 argo/                      ← ArgoCD, Workflows, Events
📁 dev-platform/              ← Coder, GitHub Actions runners, Temporal, TFC
📁 serverless/                ← Knative Serving/Eventing + Gateway API integration
📁 vpa/                       ← Vertical Pod Autoscaler configs
📁 routes/                    ← Gateway API HTTPRoutes (*.wcloud.sh)
📁 docs/                      ← Deep-dive documentation
```

### 🔗 FluxCD Dependency Chain

```
flux-system ─→ namespaces ─→ crds ─┬─→ base ─→ operators ─┬─→ observability ─→ vpa
                                   │                      ├─→ messaging
                                   ├─→ routes              ├─→ network
                                                          ├─→ dev-platform
                                                          ├─→ serverless
                                                          ├─→ security
                                                          └─→ argo
```

---

## 🔄 GitOps Workflow

All changes go through Git. Never `kubectl apply` directly — Flux handles reconciliation.

```bash
# 1. Edit your YAML
vim operators/my-app.yaml  # or observability/my-app.yaml

# 2. Commit & push
git add . && git commit -m "update my-app" && git push

# 3. Flux picks it up automatically, or force it:
flux reconcile kustomization operators --with-source
```

---

## 🔑 Access

Requires [Tailscale](https://tailscale.com) — the cluster API server is only reachable via the tailnet.

```bash
# Connect
tailscale up

# Verify
kubectl get nodes

# API server
# https://k8s.solarflare-jazz.ts.net
```

---

## 🛠️ Useful Commands

```bash
# 🔍 What's deployed?
flux get kustomizations
flux get helmreleases -A

# 🔥 Something broken?
flux logs -f
kubectl get events -A --sort-by='.lastTimestamp'

# 🐛 Debug a node (no SSH on Talos!)
kubectl debug node/talos-xxx -it --image=alpine
```

---

## 📚 Docs

More details in [`docs/`](./docs/):

| Doc | What you'll learn |
|-----|-------------------|
| 📋 [Cluster Info](./docs/cluster-info.md) | Node architecture, baseline, prerequisites |
| 🔄 [FluxCD & GitOps](./docs/flux-gitops.md) | How GitOps works here, Flux commands |
| 🌐 [Networking](./docs/networking.md) | MetalLB, Gateway API, DNS, TLS |
| 📦 [Operators](./docs/operators.md) | Full list of installed operators |
| 🏗️ [Dev Platform](./docs/dev-platform.md) | Coder, Temporal, ARC runners, TFC |
| 📬 [Messaging](./docs/messaging.md) | Kafka, NATS, RabbitMQ setup |
| ⚡ [Serverless](./docs/serverless.md) | Knative Serving/Eventing + Kafka broker |
| 📝 [Creating Namespaces](./docs/creating-namespaces.md) | PodSecurity labels (Talos needs `privileged`) |
| 📦 [Adding Helm Charts](./docs/adding-helm-charts.md) | Step-by-step Helm chart deployment |
| 🔑 [Secrets Management](./docs/secrets-management.md) | External Secrets + Infisical |
| 🔐 [Keycloak OAuth](./docs/adding-keycloak-oauth-client.md) | Adding OIDC clients |
| 🔧 [Troubleshooting](./docs/troubleshooting.md) | When things go sideways |
