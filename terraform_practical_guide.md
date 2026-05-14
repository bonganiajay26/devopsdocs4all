# Terraform Practical Guide: From Theory to Hands-On Practice

> **Audience:** Learners who understand Terraform concepts and are ready to write and run real code locally.
> **Prerequisites:** Terraform CLI installed (`terraform -v` to verify), a code editor (VS Code recommended), basic command-line familiarity.

---

# SECTION 1: Step-by-Step Examples

---

## Example A — Terraform WITHOUT Modules (Flat Structure)

### What You'll Build
A local simulation of infrastructure using the **`hashicorp/local`** provider — no cloud account needed. You'll create local files that represent "servers" and "configs."

---

### Step 1: Create Your Project Folder

```bash
mkdir terraform-no-modules
cd terraform-no-modules
```

> **Why:** Terraform tracks state per directory. A dedicated folder keeps your project isolated.

---

### Step 2: Create the Main Configuration File

Create a file named `main.tf`:

```hcl
terraform {
  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "~> 2.4"
    }
  }
}

provider "local" {}

# Simulates a "web server config" file
resource "local_file" "web_server" {
  filename = "${path.module}/output/web_server.conf"
  content  = <<-EOT
    server_name = "web-01"
    port        = 8080
    environment = "production"
  EOT
}

# Simulates a "database config" file
resource "local_file" "database" {
  filename = "${path.module}/output/database.conf"
  content  = <<-EOT
    db_name  = "app_db"
    db_port  = 5432
    host     = "localhost"
  EOT
}
```

> **Why:** The `local` provider is perfect for learning — it creates real files on your machine without any cloud credentials. `path.module` refers to the current directory.

---

### Step 3: Create the Output Directory

```bash
mkdir output
```

> **Why:** Terraform won't create missing parent directories by default. Pre-creating `output/` prevents errors.

---

### Step 4: Initialize Terraform

```bash
terraform init
```

**What happens:**
- Terraform reads `main.tf` and identifies the `hashicorp/local` provider.
- It downloads the provider plugin into a `.terraform/` folder.
- A `.terraform.lock.hcl` file is created to lock the provider version.

**Expected output:**
```
Initializing provider plugins...
- Installing hashicorp/local v2.4.x...
Terraform has been successfully initialized!
```

---

### Step 5: Preview the Plan

```bash
terraform plan
```

**What happens:**
- Terraform compares your configuration against the current state (empty at first).
- It shows exactly what will be **created**, **changed**, or **destroyed** — no changes are made yet.

**Expected output:**
```
Plan: 2 to add, 0 to change, 0 to destroy.
```

> **Pro Tip:** Always run `plan` before `apply`. Treat it like a "dry run" — your safety net.

---

### Step 6: Apply the Configuration

```bash
terraform apply
```

- Terraform shows the plan again and asks for confirmation.
- Type `yes` and press Enter.

```bash
Do you want to perform these actions? Enter a value: yes
```

**What happens:**
- The two `.conf` files are created inside your `output/` folder.
- A `terraform.tfstate` file is created — this is Terraform's memory of what it built.

**Verify:**
```bash
cat output/web_server.conf
cat output/database.conf
```

---

### Step 7: Modify and Re-Apply

Edit `main.tf` — change `port = 8080` to `port = 9090` in the `web_server` resource. Then:

```bash
terraform plan   # See only the web_server resource shows as changed
terraform apply  # Apply the change
```

> **Key Learning:** Terraform is **idempotent** — it only changes what's different. The database file is untouched.

---

### Step 8: Destroy Everything

```bash
terraform destroy
```

Type `yes` to confirm. Terraform removes all resources it created and clears the state.

> **Why this matters:** In real environments, `destroy` is used to tear down staging environments or clean up after testing.

---

### Flat Structure Summary

```
terraform-no-modules/
├── main.tf
├── terraform.tfstate
├── .terraform/
├── .terraform.lock.hcl
└── output/
    ├── web_server.conf
    └── database.conf
```

**Limitation of this approach:** As infrastructure grows, `main.tf` becomes hundreds of lines — hard to reuse, test, or share across teams. This is where **modules** solve the problem.

---

---

