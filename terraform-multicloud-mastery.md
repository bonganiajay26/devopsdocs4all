# Terraform Multi-Cloud Mastery

**A progressive learning system from beginner to production-ready expert**
**Author Persona:** Senior DevOps Architect | Multi-cloud (AWS, Azure, GCP)

---

## How To Use This Guide

Each level has two parts:
- **Part A — Concepts:** what to know, why it matters, what to avoid.
- **Part B — Hands-on Lab:** real infrastructure you can build, file by file.

Levels build on each other. Don't skip the labs — Terraform reveals itself only when you watch a plan turn into real resources, and then watch it destroy them cleanly.

---
---

# LEVEL 1 — BEGINNER

## PART A — CONCEPTS

### What Terraform Actually Is
Terraform is a declarative infrastructure-as-code (IaC) tool. You describe the *desired end state* of your infrastructure in HCL (HashiCorp Configuration Language). Terraform figures out the API calls needed to reach that state and stores a record of what it built in a **state file**.

Three things to internalize on day one:
1. **Declarative, not imperative.** You don't tell it *how* to build — you tell it *what* you want.
2. **State is sacred.** The state file is Terraform's memory. Lose it and Terraform forgets your infrastructure exists.
3. **Providers are plugins.** AWS, Azure, GCP, Datadog, GitHub — every integration is a provider.

### Core Vocabulary
- **Provider** — Plugin that talks to a cloud/SaaS API (e.g., `hashicorp/aws`).
- **Resource** — A managed object (an EC2 instance, a GCS bucket).
- **Data source** — A read-only lookup (existing VPC, latest AMI).
- **Variable** — Input parameter to your configuration.
- **Output** — Value exposed after apply (an IP address, an ARN).
- **State** — JSON file mapping config → real-world resources.

### Real-world use cases (beginner scope)
- Spin up a single VM or storage bucket.
- Provision a developer sandbox.
- Codify a DNS zone.

### Best Practices
- **Always pin provider versions.** Floating versions cause silent breakage on Monday morning.
- **Use `.tfvars` files**, never hard-code secrets or environment-specific values.
- **`terraform fmt` and `terraform validate`** on every save.
- **Commit `.terraform.lock.hcl`**, never commit `.terraform/` or `*.tfstate`.

### Common Mistakes
- Editing the state file by hand. (Use `terraform state` commands instead.)
- Storing state in Git. (It contains secrets in plaintext.)
- Hard-coding regions, AMIs, project IDs.
- Running `apply` without reading the plan output.

### Pro Tips
- Read the plan **out loud**. If you can't explain each `+`, `~`, and `-`, stop and investigate.
- Use `terraform console` to test expressions interactively — faster than guessing.
- The first time you run `apply`, do it on something you can lose. The first time you run `destroy`, also do it on something you can lose.

---

## PART B — HANDS-ON LAB: Your First AWS Resource

### 1. Objective
Provision an S3 bucket on AWS with versioning enabled and tags applied — using a clean module-free layout to learn the basics.

### 2. Architecture (text diagram)
```
+--------------------------+
|  Local machine (HCL)     |
|     terraform CLI        |
+------------+-------------+
             |
             | AWS API (HTTPS)
             v
+--------------------------+
|   AWS account / region   |
|                          |
|   [ S3 Bucket ]          |
|     - versioning: on     |
|     - tags applied       |
+--------------------------+
```

### 3. Step-by-Step
1. Create a folder `lab1-beginner/`.
2. Create the four files below.
3. Export AWS credentials (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`).
4. Run `init`, `plan`, `apply`.
5. Verify the bucket in the AWS console.
6. Run `destroy`.

### 4. Files

#### `versions.tf`
**What:** Locks Terraform and provider versions.
**Why:** Reproducibility. The same code should produce the same plan a year from now.
**When:** Every project. Always.

```hcl
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.40"
    }
  }
}
```
**Key lines:**
- `required_version` — fail fast if someone uses an old CLI.
- `~> 5.40` — pessimistic constraint: allow 5.40.x and 5.41.x, but never 6.x.

#### `providers.tf`
**What:** Configures the AWS provider.
**Why:** Tells Terraform *which account and region* to act in.

```hcl
provider "aws" {
  region = var.region

  default_tags {
    tags = {
      Project     = var.project
      Environment = var.environment
      ManagedBy   = "Terraform"
    }
  }
}
```
**Key lines:**
- `default_tags` — applied to every taggable resource. Saves you from forgetting tags on resource #47.

#### `variables.tf`
**What:** Declares inputs.
**Why:** Removes hard-coded values; makes the config reusable across environments.

```hcl
variable "region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "project" {
  description = "Project name (used for tagging and bucket naming)"
  type        = string
}

variable "environment" {
  description = "Environment name: dev, staging, prod"
  type        = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "environment must be dev, staging, or prod."
  }
}
```
**Key lines:**
- `validation` block — fail at `plan` time, not after deploying garbage.

#### `main.tf`
**What:** The actual resources.

```hcl
resource "random_id" "suffix" {
  byte_length = 4
}

