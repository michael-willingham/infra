# Multus CNI + SR-IOV for DGX Spark QSFP Interconnect

## Overview

This cluster uses [Multus CNI](https://github.com/k8snetworkplumbingwg/multus-cni) as a meta-plugin alongside Cilium to attach secondary high-speed network interfaces to pods. SR-IOV Virtual Functions (VFs) are created at boot via a standalone DaemonSet on the Mellanox ConnectX-7 NICs in the DGX Spark nodes, enabling near-native network performance with RDMA support.

**Why not the SR-IOV Network Operator?** Talos Linux has a read-only root filesystem. The SR-IOV operator's config daemon crashes trying to `mkdir /etc/sriov-operator`. Instead we use standalone components: an init container creates VFs via sysfs, and the [SR-IOV device plugin](https://github.com/k8snetworkplumbingwg/sriov-network-device-plugin) advertises them to kubelet.

## Architecture

```
Pod
├── eth0 (Cilium — primary CNI, home network)
└── net1 (SR-IOV VF — 200Gbps QSFP direct-attach link)
         │
    Multus (meta-plugin, delegates to both CNIs)
```

**Components:**
- **Multus CNI** (`kube-system`) — meta-plugin that wraps Cilium and enables additional network attachments per pod
- **SR-IOV Device Plugin DaemonSet** (`kube-system`) — standalone DaemonSet that:
  - **Init: `sriov-cni`** — installs the `sriov` CNI binary to `/opt/cni/bin/`
  - **Init: `create-vfs`** — creates 4 VFs on the CX-7 by writing to sysfs (no-op on non-DGX nodes)
  - **Main: `sriov-device-plugin`** — advertises VFs to kubelet as `nvidia.com/cx7_qsfp`

## Hardware Inventory

Each DGX Spark has two Mellanox ConnectX-7 (MT2910) dual-port NICs:

| Interface | PCI Bus | Link | Speed | Status | Purpose |
|---|---|---|---|---|---|
| `enP7s7` | `0007:01:00.0` | Realtek r8169 | 1G | Primary NIC | Cilium / home network |
| `enp1s0f0np0` | `0000:01:00.0` | CX-7 mlx5 | — | No cable | Unused (port 0) |
| **`enp1s0f1np1`** | `0000:01:00.1` | **CX-7 mlx5** | **200G** | **DAC link UP** | **SR-IOV configured** |
| `enP2p1s0f0np0` | `0002:01:00.0` | CX-7 mlx5 | — | No cable | Unused (port 0) |
| `enP2p1s0f1np1` | `0002:01:00.1` | CX-7 mlx5 | 200G | DAC link UP | Future expansion |

**Nodes:**
- `talos-7aj-lwl` (192.168.4.186)
- `talos-ysi-4k0` (192.168.4.185)

**SR-IOV capability:** 8 VFs per port (currently 4 configured per node).

**Kernel modules (verified loaded):**
- `mlx5_core` — ConnectX-7 base driver
- `mlx5_ib` — InfiniBand / RDMA verbs
- `ib_uverbs` — userspace RDMA verbs (4 devices: `uverbs0-3`)
- `vfio_pci` — VFIO device passthrough

## Prerequisite: Cilium `cni.exclusive=false`

Cilium must be configured to not overwrite other CNI configs in `/etc/cni/net.d/`. This is set in the Omni `200-k8s` config patch (the inline Cilium install Job) and applied to the live cluster with the `cilium` CLI.

**Omni patch** (`200-k8s`): includes `--set cni.exclusive=false` in the `cilium install` command. This config is managed in the [Omni Dashboard](https://willingham.na-west-1.omni.siderolabs.io) (not in this Git repo — Talos/Omni configs are managed separately per `AGENTS.md`). To verify or update, run:
```bash
omnictl get configpatch 200-k8s -o yaml | grep -A2 exclusive
```

**Live update** (one-time, already applied):
```bash
cilium config set cni-exclusive false
```

Without this, Cilium periodically overwrites the Multus CNI config and breaks secondary network attachments.

**Reference:** [Talos Multus guide — Cilium integration](https://docs.siderolabs.com/kubernetes-guides/cni/multus)

## GitOps Layout

```
base/
├── multus/
│   └── kustomization.yaml        # Multus thick DaemonSet (upstream manifest + Talos patches)
└── sriov-device-plugin.yaml      # Standalone SR-IOV device plugin DaemonSet + ConfigMap

network/
└── dgx-spark-sriov.yaml          # NetworkAttachmentDefinition for QSFP VFs
```

**FluxCD dependency chain:** The NetworkAttachmentDefinition in `network/` depends on `base/` (via `operators/`), so the device plugin is running before the NetworkAttachmentDefinition is created.

## Using the QSFP Network in Pods

### Basic pod with secondary QSFP interface

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-gpu-pod
  annotations:
    k8s.v1.cni.cncf.io/networks: dgx-qsfp
spec:
  runtimeClassName: nvidia
  nodeSelector:
    kubernetes.io/hostname: talos-7aj-lwl  # must land on a DGX Spark
  containers:
    - name: app
      image: nvcr.io/nvidia/pytorch:<tag>
      resources:
        limits:
          nvidia.com/gpu: 1
          nvidia.com/cx7_qsfp: 1  # request an SR-IOV VF
```

The pod gets:
- `eth0` — Cilium interface (home network, cluster connectivity)
- `net1` — SR-IOV VF on the 200Gbps QSFP link (IP from 10.100.0.0/24)

### NCCL multi-node GPU training

For distributed training across both DGX Sparks, configure NCCL to use the QSFP interface:

```yaml
env:
  - name: NCCL_SOCKET_IFNAME
    value: net1
  - name: NCCL_IB_DISABLE
    value: "0"            # Enable InfiniBand/RoCE (RDMA)
  - name: NCCL_NET_GDR_LEVEL
    value: "5"            # Enable GPU-Direct RDMA
  - name: NCCL_P2P_LEVEL
    value: "SYS"          # Allow P2P across PCIe switches
  - name: NCCL_DEBUG
    value: "INFO"         # Useful for initial debugging
```

## Talos-Specific Patches (Multus)

The Multus thick DaemonSet from upstream requires these patches for Talos Linux:

1. **`/var/run/netns/` volume path** — Talos uses `/var/run/netns/` instead of `/run/netns/`. Without this fix, new pods fail with network sandbox creation errors.

2. **Resource limits** — bumped from 100m/50Mi to 300m/150Mi to prevent OOM on nodes with many pods.

These are applied via kustomize patches in `base/multus/kustomization.yaml`.

## SR-IOV Configuration Details

### Device Plugin ConfigMap (`sriov-dp-config`)

```json
{
  "resourceList": [{
    "resourceName": "cx7_qsfp",
    "resourcePrefix": "nvidia.com",
    "selectors": {
      "pfNames": ["enp1s0f1np1#0-3"],
      "vendors": ["15b3"],
      "devices": ["101e"],
      "isRdma": true
    }
  }]
}
```

- **Resource:** `nvidia.com/cx7_qsfp` (4 per DGX Spark node)
- **RDMA:** enabled — each VF allocation includes both the netdev and RDMA device
- **VF range:** `#0-3` selects VFs 0 through 3 on the PF

### NetworkAttachmentDefinition (`dgx-qsfp`)

Created manually in `network/dgx-spark-sriov.yaml`:
- **IPAM:** `host-local` with subnet `10.100.0.0/24` (range 10.100.0.1–10.100.0.10)
- **CNI type:** `sriov` (uses the binary installed by the init container)
- **Resource annotation:** `nvidia.com/cx7_qsfp` (tells Multus which device plugin to request VFs from)

### Adding the Second CX-7 Interface

The NVLink bridge CX-7 (`enP2p1s0f1np1`, PCI `0002:01:00.1`) also has a 200G DAC link. To add it:

1. Update the `create-vfs` init container in `base/sriov-device-plugin.yaml` to also create VFs on `enP2p1s0f1np1`
2. Add a second entry to the `sriov-dp-config` ConfigMap
3. Add a second `NetworkAttachmentDefinition` in `network/dgx-spark-sriov.yaml`

## Troubleshooting

### Check Multus

```bash
# Verify Multus DaemonSet is running on all nodes
kubectl get ds -n kube-system kube-multus-ds

# Check Multus logs
kubectl logs -n kube-system -l app=multus --tail=50
```

### Check SR-IOV Device Plugin

```bash
# Device plugin pods
kubectl get pods -n kube-system -l app=sriov-device-plugin

# Check init container logs (VF creation)
kubectl logs -n kube-system -l app=sriov-device-plugin -c create-vfs

# Check device plugin logs
kubectl logs -n kube-system -l app=sriov-device-plugin -c sriov-device-plugin --tail=50

# Check NetworkAttachmentDefinition
kubectl get net-attach-def -A

# Check allocatable resources on a DGX node
kubectl get node talos-7aj-lwl -o jsonpath='{.status.allocatable}' | jq .
# Should show "nvidia.com/cx7_qsfp": "4"
```

### Check RDMA

```bash
# Run a test pod with RDMA tools
kubectl run rdma-test --image=ghcr.io/mellanox/rdma-tools --restart=Never \
  --overrides='{"metadata":{"annotations":{"k8s.v1.cni.cncf.io/networks":"dgx-qsfp"}},"spec":{"nodeSelector":{"kubernetes.io/hostname":"talos-7aj-lwl"},"containers":[{"name":"rdma-test","image":"ghcr.io/mellanox/rdma-tools","command":["sleep","infinity"],"resources":{"limits":{"nvidia.com/cx7_qsfp":"1"}}}]}}'

# Inside the pod:
kubectl exec rdma-test -- ibv_devinfo
kubectl exec rdma-test -- ibv_rc_pingpong
```

### Test connectivity

```bash
# Run a test pod on each DGX Spark
kubectl run test-a --image=nicolaka/netshoot --restart=Never \
  --overrides='{"metadata":{"annotations":{"k8s.v1.cni.cncf.io/networks":"dgx-qsfp"}},"spec":{"nodeSelector":{"kubernetes.io/hostname":"talos-7aj-lwl"},"containers":[{"name":"test-a","image":"nicolaka/netshoot","command":["sleep","infinity"],"resources":{"limits":{"nvidia.com/cx7_qsfp":"1"}}}]}}'

kubectl run test-b --image=nicolaka/netshoot --restart=Never \
  --overrides='{"metadata":{"annotations":{"k8s.v1.cni.cncf.io/networks":"dgx-qsfp"}},"spec":{"nodeSelector":{"kubernetes.io/hostname":"talos-ysi-4k0"},"containers":[{"name":"test-b","image":"nicolaka/netshoot","command":["sleep","infinity"],"resources":{"limits":{"nvidia.com/cx7_qsfp":"1"}}}]}}'

# Check IPs
kubectl exec test-a -- ip addr show net1
kubectl exec test-b -- ip addr show net1

# Ping across the 200G link
kubectl exec test-a -- ping -c 3 <test-b-net1-ip>

# Bandwidth test (iperf3)
kubectl exec test-a -- iperf3 -s &
kubectl exec test-b -- iperf3 -c <test-a-net1-ip> -t 10
```

## Future Work

- **GPU-Direct RDMA benchmarks** — the `nvidia-gdrdrv-device` Talos extension is already installed; test with NCCL all-reduce benchmarks
- **RDMA Shared Device Plugin** — for workloads that need RDMA without SR-IOV VFs
- **Whereabouts IPAM** — cluster-wide IPAM for dynamic IP allocation across namespaces (replaces `host-local` per-node IPAM)
- **Second CX-7 interface** — add `enP2p1s0f1np1` for 400Gbps aggregate bandwidth