## Example B — Terraform WITH Modules (Reusable Structure)

### What You'll Build
The same infrastructure as above, but organized so the "server config" logic lives in a **reusable module**. You'll call it twice — once for `web` and once for `database` — with different inputs.

---

### Step 1: Set Up the Project Structure

```bash
mkdir terraform-with-modules
cd terraform-with-modules
mkdir -p modules/server_config
mkdir output
```

> **Why:** By convention, modules live in a `modules/` subfolder. Each module is a self-contained directory with its own `.tf` files.

---

### Step 2: Create the Module Files

#### `modules/server_config/variables.tf`

```hcl
variable "server_name" {
  description = "Name of the server"
  type        = string
}

variable "port" {
  description = "Port number for the server"
  type        = number
}

variable "environment" {
  description = "Deployment environment (e.g., dev, staging, production)"
  type        = string
  default     = "dev"
}

variable "output_dir" {
  description = "Directory to write the config file"
  type        = string
}
```

> **Why:** Variables are the "inputs" to your module — this is what makes modules reusable. The caller decides the values.

---

#### `modules/server_config/main.tf`

```hcl
terraform {
  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "~> 2.4"
    }
  }
}

resource "local_file" "config" {
  filename = "${var.output_dir}/${var.server_name}.conf"
  content  = <<-EOT
    server_name = "${var.server_name}"
    port        = ${var.port}
    environment = "${var.environment}"
  EOT
}
```

> **Why:** The module uses `var.*` to reference inputs. It doesn't hardcode anything — the calling root module provides all values.

---

#### `modules/server_config/outputs.tf`

```hcl
output "config_file_path" {
  description = "Path to the generated config file"
  value       = local_file.config.filename
}
```

> **Why:** Outputs let the calling module access values from inside the module — like a function's return value.

---

### Step 3: Create the Root Configuration

#### `main.tf` (at root level)

```hcl
terraform {
  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "~> 2.4"
    }
  }
}

provider "local" {}

# First call to the module — Web Server
module "web_server" {
  source      = "./modules/server_config"
  server_name = "web-01"
  port        = 8080
  environment = "production"
  output_dir  = "${path.module}/output"
}

# Second call to the module — Database Server
module "database_server" {
  source      = "./modules/server_config"
  server_name = "db-01"
  port        = 5432
  environment = "production"
  output_dir  = "${path.module}/output"
}
```

> **Key Insight:** The same module is called twice with different inputs. You wrote the logic once and reused it — this is the core value of modules.

---

#### `outputs.tf` (at root level)

```hcl
output "web_config_path" {
  value = module.web_server.config_file_path
}

output "db_config_path" {
  value = module.database_server.config_file_path
}
```

---

### Step 4: Initialize

```bash
terraform init
```

Terraform initializes both the root module and the child module. You'll see it download the provider once and recognize the module source.

---

### Step 5: Plan and Apply

```bash
terraform plan
terraform apply
```

Type `yes` when prompted.

**Verify output files:**
```bash
cat output/web-01.conf
cat output/db-01.conf
```

**View outputs in terminal:**
```bash
terraform output
```

Expected:
```
db_config_path  = "./output/db-01.conf"
web_config_path = "./output/web-01.conf"
```

---

### Step 6: Add a Third Server (Power of Reuse)

In `main.tf`, add:

```hcl
module "cache_server" {
  source      = "./modules/server_config"
  server_name = "cache-01"
  port        = 6379
  environment = "staging"
  output_dir  = "${path.module}/output"
}
```

Then:

```bash
terraform plan   # Shows only 1 new resource to add
terraform apply  # Only creates cache-01.conf
```

> **This is the power of modules** — adding infrastructure is as simple as calling the module with new inputs. Zero code duplication.

---

### Module Structure Summary

```
terraform-with-modules/
├── main.tf
├── outputs.tf
├── terraform.tfstate
├── .terraform/
├── .terraform.lock.hcl
├── modules/
│   └── server_config/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
└── output/
    ├── web-01.conf
    ├── db-01.conf
    └── cache-01.conf
```

---

---

# SECTION 2: Advanced Tips & Alternative Approaches

---

## Tip 1: Use `terraform.tfvars` for Input Management

