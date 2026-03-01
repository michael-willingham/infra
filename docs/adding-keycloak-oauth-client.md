# Adding a Keycloak OAuth Client

This guide covers how to add OIDC/OAuth authentication for a new service using the cluster's Keycloak instance.

**Realm:** `wcloud`
**Issuer URL:** `https://keycloak.wcloud.sh/realms/wcloud`
**Identity Provider:** Google (all logins redirect through Google OAuth)

## Overview

Adding a new OAuth client involves three places:

1. **Keycloak** — register the client (via `task` CLI)
2. **Realm import** — optionally declare the client in `security/wcloud-realm-import.yaml` for GitOps reproducibility
3. **Service config** — create an ExternalSecret and wire the client secret into the service's Helm values

## Step 1: Create the Client in Keycloak

Use the Taskfile to create the client and store its secret in Infisical:

```bash
task keycloak:create-client -- <client-id> <redirect-uri> [web-origin]
```

Example:

```bash
task keycloak:create-client -- my-app https://my-app.wcloud.sh/callback https://my-app.wcloud.sh
```

This does four things:
1. Generates a random 32-byte client secret
2. Stores it in Infisical as `MY_APP_OIDC_CLIENT_SECRET` (uppercased, hyphens become underscores)
3. Creates a confidential OIDC client in the `wcloud` realm with standard settings
4. Adds a `groups` protocol mapper so tokens include group membership

**Prerequisites:** `infisical` CLI logged in, `kubectl` connected to the cluster, Keycloak pod running.

### Client defaults set by the task

| Setting | Value |
|---------|-------|
| Protocol | OpenID Connect |
| Public client | false (confidential) |
| Standard flow | enabled (Authorization Code) |
| Direct access grants | disabled |
| Service accounts | disabled |
| Default scopes | `openid`, `profile`, `email`, `groups` |

## Step 2: Add to Realm Import (Recommended)

For GitOps reproducibility, declare the client in `security/wcloud-realm-import.yaml`. This ensures the client is recreated if the realm is re-imported.

### 2a. Add an ExternalSecret in the keycloak namespace

Add this to the top of `security/wcloud-realm-import.yaml` (alongside the existing ExternalSecrets):

```yaml
---
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: my-app-oidc
  namespace: keycloak
spec:
  refreshInterval: 10m
  secretStoreRef:
    kind: ClusterSecretStore
    name: infisical
  target:
    name: my-app-oidc
    creationPolicy: Owner
  data:
    - secretKey: MY_APP_OIDC_CLIENT_SECRET
      remoteRef:
        key: MY_APP_OIDC_CLIENT_SECRET
```

### 2b. Add a placeholder reference

In the `spec.placeholders` section of the `KeycloakRealmImport`:

```yaml
  placeholders:
    # ... existing placeholders ...
    MY_APP_OIDC_CLIENT_SECRET:
      secret:
        name: my-app-oidc
        key: MY_APP_OIDC_CLIENT_SECRET
```

### 2c. Add the client definition

In the `spec.realm.clients` list:

```yaml
      - clientId: my-app
        name: My App
        enabled: true
        protocol: openid-connect
        publicClient: false
        standardFlowEnabled: true
        directAccessGrantsEnabled: false
        serviceAccountsEnabled: false
        secret: ${MY_APP_OIDC_CLIENT_SECRET}
        redirectUris:
          - "https://my-app.wcloud.sh/callback"
        webOrigins:
          - "https://my-app.wcloud.sh"
        defaultClientScopes:
          - openid
          - profile
          - email
```

## Step 3: Wire the Secret into the Service

In the service's YAML file (e.g., `operators/my-app.yaml` or `dev-platform/my-app.yaml`), add an ExternalSecret that pulls the client secret into the service's namespace:

```yaml
---
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: my-app-oidc
  namespace: my-app
spec:
  refreshInterval: 10m
  secretStoreRef:
    kind: ClusterSecretStore
    name: infisical
  target:
    name: my-app-oidc
    creationPolicy: Owner
  data:
    - secretKey: clientSecret
      remoteRef:
        key: MY_APP_OIDC_CLIENT_SECRET
```

Then reference it in the HelmRelease. The exact method depends on the Helm chart:

### Option A: `valuesFrom` (preferred, used by Headlamp)

```yaml
spec:
  valuesFrom:
    - kind: Secret
      name: my-app-oidc
      valuesKey: clientSecret
      targetPath: config.oidc.clientSecret
  values:
    config:
      oidc:
        clientID: my-app
        issuerURL: https://keycloak.wcloud.sh/realms/wcloud
        scopes: "openid,email,profile"
```

### Option B: Environment variables (used by Temporal)

```yaml
spec:
  values:
    env:
      - name: AUTH_CLIENT_ID
        value: my-app
      - name: AUTH_PROVIDER_URL
        value: https://keycloak.wcloud.sh/realms/wcloud
      - name: AUTH_CLIENT_SECRET
        valueFrom:
          secretKeyRef:
            name: my-app-oidc
            key: clientSecret
```

### Option C: Inline Helm values (used by Grafana)

Some charts accept OIDC config as nested values. Example from Grafana:

```yaml
spec:
  values:
    grafana:
      grafana.ini:
        auth.generic_oauth:
          enabled: true
          client_id: grafana
          scopes: openid profile email
          auth_url: https://keycloak.wcloud.sh/realms/wcloud/protocol/openid-connect/auth
          token_url: https://keycloak.wcloud.sh/realms/wcloud/protocol/openid-connect/token
          api_url: https://keycloak.wcloud.sh/realms/wcloud/protocol/openid-connect/userinfo
```

## Step 4: Update Documentation

1. Add the new secret to `docs/secrets-management.md` — both the "Current Secret Assignments" table and the "Secrets Available in Infisical" list.
2. Commit all changes to Git and let FluxCD reconcile.

## Common OIDC Endpoints

All under `https://keycloak.wcloud.sh/realms/wcloud`:

| Endpoint | Path |
|----------|------|
| Discovery | `/.well-known/openid-configuration` |
| Authorization | `/protocol/openid-connect/auth` |
| Token | `/protocol/openid-connect/token` |
| UserInfo | `/protocol/openid-connect/userinfo` |
| JWKS | `/protocol/openid-connect/certs` |
| Logout | `/protocol/openid-connect/logout` |

## Existing Clients (Reference)

| Client ID | Service | Redirect URI | File |
|-----------|---------|-------------|------|
| `argo-cd` | ArgoCD | `https://argo-cd.wcloud.sh/auth/callback` | `security/wcloud-realm-import.yaml`, `argo/argo-cd.yaml` |
| `argo-workflows` | Argo Workflows | `https://argo-workflows.wcloud.sh/oauth2/callback` | `security/wcloud-realm-import.yaml`, `argo/argo-workflows.yaml` |
| `headlamp` | Headlamp | `https://k8s.wcloud.sh/oidc-callback` | `security/wcloud-realm-import.yaml`, `observability/headlamp.yaml`, `routes/headlamp-auth.yaml` |
| `grafana` | Grafana | `https://grafana.wcloud.sh/login/generic_oauth` | `security/wcloud-realm-import.yaml`, `observability/vmks.yaml` |
| `rabbitmq` | RabbitMQ | `https://rabbitmq.wcloud.sh/js/oidc-oauth/login-callback.html` | `security/wcloud-realm-import.yaml`, `messaging/rabbitmq.yaml` |
| `temporal-ui` | Temporal UI | `https://temporal.wcloud.sh/auth/sso/callback` | `dev-platform/temporal.yaml` |
| `kafbat-ui` | Kafbat UI | `{baseUrl}/login/oauth2/code/{registrationId}` | `messaging/kafbat-ui.yaml` |

`temporal-ui` and `kafbat-ui` are wired in service configs but are not currently declared in `security/wcloud-realm-import.yaml`.

## Naming Conventions

| Item | Pattern | Example |
|------|---------|---------|
| Client ID | lowercase with hyphens | `my-app` |
| Infisical key | `{CLIENT_ID_UPPER_SNAKE}_OIDC_CLIENT_SECRET` | `MY_APP_OIDC_CLIENT_SECRET` |
| ExternalSecret name | `{client-id}-oidc` | `my-app-oidc` |
| K8s Secret name | `{client-id}-oidc` | `my-app-oidc` |