resource "aws_s3_bucket" "this" {
  bucket = "${var.project}-${var.environment}-${random_id.suffix.hex}"
}

resource "aws_s3_bucket_versioning" "this" {
  bucket = aws_s3_bucket.this.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_public_access_block" "this" {
  bucket                  = aws_s3_bucket.this.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```
**Key lines:**
- `random_id` — bucket names are globally unique; the suffix prevents collisions.
- `public_access_block` — secure-by-default; never ship a bucket without this.

> Add `random` to `required_providers`:
> ```hcl
> random = { source = "hashicorp/random", version = "~> 3.6" }
> ```

#### `outputs.tf`
**What:** Surfaces useful values after apply.

```hcl
output "bucket_name" {
  description = "Name of the created S3 bucket"
  value       = aws_s3_bucket.this.bucket
}

output "bucket_arn" {
  value = aws_s3_bucket.this.arn
}
```

#### `terraform.tfvars`
```hcl
project     = "tf-mastery"
environment = "dev"
region      = "us-east-1"
```

### 5. Commands
```bash
terraform init      # download providers, build .terraform/
terraform plan      # show what WILL happen — read every line
terraform apply     # execute; type "yes" to confirm
terraform destroy   # tear it all down
```

### 6. Expected Output
```
Apply complete! Resources: 4 added, 0 changed, 0 destroyed.

Outputs:
bucket_arn  = "arn:aws:s3:::tf-mastery-dev-7af3c901"
bucket_name = "tf-mastery-dev-7af3c901"
```

### Advanced Thinking Layer
- **Why `random_id` over a timestamp?** Timestamps drift across runs and cause unnecessary recreation. `random_id` is generated once and stored in state.
- **Trade-off:** Hard-coding the bucket name is simpler but breaks reusability. The cost of one extra resource is worth it.
- **Anti-pattern:** Using `aws_s3_bucket`'s deprecated inline `versioning {}` block. Modern AWS provider splits each concern into its own resource. Embrace it.

---
---

# LEVEL 2 — INTERMEDIATE

## PART A — CONCEPTS

### Remote State and Backends
The local `terraform.tfstate` works for one person on one laptop. The moment a teammate joins, you need a **remote backend** with **state locking**.

| Backend | Locking | Best for |
|---|---|---|
| `s3` + DynamoDB | Yes | AWS-centric teams |
| `azurerm` | Yes (blob lease) | Azure-centric teams |
| `gcs` | Yes | GCP-centric teams |
| Terraform Cloud | Yes | Mixed/multi-cloud, managed UX |

### Modules — The Reusability Unit
A **module** is just a folder with `.tf` files. You call it from another configuration. Modules give you:
- DRY infrastructure code.
- Versioned, testable building blocks.
- A contract (inputs/outputs) between teams.

Module structure:
```
modules/
  vpc/
    main.tf
    variables.tf
    outputs.tf
    README.md
```

### Workspaces and Environments
Terraform workspaces let one config track multiple state files (`dev`, `staging`, `prod`). They're **fine for small teams** but break down at scale.

> **Pro tip:** For real production, prefer **directory-per-environment** over workspaces. Workspaces hide environment differences; directories make them explicit.

### Data Sources
Use `data` blocks to *look up* things Terraform doesn't manage:
```hcl
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
}
```

### Dependency Handling
Terraform builds an implicit dependency graph from references. When you can't express a dependency naturally, use `depends_on` — but treat it as a smell, not a tool.

### Best Practices
- One backend per environment, locked down with IAM.
- Module versioning via Git tags or a registry.
- Outputs from a module should be **stable contracts** — don't rename them lightly.
- Group resources by lifecycle, not by type.

### Common Mistakes
- Using a single state file for everything. Blast radius = entire company.
- Modules that take 50 inputs. If your module needs 50 knobs, it's two modules.
- `depends_on` everywhere. Usually means missing references.

### Pro Tips
- **`terraform state list` and `terraform state show`** are your investigation tools, not editing tools.
- Use `for_each` over `count` whenever items have stable identities. `count` re-orders on insertion and causes destructive diffs.

---

## PART B — HANDS-ON LAB: Modular AWS VPC with Remote State

### 1. Objective
Build a reusable VPC module, store state in S3 with DynamoDB locking, and consume the module from a `dev` environment.

### 2. Architecture
```
+----------------------- AWS Account -----------------------+
|                                                           |
|  S3 (tf-state)  <------ state ------+                     |
|  DynamoDB (tf-locks) <-- lock ------+                     |
|                                                           |
|  +------------------ VPC (10.0.0.0/16) ------------------+|
|  |                                                       ||
|  | +-- public subnet AZ-a --+   +-- public subnet AZ-b --+|
|  | | IGW route               |   | IGW route             ||
|  | +-------------------------+   +-----------------------+|
|  |                                                       ||
|  | +-- private subnet AZ-a -+   +-- private subnet AZ-b -+|
|  | | NAT GW route            |   | NAT GW route          ||
|  | +-------------------------+   +-----------------------+|
|  +-------------------------------------------------------+|
+-----------------------------------------------------------+
```

### 3. Step-by-Step
1. Bootstrap the backend (one-time): create S3 bucket + DynamoDB table.
2. Build the VPC module under `modules/vpc/`.
3. Build a `dev/` environment that consumes it.
4. `init` against the remote backend, then `plan`/`apply`.

### 4. Files

#### `bootstrap/main.tf` (run once, with local state)
**What:** Creates the resources Terraform itself needs for remote state.
**Why:** Chicken-and-egg: state backends must exist before you point Terraform at them.
**When:** Once per account/org. Then commit and forget.

```hcl
resource "aws_s3_bucket" "tf_state" {
  bucket = "tf-mastery-state-${data.aws_caller_identity.me.account_id}"
}