Instead of hardcoding values in `main.tf`, create a `terraform.tfvars` file:

```hcl
# terraform.tfvars
environment = "production"
```

And reference it in `variables.tf` at the root level:

```hcl
variable "environment" {
  type    = string
  default = "dev"
}
```

Terraform automatically loads `terraform.tfvars`. For environment-specific configs:

```bash
terraform apply -var-file="prod.tfvars"
terraform apply -var-file="staging.tfvars"
```

> **Real-world use:** Different `.tfvars` files for `dev`, `staging`, and `prod` environments — same codebase, different values.

---

## Tip 2: Use `for_each` to Avoid Repetitive Module Calls

Instead of repeating `module "web_server"`, `module "db_server"` etc., use `for_each`:

```hcl
locals {
  servers = {
    web   = { port = 8080, env = "production" }
    db    = { port = 5432, env = "production" }
    cache = { port = 6379, env = "staging" }
  }
}

module "servers" {
  for_each    = local.servers
  source      = "./modules/server_config"
  server_name = each.key
  port        = each.value.port
  environment = each.value.env
  output_dir  = "${path.module}/output"
}
```

> **Why it matters:** Adding a new server only requires adding one entry to the `locals` map — no new `module` block needed.

---

## Tip 3: Remote State Management (Team Collaboration)

By default, state is stored locally in `terraform.tfstate`. For teams, use a remote backend:

```hcl
# Example: AWS S3 backend (replace with your bucket)
terraform {
  backend "s3" {
    bucket = "my-terraform-state-bucket"
    key    = "prod/terraform.tfstate"
    region = "us-east-1"
  }
}
```

**Alternative backends:** Azure Blob Storage, Google Cloud Storage, HashiCorp Terraform Cloud (free tier available).

> **Critical rule:** Never commit `terraform.tfstate` to Git. Add it to `.gitignore`. Use remote state with state locking to prevent concurrent modifications.

---

## Tip 4: Use `terraform validate` and `terraform fmt`

Before every `plan`, run:

```bash
terraform fmt      # Auto-formats all .tf files to HCL standard style
terraform validate # Checks for syntax and logical errors without connecting to any provider
```

> **Team habit:** Add these as pre-commit hooks using tools like `pre-commit` + `terraform-docs` to enforce code quality automatically.

---

## Tip 5: Understand the Dependency Graph

Terraform builds a dependency graph internally. Visualize it:

```bash
terraform graph | dot -Tsvg > graph.svg
```

(Requires `graphviz` installed: `brew install graphviz` or `apt install graphviz`)

> **Why this matters:** Understanding implicit vs. explicit dependencies (`depends_on`) helps you debug apply ordering issues in complex infrastructures.

---

## Tip 6: Workspaces for Environment Isolation

```bash
terraform workspace new staging
terraform workspace new production
terraform workspace list
terraform workspace select staging
```

Each workspace maintains its own state file. Combined with `.tfvars`, this is a lightweight multi-environment strategy.

> **Alternative:** Many teams prefer separate directories or separate Terraform Cloud workspaces per environment for stronger isolation and access control.

---

## Tip 7: Terraform Module Registry

