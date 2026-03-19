# AGENTS.md

Guidance for AI coding agents working with this repository.

## Repository Overview

A collection of Agent Skills for the Conduktor platform. Skills teach AI coding assistants about Console, Gateway, and CLI so they produce correct Conduktor configurations instead of guessing from generic Kafka knowledge.

## Structure

```
skills/
  {skill-name}/
    SKILL.md              # Required: entry point with frontmatter + intent routing
    references/           # Core concepts, anti-patterns, terminology
    use-cases/            # Task-specific guides organized by persona
```

## Naming Conventions

- Skill directory: `kebab-case`
- `SKILL.md`: always uppercase, always this exact filename
- Reference files: `kebab-case.md`
- Use-case files: `kebab-case.md`, organized under `platform/` or `app-developer/`

## Content Rules

- Every YAML example must use the correct `apiVersion` (`gateway/v2`, `self-serve/v1`, `kafka/v2`, `v2`, `v1`)
- Every interceptor `pluginClass` must be the exact fully-qualified class name from Conduktor docs
- Every CLI flag and env var must match the actual CLI documentation
- No hallucinated plugin names, field names, or API endpoints
- Keep `SKILL.md` under 100 lines — it's the router, not the content
- Keep use-case files under 300 lines — use progressive disclosure
- End every use-case file with a "Common mistakes" table

## Verification

Before merging changes, verify against source documentation:
- Gateway resources and interceptors: [Conduktor docs](https://docs.conduktor.io)
- CLI commands and flags: `conduktor --help` or CLI docs
- Terraform resources: [Terraform registry](https://registry.terraform.io/providers/conduktor/conduktor/latest/docs)
- Live documentation via MCP: query the Conduktor docs server if available (endpoint: `https://docs.conduktor.io/mcp`)

## Anti-Patterns to Avoid

See `skills/conduktor/references/anti-patterns.md` for the full list. The most common:
- Using generic Kafka answers for Conduktor-specific questions
- Inventing interceptor plugin class names
- Confusing Gateway auth (SASL/PLAIN) with Console auth (API key)
- Using `apiVersion: v2` for Topics (correct: `kafka/v2`)