resource "aws_s3_bucket_versioning" "tf_state" {
  bucket = aws_s3_bucket.tf_state.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "tf_state" {
  bucket = aws_s3_bucket.tf_state.id
  rule {
    apply_server_side_encryption_by_default { sse_algorithm = "AES256" }
  }
}

resource "aws_dynamodb_table" "tf_locks" {
  name         = "tf-mastery-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  attribute {
    name = "LockID"
    type = "S"
  }
}

data "aws_caller_identity" "me" {}
```
**Key lines:**
- Versioning on the state bucket — recover from accidental corruption.
- Encryption — state files contain secrets.
- DynamoDB `LockID` schema — required by the S3 backend's locking contract.

#### `modules/vpc/variables.tf`
```hcl
variable "name"        { type = string }
variable "cidr_block"  { type = string }
variable "azs"         { type = list(string) }
variable "public_subnet_cidrs"  { type = list(string) }
variable "private_subnet_cidrs" { type = list(string) }
variable "tags"        { type = map(string)  default = {} }
```

#### `modules/vpc/main.tf`
**What:** The reusable VPC.
**Why:** Every team in the org will need a VPC; encode it once correctly.

```hcl
resource "aws_vpc" "this" {
  cidr_block           = var.cidr_block
  enable_dns_hostnames = true
  enable_dns_support   = true
  tags = merge(var.tags, { Name = var.name })
}

resource "aws_internet_gateway" "this" {
  vpc_id = aws_vpc.this.id
  tags   = merge(var.tags, { Name = "${var.name}-igw" })
}

resource "aws_subnet" "public" {
  for_each                = { for idx, az in var.azs : az => idx }
  vpc_id                  = aws_vpc.this.id
  cidr_block              = var.public_subnet_cidrs[each.value]
  availability_zone       = each.key
  map_public_ip_on_launch = true
  tags = merge(var.tags, { Name = "${var.name}-public-${each.key}", Tier = "public" })
}

resource "aws_subnet" "private" {
  for_each          = { for idx, az in var.azs : az => idx }
  vpc_id            = aws_vpc.this.id
  cidr_block        = var.private_subnet_cidrs[each.value]
  availability_zone = each.key
  tags = merge(var.tags, { Name = "${var.name}-private-${each.key}", Tier = "private" })
}

resource "aws_eip" "nat" {
  for_each = aws_subnet.public
  domain   = "vpc"
}

resource "aws_nat_gateway" "this" {
  for_each      = aws_subnet.public
  allocation_id = aws_eip.nat[each.key].id
  subnet_id     = each.value.id
  tags          = merge(var.tags, { Name = "${var.name}-nat-${each.key}" })
}

# Route tables (public + private per AZ) omitted for brevity — see outputs.
```
**Key lines:**
- `for_each` keyed by AZ — adding/removing an AZ doesn't shuffle indices.
- `merge(var.tags, ...)` — caller-supplied tags always win or compose cleanly.

#### `modules/vpc/outputs.tf`
```hcl
output "vpc_id"             { value = aws_vpc.this.id }
output "public_subnet_ids"  { value = [for s in aws_subnet.public  : s.id] }
output "private_subnet_ids" { value = [for s in aws_subnet.private : s.id] }
```

#### `environments/dev/backend.tf`
**What:** Tells Terraform where state lives.
**Why:** Shared, locked state is the foundation of team collaboration.

```hcl
terraform {
  backend "s3" {
    bucket         = "tf-mastery-state-123456789012"
    key            = "dev/network/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "tf-mastery-locks"
    encrypt        = true
  }
}
```
**Key lines:**
- `key` includes environment + component — never collide state across components.
- `dynamodb_table` — without this, two engineers can corrupt state simultaneously.

#### `environments/dev/main.tf`
```hcl
module "vpc" {
  source = "../../modules/vpc"

