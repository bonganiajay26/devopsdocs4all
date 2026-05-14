# Terraform Next Steps — Phase 3: CI/CD Integration

> **Goal:** Automate Terraform in a GitHub Actions pipeline — the way it runs in every real production team.
> **Timeline:** Week 6–7
> **Prerequisites:** Phase 2 complete — remote state configured, 3-tier module structure working

---

## Why CI/CD for Terraform?

In real teams, **nobody runs `terraform apply` from their laptop.** Reasons:

- **State drift:** Two engineers applying simultaneously can corrupt state or create duplicate resources
- **No peer review:** A manual apply has no approval step before infrastructure changes
- **No audit trail:** CI/CD logs every plan and apply with timestamps and who triggered it
- **Consistency:** CI environment is identical every run; your laptop isn't

**The standard pattern:**
- Pull Request opened → `terraform plan` runs automatically → output posted as PR comment
- PR merged to `main` → `terraform apply` runs automatically

---

## Step 1: Add a GitHub Actions Workflow

Create this file in your repository:

### `.github/workflows/terraform.yml`

```yaml
name: Terraform CI/CD

on:
  pull_request:
    branches: [main]
    paths:
      - '**.tf'
      - '**.tfvars'
  push:
    branches: [main]
    paths:
      - '**.tf'
      - '**.tfvars'

# Prevent concurrent applies on the same branch
concurrency:
  group: terraform-${{ github.ref }}
  cancel-in-progress: false  # Don't cancel — wait for the lock

env:
  TF_VERSION: "1.7.0"
  AWS_REGION: "us-east-1"

jobs:
  terraform:
    name: Terraform Plan & Apply
    runs-on: ubuntu-latest

    permissions:
      contents: read
      pull-requests: write  # Needed to post plan output as PR comment

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Terraform Format Check
        id: fmt
        run: terraform fmt -check -recursive
        continue-on-error: true   # Show result but don't fail the whole job

      - name: Terraform Init
        id: init
        run: terraform init

      - name: Terraform Validate
        id: validate
        run: terraform validate -no-color

      - name: Terraform Plan (on Pull Request)
        id: plan
        if: github.event_name == 'pull_request'
        run: terraform plan -no-color -out=tfplan
        continue-on-error: true

      - name: Post Plan as PR Comment
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const output = `#### Terraform Format Check: \`${{ steps.fmt.outcome }}\`
            #### Terraform Init: \`${{ steps.init.outcome }}\`
            #### Terraform Validate: \`${{ steps.validate.outcome }}\`
            #### Terraform Plan: \`${{ steps.plan.outcome }}\`

            <details><summary>Show Plan</summary>

            \`\`\`terraform
            ${{ steps.plan.outputs.stdout }}
            \`\`\`

            </details>

            *Triggered by @${{ github.actor }}, Action: \`${{ github.event_name }}\`*`;

            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            });

      - name: Terraform Plan Exit Code Check
        if: steps.plan.outcome == 'failure'
        run: exit 1

      - name: Terraform Apply (on merge to main)
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        run: terraform apply -auto-approve -no-color
```

---

## Step 2: Store AWS Credentials in GitHub Secrets

**Never put credentials in the workflow file.** Store them in GitHub Secrets:

1. Go to your repository on GitHub
2. Click **Settings → Secrets and variables → Actions**
3. Click **New repository secret**
4. Add these two secrets:
   - `AWS_ACCESS_KEY_ID` — your AWS access key
   - `AWS_SECRET_ACCESS_KEY` — your AWS secret key

### Better Alternative: OIDC (No Long-Lived Credentials)

GitHub Actions can assume an AWS IAM role directly — no stored secrets needed:

```yaml
# In your workflow, replace the aws-credentials step with:
- name: Configure AWS credentials via OIDC
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789012:role/github-actions-terraform
    aws-region: us-east-1
```

```hcl
# In Terraform — create the IAM role that GitHub can assume
resource "aws_iam_openid_connect_provider" "github" {
  url             = "https://token.actions.githubusercontent.com"
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = ["6938fd4d98bab03faadb97b34396831e3780aea1"]
}

resource "aws_iam_role" "github_actions" {
  name = "github-actions-terraform"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Federated = aws_iam_openid_connect_provider.github.arn }
      Action    = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringLike = {
          "token.actions.githubusercontent.com:sub" = "repo:YOUR_ORG/YOUR_REPO:*"
        }
      }
    }]
  })
}
```

> **Pro tip:** OIDC is the gold standard — no long-lived credentials to rotate or accidentally expose. Use it from the start on real projects.

---

## Step 3: Auto-Generate Documentation with terraform-docs

`terraform-docs` generates README files from your `variables.tf` and `outputs.tf` automatically. Keeps documentation in sync with code.

### Install

```bash
# macOS
brew install terraform-docs

# Linux
curl -sSLo ./terraform-docs.tar.gz \
  https://terraform-docs.io/dl/v0.17.0/terraform-docs-v0.17.0-linux-amd64.tar.gz
tar -xzf terraform-docs.tar.gz
chmod +x terraform-docs
mv terraform-docs /usr/local/bin/
```

### Generate Docs for a Module

```bash
# Generate markdown docs for the networking module
terraform-docs markdown ./modules/networking

# Write directly to README.md
terraform-docs markdown ./modules/networking > ./modules/networking/README.md

