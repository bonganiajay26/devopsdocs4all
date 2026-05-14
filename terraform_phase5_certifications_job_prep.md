# Terraform Next Steps — Phase 5: Certifications & Job Prep

> **Goal:** Validate your skills with the industry-recognized HashiCorp certification, build a portfolio that gets interviews, and ace the most common Terraform interview formats.
> **Timeline:** Week 11–12
> **Prerequisites:** Phases 1–4 complete — real cloud infrastructure built, CI/CD running, advanced patterns practiced

---

## The HashiCorp Terraform Associate (003) Certification

### Exam Details

| Detail | Info |
|---|---|
| **Exam code** | TA-003 |
| **Questions** | 57 questions |
| **Duration** | 1 hour |
| **Cost** | ~$70 USD |
| **Format** | Online proctored (from home) or in-person test center |
| **Validity** | 2 years |
| **Passing score** | ~70% (HashiCorp does not publish exact passing threshold) |

Register at: `https://www.hashicorp.com/certifications/terraform-associate`

---

## Exam Topic Breakdown

### Domain 1: Understand Infrastructure as Code (IaC) Concepts (~6%)

```
- Benefits of IaC (consistency, version control, automation, documentation)
- What Terraform does vs what providers do
- Difference between declarative (Terraform) and imperative (scripts) approaches
- Idempotency — why running terraform apply twice has no effect if nothing changed
```

### Domain 2: Understand Terraform's Purpose (~8%)

```
- Multi-cloud and provider-agnostic approach
- Terraform vs other tools: Ansible (config mgmt), CloudFormation (AWS-only), Pulumi (imperative)
- What Terraform can and cannot do
- Terraform Open Source vs Terraform Enterprise vs HCP Terraform
```

### Domain 3: Understand Terraform Basics (~26%)

**This is the largest domain — study it hardest.**

```hcl
# Provider version constraints — know all operators
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"    # Allows 5.x but not 6.x (most common in exams)
      # version = ">= 4.0"  # Any version 4.0 or higher
      # version = "= 5.1.0" # Exactly this version only
      # version = "!= 4.5"  # Any version except 4.5
    }
  }
}

# Variable types — know all of them
variable "example_string" { type = string }
variable "example_number" { type = number }
variable "example_bool"   { type = bool }
variable "example_list"   { type = list(string) }
variable "example_map"    { type = map(string) }
variable "example_set"    { type = set(string) }
variable "example_object" {
  type = object({
    name = string
    port = number
  })
}

# Variable precedence (lowest to highest):
# 1. Default value in variable block
# 2. terraform.tfvars
# 3. *.auto.tfvars files (alphabetical order)
# 4. -var-file flag
# 5. -var flag
# 6. TF_VAR_* environment variables  ← highest priority

# Resource meta-arguments — all tested
resource "aws_instance" "example" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"

  count      = 3                    # Create 3 instances
  # for_each = toset(["a","b","c"]) # OR use for_each (not both)
  depends_on = [aws_vpc.main]       # Explicit dependency
  provider   = aws.us_west          # Use a specific provider alias

  lifecycle {
    create_before_destroy = true    # Zero-downtime replacements
    prevent_destroy       = true    # Block terraform destroy (use carefully)
    ignore_changes        = [tags]  # Don't track tag changes in state
    replace_triggered_by  = [aws_security_group.web.id]  # Force replacement
  }
}
```

### Domain 4: Use Terraform Outside the Core Workflow (~14%)

```bash
# Terraform state commands — all tested
terraform state list                          # List all resources in state
terraform state show aws_instance.web         # Show details of one resource
terraform state mv aws_instance.old aws_instance.new  # Rename in state
terraform state rm aws_instance.orphaned      # Remove from state (not from cloud)
terraform state pull                          # Download remote state as JSON
terraform state push state.tfstate            # Upload state file (dangerous!)

# Terraform workspace commands
terraform workspace list                      # List all workspaces
terraform workspace new staging               # Create new workspace
terraform workspace select production         # Switch to workspace
terraform workspace show                      # Show current workspace
terraform workspace delete staging            # Delete workspace (must be empty)

# Other commands tested
terraform console                             # Interactive expression evaluator
terraform graph                               # Output dependency graph (DOT format)
terraform force-unlock LOCK_ID                # Release stuck state lock
terraform get                                 # Download modules without full init
```

