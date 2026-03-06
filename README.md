# Conduktor Agent Skills

Agent Skills for AI coding assistants — teaches your agent the Conduktor platform.

Skills follow the [Agent Skills](https://agentskills.io/) format and work with Claude Code, Cursor, VS Code (Copilot), Gemini CLI, and [30+ other tools](https://agentskills.io).

## Installation

```bash
npx skills add conduktor/skills
```

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

## Usage

Once installed, just ask. The agent doesn't just explain — it discovers your environment, asks the right questions, generates configs with real values, and executes.

**Examples:**

| You say | What the agent does |
|---------|---------------------|
| "Install Conduktor" | Checks Docker, asks quick test vs existing cluster, runs `docker compose up`, polls health, sets up CLI auth |
| "Encrypt PII fields in my Kafka topics" | Discovers your interceptors and topics via CLI, asks which fields, generates EncryptPlugin YAML, offers `conduktor apply` |
| "Set up team isolation" | Lists existing virtual clusters, asks how many teams, generates VirtualCluster + ServiceAccount + Group YAML |
| "I need access to the payments topic" | Discovers topics and your ApplicationInstance, identifies the owner, generates the permission YAML |
| "Create a topic for my service" | Checks your AppInstance's TopicPolicy constraints, asks topic name and config, validates, applies |
| "Write Terraform for our Conduktor setup" | Exports current resources via CLI, generates matching HCL, offers `terraform plan` |

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
