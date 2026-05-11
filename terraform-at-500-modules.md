# Terraform at 500+ Modules: The Structure That Keeps a Multi-Cloud Org Sane

> *A senior cloud architect's opinionated guide to scaling Terraform across AWS, Azure, and GCP without losing your mind — or your engineers.*

---

## Preface: Why This Article Exists

Most Terraform articles are written for teams of two managing a startup's infrastructure. They tell you to use remote state, pin module versions, and use workspaces for environments. That advice isn't wrong. It's just irrelevant at the scale where Terraform actually becomes hard.

This article is for organizations that have already scaled past "it works" and are now dealing with the consequences: engineers afraid to touch modules because they don't know what depends on them, state files that take 45 minutes to plan, 12 different tagging conventions across business units, and a CI/CD pipeline that nobody fully understands.

Everything here is based on patterns I've seen work and fail at real organizations operating hundreds to thousands of infrastructure resources across multiple clouds. The goal is not to give you a template to copy — it's to give you a framework for making the right decisions for your specific organization.

---

## Table of Contents

1. Why Terraform Breaks at Scale
2. The Organizational Structure Decision
3. Module Taxonomy
4. Folder and Repository Structure
5. Environment Strategy
6. State Management at Scale
7. Governance and Standards
8. Versioning Strategy
9. CI/CD Architecture
10. Multi-Cloud Realities
11. Anti-Patterns
12. Reference Architecture
13. What I'd Do Starting From Scratch
14. Summary Checklist

---

## 1. Why Terraform Breaks at Scale

Terraform does not fail dramatically at scale. It fails gradually, then all at once. Understanding the failure modes at each growth stage is the first step to avoiding them.

### The 50-Module Wall

At 50 modules, the problems are social before they are technical. You have:

