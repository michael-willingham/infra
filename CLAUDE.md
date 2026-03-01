# Claude Code Instructions

See [AGENTS.md](AGENTS.md) for full project documentation and cluster details.

## Key Rules

- **Ask before running destructive kubectl/helm/k8s commands** — confirm the right kubecontext first.
- **All changes go through Git** — never `kubectl apply` directly. Edit YAML, commit, push, and let FluxCD reconcile.
- **Update docs when you change infrastructure** — if you add/remove directories, kustomizations, operators, namespaces, or network config, update `AGENTS.md` and relevant files in `docs/`.
- **Prefer maintainable docs** — do not hardcode concrete version numbers in Markdown docs; keep versions in YAML manifests.

## Cluster Access

- Requires **Tailscale** connected (`tailscale up`)
- API server: `https://k8s.solarflare-jazz.ts.net`
- **Talos Linux** — no SSH. Use `kubectl debug node/<name>` if needed.

## Repository Layout

```
clusters/willingham-k8s/   # FluxCD Kustomization entry points
namespaces/                # Namespace definitions (need privileged PodSecurity labels)
base/                      # Foundational operators (storage, certs, LB, secrets)
operators/                 # HelmRepository + HelmRelease for infrastructure
observability/             # Metrics, tracing, and cluster UI (VMKS, Jaeger, Headlamp)
crds/                      # Custom Resource Definitions
network/                   # MetalLB, Gateway API configs
messaging/                 # Messaging infrastructure (Kafka, NATS, RabbitMQ)
security/                  # Keycloak
argo/                      # ArgoCD, Workflows, Events
dev-platform/              # Coder, Temporal, ARC runners, TFC agents, Armada
serverless/                # Knative Serving/Eventing + Gateway API integration
vpa/                       # VPA resources for application workloads
routes/                    # Gateway API routes
docs/                      # Detailed documentation
```

## FluxCD Dependency Chain

```
namespaces ─┬─→ crds ─┬─→ base ─┬─→ operators ─┬─→ observability ─→ vpa
            │         │         │              ├─→ messaging
            │         │         │              ├─→ network
            │         │         │              ├─→ dev-platform
            │         │         │              ├─→ serverless
            │         │         │              └─→ security ─→ argo
            │         └─→ routes
```

## Common Patterns

### Namespaces must have privileged PodSecurity labels (Talos requirement)

```yaml
labels:
  pod-security.kubernetes.io/audit: privileged
  pod-security.kubernetes.io/enforce: privileged
  pod-security.kubernetes.io/warn: privileged
```

### OCI Helm repos — use base registry path only

```yaml
# Flux appends the chart name automatically
url: oci://docker.io/org  # not oci://docker.io/org/chart-name
```

### Checking status

```bash
flux get kustomizations
flux get helmreleases -A
flux logs --all-namespaces --follow
```
