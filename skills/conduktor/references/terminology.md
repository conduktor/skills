# Conduktor terminology and quick reference

## Terminology mapping

| Conduktor term | Kafka equivalent | Notes |
|---|---|---|
| Virtual Cluster | no equivalent | Logical namespace in Gateway |
| Interceptor | no equivalent | Gateway plugin, not Kafka interceptor |
| Service Account (Gateway) | no equivalent | LOCAL or EXTERNAL type |
| Application | no equivalent | Self-service: groups business context |
| ApplicationInstance | no equivalent | Binds cluster + service account + resources |
| Console Group | no equivalent | RBAC group, NOT Kafka consumer group |
| `conduktor apply` | `kubectl apply` | Declarative resource management |
| Gateway port 6969 | Kafka port 9092 | Clients connect to Gateway instead |

## apiVersion quick reference

Only the version number is significant: the CLI reads the digit after `v`, and Console ignores the prefix. The prefixes below are the documented convention. Never rewrite them on resources managed with CLI state, because that deletes and recreates the resource ([guardrails.md](guardrails.md) §3).

| apiVersion | Scope | Example kinds |
|---|---|---|
| `gateway/v2` | Gateway resources | Interceptor, VirtualCluster, GatewayServiceAccount, GatewayGroup, AliasTopic, ConcentrationRule |
| `self-serve/v1` | Self-service | Application, ApplicationInstance, ApplicationInstancePermission, ApplicationGroup, ResourcePolicy, TopicPolicy (creation blocked since 1.47) |
| `kafka/v2` | Console Kafka resources | Topic, Subject, Connector |
| `iam/v2` | Console IAM | Group, User |
| `console/v2` | Console connections | KafkaCluster, KafkaConnectCluster |
| `v1` | Kafka ACLs and data quality | ServiceAccount, DataQualityRule, DataQualityPolicy |

## CLI environment variables

### Console

| Variable | Purpose |
|---|---|
| `CDK_BASE_URL` | Console instance URL (required) |
| `CDK_API_KEY` | API key authentication (recommended) |
| `CDK_USER` | Username authentication |
| `CDK_PASSWORD` | Password authentication |
| `CDK_AUTH_MODE` | `conduktor` (default) or `external` |

### Gateway

| Variable | Purpose |
|---|---|
| `CDK_GATEWAY_BASE_URL` | Gateway instance URL (required) |
| `CDK_GATEWAY_USER` | Gateway username (required) |
| `CDK_GATEWAY_PASSWORD` | Gateway password (required) |

### TLS

These apply to the Console client only. The CLI does not use them for the Gateway admin API.

| Variable | Purpose |
|---|---|
| `CDK_INSECURE` | `true` to skip TLS verification |
| `CDK_CACERT` | CA certificate file path |
| `CDK_CERT` | Client certificate file path |
| `CDK_KEY` | Client private key file path |

### State management

| Variable | Purpose |
|---|---|
| `CDK_STATE_ENABLED` | Enable state tracking (`true`/`false`) |
| `CDK_STATE_FILE` | Custom local state file path |
| `CDK_STATE_REMOTE_URI` | Remote storage URI (e.g. `s3://bucket/path/`) |

## CLI commands quick reference

| Command | Description |
|---|---|
| `apply` | Upsert a resource on Conduktor |
| `get` | Get resource of a given kind |
| `delete` | Delete resource of a given kind and name |
| `edit` | Edit a resource in a text editor and apply changes |
| `template` | Get a YAML example for a given kind |
| `login` | Exchange `CDK_USER`/`CDK_PASSWORD` for a JWT (prints it; fails with only an API key) |
| `token` | Manage Admin and Application Instance tokens (`token create admin <name>` works with `CDK_USER`/`CDK_PASSWORD`) |
| `run` | Call an API action: `whoami`, consumer groups (describe, reset offsets), connectors (pause, stop, offsets), topics (add partitions, empty). `conduktor run --help` lists what your Console exposes. Exits 0 on API errors |
| `sql` | Run a SQL command on indexed topics |
| `version` | Display the version of conduktor |
