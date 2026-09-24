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
  version: "1.1"
---

## Conduktor Platform

Conduktor is three products that work together around Apache Kafka:

- **Console** (`conduktor/conduktor-console`, port 8080) observes and manages Kafka clusters. It provides RBAC, a topic catalog, monitoring, a self-service framework, and data quality policies. Requires PostgreSQL. Without a license it runs as Community Edition: no self-service or data quality policies, and capped users, clusters, alerts and masking policies.
- **Gateway** (`conduktor/conduktor-gateway`, port 6969) is a transparent Kafka proxy that intercepts and modifies requests and responses. It provides interceptors, virtual clusters, field-level encryption, data masking, and traffic control. Clients connect to Gateway instead of Kafka directly. It won't start without `GATEWAY_LICENSE_KEY` (3.18+).
- **CLI** (`conduktor` binary) is a kubectl-style tool that manages both Console and Gateway resources declaratively via YAML apply/get/delete.

Console and Gateway are separate services. Console can manage Gateway, but they deploy independently.

Version baseline: this skill was written for Console 1.43–1.45, Gateway 3.17 and CLI 0.7. Guardrails and command forms were re-verified on Console 1.47.2, Gateway 3.21.1, CLI 0.9.2 and Terraform provider 1.5.1 (2026-09). For version-sensitive details (env vars, plugin options, features added since), confirm with the docs MCP.

## Always read first

Before running any command that changes state, read [references/guardrails.md](references/guardrails.md). It covers the preflight, the error playbook, the CLI-state and Terraform data-loss traps, secrets handling, and the command forms that work.

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
| Set up self-service GitHub CI/CD (workflows, CODEOWNERS, tokens, policies) — based on the official [conduktor/self-service-template](https://github.com/conduktor/self-service-template) | [use-cases/platform/self-service-github-cicd-cli.md](use-cases/platform/self-service-github-cicd-cli.md) |
| Bootstrap self-service from existing clusters, topics, and permissions (generates a repo matching the [official template](https://github.com/conduktor/self-service-template)) | [use-cases/platform/bootstrap-self-service-cli.md](use-cases/platform/bootstrap-self-service-cli.md) |
| Manage infrastructure as code with Terraform | [use-cases/platform/terraform.md](use-cases/platform/terraform.md) |

### Application developer

| I want to... | Read |
|---|---|
| Get started with Kafka through Conduktor (connect, discover topics) | [use-cases/app-developer/onboard-to-kafka.md](use-cases/app-developer/onboard-to-kafka.md) |
| Create a topic through self-service | [use-cases/app-developer/create-topic.md](use-cases/app-developer/create-topic.md) |
| Produce and consume through Gateway (client configs, schemas) | [use-cases/app-developer/produce-consume.md](use-cases/app-developer/produce-consume.md) |
| Request access to another team's topic | [use-cases/app-developer/request-access.md](use-cases/app-developer/request-access.md) |

## Agent behavior

**IMPORTANT: Discovery comes first. The skill files plus CLI discovery give you the user's real state. Use the docs MCP (step 5) for what they don't cover and for version-sensitive details.**

When a user asks about Conduktor, do not just explain how things work. Be an active assistant:

1. **Discover first with CLI** — run the preflight ([guardrails.md](references/guardrails.md) §1: `conduktor run whoami`, license plan), read the matching use-case file, then run the `conduktor get` commands from its "Agent workflow" section. The CLI gives you real state; docs give you generic examples.
2. **Ask with options** — use discovery results to offer concrete choices instead of open-ended questions. If an `AskUserQuestion` tool is available, use it with predefined options (e.g. topic name suggestions, partition counts, retention presets). Ask one question at a time, not a list of numbered questions. Prefer sensible defaults — only ask when the choice genuinely matters.
3. **Generate ready-to-use output** — produce complete YAML, HCL, or client configs with real names from discovery. Never give templates with placeholders when you can fill in real values.
4. **Execute with confirmation** — run `conduktor apply -f <file> --dry-run` first and read its output: a license 403, or a `Deleted (dry-run)` line for a resource you meant to keep, means stop ([guardrails.md](references/guardrails.md) §2–3). Then `conduktor apply -f <file>` on approval. For Terraform, run `terraform plan`, stop on any replace/destroy of an existing resource, then `terraform apply`.
5. **Look up what you don't know** — after checking skill files and CLI discovery, or right away for version-sensitive details, use the `search_conduktor_documentation` tool (Conduktor's MCP docs server) if you have it. Never invent env var names, config fields, or API endpoints that are not in these files or the docs. If the MCP tool is not available, tell the user to add it with this config:
   ```json
   { "mcpServers": { "conduktor-docs": { "type": "url", "url": "https://docs.conduktor.io/mcp" } } }
   ```

The CLI requires auth. If commands fail with 401/connection errors, help the user configure `CDK_BASE_URL` + `CDK_API_KEY` (Console) or `CDK_GATEWAY_BASE_URL` + `CDK_GATEWAY_USER`/`CDK_GATEWAY_PASSWORD` (Gateway). A 403 saying the feature is not available with your license is a plan limit, not an auth problem.

## Intent routing

When a user mentions these keywords, load the corresponding file:

- **error, 403, license, failed, deleted, data loss, state, dry-run** -> `references/guardrails.md`
- **encrypt, mask, PII, GDPR, shield** -> `use-cases/platform/encrypt-kafka-data.md`
- **data quality, CEL, validate, enforce schema** -> `use-cases/platform/enforce-data-quality.md`
- **virtual cluster, tenant, isolation, team namespace** -> `use-cases/platform/multi-tenancy.md`
- **rate limit, throttle, safeguard, quota, traffic** -> `use-cases/platform/traffic-control.md`
- **deploy, Docker, Helm, Kubernetes, install** -> `use-cases/platform/deploy-conduktor.md`
- **conduktor CLI, apply, GitOps, CI/CD, pipeline, automation** -> `use-cases/platform/gitops-automation.md`
- **GitHub Actions, CODEOWNERS, workflow, token scope, ResourcePolicy examples, onboarding app** -> `use-cases/platform/self-service-github-cicd-cli.md`
- **bootstrap, self-service, adopt, migrate, ownership, reverse-engineer** -> `use-cases/platform/bootstrap-self-service-cli.md`
- **Terraform, IaC, HCL, provider** -> `use-cases/platform/terraform.md`
- **onboard, connect, credentials, getting started, bootstrap** -> `use-cases/app-developer/onboard-to-kafka.md`
- **create topic, new topic, self-service topic** -> `use-cases/app-developer/create-topic.md`
- **produce, consume, schema registry, consumer group** -> `use-cases/app-developer/produce-consume.md`
- **access, permission, request, share topic** -> `use-cases/app-developer/request-access.md`
