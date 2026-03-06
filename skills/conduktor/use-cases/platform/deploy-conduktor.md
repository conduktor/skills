# Deploy Conduktor

## Agent workflow

1. Check prereqs: run `docker --version`, `kubectl version --client`, `helm version` to see what's available
2. Ask the user's goal:
   - **Quick test**: just try Conduktor with a demo Kafka cluster → use the official quick-start
   - **Connect to existing cluster**: the user already has Kafka (vanilla, Confluent Cloud, AWS MSK, Aiven, Redpanda) → generate a custom config
   - **Production deployment**: full setup with Helm/K8s, SSO, monitoring → walk through all options
3. **Quick test path**:
   - Run `curl -L https://releases.conduktor.io/quick-start -o docker-compose.yml && docker compose up -d`
   - Poll health: `until curl -sf http://localhost:8080/platform/api/modules/health/live; do sleep 5; done`
   - Tell the user: Console at http://localhost:8080, login admin@conduktor.io / admin
   - This includes Console + Gateway + Redpanda + Schema Registry + sample data
4. **Existing cluster path**:
   - Ask: what Kafka flavor? (vanilla, Confluent Cloud, AWS MSK, Aiven, Redpanda)
   - Ask: bootstrap servers, auth type (PLAINTEXT, SASL_PLAINTEXT, SASL_SSL, SSL), credentials
   - Ask: schema registry URL if applicable
   - Ask: want Gateway too or Console only?
   - Generate a `docker-compose.yml` or `helm values.yaml` with the correct `KAFKA_*` env vars for that flavor
   - For Confluent Cloud: include `kafkaFlavor` config with cloud API key, environment ID, cluster ID
   - For AWS MSK with IAM: include IAM callback handler config
   - Offer to run `docker compose up -d` or `helm install`
   - Verify with health check, then help add the cluster in Console if needed
5. **Production path**:
   - Ask: Docker Compose or Helm/K8s?
   - Ask: Console only, Gateway only, or both?
   - Ask: auth method? (local users, LDAP/AD, OAuth2/OIDC via Auth0/Okta/Cognito)
   - Ask: license key? (omit for free plan)
   - Ask: external PostgreSQL connection details
   - Generate complete config with all env vars, SSO blocks, monitoring, health probes
   - Include security checklist items in comments
6. After any deployment, verify health endpoints and help configure CLI auth:
   - `export CDK_BASE_URL=http://localhost:8080`
   - `export CDK_API_KEY=<from Console UI: Settings > API Keys>`
   - `conduktor login` or `conduktor get all --console` to verify

## When to use this

When deploying Conduktor Console (UI + API) and/or Gateway (Kafka proxy) via Docker Compose or Kubernetes. Covers minimal bootable configs, env var reference, health checks, and combined deployments.

## Console

### Docker image and ports

- Image: `conduktor/conduktor-console`
- Default port: `8080` (configurable via `CDK_LISTENING_PORT`)
- Runs as non-root user `conduktor-platform` (UID `10001`, GID `0`)
- Volume: `/var/conduktor` for internal data
- JVM: uses container CGroups limits, 80% of container memory for max heap (`-XX:MaxRAMPercentage=80`)

### Essential environment variables

| Variable | Description | Required |
|---|---|---|
| `CDK_DATABASE_URL` | PostgreSQL connection URL: `postgresql://user:pass@host:5432/dbname` | Yes |
| `CDK_ORGANIZATION_NAME` | Organization name | No (default: `"default"`) |
| `CDK_ADMIN_EMAIL` | Root admin account email | Yes |
| `CDK_ADMIN_PASSWORD` | Root admin password. Min 8 chars, mixed case, number, symbol | Yes |
| `CDK_LICENSE` | Enterprise license key. Omit for free plan | No |

Database URL format: `[jdbc:]postgresql://[user[:password]@][[netloc][:port],...][/dbname][?param1=value1&...]`

Alternative: decompose via `CDK_DATABASE_HOST`, `CDK_DATABASE_PORT`, `CDK_DATABASE_NAME`, `CDK_DATABASE_USERNAME`, `CDK_DATABASE_PASSWORD`.

### Docker Compose example