  name       = "dev"
  cidr_block = "10.0.0.0/16"
  azs        = ["us-east-1a", "us-east-1b"]

  public_subnet_cidrs  = ["10.0.0.0/24",  "10.0.1.0/24"]
  private_subnet_cidrs = ["10.0.10.0/24", "10.0.11.0/24"]

  tags = {
    Environment = "dev"
    Owner       = "platform-team"
  }
}
```

### 5. Commands
```bash
# One-time
cd bootstrap && terraform init && terraform apply

# Then for the env
cd ../environments/dev
terraform init        # configures the S3 backend
terraform plan
terraform apply
terraform destroy
```

### 6. Expected Output
A VPC, 2 public subnets, 2 private subnets, 2 NAT gateways, IGW, EIPs — all visible in the AWS console, all locked behind a remote state file.

### Advanced Thinking Layer
- **Why `for_each` over `count`?** If you remove `us-east-1a` from a `count`-indexed list, every later subnet shifts index → Terraform plans to destroy and recreate them. With `for_each`, only the removed AZ is affected.
- **Why one state file per component?** Smaller state = faster plans, smaller blast radius, parallel changes by different teams.
- **Anti-pattern:** A "god module" that creates VPC + EKS + RDS + IAM. Compose small modules instead.

---
---

# LEVEL 3 — ADVANCED

## PART A — CONCEPTS

### Multi-Cloud — Honestly
Multi-cloud is *not* "the same workload running on three clouds." That's a fantasy that costs millions. Real multi-cloud means:
1. **Best-tool-per-job** (e.g., GCP BigQuery, AWS S3, Azure AD).
2. **Per-region/per-customer placement** (data residency).
3. **Disaster-recovery diversity** (cross-cloud failover for the few things that justify it).

Terraform's value: a single tool, single workflow, and a single grammar across all three.

### Multi-cloud configuration patterns

**Pattern A — One config, multiple providers**
```hcl
provider "aws"     { region  = "us-east-1" }
provider "azurerm" { features {} }
provider "google"  { project = "my-proj"; region = "us-central1" }
```
Use when resources are tightly coupled (e.g., AWS Route53 record pointing at an Azure endpoint).

**Pattern B — Provider aliases for multi-region/multi-account**
```hcl
provider "aws"            { region = "us-east-1" }
provider "aws" { alias = "eu"; region = "eu-west-1" }

resource "aws_s3_bucket" "eu_logs" {
  provider = aws.eu
  bucket   = "..."
}
```

**Pattern C — Per-cloud root modules, glued by shared modules**
Each cloud has its own root module and state. Cross-cloud references happen via outputs in a shared registry or via `terraform_remote_state`.

### Provisioners and lifecycle rules
- **Provisioners (`local-exec`, `remote-exec`)** are a last resort. Prefer cloud-init, AMIs baked with Packer, or config-management.
- **Lifecycle blocks** are powerful and dangerous:
  - `create_before_destroy = true` — zero-downtime replacements.
  - `prevent_destroy = true` — guards prod databases.
  - `ignore_changes = [tags["LastModified"]]` — ignore drift you don't care about.

### Drift detection and recovery
Drift = real world ≠ state file. Sources: hand-edits, other tools, emergency hotfixes.
- Detect: `terraform plan` (shows the diff).
- Reconcile: either `terraform apply` (force config to win) or `terraform import`/`state rm` (force reality to win).

### CI/CD Integration
Treat Terraform like application code:
- PR → `terraform fmt -check`, `validate`, `plan` posted as a comment.
- Merge to main → `apply` from a protected runner with cloud credentials in OIDC, **never** long-lived secrets.
- Use environment promotion (dev → staging → prod) gated by approvals.

### Best Practices
- One state per (environment × component × region).
- Workload identity / OIDC for CI runners — no static cloud keys.
- Tag everything with `Owner`, `CostCenter`, `Environment`, `ManagedBy`.

### Common Mistakes
- Mixing manual changes with Terraform-managed resources.
- Cross-state references via copy-pasted IDs instead of `terraform_remote_state` or SSM/Parameter Store.
- Single global state — the "monorepo of doom."

### Pro Tips
- `terraform plan -out=tfplan` then `terraform apply tfplan` — guarantees what you reviewed is what runs.
- For destructive changes, run `plan` twice, several minutes apart, in production. Drift caught here is gold.

---

## PART B — HANDS-ON LAB: Multi-Cloud Site (AWS + Azure + GCP) with CI/CD

### 1. Objective
Provision:
- An **S3 static site** (AWS).
- An **Azure Storage account** for asset backups.
- A **GCS bucket** for analytics exports.
- A **Route53 record** that resolves to the AWS site.

Wire it through **GitHub Actions** with OIDC for AWS auth.

### 2. Architecture
```
                 +-------------------+
   PR / merge -->|  GitHub Actions   |
                 |  (OIDC -> AWS)    |
                 +---------+---------+
                           |
                           v
