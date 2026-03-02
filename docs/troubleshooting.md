# Troubleshooting Guide

## Cluster Access

### Cannot connect to kubectl

- ✅ Ensure Tailscale is connected: `tailscale up`
- ✅ Check kubeconfig points to `k8s.solarflare-jazz.ts.net`
- ✅ Verify Tailscale is authenticating: `tailscale status`

### Node access needed

- ✅ Use `kubectl debug node/<name>` with `--image=nicolaka/netshoot`
- ✅ Add `--overrides='{"spec":{"hostNetwork":true}}'` for host network
- ✅ Use `-n kube-system` to bypass PodSecurity restrictions

## FluxCD Issues

### Service changes not applying

- ✅ Check if resource is managed by Flux (has `kustomize.toolkit.fluxcd.io/` labels)
- ✅ Commit changes to Git instead of `kubectl apply`
- ✅ Force reconciliation: `flux reconcile kustomization <name>`

### Flux reconciliation failing

- ✅ Check Flux logs: `flux logs --all-namespaces --follow`
- ✅ Check Kustomization status: `flux get kustomizations`
- ✅ Check HelmRelease status: `flux get helmreleases -A`
- ✅ Verify dependencies are in correct order

### All Kustomizations blocked by `flux-system` not ready

**Symptom:** `flux get kustomizations -A` shows most entries blocked on `dependency 'flux-system/flux-system' is not ready`

**Common causes:**
- `kustomize-controller` crash-looping due to an unsupported startup argument (for example `--allow-remote-bases=true`)
- The root `flux-system` Kustomization has `spec.wait: true`, which causes it to wait on child Kustomizations that themselves depend on `flux-system` (dependency loop)

**Solution:**
- Keep `clusters/willingham-k8s/flux-system/kustomization.yaml` minimal (`gotk-components.yaml` and `gotk-sync.yaml` only)
- Remove bootstrap patches that mutate `Kustomization/flux-system` with `spec.wait: true`
- Remove unsupported `kustomize-controller` args if present
- Commit and push so Flux can reconcile a healthy `kustomize-controller` deployment

**Verification:**
```bash
kubectl logs -n flux-system deploy/kustomize-controller --all-pods=true --tail=100
flux reconcile kustomization flux-system -n flux-system --with-source
flux get kustomizations -A
```

### HelmRelease failing to install

- ✅ Verify Helm repo URL is correct
- ✅ Check chart version exists: `helm search repo <chart>`
- ✅ Verify namespace exists
- ✅ Check chart values are valid
- ✅ Review HelmRelease events: `kubectl describe helmrelease -n <ns> <name>`

### OpenTelemetryCollector fails with `no matches for kind`

**Symptom:** Flux reports `no matches for kind "OpenTelemetryCollector"` while reconciling `observability`.

**Cause:** The `opentelemetry-operator` CRDs are not installed/ready yet (or the operator HelmRelease is failing), so Kubernetes does not recognize `OpenTelemetryCollector`.

**Solution:**
- Reconcile `Kustomization/operators` first (the operator is deployed from `operators/opentelemetry-operator.yaml`).
- Wait for `HelmRelease/opentelemetry-operator` in `otel-system` to become Ready.
- Ensure `OpenTelemetryCollector.spec.config` is a structured map (not a YAML string block).
- Ensure `OpenTelemetryCollector.spec.ports[].name` is 15 characters or fewer.
- Reconcile `Kustomization/observability` after the CRD exists.

**Verification:**
```bash
flux get helmreleases -A | grep opentelemetry-operator
kubectl get crd opentelemetrycollectors.opentelemetry.io
flux reconcile kustomization observability -n flux-system --with-source
kubectl get opentelemetrycollector -n jaeger
```

### Temporal HelmRelease fails with frontend deployment progress deadline

**Symptom:** `HelmRelease/temporal` reports `Deployment/temporal-frontend status: 'Failed'` and Flux marks `dev-platform` not ready.