- No agreed-upon module structure. Every module was written by whoever needed it.
- No version pinning strategy. Some modules are called with `source = "../modules/vpc"`, others with `?ref=main` pointing at HEAD.
- State files that are either too large (one state for everything) or too scattered (one state per engineer's experiment).
- No tagging standard. AWS Cost Explorer is useless.

The technical debt is survivable at 50 modules. The damage is done in the next six months when 10 new engineers join and learn the wrong patterns by example.

### The 100-Module Inflection Point

At 100 modules, technical problems become unavoidable:

**Blast radius anxiety.** A change to a foundational networking module touches dozens of downstream consumers. Engineers start avoiding changes. The module rots.

**Plan times.** If you're running `terraform plan` against a state file with 500+ resources, you're waiting. If your CI pipeline runs the full graph on every PR, you're blocked.

**Drift accumulation.** Resources get manually changed in the cloud console to unblock urgent situations. Nobody updates the Terraform. Over six months you have significant configuration drift that nobody wants to reconcile.

**Module discovery failure.** New engineers write new modules for things that already exist because the existing module was in a repository named after an old project. Duplication compounds.

### The 500-Module Reality

At 500+ modules across multiple clouds, you are no longer dealing with infrastructure engineering problems. You are dealing with **platform engineering at organizational scale**. The problems are:

- Dependency graphs so complex that removing a module requires weeks of archaeology.
- Multiple teams running incompatible versions of the same foundational module.
- State locking conflicts becoming a daily occurrence.
- Security and compliance teams unable to enforce policy because modules are inconsistent.
- Cost attribution impossible because tagging is anarchic.
- Cloud provider API changes breaking multiple modules simultaneously with no coordinated upgrade path.

The organizations that handle this well have one thing in common: they stopped thinking about Terraform as a collection of configuration files and started thinking about it as a **product with users, a lifecycle, and governance requirements**.

---

## 2. The Organizational Structure Decision

This is the highest-leverage decision you will make. It shapes everything that follows. There are three viable patterns.

### Mono-Repo

All Terraform lives in a single repository. Every team, every environment, every module.

```
infrastructure/
├── modules/
├── environments/
├── stacks/
└── policies/
```

**Where it works:** Organizations with a strong platform team that owns the entire infrastructure surface, fewer than 200 engineers, and a culture of shared ownership.

**Where it fails:** When individual product teams need autonomy over their infrastructure release cycles. When the repository becomes so large that `git clone` and CI runs are painfully slow. When different business units have radically different security requirements.

**Real tradeoff:** You get consistency and discoverability at the cost of autonomy and velocity. A platform team that gates every infrastructure change through a single repo will become a bottleneck. Plan for this.

### Multi-Repo

Modules and infrastructure configurations live in separate repositories by team, domain, or product.

```
github.com/org/
├── tf-module-networking
├── tf-module-security
├── tf-module-kubernetes
├── infra-platform-team
├── infra-product-team-a
└── infra-product-team-b
```

**Where it works:** Large organizations with distinct product teams that have meaningful infrastructure ownership. Organizations where different teams operate at different velocity and risk tolerance.

**Where it fails:** When nobody owns cross-cutting concerns. When teams drift into incompatible patterns because there's no shared standard. When you need to coordinate a breaking change across 30 repositories simultaneously.

**Real tradeoff:** You get autonomy and velocity at the cost of consistency. This is manageable with strong governance tooling and a dedicated platform team whose job is coordination, not control.

### Hybrid (The Pattern That Actually Scales)

A dedicated repository for shared foundational modules, with product teams maintaining their own infrastructure repositories that consume those shared modules via version-pinned references.

```
github.com/org/
├── tf-platform-modules     # Owned by platform team. Versioned releases.
├── tf-security-baseline    # Owned by security team. Policy as code.
├── infra-product-team-a    # Owned by product team A. Consumes platform modules.
├── infra-product-team-b    # Owned by product team B. Consumes platform modules.
└── infra-environments      # Owned by platform team. Account/project bootstrapping.
```

This is the pattern I recommend for any organization over 150 engineers or 100 modules. It gives you:

- A clear ownership model (who owns what is in the repository name)
- A forcing function for module quality (shared modules get real code review)
- Release independence (product teams pin to stable versions and upgrade on their own schedule)
- A governance surface (platform modules are the place to enforce security and compliance)

The cost is coordination overhead for breaking changes in shared modules. This is real but manageable with the versioning strategy described in Section 8.

---

## 3. Module Taxonomy

Not all modules are equal. The biggest structural mistake organizations make is treating a low-level S3 bucket module and a high-level "production application environment" module as the same kind of thing. They have completely different ownership, lifecycle, and consumption patterns.

The taxonomy that works at scale has four tiers.

### Tier 1: Foundational Modules

These are cloud-primitive wrappers. They encapsulate a single cloud resource type with organizational standards baked in. A foundational module takes a complex cloud resource API and presents a simpler, opinionated interface.

**Characteristics:**
- Wraps exactly one or two tightly-coupled cloud resources
- Enforces organizational standards (mandatory tags, encryption defaults, logging)
- Owned by the platform team
- Changes require careful backward compatibility
- Consumed by hundreds of configurations

**Examples:** `aws-s3-bucket`, `azure-storage-account`, `gcp-gcs-bucket`, `aws-iam-role`, `azure-vnet`

```hcl
# modules/foundational/aws-s3-bucket/main.tf
# This module wraps aws_s3_bucket with org standards baked in.
# It is NOT a thin wrapper — it enforces encryption, versioning, 
# and logging that our security baseline requires everywhere.

resource "aws_s3_bucket" "this" {
  bucket = var.bucket_name
  tags   = merge(local.mandatory_tags, var.additional_tags)
}

resource "aws_s3_bucket_server_side_encryption_configuration" "this" {
  bucket = aws_s3_bucket.this.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = var.kms_key_id != null ? "aws:kms" : "AES256"
      kms_master_key_id = var.kms_key_id
    }
  }
}

resource "aws_s3_bucket_versioning" "this" {
  bucket = aws_s3_bucket.this.id
  versioning_configuration {
    status = var.versioning_enabled ? "Enabled" : "Disabled"
  }
}

# Mandatory tags are non-negotiable. No caller can override them.
locals {
  mandatory_tags = {
    managed_by   = "terraform"
    environment  = var.environment
    cost_center  = var.cost_center
    data_class   = var.data_classification
    team         = var.owning_team
  }
}
```

### Tier 2: Platform Modules

These compose multiple foundational modules into a coherent infrastructure capability. A platform module represents a reusable architectural pattern.

**Characteristics:**
- Composes 3–15 foundational modules
- Represents a well-understood architectural pattern
- Owned by the platform team
- Stable interface, infrequent breaking changes
- Consumed by dozens of stack configurations

**Examples:** `aws-eks-cluster`, `azure-aks-platform`, `gcp-gke-standard`, `three-tier-vpc`, `data-lake-foundation`

```hcl
# modules/platform/aws-eks-cluster/main.tf
# This is NOT a thin EKS wrapper. It is an opinionated EKS cluster
# that includes our standard node group configuration, IRSA setup,
# cluster add-ons, and logging configuration. 
# Teams consume this, not the raw aws_eks_cluster resource.

module "vpc" {
  source  = "../../foundational/aws-vpc"
  version = "~> 3.0"
  # ... vpc config
}

module "eks" {
  source  = "../../foundational/aws-eks-control-plane"
  version = "~> 2.0"
  vpc_id  = module.vpc.vpc_id
  # ...
}

module "node_group_standard" {
  source        = "../../foundational/aws-eks-node-group"
  version       = "~> 2.0"
  cluster_name  = module.eks.cluster_name
  instance_type = var.node_instance_type
  # ...
}
```

### Tier 3: Product Modules

These are organization-specific compositions that represent a complete infrastructure unit for a specific product or domain. They may be multi-cloud or cloud-specific.

**Characteristics:**
- Owned by product teams or a shared platform function
- May include business logic (naming conventions tied to product, specific integrations)
- Can have faster breaking change cycles because they serve a narrower audience
- Often the entry point for a product team's infrastructure

**Examples:** `ecommerce-checkout-environment`, `data-platform-streaming-pipeline`, `ml-training-cluster`

### Tier 4: Application Modules

These are the leaf nodes of the graph. They describe a specific application's infrastructure requirements and are consumed once per deployment target.

**Characteristics:**
- Owned entirely by the application team
- May live in the application repository rather than the infrastructure repository
- Consumes platform and product modules; rarely foundational modules directly
- Changes frequently; no backward-compatibility requirements

```hcl
# application/payment-service/infrastructure/main.tf
# This is the payment service's infrastructure configuration.
# It consumes platform modules and is owned by the payments team.

module "service_environment" {
  source  = "git::https://github.com/org/tf-platform-modules//modules/product/ecs-service?ref=v4.2.1"
  
  service_name    = "payment-service"
  environment     = var.environment
  container_image = var.container_image
  cpu             = 1024
  memory          = 2048
  
  # Payments-specific config
  enable_waf      = true
  data_class      = "pci-in-scope"
  
  secrets = {
    STRIPE_API_KEY = "arn:aws:secretsmanager:..."
    DB_PASSWORD    = "arn:aws:secretsmanager:..."
  }
}
```

### Why This Taxonomy Matters

Without a clear taxonomy, you get module sprawl: 500 modules that are all the same "type" with no clear ownership, no clear consumer expectations, and no clear upgrade path. With the taxonomy, every module has a natural home, a natural owner, and a natural stability expectation.

The most important boundary to enforce: **application teams should not be writing foundational modules**. When they do, you get 12 different VPC modules with slightly different opinions about subnet sizing, and a future migration project that consumes quarters of engineering time.

---

## 4. Folder and Repository Structure

Theory without structure is useless. Here are the actual directory layouts that work in practice.

### Platform Modules Repository

```
tf-platform-modules/
├── modules/
│   ├── foundational/
│   │   ├── aws/
│   │   │   ├── s3-bucket/
│   │   │   │   ├── main.tf
│   │   │   │   ├── variables.tf
│   │   │   │   ├── outputs.tf
│   │   │   │   ├── versions.tf
│   │   │   │   └── README.md
│   │   │   ├── iam-role/
│   │   │   ├── vpc/
│   │   │   ├── rds-instance/
│   │   │   └── eks-control-plane/
│   │   ├── azure/
│   │   │   ├── storage-account/
│   │   │   ├── vnet/
│   │   │   ├── aks-cluster/
│   │   │   └── key-vault/
│   │   └── gcp/
│   │       ├── gcs-bucket/
│   │       ├── vpc-network/
│   │       └── gke-cluster/
│   ├── platform/
│   │   ├── aws/
│   │   │   ├── eks-platform/
│   │   │   ├── ecs-platform/
│   │   │   ├── data-lake/
│   │   │   └── three-tier-app/
│   │   ├── azure/
│   │   │   └── aks-platform/
│   │   └── gcp/
│   │       └── gke-platform/
│   └── product/
│       ├── microservice-aws/
│       ├── microservice-azure/
│       └── ml-training-gcp/
├── examples/
│   ├── aws-eks-basic/
│   ├── azure-aks-production/
│   └── gcp-gke-autopilot/
├── tests/
│   ├── foundational/
│   └── platform/
├── docs/
│   ├── module-authoring-guide.md
│   ├── breaking-change-policy.md
│   └── taxonomy.md
├── CHANGELOG.md
└── .github/
    ├── workflows/
    │   ├── validate.yml
    │   ├── test.yml
    │   └── release.yml
    └── CODEOWNERS
```

### Product Team Infrastructure Repository

```
infra-payments-team/
├── stacks/
│   ├── aws-us-east-1/
│   │   ├── production/
│   │   │   ├── main.tf           # Calls platform modules
│   │   │   ├── variables.tf
│   │   │   ├── outputs.tf
│   │   │   ├── terraform.tfvars  # Non-secret config
│   │   │   └── backend.tf        # Remote state config
│   │   ├── staging/
│   │   │   └── ...
│   │   └── development/
│   │       └── ...
│   └── azure-westeurope/
│       ├── production/
│       └── staging/
├── modules/
│   └── payment-service/          # Product-specific module (Tier 3)
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
├── policies/
│   └── payment-specific-checks.rego
├── .github/
│   └── workflows/
│       ├── plan.yml
│       └── apply.yml
└── docs/
    └── runbook.md
```

### Stack Configuration Example

This is the pattern for a leaf-level stack that ties everything together. Stacks are not modules. They are configuration that calls modules.

```hcl
# stacks/aws-us-east-1/production/main.tf

terraform {
  required_version = ">= 1.6.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  
  backend "s3" {
    bucket         = "org-terraform-state-prod"
    key            = "payments/aws-us-east-1/production/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-locks"
    encrypt        = true
    kms_key_id     = "arn:aws:kms:us-east-1:123456789:key/..."
  }
}

# Module versions are pinned to exact tags. Never use branches.
module "payment_service" {
  source = "git::https://github.com/org/tf-platform-modules//modules/product/microservice-aws?ref=v5.1.2"

  environment      = "production"
  aws_region       = "us-east-1"
  cost_center      = "payments-1234"
  owning_team      = "payments-engineering"
  data_class       = "pci-in-scope"
  
  service_config = {
    name          = "payment-processor"
    cpu           = 2048
    memory        = 4096
    min_capacity  = 3
    max_capacity  = 50
    enable_waf    = true
    enable_shield = true
  }
  
  # Database is a separate state. Referenced via data source.
  # This is intentional — database lifecycle != application lifecycle.
  database_connection = {
    host     = data.terraform_remote_state.database.outputs.endpoint
    port     = 5432
    name     = "payments_prod"
  }
}

# Cross-stack reference. Read-only. Explicit dependency.
data "terraform_remote_state" "database" {
  backend = "s3"
  config = {
    bucket = "org-terraform-state-prod"
    key    = "payments/aws-us-east-1/production/database/terraform.tfstate"
    region = "us-east-1"
  }
}
```

---

## 5. Environment Strategy

"Use workspaces for environments" is advice that works for simple cases and fails at enterprise scale. Here is why, and what to do instead.

### The Workspace Problem

Terraform workspaces are a state isolation mechanism, not an environment isolation mechanism. When you use workspaces for environments, you end up with:

- A single backend configuration that serves production and development equally
- Risk of running `terraform apply -workspace=production` from a development laptop
- No natural place to enforce different IAM permissions per environment
- Prod and dev state files in the same storage bucket with the same blast radius

### Account/Subscription/Project-Level Isolation

The pattern that scales is using separate cloud accounts (AWS), subscriptions (Azure), or projects (GCP) per environment tier, with separate state backends for each. This gives you:

- Hard IAM boundaries enforced by the cloud provider
- Blast radius limited by cloud account
- Independent billing and cost attribution
- Ability to have completely different configurations per environment without `count` conditionals everywhere

```
Environment Hierarchy
─────────────────────────────────────────────────────────────────
                    Organization Root
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
  Development         Staging           Production
  Account/Sub       Account/Sub         Account/Sub
        │                 │                 │
  ┌─────┴─────┐     ┌─────┴─────┐     ┌─────┴──────┐
  │ us-east-1 │     │ us-east-1 │     │ us-east-1  │
  │ eu-west-1 │     │ eu-west-1 │     │ eu-west-1  │
  └───────────┘     └───────────┘     │ ap-south-1 │
                                      └────────────┘
  (single region)   (single region)   (multi-region)
─────────────────────────────────────────────────────────────────
```

### Environment Promotion Pattern

Rather than parameterizing everything with `var.environment`, use a consistent stack structure where each environment is a separate Terraform root module.

```
stacks/
├── aws-us-east-1/
│   ├── development/
│   │   ├── backend.tf     # Points to dev state bucket
│   │   ├── provider.tf    # Assumes dev account role
│   │   └── main.tf        # May have dev-specific config
│   ├── staging/
│   │   ├── backend.tf     # Points to staging state bucket
│   │   ├── provider.tf    # Assumes staging account role
│   │   └── main.tf        # Largely identical to dev
│   └── production/
│       ├── backend.tf     # Points to prod state bucket
│       ├── provider.tf    # Assumes prod account role
│       └── main.tf        # May have prod-specific additions
```

The `main.tf` files will be nearly identical. That's fine. The duplication is intentional. It makes each environment independently auditable and removes the risk of a conditional expression accidentally toggling the wrong configuration in production.

### The Configuration Difference Problem

In practice, environments differ in:

- Scale parameters (instance sizes, replica counts, retention periods)
- Feature flags (not every feature reaches production simultaneously)
- External dependencies (dev uses a shared database, prod uses a dedicated one)
- Security controls (prod has WAF, Shield, and additional logging that dev doesn't need)

Handle this with `terraform.tfvars` files per environment rather than complex variable conditionals in module code:

```hcl
# stacks/aws-us-east-1/production/terraform.tfvars
environment           = "production"
service_cpu           = 2048
service_memory        = 4096
service_min_capacity  = 3
service_max_capacity  = 50
enable_waf            = true
enable_shield         = true
log_retention_days    = 365
db_instance_class     = "db.r6g.2xlarge"
db_multi_az           = true
```

```hcl
# stacks/aws-us-east-1/development/terraform.tfvars
environment           = "development"
service_cpu           = 256
service_memory        = 512
service_min_capacity  = 1
service_max_capacity  = 3
enable_waf            = false
enable_shield         = false
log_retention_days    = 7
db_instance_class     = "db.t4g.medium"
db_multi_az           = false
```

---

## 6. State Management at Scale

State management is where Terraform at scale gets genuinely dangerous. A poorly organized state structure can make incidents worse, make refactoring nearly impossible, and create blast radius problems that keep you up at night.

### State Decomposition Principles

The goal is to decompose state so that:

1. The blast radius of any single `terraform apply` is bounded and acceptable
2. State operations (plan, apply) complete in under 5 minutes
3. Engineers can reason about what a state file contains without running `terraform state list`
4. Cross-team state access is explicit and read-only

**Rule of thumb:** A state file with more than 200 resources is getting large. A state file with more than 500 resources is a problem waiting to happen.

### State Naming Convention

Your state key naming convention is your operational documentation. Make it hierarchical and consistent:

```
# Format: {product}/{cloud-region}/{environment}/{layer}/terraform.tfstate

# Examples:
payments/aws-us-east-1/production/networking/terraform.tfstate
payments/aws-us-east-1/production/database/terraform.tfstate
payments/aws-us-east-1/production/application/terraform.tfstate
payments/aws-us-east-1/production/monitoring/terraform.tfstate

platform/aws-us-east-1/production/eks-cluster/terraform.tfstate
platform/aws-us-east-1/production/eks-addons/terraform.tfstate

shared/aws-us-east-1/production/dns/terraform.tfstate
shared/aws-us-east-1/production/certificates/terraform.tfstate
```

This naming convention means you can look at a state key and immediately understand: which product owns it, in which cloud and region, in which environment, and at which architectural layer.

### Layered State Architecture

Decompose state by lifecycle and change frequency, not just by resource type.

```
Layered State Architecture
────────────────────────────────────────────────────────────
Layer 0: Account Bootstrap     (changes: quarterly)
  └── IAM boundaries, billing, root config

Layer 1: Networking            (changes: monthly)
  └── VPCs, subnets, transit gateways, peering
  └── State outputs consumed by everything below

Layer 2: Platform Services     (changes: weekly)
  └── EKS/AKS/GKE clusters, shared databases
  └── Secrets management, certificate infrastructure

Layer 3: Application Services  (changes: daily/hourly)
  └── Application deployments, auto-scaling config
  └── Reads networking and platform state as data sources

Layer 4: Observability         (changes: weekly)
  └── Monitoring, alerting, log routing
  └── Consumes outputs from application layer
────────────────────────────────────────────────────────────
```

The key principle: **higher layers never depend on lower layers**. Layer 3 (application) can read Layer 1 (networking) outputs via `terraform_remote_state` data sources, but networking never reads application state. This prevents circular dependencies and ensures that a broken application deployment cannot block a networking change.

### Locking Strategy

Use DynamoDB for state locking in AWS. Configure lock timeout aggressively — a 15-minute lock timeout in CI means a crashed pipeline can block all infrastructure changes for 15 minutes. Set it to 2–3 minutes in CI and use retry logic:

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket               = "org-terraform-state-${var.environment}"
    key                  = "${var.product}/${var.region}/${var.layer}/terraform.tfstate"
    region               = "us-east-1"
    dynamodb_table       = "terraform-state-locks"
    encrypt              = true
    kms_key_id           = data.aws_kms_key.state.arn
    
    # Force path-style for private endpoints
    force_path_style     = false
    
    # Workspace prefix for any workspaces used below env isolation
    workspace_key_prefix = "workspaces"
  }
}
```

### Blast Radius Reduction

The most important state management decision is not the backend type or the naming convention — it's how aggressively you decompose state to limit blast radius.

A production networking state file that includes VPCs, subnets, route tables, NAT gateways, and transit gateway attachments is dangerous. A `terraform apply` error that puts this state into an inconsistent condition could take down all connectivity. Split it:

```
platform/aws-us-east-1/production/
├── vpc/                     # VPC, subnets, route tables only
├── transit-gateway/         # Transit gateway and attachments
├── nat-gateway/             # NAT gateways (can be recreated)
├── security-groups-shared/  # Shared security groups
└── vpc-endpoints/           # Interface/gateway endpoints
```

Each of these can be changed independently. If `nat-gateway` runs into an issue, your VPC is unaffected.

---

## 7. Governance and Standards

At 500+ modules, governance is not a bureaucratic overlay. It is a survival mechanism. Without it, you have entropy that compounds until your infrastructure is unmaintainable.

### Naming Conventions

The naming convention must be machine-enforceable. Human enforcement does not scale.

```hcl
# Naming convention: {org}-{env}-{product}-{resource-type}-{identifier}
# Examples:
#   acme-prod-payments-eks-main
#   acme-staging-platform-rds-primary
#   acme-dev-analytics-s3-raw-data

