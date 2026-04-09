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
   - **Topic ownership candidates**: tokenize topic names by all common delimiters (`.`, `-`, `_`), build a prefix tree, and find the depth that produces coherent groupings — propose each group as a candidate Application
   - **Service account mapping**: for each service account, list which topic prefixes its ACLs cover — these map to ApplicationInstance `serviceAccount` fields
   - **Group permissions**: for each Console Group, list which clusters and topic patterns it has access to — these inform ApplicationGroup definitions
   - **Cross-team consumption**: identify service accounts or groups that have READ access to topics outside their primary prefix — these become ApplicationInstancePermission candidates
6. Ask the user to confirm or adjust the proposed application boundaries:
   - Which prefix groups should be merged or split?
   - What should each Application be named?
   - Which environments exist (dev/stag/prod) and how do they map to clusters?
   - Which Console Groups map to which Applications?
7. Ask the user for a target directory (default: `conduktor-self-service/`)
8. Generate the complete self-service resource set (see [self-service-github-cicd-cli.md](self-service-github-cicd-cli.md) for repo structure, workflows, CODEOWNERS, and ResourcePolicy examples):
   - `Application` for each confirmed team/service boundary
   - `ApplicationInstance` for each app/cluster combination with resource prefixes derived from topic naming, service account from ACL analysis, and env labels from cluster mapping
   - `ApplicationInstancePermission` for each cross-team READ pattern discovered — place each `instance-permissions.yml` in the **owning** application's `applications/<owning-app>/<env>/` folder (see [section 4](#4-generating-applicationinstancepermission-from-cross-team-read-patterns))
   - `ApplicationGroup` for each Console Group mapped to an Application, with permissions scoped to the correct ApplicationInstance
   - `ResourcePolicy` files in `platform/policies/` — start from the starter policies in [references/resource-policy-examples.md](../../references/resource-policy-examples.md) (topic-naming, topic-labels, topic-rules-dev, topic-rules-prod, subject-rules, connector-rules, appgroup-restrictions). Before generating, present the observed configuration ranges to the user (e.g., "partition counts range from 3–6, replication factor is 3 everywhere, retention ranges from 7d–28d") and ask them to confirm or adjust the policy bounds. Do not silently tune values.
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

Topic names use varied conventions — dots, hyphens, underscores, or combinations — and the ownership-relevant segment is not always the first one. Do not assume a single delimiter or a fixed prefix depth.

**Approach:**

1. Collect all topic names on the cluster.
2. Tokenize each name by splitting on all common delimiters (`.`, `-`, `_`).
3. Build a prefix tree from the tokens and find the depth that produces the most coherent groupings — the level where distinct groups emerge without collapsing everything into one bucket or fragmenting into single topics.
4. Present the candidate groupings to the user for confirmation.

**Simple naming** — ownership at first segment:

```
payments.transactions       ─┐
payments.settlements        ─┤── candidate Application: "payments"
payments.refunds            ─┘
```

**Multi-segment naming** — ownership may be buried deeper:

```
prod.us.payments.tx-created     ─┐
prod.us.payments.tx-settled     ─┤── candidate Application: "payments"
prod.us.payments.refund-issued  ─┘

prod.us.inventory.stock-levels  ─┐
prod.us.inventory.reservations  ─┤── candidate Application: "inventory"
prod.us.inventory.warehouse     ─┘
```

Here `prod` and `us` are shared by all topics — the meaningful split is at depth 3. The prefix tree makes this visible: depth 1 gives one group, depth 2 gives one group, depth 3 gives two distinct groups.

**Mixed delimiters** — tokenize before grouping:

```
stage.abc.xyz.payments-summary.def_ghi   ─┐
stage.abc.xyz.payments-invoices.foo_bar   ─┤── candidate Application: "payments"
stage.abc.xyz.orders-created.baz_qux      ── candidate Application: "orders"
```

Splitting on `.`, `-`, and `_` gives tokens `[stage, abc, xyz, payments, summary, def, ghi]`. The grouping emerges at the 4th token.

**When grouping fails:** topics with flat names, no shared prefixes, or inconsistent conventions won't cluster. Flag these as unassigned and ask the user to assign them manually.

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

