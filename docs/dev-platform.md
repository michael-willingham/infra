# Dev Platform

Components in the `dev-platform` Kustomization that provide developer tooling and CI/CD infrastructure.

## Coder

- **Purpose:** Cloud development environments (CDEs) platform
- **Namespace:** `coder`
- **Chart:** `coder` (pinned in `dev-platform/coder.yaml`)
- **Replicas:** 1 (HA requires enterprise license)
- **Database:** Percona PostgreSQL (`coder-db` in `coder` namespace)
- **URL:** `https://coder.wcloud.sh`
- **Auth:** GitHub OAuth (default provider)

## Temporal

- **Purpose:** Workflow orchestration platform
- **Namespace:** `temporal`
- **Chart:** `temporal` (from `https://go.temporal.io/helm-charts/`, pinned in `dev-platform/temporal.yaml`)
- **Database:** Percona PostgreSQL (`temporal-db` in `temporal` namespace)
- **Schema Management:** Managed by Temporal chart (`manageSchema: true`, `createDatabase: false`)
- **Replicas:** server components `2`, web UI `2`
- **Auth:** Keycloak OIDC (`client_id: temporal-ui`, scopes `openid profile email`)
- **Authorization:** No Temporal UI RBAC policy configured (authenticated users have full UI access)
- **VPA:** Enabled for `temporal-frontend`, `temporal-history`, `temporal-matching`, `temporal-worker`, and `temporal-web`
- **Web UI URL:** `https://temporal.wcloud.sh`

## GitHub Actions Runners (ARC)

- **Purpose:** Self-hosted GitHub Actions runners
- **Namespace:** `arc-runners`
- **Chart:** `gha-runner-scale-set` (from ARC controller)
- **Variants:**
  - `arc-runner-amd64` — AMD64 runners (0-4 replicas)
  - `arc-runner-arm64` — ARM64 runners (0-4 replicas)
- **Auth:** GitHub App credentials via ExternalSecret

## Apache Camel K & Kaoto

- **Purpose:** Cloud-native integration framework (Apache Camel K) with a visual low-code integration designer (Kaoto)
- **Namespace:** `camel-k`
- **Operator:** Apache Camel K operator (HelmRelease `operators/camel-k.yaml`, chart/version pinned there)
- **Designer:** Kaoto, deployed via `Kaoto` CR (managed by Kaoto operator from OLM)
- **Image Registry:** `ghcr.io/willingham-cloud` (credentials via `ExternalSecret/camel-k-registry`)
- **Kaoto URL:** `https://camel.wcloud.sh`

## Armada

- **Purpose:** Multi-cluster workload orchestration platform
- **Namespace:** `armada-system`
- **Chart:** `armada-operator` (from `https://g-research.github.io/charts/`)
- **Deployed from:** `dev-platform/armada.yaml`

## HCP Terraform Agents

- **Purpose:** Self-hosted HCP Terraform agents for remote runs
- **Namespace:** `tfc-operator-system`
- **Resource:** `AgentPool` (managed by HCP Terraform Operator)
- **Agent Image:** pinned in `dev-platform/tfc-agents.yaml`
- **Variants:**
  - `tfc-agent-amd64` — AMD64 agents (min 3, max 10)
  - `tfc-agent-arm64` — ARM64 agents (min 3, max 10)
- **Auth:** TFC API token via `ExternalSecret` (`tfc-token`)