locals {
  resource_prefix = "${var.org_code}-${var.environment}-${var.product_code}"
  
  name_validators = {
    # Enforced in foundational modules — callers cannot bypass these
    max_length    = 64
    allowed_chars = "[a-z0-9-]"
    no_underscores = true
  }
}

# In foundational modules, validate naming at module entry:
variable "name_suffix" {
  type        = string
  description = "Suffix appended to the resource prefix. Lowercase alphanumeric and hyphens only."
  
  validation {
    condition     = can(regex("^[a-z0-9][a-z0-9-]{0,30}[a-z0-9]$", var.name_suffix))
    error_message = "name_suffix must be lowercase alphanumeric with hyphens, 2-32 chars, no leading/trailing hyphens."
  }
}
```

### Mandatory Tagging

Every resource must have a minimum set of tags. Enforce this in foundational modules, not in documentation.

```hcl
# In your foundational module tag merge logic:
locals {
  # These tags are non-negotiable. Consumers add to them, not replace them.
  mandatory_tags = {
    "org:managed-by"      = "terraform"
    "org:environment"     = var.environment
    "org:product"         = var.product_code
    "org:team"            = var.owning_team
    "org:cost-center"     = var.cost_center
    "org:data-class"      = var.data_classification
    "org:terraform-repo"  = var.terraform_repo
    "org:terraform-stack" = var.terraform_stack
  }
  
  # Consumer tags are merged OVER the mandatory tags except for org: prefixed keys
  final_tags = merge(var.additional_tags, local.mandatory_tags)
}
```

The `org:terraform-repo` and `org:terraform-stack` tags are operationally critical. When an engineer sees an unrecognized resource in the console, these tags tell them exactly where to find the Terraform that manages it.

### Policy as Code

At scale, policy must be automated. There are two complementary approaches:

**Terraform Validate + Custom Rules (pre-apply)**

Use `terraform validate` in CI and augment it with tools like `tflint` with custom rule plugins specific to your organization:

```yaml
# .tflint.hcl (custom rules for your org)
plugin "aws" {
  enabled = true
  version = "0.27.0"
  source  = "github.com/terraform-linters/tflint-ruleset-aws"
}