# All modules at once
for dir in modules/*/; do
  terraform-docs markdown "$dir" > "$dir/README.md"
  echo "Generated docs for $dir"
done
```

### Example Output

Given a `variables.tf` with:
```hcl
variable "environment" {
  description = "Deployment environment (dev, staging, prod)"
  type        = string
}
```

`terraform-docs` generates:

```markdown
## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| environment | Deployment environment (dev, staging, prod) | `string` | n/a | yes |
```

### Add as a Pre-Commit Hook

```bash
# Install pre-commit
pip install pre-commit

# Create .pre-commit-config.yaml in your repo root
```

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.83.5
    hooks:
      - id: terraform_fmt
      - id: terraform_validate
      - id: terraform_docs
        args:
          - --hook-config=--path-to-file=README.md
          - --hook-config=--add-to-existing-file=true
```

```bash
# Install hooks
pre-commit install

# Run against all files once
pre-commit run --all-files
```

> **Pro tip:** Add `terraform-docs` as a pre-commit hook so docs auto-update when you change variables or outputs. Interviewers notice when module READMEs are out of date — it signals sloppy habits.

---

## Step 4: Add Security Scanning

Static analysis tools scan your `.tf` files for security misconfigurations before they reach production.

### Option A: tfsec

```bash
# Install
brew install tfsec           # macOS
# or: docker pull aquasec/tfsec

# Run scan
tfsec .

# Run with specific output format
tfsec . --format=json

# Suppress a specific rule (with documented reason)
# In your .tf file:
resource "aws_s3_bucket" "logs" {
  bucket = "my-log-bucket"
  #tfsec:ignore:aws-s3-enable-bucket-logging -- This IS the log bucket
}
```

### Option B: Checkov (more rules, Python-based)

```bash
# Install
pip install checkov

# Run scan
checkov -d .

# Run with specific checks only
checkov -d . --check CKV_AWS_18,CKV_AWS_19

# Skip specific checks
checkov -d . --skip-check CKV_AWS_144,CKV_AWS_145

# Soft fail — report but don't fail the build (useful when starting out)
checkov -d . --soft-fail
```

### Add Security Scanning to GitHub Actions

```yaml
# Add this job to your terraform.yml workflow
  security-scan:
    name: Security Scan
    runs-on: ubuntu-latest
    needs: []  # Run in parallel with terraform job

    steps:
      - uses: actions/checkout@v4

      - name: Run Checkov
        uses: bridgecrewio/checkov-action@master
        with:
          directory: .
          framework: terraform
          soft_fail: true          # Don't block PRs while learning
          output_format: sarif
          output_file_path: results.sarif

      - name: Upload SARIF results
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: results.sarif
```

> **Pro tip:** Both tools produce warnings on learning projects (e.g., "S3 bucket logging not enabled"). Use `#tfsec:ignore:rule-id` or `#checkov:skip=CKV_AWS_XXX` comments to suppress rules you're intentionally skipping — always add a comment explaining why.

---

## Step 5: Add Terraform Format and Validate as Gates

These two commands should be gates on every PR — they catch the most common problems:

```bash
# terraform fmt — auto-formats all .tf files to HCL standard style
# The -check flag fails if any file needs formatting (CI mode)
# The -recursive flag includes subdirectories (modules)
terraform fmt -check -recursive

# To fix locally: run without -check
terraform fmt -recursive

# terraform validate — checks for syntax errors and configuration issues
# Does NOT connect to any cloud provider
# Fast — runs in seconds even on large configs
terraform validate
```

### What Each Catches

| Command | Catches | Doesn't Catch |
|---|---|---|
| `terraform fmt` | Inconsistent indentation, alignment, formatting | Logic errors, missing resources |
| `terraform validate` | Syntax errors, missing required variables, invalid references | Cloud-specific errors (wrong AMI ID, insufficient permissions) |
| `terraform plan` | Everything above + what will actually change | Errors that only appear at apply time |
| Security scanners | Misconfigurations (public buckets, open ports) | Runtime errors |

---

## Complete CI/CD Pipeline Summary

```
Developer pushes code
         │
         ▼
  Pull Request opened
         │
         ├─── terraform fmt -check    (formatting)
         ├─── terraform validate      (syntax)
         ├─── checkov / tfsec         (security)
         └─── terraform plan          (what changes?)
                      │
                      ▼
            Plan output posted as PR comment
                      │
                      ▼
         Code review + plan review
                      │
                      ▼
              PR merged to main
                      │
                      ▼
           terraform apply -auto-approve
                      │
                      ▼
         Real infrastructure updated
```

---

## Phase 3 Checklist

- [ ] `.github/workflows/terraform.yml` created and pushed
- [ ] AWS credentials stored in GitHub Secrets (or OIDC configured)
- [ ] Pipeline runs on PR: `fmt` + `validate` + `plan`
- [ ] Plan output appears as a PR comment
- [ ] Pipeline runs `apply` on merge to main
- [ ] `terraform-docs` generating README files for each module
- [ ] Security scan (tfsec or Checkov) passing in CI
- [ ] Pre-commit hooks installed locally

---

## What's Next

**Phase 4** covers advanced patterns: Terragrunt for DRY multi-environment setups, `terraform import` for existing resources, HCP Terraform Cloud, and automated testing with Terratest.

---

*Part of the Terraform Next Steps series — 5 phases from local practice to production-ready skills.*
