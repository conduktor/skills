# Self-service GitHub CI/CD

How to structure a GitHub repository for Conduktor self-service, where application teams manage their own Kafka resources through code while a platform team controls admin-level resources.

## Repository structure

```
conduktor-self-service/
├── .github/
│   ├── CODEOWNERS
│   └── workflows/
│       ├── apply-platform.yml
│       └── apply-apps.yml
├── platform/                  # Admin resources (platform team only)
│   ├── applications/          # Application + ApplicationInstance definitions
│   │   └── <app>/
│   │       ├── application.yml        # Application resource
│   │       └── <env>.yml              # ApplicationInstance per environment
│   ├── policies/              # ResourcePolicy rules
│   └── exceptions/            # Policy exception overrides
│       └── <app>/<env>/       # Applied with AdminToken to bypass policies
├── applications/              # App-managed resources (each team owns their folder)
│   └── <app>/
│       └── <env>/
│           ├── topics.yml
│           ├── subjects.yml
│           ├── connectors.yml
│           ├── application-groups.yml
│           └── instance-permissions.yml
└── README.md
```

- **`platform/`** — admin resources that define self-service boundaries. Only the platform team modifies these. Applied with an **AdminToken**.
- **`applications/`** — day-to-day Kafka resources and Console UI permissions that application teams own. Applied with scoped **ApplicationInstanceTokens**.
- **`platform/exceptions/`** — resources that legitimately need to bypass a ResourcePolicy. Applied by the platform workflow with an AdminToken, which skips policy validation. The application team opens a PR here; only the platform team can approve (via CODEOWNERS).

## Token types

| Token Type | Scope | Use in CI/CD |
|---|---|---|
| **AdminToken** | Full platform access | Platform workflow (`apply-platform.yml`) only |
| **ApplicationInstanceToken** | Scoped to a single application instance | Application workflows (`apply-apps.yml`) — one per app/env |

**Do not use AdminTokens for application workflows.** ApplicationInstanceTokens enforce that a team can only modify resources within their own application instance boundaries.

## GitHub Environments

Create a GitHub Environment for each scope. Each stores its own secrets and variables that the workflow maps to CLI env vars.

| GitHub Environment | Token Type | Description |
|---|---|---|
| `platform` | AdminToken | Applies platform resources |
| `<app>-<env>` (e.g. `payments-dev`) | ApplicationInstanceToken | Scoped to that app/env |

Each environment needs:
- `CDK_API_KEY` (secret) — AdminToken or ApplicationInstanceToken depending on scope
- `CDK_BASE_URL` (variable) — Console URL (per-environment if dev/stag/prod use different instances)
- `CDK_STATE_REMOTE_URI` (variable) — per-app/env remote state path (e.g. `s3://conduktor-state/payments/dev/`)
- `AWS_ROLE_ARN` (variable) — IAM role scoped to this environment's state path (see State isolation below)

For production environments, add deployment protection rules such as required reviewers or wait timers.

### State isolation

Each app/env gets its own state file and cloud IAM role. This prevents one application's workflow from reading or writing another application's state.

**AWS (S3):** Create an IAM role per app/env with a trust policy pinned to the GitHub Environment via OIDC, and an S3 policy scoped to that app's state prefix:

```json
{
  "Effect": "Allow",
  "Action": ["s3:GetObject", "s3:PutObject"],
  "Resource": "arn:aws:s3:::conduktor-state/payments/dev/*"
}
```

The workflows use GitHub OIDC federation (`aws-actions/configure-aws-credentials` with `role-to-assume`) — no static AWS keys.

**GCS / Azure Blob:** The same principle applies. Use Workload Identity Federation (GCS) or federated credentials (Azure) instead of OIDC, and scope the IAM/RBAC policy to the app's state prefix. The workflow steps will differ — replace the `aws-actions/configure-aws-credentials` step with `google-github-actions/auth` (GCS) or `azure/login` (Azure).

## CODEOWNERS

```
# Default: platform team owns everything
*                                  @org/platform-team

# Application teams override their own folders (last match wins)
/applications/payments/            @org/payments-team @org/platform-team
/applications/inventory/           @org/inventory-team @org/platform-team
```

Enable on `main`: require PR reviews, require Code Owner review, require status checks to pass.

