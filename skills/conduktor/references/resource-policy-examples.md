# ResourcePolicy examples

Resource policies (`apiVersion: self-serve/v1`, `kind: ResourcePolicy`) enforce CEL-expression-based rules on resources. They can target `Topic`, `Connector`, `Subject`, or `ApplicationGroup` via `spec.targetKind`. Policies are not applied automatically — link them via `spec.policyRef` on an Application or ApplicationInstance, or via `spec.policiesRef` on a KafkaCluster or KafkaConnectCluster. Only AdminTokens can manage policies.

Below are starter policies for self-service. Place in `platform/policies/` and tune values to match your environment.

## Topic naming (`topic-naming.yml`)

```yaml
apiVersion: self-serve/v1
kind: ResourcePolicy
metadata:
  name: topic-naming
spec:
  targetKind: Topic
  description: "Enforces topic naming convention and required labels"
  rules:
    - condition: metadata.name.matches("^[a-z0-9-]+\\.[a-z0-9.-]+$")
      errorMessage: "Topic name must follow the pattern <app>.<descriptive-name>"
    - condition: has(metadata.labels.confidentiality) && metadata.labels["confidentiality"] in ["public", "internal", "restricted"]
      errorMessage: "Topics must have a 'confidentiality' label set to one of: public, internal, restricted"
```

## Dev topic rules (`topic-rules-dev.yml`)

```yaml
apiVersion: self-serve/v1
kind: ResourcePolicy
metadata:
  name: topic-rules-dev
spec:
  targetKind: Topic
  description: "Relaxed topic rules for dev environments"
  rules:
    - condition: spec.replicationFactor >= 1
      errorMessage: "Replication factor has to be at least 1"
    - condition: spec.partitions >= 1 && spec.partitions <= 3
      errorMessage: "Partition count has to be between 1 and 3"
```

## Prod topic rules (`topic-rules-prod.yml`)

```yaml
apiVersion: self-serve/v1
kind: ResourcePolicy
metadata:
  name: topic-rules-prod
spec:
  targetKind: Topic
  description: "Strict topic rules for production environments"
  rules:
    - condition: spec.replicationFactor == 3
      errorMessage: "Replication factor has to be exactly 3 in production"
    - condition: spec.partitions >= 1 && spec.partitions <= 12
      errorMessage: "Production topics need less than or equal to 12 partitions. If you need an exception, plead your case to the platform team."
    - condition: "\"retention.ms\" in spec.configs && int(string(spec.configs[\"retention.ms\"])) >= 3600000"
      errorMessage: "Retention has to be explicitly set and at least 1 hour in production"
    - condition: "\"min.insync.replicas\" in spec.configs && int(string(spec.configs[\"min.insync.replicas\"])) >= 2"
      errorMessage: "min.insync.replicas has to be at least 2 in production"
```

## Subject rules (`subject-rules.yml`)

```yaml
apiVersion: self-serve/v1
kind: ResourcePolicy
metadata:
  name: subject-rules
spec:
  targetKind: Subject
  description: "Enforces subject naming and compatibility standards"
  rules:
    - condition: metadata.name.matches("^[a-z0-9-]+\\.[a-z0-9.-]+-(?:key|value)$")
      errorMessage: "Subject name must end with -key or -value suffix"
    - condition: has(spec.compatibility) && spec.compatibility in ["BACKWARD", "BACKWARD_TRANSITIVE", "FORWARD", "FORWARD_TRANSITIVE", "FULL", "FULL_TRANSITIVE"]
      errorMessage: "Compatibility level has to be explicitly set and cannot be NONE"
```

## Connector rules (`connector-rules.yml`)

```yaml
apiVersion: self-serve/v1
kind: ResourcePolicy
metadata:
  name: connector-rules
spec:
  targetKind: Connector
  description: "Restricts connector plugin classes and task limits"
  rules:
    - condition: >
        spec.config["connector.class"] in [
          "io.connect.jdbc.JdbcSourceConnector",
          "io.connect.jdbc.JdbcSinkConnector",
          "io.debezium.connector.postgresql.PostgresConnector",
          "org.apache.kafka.connect.mirror.MirrorSourceConnector",
          "com.amazonaws.kafka.connect.s3.S3SinkConnector"
        ]
      errorMessage: "Only approved connector classes are allowed — contact the platform team to request a new class"
    - condition: int(spec.config["tasks.max"]) <= 8
      errorMessage: "tasks.max cannot exceed 8"
```

## ApplicationGroup restrictions (`appgroup-restrictions.yml`)

```yaml
apiVersion: self-serve/v1
kind: ResourcePolicy
metadata:
  name: appgroup-restrictions
spec:
  targetKind: ApplicationGroup
  description: "Enforces group-based membership and restricts production permissions"
  rules:
    - condition: "!has(spec.members) || spec.members.size() == 0"
      errorMessage: "Direct user membership is not allowed — use externalGroups to manage membership via your identity provider"
    - condition: >
        spec.permissions
            .filter(p, p.appInstance.endsWith("-prod") && p.resourceType == "TOPIC")
            .all(p, p.permissions.all(perm, perm in ["topicConsume", "topicViewConfig", "topicDataQualityManage"]))
      errorMessage: "For production topics, only consume, view config, and manage data quality are allowed"
```
