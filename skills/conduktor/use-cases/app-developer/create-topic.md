# Create a topic through Conduktor self-service

## Agent workflow

1. Run `conduktor get ApplicationInstance -o yaml` to find the user's ApplicationInstance and its topic ownership prefix
2. Run `conduktor get ResourcePolicy -o yaml` to discover naming rules, partition limits, and required labels
3. Ask the topic name, purpose, and any specific config needs (partitions, retention, cleanup policy)
4. Validate the proposed name against ResourcePolicy constraints and ownership prefix
5. Generate the complete `Topic` YAML with correct `apiVersion: kafka/v2`, metadata labels, and spec
6. Show the YAML and offer to run `conduktor apply -f --dry-run`
7. On approval, run `conduktor apply -f`

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
    conduktor.io/application: clickstream-app
    conduktor.io/application-instance: clickstream-dev
  description: |
    # Order Events
    Captures user order events from the clickstream pipeline.
  catalogVisibility: PUBLIC
spec:
  replicationFactor: 3
  partitions: 6
  configs:
    cleanup.policy: delete
    retention.ms: '86400000'
    min.insync.replicas: '2'
```

Key fields:
- `metadata.cluster` -- must match your ApplicationInstance's `spec.cluster`.
- `metadata.name` -- must match your ownership pattern (e.g. prefix `click.` if `patternType: PREFIXED`).
- `metadata.labels` -- key-value pairs. Policies can enforce specific labels via `metadata.labels.<key>` constraints.
- `metadata.description` -- markdown, shown in Topic Catalog. `descriptionIsEditable: false` locks UI edits.
- `metadata.catalogVisibility` -- `PUBLIC` (visible in Topic Catalog) or `PRIVATE`. Defaults to ApplicationInstance's `spec.defaultCatalogVisibility`.
- `spec.partitions`, `spec.replicationFactor` -- immutable after creation.
- `spec.configs` -- standard Kafka topic configs as string values.

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
# Dry-run first to validate against policies and Kafka
conduktor apply -f my-topic.yaml --dry-run

# Apply
conduktor apply -f my-topic.yaml
```

The CLI validates your topic against:
1. Ownership -- topic name must match an owned `resources[].name` + `patternType` in your ApplicationInstance.
2. ResourcePolicy -- all CEL rules from referenced policies with `targetKind: Topic` must pass.
3. Kafka -- partition count, replication factor, and configs are validated against the broker using `validateOnly`.

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

- **Name does not match ownership prefix.** If your ApplicationInstance owns `click.` with `PREFIXED`, topic name must start with `click.`. A name like `clicks.foo` will be rejected.
- **Policy violation on missing label.** If a ResourcePolicy rule checks `metadata.labels["data-criticality"]` without a `has()` guard, you must include that label.
- **Config values must be strings.** Kafka configs in `spec.configs` are string values: `retention.ms: '60000'` not `retention.ms: 60000`.
- **Partitions and replicationFactor are immutable.** You cannot change them after the topic is created. Plan ahead.
- **Overlapping ownership.** Two ApplicationInstances cannot own overlapping prefixes on the same cluster. If `click.` is taken, `click.orders.` cannot be owned by another instance.