### Domain 5: Interact with Terraform Modules (~21%)

```hcl
# Module sources — all tested
module "local" {
  source = "./modules/networking"       # Local path
}

module "registry" {
  source  = "hashicorp/consul/aws"      # Terraform Registry
  version = "0.1.0"                     # Required for registry modules
}

module "github" {
  source = "github.com/org/repo//modules/networking"
}

module "s3" {
  source = "s3::https://s3.amazonaws.com/my-bucket/module.zip"
}

# Module outputs — accessing child module outputs
output "vpc_id" {
  value = module.networking.vpc_id      # module.<name>.<output_name>
}

# Passing module outputs to another module
module "compute" {
  source    = "./modules/compute"
  vpc_id    = module.networking.vpc_id  # Cross-module reference via output
}
```

### Domain 6: Navigate Terraform Workflow (~25%)

```bash
# Core workflow — every step and its purpose
terraform init      # Download providers + modules, set up backend
terraform validate  # Syntax check (no cloud calls)
terraform fmt       # Format code to HCL standard style
terraform plan      # Preview changes (reads state, calls cloud APIs read-only)
terraform apply     # Apply changes (modifies real infrastructure)
terraform destroy   # Destroy all managed resources

# Plan flags
terraform plan -out=tfplan              # Save plan to file
terraform plan -target=aws_instance.web # Plan only one resource
terraform plan -var="env=prod"          # Pass variable inline
terraform plan -refresh=false           # Skip state refresh (faster, less accurate)

# Apply flags
terraform apply tfplan                  # Apply from saved plan (no prompt)
terraform apply -auto-approve           # Skip confirmation prompt (CI use only)
terraform apply -replace=aws_instance.web  # Force resource replacement (replaces -taint)

# Destroy flags
terraform destroy -target=aws_instance.web  # Destroy one resource only
```

### Domain 7: Implement and Maintain State (~26%)

**The second largest domain — remote state and locking are heavily tested.**

```bash
# State storage backends — know the difference
# Local:  terraform.tfstate on disk — default, not for teams
# S3:     AWS S3 + DynamoDB for locking — most common in AWS shops
# Azure:  Azure Blob Storage — for Azure shops
# GCS:    Google Cloud Storage — for GCP shops
# HCP:    Terraform Cloud/Enterprise — cross-cloud, managed

# Sensitive state — state files contain secrets in plaintext
# Always encrypt your remote backend:
# S3: encrypt = true
# Azure: automatically encrypted at rest
# GCP: automatically encrypted at rest

# terraform refresh — syncs state with real infrastructure
# Deprecated in 1.5+ — prefer:
terraform apply -refresh-only    # See what drift exists
terraform apply -refresh-only -auto-approve  # Accept drift into state

# Partial backends — configure at init time, not in code
terraform init \
  -backend-config="bucket=my-state-bucket" \
  -backend-config="key=prod/terraform.tfstate"
```

---

## Built-in Functions You Must Know

```hcl
# String functions
upper("hello")           # "HELLO"
lower("HELLO")           # "hello"
format("Hello, %s!", "world")  # "Hello, world!"
replace("hello world", "world", "terraform")  # "hello terraform"
split(",", "a,b,c")     # ["a", "b", "c"]
join(",", ["a","b","c"]) # "a,b,c"
trimspace("  hello  ")   # "hello"

# Collection functions
length(["a","b","c"])    # 3
toset(["a","a","b"])     # {"a", "b"}  — removes duplicates
tolist(toset(["b","a"])) # ["a", "b"]  — converts set to sorted list
tomap({"a" = 1})         # {"a" = 1}
lookup({"a" = 1}, "a", 0) # 1  (with default 0 if key missing)
merge({"a" = 1}, {"b" = 2}) # {"a" = 1, "b" = 2}
flatten([[1,2],[3,4]])   # [1, 2, 3, 4]
concat(["a","b"],["c"])  # ["a", "b", "c"]
contains(["a","b"], "a") # true
keys({"a" = 1, "b" = 2}) # ["a", "b"]
values({"a" = 1, "b" = 2}) # [1, 2]

# Numeric functions
max(1, 5, 3)             # 5
min(1, 5, 3)             # 1
ceil(1.2)                # 2
floor(1.8)               # 1

# Type conversion
tostring(42)             # "42"
tonumber("42")           # 42
tobool("true")           # true

# Encoding
jsonencode({"key" = "value"}) # "{\"key\":\"value\"}"
jsondecode("{\"key\": \"value\"}") # {"key" = "value"}
base64encode("hello")    # "aGVsbG8="
base64decode("aGVsbG8=") # "hello"

# Filesystem (in terraform configs)
file("./scripts/user_data.sh")   # Read file content as string
templatefile("./tmpl.tpl", {name = "web"})  # Render template file

# Date and time
timestamp()              # Current UTC time: "2024-01-15T10:30:00Z"
formatdate("YYYY-MM-DD", timestamp())  # "2024-01-15"
```