Don't rebuild from scratch — use verified community modules:

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.0"
  # ... inputs
}
```

Browse at: [registry.terraform.io](https://registry.terraform.io)

> **Interview tip:** Knowing when to use public modules vs. writing custom ones is a sign of Terraform maturity. Public modules for standard patterns; custom modules for organization-specific logic.

---

---

# SECTION 3: Interview Questions — Current Market Expectations (2024–2025)

---

## Q1: What is the difference between Terraform state and Terraform plan, and why does state matter in team environments?

**Strong Answer:**

Terraform **plan** is a preview — it compares your `.tf` configuration against the **state file** and shows what changes will be made. The **state file** (`terraform.tfstate`) is Terraform's single source of truth about what infrastructure currently exists.

In team environments, state must be stored remotely (S3, Azure Blob, Terraform Cloud) with **state locking** enabled (via DynamoDB for AWS) to prevent two engineers from running `apply` simultaneously — which could corrupt the state or create duplicate resources.

**Follow-up they may ask:** *"What happens if state gets out of sync with real infrastructure?"*
Answer: Use `terraform refresh` (deprecated in newer versions — prefer `terraform apply -refresh-only`) or `terraform import` to bring existing resources under Terraform management.

---

## Q2: Explain the difference between `count` and `for_each`. When would you use one over the other?

**Strong Answer:**

Both create multiple instances of a resource or module, but they behave differently:

| | `count` | `for_each` |
|---|---|---|
| Input type | Integer | Map or Set of strings |
| Resource ID | `resource[0]`, `resource[1]` | `resource["key"]` |
| Deletion behavior | Removing middle item re-creates subsequent resources | Removing one key only affects that resource |

**Use `count`** when resources are identical and order doesn't matter (e.g., 3 identical worker nodes).

**Use `for_each`** when resources have unique identities (e.g., multiple servers with different names and ports). Removing one entry from a `for_each` map only destroys that specific resource — `count` would re-index and potentially destroy/recreate unintended resources.

---

## Q3: How do you manage secrets in Terraform? What approaches do you use to avoid hardcoding sensitive values?

**Strong Answer:**

Never hardcode secrets in `.tf` files or commit them to version control. Recommended approaches:

1. **Environment variables:** `export TF_VAR_db_password="secret"` — Terraform automatically picks up `TF_VAR_*` variables.
2. **`.tfvars` files** marked in `.gitignore` — passed at runtime with `-var-file`.
3. **HashiCorp Vault provider** — dynamically retrieve secrets at apply time.
4. **AWS Secrets Manager / Azure Key Vault** — use the respective Terraform data source to fetch secrets.
5. **Terraform Cloud / Enterprise** — supports encrypted variable storage with team access controls.

> Mark sensitive variables with `sensitive = true` in Terraform to prevent them from appearing in logs and plan output:
```hcl
variable "db_password" {
  type      = string
  sensitive = true
}
```

---

## Q4: What is a Terraform module, and what makes a "good" module design?

**Strong Answer:**

A module is a container for multiple related resources treated as a single unit — essentially a reusable function for infrastructure. Every Terraform configuration is technically a module (the root module).

**Good module design principles:**

- **Single responsibility:** A module does one thing well (e.g., "VPC setup" not "VPC + EKS + RDS").
- **Well-defined interface:** Clear `variables.tf` with descriptions, types, and sensible defaults. Clear `outputs.tf` exposing only what callers need.
- **Version pinning:** When publishing to a registry, use semantic versioning so callers can pin versions.
- **No hardcoded values:** All environment-specific values come in as variables.
- **Documentation:** `README.md` with usage examples and variable descriptions (use `terraform-docs` to auto-generate).
- **Avoid deeply nested modules:** More than 2–3 levels of nesting makes debugging difficult.

---

## Q5: Walk me through how you would structure a Terraform project for a production environment that has three stages: dev, staging, and production.

**Strong Answer:**

There are two common patterns:

**Pattern 1 — Workspaces (lighter weight):**
```
project/
├── main.tf
├── variables.tf
├── dev.tfvars
├── staging.tfvars
└── prod.tfvars
```
Use `terraform workspace select prod && terraform apply -var-file="prod.tfvars"`.

*Drawback:* All environments share the same code version. A bug in `main.tf` affects all workspaces.

**Pattern 2 — Directory-per-environment (more isolation, industry preferred):**
```
project/
├── modules/
│   ├── networking/
│   ├── compute/
│   └── database/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   └── terraform.tfvars
│   ├── staging/
│   │   ├── main.tf
│   │   └── terraform.tfvars
│   └── prod/
│       ├── main.tf
│       └── terraform.tfvars
```

Each environment directory calls the shared modules with environment-specific variables. State is stored separately per environment. This is the approach recommended by HashiCorp for production systems.

**Bonus mention:** Tools like **Terragrunt** automate the boilerplate of this pattern and add DRY (Don't Repeat Yourself) capabilities across environments.

---

*End of Guide*

---

> **Next Steps:** Practice these examples locally, then re-implement Example B targeting a real cloud provider (AWS, GCP, or Azure) by swapping the `local` provider for cloud resources. The module structure and commands remain identical — only the resource types change.
