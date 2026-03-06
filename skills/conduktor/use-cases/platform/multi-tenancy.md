# Multi-tenancy with Conduktor Gateway

## Agent workflow

1. Run `conduktor get VirtualCluster -o yaml` to list existing virtual clusters
2. Run `conduktor get GatewayServiceAccount -o name` and `conduktor get GatewayGroup -o name` to see existing accounts and groups
3. Ask how many teams/tenants, their names, and isolation requirements
4. Ask the ACL mode for each virtual cluster: `ALLOW_ALL` (dev), `REST_API` (recommended), or `KAFKA_API`
5. Generate the complete set of resources: `VirtualCluster`, `GatewayServiceAccount`, and `GatewayGroup` YAML for each tenant
6. If topic aliasing is needed, generate `AliasTopic` resources
7. Show all YAMLs and offer to run `conduktor apply -f --dry-run`
8. On approval, run `conduktor apply -f`

Gateway turns a single physical Kafka cluster into N isolated logical clusters using Virtual Clusters.
Each tenant gets its own namespace for topics, consumer groups, service accounts, ACLs, and interceptors -- all on shared infrastructure.

## When to use this

- Multiple teams/applications sharing one Kafka cluster
- Dev/staging/prod isolation without separate clusters
- SaaS platforms offering per-customer Kafka environments
- Cost reduction: consolidate N physical clusters into one + Gateway

Virtual Clusters are optional. Without any, Gateway acts as a transparent proxy.

## Architecture: virtual clusters

A Virtual Cluster prefixes all resources (topics, consumer groups) on the backing Kafka cluster.
Tenants see clean names; the physical cluster stores prefixed names.

```
Tenant Alice sees:    orders
Tenant Bob sees:      orders
Physical Kafka has:   vc-alice.orders, vc-bob.orders
```

Tenants are fully isolated: Alice cannot see or access Bob's resources.

## Step by step

### 1. Create a VirtualCluster

```yaml
---
apiVersion: gateway/v2
kind: VirtualCluster
metadata:
  name: "team-payments"
spec:
  aclEnabled: false
```

- `metadata.name` must be a valid topic prefix (it becomes the physical prefix).
- `spec.aclEnabled`: set `false` to skip authorization checks (any client on this vcluster can do anything).
- `spec.type`: `Standard` (default) or `Partner`.

### 2. Create service accounts

Local service account (Gateway-managed mode, SASL):

```yaml
---
apiVersion: gateway/v2
kind: GatewayServiceAccount
metadata:
  vCluster: team-payments
  name: payments-app
spec:
  type: LOCAL
```

External service account (mapping an IdP identity):

```yaml
---
apiVersion: gateway/v2
kind: GatewayServiceAccount
metadata:
  vCluster: team-payments
  name: payments-app
spec:
  type: EXTERNAL
  externalNames:
    - 00u9vme99nxudvxZA0h7
```

- `EXTERNAL`: `externalNames` must be a non-empty list (currently limited to one element). Each name must be unique across all GatewayServiceAccounts.
- `LOCAL`: use `/gateway/v2/tokens` to generate credentials for this account.

### 3. Group service accounts

Groups exist for interceptor scoping only. They cannot be used for ACL management.

```yaml
---
apiVersion: gateway/v2
kind: GatewayGroup
metadata:
  name: payments-readers
spec:
  members:
    - vCluster: team-payments
      name: payments-app
    - vCluster: team-payments
      name: payments-monitoring
```

- `spec.members[].name` must refer to an existing GatewayServiceAccount.
- `spec.members[].vCluster` is optional; omit when not using virtual clusters.

### 4. Scope interceptors per tenant

Interceptors can target a vcluster, a group, or a specific username. Precedence (highest to lowest): ServiceAccount > Group > VirtualCluster > Global.

```yaml
---
apiVersion: gateway/v2
kind: Interceptor
metadata:
  name: enforce-partition-limit
  scope:
    vCluster: team-payments
spec:
  pluginClass: "io.conduktor.gateway.interceptor.safeguard.CreateTopicPolicyPlugin"
  priority: 100
  config:
    topic: ".*"
    numPartition:
      min: 3
      max: 12
      action: "BLOCK"
```

Target a group instead:

```yaml
---
apiVersion: gateway/v2
kind: Interceptor
metadata:
  name: enforce-partition-limit
  scope:
    vCluster: team-payments
    group: payments-readers
spec:
  pluginClass: "io.conduktor.gateway.interceptor.safeguard.CreateTopicPolicyPlugin"
  priority: 100
  config:
    topic: ".*"
    numPartition:
      min: 3
      max: 12
      action: "BLOCK"
```

Global interceptor including vclusters (set all scope fields to `null`):

```yaml
metadata:
  scope:
    vCluster: null
    group: null
    username: null
```

Global interceptor excluding vclusters (omit scope entirely):

