# Deploy Conduktor

## Agent workflow

1. Check prereqs: run `docker --version`, `kubectl version --client`, `helm version` to see what's available
2. Ask the user's goal:
   - **Quick test**: just try Conduktor with a demo Kafka cluster → use the official quick-start
   - **Connect to existing cluster**: the user already has Kafka (vanilla, Confluent Cloud, AWS MSK, Aiven, Redpanda) → generate a custom config
   - **Production deployment**: full setup with Helm/K8s, SSO, monitoring → walk through all options
3. **Quick test path**:
   - **Before starting**: check if port 8080 is available with `lsof -i :8080 -sTCP:LISTEN`. If taken, either free it or remap to another port (e.g., `"8088:8080"`) in the compose file.
   - Run `curl -L https://releases.conduktor.io/quick-start -o docker-compose.yml && docker compose up -d`
   - Poll health: `until curl -sf http://localhost:8080/api/health/ready; do sleep 5; done`
   - Tell the user: Console at http://localhost:8080 — it will show an onboarding screen to create the first admin account
   - This includes Console + 2x PostgreSQL (metadata + SQL) + Monitoring (Cortex) + Redpanda + Schema Registry + sample data generator (no Gateway)
   - **If Console crashes**: always use `docker compose down && docker compose up -d` (full network recreation), NOT `docker compose restart`. DNS resolution failures on macOS Docker Desktop are common and require the network to be torn down and recreated.
4. **Existing cluster path**:
   - Ask: what Kafka flavor? (vanilla, Confluent Cloud, AWS MSK, Aiven, Redpanda)
   - Ask: bootstrap servers, auth type (PLAINTEXT, SASL_PLAINTEXT, SASL_SSL, SSL), credentials
   - Ask: schema registry URL if applicable
   - Ask: want Gateway too or Console only?
   - For Gateway, generate the `KAFKA_*` env vars for that flavor (see below). Console ignores `KAFKA_*`: register the cluster in Console through the UI, `CDK_CLUSTERS_0_*` env vars, a `clusters:` entry in platform-config, or a `KafkaCluster` resource
   - For Confluent Cloud: include `kafkaFlavor` config with cloud API key, environment ID, cluster ID
   - For AWS MSK with IAM: include IAM callback handler config
   - Offer to run `docker compose up -d` or `helm install`
   - Verify with health check, then help add the cluster in Console if needed
5. **Production path**:
   - Ask: Docker Compose or Helm/K8s?
   - Ask: Console only, Gateway only, or both?
   - Ask: auth method? (local users, LDAP/AD, OAuth2/OIDC via Auth0/Okta/Cognito)
   - Ask: license key? Gateway won't start without one (`GATEWAY_LICENSE_KEY`, 3.18+). Console without a license runs as Community Edition: 50 users, 3 clusters, 2 custom alerts, 1 masking policy, and no self-service, data quality policies or group permissions (`GET /api/public/info/v1/license` shows the exact limits)
   - Ask: external PostgreSQL connection details
   - Generate complete config with all env vars, SSO blocks, monitoring, health probes
   - Include security checklist items in comments
6. After any deployment, verify health endpoints and help configure CLI auth:
   - `export CDK_BASE_URL=http://localhost:8080`
   - `export CDK_API_KEY=<from Console UI: Settings > API Keys>`, or without the UI: `CDK_USER=admin@company.io CDK_PASSWORD='<pwd>' conduktor token create admin cli`
   - `conduktor run whoami` to verify (read the output: `run` exits 0 even on errors)
   - Gateway: `curl -f http://localhost:8888/health/ready`

## When to use this

When deploying Conduktor Console (UI + API) and/or Gateway (Kafka proxy) via Docker Compose or Kubernetes. Covers minimal bootable configs, env var reference, health checks, and combined deployments.

## Console

### Docker image and ports

- Image: `conduktor/conduktor-console`
- Default port: `8080` (configurable via `CDK_LISTENING_PORT`)
- Runs as non-root user `conduktor` (UID `10001`, GID `10001`); use the numeric IDs in `securityContext`
- Volume: `/var/conduktor` for internal data
- JVM: uses container CGroups limits, 70% of container memory for heap (`-XX:MaxRAMPercentage=70`, override with `CONSOLE_MEMORY_OPTS`)