```yaml
services:
  postgresql:
    image: postgres:14
    hostname: postgresql
    volumes:
      - pg_data:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: "conduktor"
      POSTGRES_USER: "conduktor"
      POSTGRES_PASSWORD: "change_me"
      POSTGRES_HOST_AUTH_METHOD: "scram-sha-256"

  conduktor-console:
    image: conduktor/conduktor-console
    depends_on:
      - postgresql
    ports:
      - "8080:8080"
    volumes:
      - conduktor_data:/var/conduktor
    healthcheck:
      test: curl -f http://localhost:8080/platform/api/modules/health/live || exit 1
      interval: 10s
      start_period: 10s
      timeout: 5s
      retries: 3
    environment:
      CDK_DATABASE_URL: "postgresql://conduktor:change_me@postgresql:5432/conduktor"
      CDK_ORGANIZATION_NAME: "demo"
      CDK_ADMIN_EMAIL: "admin@company.io"
      CDK_ADMIN_PASSWORD: "Change_me1!"
      CDK_LICENSE: "${CDK_LICENSE}"

volumes:
  pg_data: {}
  conduktor_data: {}
```

### Health check

```bash
curl -f http://localhost:8080/platform/api/modules/health/live
```

Returns HTTP 200 when Console is ready. Use in Docker healthcheck and Kubernetes liveness probes.

## Gateway

### Docker image and ports

- Image: `conduktor/conduktor-gateway`
- Default Kafka proxy port: `6969` (set via `GATEWAY_PORT_START`)
- HTTP management API port: `8888` (set via `GATEWAY_HTTP_PORT`)
- In port-based routing (default), Gateway opens `GATEWAY_PORT_COUNT` ports starting at `GATEWAY_PORT_START`

### Essential environment variables

| Variable | Description | Default |
|---|---|---|
| `KAFKA_BOOTSTRAP_SERVERS` | Comma-separated Kafka brokers | (required) |
| `GATEWAY_PORT_START` | First listening port | `6969` |
| `GATEWAY_ADVERTISED_HOST` | Hostname returned in metadata for clients | Container hostname |
| `GATEWAY_ROUTING_MECHANISM` | `port` or `host` (SNI) | `port` |
| `GATEWAY_SECURITY_MODE` | `GATEWAY_MANAGED` or `KAFKA_MANAGED` | Derived from protocols |
| `GATEWAY_USER_POOL_SECRET_KEY` | Base64 256-bit key for local service account tokens | (required for SASL) |
| `GATEWAY_ADMIN_API_USERS` | Admin API credentials JSON | `[{username: admin, password: conduktor, admin: true}]` |
| `GATEWAY_SECURED_METRICS` | Require auth for HTTP management API | `true` |

Kafka authentication (when Kafka requires auth):

| Variable | Description |
|---|---|
| `KAFKA_SECURITY_PROTOCOL` | `PLAINTEXT`, `SASL_PLAINTEXT`, `SASL_SSL`, `SSL` |
| `KAFKA_SASL_MECHANISM` | `PLAIN`, `SCRAM-SHA-256`, `SCRAM-SHA-512`, etc. |
| `KAFKA_SASL_JAAS_CONFIG` | Full JAAS config string |

### Docker Compose example

```yaml
services:
  conduktor-gateway:
    image: conduktor/conduktor-gateway
    ports:
      - "6969:6969"
    environment:
      KAFKA_BOOTSTRAP_SERVERS: kafka1:9092,kafka2:9092
      GATEWAY_ADVERTISED_HOST: localhost
      GATEWAY_USER_POOL_SECRET_KEY: "${GATEWAY_SECRET}" # openssl rand -base64 32
      GATEWAY_ADMIN_API_USERS: '[{username: admin, password: change_me, admin: true}]'
```

### Routing (port-based vs SNI)

**Port-based** (default, `GATEWAY_ROUTING_MECHANISM=port`):
- One port per broker. Ports: `GATEWAY_PORT_START` to `GATEWAY_PORT_START + GATEWAY_PORT_COUNT - 1`
- `GATEWAY_MIN_BROKERID`: must match lowest `broker.id` in cluster (default `0`)
- `GATEWAY_PORT_COUNT`: defaults to `(maxBrokerId - minBrokerId) + 3`
- Simple, no TLS required. Expose port range to clients.

