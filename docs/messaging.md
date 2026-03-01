# Messaging

Components in the `messaging` Kustomization that provide messaging and event streaming infrastructure.

## Apache Kafka (Strimzi)

- **Purpose:** Distributed event streaming platform
- **Namespace:** `kafka`
- **Operator:** Strimzi (`operators/strimzi.yaml`)
- **Kafka Version:** pinned in `messaging/kafka.yaml`
- **Mode:** KRaft (no ZooKeeper)
- **Topology:** 3 combined controller+broker nodes (`KafkaNodePool` with both roles)
- **Storage:** 50Gi per node (Longhorn, JBOD)
- **Listeners:**
  - `plain` — port 9092 (internal, no TLS)
  - `tls` — port 9093 (internal, TLS)
- **Bootstrap:** `kafka-cluster-kafka-bootstrap.kafka.svc:9092`
- **Entity Operator:** Topic and User operators enabled
- **Replication:** factor 3, min ISR 2

## Kafbat UI

- **Purpose:** Web UI for Apache Kafka cluster management
- **Namespace:** `kafka`
- **Chart:** `kafka-ui` (from `https://ui.charts.kafbat.io`)
- **URL:** `https://kafka.wcloud.sh`
- **Auth:** Keycloak OIDC (`client_id: kafbat-ui`, scope `openid`)

## NATS

- **Purpose:** High-performance messaging system with persistent streaming
- **Namespace:** `nats`
- **Chart:** `nats` (from `https://nats-io.github.io/k8s/helm/charts/`)
- **Replicas:** 3 (clustered)
- **JetStream:** Enabled with 20Gi PVC per node (Longhorn)

## RabbitMQ

- **Purpose:** Message broker with routing, queuing, and pub/sub
- **Namespace:** `rabbitmq`
- **Operator:** RabbitMQ Cluster Operator (`operators/rabbitmq-cluster-operator.yaml`, OLM via OperatorHub)
- **Replicas:** 3
- **Storage:** 20Gi per node (Longhorn)
- **Management UI:** Port 15672
- **URL:** `https://rabbitmq.wcloud.sh`
- **Auth:** Keycloak OAuth2 via `rabbitmq_auth_backend_oauth2` plugin (public client, scopes: `rabbitmq.tag:administrator`, `rabbitmq.configure:*/*`, `rabbitmq.read:*/*`, `rabbitmq.write:*/*`). Internal auth (`rabbit_auth_backend_internal`) remains as fallback for the default user and AMQP clients.
- **Topology Spread:** Pods spread across nodes via `topologySpreadConstraints`
