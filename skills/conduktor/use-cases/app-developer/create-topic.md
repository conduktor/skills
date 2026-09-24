# Create a topic through Conduktor self-service

## Agent workflow

1. Run `conduktor run whoami` (an application-instance token names its ApplicationInstance), then `conduktor get ApplicationInstance <name> -o yaml` to read the cluster and the owned topic prefix. A 403 mentioning the license means self-service isn't available on this Console ([guardrails](../../references/guardrails.md) §1)
2. Run `conduktor get ResourcePolicy -o yaml` to discover naming rules, partition limits, and required labels
3. Ask the topic name, purpose, and any specific config needs (partitions, retention, cleanup policy)
4. Validate the proposed name against ResourcePolicy constraints and ownership prefix
5. Generate the complete `Topic` YAML (`apiVersion: v2` or `kafka/v2`, both valid; in an existing repo keep the one already used), metadata labels, and spec
6. Show the YAML and run `conduktor apply -f <file> --dry-run`. If the repo is applied with CLI state, dry-run with the same state and stop on any `Deleted (dry-run)` line ([guardrails](../../references/guardrails.md) §3)
7. On approval, run `conduktor apply -f <file>`

## When to use this

You are an application team member who needs to create a Kafka topic for your application. Your platform team has already set up an `Application`, `ApplicationInstance`, and optionally `ResourcePolicy` resources. You create topics using an **AppToken** (application instance API key) scoped to your instance.

## Prerequisites

- An `ApplicationInstance` exists on the target cluster with `resources[].type: TOPIC` granting you ownership over a name prefix (or literal name).
- You have an **AppToken** for that application instance (generated from Console UI or by the platform team).
- You know which `ResourcePolicy` constraints apply (check `spec.policyRef` on your ApplicationInstance).

## Topic YAML

```yaml
---
apiVersion: kafka/v2
kind: Topic
metadata:
  cluster: shadow-it
  name: click.order-events.avro
  labels:
    data-criticality: C2
  description: |
    # Order Events
    Captures user order events from the clickstream pipeline.
  catalogVisibility: PUBLIC
spec:
  replicationFactor: 3
  partitions: 6
  configs:
    cleanup.policy: delete
    retention.ms: '3600000'       # within the 1m–1h range of the policy below
    min.insync.replicas: '2'
```

Key fields:
- `metadata.cluster` -- must match your ApplicationInstance's `spec.cluster`.
- `metadata.name` -- must match your ownership pattern (e.g. prefix `click.` if `patternType: PREFIXED`).
- `metadata.labels` -- key-value pairs. Policies can enforce specific labels via `metadata.labels.<key>` constraints. `conduktor.io/*` labels are managed by Console and dropped if you set them.
- `metadata.description` -- markdown, shown in Topic Catalog. `descriptionIsEditable: false` locks UI edits.
- `metadata.catalogVisibility` -- `PUBLIC` (visible in Topic Catalog) or `PRIVATE`. Defaults to ApplicationInstance's `spec.defaultCatalogVisibility`.
- `spec.replicationFactor` -- immutable. `spec.partitions` can't change through `apply`; to add partitions, run `conduktor run topicAddPartitions --cluster <c> --topic-name <t> --partition-count <n>` (Console 1.47+, CLI 0.9+), then update the YAML.
- `spec.configs` -- standard Kafka topic configs. Quote decimals and booleans; integers are accepted either way.

## ResourcePolicy constraints

ResourcePolicies are linked to your ApplicationInstance via `spec.policyRef`. All policies with `targetKind: Topic` are evaluated on every `apply`. Rules use CEL (Common Expression Language) expressions.

Example policies your platform team might have set:

```yaml
---
apiVersion: self-serve/v1
kind: ResourcePolicy
metadata:
  name: generic-dev-topic
  labels:
    business-unit: delivery
spec:
  targetKind: Topic
  description: Standard topic creation rules
  rules:
    - condition: "metadata.labels[\"data-criticality\"] in [\"C0\", \"C1\", \"C2\"]"
      errorMessage: "data-criticality label must be one of C0, C1, C2"
    - condition: "int(string(spec.configs[\"retention.ms\"])) >= 60000 && int(string(spec.configs[\"retention.ms\"])) <= 3600000"
      errorMessage: "retention.ms must be between 1m and 1h"
    - condition: "spec.replicationFactor == 3"
      errorMessage: "replication factor must be 3"
---
apiVersion: self-serve/v1
kind: ResourcePolicy
metadata:
  name: clickstream-naming-rule
spec:
  targetKind: Topic
  rules:
    - condition: "metadata.name.matches(\"^click\\\\.[a-z0-9-]+\\\\.(avro|json)$\")"
      errorMessage: "topic name must match ^click.<event>.(avro|json)"
```

CEL tips:
- Config values are strings: use `int(string(spec.configs["retention.ms"]))` for numeric comparisons.
- Dotted or dashed keys need bracket notation: `metadata.labels["data-criticality"]`.
- Check optional fields with `has()`: `has(metadata.labels.criticality) && metadata.labels["criticality"] in ["C0"]`.

## How to create

### Via CLI

```bash
# Dry-run first to validate ownership and policies
conduktor apply -f my-topic.yaml --dry-run

# Apply
conduktor apply -f my-topic.yaml
```

Console (not the CLI) validates your topic against:
1. Ownership -- topic name must match an owned `resources[].name` + `patternType` in your ApplicationInstance.
2. ResourcePolicy -- all CEL rules from referenced policies with `targetKind: Topic` must pass.
3. Kafka -- only on the real apply. For a new topic, the dry-run does not call the broker, so an impossible replication factor or an invalid config passes the dry-run and fails at apply.

### Via Console UI

1. Navigate to your Application in the **Application Catalog**.
2. Select the target ApplicationInstance.
3. Click **Create Topic** and fill in the form -- the UI enforces the same policies.

## Topic catalog visibility

Topics owned by an ApplicationInstance appear in the **Topic Catalog** based on visibility:
- `PUBLIC` -- visible to everyone in Console. Searchable by application, cluster, and labels.
- `PRIVATE` -- hidden from the catalog. Only visible to the owning application team.

If `metadata.catalogVisibility` is not set on the topic, it inherits from `ApplicationInstance.spec.defaultCatalogVisibility` (defaults to `PUBLIC`).

## Common mistakes

| Mistake | Fix |
|---|---|
| Name does not match the ownership prefix | If the ApplicationInstance owns `click.` (`PREFIXED`), the name must start with `click.`; `clicks.foo` is rejected |
| Policy violation on a missing label | A rule reading `metadata.labels["data-criticality"]` without a guard (`"data-criticality" in metadata.labels`) requires the label |
| Trusting the dry-run for Kafka-level checks | Dry-run of a new topic skips the broker; RF and config errors only show at apply |
| Changing partitions or RF through `apply` | RF is immutable; add partitions with `conduktor run topicAddPartitions`, then update the YAML |
| Editing `description`, `catalogVisibility` or adding `labels` in a repo applied with CLI state | Under state this deletes and recreates the topic: dry-run with the state first ([guardrails](../../references/guardrails.md) §3) |
| Overlapping ownership | Two ApplicationInstances cannot own overlapping prefixes on the same cluster: if `click.` is taken, another instance can't own `click.orders.` |
