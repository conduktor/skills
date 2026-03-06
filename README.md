# Conduktor Agent Skills

Agent Skills for AI coding assistants — teaches your agent the Conduktor platform.

Skills follow the [Agent Skills](https://agentskills.io/) format and work with Claude Code, Cursor, VS Code (Copilot), Gemini CLI, and [30+ other tools](https://agentskills.io).

## Available Skills

### conduktor

Conduktor platform expertise for Apache Kafka management, governance, and self-service. Covers Console, Gateway, and CLI.

**Use when:**
- Deploying Console and Gateway (Docker, Helm, Kubernetes)
- Configuring Gateway interceptors (encryption, data quality, traffic control)
- Setting up multi-tenancy with virtual clusters
- Automating with CLI, GitOps, or Terraform
- Onboarding application developers to Conduktor-managed Kafka
- Creating topics, requesting access, or connecting through Gateway

**Covers two personas:**

| Platform engineer | Application developer |
|---|---|
| Deploy Console & Gateway | Connect to Kafka through Gateway |
| Encrypt or mask Kafka data | Create topics via self-service |
| Enforce data quality rules | Produce & consume through Gateway |
| Set up multi-tenancy | Request access to other teams' topics |
| Apply traffic control & safeguards | |
| Automate with CLI & GitOps | |
| Manage infrastructure with Terraform | |

## Installation

```bash
npx skills add conduktor/skills
```

## Usage

Once installed, just work normally. When you mention Conduktor topics, your agent loads the relevant knowledge automatically.

**Examples:**

| You ask | Agent loads |
|---------|------------|
| "Deploy Conduktor with Docker Compose" | `deploy-conduktor.md` |
| "Encrypt PII fields in Kafka messages" | `encrypt-kafka-data.md` |
| "Set up virtual clusters for team isolation" | `multi-tenancy.md` |
| "Create a topic through self-service" | `create-topic.md` |
| "How do I connect my app to Gateway?" | `onboard-to-kafka.md` |
| "Write Terraform for Conduktor resources" | `terraform.md` |

The skill uses progressive disclosure: only the `SKILL.md` metadata (~100 tokens) is loaded at startup. Use-case files are pulled in on demand based on your conversation.

## Skill Structure

```
skills/
└── conduktor/
    ├── SKILL.md                          # Entry point (auto-discovered)
    ├── references/
    │   ├── mental-model.md               # Product boundaries, resource model
    │   ├── anti-patterns.md              # Common AI assistant mistakes
    │   └── terminology.md                # Conduktor-to-Kafka mapping
    └── use-cases/
        ├── platform/                     # Platform engineer use cases
        │   ├── deploy-conduktor.md
        │   ├── encrypt-kafka-data.md
        │   ├── enforce-data-quality.md
        │   ├── multi-tenancy.md
        │   ├── traffic-control.md
        │   ├── gitops-automation.md
        │   └── terraform.md
        └── app-developer/                # Application developer use cases
            ├── onboard-to-kafka.md
            ├── create-topic.md
            ├── produce-consume.md
            └── request-access.md
```

## License

[Apache-2.0](skills/conduktor/LICENSE)
