# Terraform Next Steps — Phase 4: Advanced Patterns

> **Goal:** Learn the patterns that separate senior Terraform engineers from juniors — Terragrunt, import, HCP Terraform, and automated testing.
> **Timeline:** Week 8–10
> **Prerequisites:** Phase 3 complete — CI/CD pipeline working with plan on PR and apply on merge

---

## Overview

By the end of Phase 3, you can build real infrastructure and automate it in a pipeline. Phase 4 adds the patterns you'll encounter in production environments and senior-level interviews:

1. **Terragrunt** — eliminate boilerplate across environments
2. **`terraform import`** — bring existing resources under management
3. **HCP Terraform Cloud** — team collaboration and policy enforcement
4. **Terratest** — automated testing for reusable modules

---

## Step 1: Terragrunt for DRY Multi-Environment Setups

Terragrunt is a thin wrapper around Terraform that eliminates the boilerplate of managing many environments. Without Terragrunt, you copy `backend.tf` and `provider.tf` into every environment directory. With Terragrunt, you define them once.

### Install

```bash
# macOS
brew install terragrunt

# Linux — check latest release at github.com/gruntwork-io/terragrunt/releases
curl -sSLo terragrunt \
  https://github.com/gruntwork-io/terragrunt/releases/download/v0.55.0/terragrunt_linux_amd64
chmod +x terragrunt
mv terragrunt /usr/local/bin/
```

### Project Structure WITH Terragrunt

```
infrastructure/
├── terragrunt.hcl                  # Root config — shared by all environments
└── environments/
    ├── dev/
    │   ├── networking/
    │   │   └── terragrunt.hcl      # Just the inputs — no main.tf or backend.tf
    │   ├── compute/
    │   │   └── terragrunt.hcl
    │   └── env.hcl                 # Environment-specific variables
    ├── staging/
    │   ├── networking/
    │   │   └── terragrunt.hcl
    │   └── compute/
    │       └── terragrunt.hcl
    └── prod/
        ├── networking/
        │   └── terragrunt.hcl
        └── compute/
            └── terragrunt.hcl
```

### Root `terragrunt.hcl` — Define Backend and Provider Once

```hcl
# infrastructure/terragrunt.hcl

locals {
  # Read environment-specific variables from the env.hcl in the environment folder
  env_vars = read_terragrunt_config(find_in_parent_folders("env.hcl"))
  env      = local.env_vars.locals.environment
}

# Generate backend.tf for every child module automatically
generate "backend" {
  path      = "backend.tf"
  if_exists = "overwrite_terragrunt"

  contents = <<EOF
terraform {
  backend "s3" {
    bucket         = "my-company-terraform-state"
    key            = "${path_relative_to_include()}/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-locks"
    encrypt        = true
  }
}
EOF
}

# Generate provider.tf for every child module automatically
generate "provider" {
  path      = "provider.tf"
  if_exists = "overwrite_terragrunt"

  contents = <<EOF
provider "aws" {
  region = "us-east-1"

  default_tags {
    tags = {
      Environment = "${local.env}"
      ManagedBy   = "terraform"
    }
  }
}
EOF
}
```

### Environment `env.hcl`

```hcl
# infrastructure/environments/dev/env.hcl
locals {
  environment = "dev"
}
```

### Module `terragrunt.hcl` — Just the Inputs

```hcl
# infrastructure/environments/dev/networking/terragrunt.hcl

# Point to the shared Terraform module
terraform {
  source = "../../../../modules//networking"
}

# Inherit root config (backend, provider)
include "root" {
  path = find_in_parent_folders()
}

# Read environment variables
locals {
  env_vars = read_terragrunt_config(find_in_parent_folders("env.hcl"))
}

# Inputs — the only thing that changes between environments
inputs = {
  environment = local.env_vars.locals.environment
  vpc_cidr    = "10.0.0.0/16"
}
```

```hcl
# infrastructure/environments/dev/compute/terragrunt.hcl

terraform {
  source = "../../../../modules//compute"
}

include "root" {
  path = find_in_parent_folders()
}

# Declare dependency on networking — Terragrunt handles apply order
dependency "networking" {
  config_path = "../networking"
}

locals {
  env_vars = read_terragrunt_config(find_in_parent_folders("env.hcl"))
}

inputs = {
  environment  = local.env_vars.locals.environment
  vpc_id       = dependency.networking.outputs.vpc_id
  subnet_ids   = dependency.networking.outputs.private_subnet_ids
}
```

### Terragrunt Commands

```bash
# Apply a single module
cd environments/dev/networking
terragrunt apply

# Apply ALL modules in the correct dependency order
cd environments/dev
terragrunt run-all apply

# Plan all modules at once
terragrunt run-all plan

# Destroy all modules (reverse dependency order)
terragrunt run-all destroy

# Apply only modules that have changed since last apply
terragrunt run-all apply --terragrunt-include-dir networking
```

> **Pro tip:** Terragrunt's `run-all` respects `dependency` blocks — it applies networking before compute automatically. You define the graph once; Terragrunt handles the order every time.

