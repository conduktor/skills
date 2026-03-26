# Conduktor mental model

## 1. Product boundary

**Console** -- `conduktor/conduktor-console`, port 8080, requires PostgreSQL.
Provides: RBAC, topic catalog, monitoring, self-service framework, data quality UI.
Console resources: `apiVersion: v2` (or `apiVersion: kafka/v2` for Topics).

**Gateway** -- `conduktor/conduktor-gateway`, port 6969.
Transparent Kafka proxy. Handles: interceptors, virtual clusters, encryption, masking, traffic control.
Gateway resources: `apiVersion: gateway/v2`.

**CLI** -- `conduktor` binary (Go). kubectl-style verbs: `apply`, `get`, `delete`, `edit`, `template`.
Targets Console via `CDK_BASE_URL` or Gateway via `CDK_GATEWAY_BASE_URL`.

Console and Gateway are **separate services**. Console can manage Gateway resources, but they deploy independently.

## 2. Gateway resource model

All Gateway resources: `apiVersion: gateway/v2`. 6 resource kinds:

| Kind | Purpose |
|---|---|
| `Interceptor` | Plugin that intercepts/modifies Kafka requests and responses |
| `VirtualCluster` | Logical namespace isolating service accounts and resources |
| `GatewayServiceAccount` | Identity for client authentication (LOCAL or EXTERNAL) |
| `GatewayGroup` | Groups service accounts for Interceptor targeting |
| `AliasTopic` | Maps a physical Kafka topic as a logical topic in a vCluster |
| `ConcentrationRule` | Routes virtual topic creation into shared physical topics |

### Interceptors

- Execute in **priority order** (lowest number first). `spec.pluginClass` (mandatory): `io.conduktor.gateway.interceptor.*`. `spec.priority` (mandatory). `spec.config`: plugin-specific.
- **Scope precedence** (highest to lowest): ServiceAccount > Group > VirtualCluster > Global.
- Same `metadata.name` with different scopes = override per precedence.

#### Targeting matrix

| Use case | `scope.vCluster` | `scope.group` | `scope.username` |
|---|---|---|---|
| Global (including vClusters) | `null` | `null` | `null` |
| Global (excluding vClusters) | omitted | omitted | omitted |
| Username targeting | omitted | omitted | set |
| Group targeting | omitted | set | omitted |
| VirtualCluster targeting | set | omitted | omitted |
| VirtualCluster + Username | set | omitted | set |
| VirtualCluster + Group | set | set | omitted |

Critical: `scope.vCluster: null` = global **including** vClusters. Omitted scope = global **excluding** vClusters.

```yaml
apiVersion: gateway/v2
kind: Interceptor
metadata:
  name: enforce-partition-limit
  scope:
    vCluster: null  # null = applies everywhere including vClusters
spec:
  pluginClass: "io.conduktor.gateway.interceptor.safeguard.CreateTopicPolicyPlugin"
  priority: 100
  config:
    topic: "myprefix-.*"
    numPartition: { min: 5, max: 5, action: "INFO" }
```

### GatewayServiceAccount

- `EXTERNAL`: `spec.externalNames` required, non-empty list (currently max 1), unique across all GatewayServiceAccounts.
- `LOCAL`: enables `/gateway/v2/tokens` endpoint for credential generation with configurable TTL.
- Stored in internal topic `_conduktor_${GATEWAY_CLUSTER_ID}_usermappings`.
- `metadata.name` = friendly name (used in Interceptor scopes, ACLs, audit logs). `spec.externalNames` = provider identity (mapping only).

```yaml
apiVersion: gateway/v2
kind: GatewayServiceAccount
metadata:
  name: application1
spec:
  type: EXTERNAL
  externalNames: [00u9vme99nxudvxZA0h7]
---
apiVersion: gateway/v2
kind: GatewayServiceAccount
metadata:
  vCluster: vc-B
  name: admin
spec:
  type: LOCAL
```

### GatewayGroup