**SNI / host-based** (`GATEWAY_ROUTING_MECHANISM=host`):
- Single port, routes via TLS SNI hostname
- Requires `SSL` or `SASL_SSL` as `GATEWAY_SECURITY_PROTOCOL`
- `GATEWAY_ADVERTISED_SNI_PORT`: port returned in metadata (default: `GATEWAY_PORT_START`)
- `GATEWAY_ADVERTISED_HOST_PREFIX`: broker name prefix (default: `broker`)
- Requires wildcard TLS cert and DNS setup

### Security checklist

1. Generate `GATEWAY_USER_POOL_SECRET_KEY`: `openssl rand -base64 32`
2. Change default `GATEWAY_ADMIN_API_USERS` credentials
3. Set `GATEWAY_SECURED_METRICS: true` (default) to require auth on HTTP API
4. Configure TLS between clients and Gateway in production
5. Configure TLS between Gateway and Kafka if on untrusted network

## Combined deployment (Console + Gateway + PostgreSQL)

### Full Docker Compose

```yaml
services:
  postgresql:
    image: postgres:14
    hostname: postgresql
    volumes:
      - pg_data:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: "conduktor"
      POSTGRES_USER: "conduktor"
      POSTGRES_PASSWORD: "change_me"
      POSTGRES_HOST_AUTH_METHOD: "scram-sha-256"

  conduktor-console:
    image: conduktor/conduktor-console
    depends_on:
      - postgresql
    ports:
      - "8080:8080"
    volumes:
      - conduktor_data:/var/conduktor
    healthcheck:
      test: curl -f http://localhost:8080/platform/api/modules/health/live || exit 1
      interval: 10s
      start_period: 10s
      timeout: 5s
      retries: 3
    environment:
      CDK_DATABASE_URL: "postgresql://conduktor:change_me@postgresql:5432/conduktor"
      CDK_ORGANIZATION_NAME: "demo"
      CDK_ADMIN_EMAIL: "admin@company.io"
      CDK_ADMIN_PASSWORD: "Change_me1!"
      CDK_LICENSE: "${CDK_LICENSE}"

  conduktor-gateway:
    image: conduktor/conduktor-gateway
    ports:
      - "6969:6969"
    environment:
      KAFKA_BOOTSTRAP_SERVERS: kafka1:9092,kafka2:9092
      GATEWAY_ADVERTISED_HOST: localhost
      GATEWAY_USER_POOL_SECRET_KEY: "${GATEWAY_SECRET}"
      GATEWAY_ADMIN_API_USERS: '[{username: admin, password: change_me, admin: true}]'

volumes:
  pg_data: {}
  conduktor_data: {}
```

Assumes Kafka is reachable at `kafka1:9092,kafka2:9092` from the Docker network.

## Helm / Kubernetes notes

Helm repo: `https://helm.conduktor.io`

```bash
helm repo add conduktor https://helm.conduktor.io
helm repo update
```

**Console chart:**
```bash
helm install console conduktor/console \
  --create-namespace -n conduktor \
  --set config.organization.name="myorg" \
  --set config.admin.email="admin@company.io" \
  --set config.admin.password="Change_me1!" \
  --set config.database.host="postgres-host" \
  --set config.database.port="5432" \
  --set config.database.username="conduktor" \
  --set config.database.password="change_me" \
  --set config.license="${CDK_LICENSE}"
```

**Gateway chart:**
```bash
helm install gateway conduktor/conduktor-gateway -f values.yaml
```

Gateway `values.yaml` uses `gateway.env` for environment variables. Full chart reference: `https://github.com/conduktor/conduktor-public-charts`.

You must provide your own PostgreSQL for Console. Conduktor does not ship a database dependency in the Helm chart.

## Connecting to existing Kafka clusters

When generating configs for an existing cluster, use the correct `KAFKA_*` env vars for Gateway or Console cluster YAML.

### Confluent Cloud

Gateway env vars:
```
KAFKA_BOOTSTRAP_SERVERS: pkc-xxxxx.region.aws.confluent.cloud:9092
KAFKA_SECURITY_PROTOCOL: SASL_SSL
KAFKA_SASL_MECHANISM: PLAIN
KAFKA_SASL_JAAS_CONFIG: >
  org.apache.kafka.common.security.plain.PlainLoginModule required
  username="<cluster-api-key>" password="<cluster-api-secret>";
```