**Cause:** PgBouncer query queue saturation (`query_wait_timeout`) can cause Temporal services to lose DB connectivity and crash-loop during rollout.

**Solution:**
- Increase PgBouncer headroom in `dev-platform/temporal.yaml` under `temporal-db` values:
  - `proxy.pgBouncer.config.global.default_pool_size`
  - `proxy.pgBouncer.config.global.reserve_pool_size`
  - `proxy.pgBouncer.config.global.query_wait_timeout`
- Reconcile `HelmRelease/temporal-db`, then `HelmRelease/temporal`.

**Verification:**
```bash
kubectl -n temporal logs deploy/temporal-db-pg-db-pgbouncer -c pgbouncer --tail=200 | grep query_wait_timeout
flux reconcile helmrelease temporal-db -n temporal
flux reconcile helmrelease temporal -n temporal
flux get helmreleases -A | grep temporal
```

### Headlamp asks for a service-account token or loops after Keycloak login

**Symptom:**
- Headlamp shows "Please paste your authentication token"
- Or Keycloak login succeeds but returns to login / "Lost connection to cluster" with 401s

**Cause:**
- Upstream Headlamp in-cluster OIDC still requires Kubernetes API authentication support for the OIDC issuer.
- If kube-apiserver does not trust that OIDC issuer, Headlamp cluster API calls fail.
- Plain in-cluster mode also prompts for a token unless Headlamp has a kubeconfig token source.
- If an `Authorization: Bearer ...` header is forwarded to Headlamp, Headlamp will try that token against kube-apiserver; non-Kubernetes OIDC tokens return `401 Unauthorized` and the UI falls back to token login.

**Fix used in this repo:**
- `observability/headlamp.yaml` mounts a kubeconfig (`ConfigMap/headlamp-kubeconfig`) that points to:
  - `tokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token`
  - `certificate-authority: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt`
- Headlamp runs with `-kubeconfig=/etc/headlamp/kubeconfig/config` and `clusterRoleBinding.create: true`.
- `routes/headlamp-auth.yaml` applies native `kgateway` OAuth2 protection:
  - `GatewayExtension/headlamp-keycloak-oauth2`
    - `forwardAccessToken: false`
  - `TrafficPolicy/headlamp-oauth2` targeting `HTTPRoute/headlamp`
    - `headerModifiers.request.remove: ["Authorization"]` to force service-account kubeconfig auth on backend API calls
  - `ExternalSecret/headlamp-kgateway-oauth2` (`client-secret` key)
  - `ReferenceGrant` to allow cross-namespace reference to `Service/keycloak-service`

**Verification:**
```bash
kubectl get deploy -n headlamp headlamp -o jsonpath='{.spec.template.spec.containers[0].args}{"\n"}'
kubectl get cm -n headlamp headlamp-kubeconfig -o yaml
kubectl get gatewayextension -n headlamp headlamp-keycloak-oauth2 -o yaml
kubectl get trafficpolicy -n headlamp headlamp-oauth2 -o yaml

# Optional runtime check (no Authorization header):
kubectl port-forward -n headlamp svc/headlamp 14466:80
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:14466/clusters/main/api

# Through gateway, expect redirect to Keycloak when not authenticated:
curl -I https://k8s.wcloud.sh
```

**Security note:**
- Headlamp backend access is still service-account based.
- Route access is enforced at the gateway via Keycloak OAuth2.
- In this cluster, `KeycloakRealmImport` is treated as bootstrap/disaster-recovery and is not relied on for routine realm/client updates.

### Temporal stuck `InProgress` / Flux deadlock on `dev-platform`

**Symptom:**
- `Kustomization/flux-system/dev-platform` stays `Reconciliation in progress` or fails on `HelmRelease/temporal`
- `HelmRelease/temporal` repeatedly times out in upgrade
- Temporal logs show DB errors like `too many clients already`, `query_wait_timeout`, or prepared-statement related failures