### Essential environment variables

| Variable | Description | Required |
|---|---|---|
| `CDK_DATABASE_URL` | PostgreSQL connection URL: `postgresql://user:pass@host:5432/dbname` | Yes |
| `CDK_ORGANIZATION_NAME` | Organization name | No (default: `"Conduktor"`) |
| `CDK_ADMIN_EMAIL` | Root admin account email | Yes (Helm); in Docker, the first-run onboarding screen can create it |
| `CDK_ADMIN_PASSWORD` | Root admin password. Min 8 chars, mixed case, number, symbol; a weak one stops Console at startup | With `CDK_ADMIN_EMAIL` |
| `CDK_LICENSE` | License key. Without one, Console runs as Community Edition (limited, see workflow step 5) | No |

Database URL format: `[jdbc:]postgresql://[user[:password]@][[netloc][:port],...][/dbname][?param1=value1&...]`

Alternative: decompose via `CDK_DATABASE_HOSTS_0_HOST`, `CDK_DATABASE_HOSTS_0_PORT`, `CDK_DATABASE_NAME`, `CDK_DATABASE_USERNAME`, `CDK_DATABASE_PASSWORD`. The older `CDK_DATABASE_HOST`/`_PORT` still work but are deprecated.

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
    healthcheck:
      test: pg_isready -U conduktor -d conduktor
      interval: 5s
      retries: 10

  conduktor-console:
    image: conduktor/conduktor-console:1.47.2   # pin the version
    depends_on:
      postgresql:
        condition: service_healthy   # Console exits if PG isn't accepting connections yet
    restart: unless-stopped
    ports:
      - "8080:8080"
    volumes:
      - conduktor_data:/var/conduktor
    environment:
      CDK_DATABASE_URL: "postgresql://conduktor:change_me@postgresql:5432/conduktor"
      CDK_ORGANIZATION_NAME: "demo"
      CDK_ADMIN_EMAIL: "admin@company.io"
      CDK_ADMIN_PASSWORD: "Change_me1!"
      CDK_LICENSE: "${CDK_LICENSE:-}"

volumes:
  pg_data: {}
  conduktor_data: {}
