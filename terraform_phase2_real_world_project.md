# Terraform Next Steps — Phase 2: Real-World Project

> **Goal:** Build a complete 3-tier web architecture using proper modules, data sources, and remote state — exactly what take-home interviews ask for.
> **Timeline:** Week 3–5
> **Prerequisites:** Phase 1 complete — cloud credentials configured, `terraform apply` working on real resources

---

## Architecture Overview

You'll build a standard web application stack across three layers:

```
VPC (10.0.0.0/16)
├── Public Subnets  (10.0.1.0/24, 10.0.2.0/24)  — AZ-a, AZ-b
│   └── Application Load Balancer
├── Private Subnets (10.0.10.0/24, 10.0.11.0/24) — AZ-a, AZ-b
│   └── EC2 Instances (Auto Scaling Group)
└── Database Subnets (10.0.20.0/24, 10.0.21.0/24)
    └── RDS Multi-AZ (PostgreSQL)
```

**Why this architecture?** It's the standard 3-tier pattern used in nearly every web application. It also happens to be the most common take-home Terraform interview exercise.

---

## Project Structure

```
project/
├── main.tf               # Root module — calls child modules
├── variables.tf          # Root-level variables
├── outputs.tf            # Root-level outputs
├── terraform.tfvars      # Variable values (not committed to git)
├── .gitignore
└── modules/
    ├── networking/       # VPC, subnets, IGW, route tables, NACLs
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    ├── compute/          # EC2, security groups, key pairs, ASG
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    └── storage/          # S3 buckets, bucket policies
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

> **Pro tip:** Build bottom-up: networking first (no dependencies), then compute (needs VPC/subnet IDs as inputs), then storage (may need compute ARNs). Each layer's outputs become the next layer's inputs.

---

## Step 1: Build the Networking Module

### `modules/networking/variables.tf`

```hcl
variable "environment" {
  description = "Deployment environment (dev, staging, prod)"
  type        = string
}

variable "vpc_cidr" {
  description = "CIDR block for the VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "public_subnet_cidrs" {
  description = "CIDR blocks for public subnets (one per AZ)"
  type        = list(string)
  default     = ["10.0.1.0/24", "10.0.2.0/24"]
}

variable "private_subnet_cidrs" {
  description = "CIDR blocks for private subnets (one per AZ)"
  type        = list(string)
  default     = ["10.0.10.0/24", "10.0.11.0/24"]
}

variable "availability_zones" {
  description = "Availability zones to deploy into"
  type        = list(string)
  default     = ["us-east-1a", "us-east-1b"]
}
```

### `modules/networking/main.tf`

```hcl
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name        = "${var.environment}-vpc"
    Environment = var.environment
  }
}

# Public subnets — one per AZ using for_each
resource "aws_subnet" "public" {
  count             = length(var.public_subnet_cidrs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.public_subnet_cidrs[count.index]
  availability_zone = var.availability_zones[count.index]

  map_public_ip_on_launch = true

  tags = {
    Name        = "${var.environment}-public-subnet-${count.index + 1}"
    Environment = var.environment
    Tier        = "public"
  }
}

# Private subnets
resource "aws_subnet" "private" {
  count             = length(var.private_subnet_cidrs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_subnet_cidrs[count.index]
  availability_zone = var.availability_zones[count.index]

  tags = {
    Name        = "${var.environment}-private-subnet-${count.index + 1}"
    Environment = var.environment
    Tier        = "private"
  }
}

# Internet Gateway — allows public subnets to reach the internet
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name        = "${var.environment}-igw"
    Environment = var.environment
  }
}

# Route table for public subnets — route 0.0.0.0/0 through IGW
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name        = "${var.environment}-public-rt"
    Environment = var.environment
  }
}

# Associate public subnets with the public route table
resource "aws_route_table_association" "public" {
  count          = length(aws_subnet.public)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}
```

### `modules/networking/outputs.tf`

```hcl
output "vpc_id" {
  description = "ID of the VPC"
  value       = aws_vpc.main.id
}

output "public_subnet_ids" {
  description = "IDs of the public subnets"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "IDs of the private subnets"
  value       = aws_subnet.private[*].id
}
```

---

## Step 2: Use Data Sources for Existing Resources

Data sources let you read existing infrastructure — things **not** managed by your Terraform code. This is critical for working in teams where some infrastructure already exists.

```hcl
# Look up the latest Amazon Linux 2 AMI — changes regularly, never hardcode AMI IDs
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

# Use it in a resource — always up to date
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t2.micro"   # Free tier eligible
  subnet_id     = var.subnet_id

  tags = {
    Name        = "${var.environment}-web-server"
    Environment = var.environment
  }
}
```

### Other Useful Data Sources

```hcl
# Look up your current AWS account ID
data "aws_caller_identity" "current" {}
output "account_id" { value = data.aws_caller_identity.current.account_id }

# Look up the current AWS region
data "aws_region" "current" {}
output "region" { value = data.aws_region.current.name }

