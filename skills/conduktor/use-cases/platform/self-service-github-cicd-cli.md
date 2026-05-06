# Self-service GitHub CI/CD

How to structure a GitHub repository for Conduktor self-service, where application teams manage their own Kafka resources through code while a platform team controls admin-level resources.

## Start from the official template

The canonical scaffolding lives at **<https://github.com/conduktor/self-service-template>**. It is maintained by Conduktor and updated as the platform evolves. Direct users to clone or fork it as the starting point — do not hand-write a repo from scratch when the template already exists.

### Agent: how to bootstrap a copy

Before running anything, check whether the GitHub CLI is available with `gh --version`. Then:

**If `gh` is available** — use it to create a new private repo from the template (cleanest: fresh history, repo created on GitHub in one step):

```
gh repo create my-org/conduktor-self-service \
  --template conduktor/self-service-template \
  --private --clone
```

**If `gh` is NOT available** — tell the user `gh` was not found and ask whether they'd like to fall back to a shallow `git clone` instead. Wait for approval before running. Once approved:

```
git clone --depth 1 https://github.com/conduktor/self-service-template.git my-repo
cd my-repo
rm -rf .git
git init && git add . && git commit -m "Bootstrap from conduktor/self-service-template"
```

Then ask the user to create the destination remote (GitHub UI or their VCS of choice) and push. Mention that installing `gh` (`brew install gh` on macOS, see <https://cli.github.com/> otherwise) gives a one-step alternative if they prefer.

If the user has neither `gh` nor `git` (rare), fall back to a tarball: `curl -L https://github.com/conduktor/self-service-template/archive/refs/heads/main.tar.gz | tar xz`.

Use this skill file to explain the resource model, adapt the template to a customer's setup, or generate a matching layout when bootstrapping from existing Console state (see [bootstrap-self-service-cli.md](bootstrap-self-service-cli.md)). For workflow YAML, CODEOWNERS, and starter ResourcePolicies, defer to the template — that is where they are kept current.

## Repository structure

The template ships with this layout. Any deviation should be a deliberate choice the platform team makes, not a default.

```
conduktor-self-service/
├── .github/
│   ├── CODEOWNERS
│   └── workflows/
│       ├── apply-platform.yml      # AdminToken — platform resources (excl. clusters)
│       ├── apply-clusters.yml      # AdminToken — cluster resources, scoped per instance
│       └── apply-apps.yml          # ApplicationInstanceToken — scoped per app/instance
├── applications/                   # App-managed resources (each team owns their folder)
│   └── <app>/
│       └── <instance>/
│           ├── topics.yml
│           ├── subjects.yml
│           ├── connectors.yml
│           ├── application-groups.yml       # Console UI permissions
│           └── instance-permissions.yml     # Cross-team topic access (owner-side)
├── platform/                       # Platform team resources only
│   ├── applications/
│   │   └── <app>/
│   │       ├── application.yml     # Application — owner is a Console Group
│   │       └── <instance>.yml      # ApplicationInstance per instance
│   ├── clusters/                   # KafkaCluster / KafkaConnectCluster definitions
│   │   └── <instance>/             # Applied with instance-scoped cluster credentials
│   ├── groups/                     # Console Groups (map external IdP groups → Console)
│   ├── policies/                   # ResourcePolicy rules
│   └── exceptions/                 # Policy exception overrides
│       └── <app>/<instance>/       # Applied with AdminToken to bypass policies
└── README.md
```

| Directory | Owner | Token Type | Purpose |
|---|---|---|---|
| `platform/applications/` | Platform team | AdminToken | Application + ApplicationInstance definitions |
| `platform/clusters/<instance>/` | Platform team | AdminToken (instance-scoped credentials) | KafkaCluster / KafkaConnectCluster per instance |
| `platform/groups/` | Platform team | AdminToken | Console Groups mirrored from external IdP |
| `platform/policies/` | Platform team | AdminToken | ResourcePolicy rules |
| `platform/exceptions/` | Platform team approves; app team authors | AdminToken | Policy exception overrides |
| `applications/<app>/<instance>/` | Application team | ApplicationInstanceToken | Day-to-day Kafka resources |

### About the `<instance>` slot

`<instance>` corresponds 1:1 to a Self-Service `ApplicationInstance` — a distinct cluster binding, service account, and (usually) policy set. The template uses `dev` and `prod` as examples, but the axis can be anything that warrants its own scope:

- **Environment** — `dev`, `stag`, `prod`
- **Region / data residency** — `prod-us-east`, `prod-eu-west` (latency, active-active DR, regional data laws)
- **Data classification** — `pii` vs `non-pii` for tighter ACL/encryption boundaries
- **Regulatory domain** — `sox`, `pci`, `hipaa`
- **Workload tier** — `critical`, `batch`, `analytics`
- **Tenant** (multi-tenant apps) — `tenant-acme`, `tenant-globex`
- **Cluster migration** — `legacy` vs `next-gen` during an upgrade

Do not assume `dev/stag/prod`. Ask the user what dimensions matter to them.

## Token types

| Token Type | Scope | Use in CI/CD |
|---|---|---|
| **AdminToken** | Full platform access | `apply-platform.yml` and `apply-clusters.yml` |
| **ApplicationInstanceToken** | Scoped to a single ApplicationInstance | `apply-apps.yml` — one per app/instance |

**Do not use AdminTokens for application workflows.** ApplicationInstanceTokens enforce that a team can only modify resources within their own instance boundaries.

## GitHub Environments

Each scope maps to a GitHub Environment with its own secrets and variables.

| GitHub Environment | Token Type | Used by | Description |
|---|---|---|---|
| `platform` | AdminToken | `apply-platform.yml` | Applies all `platform/` resources except `platform/clusters/` |
| `kafka-<instance>` (e.g. `kafka-prod`) | AdminToken + cluster credentials | `apply-clusters.yml` | Applies KafkaCluster/KafkaConnectCluster for that instance — needs cluster credential secrets |
| `<app>-<instance>` (e.g. `payments-prod`) | ApplicationInstanceToken | `apply-apps.yml` | Scoped to that app/instance |

Each environment needs:
- `CDK_API_KEY` (secret) — AdminToken or ApplicationInstanceToken depending on scope
- `CDK_BASE_URL` (variable) — Console URL
- `CDK_STATE_REMOTE_URI` (variable) — distinct remote state path (e.g. `s3://conduktor-state/payments/prod/`)
- `AWS_ROLE_ARN` (variable) — IAM role scoped to this environment's state path

`kafka-<instance>` environments additionally hold cluster credential secrets that the cluster YAML references via `${VAR}` placeholders: `KAFKA_BOOTSTRAP_SERVERS`, `KAFKA_CREDENTIALS`, `SR_USER`, `SR_PASSWORD`, `KAFKA_CONNECT_URL`, `KAFKA_CONNECT_USERNAME`, `KAFKA_CONNECT_PASSWORD`.

For production environments, add deployment protection rules (required reviewers, wait timers).

### State isolation

Each environment carries its own `CDK_STATE_REMOTE_URI` and `AWS_ROLE_ARN`. One workflow's state cannot be read or written by another. The IAM role's trust policy is pinned to its corresponding GitHub Environment via OIDC.

**AWS (S3):** Each app/instance gets an IAM role with a trust policy pinned to its GitHub Environment and an S3 policy scoped to its state prefix:

```json
{
  "Effect": "Allow",
  "Action": ["s3:GetObject", "s3:PutObject"],
  "Resource": "arn:aws:s3:::conduktor-state/payments/prod/*"
}
```

The workflows use GitHub OIDC federation (`aws-actions/configure-aws-credentials` with `role-to-assume`) — no static AWS keys.

**GCS / Azure Blob:** The same principle applies. Use Workload Identity Federation (GCS) or federated credentials (Azure) instead of OIDC, scoped to the app/instance state prefix.

## CODEOWNERS

```
# Default: platform team owns everything
*                                  @org/platform-team

# Application teams override their own folders (last match wins)
/applications/payments/            @org/payments-team @org/platform-team
/applications/inventory/           @org/inventory-team @org/platform-team
```

Enable on `main`: require PR reviews, require Code Owner review, require status checks to pass.

## How the workflows work

The three workflows in the template repo handle the three scopes:

- **`apply-platform.yml`** — triggered by changes under `platform/` (except `platform/clusters/`). Uses the `platform` GitHub Environment with an AdminToken. Applies Applications, ApplicationInstances, Groups, ResourcePolicies, and exceptions.
- **`apply-clusters.yml`** — triggered by changes under `platform/clusters/<instance>/`. Detects the changed instance from the diff, selects the matching `kafka-<instance>` GitHub Environment so cluster credential secrets resolve correctly, and applies with an AdminToken. Changes must be scoped to a single instance per PR.
- **`apply-apps.yml`** — triggered by changes under `applications/<app>/<instance>/`. Detects the changed app/instance from the diff, selects the matching `<app>-<instance>` GitHub Environment for a scoped ApplicationInstanceToken. Changes must be scoped to a single app/instance per PR.

**On pull request:** each workflow runs `conduktor apply --dry-run` against the live Console instance. Policy violations surface here before merge.

**On push to main:** the workflow applies the resources. With `--enable-state`, resources removed from YAML are deleted from Conduktor on the next apply.

The exact YAML lives in the template — do not duplicate it in this skill or in a generated repo. If a customer needs a workflow tweak (e.g., self-hosted runner labels, additional pre-apply steps), edit the file from the template, do not regenerate from scratch.

## Policy violations and exceptions

Conduktor validates resources against any `ResourcePolicy` linked via `spec.policyRef`. When a rule fails:

```
Error applying Topic "orders.events":
  Policy "topic-naming" violated: Topic name must follow the pattern <app>.<descriptive-name>
```

The dry-run step catches these before merge. For legitimate exceptions, place the resource in `platform/exceptions/<app>/<instance>/`. The platform workflow applies it with an AdminToken, bypassing policies. The application team opens the PR; only the platform team can approve (CODEOWNERS).

## ResourcePolicy examples

The template ships with starter policies in `platform/policies/`:

| Policy | Target | Description |
|---|---|---|
| `topic-naming` | Topic | Enforces `<app>.<descriptive-name>` naming |
| `topic-labels` | Topic | Requires `instance`, `business-unit`, `confidentiality`, `team` labels |
| `topic-rules-dev` | Topic | Dev rules (RF = 3, partitions 1–3) |
| `topic-rules-prod` | Topic | Strict prod rules (RF = 3, partitions ≤ 12, retention ≥ 1h, ISR ≥ 2) |
| `subject-rules` | Subject | Requires `-key` or `-value` suffix, explicit compatibility |
| `connector-rules` | Connector | Allowlists plugin classes, `tasks.max` ≤ 8 |
| `appgroup-restrictions` | ApplicationGroup | No direct members, read-only on prod topics |

For the YAML bodies and CEL syntax notes, see [references/resource-policy-examples.md](../../references/resource-policy-examples.md). Tune values to the customer's environment.

## Onboarding

### Platform bootstrap (one-time)

Before any application can be onboarded, the platform team sets up shared infrastructure:

1. Create `platform/clusters/<instance>/*.yml` for each Kafka and Kafka Connect cluster. Use `${VAR}` placeholders for credentials.
2. Create `platform/groups/*.yml` for each Console `Group` mirroring an external IdP group.
3. Seed `platform/policies/` with the ResourcePolicies you want enforced.
4. Create the `platform` GitHub Environment with `CDK_API_KEY` (AdminToken), `CDK_BASE_URL`, `CDK_STATE_REMOTE_URI`, `AWS_ROLE_ARN`.
5. Create a `kafka-<instance>` GitHub Environment for each cluster instance with the same four variables plus the cluster credential secrets (`KAFKA_BOOTSTRAP_SERVERS`, `KAFKA_CREDENTIALS`, `SR_USER`, `SR_PASSWORD`, `KAFKA_CONNECT_URL`, `KAFKA_CONNECT_USERNAME`, `KAFKA_CONNECT_PASSWORD`).

### Onboard a new application

**Platform team:**

1. Create `platform/applications/<app>/application.yml` (Application with `spec.owner` → Console Group)
2. Create `platform/applications/<app>/<instance>.yml` per instance (ApplicationInstance with cluster, serviceAccount, policyRef, resources)
3. Create an IAM role per app/instance scoped to its state prefix (e.g. `s3://conduktor-state/<app>/<instance>/`), with OIDC trust pinned to the GitHub Environment
4. Create GitHub Environments (`<app>-<instance>`) with:
   - `CDK_API_KEY` (secret) — ApplicationInstanceToken
   - `CDK_BASE_URL`, `CDK_STATE_REMOTE_URI`, `AWS_ROLE_ARN` (variables)
5. Add CODEOWNERS entry: `/applications/<app>/  @org/<app>-team @org/platform-team`
6. Grant the team repo write access

**Application team:**

1. Create `applications/<app>/<instance>/topics.yml` with topics matching the ApplicationInstance resource prefix
2. Add `application-groups.yml` for Console UI permissions
3. Add `instance-permissions.yml` if cross-team topic access is needed (see [request-access](../app-developer/request-access.md))
4. Open a PR — dry-run validates against policies. After review and merge, resources apply automatically.

No workflow changes needed — the detection logic handles new apps and instances automatically.

## Labels convention

| Label | Purpose | Example |
|---|---|---|
| `instance` | ApplicationInstance identifier | `dev`, `stag`, `prod`, `prod-us-east` |
| `business-unit` | Organizational grouping | `finance`, `risk`, `logistics` |
| `confidentiality` | Data classification | `public`, `internal`, `restricted` |
| `team` | Owning team | `payments-owners` |

Note: the template uses `instance` (matching the `ApplicationInstance` concept) rather than `env`, since an instance can correspond to environment, region, classification, tenant, etc. The `topic-labels` ResourcePolicy in [resource-policy-examples.md](../../references/resource-policy-examples.md) enforces this label.

## Generated README

When generating a repo from scratch (e.g. via [bootstrap-self-service-cli.md](bootstrap-self-service-cli.md)), use the template's [README.md](https://github.com/conduktor/self-service-template/blob/main/README.md) as the baseline. It already covers Key Concepts, Repository Structure, How CI/CD Works, State Isolation, Onboarding, Labels, and Included Resource Policies. Customize it with the customer's specific applications, instances, and Console URL — do not rewrite the structural sections.

## Common mistakes

| Mistake | Fix |
|---|---|
| Hand-rolling the repo instead of cloning the template | Use `gh repo create --template conduktor/self-service-template`. The template is maintained — your hand-rolled version drifts. |
| Using `<env>` folders instead of `<instance>` | The template uses `<instance>` to align with the `ApplicationInstance` resource. Folder name should match the instance label. |
| Skipping `apply-clusters.yml` and putting clusters in `apply-platform.yml` | Cluster resources need instance-scoped credentials (`KAFKA_BOOTSTRAP_SERVERS` etc.). Keeping them in a separate workflow with per-instance environments isolates those secrets. |
| Using AdminToken for application workflows | Use ApplicationInstanceTokens — they enforce app/instance boundaries |
| Changes spanning multiple app/instance folders in one PR | The detection logic validates a single folder per PR. Split into separate PRs. |
| Not creating GitHub Environments before merging the first PR | Workflows select environments by name. Missing environments cause failures. |
| Applying exceptions through the app workflow | Exceptions must go through `platform/exceptions/` and `apply-platform.yml` (AdminToken bypasses policies) |
| Missing CODEOWNERS entry for a new app | Without it, only the platform team can approve — the app team won't be listed as required reviewers for their own folder |
| Confusing `Group` (kind `Group`, `apiVersion: v2`) with `ApplicationGroup` (kind `ApplicationGroup`, `apiVersion: self-serve/v1`) | They are distinct resources. `Group` mirrors an IdP group into Console; `ApplicationGroup` grants Console UI access to members of a `Group` within an Application. |
