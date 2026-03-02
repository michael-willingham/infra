# Kubernetes Cluster - Agent Guide

This document provides essential information for AI agents working with the **willingham-k8s** cluster.

## ⚠️ IMPORTANT: Keep Documentation Updated

**AI Agents: You are responsible for maintaining these documentation files.**

When making changes to the cluster configuration, UPDATE the relevant documentation if you:
- Add or remove top-level directories
- Create new Kustomizations
- Add major operators or change infrastructure
- Modify network configuration (IP pools, interfaces, etc.)
- Change FluxCD structure or dependencies
- Update intentionally documented version callouts for Kubernetes, Talos, or major components
- Add new namespaces with special requirements
- Discover important troubleshooting steps
- Add, remove, or change a service that has a web UI / route

**Keep `README.md` in sync too.** The README has a Service Dashboard table with links to all `*.wcloud.sh` services, a stack table, hardware table, and repo layout. If you add/remove a service with a route, update the Service Dashboard. If you intentionally maintain a version callout, keep it in sync; otherwise prefer version-light docs and use manifests as source of truth.

**Documentation should be self-updating. Do not ask the user to update it.**

Commit changes to documentation in the same PR/commit as the related infrastructure changes.

**Maintainability rule:** do not include concrete version numbers in Markdown docs. Keep versions in YAML manifests and treat manifests as source of truth.

---

## Quick Reference

### Critical Prerequisites

**⚠️ You MUST have Tailscale connected to interact with this cluster.**

- **API Server:** `https://k8s.solarflare-jazz.ts.net`
- **Connect:** `tailscale up`
- **Disconnect:** `tailscale down`