## Platform workflow (`apply-platform.yml`)

Applies admin resources (applications, instances, policies, exceptions) using an AdminToken.

```yaml
name: Apply Platform Resources
on:
  push:
    branches: [main]
    paths: ['platform/**']
  pull_request:
    paths: ['platform/**']

permissions:
  id-token: write
  contents: read

jobs:
  apply-platform:
    runs-on: ubuntu-latest
    environment: platform
    steps:
      - uses: actions/checkout@v4

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.AWS_ROLE_ARN }}
          aws-region: us-east-1

      - name: Install Conduktor CLI
        run: |
          curl -sL https://github.com/conduktor/ctl/releases/latest/download/conduktor-linux-amd64 -o conduktor
          chmod +x conduktor
          sudo mv conduktor /usr/local/bin/

      - name: ${{ github.event_name == 'pull_request' && 'Plan' || 'Apply' }} platform resources
        run: |
          conduktor apply \
            -f platform/ -r \
            --parallelism 5 \
            --enable-state \
            ${{ github.event_name == 'pull_request' && '--dry-run' || '' }}
        env:
          CDK_API_KEY: ${{ secrets.CDK_API_KEY }}
          CDK_BASE_URL: ${{ vars.CDK_BASE_URL }}
          CDK_STATE_REMOTE_URI: ${{ vars.CDK_STATE_REMOTE_URI }}
```

## Application workflow (`apply-apps.yml`)

Detects the changed `applications/<app>/<env>/` folder from the git diff and applies with the correct scoped token. Each app/env has its own IAM role and state file for isolation. Fails if changes span multiple app/env combinations or target an unknown environment.

```yaml
name: Apply Application Resources
on:
  push:
    branches: [main]
    paths: ['applications/**']
  pull_request:
    paths: ['applications/**']

permissions:
  id-token: write
  contents: read

jobs:
  detect:
    runs-on: ubuntu-latest
    outputs:
      app: ${{ steps.find.outputs.app }}
      env: ${{ steps.find.outputs.env }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - id: find
        run: |
          if [ "${{ github.event_name }}" = "pull_request" ]; then
            BASE=${{ github.event.pull_request.base.sha }}
            HEAD=${{ github.event.pull_request.head.sha }}
          else
            BASE=${{ github.event.before }}
            HEAD=${{ github.sha }}
          fi

          TARGETS=$(git diff --name-only "$BASE" "$HEAD" -- applications/ \
            | awk -F'/' 'NF>=3 {print $2 "/" $3}' \
            | sort -u)

          if [ -z "$TARGETS" ]; then
            echo "::error::No valid app/env changes detected under applications/"
            exit 1
          fi

          COUNT=$(echo "$TARGETS" | wc -l)
          if [ "$COUNT" -ne 1 ]; then
            echo "::error::Changes must be scoped to a single app/env folder. Found: $TARGETS"
            exit 1
          fi

          APP=$(echo "$TARGETS" | cut -d/ -f1)
          ENV=$(echo "$TARGETS" | cut -d/ -f2)

          VALID_ENVS="dev stag prod"
          if ! echo "$VALID_ENVS" | grep -qw "$ENV"; then
            echo "::error::Unknown environment '$ENV'. Must be one of: $VALID_ENVS"
            exit 1
          fi

          echo "app=$APP" >> "$GITHUB_OUTPUT"
          echo "env=$ENV" >> "$GITHUB_OUTPUT"
          echo "Detected target: $APP/$ENV"

  apply:
    needs: detect
    runs-on: ubuntu-latest
    environment: ${{ needs.detect.outputs.app }}-${{ needs.detect.outputs.env }}
    steps:
      - uses: actions/checkout@v4

      # OIDC federation — each GitHub Environment has its own AWS_ROLE_ARN,
      # and the IAM role trust policy is pinned to that environment.
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.AWS_ROLE_ARN }}
          aws-region: us-east-1

      - name: Install Conduktor CLI
        run: |
          curl -sL https://github.com/conduktor/ctl/releases/latest/download/conduktor-linux-amd64 -o conduktor
          chmod +x conduktor
          sudo mv conduktor /usr/local/bin/

      - name: ${{ github.event_name == 'pull_request' && 'Plan' || 'Apply' }} resources
        run: |
          conduktor apply \
            -f "applications/${{ needs.detect.outputs.app }}/${{ needs.detect.outputs.env }}" \
            --parallelism 5 \
            --enable-state \
            ${{ github.event_name == 'pull_request' && '--dry-run' || '' }}
        env:
          CDK_API_KEY: ${{ secrets.CDK_API_KEY }}
          CDK_BASE_URL: ${{ vars.CDK_BASE_URL }}
          CDK_STATE_REMOTE_URI: ${{ vars.CDK_STATE_REMOTE_URI }}
```

