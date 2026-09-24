# GitOps and automation with the Conduktor CLI

## Agent workflow

1. Run `conduktor run whoami` to verify CLI auth. It exits 0 even on API errors, so read the output
2. If not authenticated, help configure `CDK_BASE_URL` + `CDK_API_KEY` (Console) or `CDK_GATEWAY_BASE_URL` + `CDK_GATEWAY_USER`/`CDK_GATEWAY_PASSWORD` (Gateway)
3. Ask what to automate: export existing state, set up CI/CD pipeline, or manage resources declaratively
4. To export: `conduktor get all --console -o yaml` and `conduktor get all --gateway -o yaml` return root kinds only. Also run `conduktor get Topic --cluster <id> -o yaml` (and Subject, ServiceAccount, KafkaConnectCluster) for each cluster. Read stderr: `get all` exits 0 even when some kinds fail. Redact credentials before writing files
5. To set up CI/CD: ask which platform (GitHub Actions, GitLab CI, etc.) and generate pipeline config with a stateful `--dry-run` step on PRs (see below)
6. Generate resource YAML files organized by kind or by team
7. Run `conduktor apply -f <file-or-dir> -r --dry-run` to validate before applying. If the repo uses state, dry-run with the same state and stop on any `Deleted (dry-run)` line you didn't intend ([guardrails](../../references/guardrails.md) §3)

Manage Conduktor Console and Gateway resources declaratively using the `conduktor` CLI (Go binary). Apply YAML definitions from Git from CI/CD, with state to clean up resources removed from Git.

## When to use this

- Infrastructure-as-code for Kafka topics, users, groups, interceptors, virtual clusters.
- CI/CD pipelines that provision or update Conduktor resources on merge.
- Multi-environment promotion (dev/staging/prod) with isolated state per environment.
- Orphan cleanup via state management. State stores identities, not specs, so it does not detect drift.

## Installation

From GitHub releases: <https://github.com/conduktor/ctl/releases>. Assets are archives, e.g. `conduktor-v0.9.2-linux-amd64.tar.gz`, containing a single `conduktor` binary. Homebrew: `brew install conduktor/brew/conduktor-cli`.

Docker (pin the tag):

```bash
docker pull conduktor/conduktor-ctl:v0.9.2
```

From source (Go 1.25+):

```bash
make build # produces ./conduktor
```

## Authentication

### Console only

```bash
export CDK_BASE_URL="https://console.example.com"
export CDK_API_KEY="<admin-token>"          # recommended
# or username/password:
# export CDK_USER="user" CDK_PASSWORD="pass"
```

Set `CDK_AUTH_MODE=external` when Console delegates auth to an API gateway (token or basic auth).

### Gateway only

```bash
export CDK_GATEWAY_BASE_URL="https://gateway.example.com"
export CDK_GATEWAY_USER="admin"
export CDK_GATEWAY_PASSWORD="conduktor"
```

### Dual setup

Set all five variables. `get all` accepts `--console` / `--gateway` to scope the dump; single-kind commands route automatically.

### TLS

These apply to the Console client only.

```bash
export CDK_CACERT="/path/to/ca.crt"       # custom CA
export CDK_CERT="/path/to/client.crt"      # client cert (e.g. Teleport)
export CDK_KEY="/path/to/client.key"       # client key
export CDK_INSECURE="true"                 # skip TLS verification
```

## Core commands

### apply

Upsert resources from YAML/JSON files.

```bash
conduktor apply -f resource.yaml
conduktor apply -f file1.yaml -f file2.yaml
conduktor apply -f ./configs --recursive
conduktor apply -f resource.yaml --dry-run
conduktor apply -f existing.yaml --dry-run --print-diff   # existing resources only
conduktor apply -f ./configs -r --parallelism 8
```

| Flag | Description |
|------|-------------|
| `-f, --file` | File or folder path (required, repeatable). The path must come right after `-f` |
| `-r, --recursive` | Recurse into subfolders for `.yaml/.yml` files |
| `--dry-run` | Validate server-side without applying |
| `--print-diff` | Show diff against the current resource. Errors (`could not find any matching resource`) if it doesn't exist yet, or was created seconds ago |
| `--parallelism` | Parallel operations, 1-100 (default 1) |
| `--enable-state` | Enable state tracking |
| `--state-file` | Custom local state file path |
| `--state-remote-uri` | Remote state URI (S3/GCS/Azure) |

### get

Retrieve resources.