+------------------ Terraform ------------------+
|   provider aws       provider azurerm         |
|   provider google                             |
+----+-------------+--------------+-------------+
     |             |              |
     v             v              v
+---------+   +---------+   +-----------+
|  AWS    |   |  Azure  |   |   GCP     |
| S3 site |   | Storage |   | GCS bkt   |
| Route53 |   | Account |   | (export)  |
+---------+   +---------+   +-----------+
```

### 3. Step-by-Step
1. Set up backend (S3+DynamoDB) — reuse Level 2 bootstrap.
2. Configure cloud auth: AWS via OIDC, Azure via service principal, GCP via workload identity federation.
3. Author the multi-provider config below.
4. Wire GitHub Actions.
5. Open a PR → see the plan posted automatically.

### 4. Files

#### `versions.tf`
```hcl
terraform {
  required_version = ">= 1.6.0"
  required_providers {
    aws     = { source = "hashicorp/aws",     version = "~> 5.40" }
    azurerm = { source = "hashicorp/azurerm", version = "~> 3.100" }
    google  = { source = "hashicorp/google",  version = "~> 5.20" }
    random  = { source = "hashicorp/random",  version = "~> 3.6" }
  }
  backend "s3" {
    bucket         = "tf-mastery-state-123456789012"
    key            = "prod/multicloud/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "tf-mastery-locks"
    encrypt        = true
  }
}
```
**Why:** Every provider, every version, pinned. The backend block is the single source of truth for *where* state lives.

#### `providers.tf`
```hcl
provider "aws" {
  region = var.aws_region
  default_tags { tags = local.common_tags }
}

provider "azurerm" {
  features {}
  subscription_id = var.azure_subscription_id
}

provider "google" {
  project = var.gcp_project
  region  = var.gcp_region
}
```
**Key lines:**
- `features {}` — required even if empty for the AzureRM provider.
- Per-provider region/project keeps placement explicit.

#### `locals.tf`
```hcl
locals {
  common_tags = {
    Project     = "multicloud-site"
    Environment = var.environment
    ManagedBy   = "Terraform"
    Owner       = "platform-team"
  }
}
```

#### `aws.tf`
```hcl
resource "random_id" "suffix" { byte_length = 4 }

resource "aws_s3_bucket" "site" {
  bucket = "site-${var.environment}-${random_id.suffix.hex}"
}

resource "aws_s3_bucket_website_configuration" "site" {
  bucket = aws_s3_bucket.site.id
  index_document { suffix = "index.html" }
  error_document { key    = "404.html" }
}

data "aws_route53_zone" "primary" {
  name = var.dns_zone_name
}

resource "aws_route53_record" "site" {
  zone_id = data.aws_route53_zone.primary.zone_id
  name    = "site.${var.dns_zone_name}"
  type    = "A"
  alias {
    name                   = aws_s3_bucket_website_configuration.site.website_domain
    zone_id                = aws_s3_bucket.site.hosted_zone_id
    evaluate_target_health = false
  }
}
```

#### `azure.tf`
```hcl
resource "azurerm_resource_group" "backups" {
  name     = "rg-backups-${var.environment}"
  location = var.azure_region
  tags     = local.common_tags
}

resource "azurerm_storage_account" "backups" {
  name                     = "stbackups${var.environment}${random_id.suffix.hex}"
  resource_group_name      = azurerm_resource_group.backups.name
  location                 = azurerm_resource_group.backups.location
  account_tier             = "Standard"
  account_replication_type = "GRS"
  min_tls_version          = "TLS1_2"
  tags                     = local.common_tags
}
```

#### `gcp.tf`
```hcl
resource "google_storage_bucket" "exports" {
  name          = "exports-${var.gcp_project}-${random_id.suffix.hex}"
  location      = var.gcp_region
  force_destroy = false

  uniform_bucket_level_access = true
  versioning { enabled = true }

  lifecycle_rule {
    condition { age = 90 }
    action    { type = "Delete" }
  }

  labels = {
    environment = var.environment
    managed_by  = "terraform"
  }
}
```

#### `outputs.tf`
```hcl
output "site_url"        { value = "http://site.${var.dns_zone_name}" }
output "azure_storage"   { value = azurerm_storage_account.backups.name }
output "gcs_bucket_name" { value = google_storage_bucket.exports.name }
```

#### `.github/workflows/terraform.yml`
**What:** CI pipeline.
**Why:** Code review for infrastructure. Plans visible in PRs; applies are gated.

```yaml
name: terraform

on:
  pull_request:
  push:
    branches: [main]

permissions:
  id-token: write   # required for AWS OIDC
  contents: read
  pull-requests: write