rule "aws_s3_bucket_lifecycle_rule" {
  enabled = true
}

# Custom org rules:
rule "require_cost_center_tag" {
  enabled = true
}
```

**OPA/Conftest (policy-as-code for plan output)**

Use Open Policy Agent with Conftest to evaluate Terraform plan JSON output against policies before apply:

```rego
# policies/aws/s3_encryption.rego
package main

deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "aws_s3_bucket"
  resource.change.actions[_] == "create"
  not resource.change.after.server_side_encryption_configuration
  
  msg := sprintf(
    "S3 bucket '%s' must have server-side encryption configured",
    [resource.address]
  )
}

deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "aws_s3_bucket"
  not resource.change.after.tags["org:cost-center"]
  
  msg := sprintf(
    "S3 bucket '%s' is missing mandatory tag 'org:cost-center'",
    [resource.address]
  )
}
```

**Sentinel (for HCP Terraform / Terraform Enterprise)**

If you are on the Terraform Enterprise stack, Sentinel gives you policy enforcement as a first-class citizen of the run lifecycle:

```python
# sentinel/require-approved-module-sources.sentinel
import "tfconfig/v2" as tfconfig

approved_sources = [
  "git::https://github.com/org/tf-platform-modules",
  "registry.terraform.io/hashicorp/",
]