**Cause:**
- Temporal's default rollout + DB concurrency can overload Postgres/PgBouncer on this cluster.
- PgBouncer `transaction` mode is a poor fit for Temporal's SQL access pattern.
- Manual `kubectl patch` can leave `HelmRelease/temporal.spec.suspend: true`; if Git does not explicitly set `suspend: false`, Flux will not always clear it.
- A 5m Helm action timeout is too short for occasional Temporal frontend rollout stalls on this hardware.

**Fix (GitOps):**
- Use PgBouncer `session` mode for Temporal DB:
  - `proxy.pgBouncer.config.global.pool_mode: session`
  - `proxy.pgBouncer.config.global.default_pool_size: "10"`
  - `proxy.pgBouncer.config.global.reserve_pool_size: "3"`
  - `proxy.pgBouncer.config.global.max_prepared_statements: "0"`
- Keep Temporal DB fan-out low:
  - `server.replicaCount: 2` (HA without overloading DB at startup)
  - `web.replicaCount: 2`
  - `maxConns/maxIdleConns: 2` for default + visibility datastores
- Make Temporal reconciliation self-healing:
  - Set `spec.suspend: false` on `HelmRelease/temporal`
  - Set `spec.timeout: 15m` on `HelmRelease/temporal`
  - Set `server.frontend.readinessProbe.timeoutSeconds: 5`
  - Use Helm `postRenderers.kustomize.patches` to raise server deployment liveness `initialDelaySeconds` to `300` for `temporal-frontend`, `temporal-history`, and `temporal-matching`

**Break-glass (live cluster):**
```bash
# Unsuspend/reconcile Temporal and dev-platform
kubectl patch helmrelease temporal -n temporal --type merge -p '{"spec":{"suspend":false}}'
flux reconcile helmrelease temporal -n temporal --with-source
kubectl patch kustomization dev-platform -n flux-system --type merge -p '{"spec":{"suspend":false}}'
flux reconcile kustomization dev-platform -n flux-system
```

### 401 Unauthorized with OCI Helm charts

**Symptom:** HelmRelease fails with 401 error when pulling OCI chart

**Cause:** HelmRepository URL includes the chart name, causing Flux to double it in the path

**Solution:**
```yaml
# ❌ WRONG
url: oci://docker.io/org/chart-name

# ✅ CORRECT  
url: oci://docker.io/org  # Base path only
```

**Verification:**
```bash
# Test the chart can be pulled
helm show chart oci://docker.io/org/chart-name --version <chart-version>
```

## Pod & Namespace Issues

### Pods failing with PodSecurity violations

- ✅ Check namespace has privileged Pod Security labels
- ✅ See [Creating Namespaces](./creating-namespaces.md)
- ✅ Most operators need privileged labels on Talos

**Common errors:**
- `violates PodSecurity "baseline:latest": host namespaces (hostNetwork=true)`
- `violates PodSecurity "baseline:latest": privileged`
- Pods stuck in `CreateContainerConfigError`

**Fix:** Add privileged labels to namespace:
```yaml
labels:
  pod-security.kubernetes.io/audit: privileged
  pod-security.kubernetes.io/enforce: privileged
  pod-security.kubernetes.io/warn: privileged
```

## Networking Issues

### MetalLB IP not accessible on local network

- ✅ Verify L2Advertisements have correct interface selectors
- ✅ Check `externalTrafficPolicy` is `Cluster` not `Local`
- ✅ Ensure Tailscale is **disabled** for local network testing
- ✅ Test TCP connectivity: `nc -vn 192.168.4.2 443`

### TLS timeout issues with Gateway

- ✅ Verify `externalTrafficPolicy: Cluster` on LoadBalancer services
- ✅ Check if accessing from correct network (local vs Tailscale)
- ✅ Verify gateway pod is running: `kubectl get pods -n kgateway-system`

### Service not receiving traffic

- ✅ Check MetalLB assigned an IP: `kubectl get svc -A | grep LoadBalancer`
- ✅ Verify L2Advertisement interface matches node's physical interface
- ✅ Check service selector matches pod labels
- ✅ Test from inside cluster: `kubectl run -it --rm debug --image=nicolaka/netshoot`