```bash
conduktor get all
conduktor get Topic --cluster my-cluster
conduktor get User alice@mycompany.io
conduktor get Group -o json
conduktor get all --gateway
conduktor get all --console
```

Kind names are singular and exact (`topic` works as a lowercase alias; `topics` and `Groups` don't).

| Flag | Description |
|------|-------------|
| `-o, --output` | Output format: `yaml`, `json`, `name` (default `yaml`) |
| `--cluster` | Required for cluster-scoped kinds: Topic, Subject, ServiceAccount, KafkaConnectCluster (Connector also needs `--connectCluster`) |
| `--gateway` / `--console` | `get all` only: dump Gateway or Console root kinds |

### delete

Remove resources.

```bash
conduktor delete -f resource.yaml
conduktor delete Topic my-topic --cluster my-cluster
conduktor delete -f ./configs -r --dry-run
```

| Flag | Description |
|------|-------------|
| `-f, --file` | File or folder path |
| `-r, --recursive` | Recurse into subfolders |
| `--dry-run` | Print what would be deleted; no server call, no validation |
| `--enable-state` | Enable state tracking |
| `--state-file` / `--state-remote-uri` | Local or remote state location |

### run, edit, template, login, token, sql

| Command | Purpose | Example |
|---------|---------|---------|
| `run` | API actions: `whoami`, consumer groups, connectors, topics. Exits 0 on API errors | `conduktor run whoami` |
| `edit` | Open resource in `$EDITOR`, apply on save, with no dry-run and no state (interactive: not for agents) | `conduktor edit Topic my-topic --cluster my-cluster` |
| `template` | Emit a YAML scaffold for a kind. `-e` opens `$EDITOR`, `-a` applies on save: avoid both in agent workflows | `conduktor template Topic -o topic.yaml` |
| `login` | Print a JWT from `CDK_USER`/`CDK_PASSWORD`; not a credential check | `conduktor login` |
| `token` | List or create admin and application-instance tokens | `conduktor token create admin ci-bot` |
| `sql` | Query indexed topics via SQL | `conduktor sql "SELECT * FROM t LIMIT 10"` |

Global flags on every command: `-v` (debug), `-vv` (trace). `--permissive` has no effect in 0.9.2.

## Resource YAML format

```yaml
apiVersion: kafka/v2
kind: Topic
metadata:
  name: my-topic
  cluster: my-cluster
  labels:
    team: app-a            # conduktor.io/* labels are managed by Console and dropped
spec:
  partitions: 3
  replicationFactor: 3
  configs:
    cleanup.policy: delete
    retention.ms: "86400000"
```

Multiple resources per file separated by `---`. The CLI processes all `.yaml/.yml` files when using `-f <folder>`.

The CLI replaces `${VAR}` and `${VAR:-default}` in files before sending. It fails if `VAR` is unset or empty. Write `$${VAR}` to send a literal `${VAR}`, e.g. a secret that Gateway must resolve itself.

## State management

### Enable

Disabled by default. Enable it per command with `--enable-state`, or globally with `CDK_STATE_ENABLED=true`.

### Local state

Default location is OS-specific (`~/Library/Application Support/conduktor/cli-state.json` on macOS). Override with:

```bash
conduktor apply -f res.yaml --enable-state --state-file ./project-state.json
# or
export CDK_STATE_FILE=./project-state.json
```

### Remote backends (S3, GCS, Azure Blob)

```bash
# S3
--state-remote-uri "s3://bucket/path/?region=us-east-1"

# GCS
--state-remote-uri "gs://bucket/path/"

# Azure Blob
--state-remote-uri "azblob://container/path/"
```

Or set `CDK_STATE_REMOTE_URI` globally. Authentication uses standard provider mechanisms (AWS env vars / IAM role, `GOOGLE_APPLICATION_CREDENTIALS`, `AZURE_STORAGE_ACCOUNT` + key/SAS).

### Orphan detection

When state is enabled, `apply` compares the resource list in the YAML files against the stored state. Resources present in state but absent from files are treated as orphans and deleted automatically, before anything is applied. This is how the CLI achieves declarative convergence.

Three consequences that delete data:
- A resource's identity in state is its exact `apiVersion` string, `kind` and `metadata` (label values excepted). Editing a topic's `description` or `catalogVisibility`, adding a `labels` block, or rewriting `kafka/v2` as `v2` makes the CLI delete the topic, then recreate it empty. The exit code is still 0. Always dry-run with the same state first and stop on `Deleted (dry-run)` ([guardrails](../../references/guardrails.md) §3).
- The state covers everything applied through its location. Applying only some of the files against a shared state deletes the others. Use one state location per complete resource set.
- Deleting a YAML file deletes its resources on the next run, even when you only meant to hand them over to Terraform or the UI. See [guardrails](../../references/guardrails.md) §3 for a safe handover.

## CI/CD patterns

For the full self-service GitHub repo pattern (three-workflow structure, CODEOWNERS, token scoping, ResourcePolicy examples, onboarding checklist), see [self-service-github-cicd-cli.md](self-service-github-cicd-cli.md). The canonical scaffolding lives at [conduktor/self-service-template](https://github.com/conduktor/self-service-template) — start there rather than hand-rolling a repo.

### Simple single-workflow example

For non-self-service use cases (e.g., managing Gateway interceptors or Console resources without the two-folder pattern):

```yaml
name: Apply Conduktor resources
on:
  push:
    branches: [main]
    paths: ["conduktor/**"]

permissions:
  id-token: write   # OIDC to AWS for the remote state, no static keys
  contents: read

jobs:
  apply:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.AWS_ROLE_ARN }}
          aws-region: us-east-1

      - name: Install CLI
        run: |
          curl -sfL https://github.com/conduktor/ctl/releases/download/v0.9.2/conduktor-v0.9.2-linux-amd64.tar.gz \
            | sudo tar xz -C /usr/local/bin conduktor

      - name: Apply
        env:
          CDK_BASE_URL: ${{ secrets.CDK_BASE_URL }}
          CDK_API_KEY: ${{ secrets.CDK_API_KEY }}
          CDK_STATE_REMOTE_URI: "s3://state-bucket/conduktor/prod/?region=us-east-1"
        run: conduktor apply -f conduktor/ --recursive --enable-state
```

### Dry-run validation

Run on pull requests, with the same state as the main job, so the PR shows what the merge will delete:

```yaml
- name: Dry run
  env:
    CDK_BASE_URL: ${{ secrets.CDK_BASE_URL }}
    CDK_API_KEY: ${{ secrets.CDK_API_KEY }}
    CDK_STATE_REMOTE_URI: "s3://state-bucket/conduktor/prod/?region=us-east-1"
  run: |
    set -o pipefail
    conduktor apply -f conduktor/ --recursive --enable-state --dry-run 2>&1 | tee dry-run.txt
    if ! grep -q 'Loading state from remote storage' dry-run.txt; then
      echo "::error::The dry-run did not read the shared state. Check CDK_STATE_REMOTE_URI and the bucket credentials."
      exit 1
    fi
    if grep -q 'Deleted (dry-run)' dry-run.txt; then
      echo "::error::This PR deletes resources (or makes the CLI delete and recreate them). Review dry-run.txt."
      exit 1
    fi
```

The PR job needs the same bucket credentials as the apply job. Without them, the CLI either fails to load the state (and `| tee` hides the exit code unless `pipefail` is set) or silently falls back to an empty local state, and the dry-run then shows no deletions. The first `grep` catches both. A dry-run doesn't write the state. `--print-diff` fails for resources that don't exist yet, so keep it out of PR jobs.

## Common mistakes

| Mistake | Fix |
|---------|-----|
| Forgetting `--enable-state` -- orphans never cleaned | Set `CDK_STATE_ENABLED=true` globally in CI env |
| Editing metadata or `apiVersion` of resources already in state | The CLI deletes and recreates them (topics lose their data). Dry-run with state and stop on `Deleted (dry-run)` |
| Deleting YAML to hand resources over to Terraform or the UI | The next stateful apply deletes them. Switch to a new state location in the same commit, or edit the state with the pipeline paused ([guardrails](../../references/guardrails.md) §3) |
| Several jobs applying different file sets against one state location | Each run deletes what the other applied. Use one state location per complete resource set; serializing jobs doesn't help |
| Using `--state-file` in CI instead of remote backend | Use `--state-remote-uri`; local files are lost between runs |
| Missing `-r` when resources live in subdirectories | Always pass `--recursive` with folder paths |
| Hardcoding secrets in YAML | Use env vars and CI secret stores; never commit `CDK_API_KEY` |
| Skipping `--dry-run` on PR pipelines | Always validate before merge, with the same state, to catch schema errors and deletions early |
| Treating `get all` as a full export | It returns root kinds only and exits 0 on errors; loop `get Topic --cluster <id>` etc. per cluster and read stderr |
