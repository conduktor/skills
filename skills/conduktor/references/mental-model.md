# Conduktor mental model

## 1. Product boundary

**Console** -- `conduktor/conduktor-console`, port 8080, requires PostgreSQL.
Provides: RBAC, topic catalog, monitoring, self-service framework, data quality UI.
Console resources: the version number must match the kind (`v2` for Topic, Group, User, KafkaCluster, KafkaConnectCluster; `v1` for self-service kinds, ServiceAccount and data quality). Prefixes such as `kafka/`, `iam/`, `console/` or `self-serve/` are conventions: the CLI and Console ignore them.

**Gateway** -- `conduktor/conduktor-gateway`, port 6969.
Transparent Kafka proxy. Handles: interceptors, virtual clusters, encryption, masking, traffic control.
Gateway resources: `apiVersion: gateway/v2`.

**CLI** -- `conduktor` binary (Go). kubectl-style verbs: `apply`, `get`, `delete`, `edit`, `template`.
Targets Console via `CDK_BASE_URL` or Gateway via `CDK_GATEWAY_BASE_URL`.

Console and Gateway are **separate services**. Console can manage Gateway resources, but they deploy independently.

## 2. Gateway resource model

All Gateway resources: `apiVersion: gateway/v2`. 6 resource kinds, plus `TopicView` (GA in 3.20):

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
- `LOCAL`: credentials come from `POST /gateway/v2/token` (singular) with a mandatory TTL, or `conduktor run generateServiceAccountToken --v-cluster <vc> --username <sa> --life-time-seconds <n>`.
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

Routes virtual topic creation into shared physical topics based on `cleanup.policy`. `autoManaged: true` = auto-create physical topics. `offsetCorrectness` is deprecated since 3.21 (removal planned in 3.24): leave it `false`. Spec changes do NOT affect previously created concentrated topics.

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
| `LOCAL` | Gateway-managed only (SASL) | `/gateway/v2/token` |
| `EXTERNAL` | Gateway-managed (mTLS/OAuth) or Kafka-managed | N/A |

| Security Mode | Virtual Clusters | Local SA | External SA |
|---|---|---|---|
| `GATEWAY_MANAGED` | Yes (needs a SASL or mTLS listener) | Yes (SASL only) | Yes (mTLS or OAuth) |
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
  owner: "clickstream-owners"   # Console Group name: lowercase, [0-9a-z_.-] only
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

CEL-expression-based policy enforcement. Replaces the legacy TopicPolicy. `spec.targetKind`: `Topic`, `Connector`, `Subject`, `ApplicationGroup` or (1.47+) `ApplicationInstancePermission`. Where to link it depends on the kind:
- Topic, Connector and Subject policies: the ApplicationInstance `spec.policyRef`, which accepts only these three kinds.
- ApplicationGroup policies: the Application `spec.policyRef`.
- ApplicationInstancePermission policies: the KafkaCluster `spec.policiesRef`.

A cluster referencing a policy in `policiesRef` needs that policy to exist first. Rules use `condition` (CEL expr) + `errorMessage`.

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

CEL tips: use `int(string(...))` for config values, bracket notation for dotted/dashed keys. For optional field checks: `has(obj.field)` works only with dot-accessible keys (no hyphens/dots in the key name). For hyphenated keys like `data-classification`, use `"data-classification" in metadata.labels` instead — `has()` does not support bracket notation.

### TopicPolicy (deprecated — do not use)

Legacy constraint-based policy for topics only, replaced by ResourcePolicy. Since Console 1.47, creating or updating a TopicPolicy is refused, so never generate TopicPolicy YAML. Use `conduktor get TopicPolicy` only to list what to migrate. Existing ones keep applying through `topicPolicyRef` until migrated; new configs use `policyRef`.

### ApplicationGroup

Console RBAC scoped to an Application. Grants permissions on resources within specific ApplicationInstances. `spec.members` is **required by the API** — set to `[]` when using `externalGroups` for IdP-managed membership.

```yaml
apiVersion: self-serve/v1
kind: ApplicationGroup
metadata:
  application: "clickstream-app"
  name: "clickstream-support"
spec:
  displayName: "Clickstream Support"
  description: "Read access to clickstream resources"
  members: []                          # required even if empty
  externalGroups:
    - clickstream-developers           # IdP group name (SSO/LDAP groups claim)
  permissions:
    - appInstance: "clickstream-dev"
      resourceType: TOPIC
      patternType: LITERAL
      name: "*"
      permissions: ["topicViewConfig", "topicConsume"]
```

- `spec.members`: list of user emails for direct membership. Required field — use `[]` if relying solely on `externalGroups`.
- `spec.externalGroups`: list of identity-provider group names (LDAP/OIDC groups claim), synchronized at login. Users in those IdP groups inherit the ApplicationGroup's permissions. Local members of a Console Group inherit nothing, unless its name happens to match the IdP group (the template's convention).
- `spec.permissions`: list of permission entries, each scoped to an `appInstance` and `resourceType`.

### ApplicationInstancePermission

Cross-team topic sharing. `spec` is **immutable** (delete and re-create to modify). `grantedTo` must be on same cluster.

```yaml
apiVersion: self-serve/v1
kind: ApplicationInstancePermission
metadata:
  application: "clickstream-app"
  appInstance: "clickstream-dev"
  name: "perm-to-another"
spec:
  resource: { type: TOPIC, name: "click.event-stream.avro", patternType: LITERAL }
  serviceAccountPermission: READ
  userPermission: NONE
  grantedTo: "another-appinstance-dev"
```

## 5. Console resource model

Key kinds and versions: `Topic`, `Subject`, `Connector`, `Group`, `User`, `KafkaCluster`, `KafkaConnectCluster` (v2); `ServiceAccount` (v1, Kafka ACLs of a principal on a cluster). Only the version number matters: `kafka/v2` and `v2` are equivalent. Never rewrite the string on resources managed with CLI state (see [guardrails.md](guardrails.md) §3).

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
