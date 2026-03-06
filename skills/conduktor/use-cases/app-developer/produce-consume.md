# Produce and consume through Conduktor Gateway

## When to use this

You are an application developer whose team has been given access to Kafka through Conduktor Gateway.
Your platform team has created an Application and ApplicationInstance for you (Self-service).
You need to configure your Kafka clients to produce and consume through Gateway instead of connecting directly to Kafka brokers.

## Connecting your application

### Bootstrap servers point to Gateway

Your Kafka clients MUST connect to Gateway, not to the backing Kafka cluster.

| Setting | Value |
|---|---|
| `bootstrap.servers` | `gateway-host:6969` (NOT `kafka-host:9092`) |

The Gateway port is `6969` by default. Your platform team will provide the exact hostname and port.

### Java producer/consumer config example

**Local service account (SASL_PLAINTEXT + PLAIN):**

```java
Properties props = new Properties();
props.put("bootstrap.servers", "gateway:6969");
props.put("security.protocol", "SASL_PLAINTEXT");
props.put("sasl.mechanism", "PLAIN");
props.put("sasl.jaas.config",
    "org.apache.kafka.common.security.plain.PlainLoginModule required "
    + "username='my-sa' password='my-token';");

// Producer
props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
KafkaProducer<String, String> producer = new KafkaProducer<>(props);

// Consumer
props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
props.put("group.id", "click.my-consumer-group");
KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
```

**External service account (OAUTHBEARER via OIDC):**

```java
props.put("security.protocol", "SASL_SSL");
props.put("sasl.mechanism", "OAUTHBEARER");
props.put("sasl.jaas.config",
    "org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required "
    + "clientId='my-client-id' clientSecret='my-secret' scope='kafka';");
props.put("sasl.login.callback.handler.class",
    "org.apache.kafka.common.security.oauthbearer.secured.OAuthBearerLoginCallbackHandler");
props.put("sasl.oauthbearer.token.endpoint.url", "https://idp.company.org/oauth2/token");
```

### Python (confluent-kafka) config example

```python
from confluent_kafka import Producer, Consumer

config = {
    "bootstrap.servers": "gateway:6969",
    "security.protocol": "SASL_PLAINTEXT",
    "sasl.mechanism": "PLAIN",
    "sasl.username": "my-sa",
    "sasl.password": "my-token",
}

producer = Producer(config)
producer.produce("click.screen-events", key="k1", value="v1")
producer.flush()

consumer = Consumer({**config, "group.id": "click.my-consumer-group", "auto.offset.reset": "earliest"})
consumer.subscribe(["click.screen-events"])
```

### Node.js (kafkajs) config example

```javascript
const { Kafka } = require("kafkajs");

const kafka = new Kafka({
  clientId: "my-app",
  brokers: ["gateway:6969"],
  sasl: {
    mechanism: "plain",
    username: "my-sa",
    password: "my-token",
  },
});

const producer = kafka.producer();
await producer.connect();
await producer.send({ topic: "click.screen-events", messages: [{ value: "hello" }] });

const consumer = kafka.consumer({ groupId: "click.my-consumer-group" });
await consumer.connect();
await consumer.subscribe({ topic: "click.screen-events", fromBeginning: true });
await consumer.run({ eachMessage: async ({ message }) => console.log(message.value.toString()) });
```

## Schema Registry

If your platform team has configured a Schema Registry behind Gateway, point your schema registry URL to Gateway as well.
Use the same credentials. Gateway proxies Schema Registry requests and applies interceptors (validation, compatibility) transparently.

## Virtual cluster isolation

When your ApplicationInstance is bound to a Virtual Cluster:

- You only see topics and consumer groups within your namespace.
- Listing topics returns only resources owned by or shared with your ApplicationInstance.
- Consumer groups must match your ownership pattern (e.g., `click.` prefix as defined in your ApplicationInstance).
- Other teams' resources are invisible to your clients -- no cross-contamination.

This isolation is enforced at the Gateway level. Your application code does not need to be aware of it.

## What Gateway does transparently

Gateway interceptors process your traffic in-flight. These are configured by the platform team and require zero changes in your application code.

### Encryption/decryption

If field-level encryption is configured (via `EncryptPlugin` or `EncryptSchemaBasedPlugin`), Gateway encrypts designated fields on produce and decrypts them on consume (via `DecryptPlugin`) -- provided your service account has the decrypt permission. Your application sends and receives plaintext.

### Data masking

If `FieldLevelDataMaskingPlugin` is active, consumers without decrypt permission receive masked values for sensitive fields. This is applied at the Gateway level on consume. Your consumer code sees masked data with no indication that masking occurred -- the message structure stays the same.

### Schema validation

If a schema validation interceptor is active (e.g., `SchemaPayloadValidationPolicyPlugin`), Gateway rejects produce requests whose payload does not conform to the registered schema. Your producer receives a standard Kafka error response. Fix the payload to match the expected schema.

## Common mistakes

| Mistake | Fix |
|---|---|
| Connecting to `kafka:9092` instead of `gateway:6969` | Update `bootstrap.servers` to the Gateway endpoint |
| Using SCRAM or other mechanism when Gateway expects PLAIN | Use `SASL_PLAINTEXT` + `PLAIN` for local service accounts |
| Consumer group name outside your ownership prefix | Use the prefix from your ApplicationInstance resources (e.g., `click.`) |
| Topic name outside your namespace | You can only produce/consume topics owned by or shared with your ApplicationInstance |
| Producer rejected with schema error | Your payload does not conform to the registered schema -- check the Schema Registry |
| Seeing masked/encrypted data | Your service account lacks decrypt permission -- ask your platform team |
| TLS errors with OAUTHBEARER | External OIDC requires `SASL_SSL`, not `SASL_PLAINTEXT` |