### 4. Generating ApplicationInstancePermission from cross-team READ patterns

Each service account with READ ACLs on a prefix owned by another Application is a cross-team consumer. The **owning** ApplicationInstance creates an `ApplicationInstancePermission` granting read access to the **consuming** ApplicationInstance.

**Mapping ACLs to permissions:**

```
SA "sa-notifications-prod" has ACLs:
  WRITE on Topic PREFIXED "notifications."  → owner of notifications (skip — self)
  READ  on Topic PREFIXED "payments."       → payments team owns this prefix
  READ  on Topic LITERAL  "orders.completed" → orders team owns this topic

Maps to:
  1. payments-prod grants READ to notifications-prod on PREFIXED "payments."
  2. orders-prod grants READ to notifications-prod on LITERAL "orders.completed"
```

**Generated `instance-permissions.yml`** (placed in `applications/<owning-app>/<env>/`):

```yaml
---
apiVersion: self-serve/v1
kind: ApplicationInstancePermission
metadata:
  application: "payments"
  appInstance: "payments-prod"
  name: "payments-prod-to-notifications-prod"
spec:
  resource:
    type: TOPIC
    name: "payments."
    patternType: PREFIXED
  serviceAccountPermission: READ
  userPermission: READ
  grantedTo: "notifications-prod"
```

**Key rules:**
- **Placement:** `instance-permissions.yml` goes in the **owning** application's folder, not the consumer's. The owner controls who can access their resources.
- **Naming:** `<owning-instance>-to-<granted-instance>`. Add a suffix when one instance grants multiple permissions to the same target (e.g., `-topics`, `-events`).
- **Validation:** `spec.resource.name` must fall under the owning ApplicationInstance's declared resource pattern. Both instances must be on the same cluster. `spec` is immutable — delete and recreate to change.

### 5. Mapping Console Groups to ApplicationGroups

Console Groups define UI permissions. Each group's cluster-scoped permissions indicate which Application it belongs to:

```
Group "payments-developers":
  dev-cluster: topicViewConfig, topicConsume, topicProduce on "payments.*"
  prod-cluster: topicViewConfig, topicConsume on "payments.*"

→ ApplicationGroup "payments-developers-dev"  (full access in dev)
→ ApplicationGroup "payments-developers-prod" (read-only in prod)
```

If a group has permissions spanning multiple prefix groups, it may be an admin/platform group rather than an application group. Flag these for the user to decide.

### 6. Mapping clusters to environments

Ask the user how clusters map to environments. Common patterns:

| Cluster ID | Environment |
|---|---|
| `dev-cluster` | `dev` |
| `staging-cluster` or `stag-cluster` | `stag` |
| `prod-cluster` or `production` | `prod` |

This mapping determines the `metadata.labels.env` on each ApplicationInstance and the folder structure under `applications/<app>/<env>/`.

### 7. Rollout strategy

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
| Splitting topic names on the first `.` only | Real topic names use mixed delimiters and multi-segment prefixes (e.g., `prod.us.payments.tx-created`). Tokenize by all delimiters and find the grouping depth from the prefix tree. |
| Missing cross-team READ patterns | Check all SA ACLs for READ on prefixes they don't own. Each one needs an ApplicationInstancePermission. |
| Creating ApplicationInstances with overlapping resource prefixes on the same cluster | Resource prefixes must not overlap between ApplicationInstances on the same cluster. Resolve conflicts before applying. |
| Forgetting to set `serviceAccount` on ApplicationInstance | Each instance needs a unique SA per cluster. Conduktor creates Kafka ACLs for this SA automatically. |
| Applying ApplicationGroups before the Application and ApplicationInstance exist | Resources must be applied in dependency order: Application → ApplicationInstance → ApplicationGroup. |
| Treating Console Group permissions as the only ownership signal | Console Groups control UI access only. Service account ACLs are the authoritative Kafka-level ownership signal. |
| Placing `instance-permissions.yml` in the consuming app's folder | The **owning** ApplicationInstance creates the permission. Place it in `applications/<owning-app>/<env>/`. |
| Ignoring READ ACLs on foreign prefixes | READ ACLs on prefixes owned by another Application are cross-team consumption patterns — generate an `ApplicationInstancePermission` for each one. |
