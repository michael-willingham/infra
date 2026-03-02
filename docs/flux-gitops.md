# FluxCD & GitOps Workflow

**⚠️ All cluster resources are managed via FluxCD**

## How It Works

1. All configuration is stored in this Git repository
2. FluxCD watches the `master` branch
3. Changes committed to Git are automatically applied to the cluster
4. **DO NOT use `kubectl apply` directly** - changes will be overwritten by Flux

## Making Changes

```bash
# 1. Edit YAML files in this repository
vim operators/my-operator.yaml  # or observability/my-operator.yaml

# 2. Commit and push to master
git add .
git commit -m "Description of change"
git push

# 3. Flux automatically applies (1-2 minutes)
# OR force reconciliation:
flux reconcile kustomization <name> --with-source
```

## Checking Flux Status

```bash
# List all Kustomizations
flux get kustomizations

# Check HelmReleases
flux get helmreleases -A

# View logs
flux logs --all-namespaces --follow
```

## Kustomization Dependencies

Kustomizations in `clusters/willingham-k8s/` define the order:

```yaml
spec:
  dependsOn:
    - name: operators  # This Kustomization waits for 'operators' to be ready
```

**Dependency graph:**
```
flux-system ─→ namespaces ─→ crds ─┬─→ base ─→ operators ─┬─→ observability ─→ vpa
                                   │                      ├─→ messaging
                                   ├─→ routes              ├─→ network
                                                          ├─→ dev-platform
                                                          ├─→ serverless
                                                          ├─→ security
                                                          └─→ argo
```

- Namespaces must exist before anything else
- CRDs must be installed before operators that use them
- Operators must be ready before resources that depend on them
- Network (MetalLB, Gateway) should be ready before services that need LoadBalancers

## Repository Structure

```
k8s-config/
├── clusters/
│   └── willingham-k8s/        # FluxCD cluster configuration
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
├── base/                      # Foundational operators and cluster services
├── operators/                 # Core operators (Helm charts)
├── observability/             # Metrics, dashboards, tracing, and cluster UI stack
├── crds/                      # Custom Resource Definitions
├── network/                   # MetalLB + Gateway configs
├── messaging/                 # Kafka, NATS, RabbitMQ, and Kafbat UI
├── security/                  # Keycloak realm and auth resources
├── argo/                      # ArgoCD, Workflows, Events
├── dev-platform/              # Developer tooling and platform services (Coder, Temporal, ARC, TFC)
├── serverless/                # Knative Serving/Eventing and Gateway API resources
├── vpa/                       # Vertical Pod Autoscaler resources
└── routes/                    # Gateway API routes
```

## Troubleshooting

### Service changes not applying

- ✅ Check if resource is managed by Flux (has `kustomize.toolkit.fluxcd.io/` labels)
- ✅ Commit changes to Git instead of `kubectl apply`
- ✅ Force reconciliation: `flux reconcile kustomization <name>`

### Flux reconciliation failing

- ✅ Check Flux logs: `flux logs --all-namespaces --follow`
- ✅ Check Kustomization status: `flux get kustomizations`
- ✅ Check HelmRelease status: `flux get helmreleases -A`
- ✅ Verify dependencies are in correct order

## External Resources

- **FluxCD Docs:** https://fluxcd.io/flux/