main = rule {
  all tfconfig.module_calls as _, calls {
    all calls as _, call {
      any approved_sources as source {
        strings.has_prefix(call.source, source)
      }
    }
  }
}
```

### Code Ownership

Use `CODEOWNERS` files to enforce review requirements. In a hybrid repo setup:

```
# tf-platform-modules/CODEOWNERS

# All foundational modules require platform team approval
/modules/foundational/          @org/platform-team

# AWS-specific foundational modules also require cloud-ops review
/modules/foundational/aws/      @org/platform-team @org/aws-cloud-ops

# Security baseline modules require security team approval
/modules/foundational/aws/iam-role/     @org/platform-team @org/security-team

# Platform modules require platform team + one senior engineer
/modules/platform/              @org/platform-team @org/senior-engineers
```

```
# infra-payments-team/CODEOWNERS

# Production stacks require two payments engineers + platform team
/stacks/aws-us-east-1/production/   @org/payments-engineering @org/platform-team

# All other environments just need payments team
/stacks/                            @org/payments-engineering
```

---

## 8. Versioning Strategy

Module versioning is where most organizations make decisions that look reasonable today and cause migrations that take months of engineering time two years from now.

### Semantic Versioning for Modules

All shared modules must follow semantic versioning. This is not optional at scale.

```
MAJOR.MINOR.PATCH

MAJOR: Breaking changes. Existing consumers must update their configurations.
MINOR: New features, backward compatible. Consumers can adopt at their leisure.
PATCH: Bug fixes, documentation, backward compatible.
```

The hardest discipline to maintain: **be honest about what constitutes a breaking change**.

Breaking changes in Terraform modules include:

- Removing a variable
- Renaming a variable
- Changing a variable's type
- Changing a variable's default in a way that alters behavior
- Renaming an output
- Removing an output
- Changing the type of a resource so it requires destruction and recreation
- Changing the internal resource address (even if behavior is identical) — this causes unwanted destroys

Not breaking changes:

- Adding a new optional variable with a sensible default
- Adding new outputs
- Bug fixes that preserve the intended behavior
- Adding new resources that do not affect existing resource addresses

### Breaking Change Process

When a breaking change is unavoidable (and sometimes they are), follow a structured migration process:

**1. Deprecation notice in MINOR release:**

```hcl
# variables.tf — in v3.2.0

variable "instance_size" {
  type        = string
  description = "DEPRECATED: Use instance_type instead. Will be removed in v4.0.0."
  default     = null
}

variable "instance_type" {
  type        = string
  description = "EC2 instance type. Replaces instance_size."
  default     = "t3.medium"
}
```

**2. Migration guide in CHANGELOG:**

```markdown
## v4.0.0 (Breaking)

### Breaking Changes

- **Removed:** `instance_size` variable. Use `instance_type` instead.
  - Migration: Replace `instance_size = "large"` with `instance_type = "m5.large"`.
  - This change was deprecated in v3.2.0.

### Migration Steps

1. Update all `source` references from `?ref=v3.x.x` to `?ref=v4.0.0`
2. Replace any `instance_size` variable usage with `instance_type`
3. Run `terraform plan` and verify no unexpected changes
4. Apply
```

**3. Automated detection:**

Run a script in CI that scans all consuming repositories for deprecated variable usage before releasing the major version. This tells you how many consumers you need to notify.

### Version Pinning Policy

This must be enforced by policy, not by convention.

```hcl
# GOOD: Pinned to exact version
module "vpc" {
  source  = "git::https://github.com/org/tf-platform-modules//modules/foundational/aws/vpc?ref=v2.3.1"
}

# ACCEPTABLE: Pinned to patch-compatible range
module "vpc" {
  source  = "git::https://github.com/org/tf-platform-modules//modules/foundational/aws/vpc?ref=v2.3.1"
  # Internal Terraform Registry:
  version = "~> 2.3"
}

