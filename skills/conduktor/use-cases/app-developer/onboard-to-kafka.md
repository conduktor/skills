# Get started with Kafka through Conduktor

## Agent workflow

1. Run `conduktor run whoami`. With an application-instance token it names the user's ApplicationInstance. Then `conduktor get ApplicationInstance <name> -o yaml` gives its cluster and service account name. Self-service needs a paid Console license ([guardrails](../../references/guardrails.md) §1).
2. Ask for the Gateway bootstrap address and the client credentials (from the platform team, or a token for a LOCAL service account). Neither comes from the CLI: `get KafkaCluster` needs platform permissions and returns Console's own connection settings, not the app's credentials.
3. Ask what language/framework the user's application uses (Java, Python, Node.js, Go, etc.)
4. Generate a complete client connection config with the real bootstrap servers, credentials, and SASL settings
5. Run `conduktor get Topic --cluster <cluster> -o name` to show topics the user has access to
6. If no ApplicationInstance exists, guide through self-service setup or direct to their platform team

Your platform team runs Conduktor Gateway in front of Kafka. This guide gets you connected and producing/consuming in minutes.

## When to use this

- You received a Gateway endpoint (host:6969) and credentials from your platform team
- You need to connect a Java, Python, or any Kafka client to produce/consume data
- You want to understand what Gateway does for you transparently

## Your platform team uses Conduktor -- what this means for you

Conduktor Gateway is a Kafka-protocol proxy. Your application talks the normal Kafka protocol, but connects to **Gateway** instead of Kafka directly.

What changes for you:
- **Bootstrap server**: Gateway address (port `6969`), not the Kafka brokers (port `9092`)
- **Credentials**: issued by Gateway or your identity provider, not Kafka
- **Everything else**: standard Kafka client APIs, no proprietary SDK

What stays invisible:
- Gateway forwards your requests to Kafka
- Policies (encryption, data quality, rate limiting) are applied in-flight
- You never touch the real Kafka cluster

## How to connect

### Your bootstrap server is Gateway (host:6969), not Kafka

Your platform team provides a Gateway hostname. The default port is `6969`.

```
# Instead of:
bootstrap.servers=kafka-broker:9092

# Use:
bootstrap.servers=gateway.company.internal:6969
```

### Authentication

Ask your platform team which mode they use. There are two options:

#### LOCAL credentials (SASL_PLAINTEXT or SASL_SSL)

Your platform team creates a local service account in Gateway and gives you a username/password token generated from the Gateway token endpoint (`POST /gateway/v2/token`, or `conduktor run generateServiceAccountToken`). Each token has the TTL chosen when it is generated.

Client config:

```properties
# use SASL_SSL in production
security.protocol=SASL_PLAINTEXT
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required \
  username="my-service-account" \
  password="eyJhbGciOiJIUzI1NiJ9...";
```

Available when Gateway runs in `GATEWAY_MANAGED` mode.

#### EXTERNAL credentials (OIDC / mTLS)

Your organization's identity provider handles authentication. Gateway verifies the token or certificate.

**OIDC (OAUTHBEARER):**

```properties
security.protocol=SASL_SSL
sasl.mechanism=OAUTHBEARER
sasl.login.callback.handler.class=org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginCallbackHandler
sasl.oauthbearer.token.endpoint.url=https://idp.company.internal/oauth/token
sasl.jaas.config=org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required \
  clientId="my-client-id" \
  clientSecret="my-client-secret" \
  scope="kafka";
```

This handler class works with Kafka clients 3.x and 4.x; the `.secured.` variant was removed in 4.0. Kafka 4 clients also refuse token URLs that aren't allow-listed, so start the JVM with `-Dorg.apache.kafka.sasl.oauthbearer.allowed.urls=https://idp.company.internal/oauth/token`.

**mTLS:**

```properties
security.protocol=SSL
ssl.keystore.location=/path/to/client.keystore.jks
ssl.keystore.password=changeit
ssl.truststore.location=/path/to/truststore.jks
ssl.truststore.password=changeit
```