Groups service accounts for Interceptor targeting only. **Cannot** be used for ACL management.

```yaml
apiVersion: gateway/v2
kind: GatewayGroup
metadata:
  name: group-a
spec:
  members:
    - name: admin
    - { vCluster: vc-B, name: "0000-AAAA-BBBB-CCCC" }
```

### AliasTopic

Maps a physical topic into a vCluster: `metadata: { name: alias, vCluster: vc }`, `spec: { physicalName: real-topic }`.

### ConcentrationRule

Routes virtual topic creation into shared physical topics based on `cleanup.policy`. `autoManaged: true` = auto-create physical topics. `offsetCorrectness: true` = proper lag tracking. Spec changes do NOT affect previously created concentrated topics.

```yaml
apiVersion: gateway/v2
kind: ConcentrationRule
metadata:
  name: concentration1
spec:
  pattern: titi-.*
  physicalTopics: { delete: titi-delete, compact: titi-compact, deleteCompact: titi-cd }
  autoManaged: false
  offsetCorrectness: false
```

## 3. Virtual clusters

A Virtual Cluster is a **logical namespace** inside Gateway, NOT a separate Kafka cluster. Topics and consumer groups are prefixed on the physical cluster and visible only within that vCluster. `metadata.name` must be a valid topic prefix.

- `spec.type`: `Standard` (default) or `Partner`.
- `spec.aclEnabled`: default `false`. When false, no authorization checks.

```yaml
apiVersion: gateway/v2
kind: VirtualCluster
metadata:
  name: "mon-app-A"
spec:
  aclEnabled: true
  aclMode: REST_API
  acls:
    - resourcePattern: { resourceType: TOPIC, name: customers, patternType: LITERAL }
      principal: User:username1
      host: "*"
      operation: READ
      permissionType: ALLOW
```

### ACL modes (immutable after creation)

| Mode | Behavior | Required | Forbidden |
|---|---|---|---|
| `KAFKA_API` | Cumulative. Managed via Kafka Admin API by superUsers. | `superUsers` | `acls` |
| `REST_API` | Idempotent. Full ACL list declared in YAML. | `acls` | `superUsers` |

`aclMode` cannot change after creation: `KAFKA_API` is cumulative, `REST_API` is idempotent.

### Service accounts and authentication

| SA Type | Auth mode | Token endpoint |
|---|---|---|
| `LOCAL` | Gateway-managed only (SASL) | `/gateway/v2/tokens` |
| `EXTERNAL` | Gateway-managed (mTLS/OAuth) or Kafka-managed | N/A |

| Security Mode | Virtual Clusters | Local SA | External SA |
|---|---|---|---|
| `GATEWAY_MANAGED` | Yes | Yes (SASL only) | Yes (mTLS or OAuth) |
| `KAFKA_MANAGED` | No | No | Yes |

Set via `GATEWAY_SECURITY_MODE` env var. Authentication methods: SASL (PLAIN, SCRAM, OAUTHBEARER), mTLS, anonymous.

## 4. Self-service framework

All self-service resources: `apiVersion: self-serve/v1`.

### Hierarchy

```
Application
  └── ApplicationInstance (binds: cluster + serviceAccount + resources + policies)
        ├── ApplicationInstancePermission (cross-team topic sharing)
        └── ApplicationGroup (Console RBAC for app members)
```

### Application

Umbrella for multiple deployments. `spec.owner` must be a valid Console group. Cannot delete if ApplicationInstances exist.

```yaml
apiVersion: self-serve/v1
kind: Application
metadata:
  name: "clickstream-app"
spec:
  title: "Clickstream App"
  owner: "groupA"            # Console Group technical-id
```

### ApplicationInstance

Core concept: **ties everything together** -- cluster, service account, resource ownership, policies.