# NEVER: Pinned to branch or tag without version
module "vpc" {
  source = "git::https://github.com/org/tf-platform-modules//modules/foundational/aws/vpc?ref=main"
  # This means every plan could use different code. Catastrophic at scale.
}
```

Use a linting rule or `conftest` policy to fail CI on any `?ref=main` or `?ref=master` references in production stacks.

---

## 9. CI/CD Architecture

Your CI/CD pipeline for infrastructure is a different class of problem from application CI/CD. The blast radius of a bad deploy is the entire production environment, not a single service. Design accordingly.

### The Core Pipeline Stages

```
PR Opened / Updated
        │
        ▼
┌───────────────────┐
│  1. Static checks  │  terraform fmt, validate, tflint, tfsec
│     (2 min)        │  Fails fast. No cloud API calls.
└─────────┬─────────┘
          │ Pass
          ▼
┌───────────────────┐
│  2. Policy checks  │  conftest + OPA policies against config
│     (2 min)        │  No cloud API calls. Pure logic.
└─────────┬─────────┘
          │ Pass
          ▼
┌───────────────────┐
│  3. Speculative    │  terraform plan -out=plan.tfplan
│     plan (5 min)   │  Read-only cloud creds. Plan stored as artifact.
└─────────┬─────────┘
          │ Pass
          ▼
┌───────────────────┐
│  4. Plan policy    │  conftest against plan JSON output
│     (1 min)        │  Catches what static analysis cannot.
└─────────┬─────────┘
          │ Pass
          ▼
┌───────────────────┐
│  5. Human review   │  Required for staging and production.
│     (async)        │  Plan output shown in PR comment.
└─────────┬─────────┘
          │ Approved
          ▼
┌───────────────────┐
│  6. Apply          │  terraform apply plan.tfplan
│     (variable)     │  Write creds. Uses stored plan only.
└─────────┬─────────┘
          │
          ▼
┌───────────────────┐
│  7. Post-apply     │  Smoke tests, drift check, notification
└───────────────────┘
```

**Critical:** Use the saved plan file for apply. Never run `terraform plan` and then `terraform apply` separately in a pipeline — the infrastructure can change between those two operations, producing a different apply than the reviewed plan.

### Drift Detection

Drift is the silent killer of large Terraform estates. Schedule daily drift detection across all production stacks:

```yaml
# .github/workflows/drift-detection.yml
name: Drift Detection
on:
  schedule:
    - cron: '0 6 * * *'   # Run daily at 6 AM

jobs:
  detect-drift:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        stack:
          - { product: payments, env: production, region: aws-us-east-1 }
          - { product: platform, env: production, region: aws-us-east-1 }
          # ... all production stacks
    steps:
      - name: Terraform Init
        run: terraform init
        working-directory: stacks/${{ matrix.stack.region }}/${{ matrix.stack.env }}
      
      - name: Detect Drift
        id: plan
        run: |
          terraform plan -detailed-exitcode -out=drift.tfplan 2>&1
          echo "exit_code=$?" >> $GITHUB_OUTPUT
        working-directory: stacks/${{ matrix.stack.region }}/${{ matrix.stack.env }}
        continue-on-error: true
      
      - name: Alert on Drift
        if: steps.plan.outputs.exit_code == '2'
        run: |
          # Parse plan output, post to Slack, create Jira ticket
          python scripts/parse-drift-alert.py drift.tfplan
```

`terraform plan -detailed-exitcode` returns exit code 2 when there are changes to apply, 0 when there are none, and 1 on error. This is the mechanism for programmatic drift detection.

### Approval Gates

For production deployments, implement explicit approval gates with time-boxed windows:

```yaml
# GitHub Actions with environment protection rules
environment: production
# GitHub environment rules: require 2 approvers from @org/senior-engineers
# Auto-cancel if not approved within 2 hours
# Only allow deployment from main branch
```

For high-risk changes (anything that involves destroying resources), require explicit acknowledgment in the PR:

```yaml
# In your CI pipeline:
- name: Check for destroys
  run: |
    DESTROYS=$(terraform show -json plan.tfplan | jq '[.resource_changes[] | select(.change.actions[] == "delete")] | length')
    if [ "$DESTROYS" -gt "0" ]; then
      echo "::warning ::This plan includes $DESTROYS resource deletions"
      # Require explicit label on PR to proceed
      if ! gh pr view $PR_NUMBER --json labels | grep -q "approved-destroys"; then
        echo "::error ::Add label 'approved-destroys' to this PR to proceed with deletions"
        exit 1
      fi
    fi
```

### Ephemeral Environments

For product teams that need to test infrastructure changes before they reach staging, implement ephemeral environment creation as part of the PR workflow:

```hcl
# stacks/aws-us-east-1/ephemeral/main.tf
# This stack is created fresh for each PR and destroyed after merge.

locals {
  # Ephemeral env name derived from PR number
  env_name = "pr-${var.pr_number}"
  
  # Ephemeral environments use minimal resource sizes
  resource_scale = "minimal"
}

module "ephemeral_service" {
  source = "..."
  
  environment      = local.env_name
  # Force minimal resources regardless of what was configured
  override_profile = local.resource_scale
  
  # Time-to-live tag — a separate process cleans up expired envs
  additional_tags = {
    "org:ttl"        = "48h"
    "org:pr-number"  = var.pr_number
  }
}
```

---

## 10. Multi-Cloud Realities

Multi-cloud Terraform is harder than most organizations anticipate. The temptation is to create a beautiful abstraction layer where you call the same module and get equivalent infrastructure on AWS, Azure, or GCP. This is largely a fantasy, and pursuing it creates modules of enormous complexity that serve every cloud poorly.

### What Should Be Standardized

**Standardize the interface, not the implementation.**

The things that should be consistent across clouds:

- Naming conventions (the naming schema is cloud-agnostic)
- Tagging/labeling schema (AWS tags, Azure tags, GCP labels are all different mechanisms, same logical schema)
- Security baseline requirements (encryption at rest/transit, access logging, network segmentation)
- Operational metadata (cost center, environment, owning team)
- Module taxonomy and ownership model
- CI/CD pipeline structure and approval requirements

### What Should Be Cloud-Specific

**Never abstract these:**

- Network topology (AWS VPCs, Azure VNets, and GCP VPC Networks are fundamentally different)
- IAM model (AWS IAM, Azure RBAC, and GCP IAM have different primitives — force-fitting them into one module produces unusable code)
- Compute models (EC2 vs. Azure VMs vs. GCE are similar in concept but different in detail that matters)
- Managed Kubernetes (EKS, AKS, and GKE have enough differences that a single module becomes a maze of conditionals)

### The Right Abstraction Level

The right place for multi-cloud abstraction is the **product module tier**, not the foundational tier.

```hcl
# modules/product/object-storage/main.tf
# A multi-cloud object storage module.
# It presents a consistent interface while delegating to cloud-specific modules.