OIDC works in both `GATEWAY_MANAGED` and `KAFKA_MANAGED` modes (in `KAFKA_MANAGED`, the broker validates the token). mTLS client authentication exists only in `GATEWAY_MANAGED`.

### Java client configuration example

```java
Properties props = new Properties();
props.put("bootstrap.servers", "gateway.company.internal:6969");
props.put("security.protocol", "SASL_PLAINTEXT");
props.put("sasl.mechanism", "PLAIN");
props.put("sasl.jaas.config",
    "org.apache.kafka.common.security.plain.PlainLoginModule required "
    + "username=\"my-service-account\" "
    + "password=\"eyJhbGciOiJIUzI1NiJ9...\";");

// Then use KafkaProducer / KafkaConsumer as usual
KafkaProducer<String, String> producer = new KafkaProducer<>(props);
```

### Python client configuration example

```python
from confluent_kafka import Producer

producer = Producer({
    "bootstrap.servers": "gateway.company.internal:6969",
    "security.protocol": "SASL_PLAINTEXT",
    "sasl.mechanisms": "PLAIN",
    "sasl.username": "my-service-account",
    "sasl.password": "eyJhbGciOiJIUzI1NiJ9...",
})

producer.produce("my-topic", key="key1", value="hello from gateway")
producer.flush()
```

## Your virtual cluster

Gateway uses Virtual Clusters to isolate tenants on a shared Kafka cluster.

### You only see your topics

Your platform team assigns your service account to a virtual cluster. Inside it, you see only your own topics and consumer groups -- no other team's resources are visible.

Behind the scenes, Gateway prefixes your resource names on the physical cluster with the virtual cluster name, with no separator: `orders` in virtual cluster `teamname` becomes `teamnameorders`. You never see or manage these prefixes.

### Topic naming follows your ApplicationInstance pattern

If your organization uses Conduktor Self-service, your ApplicationInstance defines which topic prefixes you own. For example, if your resource pattern is `click.` with `PREFIXED`, you can create topics like `click.events`, `click.sessions`, etc.

## Discover available topics

### Console UI: Topic Catalog

Open the Conduktor Console web UI and navigate to the **Topic Catalog**. It lists all topics marked as public across the organization. You can filter by application, cluster, and metadata labels.

From a topic's detail page, you can request access for your application instance if you are not already subscribed.

### CLI: conduktor get Topic

```bash
conduktor get Topic --cluster my-cluster -o name
```

Lists the topics your credentials can see on that cluster. The kind name is singular and `--cluster` is required.

## What happens transparently

Gateway applies interceptor plugins to your traffic without requiring any changes to your client code.

### Encryption / decryption

If your platform team configured field-level encryption, sensitive fields in your messages are encrypted on produce and decrypted on consume -- automatically. Your application reads and writes plain data.

### Data quality enforcement

Gateway can validate messages against schemas or business rules before they reach Kafka. Invalid messages are rejected with a clear error, protecting downstream consumers from bad data.

### Rate limiting

Your platform team may set throughput limits per virtual cluster or service account. If you hit a limit, the Kafka protocol returns a throttle response, and well-behaved clients back off automatically.

## Next steps

- [Create a topic](create-topic.md) -- create and configure topics within your namespace
- [Produce and consume](produce-consume.md) -- end-to-end produce/consume walkthrough
- [Request access to another team's topic](request-access.md) -- use ApplicationInstancePermission or the Console UI

## Common mistakes

| Mistake | Fix |
|---|---|
| Pointing clients at the Kafka brokers | Use the Gateway bootstrap address your platform team gives you |
| Expecting the CLI to hand out client credentials | Credentials come from the platform team, a Gateway token (LOCAL service account), or your IdP |
| `...oauthbearer.secured.OAuthBearerLoginCallbackHandler` on Kafka clients 4.x | Use `org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginCallbackHandler`, and allow-list the token URL with `-Dorg.apache.kafka.sasl.oauthbearer.allowed.urls` |
| A comment at the end of a line in a `.properties` file | Java properties keep the comment as part of the value; put comments on their own line |
| `conduktor get topics` | `conduktor get Topic --cluster <cluster>`: singular kind, `--cluster` required |
