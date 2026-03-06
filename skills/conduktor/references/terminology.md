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

| apiVersion | Scope | Example kinds |
|---|---|---|
| `gateway/v2` | Gateway resources | Interceptor, VirtualCluster, GatewayServiceAccount, GatewayGroup, AliasTopic, ConcentrationRule |
| `self-serve/v1` | Self-service | Application, ApplicationInstance, ApplicationInstancePermission, ApplicationGroup, TopicPolicy, ResourcePolicy |
| `kafka/v2` | Console Kafka resources | Topic |
| `v2` | Console resources | Group, User, KafkaCluster, KafkaConnect, ServiceAccount |
| `v1` | Data quality | DataQualityRule, DataQualityPolicy |

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
| `login` | Login user using username/password to get a JWT token |
| `token` | Manage Admin and Application Instance tokens |
| `sql` | Run a SQL command on indexed topics |
| `version` | Display the version of conduktor |
