# Guardrails: check before you act, stop before you break things

Read this before running any command that changes state (`apply`, `delete`, `run`, `terraform apply`, `helm install`). The behaviors described here were verified on Console 1.47.2, Gateway 3.21.1, CLI 0.9.2 and Terraform provider 1.5.1.

## 1. Preflight: plan and access

Run these before generating YAML for a workflow:

1. `conduktor run whoami` prints the token type (AdminToken, ApplicationInstanceToken…) and the user. `run` exits 0 even on API errors, so read the output.
2. `curl -s -H "Authorization: Bearer $CDK_API_KEY" "$CDK_BASE_URL/api/public/info/v1/license"` returns the Console `plan` and its `limitedFeatures` (e.g. `console.clusters.limit=3`). Admin tokens can read it.
3. If `plan` is `free` (Community Edition, no license), tell the user before writing any YAML: self-service (Application, ApplicationInstance, ApplicationGroup, ApplicationInstancePermission, ResourcePolicy), DataQualityPolicy, the Alert API, templates and group permissions are refused, and `limitedFeatures` caps the rest. Topics, Groups without permissions, Users, KafkaCluster, ServiceAccount and DataQualityRule still work.
4. Gateway 3.18+ refuses to start without `GATEWAY_LICENSE_KEY`. From 3.19, creating an Interceptor whose feature isn't in the license returns 403, and the response names the feature.

## 2. Error playbook

| Output | Meaning | Do this |
|---|---|---|
| `This feature is not available with your license. Please upgrade your plan.` (403) | Plan limitation, not an auth problem | Say which feature needs a paid plan; don't touch credentials |
| `Require permission …` or `The user doesn't have the required permission(s)` (403) | RBAC: the token lacks a permission | Name the permission; use an admin token or ask the platform team |
| 401, `Please set CDK_BASE_URL` | Missing or invalid credentials | Configure `CDK_BASE_URL` + `CDK_API_KEY` (or `CDK_GATEWAY_*`) |
| `required flag(s) "cluster" not set` | Cluster-scoped kind | Add `--cluster <id>` |
| `Could not find version v2 for kind X` / `kind X not found` | Wrong version number or kind name | Check `conduktor get --help` for the exact kind |
| `Missing environment variables: X` | The CLI substitutes `${X}` before sending | Export X, or write `$${X}` to keep the placeholder (see §4) |
| Gateway exits 98, `No license found!` | No `GATEWAY_LICENSE_KEY` | Add the license key; config changes won't get past this check |
| Gateway exits, `Both listener-style configuration and legacy environment variables are present` | Mixed network config (3.20+) | Keep `GATEWAY_LISTENER_*` only, drop the legacy vars it lists |

## 3. CLI state can delete data

With `--enable-state` or `CDK_STATE_ENABLED=true`, the CLI identifies a resource by its `apiVersion` string, its `kind` and its whole `metadata`: every key and value, except the values inside `labels`. If any of these changes, the CLI treats the resource as removed: it deletes it, then recreates it. For a Topic, all records are lost, and the command still exits 0.

Changes that trigger delete + recreate on a state-managed resource:
- editing `metadata.description`, `catalogVisibility`, `descriptionIsEditable` or `sqlStorage`
- adding or removing the `labels` block (changing a label value is safe)
- rewriting `apiVersion`, even `kafka/v2` → `v2`, the same version

Before applying any change to an existing resource under state:
1. Run the exact apply the pipeline runs (same files or folder, same state via `--state-file` or `CDK_STATE_REMOTE_URI`), with `--dry-run` added. A state location alone doesn't turn state on: keep `--enable-state` (or `CDK_STATE_ENABLED=true`), or the dry-run checks no deletion at all. Stderr prints `Loading state from …` when state is on. If you dry-run fewer files than the state tracks, everything else also shows up as `Deleted (dry-run)`.
2. If the output contains `Deleted (dry-run)` for a resource that is still in your files, stop. Do not apply. Tell the user this change will delete and recreate the resource, then offer:
   (a) drop the metadata or apiVersion change;
   (b) accept delete + recreate, only if the topic is empty or disposable;
   (c) keep the data: in one commit, make the change and point every job at a new, unused state location, as in the handover steps below. Editing the state entry by hand also works, but only with the pipeline paused until the change is merged: a run in between deletes the topic.

Also:
- Never rewrite `apiVersion` prefixes in a repo that uses state. Only the version number matters to the CLI and Console, so there is nothing to fix.
- Use one state location per complete resource set. Applying a subset of files against a shared state deletes everything else it tracks.
- PR pipelines must dry-run with the same state and remote URI, otherwise the PR won't show the deletions the merge will do.

