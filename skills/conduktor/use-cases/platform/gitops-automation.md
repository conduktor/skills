# GitOps and automation with the Conduktor CLI

## Agent workflow

1. Run `conduktor token list admin` to verify CLI auth is configured
2. If not authenticated, help configure `CDK_BASE_URL` + `CDK_API_KEY` (Console) or `CDK_GATEWAY_BASE_URL` + `CDK_GATEWAY_USER`/`CDK_GATEWAY_PASSWORD` (Gateway)
3. Ask what to automate: export existing state, set up CI/CD pipeline, or manage resources declaratively
4. To export: run `conduktor get all --console -o yaml` or `conduktor get all --gateway -o yaml` to dump current state
5. To set up CI/CD: ask which platform (GitHub Actions, GitLab CI, etc.) and generate pipeline config with `--dry-run` validation step
6. Generate resource YAML files organized by kind or by team
7. Offer to run `conduktor apply -f --dry-run` to validate before applying

Manage Conduktor Console and Gateway resources declaratively using the `conduktor` CLI (Go binary). Apply YAML definitions from Git, track state, and let CI/CD reconcile drift automatically.

## When to use this

- Infrastructure-as-code for Kafka topics, users, groups, interceptors, virtual clusters.
- CI/CD pipelines that provision or update Conduktor resources on merge.
- Multi-environment promotion (dev/staging/prod) with isolated state per environment.
- Drift detection and orphan cleanup via state management.

## Installation

From GitHub releases: download the binary for your OS from <https://github.com/conduktor/ctl/releases>.

Docker:

```bash
docker pull conduktor/conduktor-ctl:latest
```

From source (Go 1.22+):

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

Set all five variables. Use `--console` / `--gateway` flags on `get` to scope queries.

### TLS

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
conduktor apply -f resource.yaml --dry-run --print-diff
conduktor apply -f ./configs -r --parallelism 8
```

| Flag | Description |
|------|-------------|
| `-f, --file` | File or folder path (required, repeatable) |
| `-r, --recursive` | Recurse into subfolders for `.yaml/.yml` files |
| `--dry-run` | Validate without applying |
| `--print-diff` | Show diff between current and desired state |
| `--parallelism` | Parallel operations, 1-100 (default 1) |
| `--enable-state` | Enable state tracking |
| `--state-file` | Custom local state file path |
| `--state-remote-uri` | Remote state URI (S3/GCS/Azure) |

### get

Retrieve resources.

```bash
conduktor get all
conduktor get topics
conduktor get User alice@mycompany.io
conduktor get Groups -o json
conduktor get all --gateway
conduktor get all --console
```

| Flag | Description |
|------|-------------|
| `-o, --output` | Output format: `yaml`, `json`, `name` (default `yaml`) |
| `--gateway` | Query Gateway resources only |
| `--console` | Query Console resources only |

### delete

Remove resources.

```bash
conduktor delete -f resource.yaml
conduktor delete topic my-topic
conduktor delete -f ./configs -r --dry-run
```

| Flag | Description |
|------|-------------|
| `-f, --file` | File or folder path |
| `-r, --recursive` | Recurse into subfolders |
| `--dry-run` | Validate without deleting |
| `--enable-state` | Enable state tracking |
| `--state-file` | Custom local state file path |

### edit, template, login, token, sql

| Command | Purpose | Example |
|---------|---------|---------|
| `edit` | Open resource in `$EDITOR`, apply on save | `conduktor edit topic my-topic` |
| `template` | Emit YAML scaffold for a resource kind | `conduktor template Topic -o topic.yaml -e -a` |
| `login` | Exchange `CDK_USER`/`CDK_PASSWORD` for a JWT | `conduktor login` |
| `token` | List admin or application-instance tokens | `conduktor token list admin` |
| `sql` | Query indexed topics via SQL | `conduktor sql "SELECT * FROM t LIMIT 10"` |

Global flags on every command: `-v` (debug), `-vv` (trace), `--permissive` (ignore undefined env vars).

## Resource YAML format

```yaml
apiVersion: kafka/v2
kind: Topic
metadata:
  name: my-topic
  cluster: my-cluster
  labels:
    conduktor.io/application: app-a
spec:
  partitions: 3
  replicationFactor: 3
  configs:
    cleanup.policy: delete
    retention.ms: "86400000"
```

Multiple resources per file separated by `---`. The CLI processes all `.yaml/.yml` files when using `-f <folder>`.

## State management

### Enable

Disabled by default. Activate per-command or globally:

```bash
# flag
conduktor apply -f res.yaml --enable-state

# env var
export CDK_STATE_ENABLED=true
```

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

When state is enabled, `apply` compares the resource list in the YAML files against the stored state. Resources present in state but absent from files are treated as orphans and deleted automatically. This is how the CLI achieves declarative convergence.

## CI/CD patterns

For the full self-service GitHub repo pattern (two-workflow structure, CODEOWNERS, token scoping, ResourcePolicy examples, onboarding checklist), see [self-service-github-cicd-cli.md](self-service-github-cicd-cli.md).

### Simple single-workflow example

For non-self-service use cases (e.g., managing Gateway interceptors or Console resources without the two-folder pattern):

```yaml
name: Apply Conduktor resources
on:
  push:
    branches: [main]
    paths: ["conduktor/**"]

jobs:
  apply:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Install CLI
        run: |
          curl -sL https://github.com/conduktor/ctl/releases/latest/download/conduktor-linux-amd64 -o conduktor
          chmod +x conduktor && sudo mv conduktor /usr/local/bin/

      - name: Apply
        env:
          CDK_BASE_URL: ${{ secrets.CDK_BASE_URL }}
          CDK_API_KEY: ${{ secrets.CDK_API_KEY }}
          CDK_STATE_REMOTE_URI: "s3://state-bucket/conduktor/prod/?region=us-east-1"
        run: conduktor apply -f conduktor/ --recursive --enable-state
```

### Dry-run validation

Run on pull requests to catch errors before merge:

```yaml
- name: Dry run
  env:
    CDK_BASE_URL: ${{ secrets.CDK_BASE_URL }}
    CDK_API_KEY: ${{ secrets.CDK_API_KEY }}
  run: conduktor apply -f conduktor/ --recursive --dry-run --print-diff
```

## Common mistakes

| Mistake | Fix |
|---------|-----|
| Forgetting `--enable-state` -- orphans never cleaned | Set `CDK_STATE_ENABLED=true` globally in CI env |
| Running concurrent applies against the same state file | Serialize jobs or use separate state paths per resource set |
| Using `--state-file` in CI instead of remote backend | Use `--state-remote-uri`; local files are lost between runs |
| Missing `-r` when resources live in subdirectories | Always pass `--recursive` with folder paths |
| Hardcoding secrets in YAML | Use env vars and CI secret stores; never commit `CDK_API_KEY` |
| Skipping `--dry-run` on PR pipelines | Always validate before merge to catch schema errors early |