jobs:
  terraform:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-actions-terraform
          aws-region: us-east-1

      - uses: azure/login@v2
        with:
          client-id:       ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id:       ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: projects/123/locations/global/workloadIdentityPools/gh/providers/gh
          service_account: terraform@my-proj.iam.gserviceaccount.com

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.7.5

      - run: terraform fmt -check
      - run: terraform init
      - run: terraform validate

      - name: Plan
        if: github.event_name == 'pull_request'
        run: terraform plan -no-color -out=tfplan

      - name: Apply
        if: github.ref == 'refs/heads/main'
        run: terraform apply -auto-approve
```
**Key lines:**
- `id-token: write` — needed for cloud OIDC. No static keys.
- `terraform plan -out=tfplan` — apply exactly what was reviewed (in production, you'd save and reuse this artifact).

### 5. Commands
```bash
terraform init
terraform plan -var-file=prod.tfvars
terraform apply -var-file=prod.tfvars
terraform destroy -var-file=prod.tfvars
```

### 6. Expected Output
- A live AWS S3 site at `site.example.com`.
- Azure storage account holding nightly backups.
- GCS bucket with a 90-day lifecycle rule.
- A successful GitHub Actions run posting the plan to the PR.

### Advanced Thinking Layer
- **Why three providers in one config?** Because the Route53 record *depends on* AWS, while the storage targets are loosely coupled. One state file gives you atomic plans across the dependency.
- **Trade-off:** Larger blast radius vs. atomicity. Acceptable here because the resources are small and tightly related.
- **Anti-pattern:** Cross-cloud "active-active" by re-implementing every resource three times. Multi-cloud is about *placement*, not *parity*.

---
---

# LEVEL 4 — EXPERT

## PART A — CONCEPTS

### Production Multi-Cloud Patterns

**1. Layered architecture**
```
   bootstrap/        # state, OIDC, org-wide IAM (rarely changes)
   platform/         # VPCs, k8s clusters, central DNS
   shared-services/  # observability, secrets manager, KMS
   workloads/<app>/  # the application itself
```
Each layer has its own state and CI pipeline. Lower layers expose outputs via `terraform_remote_state` or a parameter store.

**2. The "stamp" pattern**
A "stamp" = a fully self-contained deployment unit (region, customer, tenant). Stamps are rolled out progressively, allowing canary deploys at the *infrastructure* level.

**3. Service catalog pattern**
Internal teams consume opinionated modules from a private registry. Platform team owns the modules; product teams own the variables.

### Policy as Code

| Tool | Engine | Best for |
|---|---|---|
| Sentinel | HashiCorp (TFC/TFE) | Tight Terraform Cloud integration |
| OPA / Conftest | Rego | Cloud-agnostic, open source, multi-tool |
| Checkov / tfsec | Static analysis | Pre-commit, CI gates |

A typical policy stack:
- **Pre-commit:** `tflint`, `tfsec` — fast, local.
- **CI plan-time:** `conftest` against `terraform show -json tfplan`.
- **Run-time (TFC):** Sentinel policies as final guard.

Example OPA policy (no public S3):
```rego
package terraform

deny[msg] {
  rc := input.resource_changes[_]
  rc.type == "aws_s3_bucket_public_access_block"
  rc.change.after.block_public_acls == false
  msg := sprintf("S3 bucket %v allows public ACLs", [rc.address])
}
```

### Terraform Cloud / Enterprise
What it gives you over open-source Terraform:
- Remote state with UI and audit log.
- Run pipelines (plan → policy check → apply) without you maintaining runners.
- VCS integration, run triggers between workspaces.
- Sentinel and cost estimation.
- Private module registry.

When it's worth it: regulated industries, ≥ 5 platform engineers, audit needs. When it isn't: small teams running well on GitHub Actions + S3.

### Secrets and IAM
- Never put plain secrets in `.tf` or `.tfvars`. Use a secrets manager and read via `data` blocks at apply time.
- Mark sensitive outputs with `sensitive = true` (and remember they still appear in state).
- Use **least privilege** for the Terraform IAM role; separate roles per environment.
- Rotate state-file encryption keys (KMS CMK) on a schedule.

### Performance and scalability
- **Split state.** Plans on huge states (>1k resources) become painful.
- **Parallelism:** `terraform apply -parallelism=20` (default 10). Tune for API rate limits.
- **`-target` is for emergencies**, not workflow. It silently leaves drift behind.
- **Provider configuration aliasing** prevents reinitialization across regions.

### Drift recovery playbook
1. `terraform plan` → identify drift.
2. Decide: who's right — code or reality?
3. If code wins → `apply`.
4. If reality wins → either `terraform import` (bring it under management) or update code to match, then apply.
5. Always file a ticket on the *cause* of drift, not just the symptom.

### Pro Tips
- Maintain a **Terraform readiness checklist** for new components: backend, variables file, README, lint, plan output sanity.
- For huge multi-account orgs, consider **Terragrunt** to keep DRY without losing module clarity.
- Run `terraform graph | dot -Tsvg > graph.svg` once per project — visualizing dependencies catches surprises.

---

## PART B — HANDS-ON LAB: Production Platform with Policy, Modules, and TFC-Style CI

### 1. Objective
Deliver a production-grade pattern:
- Layered structure (`platform/` consumed by `workloads/web/`).
- Reusable internal module: `modules/secure-bucket`.
- Cross-state references via `terraform_remote_state`.
- OPA policy gate on `terraform plan` JSON.
- GitHub Actions pipeline with PR plans, manual approval to apply.

### 2. Architecture
```
+----------------- Repo monorepo ------------------+
|                                                  |
|  modules/                                        |
|    secure-bucket/        (versioned via tags)    |
|                                                  |
|  layers/                                          |
|    platform/    state: s3://.../platform.tfstate |
|    workloads/web/  state: .../workloads-web      |
|                  ^                                |
|                  |  terraform_remote_state        |
|                  |  reads VPC ID, KMS key from    |
|                  |  platform layer                |
|                                                  |
|  policies/                                        |
|    deny_public_buckets.rego                      |
|    require_tags.rego                             |
|                                                  |
|  .github/workflows/                              |
|    plan.yml   (PRs)                              |
|    apply.yml  (main, manual approval)            |
+--------------------------------------------------+
```

### 3. Step-by-Step
1. Define `modules/secure-bucket/`.
2. Implement `layers/platform/` (VPC, KMS, central log bucket).
3. Implement `layers/workloads/web/` consuming the platform outputs.
4. Add OPA policies under `policies/`.
5. Wire two GitHub Actions workflows.

### 4. Files

#### `modules/secure-bucket/main.tf`
**What:** A bucket with encryption, versioning, public-block, and access logs.
**Why:** The "right way" baked in once; teams can't get it wrong.
**When:** Anywhere your org needs S3.

```hcl
variable "name"             { type = string }
variable "kms_key_arn"      { type = string }
variable "log_bucket"       { type = string }
variable "tags"             { type = map(string) default = {} }

