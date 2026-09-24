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

- Every YAML example must use the right version number for its kind (Topic `v2`, ServiceAccount `v1`…). The CLI and Console read only the digit; prefixes (`kafka/`, `self-serve/`…) are conventions. Never tell users to rewrite `apiVersion` strings on existing resources: under CLI state that deletes and recreates them
- Every interceptor `pluginClass` must be the exact fully-qualified class name registered in the current Gateway
- Every command in an "Agent workflow" section must run as written on the current CLI (path right after `-f`, `--cluster` on cluster-scoped kinds)
- Every CLI flag and env var must match the actual CLI
- Say which license a workflow needs when Community Edition or an unlicensed Gateway can't run it
- No hallucinated plugin names, field names, or API endpoints
- Keep `SKILL.md` under 100 lines — it's the router, not the content
- Keep use-case files under 300 lines — use progressive disclosure
- End every use-case file with a "Common mistakes" table

## Verification

Verify by execution and against source code first; use the docs second, since several docs examples are wrong. Before merging changes:
- CLI commands: run them with the current release binary. Flag and kind errors show up offline, with credentials unset: `env -i conduktor <cmd>` must fail only on missing `CDK_BASE_URL`, never on `unknown flag` or `required flag`.
- Gateway env config: `docker run` the current Gateway image with the example's env vars. A config Gateway accepts runs until the license check (exit 98 without a license); a rejected one exits earlier.
- Console YAML: apply it to a disposable Console (`--dry-run`, then for real). Self-service kinds need a license.
- Helm: `helm template` the published chart with the example's values.
- Terraform resources: provider source or [Terraform registry](https://registry.terraform.io/providers/conduktor/conduktor/latest/docs).
- Docs, for context: [Conduktor docs](https://docs.conduktor.io) or the MCP docs server (`https://docs.conduktor.io/mcp`).
- Update the version baseline in `SKILL.md` to the versions you verified against.

## Anti-Patterns to Avoid

See `skills/conduktor/references/anti-patterns.md` for the full list. The most common:
- Using generic Kafka answers for Conduktor-specific questions
- Inventing interceptor plugin class names
- Confusing Gateway auth (SASL/PLAIN) with Console auth (API key)
- Rewriting `apiVersion` or metadata of resources managed with CLI state (the CLI deletes and recreates them; see `references/guardrails.md`)