# This works because object storage is conceptually similar enough
# across clouds to justify the abstraction.

module "aws_bucket" {
  count  = var.cloud_provider == "aws" ? 1 : 0
  source = "../../foundational/aws/s3-bucket"
  
  bucket_name         = var.storage_name
  versioning_enabled  = var.versioning_enabled
  # ...
}

module "azure_container" {
  count  = var.cloud_provider == "azure" ? 1 : 0
  source = "../../foundational/azure/storage-account"
  
  account_name = var.storage_name
  # ... azure-specific config
}

module "gcp_bucket" {
  count  = var.cloud_provider == "gcp" ? 1 : 0
  source = "../../foundational/gcp/gcs-bucket"
  
  bucket_name = var.storage_name
  # ... gcp-specific config
}
```

This works for object storage. It does not work for networking, IAM, or Kubernetes. Know where to draw the line.

### Provider Configuration at Scale

With multiple clouds and multiple accounts, provider configuration becomes complex. Use a consistent pattern:

```hcl
# providers.tf — in each stack configuration

provider "aws" {
  region = var.aws_region
  
  # Always assume a role — never use long-lived credentials
  assume_role {
    role_arn     = "arn:aws:iam::${var.aws_account_id}:role/TerraformDeployRole"
    session_name = "terraform-${var.product}-${var.environment}"
    external_id  = var.aws_external_id
  }
  
  default_tags {
    tags = {
      "org:managed-by"  = "terraform"
      "org:environment" = var.environment
    }
  }
}

provider "azurerm" {
  features {}
  
  subscription_id = var.azure_subscription_id
  tenant_id       = var.azure_tenant_id
  
  # Use workload identity in CI, interactive auth in local dev
  use_oidc = true
}