---

## Study Plan — 2 Weeks

### Week 1: Theory + Practice

| Day | Focus | Activity |
|---|---|---|
| Day 1 | Domains 1–2 | Read HashiCorp docs on IaC concepts |
| Day 2 | Domain 3 (basics) | Practice variable types and meta-arguments |
| Day 3 | Domain 4 (state) | Run all `terraform state` commands on a real project |
| Day 4 | Domain 5 (modules) | Build a module, publish it locally, call with different sources |
| Day 5 | Domain 6 (workflow) | Run full workflow with every flag combination |
| Day 6 | Domain 7 (backend) | Set up S3 + DynamoDB, migrate state, test locking |
| Day 7 | Functions | Run `terraform console`, test every function above |

### Week 2: Practice Exams + Weak Areas

| Day | Focus | Activity |
|---|---|---|
| Day 1 | Practice exam 1 | ExamPro or Udemy practice test |
| Day 2 | Weak areas from exam 1 | Deep dive any score below 70% |
| Day 3 | Practice exam 2 | Different practice test provider |
| Day 4 | Weak areas from exam 2 | Re-read HashiCorp docs on failing topics |
| Day 5 | Hands-on review | Build from memory without referencing docs |
| Day 6 | Final practice exam | Target 80%+ before booking the real exam |
| Day 7 | Rest + light review | Read through your notes, no heavy studying |

---

## Portfolio Project Checklist

Before applying to jobs, your public GitHub repo should demonstrate:

### Must-Have Signals

- [ ] **Multi-environment setup** — `environments/dev/`, `environments/staging/`, `environments/prod/` directories (or Terragrunt equivalent)
- [ ] **At least 2 reusable modules** — each with `variables.tf`, `outputs.tf`, and `README.md` generated by `terraform-docs`
- [ ] **Remote state** — S3 + DynamoDB or HCP Terraform (not local state)
- [ ] **GitHub Actions CI/CD** — `plan` on PR, `apply` on merge, credentials in secrets not code
- [ ] **Proper `.gitignore`** — `*.tfstate`, `.terraform/`, `*.tfvars` excluded; `.terraform.lock.hcl` committed
- [ ] **Meaningful resource tagging** — `Environment`, `Owner`, `ManagedBy = "terraform"` on all resources
- [ ] **Architecture README** — explains what the code builds, includes a diagram if possible
- [ ] **`terraform destroy` works cleanly** — no orphaned resources, no errors

### Strong Differentiators

- [ ] **Security scanning in CI** — `checkov` or `tfsec` in the GitHub Actions pipeline
- [ ] **Terragrunt** for DRY environment management
- [ ] **`terraform import`** demonstrated on a real resource
- [ ] **Terratest** test for at least one module
- [ ] **`moved` block** used for a refactoring commit
- [ ] **Sentinel or OPA policy** (even a simple one)
- [ ] **`for_each` over `count`** for resource collections (shows understanding)

> **Pro tip:** During interviews, be ready to walk through your repo live and explain every decision. Why S3 over local state? Why modules? Why `for_each` instead of `count`? The "why" matters more than the "what."

---

## The Classic Take-Home Interview Exercise

Many companies give a 48-72 hour take-home: *"Create Terraform code that deploys a web application with a load balancer, auto-scaling group, and RDS database in a VPC across 2 availability zones."*

### Architecture to Build

```
VPC (10.0.0.0/16)
│
├── Public Subnets — AZ-a (10.0.1.0/24), AZ-b (10.0.2.0/24)
│   ├── Application Load Balancer
│   └── NAT Gateways (one per AZ for HA)
│
├── Private Subnets — AZ-a (10.0.10.0/24), AZ-b (10.0.11.0/24)
│   └── Auto Scaling Group
│       ├── Launch Template (EC2 t2.micro, Amazon Linux 2)
│       ├── Min: 1, Max: 4, Desired: 2
│       └── Target Tracking Scaling (CPU 70%)
│
└── Database Subnets — AZ-a (10.0.20.0/24), AZ-b (10.0.21.0/24)
    └── RDS PostgreSQL
        ├── Multi-AZ: true
        ├── Instance: db.t3.micro
        └── DB Subnet Group spanning both AZs
```

