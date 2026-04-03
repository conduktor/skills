# Bootstrap self-service from existing Console resources

## Agent workflow

1. Run `conduktor token list admin` to verify CLI auth with an AdminToken
2. If not authenticated, help configure `CDK_BASE_URL` + `CDK_API_KEY`
3. Export the full Console state — run all discovery commands in parallel:
   - `conduktor get KafkaCluster -o yaml` — clusters and their IDs
   - `conduktor get Topic -o yaml` — all topics across clusters
   - `conduktor get Group -o yaml` — Console groups with their cluster-scoped permissions
   - `conduktor get User -o yaml` — users and their group memberships
   - `conduktor get ServiceAccount -o yaml` — service accounts and their Kafka ACLs
4. Analyze the exported resources and present findings to the user:
   - **Clusters**: list each cluster ID and its topic count
   - **Topic ownership candidates**: group topics by naming prefix (e.g., `payments.*`, `inventory.*`) and propose each prefix group as a candidate Application
   - **Service account mapping**: for each service account, list which topic prefixes its ACLs cover — these map to ApplicationInstance `serviceAccount` fields
   - **Group permissions**: for each Console Group, list which clusters and topic patterns it has access to — these inform ApplicationGroup definitions
   - **Cross-team consumption**: identify service accounts or groups that have READ access to topics outside their primary prefix — these become ApplicationInstancePermission candidates
5. Ask the user to confirm or adjust the proposed application boundaries:
   - Which prefix groups should be merged or split?
   - What should each Application be named?
   - Which environments exist (dev/stag/prod) and how do they map to clusters?
   - Which Console Groups map to which Applications?
6. Ask the user for a target directory (default: `conduktor-self-service/`)
7. Generate the complete self-service resource set:
   - `Application` for each confirmed team/service boundary
   - `ApplicationInstance` for each app/cluster combination with resource prefixes derived from topic naming, service account from ACL analysis, and env labels from cluster mapping
   - `ApplicationInstancePermission` for each cross-team consumption pattern discovered
   - `ApplicationGroup` for each Console Group mapped to an Application, with permissions scoped to the correct ApplicationInstance
   - `ResourcePolicy` stubs based on observed topic configurations (partition ranges, replication factors, naming patterns)
   - `Topic` resources for each discovered topic, placed in the correct `applications/<app>/<env>/` folder
8. Generate supporting files: `README.md`, `.github/CODEOWNERS`, `.github/workflows/apply-platform.yml`, `.github/workflows/apply-apps.yml`
9. Present a summary of everything generated and offer to review any file
10. Offer to dry-run the platform resources: `conduktor apply -f platform/ -r --dry-run`

## When to use this

- Existing Conduktor Console deployment with clusters, topics, groups, and service accounts — but no self-service resources yet (no Applications, ApplicationInstances, or ResourcePolicies).
- Platform team wants to adopt the self-service model and needs to reverse-engineer ownership boundaries from the current state.
- Migrating from manual Console RBAC to the declarative self-service framework with GitOps.

## How it works

### 1. Discovery: what the CLI tells you

Each `conduktor get` command reveals a different ownership signal:

| Command | What it reveals | Ownership signal |
|---|---|---|
| `conduktor get KafkaCluster -o yaml` | Cluster IDs and configs | Environment boundaries (dev-cluster, prod-cluster) |
| `conduktor get Topic -o yaml` | All topics with `metadata.cluster` | Naming prefixes reveal team ownership (`payments.*`, `orders.*`) |
| `conduktor get ServiceAccount -o yaml` | SAs with Kafka ACLs per cluster | ACL prefixes show which SA owns which topics |
| `conduktor get Group -o yaml` | Console Groups with cluster-scoped permissions | Which humans can see/manage which topics |
| `conduktor get User -o yaml` | Users and their group memberships | Validates group-to-team mapping |

### 2. Inferring ownership from topic prefixes

Topics typically follow a naming convention like `<team>.<domain>` or `<service>.<entity>`. Group topics by their first segment to find natural ownership boundaries:

```
payments.transactions       ─┐
payments.settlements        ─┤── candidate Application: "payments"
payments.refunds            ─┘

inventory.stock-levels      ─┐
inventory.reservations      ─┤── candidate Application: "inventory"
inventory.warehouse-events  ─┘

notifications.email         ─┐
notifications.sms           ─┤── candidate Application: "notifications"
notifications.push          ─┘
```

