# Enforce data quality with Conduktor

## When to use this

You need to validate Kafka message content at the streaming layer before bad data reaches consumers. Typical scenarios:

- Reject messages with missing or malformed fields (emails, IDs, timestamps)
- Enforce structural contracts on JSON payloads via JSON Schema
- Ensure messages carry a valid Avro schema ID from the registry
- Tag non-compliant messages with a header so downstream consumers can route or dead-letter them

Requires: Console 1.34+, Gateway 3.9+, Gateway cluster configured in Console with Provider tab credentials.

## How it works

Two resource kinds manage data quality:

1. **`DataQualityRule`** (`apiVersion: v1`) -- defines a single validation check (CEL expression, JSON Schema, or built-in).
2. **`DataQualityPolicy`** (`apiVersion: v1`) -- attaches one or more Rules to topic targets and configures actions (block/mark).

Rules do nothing alone. They must be attached to a Policy to take effect.

Flow: Producer -> Gateway -> Policy evaluates Rules against message -> block or mark -> Kafka.

## Rule types

### CEL rules (with YAML)

CEL expressions access three root variables: `key`, `value`, `headers`.
Available macros: `has()` (field existence), `matches()` (regex).

Caveat: Gateway converts all payloads to JSON before CEL evaluation. All numeric types become `number` -- no int/double distinction. `type(value.age) == int` will fail even on valid integers. Use range checks instead: `value.age > 0 && value.age < 130`.

Use bracket notation for headers with special chars: `headers['Content-Type']`.

```yaml
apiVersion: v1
kind: DataQualityRule
metadata:
  name: check-customer-email
spec:
  type: Cel
  displayName: Check customer email
  description: Verify email format on customer events.
  celExpression: >
    value.customer.email.matches(r"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$")
```

```yaml
apiVersion: v1
kind: DataQualityRule
metadata:
  name: require-json-content-type
spec:
  type: Cel
  displayName: Require JSON content type
  description: Header must declare JSON content type.
  celExpression: headers['Content-Type'] == 'application/json'
```

### JSON Schema rules (with YAML)

Validates message value against a JSON Schema definition. Defaults to draft 2020-12 if no `$schema` is specified.

```yaml
apiVersion: v1
kind: DataQualityRule
metadata:
  name: valid-order
spec:
  type: JsonSchema
  displayName: Valid order schema
  description: Orders must have id, product, and positive quantity.
  schema:
    {
      "$schema": "http://json-schema.org/draft-07/schema#",
      "type": "object",
      "required": ["id", "product", "quantity"],
      "properties": {
        "id": { "type": "string", "minLength": 1 },
        "product": { "type": "string" },
        "quantity": { "type": "integer", "minimum": 1 }
      },
      "additionalProperties": false
    }
```

### Built-in rules (EnforceAvro)

`EnforceAvro` validates that:
- The message has a schema ID prepended to the payload.
- The schema ID exists in the schema registry.
- The referenced schema is of type Avro.

Only supports Confluent and Confluent-like (e.g. Redpanda) schema registries.

Built-in rules are created in Console UI (not via YAML). Attach them to a Policy like any other rule.

## Creating a policy (YAML)

A Policy binds rules to topic targets and sets actions. Targets support `LITERAL` and `PREFIXED` pattern types. You cannot mix Gateway and non-Gateway targets in a single policy.

```yaml
apiVersion: v1
kind: DataQualityPolicy
metadata:
  name: order-validation
  group: orders-team
spec:
  displayName: Order payload validation
  description: Validate order events on the purchase pipeline.
  rules:
    - check-customer-email
    - valid-order
  targets:
    - cluster: gateway
      topic: purchase-pipeline
      patternType: LITERAL
    - cluster: gateway
      topic: order-
      patternType: PREFIXED
  actions:
    block:
      enabled: false
    mark:
      enabled: true
```

Apply with CLI:

```bash
conduktor apply -f policy.yaml
```

## Actions: block vs mark

| Action | Behavior | When both enabled |
|---|---|---|
| **Block** | Rejects the produce request. Message never reaches Kafka. | Block takes precedence. |
| **Mark** | Adds header `conduktor.dataquality.violations` to the message. Message is delivered. | Ignored when block is also enabled. |

Mark header value is JSON mapping policy names to arrays of violated rule names:

```json
{"order-validation": ["check-customer-email", "valid-order"]}
```

When multiple policies target the same topic:
- If no policy blocks: all policies are evaluated, each gets its own violation/evaluation counts.
- If any policy blocks: the first blocking policy rejects the record and increments its counts. Other policies are not evaluated.

## Viewing violations in Console

Console v1.42+ shows violations when consuming from Gateway clusters:
- A Policy badge appears in the topic header.
- The consume table adds a **Marking violations** column.
- The **Metadata** tab shows full violation details per message.

Policy detail page shows: linked rules, targets, violation/evaluation counts, violation history.

Policy statuses: **Pending** (not deployed yet), **Ready** (up-to-date on Gateway), **Failed** (deployment error -- check Gateway connectivity).

## Gateway interceptor alternative

For teams not using Console-managed data quality, Gateway provides equivalent interceptor plugins:

| Plugin class | Purpose |
|---|---|
| `io.conduktor.gateway.interceptor.safeguard.DataQualityProducerPlugin` | CEL-based validation at produce time |
| `io.conduktor.gateway.interceptor.safeguard.SchemaPayloadValidationPolicyPlugin` | Schema-based payload validation |

These are raw Gateway interceptors (`apiVersion: gateway/v2`, `kind: Interceptor`) configured with `spec.pluginClass`. Use Console-managed `DataQualityRule`/`DataQualityPolicy` instead when possible -- they provide UI, violation history, metrics, and alerting.

## Common mistakes

| Mistake | Fix |
|---|---|
| Using `type(value.field) == int` in CEL | All numerics are `number` after JSON conversion. Use range checks: `value.field > 0`. |
| Creating rules without attaching to a policy | Rules do nothing alone. Create a Policy and reference the rule names. |
| Enabling both block and mark expecting both to apply | Block takes precedence. Mark is ignored when block is enabled. |
| Mixing Gateway and non-Gateway targets in one policy | Not allowed. Create separate policies per cluster type. |
| Dot notation for headers with dashes | Use bracket notation: `headers['Content-Type']`. |
| Expecting Avro type fidelity in CEL | CEL sees JSON-converted payload only. Avro type info is lost. |
| Omitting `$schema` and expecting draft-07 behavior | Defaults to draft 2020-12. Specify `$schema` explicitly if you need an older draft. |