### What Interviewers Evaluate

| Criterion | What They're Looking For |
|---|---|
| **Module structure** | Clean separation: networking / compute / database |
| **Variable usage** | No hardcoded values — everything parameterized |
| **Security** | Private subnets for EC2 and RDS, security groups with minimal access |
| **DRY code** | `for_each` for subnets across AZs, not repeated resource blocks |
| **Outputs** | ALB DNS name output so you can actually visit the app |
| **State** | Remote state configured (S3 or HCP Terraform) |
| **Tags** | Consistent tagging on all resources |
| **Destroy** | `terraform destroy` works cleanly, no errors |
| **README** | Architecture explained, with commands to deploy |

### Commit Strategy

Interviewers read commit history to understand your thinking process:

```bash
git commit -m "Add VPC and subnet module with multi-AZ support"
git commit -m "Add security groups for ALB, EC2, and RDS layers"
git commit -m "Add ALB with HTTP listener and target group"
git commit -m "Add launch template and auto scaling group"
git commit -m "Add RDS PostgreSQL with subnet group and multi-AZ"
git commit -m "Wire ALB target group to auto scaling group"
git commit -m "Add outputs: ALB DNS name, VPC ID, subnet IDs"
git commit -m "Add remote state backend and variable files"
git commit -m "Add README with architecture diagram and deploy instructions"
```

> **Pro tip:** Commit incrementally with meaningful messages. Never a single "Initial commit" with all the code. The history shows you build layer by layer — that's what a senior engineer does.

---

## Interview Questions to Practice Answering

These are the five questions from the original guide — worth reviewing one more time with fresh knowledge:

1. **State vs plan** — What's the difference? Why does remote state with locking matter in teams?
2. **count vs for_each** — When would you use one over the other? What happens when you remove the middle item from a count list?
3. **Secrets in Terraform** — How do you avoid hardcoding sensitive values? Name three approaches.
4. **Good module design** — What makes a module reusable? What are the signs of a poorly designed module?
5. **Multi-environment structure** — How would you structure a Terraform project for dev/staging/prod?

**New questions at the senior level:**

6. **State corruption** — How would you recover from a corrupted or missing state file?
7. **Drift** — What is configuration drift and how does Terraform detect and handle it?
8. **Import** — Walk me through importing an existing resource that was created manually.
9. **Refactoring** — How do you rename a resource in Terraform without destroying and recreating it?
10. **Scale** — What are the challenges of managing 500+ resources in a single Terraform workspace, and how would you address them?

---

## Resources

### Official

- HashiCorp Learn: `developer.hashicorp.com/terraform/tutorials`
- Terraform Documentation: `developer.hashicorp.com/terraform/docs`
- Certification study guide: `developer.hashicorp.com/terraform/tutorials/certification-003`

### Practice Exams

- **ExamPro** (Andrew Brown) — most closely mirrors real exam format
- **Udemy** (Zeal Vora or Bryan Krausen) — includes hands-on labs

### Community

- HashiCorp Discuss: `discuss.hashicorp.com`
- Reddit: `r/Terraform`
- AWS/GCP/Azure Terraform modules: `registry.terraform.io`

---

## Congratulations — You've Completed the Series

| Phase | Focus | Outcome |
|---|---|---|
| Phase 1 | Cloud Provider Setup | Real terraform apply on AWS/GCP/Azure |
| Phase 2 | Real-World Project | 3-tier architecture with remote state |
| Phase 3 | CI/CD | Automated plan on PR, apply on merge |
| Phase 4 | Advanced Patterns | Terragrunt, import, HCP Terraform, testing |
| Phase 5 | Certifications & Jobs | Certified, portfolio-ready, interview-prepared |

The Terraform market is active and well-paying. Engineers who can demonstrate real infrastructure code, explain their decisions clearly, and operate confidently in CI/CD pipelines are consistently in demand across cloud-native teams.

---

*Part of the Terraform Next Steps series — 5 phases from local practice to production-ready skills.*