Not all topics fit cleanly. Shared topics, legacy names, or flat naming schemes need the user to manually assign ownership.

### 3. Correlating service accounts to applications

Service account ACLs are the strongest ownership signal. A service account with WRITE ACLs on `payments.*` topics is the natural `serviceAccount` for the payments ApplicationInstance:

```
SA "sa-payments-prod" has ACLs:
  WRITE on Topic PREFIXED "payments."     → owner of payments topics on prod
  READ  on ConsumerGroup PREFIXED "payments."

SA "sa-notifications-prod" has ACLs:
  WRITE on Topic PREFIXED "notifications."  → owner of notifications topics on prod
  READ  on Topic PREFIXED "payments."       → cross-team consumer (→ ApplicationInstancePermission)
```

When a service account has READ-only ACLs on a prefix owned by another team, that signals a cross-team consumption pattern that should become an `ApplicationInstancePermission`.

### 4. Mapping Console Groups to ApplicationGroups

Console Groups define UI permissions. Each group's cluster-scoped permissions indicate which Application it belongs to:

```
Group "payments-developers":
  dev-cluster: topicViewConfig, topicConsume, topicProduce on "payments.*"
  prod-cluster: topicViewConfig, topicConsume on "payments.*"

→ ApplicationGroup "payments-developers-dev"  (full access in dev)
→ ApplicationGroup "payments-developers-prod" (read-only in prod)
```

If a group has permissions spanning multiple prefix groups, it may be an admin/platform group rather than an application group. Flag these for the user to decide.

### 5. Mapping clusters to environments

Ask the user how clusters map to environments. Common patterns:

| Cluster ID | Environment |
|---|---|
| `dev-cluster` | `dev` |
| `staging-cluster` or `stag-cluster` | `stag` |
| `prod-cluster` or `production` | `prod` |

This mapping determines the `metadata.labels.env` on each ApplicationInstance and the folder structure under `applications/<app>/<env>/`.

### 6. Generated self-service resources

#### Application (one per team/service)

```yaml
apiVersion: self-serve/v1
kind: Application
metadata:
  name: payments
  labels:
    business-unit: <ask-user>
spec:
  title: "Payments"
  owner: "payments-group"   # Console Group technical-id that owns this app
```

#### ApplicationInstance (one per app per cluster/env)

```yaml
apiVersion: self-serve/v1
kind: ApplicationInstance
metadata:
  name: payments-prod
  application: payments
  labels:
    env: prod
spec:
  cluster: prod-cluster
  serviceAccount: sa-payments-prod    # from SA ACL analysis
  resources:
    - type: TOPIC
      name: "payments."
      patternType: PREFIXED
    - type: CONSUMER_GROUP
      name: "payments."
      patternType: PREFIXED
    - type: SUBJECT
      name: "payments."
      patternType: PREFIXED
```

Resource prefixes are derived from the topic naming analysis. Consumer group and subject prefixes default to matching the topic prefix unless the SA ACLs indicate otherwise.

#### ApplicationInstancePermission (for cross-team consumers)

```yaml
apiVersion: self-serve/v1
kind: ApplicationInstancePermission
metadata:
  application: payments
  appInstance: payments-prod
  name: payments-prod-to-notifications-prod
spec:
  resource:
    type: TOPIC
    name: "payments."
    patternType: PREFIXED
  userPermission: READ
  serviceAccountPermission: READ
  grantedTo: notifications-prod
```

Generated when a service account has READ ACLs on topics outside its own ownership prefix.

#### ApplicationGroup (from Console Group permissions)

```yaml
apiVersion: self-serve/v1
kind: ApplicationGroup
metadata:
  application: payments
  name: payments-developers-prod
spec:
  displayName: "Payments Developers (prod)"
  permissions:
    - appInstance: payments-prod
      resourceType: TOPIC
      patternType: LITERAL
      name: "*"
      permissions:
        - topicViewConfig
        - topicConsume
  externalGroups:
    - <idp-group-if-known>
```

Permissions are translated from the Console Group's cluster-scoped permissions. If the existing group uses SSO/LDAP, `externalGroups` is populated; otherwise the agent flags that the user should configure identity provider group mapping.

