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
    kmsConfig:
      vault:
        uri: http://vault:8200
        token: $${VAULT_TOKEN}   # $$ = let Gateway resolve it; ${…} would be substituted by the CLI
    recordValue:
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
**Correct:** Use only documented classes: `io.conduktor.gateway.interceptor.EncryptPlugin`, `DecryptPlugin`, `EncryptSchemaBasedPlugin`, `safeguard.CreateTopicPolicyPlugin`, etc. When unsure, list what the running Gateway knows (`curl -u <admin>:<pwd> http://<gateway>:8888/gateway/v2/plugin`) or check the docs MCP. `conduktor template Interceptor` only prints one sample, not a catalog. Removed in 3.19: `FetchEncryptPlugin`, `FetchEncryptSchemaBasedPlugin`.

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

## 10. "Fixing" apiVersion prefixes on resources managed with CLI state

**Wrong:**
```yaml
# Existing file, applied with --enable-state
apiVersion: v2          # rewritten to kafka/v2 "to follow best practices"
kind: Topic
```
**Why:** Only the version number matters: the CLI reads the digit after `v`, and Console ignores the prefix. `v2`, `kafka/v2` and `iam/v2` are the same thing, and `conduktor get` and `conduktor template` print `v2`. But CLI state identifies a resource by its exact `apiVersion` string, so rewriting it on a state-managed Topic deletes and recreates the topic, with all its data. See [guardrails.md](guardrails.md) §3.
**Correct:** Keep the `apiVersion` strings a repo already uses. For new files, any prefix works as long as the number matches the kind (Topic `v2`, ServiceAccount `v1`, DataQualityRule `v1`).

## 11. Using TopicPolicy instead of ResourcePolicy

**Wrong:**
```yaml
kind: TopicPolicy
```
**Why:** TopicPolicy is deprecated, and since Console 1.47 creating or updating one is refused (400), which breaks GitOps repos that still apply them. ResourcePolicy replaces it with CEL expressions and supports Topic, Connector, Subject, ApplicationGroup and, since 1.47, ApplicationInstancePermission.
**Correct:**
```yaml
kind: ResourcePolicy
```
Use `spec.policyRef` on ApplicationInstance, not `spec.topicPolicyRef`. To migrate, list the existing ones with `conduktor get TopicPolicy -o yaml` (still readable), then recreate them as ResourcePolicies.
