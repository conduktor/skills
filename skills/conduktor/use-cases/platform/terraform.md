# Infrastructure as Code with the Conduktor Terraform provider

## Agent workflow

1. Check if there are existing `.tf` files in the workspace
2. Ask what to manage: Console resources, Gateway resources, or both. Self-service resources need an Enterprise Console license ([guardrails](../../references/guardrails.md) §1)
3. If no provider config exists, generate the `conduktor` provider block (`version = "~> 1.5"`) with the correct `mode` and auth env vars
4. Discover existing resources: `conduktor get all --console -o yaml` / `--gateway -o yaml` (root kinds only), plus `conduktor get Topic --cluster <id> -o yaml` for each cluster
5. Generate `conduktor_*` resource blocks in HCL matching the discovered resources, **plus an `import {}` block for each one that already exists** ([Adopting existing resources](#adopting-existing-resources)), and `lifecycle { prevent_destroy = true }` on topics
6. If managing both Console and Gateway, generate provider aliases
7. Offer to run `terraform init` then `terraform plan`. Read the plan: existing resources must show as imported. Stop on any `must be replaced` or `destroy` of an existing resource
8. On approval, offer `terraform apply`

Manage Conduktor Console and Gateway resources declaratively using the official Terraform provider.

## When to use this

- Provisioning Conduktor resources (users, groups, clusters, topics, interceptors) as part of a CI/CD pipeline.
- Enforcing consistent configuration across environments (dev/staging/prod).
- Codifying RBAC (users, groups, permissions) so access control is auditable and version-controlled.
- Managing Gateway virtual clusters and interceptors alongside Console resources in a single plan.
- Bootstrapping a fresh Conduktor deployment with a known-good state.

Not a fit when you only need one-off manual changes through the UI, or when using the CLI (`conduktor apply`) is simpler for ad-hoc operations.

## Provider setup

### Provider block

```hcl
terraform {
  required_providers {
    conduktor = {
      source  = "conduktor/conduktor"
      version = "~> 1.5"   # "~> 0.1" would resolve to 0.5.0 and exclude every 1.x release
    }
  }
}

provider "conduktor" {
  mode      = "console"
  base_url  = "http://localhost:8080"
  api_token = var.conduktor_api_token
}
```

### Environment variables

All provider attributes can be set via env vars. This is the recommended approach for CI.

| Attribute        | Env vars (Console)                                       | Env vars (Gateway)                                      |
|------------------|----------------------------------------------------------|---------------------------------------------------------|
| `base_url`       | `CDK_CONSOLE_BASE_URL`, `CDK_BASE_URL`                  | `CDK_GATEWAY_BASE_URL`, `CDK_BASE_URL`                  |
| `api_token`      | `CDK_API_TOKEN`, `CDK_API_KEY`                           | N/A                                                     |
| `admin_user`     | `CDK_CONSOLE_USER`, `CDK_ADMIN_EMAIL`, `CDK_ADMIN_USER`  | `CDK_GATEWAY_USER`, `CDK_ADMIN_USER`                    |
| `admin_password` | `CDK_CONSOLE_PASSWORD`, `CDK_ADMIN_PASSWORD`              | `CDK_GATEWAY_PASSWORD`, `CDK_ADMIN_PASSWORD`             |
| `cert`           | `CDK_CONSOLE_CERT`, `CDK_CERT`                           | `CDK_GATEWAY_CERT`, `CDK_CERT`                          |
| `key`            | `CDK_CONSOLE_KEY`, `CDK_KEY`                             | `CDK_GATEWAY_KEY`, `CDK_KEY`                            |
| `cacert`         | `CDK_CONSOLE_CACERT`, `CDK_CACERT`                      | `CDK_GATEWAY_CACERT`, `CDK_CACERT`                      |
| `insecure`       | `CDK_CONSOLE_INSECURE`, `CDK_INSECURE`                  | `CDK_GATEWAY_INSECURE`, `CDK_INSECURE`                  |

### Console mode vs Gateway mode

The `mode` attribute is required. It determines which API the provider talks to and which resources are available.

- `mode = "console"` -- manages `conduktor_console_*` resources against the Console API. Supports `api_token` or `admin_user`/`admin_password` auth.
- `mode = "gateway"` -- manages `conduktor_gateway_*` resources against the Gateway Admin API. Requires `admin_user`/`admin_password` auth only.

To manage both in one Terraform root module, use provider aliases:

```hcl
provider "conduktor" {
  alias    = "console"
  mode     = "console"
  base_url = "http://localhost:8080"
  api_token = var.conduktor_api_token
}

provider "conduktor" {
  alias          = "gateway"
  mode           = "gateway"
  base_url       = "http://localhost:8888"
  admin_user     = "admin"
  admin_password = var.gateway_admin_password
}

resource "conduktor_console_user_v2" "bob" {
  provider = conduktor.console
  # ...
}

resource "conduktor_gateway_service_account_v2" "sa" {
  provider = conduktor.gateway
  # ...
}
```

## Console resources (with working HCL)

### User

```hcl
resource "conduktor_console_user_v2" "bob" {
  name = "bob@company.io"
  spec = {
    firstname = "Bob"
    lastname  = "Smith"
    permissions = [
      {
        resource_type = "PLATFORM"
        permissions   = ["userView", "datamaskingView", "auditLogView"]
      },
      {
        resource_type = "TOPIC"
        name          = "test-topic"
        cluster       = "*"
        pattern_type  = "LITERAL"
        permissions   = ["topicViewConfig", "topicConsume", "topicProduce"]
      },
    ]
  }
}
```

### Group

```hcl
resource "conduktor_console_group_v2" "qa" {
  name = "qa-team"
  spec = {
    display_name    = "QA Team"
    description     = "Quality Assurance team"
    external_groups = ["sso-group1"]
    members         = [conduktor_console_user_v2.bob.name]
    permissions = [
      {
        resource_type = "PLATFORM"
        permissions   = ["userView", "clusterConnectionsManage"]
      },
      {
        resource_type = "TOPIC"
        name          = "test-topic"
        cluster       = "*"
        pattern_type  = "LITERAL"
        permissions   = ["topicViewConfig", "topicConsume", "topicProduce"]
      }
    ]
  }
}
```

### Kafka cluster

```hcl
resource "conduktor_console_kafka_cluster_v2" "main" {
  name = "main-cluster"
  spec = {
    display_name                 = "Main Kafka Cluster"
    icon                         = "kafka"
    color                        = "#000000"
    bootstrap_servers            = "localhost:9092"
    ignore_untrusted_certificate = true
  }
}
```

### Topic

```hcl
resource "conduktor_console_topic_v2" "clickstream" {
  name    = "clickstream-events"
  cluster = conduktor_console_kafka_cluster_v2.main.name
  labels = {
    domain = "clickstream"
  }
  description = "# Clickstream events topic"
  spec = {
    partitions         = 3
    replication_factor = 1
    configs = {
      "cleanup.policy" = "delete"
    }
  }
}
```

### Application + ApplicationInstance

```hcl
resource "conduktor_console_application_v1" "myapp" {
  name = "my-application"
  spec = {
    title       = "My Application"
    description = "Processes clickstream events"
    owner       = "admin"
  }
}

resource "conduktor_console_application_instance_v1" "myapp_dev" {
  name        = "dev"
  application = conduktor_console_application_v1.myapp.name
  spec = {
    cluster = conduktor_console_kafka_cluster_v2.main.name
    resources = [
      {
        type         = "TOPIC"
        name         = "clickstream"
        pattern_type = "PREFIXED"
      }
    ]
    application_managed_service_account = false
  }
}
```

## Gateway resources (with working HCL)

### Virtual cluster

```hcl
resource "conduktor_gateway_virtual_cluster_v2" "team_a" {
  name = "team-a"
  spec = {
    type        = "Standard"
    acl_enabled = false   # no acl_mode/super_users without ACLs
  }
}
```

`acl_mode` is immutable and forces a replacement: setting it later destroys and recreates the virtual cluster. Choose it when you first enable ACLs, with `acl_enabled = true` and either `acl_mode = "REST_API"` plus `acls`, or `acl_mode = "KAFKA_API"` plus `super_users`.

### Interceptor

```hcl
resource "conduktor_gateway_interceptor_v2" "strip_headers" {
  name = "remove-headers"
  spec = {
    plugin_class = "io.conduktor.gateway.interceptor.safeguard.MessageHeaderRemovalPlugin"
    priority     = 100
    config = jsonencode(jsondecode(<<EOF
      {
        "topic": "topic-.*",
        "headerKeyRegex": "headerKey.*"
      }
      EOF
    ))
  }
}
```

### Service account

```hcl
resource "conduktor_gateway_service_account_v2" "app_sa" {
  name = "app-service-account"
  spec = {
    type = "LOCAL"
  }
}
```

## Generic resource

The `conduktor_generic` resource lets you manage any Console resource using raw YAML manifests. Useful for resources not yet covered by a typed resource, but experimental -- import is not supported and migration to typed resources requires destroy/recreate.

```hcl
resource "conduktor_generic" "alice" {
  kind    = "User"
  version = "v2"
  name    = "alice@company.io"
  manifest = yamlencode(yamldecode(<<EOF
      apiVersion: v2
      kind: User
      metadata:
        name: "alice@company.io"
      spec:
        firstName: "Alice"
        lastName: "Smith"
        permissions:
          - resourceType: PLATFORM
            permissions: ["userView", "datamaskingView", "auditLogView"]
      EOF
  ))
}
```

## Adopting existing resources

Resources that already exist must be imported before the first `apply`. Otherwise `apply` silently overwrites them with the HCL, and a later `destroy` deletes them from production.

```hcl
import {
  to = conduktor_console_topic_v2.orders_events
  id = "prod/orders.events"                 # <cluster>/<topic>
}

import {
  to = conduktor_gateway_interceptor_v2.enforce_partition_limit
  id = "enforce-partition-limit/passthrough//"   # <name>/<vcluster>/<group>/<username>; scope omitted = passthrough (check with: conduktor get Interceptor --name enforce-partition-limit -o yaml)
}

resource "conduktor_console_topic_v2" "orders_events" {
  name    = "orders.events"
  cluster = "prod"
  spec = { partitions = 12, replication_factor = 3 }
  lifecycle { prevent_destroy = true }   # partitions, replication_factor, name, cluster changes force a replacement
}
```

Other import IDs are listed in each resource's Import section in the registry docs (most use the resource `name`).

## Full resource list

As of provider 1.5.1 (no resource was added since 1.0.0; there are no data sources):
- **Console (16):** `console_user_v2`, `console_group_v2`, `console_kafka_cluster_v2`, `console_kafka_connect_v2`, `console_topic_v2`, `console_kafka_subject_v2`, `console_ksqldb_cluster_v2`, `console_connector_v2`, `console_service_account_v1` (Kafka ACLs of a principal on a cluster), `console_partner_zone_v2`, `console_resource_policy_v1` (Topic, Connector, Subject, ApplicationGroup), `console_application_v1`, `console_application_instance_v1`, `console_application_group_v1`, `console_application_instance_permission_v1`, `console_topic_policy_v1` (creation blocked since Console 1.47; the plan fails on Console > 1.46.2).
- **Gateway (4):** `gateway_virtual_cluster_v2`, `gateway_interceptor_v2`, `gateway_service_account_v2`, `gateway_token_v2`.
- **Generic (1):** `conduktor_generic`, experimental and Console-only. It covers the kinds known to the embedded CLI catalog (not Integration, GlueSchema or the templates).

All names take the `conduktor_` prefix. The self-service resources fail without an Enterprise Console license.

## Common mistakes

| Mistake | Fix |
|---|---|
| `version = "~> 0.1"` | It resolves to 0.5.0 and excludes every 1.x release. Use `~> 1.5` |
| Applying HCL for resources that already exist | Import them first (`import {}` blocks); otherwise `apply` overwrites them and `destroy` later deletes them |
| Changing a topic's `partitions`, `replication_factor`, `name` or `cluster` | Forces destroy + create, with data loss. Use `prevent_destroy`; add partitions outside Terraform (`conduktor run topicAddPartitions`) |
| Missing `mode` | Required in HCL, not settable by env var: `mode = "console"` or `mode = "gateway"` |
| Using `api_token` with Gateway mode | Gateway only supports `admin_user`/`admin_password`; `api_token` is ignored |
| No alias when managing both Console and Gateway | Two provider blocks with `alias`, and `provider = conduktor.<alias>` on each resource |
| Referencing a cluster by internal ID | `conduktor_console_topic_v2.cluster` expects the cluster name |
| Storing credentials in HCL | Use env vars or `sensitive = true` variables. Kafka cluster `properties` (e.g. `sasl.jaas.config`) are not marked sensitive and appear in plans |
