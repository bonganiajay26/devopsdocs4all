# Terraform Next Steps — Phase 1: Cloud Provider Setup

> **Goal:** Connect Terraform to a real cloud provider and run your first real `apply`/`destroy` cycle — no charges, no credit card risk.
> **Timeline:** Week 1–2
> **Prerequisites:** Completed the local Terraform guide (modules + no-modules examples)

---

## Step 1: Choose a Free-Tier Cloud Provider

All three major providers offer free tiers generous enough for Terraform practice:

| Provider | Free Tier Type | Key Free Resources |
|---|---|---|
| **AWS** | 12-month free | EC2 t2.micro, S3, RDS (db.t2.micro), Lambda |
| **GCP** | Always free | e2-micro VM, Cloud Storage, Cloud Functions |
| **Azure** | 12-month + always free | B1S VM, Blob Storage, Azure Functions |

**Recommendation:** Start with **AWS** — it is the most common in job interviews and has the largest Terraform provider with the most community examples.

> **Pro tip:** AWS is most common in job interviews. GCP has the most generous always-free compute (e2-micro VM). Pick one and go deep rather than spreading across all three.

---

## Step 2: Install Provider CLI & Configure Credentials

Each provider needs a CLI for authentication. Terraform reads credentials from environment variables or config files — **never hardcode them in `.tf` files**.

### AWS Setup

```bash
# Install AWS CLI
pip install awscli
# or: brew install awscli  (macOS)

# Configure credentials
aws configure
# You'll be prompted for:
# AWS Access Key ID     [get from IAM Console → Users → Security credentials]
# AWS Secret Access Key [shown only once at creation]
# Default region name   [e.g. us-east-1]
# Default output format [json]

# Verify credentials work
aws sts get-caller-identity
# Expected output: your Account ID, UserID, and ARN
```

### GCP Setup

```bash
# Install Google Cloud CLI
# https://cloud.google.com/sdk/docs/install

# Login and set application default credentials
gcloud auth application-default login
gcloud config set project YOUR_PROJECT_ID
```

### Azure Setup

```bash
# Install Azure CLI
brew install azure-cli  # macOS
# or: https://docs.microsoft.com/en-us/cli/azure/install-azure-cli

# Login
az login
az account set --subscription "YOUR_SUBSCRIPTION_ID"
```

> **Pro tip:** Use `aws sts get-caller-identity` to verify AWS credentials before running `terraform plan`. A failed `init` is almost always a credential problem.

---

## Step 3: Swap the Local Provider for a Real One

Replace `hashicorp/local` with your cloud provider. **The module structure stays identical — only the resource types change.** This is Terraform's superpower: same workflow, any cloud.

### `main.tf` — AWS Example

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.5"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

# Random suffix — S3 bucket names are globally unique
resource "random_id" "suffix" {
  byte_length = 4
}

# Your first real cloud resource
resource "aws_s3_bucket" "learning" {
  bucket = "my-terraform-learning-${random_id.suffix.hex}"

  tags = {
    Environment = "learning"
    Owner       = "your-name"
    ManagedBy   = "terraform"
  }
}

# Block all public access (security best practice)
resource "aws_s3_bucket_public_access_block" "learning" {
  bucket = aws_s3_bucket.learning.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

> **Pro tip:** Always add a random suffix to bucket names — S3 bucket names are globally unique across all AWS accounts. Without a suffix, `my-terraform-bucket` will fail because someone else already owns it.

---

## Step 4: Run the Full Workflow on Real Infrastructure

The exact same commands from the local guide — now making real API calls to create real cloud resources.

```bash
# Initialize — downloads the AWS provider plugin (~50MB)
terraform init

# Preview what will be created
terraform plan

# Create the real cloud resources
terraform apply
# Type 'yes' when prompted

# Verify the bucket exists in AWS
aws s3 ls | grep terraform-learning

# See what Terraform created
terraform show

# Read a specific output value
terraform output

# IMPORTANT: Destroy when done to avoid any charges
terraform destroy
# Type 'yes' when prompted
```

### What You'll See After `apply`

```
Apply complete! Resources: 2 added, 0 changed, 0 destroyed.
```

Check the AWS Console → S3 to see your bucket. Then immediately run `terraform destroy` — good habit from day one.

> **Pro tip:** Set up AWS billing alerts immediately. Go to **AWS Console → Billing → Budgets → Create Budget**. Set a $5 monthly threshold — you'll get an email before any real charges accumulate.

---

## Key Differences: Local Provider vs Real Cloud Provider

| Aspect | `hashicorp/local` | Real Cloud Provider |
|---|---|---|
| **Resources created** | Files on your disk | Real cloud infrastructure |
| **Credentials needed** | None | IAM/Service Account/Service Principal |
| **`terraform init` download** | ~1MB | ~30–80MB |
| **`apply` speed** | Instant | 5 seconds to 10+ minutes |
| **`destroy` needed?** | Not critical | Yes — avoid charges |
| **State drift possible?** | No | Yes — if someone changes infra manually |

---

## Troubleshooting Common Issues

### Error: `NoCredentialProviders`
```bash
# AWS credentials not configured
aws configure list          # Check what's set
aws configure               # Re-run setup
export AWS_PROFILE=default  # Or set profile explicitly
```

### Error: `BucketAlreadyExists`
```bash
# Add a random suffix (see Step 3 above)
# Or choose a more unique bucket name
```

### Error: `AccessDenied`
```bash
# Your IAM user lacks permissions
# Minimum permissions needed for S3:
# s3:CreateBucket, s3:DeleteBucket, s3:PutBucketPublicAccessBlock
# Easiest for learning: attach the AmazonS3FullAccess managed policy
```

### Error: `InvalidClientTokenId`
```bash
# Credentials are malformed or expired
aws sts get-caller-identity   # Verify credentials
aws configure                 # Re-enter them
```

---

## Phase 1 Checklist

Before moving to Phase 2, confirm:

- [ ] Cloud provider CLI installed and configured
- [ ] `aws sts get-caller-identity` (or equivalent) returns your account info
- [ ] `terraform init` completes without errors (downloads real provider)
- [ ] `terraform plan` shows real resources to be created
- [ ] `terraform apply` successfully creates resources in the cloud console
- [ ] `terraform destroy` cleanly removes all resources
- [ ] Billing alert set up (AWS users)

---

## What's Next

**Phase 2** builds on this foundation — you'll create a full 3-tier web architecture (VPC + compute + storage) using proper modules, data sources, and remote state storage. That's the architecture you'll build in take-home interviews.

---

*Part of the Terraform Next Steps series — 5 phases from local practice to production-ready skills.*
