# Bootstrap self-service from existing Console resources

The output of this workflow is a repo that mirrors the official [conduktor/self-service-template](https://github.com/conduktor/self-service-template). Use the template as the scaffolding base — clone or fork it, then fill in the discovered Applications, ApplicationInstances, Groups, KafkaClusters, and ResourcePolicies. Do not regenerate the workflow YAML, CODEOWNERS structure, or starter policies from scratch; those live in the template and are kept current there.

For the exact bootstrap commands (with a `gh`-available path and a `git clone` fallback that requires user approval), see [self-service-github-cicd-cli.md → Agent: how to bootstrap a copy](self-service-github-cicd-cli.md#agent-how-to-bootstrap-a-copy). This file focuses on how to populate the resulting repo from existing Console state.

## Agent workflow

1. Run `conduktor token list admin` to verify CLI auth with an AdminToken
2. If not authenticated, help configure `CDK_BASE_URL` + `CDK_API_KEY`
3. Export global Console resources and discover clusters:
   - `conduktor get all -c -o yaml` — global resources: KafkaClusters, KafkaConnectClusters, Groups, and any existing self-service resources (does NOT include cluster-scoped resources like Topics, ServiceAccounts, Subjects, or Connectors)
   - Parse the output for `kind: KafkaCluster` entries to get all cluster IDs
   - Keep `KafkaCluster`, `KafkaConnectCluster`, and `Group` definitions for later mapping into `platform/clusters/<instance>/` and `platform/groups/` (step 8). Redact bootstrap-server passwords and Schema Registry credentials to `${VAR}` placeholders before writing them to disk.
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
   - Which **instances** exist and on what axis (env, region, data classification, tenant, etc.) — and how do they map to KafkaClusters? See [self-service-github-cicd-cli.md](self-service-github-cicd-cli.md#about-the-instance-slot) for the dimensions an instance can represent. Default to `dev`/`stag`/`prod` only if the user confirms environment is the right axis.
   - Which Console Groups map to which Applications?
7. Ask the user for a target directory. Recommend bootstrapping from the official template first using the [agent bootstrap procedure](self-service-github-cicd-cli.md#agent-how-to-bootstrap-a-copy) (checks for `gh`, falls back to shallow `git clone` with user approval). Otherwise default to `conduktor-self-service/`.
8. Populate the cloned template with the discovered resources, mapped to the layout in [self-service-github-cicd-cli.md](self-service-github-cicd-cli.md#repository-structure):
   - `KafkaCluster` and `KafkaConnectCluster` in `platform/clusters/<instance>/` — credentials as `${VAR}` placeholders (resolved from `kafka-<instance>` GitHub Environment secrets)
   - `Group` in `platform/groups/` — one per discovered Console Group mirroring an IdP group
   - `Application` per team/service boundary in `platform/applications/<app>/application.yml`
   - `ApplicationInstance` per app/cluster in `platform/applications/<app>/<instance>.yml` — resource prefixes from topic naming, `serviceAccount` from ACL analysis
   - `ApplicationInstancePermission` per cross-team READ pattern, in the **owning** app's `applications/<owning-app>/<instance>/instance-permissions.yml` (see [section 4](#4-generating-applicationinstancepermission-from-cross-team-read-patterns))
   - `ApplicationGroup` per Console Group → Application mapping, in `applications/<app>/<instance>/application-groups.yml`
   - `Topic`, `Subject`, `Connector` in the matching `applications/<app>/<instance>/` folder
   - `ResourcePolicy` files in `platform/policies/` — start from [references/resource-policy-examples.md](../../references/resource-policy-examples.md). Before writing them, present the observed config ranges (e.g. "partition counts 3–6, retention 7d–28d") and ask the user to confirm or adjust the bounds — don't silently tune values.
   - Update CODEOWNERS in the template to match the discovered application teams (replace placeholder team slugs)
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
| `KafkaCluster` | Cluster IDs and configs | Instance boundaries (which cluster maps to which `<instance>` — environment, region, classification, etc.) |
| `KafkaConnectCluster` | Connect cluster registrations | Goes into `platform/clusters/<instance>/` alongside the matching KafkaCluster |
| `Group` | Console Groups with cluster-scoped permissions | Which humans can see/manage which topics; goes into `platform/groups/` |

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

**Generated `instance-permissions.yml`** (placed in `applications/<owning-app>/<instance>/`):

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

### 6. Mapping clusters to instances

Ask the user how clusters map to ApplicationInstances. The instance axis is not necessarily environment — see [self-service-github-cicd-cli.md](self-service-github-cicd-cli.md#about-the-instance-slot) for the dimensions an instance can represent. Common patterns when the axis is environment:

| Cluster ID | Instance |
|---|---|
| `dev-cluster` | `dev` |
| `staging-cluster` or `stag-cluster` | `stag` |
| `prod-cluster` or `production` | `prod` |

But also consider: region (`prod-us-east`, `prod-eu-west`), data classification (`pii`, `non-pii`), regulatory domain (`sox`, `pci`), workload tier (`critical`, `batch`), or tenant (`tenant-acme`). Ask before assuming.

This mapping determines the `metadata.labels.instance` on each ApplicationInstance and the folder structure under `applications/<app>/<instance>/`.

### 7. Rollout strategy

Self-service resources should be applied in dependency order:

1. **KafkaClusters / KafkaConnectClusters** — defined in `platform/clusters/<instance>/`, applied via `apply-clusters.yml` with instance-scoped credentials
2. **Groups** — Console Groups in `platform/groups/`, mirrored from external IdP
3. **ResourcePolicies** — define guardrails before anything that references them
4. **Applications** — create the logical groupings
5. **ApplicationInstances** — bind apps to clusters (this creates Kafka ACLs for the service accounts)
6. **Topics, Subjects, Connectors** — declare existing resources under self-service ownership (idempotent)
7. **ApplicationGroups** — migrate Console UI permissions to self-service managed groups
8. **ApplicationInstancePermissions** — formalize cross-team access

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
| Placing `instance-permissions.yml` in the consuming app's folder | The **owning** ApplicationInstance creates the permission. Place it in `applications/<owning-app>/<instance>/`. |
| Regenerating workflows, CODEOWNERS, or starter policies from scratch | These live in the official [conduktor/self-service-template](https://github.com/conduktor/self-service-template) and are kept current there. Clone the template and only customize CODEOWNERS team slugs and policy thresholds. |
| Defaulting to `dev`/`stag`/`prod` instances without asking | Instance can be region, classification, tenant, etc. Ask the user what dimension matters before naming instances. |
| Embedding cluster credentials directly in `platform/clusters/<instance>/*.yml` | Replace credentials with `${VAR}` placeholders; supply the values via `kafka-<instance>` GitHub Environment secrets. |