resource "aws_s3_bucket" "this" {
  bucket = var.name
  tags   = var.tags
}

resource "aws_s3_bucket_versioning" "this" {
  bucket = aws_s3_bucket.this.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "this" {
  bucket = aws_s3_bucket.this.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = var.kms_key_arn
    }
  }
}

resource "aws_s3_bucket_public_access_block" "this" {
  bucket                  = aws_s3_bucket.this.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_logging" "this" {
  bucket        = aws_s3_bucket.this.id
  target_bucket = var.log_bucket
  target_prefix = "${var.name}/"
}

output "bucket_arn"  { value = aws_s3_bucket.this.arn }
output "bucket_name" { value = aws_s3_bucket.this.bucket }
```
**Key lines:**
- KMS-backed encryption — meets most compliance frameworks out of the box.
- Required `log_bucket` input — you can't *not* have access logging.

#### `layers/platform/main.tf`
```hcl
terraform {
  backend "s3" {
    bucket         = "tf-mastery-state-123456789012"
    key            = "platform/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "tf-mastery-locks"
    encrypt        = true
  }
}

module "vpc" {
  source = "../../modules/vpc"
  # ... see Level 2 ...
}

resource "aws_kms_key" "platform" {
  description             = "Platform-wide CMK"
  enable_key_rotation     = true
  deletion_window_in_days = 30
}

module "log_bucket" {
  source       = "../../modules/secure-bucket"
  name         = "platform-logs-${data.aws_caller_identity.me.account_id}"
  kms_key_arn  = aws_kms_key.platform.arn
  log_bucket   = "platform-self-logs"  # bootstrap bucket
}

output "vpc_id"        { value = module.vpc.vpc_id }
output "kms_key_arn"   { value = aws_kms_key.platform.arn }
output "log_bucket_id" { value = module.log_bucket.bucket_name }
```

#### `layers/workloads/web/main.tf`
```hcl
terraform {
  backend "s3" {
    bucket         = "tf-mastery-state-123456789012"
    key            = "workloads/web/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "tf-mastery-locks"
    encrypt        = true
  }
}

data "terraform_remote_state" "platform" {
  backend = "s3"
  config = {
    bucket = "tf-mastery-state-123456789012"
    key    = "platform/terraform.tfstate"
    region = "us-east-1"
  }
}

module "assets" {
  source      = "../../../modules/secure-bucket"
  name        = "web-assets-prod"
  kms_key_arn = data.terraform_remote_state.platform.outputs.kms_key_arn
  log_bucket  = data.terraform_remote_state.platform.outputs.log_bucket_id

  tags = {
    Owner       = "web-team"
    Environment = "prod"
    CostCenter  = "CC-1042"
  }
}
```
**Key lines:**
- `terraform_remote_state` — consume platform outputs without copy/paste.
- The web team never sees the KMS key or VPC; they get a contract.

#### `policies/deny_public_buckets.rego`
```rego
package terraform.s3

