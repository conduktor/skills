# Request access to another team's topic

## Agent workflow

1. Run `conduktor get Topic -o name` to list available topics
2. Ask which topic the user needs access to and what level (READ or WRITE)
3. Run `conduktor get ApplicationInstance -o name` to find the user's ApplicationInstance
4. Run `conduktor get ApplicationInstance -o yaml` on the target topic to identify the owning ApplicationInstance
5. Generate the `ApplicationInstancePermission` YAML with real names from steps 1-4
6. Show the YAML and offer to run `conduktor apply -f --dry-run`
7. On approval, run `conduktor apply -f`

## When to use this

You need to consume or produce to a topic that belongs to another team's ApplicationInstance. Your own ApplicationInstance does not own the topic prefix -- you need an `ApplicationInstancePermission` granted by the topic owner.

## How permissions work

Conduktor Self-service uses `ApplicationInstancePermission` to enable cross-team topic access. The **topic-owning team** creates this resource to grant read or write access to a **requesting team's ApplicationInstance**.

Key constraints:
- Both ApplicationInstances must be on the **same Kafka cluster**.
- The permission `spec.resource.name` must fall within the owning ApplicationInstance's resource pattern (sub-prefix or literal match).
- `spec` is **immutable** after creation. To change a permission, delete and recreate it.
- Only `TOPIC` is supported as the resource type.

The permission controls two separate access layers:
- `serviceAccountPermission` -- Kafka ACLs applied to the grantedTo service account.
- `userPermission` -- Console UI permissions granted to the grantedTo application team members.

## YAML: ApplicationInstancePermission

Grant read access on a single topic:

```yaml
---
apiVersion: self-serve/v1
kind: ApplicationInstancePermission
metadata:
  application: "clickstream-app"
  appInstance: "clickstream-app-dev"
  name: "clickstream-app-dev-to-heatmap"
spec:
  resource:
    type: TOPIC
    name: "click.screen-events"
    patternType: LITERAL
  userPermission: READ
  serviceAccountPermission: READ
  grantedTo: "heatmap-app-dev"
```

Grant write access on a topic prefix:

```yaml
---
apiVersion: self-serve/v1
kind: ApplicationInstancePermission
metadata:
  application: "clickstream-app"
  appInstance: "clickstream-app-dev"
  name: "clickstream-app-dev-to-enrichment-write"
spec:
  resource:
    type: TOPIC
    name: "click.orders."
    patternType: PREFIXED
  userPermission: WRITE
  serviceAccountPermission: WRITE
  grantedTo: "enrichment-app-dev"
```

Metadata fields:
- `metadata.application` -- the owning Application.
- `metadata.appInstance` -- the owning ApplicationInstance (source of truth for resource ownership).
- `metadata.name` -- unique identifier for this permission.

## Request flow

### Via Console UI

1. Open the **Topic Catalog** and find the topic you need.
2. On the topic detail page, click **Request Access** for your ApplicationInstance.
3. The owning team receives the request and can approve or deny it from the same page.
4. On approval, Conduktor creates the `ApplicationInstancePermission` and syncs Kafka ACLs automatically.

Topic owners can also proactively grant access from the **Application Instance** details page under access management.

### Via CLI (owner applies the resource)

The owning team creates the YAML and applies it:

```bash
# Dry-run to validate
conduktor apply -f permission.yaml --dry-run

# Apply
conduktor apply -f permission.yaml
```

API keys accepted: **AdminToken** or **AppToken** (scoped to the owning application instance).

## Permission types and Kafka side effects

| `serviceAccountPermission` | Kafka ACLs granted to grantedTo SA | Confluent Cloud RBAC |
|---|---|---|
| `READ` | `READ`, `DESCRIBE_CONFIGS` | `DeveloperRead` |
| `WRITE` | `READ`, `WRITE`, `DESCRIBE_CONFIGS` | `DeveloperRead`, `DeveloperWrite` |
| `NONE` | No Kafka ACLs | No role bindings |

| `userPermission` | Console UI access for grantedTo team |
|---|---|
| `READ` | View topic data and configuration |
| `WRITE` | View and produce to the topic |
| `NONE` | No Console UI access |

For Confluent Cloud with RBAC enabled: `WRITE` on a topic also implicitly grants write access on subjects sharing the same topic prefix.

## Resource pattern rules

The permission `spec.resource.name` must be a sub-resource of the owning ApplicationInstance's declared resources. If the owner holds prefix `click.`:

| `patternType` | `name` | Valid? |
|---|---|---|
| `LITERAL` | `click.screen-events` | Yes -- falls under `click.` prefix |
| `PREFIXED` | `click.orders.` | Yes -- sub-prefix of `click.` |
| `PREFIXED` | `click.` | Yes -- exact prefix match |
| `LITERAL` | `payments.order` | No -- outside ownership |

## Common mistakes

| Mistake | Cause | Fix |
|---|---|---|
| Permission rejected: resource outside ownership | `spec.resource.name` does not fall under the owning AppInstance's resource pattern | Use a name that matches or is a sub-pattern of the owner's declared prefix |
| Permission rejected: different cluster | `grantedTo` AppInstance is on a different Kafka cluster than `metadata.appInstance` | Both AppInstances must target the same `spec.cluster` |
| Trying to update spec fields | `spec` is immutable after creation | Delete the existing permission and recreate with new values |
| Confusing ownership vs permission | Creating an ApplicationInstance resource for a prefix you don't own | Use `ApplicationInstancePermission` for cross-team access, not `ApplicationInstance.spec.resources` |
| Using AppToken from the wrong instance | AppToken must belong to the owning application instance, not the requesting one | Generate the AppToken from the owning team's ApplicationInstance |
| `userPermission` and `serviceAccountPermission` both set to NONE | Permission resource is valid but grants nothing | Set at least one to `READ` or `WRITE` |