provider "google" {
  project = var.gcp_project_id
  region  = var.gcp_region
}
```

---

## 11. Anti-Patterns

These are the patterns I see most often in organizations that have hit a wall with Terraform at scale. Each one was a rational decision at the time. None of them survive contact with 500+ modules.

### The God Module

A module that does everything: creates the VPC, the EKS cluster, the RDS instance, the S3 buckets, the IAM roles, and the CloudWatch alarms. It has 150 variables and manages 300 resources.

**Why it happens:** It starts as a "complete environment" module. It accretes features because adding a variable is always easier than creating a new module.

**Why it fails:** The blast radius is enormous. `terraform plan` takes 20 minutes. Nobody understands the full dependency graph. Changes to one part unexpectedly affect another. When something breaks, the blast radius is your entire environment.

**Fix:** Apply the layer model from Section 6. Decompose by lifecycle: networking, platform services, application services, observability. Each is a separate state.

### Branch-Pinned Modules

```hcl
# This will eventually destroy your on-call rotation
module "vpc" {
  source = "git::https://github.com/org/modules.git//vpc?ref=main"
}
```

**Why it happens:** Teams want to stay current without manual version bumps.

**Why it fails:** Two `terraform plan` runs on the same configuration one week apart can produce different results because `main` moved. A refactor in the shared module breaks every consumer simultaneously with no warning.

**Fix:** Enforce version pinning via CI policy. No exceptions.

### The Catch-All `variables.tf`

```hcl
# 300-line variables.tf that exposes every possible configuration option
# for every resource in the module
variable "enable_deletion_protection" { ... }
variable "deletion_protection_override_reason" { ... }
variable "detailed_monitoring" { ... }
variable "detailed_monitoring_exception_ticket" { ... }
# ... 296 more variables
```

**Why it happens:** Module authors try to make the module flexible enough for every possible use case.

**Why it fails:** The module becomes unusable. Callers don't know which variables they need to set. Defaults get wrong. The surface area for misconfiguration is enormous.

**Fix:** Opinionated defaults. Expose only what genuinely needs to vary between consumers. If 95% of consumers always set `deletion_protection = true`, make it the default and don't expose the override. You can always add a variable later; removing one is a breaking change.

### Shared State for Unrelated Resources

```hcl
# One state file containing:
# - Production VPC networking
# - Development VPC networking
# - Staging database
# - Production database
# - Shared DNS zone
# - All TLS certificates
# - S3 buckets for 12 different products
```

**Why it happens:** State management feels complex and teams defer the work of decomposing it.

**Why it fails:** A bug in applying a dev DNS record takes down your ability to modify production networking. Lock contention is constant. Plan times grow without bound.

**Fix:** Decompose early and aggressively. The right time to decompose is before you have the scale problem, not after.

### Workspace-as-Environment at Scale

```bash
# This seems elegant until you have 50 environments
terraform workspace new production
terraform workspace new staging
terraform workspace new dev-team-a
terraform workspace new dev-team-b-feature-x
terraform workspace new dev-team-b-feature-x-experiment-2
```

**Why it happens:** Workspaces look like environments. The Terraform documentation even suggests this use case.

**Why it fails:** All workspaces share a single backend configuration and IAM permissions. There's no hard boundary between production and development. `terraform workspace select production && terraform apply` from a developer's laptop is a real accident waiting to happen.

**Fix:** Use separate backend configurations per environment tier. Keep workspace usage for truly ephemeral environments (PR previews, short-lived experiments) with automatic TTL and cleanup.

---

## 12. Reference Architecture

Here is the operating model I recommend for a 500+ module organization. This is not a template — it is a set of principles applied to a concrete example.

### Repository Structure

```
github.com/acme-corp/

  tf-platform-modules/          # Platform team owns this
  │  Versioned releases via GitHub Releases
  │  ~150 foundational modules, ~30 platform modules
  │  Full test suite (Terratest)
  │  OPA policies enforced in CI
  │
  tf-security-baseline/          # Security team owns this
  │  OPA policies consumed by all other repos
  │  Sentinel policies for HCP Terraform
  │  CIS benchmark validation
  │
  infra-platform/                # Platform team owns this
  │  Account/project bootstrapping
  │  Shared networking (transit gateways, DNS)
  │  EKS/AKS/GKE platform clusters
  │  State: platform/*
  │
  infra-{product-team-name}/     # Each product team owns theirs
     Application-layer infrastructure
     Consumes platform modules at pinned versions
     State: {product}/*
     Owns environment promotion in their CI
```

### CI/CD Model

```
Pull Request Flow:
  fmt → validate → tflint → conftest (static) → speculative plan
  → conftest (plan) → human review → apply (from saved plan)

Drift Detection:
  Daily scheduled job across all production stacks
  Alert to Slack on any drift detected
  Auto-create Jira ticket for unresolved drift > 24h

Module Release Flow:
  PR to tf-platform-modules
  → Full test suite (unit + integration)
  → Policy validation
  → Human review (CODEOWNERS enforced)
  → Merge to main
  → Create GitHub Release (triggers changelog, tag)
  → Dependabot PRs created in consuming repos (optional)
```

### Governance Model

```
Decision Rights:
  Foundational module changes:    Platform team + Security team
  Platform module changes:        Platform team (+ Security for IAM modules)
  Product module changes:         Platform team + owning product team
  Application stack changes:      Owning product team (+ Platform for prod)
  Policy changes:                 Security team + Platform team

Review Requirements:
  Production stacks:              2 approvals (1 from platform team)
  Staging stacks:                 1 approval (from owning team)
  Development stacks:             1 approval (from owning team)
  Foundational modules:           2 approvals (1 from platform, 1 from security)

Metrics Tracked:
  Drift detection coverage:       % of production stacks with daily drift check
  Module adoption lag:            Avg versions behind latest for consuming stacks
  Plan time:                      P95 plan time per stack (target: < 5 min)
  Breaking change rate:           Major version releases per quarter
  Policy violation rate:          Policy failures caught in CI per week
```

---

## 13. What I'd Do Starting From Scratch

If I were joining an organization today and tasked with building a Terraform foundation that could scale to 500+ modules, here is the exact sequence of decisions I would make, and why.

**Month 1: Foundation before features**

Before writing a single module, I would define and document:
- The module taxonomy (four tiers: foundational, platform, product, application)
- The naming convention (machine-enforceable, hierarchical, documented)
- The state organization (layered by lifecycle, one backend per environment tier)
- The mandatory tag set (6–8 tags, enforced in code, not in documentation)
- The repository structure (hybrid: platform modules repo + team repos)
- The versioning policy (semver, no branch pins, breaking change process)

None of this requires writing any Terraform. All of it prevents decisions that are extremely painful to reverse later.

**Month 2: CI/CD before scale**

Before the module count grows past 20, build the full CI/CD pipeline:
- Static checks (fmt, validate, tflint)
- Policy as code (OPA/Conftest with initial policies)
- Speculative plan in PRs with output as PR comment
- Drift detection for the first production stacks
- State lock cleanup automation

The pipeline is your enforcement mechanism. Without it, every standard you document becomes a suggestion.

**Month 3: Start with foundational modules, not platform modules**

The instinct is to build high-level "easy to use" platform modules first. Resist it. Foundational modules are harder to build well but create the quality floor that everything else rests on. Five well-built foundational modules are more valuable than twenty platform modules built on inconsistent foundations.

**Months 4–6: Build the platform layer conservatively**

Add platform modules only when you have clear demand — when two or more teams are asking for the same architectural pattern. Platform modules built speculatively become maintenance burden before they have users.

**The non-negotiables from day one:**

1. No branch pins in any module source
2. Mandatory tags enforced in foundational modules
3. Separate backend per environment tier
4. All production applies require a human approval
5. Drift detection running before you have more than 10 stacks
6. A documented breaking change process before the first platform module is released

**The thing I would resist most:**

Building a "beautiful" multi-cloud abstraction layer early. It is seductive. It will consume months of engineering time and produce a module so complex that nobody wants to use it. Build cloud-specific foundational modules first. Build multi-cloud abstractions only when you have proven demand for the same capability across multiple clouds.

---

## Summary Checklist

### Foundation

- [ ] Module taxonomy defined (foundational / platform / product / application)
- [ ] Repository structure decided (mono / multi / hybrid)
- [ ] Naming convention documented and machine-enforceable
- [ ] Mandatory tag set defined and enforced in foundational modules
- [ ] State naming convention defined
- [ ] State decomposed by lifecycle layer (networking / platform / application / observability)
- [ ] Separate state backends per environment tier (dev / staging / prod)
- [ ] Version pinning policy enforced (no branch references in production)

### Governance

- [ ] CODEOWNERS configured for all repositories
- [ ] LimitRange / policy as code for PR reviews on foundational modules
- [ ] OPA/Conftest policies for static analysis and plan analysis
- [ ] Breaking change process documented and tested with at least one major version
- [ ] Semantic versioning enforced for all shared modules
- [ ] Module discovery mechanism (internal registry or documentation site)

### Operations

- [ ] Full CI/CD pipeline: fmt → validate → lint → policy → plan → review → apply
- [ ] Saved plan file used for apply (not re-plan at apply time)
- [ ] Daily drift detection across all production stacks
- [ ] Drift alerting to on-call channel
- [ ] State lock timeout configured and cleanup automation in place
- [ ] `terraform plan -detailed-exitcode` used for programmatic drift detection
- [ ] Post-apply smoke tests for critical stacks

### Multi-Cloud

- [ ] Cloud-specific foundational modules rather than forced abstractions
- [ ] Multi-cloud abstractions limited to conceptually equivalent resources (object storage, DNS)
- [ ] Provider configuration uses role assumption / workload identity (no long-lived keys)
- [ ] Default tags/labels configured at provider level
- [ ] Clear documentation of which capabilities are standardized vs. cloud-specific

### Anti-Pattern Prevention

- [ ] No "god modules" — decompose anything managing >50 resources
- [ ] No workspace-as-environment for production/staging
- [ ] No branch-pinned module sources
- [ ] LimitRange on variables.tf size (enforce opinionated defaults)
- [ ] No shared state for resources owned by different teams or lifecycles

---

*Platform Engineering · Infrastructure · Terraform at Scale*

---

> **A note on tooling**: This article is intentionally tool-agnostic where possible. The patterns described work whether you are using HCP Terraform, Atlantis, GitHub Actions, GitLab CI, or a home-grown pipeline. The organizational and structural decisions matter more than which CI/CD tool you choose. Pick the tool that fits your organization's existing workflow. Change the structure to match these patterns.