#### ResourcePolicy (stubs from observed topic configs)

The agent analyzes discovered topics to propose baseline policies:

```yaml
apiVersion: self-serve/v1
kind: ResourcePolicy
metadata:
  name: topic-naming
spec:
  targetKind: Topic
  rules:
    - condition: metadata.name.matches("^[a-z0-9-]+\\.[a-z0-9.-]+$")
      errorMessage: "Topic name must follow <app>.<descriptive-name> pattern"
```

Additional policy stubs are generated based on observed ranges (e.g., partition counts, replication factors, retention values) across discovered topics. These are marked as suggestions for the platform team to review and tune.

### 7. Generated repo structure and supporting files

```
conduktor-self-service/
├── .github/
│   ├── CODEOWNERS
│   └── workflows/
│       ├── apply-platform.yml
│       └── apply-apps.yml
├── platform/
│   ├── applications/
│   │   └── <name>.yml
│   ├── instances/
│   │   └── <name>.yml
│   ├── policies/
│   │   └── <name>.yml
│   └── exceptions/
│       └── <app>/
│           └── <env>/
│               └── topics.yml
├── applications/
│   └── <app>/
│       └── <env>/
│           ├── topics.yml
│           ├── application-groups.yml
│           └── instance-permissions.yml
└── README.md
```

The `platform/exceptions/` folder holds resources that legitimately need to bypass a ResourcePolicy (e.g., a topic requiring more partitions than the policy allows). These are applied by the platform workflow with an AdminToken, which skips policy validation. Exception files are only created when the user identifies specific resources that need policy overrides — the bootstrap process does not generate exceptions by default.

#### README content

The generated README includes:

- **Overview** — what this repo manages, the two-folder (`platform/` and `applications/`) structure, and the PR-based workflow
- **For platform teams** — how to onboard a new application (create Application + ApplicationInstance + GitHub Environment + CODEOWNERS entry), manage policies, approve exceptions
- **For application teams** — how to add/modify topics, subjects, connectors, application groups, and instance permissions within their folder; the PR workflow (dry-run on PR, apply on merge); how to request policy exceptions
- **GitHub Environments setup** — table of required environments and secrets (`CONDUKTOR_API_KEY` per app/env, `CONDUKTOR_BASE_URL`, `CONDUKTOR_STATE_URI`)
- **Discovered policies** — list of generated ResourcePolicies with their rules, so teams understand what constraints apply

### 8. Rollout strategy

Self-service resources should be applied in dependency order. Recommended sequence:

1. **ResourcePolicies** — define guardrails before anything else
2. **Applications** — create the logical groupings
3. **ApplicationInstances** — bind apps to clusters (this creates Kafka ACLs for the service accounts)
4. **Topics** — declare existing topics under self-service ownership (idempotent, no changes to Kafka)
5. **ApplicationGroups** — migrate Console UI permissions to self-service managed groups
6. **ApplicationInstancePermissions** — formalize cross-team access

The agent offers to `--dry-run` each step before applying.

## Common mistakes

| Mistake | Fix |
|---|---|
| Running discovery with an ApplicationInstanceToken instead of AdminToken | Bootstrap requires full visibility. Use an AdminToken for all `conduktor get` commands. |
| Assigning a topic to the wrong Application based on prefix alone | Validate with service account ACLs — the SA with WRITE access is the true owner. |
| Missing cross-team READ patterns | Check all SA ACLs for READ on prefixes they don't own. Each one needs an ApplicationInstancePermission. |
| Creating ApplicationInstances with overlapping resource prefixes on the same cluster | Resource prefixes must not overlap between ApplicationInstances on the same cluster. Resolve conflicts before applying. |
| Forgetting to set `serviceAccount` on ApplicationInstance | Each instance needs a unique SA per cluster. Conduktor creates Kafka ACLs for this SA automatically. |
| Applying ApplicationGroups before the Application and ApplicationInstance exist | Resources must be applied in dependency order: Application → ApplicationInstance → ApplicationGroup. |
| Treating Console Group permissions as the only ownership signal | Console Groups control UI access only. Service account ACLs are the authoritative Kafka-level ownership signal. |
| Not creating GitHub Environments before merging the first PR | The `apply-apps.yml` workflow selects a GitHub Environment by name. Missing environments cause the workflow to fail. |
