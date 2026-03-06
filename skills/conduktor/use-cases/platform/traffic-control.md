# Traffic control and safeguards with Conduktor Gateway

## Agent workflow

1. Run `conduktor get Interceptor --gateway -o yaml` to check existing safeguard interceptors
2. Ask what to control: rate limiting, topic creation standards, config protection, read-only mode, or schema enforcement
3. Ask the scope: global, per virtual cluster, per service account, or per group
4. Run `conduktor get VirtualCluster -o name` and `conduktor get GatewayServiceAccount -o name` to get real scope targets
5. Generate the complete `Interceptor` YAML with the correct safeguard `pluginClass` and action (BLOCK, INFO, OVERRIDE)
6. Show the YAML and offer to run `conduktor apply -f --dry-run`
7. On approval, run `conduktor apply -f`

Gateway interceptor policies enforce traffic rules at the proxy layer -- before requests reach Kafka. They validate, reject, or silently fix Kafka API requests in real time.

All safeguard plugins live under `io.conduktor.gateway.interceptor.safeguard.*` and are deployed as `kind: Interceptor` resources via CLI, API, Terraform, or UI.

## When to use this

- Prevent teams from creating topics with bad configs (wrong partitions, no replication, missing schema).
- Protect broker/topic configurations from dangerous changes.
- Rate-limit noisy producers or consumers.
- Make specific topics read-only (block produce, delete, alter).
- Enforce client-id naming, consumer group conventions, schema usage.
- Cap connection rates to prevent thundering herd.

## How interceptor policies work

Every safeguard interceptor intercepts a specific Kafka API call (CreateTopics, Produce, Fetch, AlterConfigs, etc.). When a request violates the policy, the configured `action` determines the outcome.

Interceptors are scoped via `metadata.scope` (vCluster, group, username) and matched by `spec.priority` (lower = first). Multiple interceptors with the same name but different scopes override each other (precedence: username > group > vCluster > global).

```yaml
apiVersion: gateway/v2
kind: Interceptor
metadata:
  name: my-policy
  scope:                    # optional targeting
    vCluster: my-vcluster   # or group: / username:
spec:
  pluginClass: io.conduktor.gateway.interceptor.safeguard.<Plugin>
  priority: 100
  config: { ... }
```

## Actions: BLOCK, INFO, OVERRIDE

| Action     | Behavior |
|:-----------|:---------|
| `BLOCK`    | Reject the request. Client gets `PolicyViolationException`. Audit log entry created. |
| `INFO`     | Allow the request as-is, but log a warning in audit. No client error. |
| `OVERRIDE` | Silently replace the violating value with `overrideValue`, then forward to Kafka. Audit log records original and replaced values. |

Some plugins also support `THROTTLE` (delay the request by `throttleTimeMs` instead of rejecting).

For range-based configs, the pattern is `min` / `max` / `action` / `overrideValue`.

## Policy catalog

### Topic creation policies

**`CreateTopicPolicyPlugin`** -- validates CreateTopics requests.

Controls: `numPartition`, `replicationFactor`, `retentionMs`, `retentionBytes`, `segmentBytes`, `minInsyncReplicas`, `maxMessageBytes`, `cleanupPolicy`, `compressionType`, `namingConvention`, and more.

- `topic` (regex, default `.*`) filters which topics the policy applies to.
- `blacklist.values` blocks specific config keys entirely.

### Broker config protection

**`AlterBrokerConfigPolicyPlugin`** -- validates AlterBrokerConfig requests.

Protects: `log.retention.bytes`, `log.retention.ms`, `log.segment.bytes`. Supports `blacklist` to block specific config keys.

### Topic config protection

**`AlterTopicConfigPolicyPlugin`** -- validates AlterTopicConfig requests.

Protects: `retention.ms`, `retention.bytes`, `segment.ms`, `segment.bytes`, `segment.jitter.ms`, `flush.messages`, `flush.ms`, `max.message.bytes`, `min.insync.replicas`, `cleanup.policy`, `unclean.leader.election.enable`.

### Rate limiting (producer and consumer)

**`ProducerRateLimitingPolicyPlugin`** -- throttles producers exceeding `maximumBytesPerSecond`.

**`ConsumerRateLimitingPolicyPlugin`** -- throttles consumers exceeding `maximumBytesPerSecond`.

Both apply per Gateway instance. Scope with `metadata.scope.username` or `group` to target specific service accounts.

### Connection limiting

**`LimitConnectionPolicyPlugin`** -- caps connection rate via `maximumConnectionsPerSecond`. Actions: BLOCK, INFO, THROTTLE.