```

The image ships its own healthcheck on `/api/health/ready`, so no override is needed.

### Health check

```bash
curl -f http://localhost:8080/api/health/ready
```

Returns HTTP 200 when Console is ready. Liveness: `/api/health/live`. Readiness: `/api/health/ready`. Use readiness for Docker healthcheck and Kubernetes readiness probes.

## Gateway

### Docker image and ports

- Image: `conduktor/conduktor-gateway` (pin the tag, e.g. `:3.21.1`)
- License required: `GATEWAY_LICENSE_KEY`. Since 3.18, Gateway exits with code 98 (`No license found!`) without it
- Kafka-facing ports: defined per listener (`GATEWAY_LISTENER_<NAME>_PORTS`), e.g. `6969-6974`
- Admin API and probes: port `8888` (`GATEWAY_HTTP_PORT`), health on `/health/live` and `/health/ready` (the old `/health` was removed in 3.21)

### Essential environment variables

Since 3.20, networking is configured per listener with `GATEWAY_LISTENER_<NAME>_*`, where `<NAME>` is alphanumeric (`DEFAULT`, `EXTERNAL`…).

| Variable | Description |
|---|---|
| `KAFKA_BOOTSTRAP_SERVERS` | Comma-separated Kafka brokers (required) |
| `GATEWAY_LICENSE_KEY` | License (required since 3.18) |
| `GATEWAY_SECURITY_MODE` | `GATEWAY_MANAGED` (Gateway authenticates clients) or `KAFKA_MANAGED` (brokers do). Required with listeners |
| `GATEWAY_ACL_ENABLED` | `true`/`false`. Required with listeners; must be `false` in `KAFKA_MANAGED` |
| `GATEWAY_LISTENER_<NAME>_SECURITY_PROTOCOL` | `PLAINTEXT`, `SASL_PLAINTEXT`, `SSL` or `SASL_SSL` |
| `GATEWAY_LISTENER_<NAME>_ROUTING` | `port` (one port per broker) or `sni` (single port, TLS required) |
| `GATEWAY_LISTENER_<NAME>_PORTS` | Port or range, e.g. `6969-6974` (about 2× the broker count) |
| `GATEWAY_LISTENER_<NAME>_ADVERTISED_HOST` | Host returned to clients in metadata; must be reachable by them |
| `GATEWAY_MIN_BROKERID` | Lowest broker ID, for port routing (default `0`) |
| `GATEWAY_USER_POOL_SECRET_KEY` | Base64 256-bit key signing LOCAL service-account tokens on SASL listeners (`openssl rand -base64 32`) |
| `GATEWAY_ADMIN_API_USERS` | Admin API users, default `[{username: admin, password: conduktor, admin: true}]`: change it |
| `GATEWAY_SECURED_METRICS` | Require auth on `/metrics` (default `true`); the admin API is always authenticated |

`GATEWAY_PORT_START`, `GATEWAY_PORT_COUNT`, `GATEWAY_ADVERTISED_HOST`, `GATEWAY_ROUTING_MECHANISM`, `GATEWAY_SECURITY_PROTOCOL`, `GATEWAY_ADVERTISED_SNI_PORT` and `GATEWAY_ADVERTISED_HOST_PREFIX` are legacy globals: deprecated in 3.20, with removal planned in 3.23. Gateway refuses to start when they are mixed with listener variables.

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
    image: conduktor/conduktor-gateway:3.21.1
    ports:
      - "6969-6974:6969-6974"   # Kafka listener ports
      - "8888:8888"             # admin API and health
    environment:
      KAFKA_BOOTSTRAP_SERVERS: kafka1:9092,kafka2:9092
      GATEWAY_LICENSE_KEY: "${GATEWAY_LICENSE_KEY}"
      GATEWAY_SECURITY_MODE: GATEWAY_MANAGED
      GATEWAY_ACL_ENABLED: "false"          # dev only
      GATEWAY_LISTENER_DEFAULT_SECURITY_PROTOCOL: PLAINTEXT
      GATEWAY_LISTENER_DEFAULT_ROUTING: port
      GATEWAY_LISTENER_DEFAULT_PORTS: "6969-6974"
      GATEWAY_LISTENER_DEFAULT_ADVERTISED_HOST: localhost
      GATEWAY_ADMIN_API_USERS: '[{username: admin, password: change_me, admin: true}]'
```

A PLAINTEXT listener can't host virtual clusters or LOCAL service accounts. For those, use `SASL_PLAINTEXT` (`SASL_SSL` in production), `GATEWAY_ACL_ENABLED: "true"` and a `GATEWAY_USER_POOL_SECRET_KEY`.

### Routing (port vs SNI)

- **Port routing** (`GATEWAY_LISTENER_<NAME>_ROUTING=port`): one port per broker, mapped in order from `GATEWAY_MIN_BROKERID`. Publish the whole range. Non-sequential broker IDs (100, 200, 300) would need a range covering 100–300, so use SNI instead.
- **SNI routing** (`…_ROUTING=sni`): a single port, routed on the TLS SNI hostname. It needs `SSL`/`SASL_SSL`, a wildcard certificate, and DNS for each broker host (`…_ADVERTISED_HOST_PATTERN`, e.g. `broker-{{nodeId}}.kafka.example.com`). Follow the docs' SNI routing tutorial.

### Security checklist

1. Load `GATEWAY_LICENSE_KEY` from a secret, not inline
2. Change the default `GATEWAY_ADMIN_API_USERS` credentials
3. SASL listeners with LOCAL service accounts: set a strong `GATEWAY_USER_POOL_SECRET_KEY` and keep it secret
4. `GATEWAY_ACL_ENABLED: "true"` outside dev in `GATEWAY_MANAGED`
5. TLS listeners (`SSL`/`SASL_SSL`) for clients in production; TLS to Kafka on untrusted networks

## Combined deployment (Console + Gateway + PostgreSQL)

Put the `postgresql` and `conduktor-console` services and the `conduktor-gateway` service above in one compose file, on the same network and with one `volumes:` block. Kafka must be reachable at `KAFKA_BOOTSTRAP_SERVERS` from that network. To manage Gateway from Console (interceptors, data quality), register Gateway as a cluster in Console and fill its Gateway provider settings (admin API URL and credentials). Console 1.44+ requires Gateway 3.12+.

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
  --set config.database.name="conduktor" \
  --set config.database.username="conduktor" \
  --set config.database.password="change_me" \
  --set config.license="${CDK_LICENSE}"