---

## Step 2: Importing Existing Resources

Real environments always have resources created manually, by other tools, or by other teams. `terraform import` brings them under Terraform management without recreating them.

### Modern Approach: Import Blocks (Terraform 1.5+)

The recommended approach — declarative, can be reviewed in a plan:

```hcl
# In main.tf — add an import block alongside the resource
import {
  to = aws_s3_bucket.legacy_data
  id = "my-existing-bucket-name"
}

resource "aws_s3_bucket" "legacy_data" {
  bucket = "my-existing-bucket-name"
  # Add tags and other settings you want Terraform to manage
  tags = {
    ManagedBy   = "terraform"
    Environment = "production"
  }
}
```

```bash
# Run plan to see what Terraform would change after importing
terraform plan
# Output shows: Plan: 1 to import, 0 to add, X to change, 0 to destroy

# Apply the import — resource added to state, existing infra unchanged
terraform apply
```

### Legacy Approach: CLI Import (Terraform < 1.5)

```bash
# Syntax: terraform import <resource_type>.<resource_name> <resource_id>

# Import an S3 bucket
terraform import aws_s3_bucket.legacy_data my-existing-bucket-name

# Import an EC2 instance
terraform import aws_instance.web i-1234567890abcdef0

# Import a security group
terraform import aws_security_group.web sg-12345678

# Import a VPC
terraform import aws_vpc.main vpc-12345678

# After importing, check what Terraform sees
terraform state show aws_s3_bucket.legacy_data
```

### Generating Configuration from Imported State (Terraform 1.5+)

```bash
# Let Terraform generate the resource config for you
terraform plan -generate-config-out=generated.tf
# Review generated.tf, clean it up, move to main.tf
```

### Import Workflow

```
1. Find the resource ID in the cloud console or CLI
   ↓
2. Write the resource block in your .tf file (or let Terraform generate it)
   ↓
3. Add an import block (Terraform 1.5+) or run terraform import (older)
   ↓
4. Run terraform plan to see what Terraform wants to change
   ↓
5. Adjust your resource config to match actual state (minimize changes)
   ↓
6. Run terraform apply to finalize import
   ↓
7. Remove the import block (it's no longer needed)
```

> **Pro tip:** After importing, always run `terraform plan` to check for drift. Terraform may want to add tags, change settings, or enable features not set manually. Review every change before applying — importing doesn't mean "no changes."

---

## Step 3: HCP Terraform Cloud

HashiCorp's managed Terraform service (formerly Terraform Cloud). The free tier supports up to 500 resources — more than enough for learning and small teams.

### What HCP Terraform Adds

| Feature | Local/S3 | HCP Terraform |
|---|---|---|
| Remote state | Manual S3 setup | Built-in |
| State locking | DynamoDB | Built-in |
| Team access controls | Manual IAM | Role-based (Read, Plan, Apply, Admin) |
| Run history | None | Full history with logs |
| Policy as code | None | Sentinel (paid) / OPA |
| Private module registry | None | Built-in |
| VCS integration | Manual GitHub Actions | Native GitHub/GitLab integration |

### Setup

```bash
# 1. Create a free account at app.terraform.io

# 2. Create an organization
# (done in the web UI)

# 3. Login from CLI
terraform login
# Opens browser for authentication — creates ~/.terraform.d/credentials.tfrc.json
```

### Configure Backend in `main.tf`

```hcl
terraform {
  cloud {
    organization = "my-org-name"

    workspaces {
      name = "production"
      # Or use tags to match multiple workspaces:
      # tags = ["production", "aws"]
    }
  }
}
```

```bash
# Initialize — state is now stored in HCP Terraform
terraform init

# Plans and applies now run in HCP Terraform (remote execution)
terraform plan
terraform apply
```

### Variable Sets — Share Variables Across Workspaces

```hcl
# Instead of setting AWS credentials in every workspace,
# create a Variable Set in HCP Terraform UI:
# Settings → Variable Sets → Create Variable Set
# Add: AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY as environment variables (sensitive)
# Apply to: All workspaces in the organization
```

### Sentinel Policy Example (Paid Feature)

```python
# policies/no-large-instances.sentinel
# Enforce that no EC2 instance larger than t3.large is deployed in dev

import "tfplan/v2" as tfplan

allowed_types = ["t2.micro", "t2.small", "t3.micro", "t3.small", "t3.medium", "t3.large"]

violations = filter tfplan.resource_changes as _, rc {
  rc.type is "aws_instance" and
  rc.change.actions is not ["delete"] and
  rc.change.after.instance_type not in allowed_types
}

main = rule {
  length(violations) is 0
}
```

> **Pro tip:** HCP Terraform's Sentinel engine enforces rules like "no EC2 instances larger than t3.medium in dev" or "all S3 buckets must have encryption enabled." This is how large organizations enforce guardrails across many teams without manual code review for every change.

---

## Step 4: Automated Testing with Terratest