**`LimitCommitOffsetPolicyPlugin`** -- caps offset commits via `maximumCommitsPerMinute` per groupId.

**`LimitJoinGroupPolicyPlugin`** -- caps join-group attempts via `maximumJoinsPerMinute` per groupId.

### Read-only topics

**`ReadOnlyTopicPolicyPlugin`** -- blocks all mutating requests (Produce, DeleteTopics, AlterConfigs, CreatePartitions, DeleteRecords, etc.) on matching topics. Only BLOCK and INFO actions.

### Schema enforcement

**`TopicRequiredSchemaIdPolicyPlugin`** -- requires (or forbids) a schema ID on produced records. `schemaIdRequired: true` blocks records without a schema. Only BLOCK and INFO actions.

### Other policies

| Plugin | Purpose |
|:-------|:--------|
| `ProducePolicyPlugin` | Enforce acks mode, compression, headers required, idempotence, transactions on produce requests. No OVERRIDE -- only BLOCK, INFO, THROTTLE. |
| `FetchPolicyPlugin` | Enforce isolation level, rack-id usage, fetch byte bounds, fetch version. Supports THROTTLE. |
| `ConsumerGroupPolicyPlugin` | Validate groupId naming, session/rebalance timeouts. Supports THROTTLE. |
| `ClientIdRequiredPolicyPlugin` | Enforce client-id naming convention. OVERRIDE replaces client-id using templates (`{{userIp}}`, `{{user}}`, `{{uuid}}`, etc.). |
| `MessageHeaderRemovalPlugin` | Strip record headers matching `headerKeyRegex` on fetch responses. No action config -- always removes. |

## Working examples

**Example 1 -- Enforce topic creation standards**

Block topics with fewer than 3 partitions; auto-fix replication factor to 3; require `delete` cleanup policy.

```yaml
apiVersion: gateway/v2
kind: Interceptor
metadata:
  name: topic-creation-standards
spec:
  pluginClass: io.conduktor.gateway.interceptor.safeguard.CreateTopicPolicyPlugin
  priority: 100
  config:
    topic: ".*"
    numPartition:
      min: 3
      max: 50
      action: BLOCK
    replicationFactor:
      min: 3
      max: 3
      action: OVERRIDE
      overrideValue: 3
    cleanupPolicy:
      values:
        - delete
      action: BLOCK
    minInsyncReplicas:
      min: 2
      max: 2
      action: OVERRIDE
      overrideValue: 2
```

**Example 2 -- Rate-limit a specific service account**

Cap producer throughput at 10 MB/s for `team-analytics`.

```yaml
apiVersion: gateway/v2
kind: Interceptor
metadata:
  name: rate-limit-analytics
  scope:
    username: team-analytics
spec:
  pluginClass: io.conduktor.gateway.interceptor.safeguard.ProducerRateLimitingPolicyPlugin
  priority: 100
  config:
    maximumBytesPerSecond: 10485760
```

**Example 3 -- Read-only + schema enforcement on production topics**

```yaml
---
apiVersion: gateway/v2
kind: Interceptor
metadata:
  name: readonly-audit-topics
spec:
  pluginClass: io.conduktor.gateway.interceptor.safeguard.ReadOnlyTopicPolicyPlugin
  priority: 100
  config:
    topic: "audit-.*"
    action: BLOCK
---
apiVersion: gateway/v2
kind: Interceptor
metadata:
  name: require-schema
spec:
  pluginClass: io.conduktor.gateway.interceptor.safeguard.TopicRequiredSchemaIdPolicyPlugin
  priority: 100
  config:
    topic: ".*"
    schemaIdRequired: true
    action: BLOCK
```

Apply any of these with:
```bash
conduktor apply -f <file>.yaml
```

## Common mistakes

- **Forgetting `topic` regex** -- defaults to `.*`, meaning the policy applies to all topics including internal ones. Be explicit.
- **Using OVERRIDE without `overrideValue`** -- the interceptor silently ignores the override and the original value passes through.
- **Rate limiting without scope** -- `ProducerRateLimitingPolicyPlugin` without a `username` or `group` scope applies to all clients per Gateway instance, which is rarely what you want.
- **Conflicting priorities** -- two interceptors of the same type with the same priority and overlapping scope produce undefined ordering. Use distinct priorities.
- **OVERRIDE on ProducePolicyPlugin** -- this plugin does not support OVERRIDE. Only BLOCK, INFO, and THROTTLE are valid.
- **ReadOnlyTopicPolicyPlugin with OVERRIDE** -- only BLOCK and INFO are supported. OVERRIDE is silently ignored.