```

`config.database.name` is mandatory: without it the pod crash-loops on `Missing mandatory database name.` Pin the chart with `--version`; a chart release can lag the latest Console patch.

**Gateway chart** (3.21+ generates the listener configuration from `gateway.listeners`):
```bash
helm install gateway conduktor/conduktor-gateway -f values.yaml
```

```yaml
gateway:
  licenseKey: "<license>"        # or gateway.secretRef: a Secret holding GATEWAY_LICENSE_KEY
  securityMode: GATEWAY_MANAGED
  aclEnabled: "true"
  env:
    KAFKA_BOOTSTRAP_SERVERS: "kafka1:9092,kafka2:9092"   # string values only
  listeners:
    internal: { securityProtocol: PLAINTEXT, routing: port, ports: ["9092-9098"] }   # chart default
    # external: { enable: true, securityProtocol: SASL_SSL, routing: sni, ports: ["9092"], advertisedHostPattern: "broker-{{nodeId}}.kafka.example.com" }
```

Never put legacy network variables (`GATEWAY_PORT_START`, `GATEWAY_ADVERTISED_HOST`…) in `gateway.env`: Gateway refuses to start when they're mixed with the chart's listener variables. Full chart reference: `https://github.com/conduktor/conduktor-public-charts`.

You must provide your own PostgreSQL for Console. Conduktor does not ship a database dependency in the Helm chart.

## Connecting to existing Kafka clusters

The `KAFKA_*` env vars below configure Gateway's connection to Kafka. Console reads none of them: it gets its clusters from the UI, `CDK_CLUSTERS_0_*` env vars, `clusters:` in platform-config.yaml, or a `KafkaCluster` resource applied with the CLI.

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

Console cluster entry in platform-config.yaml. This is not a CLI resource: with `conduktor apply`, a cluster is a `kind: KafkaCluster` (`apiVersion: v2`) with `properties` as a map, `schemaRegistry.type: ConfluentLike` and `security.type: BasicAuth`. `conduktor template KafkaCluster` prints the shape.
```yaml
clusters:
  - id: confluent-prod
    name: "Confluent Prod"      # required: Console won't start without it
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

| Mistake | Fix |
|---|---|
| Weak `CDK_ADMIN_PASSWORD` | Console refuses to start (`Password must contain at least 8 characters…` in the logs). Use 8+ chars with upper, lower, digit and symbol |
| PostgreSQL not ready when Console starts | Console exits. Use `depends_on: {postgresql: {condition: service_healthy}}` with a `pg_isready` healthcheck, plus `restart` |
| No `GATEWAY_LICENSE_KEY` | Gateway 3.18+ exits 98 (`No license found!`) |
| Legacy network vars mixed with `GATEWAY_LISTENER_*`, including in Helm `gateway.env` | Gateway refuses to start. Keep listener variables only |
| Default Gateway admin credentials | `GATEWAY_ADMIN_API_USERS` defaults to `admin/conduktor`; change it before exposing Gateway |
| Advertised host not reachable by clients | Set `GATEWAY_LISTENER_<NAME>_ADVERTISED_HOST` (Helm: `advertisedHost`) to the name clients resolve |
| Port routing with non-sequential broker IDs | IDs 100/200/300 need ports for the whole 100–300 range from `GATEWAY_MIN_BROKERID`; use SNI routing |
| Helm Console install without `config.database.name` | CrashLoopBackOff on `Missing mandatory database name.` |
| Precedence between config file and env vars | Env vars override `platform-config.yaml`. Console secrets accept a `_FILE` suffix (e.g. `CDK_LICENSE_FILE=/run/secrets/license`); Gateway has no `_FILE` support (`GATEWAY_ENV_FILE` instead) |
| Docker Desktop stale port bindings (macOS) | If Console crashes, Docker Desktop may keep the host port allocated after `docker compose down` (`lsof -i :8080` shows `com.docker`). Restart Docker Desktop, or remap the port (e.g. `"8088:8080"`) |
| DNS resolution failures on macOS Docker Desktop | Console crashes with `java.net.UnknownHostException: postgresql`. `docker compose restart` reuses the network and doesn't fix it; run `docker compose down && docker compose up -d` |
