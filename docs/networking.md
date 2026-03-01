# Networking

## Home Network

- **Network:** `192.168.4.0/24` (actually a /22: `192.168.4.0/22`)
- **Gateway:** `192.168.4.1`
- **DNS:** `192.168.4.1`
- **MetalLB Range:** `192.168.4.2-192.168.4.9`

## MetalLB (Load Balancer)

- **Namespace:** `metallb-system`
- **Mode:** L2 (Layer 2) only
- **IP Pool:** `192.168.4.2-192.168.4.9`

### L2 Advertisements

The cluster uses **separate L2Advertisement resources per node type** due to different network interface naming:

```yaml
# Custom PC advertisement
willingham-private-ip-l2-advertisement-custom-pc:
  interfaces: [enp12s0]
  nodeSelectors:
    - matchLabels: {kubernetes.io/hostname: talos-76w-3r0}

# DGX Sparks advertisement
willingham-private-ip-l2-advertisement-dgx:
  interfaces: [enP7s7]
  nodeSelectors:
    - matchLabels: {kubernetes.io/hostname: talos-7aj-lwl}
    - matchLabels: {kubernetes.io/hostname: talos-ysi-4k0}
```

**Why separate advertisements?**
- Different interface names per node type
- Prevents MetalLB from advertising on wrong interfaces (e.g., `cilium_vxlan`, NVLink)
- Ensures local network accessibility

### External Traffic Policy

Services use `externalTrafficPolicy: Cluster` (not `Local`) to ensure traffic can be routed through any node, even if the pod is only on one node.

## Gateway & Ingress

- **Gateway API:** kgateway (Envoy-based)
- **External Gateway:** `external-gateway.kgateway-system`
  - LoadBalancer IP: `192.168.4.2`
  - Handles HTTPS traffic with TLS termination
  - SNI-based routing via HTTPRoutes
- **Cert-Manager:** Automatic TLS certificates (Let's Encrypt)
- **External DNS:** Automatic DNS record management
- **Jaeger UI Route:** `https://jaeger.wcloud.sh` → `HTTPRoute/jaeger` (`jaeger/jaeger-collector:16686`)

### Testing Gateway Access

**From local network (Tailscale OFF):**
```bash
curl -k https://<service>.wcloud.sh
# Should work with externalTrafficPolicy: Cluster
```

**Via Tailscale:**
```bash
curl -k https://<service>.wcloud.sh
# Always works regardless of traffic policy
```

## SR-IOV & Secondary Networks (DGX Spark QSFP)

The DGX Spark nodes have Mellanox ConnectX-7 NICs connected via 200Gbps QSFP DAC cables. These are exposed to pods via Multus + SR-IOV.

- **Multus:** meta-CNI in `kube-system`, wraps Cilium and enables additional network interfaces
- **SR-IOV device plugin:** standalone DaemonSet in `kube-system` (`base/sriov-device-plugin.yaml`) creates VFs and advertises `nvidia.com/cx7_qsfp`
- **Configured interface:** `enp1s0f1np1` (onboard CX-7 port 1, 200Gbps)
- **VFs per node:** 4 (RDMA-enabled)
- **NetworkAttachmentDefinition:** `dgx-qsfp` in `default` namespace
- **IPAM:** `host-local`, subnet `10.100.0.0/24`

To use in a pod:
```yaml
annotations:
  k8s.v1.cni.cncf.io/networks: dgx-qsfp
resources:
  limits:
    nvidia.com/cx7_qsfp: 1
```

For full details, see [multus-sriov.md](./multus-sriov.md).

## Network Troubleshooting

### MetalLB IP not accessible on local network

- ✅ Verify L2Advertisements have correct interface selectors
- ✅ Check `externalTrafficPolicy` is `Cluster` not `Local`
- ✅ Ensure Tailscale is **disabled** for local network testing
- ✅ Test TCP connectivity: `nc -vn 192.168.4.2 443`

### TLS timeout issues with Gateway

- ✅ Verify `externalTrafficPolicy: Cluster` on LoadBalancer services
- ✅ Check if accessing from correct network (local vs Tailscale)
- ✅ Verify gateway pod is running: `kubectl get pods -n kgateway-system`
