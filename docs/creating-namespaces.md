# Creating Namespaces

**⚠️ CRITICAL: Always create namespace definitions in the `namespaces/` directory!**

## Step-by-Step

### 1. Create a YAML file in `namespaces/`

```yaml
# namespaces/<namespace-name>.yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: <namespace-name>
  labels:
    pod-security.kubernetes.io/audit: privileged
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/warn: privileged
```

### 2. Add to `namespaces/kustomization.yaml`

```yaml
resources:
  # ... existing namespaces ...
  - <namespace-name>.yaml
```

### 3. Commit to Git

```bash
git add namespaces/
git commit -m "Add <namespace-name> namespace"
git push
```

## Why Privileged Labels?

**Talos Linux requires these Pod Security Standards labels for most operators to function properly.**

Without these labels, you'll encounter errors like:
- `violates PodSecurity "baseline:latest": host namespaces (hostNetwork=true)`
- `violates PodSecurity "baseline:latest": privileged`
- Pods stuck in `CreateContainerConfigError`

### When to use privileged

- System operators (storage, networking, monitoring)
- Anything that needs hostNetwork, hostPID, or hostIPC
- Anything that mounts host paths
- Most Helm charts for infrastructure components

**Default to privileged unless you have a specific reason not to.** It's better to start permissive and restrict later.

## Namespace Checklist

- [ ] Created YAML file in `namespaces/<name>.yaml`
- [ ] Added privileged Pod Security labels
- [ ] Added to `namespaces/kustomization.yaml`
- [ ] Committed to Git
- [ ] Namespace created BEFORE deploying workloads to it

## Troubleshooting

### Pods failing with PodSecurity violations

- ✅ Check namespace has privileged Pod Security labels
- ✅ Most operators need privileged labels on Talos
- ✅ Add the labels and redeploy the namespace