## How the workflows work

**On pull request (plan):** `conduktor apply --dry-run` validates resources against the live instance. Policy violations surface here before merge.

**On push to main (apply):** The workflow selects the corresponding GitHub Environment, providing the scoped `CDK_API_KEY`. The CLI applies all YAML files, ordering resources by dependency. With `--enable-state`, resources removed from YAML are deleted from Conduktor.

## Policy violations and exceptions

Conduktor validates resources against any `ResourcePolicy` linked to the ApplicationInstance via `spec.policyRef`. If a rule fails:

```
Error applying Topic "orders.events":
  Policy "topic-naming" violated: Topic name must follow the pattern <app>.<descriptive-name>
```

The dry-run step catches these before merge. For legitimate exceptions, place the resource in `platform/exceptions/<app>/<env>/` — the platform workflow applies it with an AdminToken, bypassing policies.

## ResourcePolicy examples

See [references/resource-policy-examples.md](../../references/resource-policy-examples.md) for starter policies (topic-naming, topic-rules-dev, topic-rules-prod, subject-rules, connector-rules, appgroup-restrictions). Place these in `platform/policies/` and link via `spec.policyRef` on each ApplicationInstance.

## Onboarding a new application

**Platform team:**

1. Create `platform/applications/<app>/application.yml` (Application resource with `spec.owner` pointing to a Console Group)
2. Create `platform/applications/<app>/<env>.yml` for each environment (ApplicationInstance with cluster, serviceAccount, policyRef, resources)
3. Create an IAM role per app/env scoped to its state prefix (e.g. `s3://conduktor-state/<app>/<env>/`), with OIDC trust pinned to the GitHub Environment. For GCS use Workload Identity Federation; for Azure use federated credentials.
4. Create GitHub Environments (`<app>-<env>`) with:
   - `CDK_API_KEY` (secret) — ApplicationInstanceToken
   - `CDK_BASE_URL` (variable) — Console URL
   - `CDK_STATE_REMOTE_URI` (variable) — e.g. `s3://conduktor-state/<app>/<env>/`
   - `AWS_ROLE_ARN` (variable) — the IAM role from step 3
5. Add CODEOWNERS entry: `/applications/<app>/  @org/<app>-team @org/platform-team`
6. Grant the team repo write access

**Application team:**

1. Create `applications/<app>/<env>/` folders with `topics.yml`, `application-groups.yml`, etc.
2. Define resources within the boundaries set by the platform team (topic prefixes, policies)
3. Open a PR — dry-run validates. After review and merge, resources are applied automatically.

No workflow changes needed — the detection logic handles new applications automatically.

## Labels convention

| Label | Purpose | Example |
|---|---|---|
| `env` | Environment identifier | `dev`, `stag`, `prod` |
| `business-unit` | Organizational grouping | `finance`, `risk`, `logistics` |
| `confidentiality` | Data classification | `public`, `internal`, `restricted` |
| `team` | Owning team | `payments-team` |

## Generated README

When generating the repository, include a `README.md` at the root that explains the federated ownership model and how teams interact with the repo. Cover these sections:

### Key Concepts

Before diving in, understand the Conduktor self-service resource hierarchy:

1. **[Application](https://docs.conduktor.io/guide/reference/self-service-reference#application)** — a logical grouping representing a team or service (admin resource)
2. **[ApplicationInstance](https://docs.conduktor.io/guide/reference/self-service-reference#applicationinstance)** — links an Application to a specific Kafka cluster/environment, defines ownership, and creates service account and user permissions (admin resource)
3. **[ResourcePolicy](https://docs.conduktor.io/guide/reference/self-service-reference#resourcepolicy)** — CEL-based validation rules enforced at apply time (admin resource)
4. **[ApplicationInstancePermission](https://docs.conduktor.io/guide/reference/self-service-reference#applicationinstancepermission)** — grants another application instance access to your topics, enabling cross-team collaboration (app-managed resource)
5. **[ApplicationGroup](https://docs.conduktor.io/guide/reference/self-service-reference#applicationgroup)** — defines Console UI permissions for team members within an application (app-managed resource)
6. **[Topic](https://docs.conduktor.io/guide/reference/kafka-reference#topic), [Subject](https://docs.conduktor.io/guide/reference/kafka-reference#subject), [Connector](https://docs.conduktor.io/guide/reference/kafka-reference#connector)** — the actual Kafka resources teams manage day-to-day (app-managed resources)

Admin resources (`Application`, `ApplicationInstance`, `ResourcePolicy`) are managed exclusively by the platform team. Application teams manage their own Kafka resources within the boundaries the platform team has defined.

All resources follow a Kubernetes-style declarative format:

```yaml
apiVersion: self-serve/v1  # or kafka/v2 for Kafka resources
kind: <ResourceKind>
metadata:
  name: resource-name
  labels:
    key: value
spec:
  # Resource-specific fields
```

### Repo overview and structure

Explain the split: `platform/` is admin-level resources (Application, ApplicationInstance, ResourcePolicy) managed exclusively by the platform team with an AdminToken. `applications/<app>/<env>/` contains day-to-day Kafka resources (topics, subjects, connectors, application groups, instance permissions) owned by each application team using a scoped ApplicationInstanceToken. Include the directory tree from the [Repository structure](#repository-structure) section and a table mapping each directory to its owner, token type, and purpose.

### How CI/CD works

- **Pull requests** run `conduktor apply --dry-run` against the live Console instance. Policy violations surface before merge.
- **Merges to main** apply resources automatically. The platform workflow (`apply-platform.yml`) uses an AdminToken; the application workflow (`apply-apps.yml`) detects the changed `<app>/<env>` folder and selects the matching GitHub Environment for a scoped ApplicationInstanceToken.
- **Policy exceptions** go in `platform/exceptions/<app>/<env>/`. The platform workflow applies them with an AdminToken, bypassing policy validation. Application teams open the PR; only the platform team can approve (CODEOWNERS).

### Onboard a new application

Two checklists — platform team and application team:

**Platform team:**
1. Create `platform/applications/<app>/application.yml` (`spec.owner` → Console Group)
2. Create `platform/applications/<app>/<env>.yml` per environment (ApplicationInstance with cluster, serviceAccount, policyRef, resources)
3. Create an IAM role per app/env scoped to the state prefix (e.g., `s3://conduktor-state/<app>/<env>/`)
4. Create GitHub Environments (`<app>-<env>`) with `CDK_API_KEY`, `CDK_BASE_URL`, `CDK_STATE_REMOTE_URI`, `AWS_ROLE_ARN`
5. Add CODEOWNERS entry: `/applications/<app>/  @org/<app>-team @org/platform-team`

**Application team:**
1. Create `applications/<app>/<env>/topics.yml` with topics matching the ApplicationInstance resource prefix
2. Add `application-groups.yml` for Console UI permissions
3. Add `instance-permissions.yml` if cross-team topic access is needed (see [request-access](../app-developer/request-access.md))
4. Open a PR — dry-run validates against policies. After review and merge, resources apply automatically.

### Labels convention

Include the labels table from the [Labels convention](#labels-convention) section so teams know which labels to set on their resources.

## Common mistakes

| Mistake | Fix |
|---|---|
| Using AdminToken for application workflows | Use ApplicationInstanceTokens — they enforce app/env boundaries |
| Changes spanning multiple app/env folders in one PR | The `apply-apps.yml` workflow validates a single folder per PR. Split into separate PRs. |
| Not creating GitHub Environments before merging the first PR | The workflow selects a GitHub Environment by name. Missing environments cause failures. |
| Applying exceptions through the app workflow | Exceptions must go through `platform/exceptions/` and the platform workflow (AdminToken bypasses policies) |
| Missing CODEOWNERS entry for a new app | Without it, only the platform team can approve — the app team won't be listed as required reviewers for their own folder |
| README missing onboarding steps | Teams won't know how to get started. Always include both platform and application team checklists. |
