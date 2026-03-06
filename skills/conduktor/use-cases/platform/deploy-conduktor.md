# Deploy Conduktor

## Agent workflow

1. Ask which deployment method: Docker Compose, Helm, or raw Kubernetes manifests
2. Ask which components: Console only, Gateway only, or both
3. Check if there's an existing `docker-compose.yml`, `values.yaml`, or Kubernetes manifests in the workspace
4. Generate the complete deployment config with all required env vars, ports, and health checks
5. For Docker Compose: offer to run `docker compose up -d`
6. For Helm: offer to run `helm install` with the generated values
7. After deployment, verify with health check endpoints (`/health` on Console 8080, `/health` on Gateway 8888)

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

## Common mistakes

1. **Weak admin password** -- `CDK_ADMIN_PASSWORD` requires min 8 chars, uppercase, lowercase, number, and symbol. Console silently fails to create the admin with a weak password.
2. **PostgreSQL not ready** -- Console crashes on startup if PG is unreachable. Use `depends_on` with healthcheck or init containers.
3. **Default Gateway admin credentials** -- `GATEWAY_ADMIN_API_USERS` defaults to `admin/conduktor`. Change before exposing.
4. **Missing `GATEWAY_USER_POOL_SECRET_KEY`** -- Required for all deployments using SASL. Without it, tokens can be forged.
5. **Port-based routing with non-sequential broker IDs** -- A 3-broker cluster with IDs 100, 200, 300 allocates 203 ports. Use SNI routing instead.
6. **`GATEWAY_ADVERTISED_HOST` not set** -- Clients receive the container hostname, which is not routable externally. Always set to the host/IP clients use.
7. **Mixing config file and env vars without understanding precedence** -- Env vars override `platform-config.yaml` values. Secrets can use `_FILE` suffix (e.g., `CDK_LICENSE_FILE=/run/secrets/license`).
