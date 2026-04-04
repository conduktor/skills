# Bootstrap self-service from existing Console resources

## Agent workflow

1. Run `conduktor token list admin` to verify CLI auth with an AdminToken
2. If not authenticated, help configure `CDK_BASE_URL` + `CDK_API_KEY`
3. Export global Console resources and discover clusters:
   - `conduktor get all -c -o yaml` — global resources: KafkaClusters, Groups, and any existing self-service resources (does NOT include cluster-scoped resources like Topics, ServiceAccounts, Subjects, or Connectors)
   - Parse the output for `kind: KafkaCluster` entries to get all cluster IDs
4. Export cluster-scoped resources — for each cluster discovered in step 3, run in parallel:
   - `conduktor get Topic --cluster <cluster-id> -o yaml` — all topics on this cluster
   - `conduktor get ServiceAccount --cluster <cluster-id> -o yaml` — service accounts and their Kafka ACLs on this cluster
   - `conduktor get Subject --cluster <cluster-id> -o yaml` — schema registry subjects on this cluster
   - `conduktor get KafkaConnectCluster --cluster <cluster-id> -o yaml` — Kafka Connect clusters registered on this cluster
   - Then for each connect cluster: `conduktor get Connector --cluster <cluster-id> --connectCluster <connect-cluster-id> -o yaml` — connectors (requires both `--cluster` and `--connectCluster`)
5. Analyze the exported resources and present findings to the user:
   - **Clusters**: list each cluster ID and its topic count
   - **Topic ownership candidates**: group topics by naming prefix (e.g., `payments.*`, `inventory.*`) and propose each prefix group as a candidate Application
   - **Service account mapping**: for each service account, list which topic prefixes its ACLs cover — these map to ApplicationInstance `serviceAccount` fields
   - **Group permissions**: for each Console Group, list which clusters and topic patterns it has access to — these inform ApplicationGroup definitions
   - **Cross-team consumption**: identify service accounts or groups that have READ access to topics outside their primary prefix — these become ApplicationInstancePermission candidates
6. Ask the user to confirm or adjust the proposed application boundaries:
   - Which prefix groups should be merged or split?
   - What should each Application be named?
   - Which environments exist (dev/stag/prod) and how do they map to clusters?
   - Which Console Groups map to which Applications?
7. Ask the user for a target directory (default: `conduktor-self-service/`)
8. Generate the complete self-service resource set (see [self-service-github-cicd.md](self-service-github-cicd.md) for repo structure, workflows, CODEOWNERS, and ResourcePolicy examples):
   - `Application` for each confirmed team/service boundary
   - `ApplicationInstance` for each app/cluster combination with resource prefixes derived from topic naming, service account from ACL analysis, and env labels from cluster mapping
   - `ApplicationInstancePermission` for each cross-team consumption pattern discovered
   - `ApplicationGroup` for each Console Group mapped to an Application, with permissions scoped to the correct ApplicationInstance
   - `ResourcePolicy` stubs based on observed topic configurations (partition ranges, replication factors, naming patterns)
   - `Topic`, `Subject`, and `Connector` resources placed in the correct `applications/<app>/<env>/` folder
   - Supporting files: `README.md`, `.github/CODEOWNERS`, `.github/workflows/apply-platform.yml`, `.github/workflows/apply-apps.yml`
9. Present a summary of everything generated and offer to review any file
10. Offer to dry-run the platform resources: `conduktor apply -f platform/ -r --dry-run`

## When to use this

- Existing Conduktor Console deployment with clusters, topics, groups, and service accounts — but no self-service resources yet.
- Platform team wants to reverse-engineer ownership boundaries from the current state.
- Migrating from manual Console RBAC to the declarative self-service framework with GitOps.

## How it works

### 1. Discovery: what the CLI tells you

The CLI has two scopes: **global** resources (fetched via `conduktor get all -c`) and **cluster-scoped** resources (require `--cluster <id>`).

**Global resources** (`conduktor get all -c -o yaml`):

| Kind | What it reveals | Ownership signal |
|---|---|---|
| `KafkaCluster` | Cluster IDs and configs | Environment boundaries (dev-cluster, prod-cluster) |
| `Group` | Console Groups with cluster-scoped permissions | Which humans can see/manage which topics |

**Cluster-scoped resources** (for each cluster):

| Command | What it reveals | Ownership signal |
|---|---|---|
| `conduktor get Topic --cluster <id> -o yaml` | All topics on this cluster | Naming prefixes reveal team ownership |
| `conduktor get ServiceAccount --cluster <id> -o yaml` | SAs with Kafka ACLs on this cluster | ACL prefixes show which SA owns which topics |
| `conduktor get Subject --cluster <id> -o yaml` | Schema registry subjects | Schema ownership follows topic ownership |
| `conduktor get KafkaConnectCluster --cluster <id> -o yaml` | Connect clusters on this Kafka cluster | Needed to discover connectors |
| `conduktor get Connector --cluster <id> --connectCluster <cc> -o yaml` | Connectors on a connect cluster | Connector naming reveals team ownership |

### 2. Inferring ownership from topic prefixes

Group topics by their first segment to find natural ownership boundaries:

```
payments.transactions       ─┐
payments.settlements        ─┤── candidate Application: "payments"
payments.refunds            ─┘

inventory.stock-levels      ─┐
inventory.reservations      ─┤── candidate Application: "inventory"
inventory.warehouse-events  ─┘
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

### 6. Rollout strategy

Self-service resources should be applied in dependency order:

1. **ResourcePolicies** — define guardrails before anything else
2. **Applications** — create the logical groupings
3. **ApplicationInstances** — bind apps to clusters (this creates Kafka ACLs for the service accounts)
4. **Topics, Subjects, Connectors** — declare existing resources under self-service ownership (idempotent)
5. **ApplicationGroups** — migrate Console UI permissions to self-service managed groups
6. **ApplicationInstancePermissions** — formalize cross-team access

The agent offers to `--dry-run` each step before applying.

## Common mistakes

| Mistake | Fix |
|---|---|
| Running discovery with an ApplicationInstanceToken instead of AdminToken | Bootstrap requires full visibility. Use an AdminToken for all discovery commands. |
| Using `conduktor get all -c` and expecting Topics/ServiceAccounts | `get all` only returns global resources. Topics, ServiceAccounts, Subjects, and Connectors are cluster-scoped — fetch them per cluster with `--cluster <id>`. |
| Assigning a topic to the wrong Application based on prefix alone | Validate with service account ACLs — the SA with WRITE access is the true owner. |
| Missing cross-team READ patterns | Check all SA ACLs for READ on prefixes they don't own. Each one needs an ApplicationInstancePermission. |
| Creating ApplicationInstances with overlapping resource prefixes on the same cluster | Resource prefixes must not overlap between ApplicationInstances on the same cluster. Resolve conflicts before applying. |
| Forgetting to set `serviceAccount` on ApplicationInstance | Each instance needs a unique SA per cluster. Conduktor creates Kafka ACLs for this SA automatically. |
| Applying ApplicationGroups before the Application and ApplicationInstance exist | Resources must be applied in dependency order: Application → ApplicationInstance → ApplicationGroup. |
| Treating Console Group permissions as the only ownership signal | Console Groups control UI access only. Service account ACLs are the authoritative Kafka-level ownership signal. |