## Storage Issues

### PVC stuck in Pending

- ✅ Check Longhorn status: `kubectl get pods -n longhorn-system`
- ✅ Verify storage class exists: `kubectl get storageclass`
- ✅ Check node disk space: Use `kubectl debug node/<name>`

### Longhorn still shows large "Reserved" capacity after Git change

**Symptom:** Longhorn UI still shows high reserved capacity (for example ~800 GiB total) after changing Helm values.

**Cause:** `defaultSettings.storageReservedPercentageForDefaultDisk` only applies to new Longhorn nodes/disks. Existing node disk objects keep their current `spec.disks.*.storageReserved` value.

**Verification:**
```bash
# Cluster-wide default
kubectl -n longhorn-system get settings.longhorn.io storage-reserved-percentage-for-default-disk -o jsonpath='{.value}{"\n"}'

# Actual per-node reserved bytes that Longhorn is using
kubectl -n longhorn-system get nodes.longhorn.io -o json | jq -r '.items[] | .metadata.name as $n | .spec.disks | to_entries[] | [$n, .key, .value.storageReserved] | @tsv'
```

**Solution:**
- Set `defaultSettings.storageReservedPercentageForDefaultDisk` in `base/longhorn.yaml` for future nodes/disks.
- Apply a one-time patch to existing Longhorn node disks (via UI or `kubectl patch`) so current nodes pick up the lower reserve value.

### Longhorn Instance Manager CPU requests are too high

**Symptom:** `instance-manager-*` pods request multiple CPU cores each, which inflates cluster CPU reservations.

**Cause:** Longhorn setting `guaranteed-instance-manager-cpu` is a percentage of node allocatable CPU per Instance Manager pod. Aggressive defaults can reserve several cores per pod on larger nodes.

**Verification:**
```bash
# Current setting value
kubectl -n longhorn-system get settings.longhorn.io guaranteed-instance-manager-cpu -o jsonpath='{.value}{"\n"}'

# Current usage for instance-manager pods
kubectl top pods -n longhorn-system --containers | grep instance-manager

# Current CPU requests on instance-manager pods
kubectl -n longhorn-system get pods -o json | jq -r '.items[] | select(.metadata.name|startswith("instance-manager-")) | .metadata.name as $n | .spec.containers[] | [$n, (.resources.requests.cpu // "<none>")] | @tsv'
```

**Solution:**
- Set `defaultSettings.guaranteedInstanceManagerCPU` in `base/longhorn.yaml` to a lower percentage.
- Commit and push so Flux applies the HelmRelease.
- Keep the setting aligned with your active data-engine CPU mask so reserved CPU stays around your target headroom.
- If Longhorn rebuild/IO workloads need more reserved CPU, increase this value incrementally (for example to `6-8`) and re-check pod pressure.

## Best Practices to Avoid Issues

1. **Always commit changes to Git** - Let Flux apply them
2. **Use Tailscale for cluster access** - API server requires it
3. **Create namespaces before workloads** - With privileged labels for Talos
4. **Verify Helm chart repositories** - Test URLs with `helm` CLI before adding
5. **Pin Helm chart versions** - Never use "latest" in production
6. **Set Kustomization dependencies correctly** - e.g. namespaces → crds → {routes, base} → operators → observability → vpa → apps (non-linear graph; ensure `dependsOn` matches `clusters/willingham-k8s/*.yaml`)
7. **Check Flux status after changes** - `flux get kustomizations`
8. **Test on local network with Tailscale OFF** - Avoid routing confusion
9. **Use proper interface selectors in MetalLB** - Critical for L2 announcements
10. **Prefer `externalTrafficPolicy: Cluster`** - Unless you need source IP preservation

## Getting Help

- Check existing issues in related operator documentation
- Review FluxCD logs for detailed error messages
- Use `kubectl describe` for resource events
- Check Talos documentation for platform-specific issues
