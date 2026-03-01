# Cluster Information

## Overview

- **Name:** willingham-k8s
- **Management:** Talos Linux via Omni
- **Kubernetes:** Managed via Omni/Talos control plane (check current version with `kubectl version`)
- **CNI:** Cilium
- **GitOps:** FluxCD
- **Storage:** Longhorn (distributed block storage)
- **Load Balancer:** MetalLB (L2 mode)
- **Ingress:** kgateway (Gateway API / Envoy)

## Critical Prerequisites

### Tailscale Access Required

**⚠️ You MUST have Tailscale connected to interact with this cluster.**

- **API Server:** `https://k8s.solarflare-jazz.ts.net` (Tailscale hostname)
- **Tailnet:** solarflare-jazz
- To connect: `tailscale up`
- To disconnect: `tailscale down`
- **kubectl will NOT work without Tailscale connected** (the API server hostname only resolves on Tailscale)

### Talos Linux & Omni

**⚠️ This is an Omni-managed Talos cluster**

- **DO NOT attempt to manage Talos configs manually** - all configuration is done through Omni
- There are no local `machineconfig.yaml` files
- Use `omnictl` to interact with Omni (if needed)
- Talos configs are stored in Omni, not in this repo
- Node SSH is not available (Talos is API-driven)
- **Omni Dashboard:** https://willingham.na-west-1.omni.siderolabs.io

## Node Architecture

### 3-Node Control Plane Cluster

All nodes are control-plane nodes (no dedicated workers).

| Hostname | Node Type | IP Address | Hardware | Primary Interface |
|----------|-----------|------------|----------|-------------------|
| talos-76w-3r0 | Custom PC | 192.168.4.137 | Custom build | **enp12s0** |
| talos-7aj-lwl | DGX Spark | 192.168.4.186 | Nvidia DGX Spark (Asus OEM) | **enP7s7** |
| talos-ysi-4k0 | DGX Spark | 192.168.4.185 | Nvidia DGX Spark (Asus OEM) | **enP7s7** |

### Network Interface Details

**Custom PC (talos-76w-3r0):**
- Primary: `enp12s0` (connected to 192.168.4.0/24 home network)
- Secondary: `enp11s0` (no carrier/unused)
- Virtual: `cilium_vxlan`, `cilium_host`, various `lxc*` interfaces

**DGX Sparks (talos-7aj-lwl, talos-ysi-4k0):**
- Primary: `enP7s7` (connected to 192.168.4.0/24 home network)
- Onboard NICs: `enp1s0f0np0`, `enp1s0f1np1`
- NVLink: `enP2p1s0f0np0`, `enP2p1s0f1np1` (PCI bridge interfaces for inter-DGX communication)
- Virtual: `cilium_vxlan`, `cilium_host`, various `lxc*` interfaces

### Accessing Node Information

Since Talos doesn't have SSH:

```bash
# Debug pod with node access
kubectl debug node/<node-name> -it --image=nicolaka/netshoot

# With host network access (requires kube-system namespace):
kubectl run debug -n kube-system --image=nicolaka/netshoot --restart=Never \
  --overrides='{"spec":{"nodeName":"<node>","hostNetwork":true}}' \
  --command -- sleep 600
```

## Tailscale Integration (GitOps-Managed)

This repository deploys the Tailscale operator via `operators/tailscale.yaml`:

- **Helm chart:** `tailscale-operator` (version pinned in `operators/tailscale.yaml`)
- **Namespace:** `tailscale`
- **Operator hostname:** `k8s`
- **API server proxy mode:** enabled (`apiServerProxyConfig.mode: "true"`)

Any Tailscale `Connector`, `ProxyClass`, or subnet-router CRs are not currently declared in this repository.

## External Resources

- **Omni Dashboard:** https://willingham.na-west-1.omni.siderolabs.io
- **Talos Documentation:** https://www.talos.dev
- **Kubernetes Docs:** https://kubernetes.io/docs/