deny[msg] {
  rc := input.resource_changes[_]
  rc.type == "aws_s3_bucket_public_access_block"
  some k
  k := ["block_public_acls","block_public_policy","ignore_public_acls","restrict_public_buckets"][_]
  rc.change.after[k] == false
  msg := sprintf("Resource %s must have %s=true", [rc.address, k])
}
```

#### `policies/require_tags.rego`
```rego
package terraform.tags

required := ["Owner", "Environment", "CostCenter"]

deny[msg] {
  rc := input.resource_changes[_]
  rc.change.actions[_] == "create"
  some t
  t := required[_]
  not rc.change.after.tags[t]
  msg := sprintf("Resource %s missing required tag: %s", [rc.address, t])
}
```

#### `.github/workflows/plan.yml`
```yaml
name: terraform-plan
on: [pull_request]
permissions: { id-token: write, contents: read, pull-requests: write }

jobs:
  plan:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        layer: [layers/platform, layers/workloads/web]
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/gha-tf-readonly
          aws-region: us-east-1
      - uses: hashicorp/setup-terraform@v3
      - name: Init & Plan
        working-directory: ${{ matrix.layer }}
        run: |
          terraform init
          terraform plan -out=tfplan
          terraform show -json tfplan > tfplan.json
      - name: Policy gate
        uses: open-policy-agent/conftest-action@v1
        with:
          files: ${{ matrix.layer }}/tfplan.json
          policy: policies/
```

#### `.github/workflows/apply.yml`
```yaml
name: terraform-apply
on:
  push: { branches: [main] }

permissions: { id-token: write, contents: read }

jobs:
  apply:
    runs-on: ubuntu-latest
    environment: prod   # GitHub environment with required reviewers
    strategy:
      matrix:
        layer: [layers/platform, layers/workloads/web]
      max-parallel: 1
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/gha-tf-apply
          aws-region: us-east-1
      - uses: hashicorp/setup-terraform@v3
      - working-directory: ${{ matrix.layer }}
        run: |
          terraform init
          terraform apply -auto-approve
```
**Key lines:**
- `environment: prod` — gates apply behind manual approval in GitHub.
- Distinct read-only and write IAM roles for plan vs. apply.
- `max-parallel: 1` — platform layer always applies before workload.

### 5. Commands
```bash
# Local validation before pushing
terraform fmt -recursive
terraform -chdir=layers/platform init && terraform -chdir=layers/platform plan
terraform -chdir=layers/workloads/web init && terraform -chdir=layers/workloads/web plan

# Policy check locally
terraform -chdir=layers/platform show -json tfplan > tfplan.json
conftest test tfplan.json -p policies/

# Apply (or let CI do it)
terraform -chdir=layers/platform apply
terraform -chdir=layers/workloads/web apply

# Tear down (reverse order!)
terraform -chdir=layers/workloads/web destroy
terraform -chdir=layers/platform destroy
```

### 6. Expected Output
- Two independent state files, each lockable.
- A web team that never touches the platform layer but consumes its outputs.
- A PR that fails CI when someone forgets the `CostCenter` tag.
- A merge to `main` that pauses at "approve to apply" in GitHub.

### Advanced Thinking Layer
- **Why two layers, two states?** Decoupled lifecycles. Platform changes weekly; workloads change hourly. Combined state = endless contention.
- **Why OPA on plan JSON, not just `tfsec`?** Plan JSON shows *exactly* what will change, including computed values. Static analysis misses interpolated outputs.
- **Trade-off of `terraform_remote_state`:** It creates a runtime read coupling between layers. The cleaner alternative is publishing platform outputs to SSM Parameter Store (or equivalent) and reading those. Use SSM when you want to break the Terraform-to-Terraform link.
- **Anti-pattern:** A monolithic policy file with 200 rules. Group policies by domain (`policies/network/`, `policies/iam/`, `policies/cost/`) and version them.
- **Scaling note:** At 50+ workloads, generate the matrix dynamically from a manifest file. At 500+, move to Terraform Cloud or Spacelift; you've outgrown GitHub Actions as a Terraform runner.

---
---

# FINAL CHECKLIST — "Production-Ready" Means

- [ ] Remote state, encrypted, versioned, locked.
- [ ] No static cloud credentials anywhere — OIDC end-to-end.
- [ ] One state per (env × component); blast radius is bounded.
- [ ] Modules versioned and documented; no "v0" modules in prod.
- [ ] Policy as code on every plan.
- [ ] Drift detection scheduled (e.g., nightly `plan` job that opens an issue on diff).
- [ ] Tags enforced: Owner, Environment, CostCenter, ManagedBy.
- [ ] Disaster recovery: state backups, KMS key rotation, runbook for `terraform import` after manual recovery.
- [ ] Cost visibility: `infracost` posting estimates on PRs.
- [ ] Plan output reviewable by a non-author engineer.

If you can tick all ten, you're not just using Terraform — you're operating it.
