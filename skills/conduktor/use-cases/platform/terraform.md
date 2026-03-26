# Infrastructure as Code with the Conduktor Terraform provider

## Agent workflow

1. Check if there are existing `.tf` files in the workspace
2. Ask what to manage: Console resources, Gateway resources, or both
3. If no provider config exists, generate the `conduktor` provider block with the correct `mode` and auth env vars
4. Run `conduktor get all --console -o yaml` or `conduktor get all --gateway -o yaml` to discover existing resources
5. Generate `conduktor_*` resource blocks in HCL matching the discovered resources
6. If managing both Console and Gateway, generate provider aliases
7. Offer to run `terraform init` then `terraform plan`
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
      version = "~> 0.1"
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
    acl_enabled = false
    type        = "Standard"
    acl_mode    = "KAFKA_API"
    super_users = ["user1"]
  }
}
```

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

## Full resource list

**Console (16)**

| Resource | Description |
|----------|-------------|
| `conduktor_console_user_v2` | Local or SSO user with RBAC permissions |
| `conduktor_console_group_v2` | Group with members, external groups, permissions |
| `conduktor_console_kafka_cluster_v2` | Kafka cluster connection |
| `conduktor_console_topic_v2` | Topic with config, labels, description |
| `conduktor_console_kafka_connect_v2` | Kafka Connect cluster connection |
| `conduktor_console_kafka_subject_v2` | Schema Registry subject |
| `conduktor_console_ksqldb_cluster_v2` | ksqlDB cluster connection |
| `conduktor_console_service_account_v1` | Console service account |
| `conduktor_console_connector_v2` | Kafka Connect connector |
| `conduktor_console_topic_policy_v1` | Topic policy (deprecated, use resource_policy) |
| `conduktor_console_resource_policy_v1` | CEL-based policy for Topic, Connector, Subject |
| `conduktor_console_partner_zone_v2` | Partner zone for external access |
| `conduktor_console_application_v1` | Application definition |
| `conduktor_console_application_group_v1` | Application group |
| `conduktor_console_application_instance_v1` | Application instance on a cluster |
| `conduktor_console_application_instance_permission_v1` | Permissions for an app instance |

**Gateway (4)**

| Resource | Description |
|----------|-------------|
| `conduktor_gateway_virtual_cluster_v2` | Virtual cluster with ACL config |
| `conduktor_gateway_interceptor_v2` | Interceptor plugin (encryption, header removal, policy, etc.) |
| `conduktor_gateway_service_account_v2` | Gateway service account (LOCAL or EXTERNAL) |
| `conduktor_gateway_token_v2` | Gateway authentication token |

**Generic (1)**

| Resource | Description |
|----------|-------------|
| `conduktor_generic` | Experimental YAML-based resource for any Console kind |

## Common mistakes

1. **Missing `mode`** -- The provider will fail to initialize. Always set `mode = "console"` or `mode = "gateway"`.
2. **Using `api_token` with Gateway mode** -- Gateway only supports `admin_user`/`admin_password`. The `api_token` field is ignored in gateway mode.
3. **No alias when managing both Console and Gateway** -- You need two provider blocks with `alias` and must set `provider = conduktor.<alias>` on each resource.
4. **Dirty plans with `conduktor_generic`** -- Always wrap YAML with `yamlencode(yamldecode(...))` to normalize whitespace. Raw strings cause perpetual diffs.
5. **Referencing cluster by ID instead of name** -- `conduktor_console_topic_v2.cluster` expects the cluster resource name, not an internal ID.
6. **Forgetting `required_providers`** -- Without the source declaration, Terraform cannot locate the provider on the registry.
7. **Storing credentials in HCL** -- Use env vars or Terraform variables with `sensitive = true`. Never commit `api_token` or `admin_password` in plain text.