Console cluster YAML (for `conduktor apply` or platform-config.yaml):
```yaml
clusters:
  - id: confluent-prod
    bootstrapServers: pkc-xxxxx.region.aws.confluent.cloud:9092
    properties: |
      security.protocol=SASL_SSL
      sasl.mechanism=PLAIN
      sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="<api-key>" password="<api-secret>";
    kafkaFlavor:
      type: Confluent
      key: "<cloud-api-key>"
      secret: "<cloud-api-secret>"
      confluentEnvironmentId: "<env-id>"
      confluentClusterId: "<cluster-id>"
    schemaRegistry:
      url: https://psrc-xxxxx.region.aws.confluent.cloud
      security:
        username: "<sr-api-key>"
        password: "<sr-api-secret>"
```

### AWS MSK with IAM

```
KAFKA_BOOTSTRAP_SERVERS: b-1.mycluster.xxxxx.kafka.region.amazonaws.com:9098
KAFKA_SECURITY_PROTOCOL: SASL_SSL
KAFKA_SASL_MECHANISM: AWS_MSK_IAM
KAFKA_SASL_JAAS_CONFIG: software.amazon.msk.auth.iam.IAMLoginModule required;
KAFKA_SASL_CLIENT_CALLBACK_HANDLER_CLASS: io.conduktor.aws.IAMClientCallbackHandler
```

### AWS MSK with SCRAM

```
KAFKA_BOOTSTRAP_SERVERS: b-1.mycluster.xxxxx.kafka.region.amazonaws.com:9096
KAFKA_SECURITY_PROTOCOL: SASL_SSL
KAFKA_SASL_MECHANISM: SCRAM-SHA-512
KAFKA_SASL_JAAS_CONFIG: >
  org.apache.kafka.common.security.scram.ScramLoginModule required
  username="<user>" password="<password>";
```

### Aiven / mTLS

```
KAFKA_BOOTSTRAP_SERVERS: kafka-xxxxx.aivencloud.com:12345
KAFKA_SECURITY_PROTOCOL: SSL
KAFKA_SSL_TRUSTSTORE_LOCATION: /certs/truststore.jks
KAFKA_SSL_TRUSTSTORE_PASSWORD: <password>
KAFKA_SSL_KEYSTORE_LOCATION: /certs/keystore.jks
KAFKA_SSL_KEYSTORE_PASSWORD: <password>
```

### SSO configuration

When the user wants SSO, add to Console env vars or `platform-config.yaml`:

**LDAP**:
```yaml
sso:
  ldap:
    - name: "corporate-ldap"
      server: "ldap://ldap.company.com:389"
      managerDn: "cn=admin,dc=company,dc=com"
      managerPassword: "${LDAP_PASSWORD}"
      search-base: "ou=users,dc=company,dc=com"
      search-filter: "(uid={0})"
      groups-enabled: true
      groups-base: "ou=groups,dc=company,dc=com"
      groups-filter: "(member={0})"
```

**OAuth2 (Okta, Auth0, etc.)**:
```yaml
sso:
  oauth2:
    - name: "okta"
      client-id: "${OAUTH_CLIENT_ID}"
      client-secret: "${OAUTH_CLIENT_SECRET}"
      openid:
        issuer: "https://company.okta.com"
      scopes: [openid, profile, email]
```

Callback URL: `http(s)://<console-host>:<port>/oauth/callback/<config-name>`

## Common mistakes

1. **Weak admin password** -- `CDK_ADMIN_PASSWORD` requires min 8 chars, uppercase, lowercase, number, and symbol. Console silently fails to create the admin with a weak password.
2. **PostgreSQL not ready** -- Console crashes on startup if PG is unreachable. Use `depends_on` with healthcheck or init containers.
3. **Default Gateway admin credentials** -- `GATEWAY_ADMIN_API_USERS` defaults to `admin/conduktor`. Change before exposing.
4. **Missing `GATEWAY_USER_POOL_SECRET_KEY`** -- Required for all deployments using SASL. Without it, tokens can be forged.
5. **Port-based routing with non-sequential broker IDs** -- A 3-broker cluster with IDs 100, 200, 300 allocates 203 ports. Use SNI routing instead.
6. **`GATEWAY_ADVERTISED_HOST` not set** -- Clients receive the container hostname, which is not routable externally. Always set to the host/IP clients use.
7. **Mixing config file and env vars without understanding precedence** -- Env vars override `platform-config.yaml` values. Secrets can use `_FILE` suffix (e.g., `CDK_LICENSE_FILE=/run/secrets/license`).