To hand resources over to Terraform, the UI or another repo:
- Deleting their YAML deletes them on the next stateful apply, and `conduktor delete` deletes them too. The CLI has no command that only stops tracking a resource.
- Preferred: in one commit, delete the YAML and point every job that uses the state (apply and PR dry-run) at a new, unused state location. Nothing needs creating there: the CLI starts from an empty state, which deletes nothing on its first run, so keep other removals out of that commit. After that first run, move the old `cli-state.json` aside so no job can load it again.
- To keep the same location instead: pause the pipeline (e.g. disable the workflow), back up the state file, remove the resources' entries from it, merge the YAML deletion, and resume only after a stateful dry-run of main shows no `Deleted (dry-run)`. Any run between the state edit and the merge adds the entries back.
- The state file is JSON: a `resources` array of entries like `{"apiVersion": "kafka/v2", "kind": "Topic", "metadata": {"cluster": "prod", "name": "my-topic"}}`, matching each YAML's exact `apiVersion`, `kind` and `metadata`. Remotely it is `cli-state.json` under the URI prefix (`s3://state-bucket/conduktor/prod/?region=us-east-1` → `s3://state-bucket/conduktor/prod/cli-state.json`), unless the URI already ends in `.json`.

## 4. Secrets and `${VAR}` in YAML

- The CLI replaces `${VAR}` (and `${VAR:-default}`) client-side before sending. A missing or empty variable fails the apply, and `--permissive` has no effect in 0.9.2.
- A secret written as `token: ${VAULT_TOKEN}` is therefore stored in clear inside the Interceptor config. To let Gateway resolve it from its own environment, write `token: $${VAULT_TOKEN}`: the CLI then sends `${VAULT_TOKEN}` as-is.
- `conduktor get KafkaCluster -o yaml` prints SASL and Schema Registry passwords in clear. Redact them before writing files or showing output.

## 5. Terraform

- Pin `version = "~> 1.5"`. The old `~> 0.1` resolves to 0.5.0, which lacks `conduktor_gateway_virtual_cluster_v2`.
- If the resources already exist, generate `import {}` blocks before the first apply, for example `id = "prod/orders.events"` for a topic, or `id = "<name>/passthrough//"` for an interceptor whose scope is omitted (the non-virtual passthrough cluster). Read the real scope first (`conduktor get Interceptor --name <n> -o yaml`); an interceptor scoped `vCluster: null` (all clusters) isn't covered by this example. Without them, `apply` overwrites server state, and a later `destroy` deletes production resources.
- Add `lifecycle { prevent_destroy = true }` to topics. Changing `partitions`, `replication_factor`, `name` or `cluster` replaces the topic, which means destroy then create.
- Read every plan. If it shows `must be replaced` or `destroy` on an existing topic or virtual cluster, stop and ask.
- Self-service resources (`*_application_*`, `resource_policy_v1`) fail on Console without an Enterprise license: at plan with `admin_user`/`admin_password`, at apply with an API key.

## 6. Commands that need explicit confirmation

- `conduktor delete …`, `run topicEmpty` and `run connectorResetOffsets`.
- `run consumerGroupResetOffsets`: run `run consumerGroupResetOffsetsPreview` first and show the result.
- `template <Kind> -e -a` and `edit`: they open `$EDITOR` and apply on save, with no dry-run and no state. Don't use them as an agent. Write the file, then `conduktor apply -f <file> --dry-run`.

## 7. Command forms that work (CLI 0.9.2)

| Task | Command |
|---|---|
| Validate without applying | `conduktor apply -f <file-or-dir> [-r] --dry-run` (the path goes right after `-f`) |
| Check credentials | `conduktor run whoami` |
| Gateway resources | `conduktor get Interceptor -o yaml` (needs `CDK_GATEWAY_BASE_URL/USER/PASSWORD`). `--gateway`/`--console` exist only on `get all`. Interceptor, GatewayServiceAccount, AliasTopic and ConcentrationRule take no name argument: filter with `--name <n>`, `--vcluster <vc>` (plus `--group`, `--username`, `--global` for Interceptor) |
| Cluster-scoped kinds | `conduktor get Topic --cluster <id> -o name`. Same `--cluster` for Subject, ServiceAccount and KafkaConnectCluster, plus `--connectCluster` for Connector. `delete` and `edit` need it too |
| Kind names | Singular and exact: `get Topic`, `get Group`. `get topics` or `get Groups` fail |
| Export everything | `get all -c` lists root kinds only (no Topic, Subject, Connector, ServiceAccount, KafkaConnectCluster) and exits 0 on errors: read stderr |
| Admin API key without the UI | `CDK_USER=<admin> CDK_PASSWORD=<pwd> conduktor token create admin <name>` |
| Gateway token for a LOCAL service account | `conduktor run generateServiceAccountToken --v-cluster <vc> --username <sa> --life-time-seconds <n>` |
