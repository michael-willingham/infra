# Adding Helm Charts

**⚠️ CRITICAL: Always verify Helm repository URLs before adding new charts!**

## Step-by-Step Guide

### 1. Verify the Helm Repository

**DO NOT guess or assume repository URLs. Always verify:**

```bash
# Search Artifact Hub
# Visit: https://artifacthub.io

# For traditional Helm repos:
helm repo add <repo-name> <repo-url>
helm search repo <chart-name>

# For OCI registries:
helm show chart oci://<registry>/<chart> --version <chart-version>
```

### 2. Create Namespace (if needed)

See [Creating Namespaces](./creating-namespaces.md).

### 3. Create HelmRepository & HelmRelease

**For traditional Helm repos:**

```yaml
# operators/<name>.yaml (or the relevant stack directory)
---
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: <repo-name>
  namespace: <target-namespace>
spec:
  interval: 1m
  url: https://charts.example.com  # ⚠️ VERIFY THIS URL
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: <release-name>
  namespace: <target-namespace>
spec:
  interval: 10m
  chart:
    spec:
      chart: <chart-name>  # ⚠️ VERIFY chart name
      version: "<chart-version>"  # ⚠️ Use specific version, not "latest"
      sourceRef:
        kind: HelmRepository
        name: <repo-name>
        namespace: <target-namespace>
      interval: 10m
  values:
    # Chart-specific values here
```

**For OCI Helm charts:**

```yaml
# operators/<name>.yaml (or the relevant stack directory)
---
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: <repo-name>
  namespace: <target-namespace>
spec:
  interval: 12h
  type: oci
  url: oci://docker.io/<org>  # ⚠️ Base registry path, NOT full chart path
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: <release-name>
  namespace: <target-namespace>
spec:
  interval: 10m
  chart:
    spec:
      chart: <chart-name>
      version: "<chart-version>"
      sourceRef:
        kind: HelmRepository
        name: <repo-name>
        namespace: <target-namespace>
      interval: 10m
  values:
    # Chart-specific values
```

**⚠️ OCI Chart URL Pattern:**
- ❌ WRONG: `url: oci://docker.io/org/chart-name` (too specific)
- ✅ CORRECT: `url: oci://docker.io/org` (base path - Flux appends chart name)

### 4. Review Chart Values

Before committing:
1. Check the chart's `values.yaml` on GitHub or Artifact Hub
2. Understand what defaults you're accepting
3. Override critical values (resource limits, replicas, etc.)
4. Test locally: `helm template <name> <repo>/<chart> --values values.yaml`

### 5. Set Up Kustomization

Ensure the target directory (e.g., `operators/`) has a Kustomization in `clusters/willingham-k8s/`:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: <app-name>
  namespace: flux-system
spec:
  interval: 10m
  path: ./operators
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
  dependsOn:
    - name: namespaces  # ⚠️ Ensure dependencies make sense!
```

### 6. Add a VPA Resource (for application workloads)

If the new chart deploys a long-running application workload (not an infrastructure operator or ephemeral job runner), create a VPA resource so its CPU/memory requests are auto-tuned.

Add a `VerticalPodAutoscaler` to the appropriate file in `vpa/`:

```yaml
# vpa/<app>.yaml
---
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: <deployment-name>
  namespace: <target-namespace>
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment  # or StatefulSet
    name: <exact-deployment-name>  # must match the actual Deployment/StatefulSet name
  updatePolicy:
    updateMode: "InPlaceOrRecreate"
```

Then add the file to `vpa/kustomization.yaml`.

**Skip VPA for:**
- Infrastructure operators (cert-manager, metallb, etc.)
- Ephemeral/scale-to-zero workloads (ARC runners)
- Operator-managed StatefulSets where the operator controls resource allocation (e.g., Percona PostgreSQL instances)

**Tip:** After deploying, check the actual Deployment/StatefulSet name with `kubectl get deploy -n <ns>` since Helm charts may add prefixes.

### 7. Commit and Deploy

```bash
git add namespaces/ <stack-dir>/ vpa/
git commit -m "Add <chart-name> Helm chart"
git push
```

## Pre-Deployment Checklist

- [ ] Verified Helm repo URL is correct (tested with `helm repo add` or `helm show chart`)
- [ ] Checked chart version exists and is stable
- [ ] Reviewed default values and overridden critical settings
- [ ] Created namespace YAML in `namespaces/` if needed
- [ ] Added namespace to `namespaces/kustomization.yaml`
- [ ] Set proper `dependsOn` in Kustomization
- [ ] Chart name matches exactly (case-sensitive!)
- [ ] Using specific version, not "latest"
- [ ] Added VPA resource in `vpa/` if this is an application workload

## Post-Deployment Verification

```bash
# Check HelmRelease status
flux get helmreleases -A

# Check specific release
kubectl get helmrelease -n <namespace> <name>

# View Helm release
helm list -n <namespace>

# Check pods
kubectl get pods -n <namespace>
```

## Common Mistakes to Avoid

❌ **DON'T:**
- Guess Helm repository URLs
- Use `latest` or no version tag
- Copy URLs from outdated blog posts
- Forget to create the namespace
- Skip dependency chains
- Use HTTP URLs (use HTTPS)
- Include chart name in OCI registry base URL

✅ **DO:**
- Verify URLs on Artifact Hub or official docs
- Pin to specific, tested versions
- Check the chart's GitHub repo for correct URL
- Create namespaces before deploying
- Set up proper dependency order
- Test with `helm template` or `helm show` first
- For OCI charts, use base registry path only

## Troubleshooting

### HelmRelease failing to install

- ✅ Verify Helm repo URL is correct
- ✅ Check chart version exists: `helm search repo <chart>`
- ✅ Verify namespace exists
- ✅ Check chart values are valid
- ✅ Review HelmRelease events: `kubectl describe helmrelease -n <ns> <name>`

### 401 Unauthorized with OCI charts

- ✅ Check if URL is base registry path (e.g., `oci://docker.io/org`)
- ✅ Verify chart exists: `helm show chart oci://registry/org/chart --version <chart-version>`
- ✅ Ensure you're not duplicating the chart name in the URL

## External Resources

- **Artifact Hub:** https://artifacthub.io (for Helm charts)
- **FluxCD Helm Controller:** https://fluxcd.io/flux/components/helm/
