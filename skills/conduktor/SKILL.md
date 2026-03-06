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
| Manage infrastructure as code with Terraform | [use-cases/platform/terraform.md](use-cases/platform/terraform.md) |

### Application developer

| I want to... | Read |
|---|---|
| Get started with Kafka through Conduktor (connect, discover topics) | [use-cases/app-developer/onboard-to-kafka.md](use-cases/app-developer/onboard-to-kafka.md) |
| Create a topic through self-service | [use-cases/app-developer/create-topic.md](use-cases/app-developer/create-topic.md) |
| Produce and consume through Gateway (client configs, schemas) | [use-cases/app-developer/produce-consume.md](use-cases/app-developer/produce-consume.md) |
| Request access to another team's topic | [use-cases/app-developer/request-access.md](use-cases/app-developer/request-access.md) |

## Intent routing

When a user mentions these keywords, load the corresponding file:

- **encrypt, mask, PII, GDPR, shield** -> `use-cases/platform/encrypt-kafka-data.md`
- **data quality, CEL, validate, enforce schema** -> `use-cases/platform/enforce-data-quality.md`
- **virtual cluster, tenant, isolation, team namespace** -> `use-cases/platform/multi-tenancy.md`
- **rate limit, throttle, safeguard, quota, traffic** -> `use-cases/platform/traffic-control.md`
- **deploy, Docker, Helm, Kubernetes, install** -> `use-cases/platform/deploy-conduktor.md`
- **conduktor CLI, apply, GitOps, CI/CD, pipeline, automation** -> `use-cases/platform/gitops-automation.md`
- **Terraform, IaC, HCL, provider** -> `use-cases/platform/terraform.md`
- **onboard, connect, credentials, getting started, bootstrap** -> `use-cases/app-developer/onboard-to-kafka.md`
- **create topic, new topic, self-service topic** -> `use-cases/app-developer/create-topic.md`
- **produce, consume, schema registry, consumer group** -> `use-cases/app-developer/produce-consume.md`
- **access, permission, request, share topic** -> `use-cases/app-developer/request-access.md`