Terratest is a Go library for writing automated tests for Terraform modules. Tests spin up real infrastructure, run assertions, then tear everything down.

### When to Use Terratest

- **Public/shared modules** that many teams depend on — a bug affects everyone
- **Complex modules** with non-obvious behavior (networking, IAM, encryption)
- **Modules with many inputs** — testing common combinations

For app-specific Terraform with one team, manual testing + CI plan review is usually sufficient.

### Setup

```bash
# Prerequisites: Go installed (golang.org/dl)
go version   # Verify Go 1.21+

# Create test directory
mkdir -p test
cd test
go mod init github.com/your-org/your-repo
go get github.com/gruntwork-io/terratest/modules/terraform
go get github.com/stretchr/testify/assert
```

### Write Your First Test

```go
// test/networking_test.go
package test

import (
    "testing"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/gruntwork-io/terratest/modules/aws"
    "github.com/stretchr/testify/assert"
)

func TestNetworkingModule(t *testing.T) {
    t.Parallel()   // Run multiple tests in parallel

    // Configure Terraform options
    opts := &terraform.Options{
        TerraformDir: "../modules/networking",

        Vars: map[string]interface{}{
            "environment":          "test",
            "vpc_cidr":             "10.99.0.0/16",
            "public_subnet_cidrs":  []string{"10.99.1.0/24"},
            "private_subnet_cidrs": []string{"10.99.10.0/24"},
            "availability_zones":   []string{"us-east-1a"},
        },
    }

    // Destroy at end of test — even if test fails
    defer terraform.Destroy(t, opts)

    // Apply the module
    terraform.InitAndApply(t, opts)

    // Assert outputs are not empty
    vpcId := terraform.Output(t, opts, "vpc_id")
    assert.NotEmpty(t, vpcId)

    publicSubnetIds := terraform.OutputList(t, opts, "public_subnet_ids")
    assert.Equal(t, 1, len(publicSubnetIds))

    // Assert real AWS state matches expectations
    vpc := aws.GetVpcById(t, vpcId, "us-east-1")
    assert.Equal(t, "10.99.0.0/16", aws.GetVpcCidr(t, vpc))
    assert.True(t, aws.IsPublicSubnet(t, publicSubnetIds[0], "us-east-1"))
}
```

### Run Tests

```bash
# Run all tests (with verbose output)
go test -v -timeout 30m ./...

# Run a specific test
go test -v -run TestNetworkingModule -timeout 30m ./...

# Run tests in parallel (faster, uses more AWS resources)
go test -v -timeout 30m -parallel 4 ./...
```

### Add Terratest to GitHub Actions

```yaml
# Add to .github/workflows/terraform.yml
  test:
    name: Terratest
    runs-on: ubuntu-latest
    # Only run on PRs that change module code
    if: github.event_name == 'pull_request'

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version: '1.21'

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Run Terratest
        run: go test -v -timeout 30m ./test/...
        working-directory: .
```

> **Pro tip:** Terratest is most valuable for shared/public modules. For application-specific Terraform, manual testing plus CI plan review is usually sufficient. Don't over-engineer — Terratest tests take real time and cost real money to run.

---

## Advanced Pattern: Dynamic Blocks

When a resource has a repeating nested block, use `dynamic` to avoid repetition:

```hcl
# Without dynamic — repetitive
resource "aws_security_group" "web" {
  name = "web-sg"

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# With dynamic — DRY
locals {
  ingress_rules = [
    { port = 80,  description = "HTTP" },
    { port = 443, description = "HTTPS" },
    { port = 8080, description = "Alt HTTP" },
  ]
}

resource "aws_security_group" "web" {
  name = "web-sg"

  dynamic "ingress" {
    for_each = local.ingress_rules
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
      description = ingress.value.description
    }
  }
}
```

---

## Advanced Pattern: `moved` Block for Safe Refactoring

When you rename a resource or move it to a module, Terraform would normally destroy and recreate it. The `moved` block prevents this:

```hcl
# You had:
resource "aws_instance" "server" { ... }

# You want to rename it to:
resource "aws_instance" "web_server" { ... }

# Add a moved block — Terraform updates state without recreating
moved {
  from = aws_instance.server
  to   = aws_instance.web_server
}

# After applying, delete the moved block — it's no longer needed
```

---

## Phase 4 Checklist

- [ ] Terragrunt installed and root `terragrunt.hcl` created
- [ ] At least one environment managed with Terragrunt (no duplicated backend/provider code)
- [ ] `terragrunt run-all plan` runs successfully across all modules
- [ ] At least one existing resource imported with `import` block
- [ ] HCP Terraform account created (free tier)
- [ ] HCP Terraform backend configured in at least one workspace
- [ ] Terratest test written for at least one module
- [ ] `dynamic` block used in at least one resource

---

## What's Next

**Phase 5** covers certifications and job prep: the HashiCorp Terraform Associate (003) exam guide, portfolio checklist, and a walkthrough of the most common take-home interview architecture.

---

*Part of the Terraform Next Steps series — 5 phases from local practice to production-ready skills.*
