# Secrets Management

This cluster uses **External Secrets Operator (ESO)** integrated with **Infisical** for secrets management. Each namespace gets only the specific secrets it needs via targeted `ExternalSecret` resources.

## Architecture

```
Infisical Cloud (https://app.infisical.com)
    │
    │ Project: willingham-cloud-secrets
    │ Environment: prod
    │
    ↓ (Universal Auth)
    │
ClusterSecretStore: infisical
    │
    ↓ (referenced by per-namespace ExternalSecrets)
    │
ExternalSecret (per namespace, specific keys only)
    │
    ↓ (creates)
    │
Secret (per namespace, scoped to what the app needs)
```

## ClusterSecretStore

A single `ClusterSecretStore` named **`infisical`** connects to Infisical using Universal Auth credentials. Defined in `base/external-secrets.yaml` and managed by Flux.

- **Host:** `https://app.infisical.com`
- **Project:** `willingham-cloud-secrets`
- **Environment:** `prod`
- **Auth:** Universal Auth with credentials in `external-secrets/infisical-auth-credentials` (bootstrap secret, manually created)

### Bootstrap

The `infisical-auth-credentials` secret must be created before the ClusterSecretStore can authenticate. Use the Taskfile at the repo root:

```bash
# Requires: infisical CLI installed and logged in
task bootstrap:eso
```

This pulls `INFISICAL_MACHINE_CLIENT_ID` and `INFISICAL_MACHINE_CLIENT_SECRET` from the Infisical project, creates the `external-secrets` namespace if needed, and creates the secret.

## Per-Namespace ExternalSecrets

Each namespace that needs secrets defines its own `ExternalSecret` resource, pulling only the specific keys required.

### Current Secret Assignments

| Namespace | ExternalSecret | Secret Name | Keys | File |
|-----------|---------------|-------------|------|------|
| arc-runners | `arc-github-app` | `arc-github-app` | `github_app_id`, `github_app_installation_id`, `github_app_private_key` (from `GH_APP_*`) | `dev-platform/arc-runners.yaml` |
| argo | `argocd-oidc` | `argocd-oidc` | `oidc.clientSecret` (from `ARGOCD_OIDC_CLIENT_SECRET`) | `argo/argo-cd.yaml` |
| argo | `argo-workflows-oidc` | `argo-workflows-oidc` | `clientSecret` (from `ARGO_WORKFLOWS_OIDC_CLIENT_SECRET`) | `argo/argo-workflows.yaml` |
| camel-k | `camel-k-registry` | `camel-k-registry` | `token` (from `CAMEL_K_GHCR_TOKEN`) | `dev-platform/camel-k.yaml` |
| cert-manager | `cloudflare-dns-token` | `cloudflare-dns-token` | `api-token` (from CLOUDFLARE_DNS_API_TOKEN) | `base/cert-manager.yaml` |
| external-dns | `cloudflare-dns-token` | `cloudflare-dns-token` | `CLOUDFLARE_DNS_API_TOKEN` | `operators/external-dns.yaml` |
| headlamp | `headlamp-kgateway-oauth2` | `headlamp-kgateway-oauth2` | `client-secret` (from `HEADLAMP_OIDC_CLIENT_SECRET`) | `routes/headlamp-auth.yaml` |
| kafka | `kafbat-ui-oidc` | `kafbat-ui-oidc` | `clientSecret` (from `KAFBAT_UI_OIDC_CLIENT_SECRET`) | `messaging/kafbat-ui.yaml` |
| keycloak | `google-oauth` | `google-oauth` | `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET` | `security/wcloud-realm-import.yaml` |
| keycloak | `argo-oidc-clients` | `argo-oidc-clients` | `ARGOCD_OIDC_CLIENT_SECRET`, `ARGO_WORKFLOWS_OIDC_CLIENT_SECRET` | `security/wcloud-realm-import.yaml` |
| keycloak | `headlamp-oidc` | `headlamp-oidc` | `HEADLAMP_OIDC_CLIENT_SECRET` | `security/wcloud-realm-import.yaml` |
| keycloak | `grafana-oidc` | `grafana-oidc` | `GRAFANA_OIDC_CLIENT_SECRET` | `security/wcloud-realm-import.yaml` |
| tailscale | `operator-oauth` | `operator-oauth` | `client_id`, `client_secret` (from `TAILSCALE_*`) | `operators/tailscale.yaml` |
| temporal | `temporal-web-oidc` | `temporal-web-oidc` | `clientSecret` (from `TEMPORAL_UI_OIDC_CLIENT_SECRET`), templated into `TEMPORAL_AUTH_CLIENT_SECRET` | `dev-platform/temporal.yaml` |
| tfc-operator-system | `tfc-token` | `tfc-token` | `token` (from `TFC_API_TOKEN`) | `operators/tfc-operator.yaml` |
| vmks | `grafana-oidc` | `grafana-oidc` | `client-secret` (from `GRAFANA_OIDC_CLIENT_SECRET`) | `observability/vmks.yaml` |

