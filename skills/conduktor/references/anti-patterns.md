# What AI assistants must NOT do with Conduktor

Common mistakes LLMs make when generating Conduktor configurations. Each is verified against official docs.

## 1. Direct Kafka connection when Gateway is present

**Wrong:**
```properties
bootstrap.servers=kafka:9092
```
**Why:** When Gateway is deployed, all clients MUST connect through it. Direct Kafka access bypasses interceptors, encryption, ACLs, and virtual clusters.
**Correct:**
```properties
bootstrap.servers=gateway:6969
```

## 2. Confluent Terraform resources

**Wrong:**
```hcl
resource "confluent_kafka_topic" "orders" { ... }
```
**Why:** Conduktor has its own Terraform provider (`conduktor/conduktor`). Confluent resources do not exist in this provider.
**Correct:**
```hcl
resource "conduktor_console_topic_v2" "orders" { ... }
resource "conduktor_gateway_interceptor_v2" "encrypt" { ... }
resource "conduktor_gateway_service_account_v2" "app" { ... }
```

## 3. Conflating authentication and authorization planes

**Wrong:** Assuming Console RBAC controls Gateway access, or that Gateway service accounts are Kafka ACLs.
**Why:** Three separate systems exist:
- **Console RBAC** -- controls UI/API permissions within Conduktor Console only
- **Gateway service accounts + ACLs** -- controls client access through Gateway (managed via `GatewayServiceAccount`, `VirtualCluster` aclMode)
- **Kafka ACLs** -- native broker-level ACLs, only relevant in Kafka-managed mode

**Correct:** Configure each plane independently. Console RBAC does not propagate to Kafka or Gateway.

## 4. Application-level encryption

**Wrong:**
```java
// Encrypting in producer code
producer.send(new ProducerRecord<>(topic, encrypt(value)));
```
**Why:** Gateway handles field-level encryption transparently via interceptors. Application code should send plaintext.
**Correct:** Deploy a Gateway interceptor:
```yaml
apiVersion: gateway/v2
kind: Interceptor
metadata:
  name: encrypt-pii
spec:
  pluginClass: io.conduktor.gateway.interceptor.EncryptPlugin
  priority: 100
  config:
    topic: sensitive-data
    fields:
      - fieldName: email
        keySecretId: vault-kms://vault:8200/transit/keys/pii-key
        algorithm: AES128_GCM
```

## 5. CLI targeting confusion

**Wrong:**
```bash
export CDK_BASE_URL=http://gateway:6969
conduktor apply -f interceptor.yaml  # Fails: CLI talks to Console, not Gateway
```
**Why:** `CDK_BASE_URL` targets Console. Gateway has a separate admin API on a different port.
**Correct:**
```bash
# Console resources
export CDK_BASE_URL=http://console:8080
export CDK_API_KEY=<token>

# Gateway resources
export CDK_GATEWAY_BASE_URL=http://gateway:8888
export CDK_GATEWAY_USER=admin
export CDK_GATEWAY_PASSWORD=conduktor
```

## 6. Inventing interceptor plugin class names

**Wrong:**
```yaml
pluginClass: io.conduktor.gateway.interceptor.FieldEncryptionInterceptor  # Does not exist
```
**Why:** Plugin class names are exact. No fuzzy matching. Invalid names fail at deploy.
**Correct:** Use only documented classes: `io.conduktor.gateway.interceptor.EncryptPlugin`, `DecryptPlugin`, `EncryptSchemaBasedPlugin`, `safeguard.CreateTopicPolicyPlugin`, etc. When unsure, use `conduktor template Interceptor`.

## 7. Using Kafka CLI tools instead of Conduktor CLI

**Wrong:**
```bash
kafka-topics.sh --create --topic orders --bootstrap-server gateway:6969
```
**Why:** Kafka CLI tools bypass Console metadata (labels, descriptions, catalog visibility, SQL storage). They also cannot manage Gateway resources (interceptors, virtual clusters, service accounts).
**Correct:**
```bash
conduktor apply -f topic.yaml
# or scaffold first:
conduktor template Topic > topic.yaml
```

## 8. Treating aclMode as mutable

**Wrong:**
```yaml
# Trying to switch from KAFKA_API to REST_API after creation
spec:
  aclEnabled: true
  aclMode: REST_API  # Was KAFKA_API -- this will fail
```
**Why:** `aclMode` on a VirtualCluster is immutable after creation. `KAFKA_API` uses cumulative mutations; `REST_API` uses idempotent replacements. These are fundamentally incompatible, so switching is blocked.
**Correct:** Choose `aclMode` at VirtualCluster creation time. To change it, delete and recreate the VirtualCluster.

## 9. Interceptor scope semantics (omitted vs null)

**Wrong:** Assuming an interceptor without `scope` applies to everything.
```yaml
metadata:
  name: my-interceptor
  # no scope -- LLMs assume this means "all traffic"
```
**Why:** Omitted scope = global **EXCLUDING** virtual clusters. `vCluster: null` = global **INCLUDING** virtual clusters. This is a critical distinction.
**Correct:**
```yaml
# Global EXCLUDING virtual clusters (scope omitted entirely)
metadata:
  name: my-interceptor
spec: ...

---
# Global INCLUDING virtual clusters (scope fields set to null)
metadata:
  name: my-interceptor
  scope:
    vCluster: null
    group: null
    username: null
spec: ...
```

## 10. Wrong apiVersion for self-service resources

**Wrong:**
```yaml
apiVersion: v1
kind: ApplicationGroup
```
```yaml
apiVersion: self-service/v1
kind: Application
```
**Why:** Self-service resources require exactly `self-serve/v1`. Using `v1` or `self-service/v1` may be silently accepted by some API versions but is incorrect and can cause failures. This applies to: `Application`, `ApplicationInstance`, `ApplicationInstancePermission`, `ApplicationGroup`, and `ResourcePolicy`.
**Correct:**
```yaml
apiVersion: self-serve/v1
kind: ApplicationGroup
```

## 11. Using TopicPolicy instead of ResourcePolicy

**Wrong:**
```bash
conduktor get TopicPolicy -o yaml
```
```yaml
kind: TopicPolicy
```
**Why:** TopicPolicy is deprecated. It only supports topics and uses rigid constraint types (OneOf, Range, Match). ResourcePolicy replaces it with CEL expressions and supports Topic, Connector, Subject, and ApplicationGroup.
**Correct:**
```bash
conduktor get ResourcePolicy -o yaml
```
```yaml
kind: ResourcePolicy
```
Use `spec.policyRef` on ApplicationInstance, not `spec.topicPolicyRef`.