**This is a Talos Linux cluster managed via Omni:**
- DO NOT manage Talos configs manually
- All configuration is done through [Omni Dashboard](https://willingham.na-west-1.omni.siderolabs.io)
- No SSH access to nodes (use `kubectl debug node/<name>`)

### Break-glass: Cluster access without Tailscale

If the Tailscale operator inside the cluster is down (e.g. after a bad Helm uninstall), the `k8s.solarflare-jazz.ts.net` endpoint will be unreachable. Use `omnictl` to get a kubeconfig that routes through the Omni management plane instead:

```bash
# Install omnictl if needed (download from Omni dashboard)
# Config lives at ~/.config/omni/config

# Get a kubeconfig via Omni (bypasses Tailscale entirely)
omnictl kubeconfig --cluster k8s --force

# This creates a "willingham-k8s" context that goes through Omni
kubectl --context willingham-k8s get nodes
```

You can also use `talosctl` through Omni for lower-level node access:

```bash
# Config lives at ~/.talos/config
# Requires a node name (use omnictl to discover nodes)
talosctl -n <node-name> -c k8s kubeconfig --force
```

Once the Tailscale operator is restored, switch back to the `k8s.solarflare-jazz.ts.net` context for normal operations.

### Cluster Info

- **Name:** willingham-k8s
- **Kubernetes:** Managed via Omni/Talos (check current version with `kubectl version`)
- **Nodes:** 3 control-plane nodes (no dedicated workers)
- **CNI:** Cilium
- **GitOps:** FluxCD
- **Storage:** Longhorn
- **Load Balancer:** MetalLB (L2 mode, IPs: 192.168.4.2-192.168.4.9)
- **Ingress:** kgateway (Gateway API / Envoy)

---

## Documentation Structure

All detailed documentation is in the `docs/` directory:

### Core Documentation

| Document | Purpose |
|----------|---------|
| [Cluster Info](./docs/cluster-info.md) | Cluster overview, prerequisites, node architecture |
| [FluxCD & GitOps](./docs/flux-gitops.md) | GitOps workflow, repository structure, Flux commands |
| [Networking](./docs/networking.md) | MetalLB, Gateway API, SR-IOV, network configuration |
| [Multus & SR-IOV](./docs/multus-sriov.md) | Multus CNI, SR-IOV device plugin, DGX Spark QSFP interconnect |
| [Operators](./docs/operators.md) | List of installed operators and components |
| [Dev Platform](./docs/dev-platform.md) | Developer tooling and CI/CD (Coder, Temporal, ARC runners, TFC agents) |
| [Messaging](./docs/messaging.md) | Messaging infrastructure (Kafka, NATS, RabbitMQ, Kafbat UI) |
| [Serverless](./docs/serverless.md) | Knative Serving/Eventing, Kafka broker, and Gateway API integration |

### Operational Guides

| Document | Purpose |
|----------|---------|
| [Creating Namespaces](./docs/creating-namespaces.md) | How to create namespaces with proper PodSecurity labels |
| [Adding Helm Charts](./docs/adding-helm-charts.md) | Step-by-step guide for deploying Helm charts via Flux |
| [Secrets Management](./docs/secrets-management.md) | External Secrets Operator with Infisical integration |
| [Adding Keycloak OAuth Client](./docs/adding-keycloak-oauth-client.md) | Step-by-step guide for adding OIDC auth via Keycloak |
| [Troubleshooting](./docs/troubleshooting.md) | Common issues and solutions |

---

## Quick Start - Common Tasks

### Making Changes to the Cluster

**All resources are managed via FluxCD - changes go through Git:**

```bash
# 1. Edit YAML files
vim operators/my-operator.yaml  # or observability/my-operator.yaml

# 2. Commit and push
git add .
git commit -m "Description of change"
git push

# 3. Flux auto-applies (or force reconcile)
flux reconcile kustomization <name> --with-source
```

### Creating a Namespace

```bash
# 1. Create namespaces/<name>.yaml with privileged PodSecurity labels
# 2. Add to namespaces/kustomization.yaml
# 3. Commit to Git

# See: docs/creating-namespaces.md for details
```

### Adding a Helm Chart

```bash
# 1. Verify Helm repo URL (use helm CLI or Artifact Hub)
# 2. Create namespace if needed
# 3. Create HelmRepository & HelmRelease in the appropriate stack (for example operators/, observability/, messaging/, or dev-platform/)
# 4. Commit to Git

# See: docs/adding-helm-charts.md for step-by-step guide
```

### Checking Status

```bash
# Kustomizations
flux get kustomizations

# HelmReleases
flux get helmreleases -A

# Logs
flux logs --all-namespaces --follow

# Pods
kubectl get pods -A
```

---

## Repository Structure

```
k8s-config/
├── clusters/
│   └── willingham-k8s/        # FluxCD Kustomization definitions
│       ├── flux-system/       # Flux bootstrap
│       ├── namespaces.yaml
│       ├── crds.yaml
│       ├── base.yaml
│       ├── operators.yaml
│       ├── observability.yaml
│       ├── vpa.yaml
│       ├── network.yaml
│       ├── security.yaml
│       ├── argo.yaml
│       ├── routes.yaml
│       ├── messaging.yaml
│       ├── dev-platform.yaml
│       └── serverless.yaml
├── namespaces/                # Namespace definitions
├── base/                      # Foundational operators (longhorn, cert-manager, metallb, ESO)
├── operators/                 # Application-level operators (Helm charts)
├── observability/             # Observability stack (VictoriaMetrics, OpenTelemetry, Jaeger, Headlamp, DCGM Exporter)
├── crds/                      # Custom Resource Definitions
├── network/                   # MetalLB, Gateway configs
├── messaging/                 # Messaging infrastructure (Kafka, NATS, RabbitMQ)
├── security/                  # Keycloak
├── argo/                      # ArgoCD, Workflows, Events
├── dev-platform/              # Developer tooling and platform services
├── serverless/                # Knative Serving/Eventing and Gateway API integrations
├── vpa/                       # VPA resources for application workloads
├── routes/                    # Gateway API routes
└── docs/                      # Documentation
```

---

## Kustomization Dependencies

Flux Kustomizations follow this dependency graph:

```
flux-system ─→ namespaces ─→ crds ─┬─→ base ─→ operators ─┬─→ observability ─→ vpa
                                   │                      ├─→ messaging
                                   ├─→ routes              ├─→ network
                                                          ├─→ dev-platform
                                                          ├─→ serverless
                                                          ├─→ security
                                                          └─→ argo
```

Defined in `clusters/willingham-k8s/*.yaml` via:
```yaml
spec:
  dependsOn:
    - name: operators
```

---

## Important Notes

### Pod Security Standards (Talos Requirement)

**Most namespaces need privileged PodSecurity labels on Talos:**

```yaml
labels:
  pod-security.kubernetes.io/audit: privileged
  pod-security.kubernetes.io/enforce: privileged
  pod-security.kubernetes.io/warn: privileged
```

See [Creating Namespaces](./docs/creating-namespaces.md) for why this is required.

### OCI Helm Charts

**When using OCI Helm repositories, use the BASE registry path:**

```yaml
# ❌ WRONG
url: oci://docker.io/org/chart-name

# ✅ CORRECT
url: oci://docker.io/org  # Flux appends chart name
```

See [Adding Helm Charts](./docs/adding-helm-charts.md#for-oci-helm-charts) for details.

### Multus & SR-IOV (DGX Spark QSFP)

Multus CNI and a standalone SR-IOV device plugin DaemonSet provide secondary 200Gbps network interfaces on the DGX Spark nodes via ConnectX-7 SR-IOV VFs. **Cilium must have `cni.exclusive=false`** — this is set in the Omni `200-k8s` config patch and applied live via `cilium config set`.

See [Multus & SR-IOV docs](./docs/multus-sriov.md) for full details.

### MetalLB L2 Advertisements

**Different nodes have different primary interfaces:**
- Custom PC (talos-76w-3r0): `enp12s0`
- DGX Sparks (talos-7aj-lwl, talos-ysi-4k0): `enP7s7`

Separate L2Advertisement resources exist for each node type.

See [Networking](./docs/networking.md#l2-advertisements) for details.

---

## Best Practices

1. ✅ **Always commit changes to Git** - Let Flux apply them
2. ✅ **Create namespaces before workloads** - With privileged labels
3. ✅ **Verify Helm chart URLs** - Test with `helm` CLI first
4. ✅ **Pin Helm chart versions** - Never use "latest"
5. ✅ **Set proper dependencies** - CRDs → base → operators → observability/network → apps
6. ✅ **Check Flux status after changes** - `flux get kustomizations`
7. ✅ **Use Tailscale for cluster access** - Required for API server
8. ✅ **Wait for GitHub Copilot PR review** - Copilot is expected to auto-review PRs in this repo; after opening a PR, wait for Copilot to finish, determine whether you agree or disagree with its feedback, and address all relevant comments; if no Copilot review appears after 10 minutes, request human review
9. ✅ **Only merge when fully green** - Do not merge until all review threads are resolved, required approvals are present, and all required status checks pass

---

## External Resources

- **Omni Dashboard:** https://willingham.na-west-1.omni.siderolabs.io
- **Talos Docs:** https://www.talos.dev
- **Kubernetes Docs:** https://kubernetes.io/docs/
- **FluxCD Docs:** https://fluxcd.io/flux/
- **MetalLB Docs:** https://metallb.universe.tf
- **Artifact Hub:** https://artifacthub.io (Helm charts)