### Secrets Available in Infisical

Keys currently referenced by this repository's manifests and Taskfile automation:

- `ARGOCD_OIDC_CLIENT_SECRET` — ArgoCD Keycloak OIDC client secret
- `ARGO_WORKFLOWS_OIDC_CLIENT_SECRET` — Argo Workflows Keycloak OIDC client secret
- `CAMEL_K_GHCR_TOKEN` — GHCR token for Camel K image builds (`camel-k-registry` dockerconfig secret)
- `CLOUDFLARE_DNS_API_TOKEN` — Cloudflare DNS API token
- `GH_APP_ID` — GitHub App ID
- `GH_APP_INSTALLATION_ID` — GitHub App installation ID
- `GH_APP_PRIVATE_KEY` — GitHub App private key
- `GRAFANA_OIDC_CLIENT_SECRET` — Grafana Keycloak OIDC client secret
- `HEADLAMP_OIDC_CLIENT_SECRET` — Headlamp Keycloak OIDC client secret (kgateway route auth + realm client)
- `GOOGLE_CLIENT_ID` — Google OAuth client ID
- `GOOGLE_CLIENT_SECRET` — Google OAuth client secret
- `INFISICAL_MACHINE_CLIENT_ID` — bootstrap credential used by `task bootstrap:eso`
- `INFISICAL_MACHINE_CLIENT_SECRET` — bootstrap credential used by `task bootstrap:eso`
- `KAFBAT_UI_OIDC_CLIENT_SECRET` — Kafbat UI Keycloak OIDC client secret
- `KC_ADMIN_PASSWORD` — Keycloak admin password used by `task keycloak:create-client`
- `TAILSCALE_CLIENT_ID` — Tailscale OAuth client ID
- `TAILSCALE_CLIENT_SECRET` — Tailscale OAuth client secret
- `TEMPORAL_UI_OIDC_CLIENT_SECRET` — Temporal UI Keycloak OIDC client secret
- `TFC_API_TOKEN` — Terraform Cloud API token

## Adding Secrets to a New Namespace

1. Add a new secret to Infisical if needed (project: `willingham-cloud-secrets`, env: `prod`)

2. Add an `ExternalSecret` in the same YAML file as the app that consumes it:

```yaml
---
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: my-app-secret
  namespace: my-namespace
spec:
  refreshInterval: 10m
  secretStoreRef:
    kind: ClusterSecretStore
    name: infisical
  target:
    name: my-app-secret
    creationPolicy: Owner
  data:
    - secretKey: local-key-name    # key name in the K8s Secret
      remoteRef:
        key: INFISICAL_KEY_NAME    # key name in Infisical
```

3. Reference the secret in your app config:

```yaml
env:
  - name: MY_VAR
    valueFrom:
      secretKeyRef:
        name: my-app-secret
        key: local-key-name
```

## Managing Secrets

### Force Sync

```bash
kubectl annotate externalsecret <name> -n <namespace> \
  force-sync=$(date +%s) --overwrite
```

### Check Status

```bash
# All ExternalSecrets
kubectl get externalsecret -A

# Specific namespace
kubectl get externalsecret -n <namespace>

# ClusterSecretStore health
kubectl get clustersecretstore infisical
```

### ESO Operator Logs

```bash
kubectl logs -n external-secrets -l app.kubernetes.io/name=external-secrets
```

## Security Notes

- The bootstrap secret `infisical-auth-credentials` in `external-secrets` namespace is manually created and not managed by ESO
- Each namespace only receives the specific secrets it needs — no blanket syncing
- Rotate Infisical Universal Auth credentials periodically via the Infisical dashboard
