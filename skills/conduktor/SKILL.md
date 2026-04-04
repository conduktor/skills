---
name: conduktor
description: >
  Conduktor platform expertise for Apache Kafka management, governance,
  and self-service. Covers Console (observe and manage), Gateway (enforce
  and proxy with interceptors), and CLI (operate and automate). Use when
  working with Conduktor configuration, deployment, Kafka data governance,
  encryption, multi-tenancy, or self-service workflows.
license: Apache-2.0
metadata:
  author: conduktor
  version: "1.0"
---

## Conduktor Platform

Conduktor is three products that work together around Apache Kafka:

- **Console** (`conduktor/conduktor-console`, port 8080) observes and manages Kafka clusters. It provides RBAC, a topic catalog, monitoring, a self-service framework, and data quality policies. Requires PostgreSQL.
- **Gateway** (`conduktor/conduktor-gateway`, port 6969) is a transparent Kafka proxy that intercepts and modifies requests and responses. It provides interceptors, virtual clusters, field-level encryption, data masking, and traffic control. Clients connect to Gateway instead of Kafka directly.
- **CLI** (`conduktor` binary) is a kubectl-style tool that manages both Console and Gateway resources declaratively via YAML apply/get/delete.

Console and Gateway are separate services. Console can manage Gateway, but they deploy independently.

## Always read first

For any Conduktor question, load [references/mental-model.md](references/mental-model.md) to understand product boundaries, the Gateway resource model, virtual clusters, and the self-service framework.

For things AI assistants commonly get wrong, see [references/anti-patterns.md](references/anti-patterns.md).

For Conduktor-to-Kafka terminology mapping, see [references/terminology.md](references/terminology.md).

## Use cases

### Platform engineer

| I want to... | Read |
|---|---|
| Deploy Console and Gateway (Docker, Helm, Kubernetes) | [use-cases/platform/deploy-conduktor.md](use-cases/platform/deploy-conduktor.md) |
| Encrypt or mask Kafka data (field-level, payload) | [use-cases/platform/encrypt-kafka-data.md](use-cases/platform/encrypt-kafka-data.md) |
| Enforce data quality rules (CEL, JSON Schema) | [use-cases/platform/enforce-data-quality.md](use-cases/platform/enforce-data-quality.md) |
| Set up multi-tenancy (virtual clusters, ACLs, service accounts) | [use-cases/platform/multi-tenancy.md](use-cases/platform/multi-tenancy.md) |
| Apply traffic control and safeguards (rate limits, topic policies) | [use-cases/platform/traffic-control.md](use-cases/platform/traffic-control.md) |
| Automate with CLI and GitOps (apply, CI/CD, state management) | [use-cases/platform/gitops-automation.md](use-cases/platform/gitops-automation.md) |
| Set up self-service GitHub CI/CD (workflows, CODEOWNERS, tokens, policies) | [use-cases/platform/self-service-github-cicd.md](use-cases/platform/self-service-github-cicd.md) |
| Bootstrap self-service from existing clusters, topics, and permissions | [use-cases/platform/bootstrap-self-service-cli.md](use-cases/platform/bootstrap-self-service-cli.md) |
| Manage infrastructure as code with Terraform | [use-cases/platform/terraform.md](use-cases/platform/terraform.md) |

### Application developer

| I want to... | Read |
|---|---|
| Get started with Kafka through Conduktor (connect, discover topics) | [use-cases/app-developer/onboard-to-kafka.md](use-cases/app-developer/onboard-to-kafka.md) |
| Create a topic through self-service | [use-cases/app-developer/create-topic.md](use-cases/app-developer/create-topic.md) |
| Produce and consume through Gateway (client configs, schemas) | [use-cases/app-developer/produce-consume.md](use-cases/app-developer/produce-consume.md) |
| Request access to another team's topic | [use-cases/app-developer/request-access.md](use-cases/app-developer/request-access.md) |

## Agent behavior

**IMPORTANT: Do NOT call any documentation MCP tool before completing steps 1-2 below. These skill files + CLI discovery contain everything you need. MCP docs are a last resort (step 5).**

When a user asks about Conduktor, do not just explain how things work. Be an active assistant:

1. **Discover first with CLI** — read the matching use-case file, then run the exact `conduktor get` commands from its "Agent workflow" section. The CLI gives you real state; docs give you generic examples.
2. **Ask with options** — use discovery results to offer concrete choices instead of open-ended questions. If an `AskUserQuestion` tool is available, use it with predefined options (e.g. topic name suggestions, partition counts, retention presets). Ask one question at a time, not a list of numbered questions. Prefer sensible defaults — only ask when the choice genuinely matters.
3. **Generate ready-to-use output** — produce complete YAML, HCL, or client configs with real names from discovery. Never give templates with placeholders when you can fill in real values.
4. **Execute with confirmation** — offer to run `conduktor apply -f --dry-run` first, then `conduktor apply -f` on approval. For Terraform, offer `terraform plan` then `terraform apply`.
5. **Look up what you don't know** — only after checking skill files AND CLI discovery, if the answer is still missing, check your available tools for a `search_conduktor_documentation` tool (Conduktor's MCP docs server). This is a fallback, not a first step. Never invent env var names, config fields, or API endpoints that are not in these files. If the MCP tool is not available, tell the user to add it with this config:
   ```json
   { "mcpServers": { "conduktor-docs": { "type": "url", "url": "https://docs.conduktor.io/mcp" } } }
   ```

The CLI requires auth. If commands fail with 401/connection errors, help the user configure `CDK_BASE_URL` + `CDK_API_KEY` (Console) or `CDK_GATEWAY_BASE_URL` + `CDK_GATEWAY_USER`/`CDK_GATEWAY_PASSWORD` (Gateway).

## Intent routing

When a user mentions these keywords, load the corresponding file:

- **encrypt, mask, PII, GDPR, shield** -> `use-cases/platform/encrypt-kafka-data.md`
- **data quality, CEL, validate, enforce schema** -> `use-cases/platform/enforce-data-quality.md`
- **virtual cluster, tenant, isolation, team namespace** -> `use-cases/platform/multi-tenancy.md`
- **rate limit, throttle, safeguard, quota, traffic** -> `use-cases/platform/traffic-control.md`
- **deploy, Docker, Helm, Kubernetes, install** -> `use-cases/platform/deploy-conduktor.md`
- **conduktor CLI, apply, GitOps, CI/CD, pipeline, automation** -> `use-cases/platform/gitops-automation.md`
- **GitHub Actions, CODEOWNERS, workflow, token scope, ResourcePolicy examples, onboarding app** -> `use-cases/platform/self-service-github-cicd.md`
- **bootstrap, self-service, adopt, migrate, ownership, reverse-engineer** -> `use-cases/platform/bootstrap-self-service-cli.md`
- **Terraform, IaC, HCL, provider** -> `use-cases/platform/terraform.md`
- **onboard, connect, credentials, getting started, bootstrap** -> `use-cases/app-developer/onboard-to-kafka.md`
- **create topic, new topic, self-service topic** -> `use-cases/app-developer/create-topic.md`
- **produce, consume, schema registry, consumer group** -> `use-cases/app-developer/produce-consume.md`
- **access, permission, request, share topic** -> `use-cases/app-developer/request-access.md`
