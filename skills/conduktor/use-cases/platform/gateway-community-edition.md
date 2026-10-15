# Gateway Community Edition

## When to use this

The user wants Conduktor Gateway with a Community Edition (CE) license: the free Kafka proxy that lets clients reach a cluster sitting in another VPC, cloud, or private network. Also when they want to try Gateway and have no Kafka cluster.

## What Community Edition is

- A license-restricted mode of Conduktor Gateway: same image (`conduktor/conduktor-gateway`), same env vars. The license decides what runs.
- Passthrough: Gateway rewrites the broker addresses Kafka advertises into addresses clients can reach, and forwards client SASL to Kafka. Kafka stays the authentication and authorization authority.
- Security: `KAFKA_MANAGED` only, on `SASL_PLAINTEXT` or `SASL_SSL` listeners. `GATEWAY_MANAGED` is not available.
- Not included: interceptors, topic views, alias topics, virtual clusters, Gateway service accounts, Gateway groups, topic concentration, failover. Gateway refuses to start when its configuration requires one of them, and Admin API calls that create or modify them return `403`.
- Anonymous product-usage telemetry goes to Conduktor and can't be disabled from Gateway.
- The license is free: https://www.conduktor.io/gateway/community-edition#request-license. Without `GATEWAY_LICENSE_KEY`, Gateway exits with code 98 (`No license found!`).

## Agent workflow

1. Check prereqs: `docker --version`, `docker compose version`, and `git --version` for the quickstart.
2. Ask for the license key. No key yet: give the request URL above and wait for it. Keep the key in `.env` or a secret, never in a committed file.
3. Ask: try it locally (no cluster needed), or put it in front of an existing cluster?
4. **Try it locally**: use the Gateway CE quickstart, not the Console quick-start (which has no Gateway). It brings its own private Kafka cluster, Schema Registry (Karapace), sample data, Gateway, and two consumers in separate Docker networks that reach Kafka only through Gateway.
   ```bash
   git clone https://github.com/conduktor/gateway-community-quickstart
   cd gateway-community-quickstart
   # write .env beside docker-compose.yaml: GATEWAY_LICENSE_KEY=<key>
   docker compose up -d
   until curl -sf http://localhost:8888/health/ready; do sleep 5; done
   ./demo.sh
   ```
   `demo.sh` runs each check (a consumer can't reach Kafka directly, then reads through Gateway). It waits for Enter only on a terminal, so it runs straight through from an agent. Stop with `docker compose down`.
5. **Existing cluster**: ask for the bootstrap servers, the broker SASL mechanism and the credentials Gateway itself uses (for its internal topics), the lowest broker ID, and the host clients will use to reach Gateway. Then generate:
   ```yaml
   services:
     conduktor-gateway:
       image: conduktor/conduktor-gateway:3.21.1
       ports:
         - "6969-6974:6969-6974"   # one port per broker, about 2x the broker count
         - "8888:8888"             # admin API and health
       environment:
         KAFKA_BOOTSTRAP_SERVERS: broker1:9092,broker2:9092
         KAFKA_SECURITY_PROTOCOL: SASL_PLAINTEXT
         KAFKA_SASL_MECHANISM: PLAIN
         KAFKA_SASL_JAAS_CONFIG: org.apache.kafka.common.security.plain.PlainLoginModule required username="gateway" password="${KAFKA_PASSWORD}";
         GATEWAY_LICENSE_KEY: "${GATEWAY_LICENSE_KEY}"
         GATEWAY_SECURITY_MODE: KAFKA_MANAGED   # the only mode CE allows
         GATEWAY_ACL_ENABLED: "false"           # required in KAFKA_MANAGED; Kafka ACLs apply
         GATEWAY_MIN_BROKERID: 0                # the lowest broker ID
         GATEWAY_LISTENER_DEFAULT_SECURITY_PROTOCOL: SASL_PLAINTEXT
         GATEWAY_LISTENER_DEFAULT_ROUTING: port
         GATEWAY_LISTENER_DEFAULT_PORTS: "6969-6974"
         GATEWAY_LISTENER_DEFAULT_ADVERTISED_HOST: localhost   # the host clients resolve
         GATEWAY_ADMIN_API_USERS: '[{username: admin, password: change_me, admin: true}]'
   ```
   Match `KAFKA_SECURITY_PROTOCOL` and `KAFKA_SASL_*` to the cluster. For `SASL_SSL` listeners, the TLS keystore settings come from the docs MCP, not from memory. Verify with `curl -f http://localhost:8888/health/ready`.
6. Clients keep their own credentials and switch their bootstrap to `<advertised host>:6969`. Kafka authenticates them and Kafka ACLs (or Confluent RBAC) still apply.

Anything from the "not included" list needs a full Gateway license: https://www.conduktor.io/pricing#gateway. See [deploy-conduktor.md](deploy-conduktor.md) for the full Gateway env reference.

## Common mistakes

| Mistake | Fix |
|---|---|
| Taking the Console quick-start for "try Gateway" | It has no Gateway. Use the Gateway CE quickstart |
| `GATEWAY_SECURITY_MODE: GATEWAY_MANAGED` with a CE license | CE is `KAFKA_MANAGED` only |
| `GATEWAY_ACL_ENABLED: "true"` in `KAFKA_MANAGED` | Must be `"false"`. Kafka ACLs do the authorization |
| `PLAINTEXT` client listener | CE needs `SASL_PLAINTEXT` or `SASL_SSL`: Kafka authenticates the client's SASL |
| Configuring interceptors, virtual clusters or Gateway service accounts | Not in CE: Gateway won't start, and the Admin API returns `403` |
| Starting without a key | Exit 98 (`No license found!`). Request a free key first |
| Running `./start.sh` with no key set | It prompts on a terminal and exits without one. Write `.env` first (step 4) |