# Look up an existing VPC by tag (when you don't manage it with Terraform)
data "aws_vpc" "existing" {
  tags = { Name = "production-vpc" }
}
```

> **Pro tip:** Data sources are read-only — they never create, modify, or destroy resources. Use them to bridge Terraform-managed and externally-managed infrastructure.

---

## Step 3: Move State to Remote S3 Backend

Move your `terraform.tfstate` from local disk to S3 + DynamoDB for locking. **Non-negotiable for team environments.** Required knowledge in most Terraform job interviews.

### The Bootstrap Problem

You can't use Terraform to create the S3 bucket that stores Terraform state. Create it manually once:

```bash
# Step 1: Create the S3 bucket (choose a globally unique name)
aws s3 mb s3://my-company-terraform-state-bucket-2024

# Enable versioning — lets you recover from state corruption
aws s3api put-bucket-versioning \
  --bucket my-company-terraform-state-bucket-2024 \
  --versioning-configuration Status=Enabled

# Enable encryption
aws s3api put-bucket-encryption \
  --bucket my-company-terraform-state-bucket-2024 \
  --server-side-encryption-configuration \
  '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'

# Step 2: Create DynamoDB table for state locking
aws dynamodb create-table \
  --table-name terraform-state-locks \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST
```

### Configure the Backend in `main.tf`

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  backend "s3" {
    bucket         = "my-company-terraform-state-bucket-2024"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-locks"
    encrypt        = true
  }
}
```

### Migrate Existing Local State to Remote

```bash
# After adding the backend block, re-initialize
terraform init -migrate-state

# Terraform will ask:
# "Do you want to copy existing state to the new backend?" → yes

# Verify state is now remote
terraform state list

# Your local terraform.tfstate is now empty — safe to delete
```

> **Pro tip:** The bootstrap problem: you can't use Terraform to create the S3 bucket that stores Terraform state. Create it manually once, then never include it in `terraform destroy`. Add a comment in your README explaining this.

---

## Step 4: Wire Modules Together with Outputs

In multi-module projects, modules share values through **outputs**. The networking module produces VPC and subnet IDs; the compute module needs them as inputs.

### Root `main.tf` — Wiring All Three Modules

```hcl
module "networking" {
  source      = "./modules/networking"
  environment = var.environment
  vpc_cidr    = var.vpc_cidr
}

module "compute" {
  source      = "./modules/compute"
  environment = var.environment

  # Cross-module references — networking outputs feed compute inputs
  vpc_id            = module.networking.vpc_id
  public_subnet_ids = module.networking.public_subnet_ids
  private_subnet_ids = module.networking.private_subnet_ids
}

module "storage" {
  source      = "./modules/storage"
  environment = var.environment

  # Storage may reference compute outputs too
  instance_role_arn = module.compute.instance_role_arn
}
```

### Root `variables.tf`

```hcl
variable "environment" {
  description = "Deployment environment"
  type        = string
  default     = "dev"
}

variable "vpc_cidr" {
  description = "CIDR block for the VPC"
  type        = string
  default     = "10.0.0.0/16"
}
```

### Root `outputs.tf`

```hcl
output "vpc_id" {
  description = "VPC ID"
  value       = module.networking.vpc_id
}

output "public_subnet_ids" {
  description = "Public subnet IDs"
  value       = module.networking.public_subnet_ids
}
```

### `terraform.tfvars` (not committed to Git)

```hcl
environment = "dev"
vpc_cidr    = "10.0.0.0/16"
```

> **Pro tip:** Never reference a resource from module A directly in module B (e.g., `module.networking.aws_subnet.public.id`). Always go through an output. Direct references create tight coupling that breaks when you refactor.

---

## Step 5: `.gitignore` for Terraform Projects

```gitignore
# Local Terraform state — never commit
*.tfstate
*.tfstate.backup
*.tfstate.*.backup

# Terraform working directory
.terraform/
.terraform.lock.hcl   # Commit this one! It pins provider versions for the team

# Variable files with sensitive data
*.tfvars
!example.tfvars       # Commit example files showing format, not real values

# Override files (local developer customizations)
override.tf
override.tf.json
*_override.tf
*_override.tf.json

# Crash logs
crash.log
crash.*.log

# Sensitive outputs
*.pem
*.key
```

---

## Running the Full Project

```bash
# Initialize all modules
terraform init

# Validate configuration across all modules
terraform validate

# Format all files
terraform fmt -recursive

# Plan — shows all resources across all modules
terraform plan -out=tfplan

# Apply from the saved plan (exact changes, no drift)
terraform apply tfplan

# View all outputs
terraform output

# Always destroy when done practicing
terraform destroy
```

---

## Phase 2 Checklist

- [ ] 3-tier module structure created (networking / compute / storage)
- [ ] Data sources used for at least one resource (e.g., AMI lookup)
- [ ] Remote state configured in S3 + DynamoDB locking
- [ ] `terraform init -migrate-state` completed successfully
- [ ] Cross-module references working (networking outputs → compute inputs)
- [ ] `.gitignore` configured properly
- [ ] `terraform destroy` removes all resources cleanly

---

## What's Next

**Phase 3** adds CI/CD: a GitHub Actions pipeline that runs `plan` on every pull request and `apply` on merge to main — the way Terraform is run in every real production team.

---

*Part of the Terraform Next Steps series — 5 phases from local practice to production-ready skills.*