```yaml
metadata:
  name: enforce-partition-limit
# no scope block
spec: ...
```

### 5. Alias physical topics

Expose an existing physical topic inside a virtual cluster under a logical name.

```yaml
---
apiVersion: gateway/v2
kind: AliasTopic
metadata:
  name: orders
  vCluster: team-payments
spec:
  physicalName: shared-orders-v2
```

- `metadata.name`: the logical name tenants see.
- `spec.physicalName`: the real Kafka topic name.

### 6. Concentrate topics (optional)

Map many logical topics onto a few physical topics to reduce partition count.

```yaml
---
apiVersion: gateway/v2
kind: ConcentrationRule
metadata:
  vCluster: team-payments
  name: concentrate-events
spec:
  pattern: "events-.*"
  physicalTopics:
    delete: team-payments-events-delete
    compact: team-payments-events-compact
    deleteCompact: team-payments-events-dc
  autoManaged: false
  offsetCorrectness: false
```

- `spec.physicalTopics.delete` is mandatory. `compact` and `deleteCompact` are optional.
- Each physical topic must have the matching `cleanup.policy`.
- `autoManaged: true` auto-creates/extends physical topics.
- `offsetCorrectness: true` maintains per-concentrated-topic offsets (enables accurate lag reporting).
- Changing a ConcentrationRule does not affect previously created concentrated topics.
- A topic creation fails if its `cleanup.policy` has no matching physical topic configured.

## ACL management

Enable ACLs on the virtual cluster:

```yaml
spec:
  aclEnabled: true
  aclMode: REST_API  # or KAFKA_API
```

**`aclMode` is immutable after creation.** You cannot switch between `REST_API` and `KAFKA_API` because their mutation semantics are incompatible.

### REST_API mode

The `acls` field is a **full replacement** (idempotent). Every apply overwrites the entire ACL set.

```yaml
---
apiVersion: gateway/v2
kind: VirtualCluster
metadata:
  name: "team-payments"
spec:
  aclEnabled: true
  aclMode: REST_API
  acls:
    - resourcePattern:
        resourceType: TOPIC
        name: orders
        patternType: LITERAL
      principal: User:payments-app
      host: "*"
      operation: READ
      permissionType: ALLOW
    - resourcePattern:
        resourceType: TOPIC
        name: orders
        patternType: LITERAL
      principal: User:payments-app
      host: "*"
      operation: WRITE
      permissionType: ALLOW
```

- `superUsers` cannot be set in this mode.
- `acls` must be set when `aclMode: REST_API`.

### KAFKA_API mode

ACLs are managed cumulatively via the Kafka Admin API. A `superUsers` list defines which service accounts bypass ACL checks.

```yaml
---
apiVersion: gateway/v2
kind: VirtualCluster
metadata:
  name: "team-payments"
spec:
  aclEnabled: true
  aclMode: KAFKA_API
  superUsers:
    - payments-admin
```

- `acls` cannot be set in this mode.
- `superUsers` must be set.
- Super user names must match existing GatewayServiceAccounts on this vcluster (they are not auto-created).
- Tenants manage their own ACLs using standard Kafka Admin API calls through Gateway.

## Authentication modes

| Mode | Where auth happens | Local SA | External SA | Virtual Clusters |
|---|---|---|---|---|
| `GATEWAY_MANAGED` | Gateway | Yes (SASL) | Yes (mTLS, OAuth) | Yes |
| `KAFKA_MANAGED` | Backing Kafka | No | Yes | No |

Set via `GATEWAY_SECURITY_MODE` environment variable.

- `GATEWAY_MANAGED`: full control over service accounts and ACLs at the Gateway layer. Required for virtual clusters.
- `KAFKA_MANAGED`: delegates auth to Kafka. No virtual clusters, no alias topics, no concentrated topics. External service accounts can still be mapped for friendly naming.

## Common mistakes

| Mistake | Why it fails |
|---|---|
| Switching `aclMode` after creation | Immutable. Delete and recreate the vcluster. |
| Setting `acls` on a `KAFKA_API` vcluster | Mutually exclusive. Use `superUsers` + Kafka Admin API instead. |
| Setting `superUsers` on a `REST_API` vcluster | Mutually exclusive. Manage all ACLs declaratively in `acls`. |
| Using GatewayGroup for ACL management | Groups scope interceptors only, not ACLs. |
| `externalNames` with multiple entries | Currently limited to a list of one element. |
| `KAFKA_MANAGED` + Virtual Clusters | Virtual clusters require `GATEWAY_MANAGED` mode. |
| Forgetting to create the GatewayServiceAccount for a `superUsers` entry | Usernames in `superUsers` are not auto-created. |
| `vCluster` name with invalid topic-prefix characters | The name becomes the physical topic prefix; must be valid Kafka topic characters. |