```yaml
apiVersion: self-serve/v1
kind: ApplicationInstance
metadata:
  application: "clickstream-app"
  name: "clickstream-dev"
spec:
  cluster: "shadow-it"            # immutable
  serviceAccount: "sa-clicko"     # unique per cluster
  policyRef: ["generic-dev-topic"]  # links to ResourcePolicy
  defaultCatalogVisibility: PUBLIC
  resources:
    - { type: TOPIC, patternType: PREFIXED, name: "click." }
    - { type: CONSUMER_GROUP, patternType: PREFIXED, name: "click." }
    - { type: SUBJECT, patternType: PREFIXED, name: "click." }
    - { type: CONNECTOR, connectCluster: shadow-connect, patternType: PREFIXED, name: "click." }
```

- **Resource types**: `TOPIC`, `CONSUMER_GROUP`, `SUBJECT`, `CONNECTOR` (`connectCluster` required for CONNECTOR).
- **Pattern types**: `PREFIXED` or `LITERAL`.
- **Ownership modes**: `ALL` (default) or `LIMITED` (no create/update/delete via CLI/UI).
- Resource names must not overlap with other ApplicationInstances on the same cluster.
- Kafka side effects: SA gets ACLs -- Topic: `READ`, `WRITE`, `DESCRIBE_CONFIGS`; ConsumerGroup: `READ`.

### ResourcePolicy

CEL-expression-based policy enforcement. Replaces the legacy TopicPolicy. `spec.targetKind`: `Topic`, `Connector`, `Subject`, or `ApplicationGroup`. Linked via `spec.policyRef` on ApplicationInstance. Rules use `condition` (CEL expr) + `errorMessage`.

```yaml
apiVersion: self-serve/v1
kind: ResourcePolicy
metadata:
  name: "generic-dev-topic"
  labels:
    business-unit: delivery
spec:
  targetKind: Topic
  description: A policy for topic creation standards
  rules:
    - condition: "spec.replicationFactor == 3"
      errorMessage: "replication factor should be 3"
    - condition: "int(string(spec.configs[\"retention.ms\"])) >= 60000 && int(string(spec.configs[\"retention.ms\"])) <= 3600000"
      errorMessage: "retention should be between 1m and 1h"
    - condition: "metadata.labels[\"data-criticality\"] in [\"C0\", \"C1\", \"C2\"]"
      errorMessage: "data-criticality should be one of C0, C1, C2"
```

CEL tips: use `int(string(...))` for config values, bracket notation for dotted/dashed keys, `has(field)` for optional checks.

### TopicPolicy (deprecated — do not use)

Legacy constraint-based policy for topics only. Replaced by ResourcePolicy. Do not run `conduktor get TopicPolicy` or generate TopicPolicy YAML — always use ResourcePolicy instead. Both `topicPolicyRef` and `policyRef` can coexist on ApplicationInstance during migration, but new configs should only use `policyRef`.

### ApplicationInstancePermission

Cross-team topic sharing. `spec` is **immutable** (delete and re-create to modify). `grantedTo` must be on same cluster.

```yaml
apiVersion: self-serve/v1
kind: ApplicationInstancePermission
metadata:
  application: "clickstream-app"
  appInstance: "clickstream-app-dev"
  name: "perm-to-another"
spec:
  resource: { type: TOPIC, name: "click.event-stream.avro", patternType: LITERAL }
  serviceAccountPermission: READ
  userPermission: NONE
  grantedTo: "another-appinstance-dev"
```

## 5. Console resource model

Console resources: `apiVersion: v2` (Topics: `apiVersion: kafka/v2`).
Key kinds: `Topic`, `Group`, `User`, `KafkaCluster`, `KafkaConnect`, `ServiceAccount`.

```yaml
apiVersion: kafka/v2
kind: Topic
metadata:
  cluster: shadow-it
  name: click.event-stream.avro
  labels: { data-criticality: C2 }
spec:
  replicationFactor: 3
  partitions: 3
  configs: { cleanup.policy: delete, retention.ms: '60000' }
```

Console RBAC manages permissions within Console UI only -- does NOT create Kafka ACLs. Self-service creates ACLs based on ApplicationInstance declarations.
