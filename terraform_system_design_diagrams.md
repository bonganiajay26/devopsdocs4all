# Terraform System Design Diagrams — Complete Reference

> All major Terraform architecture patterns: multi-cloud, multi-region, multi-account, workspace strategies, CI/CD, state management, and more. Every diagram uses ASCII art for universal rendering in any Markdown viewer, IDE, or documentation platform.

---

## Table of Contents

1. [Core Terraform Architecture](#1-core-terraform-architecture)
2. [Single Cloud Single Region (Baseline)](#2-single-cloud-single-region-baseline)
3. [Multi-Region — Single Cloud](#3-multi-region--single-cloud)
4. [Multi-Account — AWS Organizations](#4-multi-account--aws-organizations)
5. [Multi-Cloud — AWS + Azure + GCP](#5-multi-cloud--aws--azure--gcp)
6. [Multi-Cloud + Multi-Region](#6-multi-cloud--multi-region)
7. [State Management Architecture](#7-state-management-architecture)
8. [Remote State with Cross-Stack References](#8-remote-state-with-cross-stack-references)
9. [Workspace Strategy Patterns](#9-workspace-strategy-patterns)
10. [CI/CD Pipeline Architecture](#10-cicd-pipeline-architecture)
11. [Module Registry Architecture](#11-module-registry-architecture)
12. [Hub-and-Spoke Network (Multi-Region)](#12-hub-and-spoke-network-multi-region)
13. [Disaster Recovery Architecture](#13-disaster-recovery-architecture)
14. [Zero-Trust Multi-Cloud Networking](#14-zero-trust-multi-cloud-networking)
15. [GitOps Terraform Architecture](#15-gitops-terraform-architecture)
16. [Terragrunt Multi-Environment Architecture](#16-terragrunt-multi-environment-architecture)
17. [Landing Zone Architecture](#17-landing-zone-architecture)
18. [Service Mesh Multi-Region](#18-service-mesh-multi-region)
19. [Data Platform Multi-Cloud](#19-data-platform-multi-cloud)
20. [Security & Compliance Architecture](#20-security--compliance-architecture)
21. [Pattern Selection Guide](#21-pattern-selection-guide)

---

## 1. Core Terraform Architecture

How the core Terraform components relate to each other.

```
┌─────────────────────────────────────────────────────────────────────┐
│                        TERRAFORM CORE FLOW                          │
└─────────────────────────────────────────────────────────────────────┘

  Developer / CI System
         │
         │  terraform init / plan / apply
         ▼
┌────────────────────┐
│   Terraform CLI    │  ◄── .tf configuration files
│  (terraform core)  │  ◄── variables.tfvars
└────────┬───────────┘  ◄── .terraform.lock.hcl
         │
         ├─────────────────────────────────────────┐
         │                                         │
         ▼                                         ▼
┌────────────────────┐                   ┌────────────────────┐
│  Provider Plugins  │                   │    State Backend   │
│                    │                   │                    │
│  ┌──────────────┐  │                   │  ┌──────────────┐  │
│  │  AWS Plugin  │  │                   │  │  Local File  │  │
│  └──────────────┘  │                   │  │ terraform    │  │
│  ┌──────────────┐  │                   │  │  .tfstate    │  │
│  │ Azure Plugin │  │                   │  └──────────────┘  │
│  └──────────────┘  │                   │        OR          │
│  ┌──────────────┐  │                   │  ┌──────────────┐  │
│  │  GCP Plugin  │  │                   │  │  Remote S3   │  │
│  └──────────────┘  │                   │  │  Azure Blob  │  │
│  ┌──────────────┐  │                   │  │  GCS Bucket  │  │
│  │  K8s Plugin  │  │                   │  │  HCP TF Cloud│  │
│  └──────────────┘  │                   │  └──────────────┘  │
└────────┬───────────┘                   └────────────────────┘
         │
         │  API Calls (HTTPS)
         ▼
┌──────────────────────────────────────────────────────────────────┐
│                    CLOUD PROVIDERS / TARGETS                      │
│                                                                   │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│   │     AWS      │  │    Azure     │  │        GCP           │  │
│   │  (EC2, S3,   │  │  (VMs, Blob, │  │  (GCE, GCS,          │  │
│   │  RDS, VPC)   │  │  SQL, VNet)  │  │   GKE, BigQuery)     │  │
│   └──────────────┘  └──────────────┘  └──────────────────────┘  │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│   │  Kubernetes  │  │   GitHub     │  │     Datadog          │  │
│   │  (Pods,      │  │  (Repos,     │  │  (Monitors,          │  │
│   │  Services)   │  │  Actions)    │  │   Dashboards)        │  │
│   └──────────────┘  └──────────────┘  └──────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘

  Terraform State (what exists)
  Terraform Config (what you want)
       │
       └── DIFF → Plan → Apply
```

---

## 2. Single Cloud Single Region (Baseline)

The simplest production-ready pattern. Master this before adding complexity.

```
┌─────────────────────────────────────────────────────────────────────┐
│              SINGLE CLOUD / SINGLE REGION — AWS us-east-1           │
└─────────────────────────────────────────────────────────────────────┘

  GitHub Repository
  ├── main.tf
  ├── variables.tf / outputs.tf / terraform.tfvars
  └── modules/
      ├── networking/
      ├── compute/
      └── storage/
             │
             │ terraform apply
             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    AWS — us-east-1                                  │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    VPC  10.0.0.0/16                         │   │
│  │                                                             │   │
│  │  ┌────────────────────┐    ┌────────────────────┐           │   │
│  │  │   Public Subnet    │    │   Public Subnet     │           │   │
│  │  │   10.0.1.0/24      │    │   10.0.2.0/24       │           │   │
│  │  │   AZ: us-east-1a   │    │   AZ: us-east-1b    │           │   │
│  │  │  ┌──────────────┐  │    │  ┌───────────────┐  │           │   │
│  │  │  │     ALB      │◄─┼────┼─►│     ALB       │  │           │   │
│  │  │  └──────┬───────┘  │    │  └───────┬───────┘  │           │   │
│  │  └─────────┼──────────┘    └──────────┼──────────┘           │   │
│  │            │                          │                       │   │
│  │  ┌─────────┼──────────┐    ┌──────────┼──────────┐           │   │
│  │  │ Private Subnet     │    │  Private Subnet      │           │   │
│  │  │ 10.0.10.0/24       │    │  10.0.11.0/24        │           │   │
│  │  │ AZ: us-east-1a     │    │  AZ: us-east-1b      │           │   │
│  │  │   ┌────────────┐   │    │   ┌────────────┐     │           │   │
│  │  │   │  EC2 / ASG │   │    │   │  EC2 / ASG │     │           │   │
│  │  │   └────────────┘   │    │   └────────────┘     │           │   │
│  │  └────────────────────┘    └──────────────────────┘           │   │
│  │                                                               │   │
│  │  ┌─────────────────────────────────────────────────────────┐  │   │
│  │  │              DB Subnet Group (Multi-AZ)                 │  │   │
│  │  │   RDS PostgreSQL — Primary (1a)    Standby (1b)          │  │   │
│  │  └─────────────────────────────────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌─────────────────┐  ┌─────────────────┐  ┌───────────────────┐  │
│  │   S3 Bucket     │  │  Terraform State │  │  DynamoDB Lock    │  │
│  │  (app assets)   │  │  (S3 Backend)    │  │  Table            │  │
│  └─────────────────┘  └─────────────────┘  └───────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘

  Module wiring:
  networking (outputs: vpc_id, subnet_ids)
       → compute (inputs: vpc_id, subnet_ids)
       → storage (inputs: compute role ARN)
```

---

## 3. Multi-Region — Single Cloud

Deploy the same infrastructure to multiple AWS regions for latency reduction and disaster recovery.

```
┌─────────────────────────────────────────────────────────────────────┐
│                 MULTI-REGION — AWS (Active/Active)                  │
└─────────────────────────────────────────────────────────────────────┘

  Project Layout:
  ┌───────────────────────────────────────────────────────────────┐
  │  environments/                                                │
  │  ├── us-east-1/   main.tf  ← provider { region="us-east-1" } │
  │  ├── eu-west-1/   main.tf  ← provider { region="eu-west-1" } │
  │  └── ap-southeast-1/ main.tf                                 │
  │  modules/          ← Shared by all regions                   │
  └───────────────────────────────────────────────────────────────┘

                        ┌──────────────────┐
                        │    Route 53      │
                        │  Latency-based   │
                        │  Routing +       │
                        │  Health Checks   │
                        └────────┬─────────┘
               ┌─────────────────┼──────────────────┐
               │                 │                  │
               ▼                 ▼                  ▼
  ┌────────────────────┐ ┌──────────────────┐ ┌────────────────────┐
  │   AWS us-east-1    │ │  AWS eu-west-1   │ │ AWS ap-southeast-1 │
  │   (N. Virginia)    │ │   (Ireland)      │ │   (Singapore)      │
  │                    │ │                  │ │                    │
  │  VPC 10.0.0.0/16   │ │  VPC 10.1.0.0/16 │ │  VPC 10.2.0.0/16  │
  │  ALB + ASG         │ │  ALB + ASG       │ │  ALB + ASG         │
  │  RDS Primary       │ │  RDS Replica     │ │  RDS Replica       │
  │  State: S3 (1)     │ │  State: S3 (2)   │ │  State: S3 (3)     │
  └────────────────────┘ └──────────────────┘ └────────────────────┘
               │                 │                  │
               └─────────────────┴──────────────────┘
                                 │
                    ┌────────────▼─────────────┐
                    │  DynamoDB Global Tables   │
                    │  (replicated all regions) │
                    └──────────────────────────┘

  VPC Peering / Transit Gateway:
  us-east-1 ◄──────────────► eu-west-1
  us-east-1 ◄──────────────► ap-southeast-1
  eu-west-1 ◄──────────────► ap-southeast-1

  Provider Aliases (multi-region in one root config):
  ┌──────────────────────────────────────────────────────────────┐
  │  provider "aws" { alias = "us_east"; region = "us-east-1" } │
  │  provider "aws" { alias = "eu_west"; region = "eu-west-1"  } │
  │                                                              │
  │  module "us_stack" {                                         │
  │    source    = "./modules/stack"                             │
  │    providers = { aws = aws.us_east }                         │
  │  }                                                           │
  │  module "eu_stack" {                                         │
  │    source    = "./modules/stack"                             │
  │    providers = { aws = aws.eu_west }                         │
  │  }                                                           │
  └──────────────────────────────────────────────────────────────┘
```

---

## 4. Multi-Account — AWS Organizations

Enterprise pattern: separate AWS accounts per environment managed through AWS Organizations.

```
┌─────────────────────────────────────────────────────────────────────┐
│              MULTI-ACCOUNT — AWS ORGANIZATIONS                      │
└─────────────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────┐
  │               AWS MANAGEMENT ACCOUNT                             │
  │           (Billing, SCPs, Organizations)                         │
  │                                                                  │
  │      Root OU                                                     │
  │      │                                                           │
  │      ├── Security OU                                             │
  │      │   ├── Security Account  (GuardDuty, CloudTrail)           │
  │      │   └── Log Archive Account                                 │
  │      │                                                           │
  │      ├── Infrastructure OU                                       │
  │      │   ├── Shared Services Account  (ECR, CI/CD, Monitoring)   │
  │      │   └── Network Account          (TGW, VPCs, DNS)           │
  │      │                                                           │
  │      └── Workloads OU                                            │
  │          ├── Dev Account                                         │
  │          ├── Staging Account                                     │
  │          └── Prod Account                                        │
  └──────────────────────────────────────────────────────────────────┘

  Cross-Account Role Assumption:
  ┌──────────────────────────────────────────────────────────────────┐
  │  provider "aws" {                                                │
  │    alias  = "prod"                                               │
  │    region = "us-east-1"                                          │
  │    assume_role {                                                  │
  │      role_arn = "arn:aws:iam::PROD_ACCT_ID:role/TerraformRole"  │
  │    }                                                             │
  │  }                                                               │
  └──────────────────────────────────────────────────────────────────┘

  Terraform Project Structure:
  ┌──────────────────────────────────────────────────────────────────┐
  │  accounts/                                                       │
  │  ├── management/    ← SCPs, Organizations config                 │
  │  ├── security/      ← GuardDuty, SecurityHub, CloudTrail         │
  │  ├── network/       ← Transit Gateway, Shared VPCs               │
  │  ├── shared-svcs/   ← AMI factory, artifact repos                │
  │  ├── dev/           ← Dev workloads                              │
  │  ├── staging/       ← Staging workloads                          │
  │  └── prod/          ← Production workloads                       │
  │  modules/           ← All reusable modules                       │
  └──────────────────────────────────────────────────────────────────┘

  State Files — One Per Account:
  ┌──────────────────────────────────────────────────────────────────┐
  │  s3://tf-state/accounts/dev/terraform.tfstate                    │
  │  s3://tf-state/accounts/staging/terraform.tfstate                │
  │  s3://tf-state/accounts/prod/terraform.tfstate                   │
  │  s3://tf-state/accounts/network/terraform.tfstate                │
  └──────────────────────────────────────────────────────────────────┘
```

---

## 5. Multi-Cloud — AWS + Azure + GCP

Deploy and manage infrastructure across all three major cloud providers.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    MULTI-CLOUD ARCHITECTURE                         │
└─────────────────────────────────────────────────────────────────────┘

  terraform { required_providers {
    aws     = { source = "hashicorp/aws",     version = "~>5.0" }
    azurerm = { source = "hashicorp/azurerm", version = "~>3.0" }
    google  = { source = "hashicorp/google",  version = "~>4.0" }
  }}

  ┌──────────────────┐  ┌──────────────────┐  ┌────────────────────┐
  │      AWS         │  │     AZURE        │  │      GCP           │
  │  (Primary)       │  │  (Secondary)     │  │  (Analytics)       │
  │                  │  │                  │  │                    │
  │  VPC             │  │  VNet            │  │  VPC Network       │
  │  EKS             │  │  AKS             │  │  GKE               │
  │  EC2             │  │  VM Scale Sets   │  │  GCE               │
  │  RDS             │  │  Azure SQL       │  │  Cloud SQL         │
  │  S3              │  │  Blob Storage    │  │  GCS               │
  │  Route53         │  │  Azure DNS       │  │  Cloud DNS         │
  │  us-east-1       │  │  eastus          │  │  us-central1       │
  └──────────────────┘  └──────────────────┘  └────────────────────┘
          │                      │                      │
          └──────────────────────┼──────────────────────┘
                                 │
                    ┌────────────▼──────────────┐
                    │   Cross-Cloud Networking   │
                    │  VPN / ExpressRoute /      │
                    │  Direct Connect            │
                    │  Cloudflare (Global LB)    │
                    └───────────────────────────┘

  Module Structure (cloud-specific subdirs):
  ┌────────────────────────────────────────────────────────────────┐
  │  modules/                                                      │
  │  ├── compute/                                                  │
  │  │   ├── aws/    resource "aws_instance"                       │
  │  │   ├── azure/  resource "azurerm_linux_virtual_machine"      │
  │  │   └── gcp/    resource "google_compute_instance"            │
  │  ├── kubernetes/                                               │
  │  │   ├── aws/    resource "aws_eks_cluster"                    │
  │  │   ├── azure/  resource "azurerm_kubernetes_cluster"         │
  │  │   └── gcp/    resource "google_container_cluster"           │
  │  └── object-storage/                                           │
  │      ├── aws/    resource "aws_s3_bucket"                      │
  │      ├── azure/  resource "azurerm_storage_account"            │
  │      └── gcp/    resource "google_storage_bucket"              │
  └────────────────────────────────────────────────────────────────┘

  State Files Per Cloud:
  s3://tf-state/aws/prod/terraform.tfstate
  s3://tf-state/azure/prod/terraform.tfstate
  s3://tf-state/gcp/prod/terraform.tfstate
```

---

## 6. Multi-Cloud + Multi-Region

The most complex pattern — multiple clouds, each deployed across multiple regions.

```
┌─────────────────────────────────────────────────────────────────────┐
│              MULTI-CLOUD + MULTI-REGION ARCHITECTURE                │
└─────────────────────────────────────────────────────────────────────┘

                    ┌─────────────────────────┐
                    │    Global Traffic Manager│
                    │  (Cloudflare / Akamai)   │
                    │  GeoDNS + Health Checks  │
                    └────────────┬────────────┘
          ┌─────────────────────┼─────────────────────┐
          │                     │                     │
          ▼                     ▼                     ▼
   NORTH AMERICA           EUROPE (EU)          ASIA-PACIFIC
          │                     │                     │
  ┌───────┴──────┐      ┌───────┴──────┐      ┌───────┴──────┐
  │              │      │              │      │              │
  ▼              ▼      ▼              ▼      ▼              ▼
┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
│  AWS     │ │  Azure   │ │  AWS     │ │  GCP     │ │  AWS     │
│us-east-1 │ │ eastus   │ │eu-west-1 │ │europe-w1 │ │ap-seast-1│
│          │ │          │ │          │ │          │ │          │
│ EKS      │ │ AKS      │ │ EKS      │ │ GKE      │ │ EKS      │
│ RDS      │ │ Azure SQL│ │ RDS Rep. │ │ Cloud SQL│ │ RDS Rep. │
│ S3       │ │ Blob     │ │ S3       │ │ GCS      │ │ S3       │
└──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘
      │             │           │             │            │
      └─────────────┴───────────┴─────────────┴────────────┘
                              │
                ┌─────────────▼──────────────┐
                │    Global Data Plane        │
                │  CockroachDB / Spanner      │
                │  Confluent Kafka            │
                │  (cross-cloud events)       │
                └─────────────────────────────┘

  Terraform Project Layout:
  ┌──────────────────────────────────────────────────────────────────┐
  │  infrastructure/                                                 │
  │  ├── global/                 ← Route53, Cloudflare, IAM Roles    │
  │  ├── aws/                                                        │
  │  │   ├── us-east-1/prod/     ← provider alias: aws.use1          │
  │  │   ├── eu-west-1/prod/                                         │
  │  │   └── ap-southeast-1/prod/                                    │
  │  ├── azure/                                                      │
  │  │   ├── eastus/prod/                                            │
  │  │   └── westeurope/prod/                                        │
  │  ├── gcp/                                                        │
  │  │   ├── us-central1/prod/                                       │
  │  │   └── europe-west1/prod/                                      │
  │  └── modules/                ← Cloud-agnostic where possible     │
  └──────────────────────────────────────────────────────────────────┘

  State Architecture (one file per cloud+region+env):
  tf-state/global/terraform.tfstate
  tf-state/aws/us-east-1/prod/terraform.tfstate
  tf-state/aws/eu-west-1/prod/terraform.tfstate
  tf-state/azure/eastus/prod/terraform.tfstate
  tf-state/gcp/us-central1/prod/terraform.tfstate
```

---

## 7. State Management Architecture

All state backend patterns and when to use each.

```
┌─────────────────────────────────────────────────────────────────────┐
│                 STATE MANAGEMENT ARCHITECTURE                       │
└─────────────────────────────────────────────────────────────────────┘

  PATTERN 1: Local State (Solo / Learning)
  ┌──────────────────────────────────────────────────────────────────┐
  │  project/                                                        │
  │  ├── main.tf                                                     │
  │  └── terraform.tfstate   ← Lives on disk, no sharing            │
  │  ⚠ Lost if disk fails, cannot collaborate                       │
  └──────────────────────────────────────────────────────────────────┘

  PATTERN 2: S3 + DynamoDB (Small-Medium Team, AWS)
  ┌──────────────────────────────────────────────────────────────────┐
  │                                                                  │
  │  Developer #1 ──┐                                                │
  │  Developer #2 ──┼──► S3 Bucket (tfstate)  encrypt=true          │
  │  CI/CD ─────────┘              versioned=true                    │
  │                                     │                            │
  │                          Lock acquired before read/write         │
  │                                     │                            │
  │                                     ▼                            │
  │                          DynamoDB Table                          │
  │                          (LockID = state path)                   │
  │                                                                  │
  │  backend "s3" {                                                  │
  │    bucket         = "company-terraform-state"                    │
  │    key            = "prod/networking/terraform.tfstate"          │
  │    region         = "us-east-1"                                  │
  │    dynamodb_table = "terraform-state-lock"                       │
  │    encrypt        = true                                         │
  │  }                                                               │
  └──────────────────────────────────────────────────────────────────┘

  PATTERN 3: HCP Terraform Cloud (Teams, Multi-Cloud)
  ┌──────────────────────────────────────────────────────────────────┐
  │  HCP Terraform Cloud — Organization: my-company                  │
  │                                                                  │
  │  Workspace: prod-networking                                      │
  │  ├── State (encrypted, versioned, history)                       │
  │  ├── Variables (sensitive env vars, non-sensitive)               │
  │  ├── Run History (plan + apply logs with timestamps)             │
  │  └── Access Control: Team "platform-eng" = Apply                 │
  │                                                                  │
  │  Workspace: prod-compute                                         │
  │  ├── State                                                       │
  │  ├── Remote state ref → prod-networking outputs                  │
  │  └── Access Control: Team "compute-eng" = Apply                  │
  │                                                                  │
  │  Policy Set (Sentinel):                                          │
  │  ├── no-public-s3-buckets.sentinel                               │
  │  ├── require-tags.sentinel                                       │
  │  └── max-instance-size.sentinel                                  │
  └──────────────────────────────────────────────────────────────────┘

  STATE KEY NAMING CONVENTIONS:
  ┌──────────────────────────────────────────────────────────────────┐
  │  Flat:      {env}/terraform.tfstate                              │
  │  By layer:  {env}/{layer}/terraform.tfstate                      │
  │  Full path: {cloud}/{region}/{account}/{env}/{layer}/tfstate     │
  │                                                                  │
  │  Examples:                                                       │
  │  prod/terraform.tfstate                                          │
  │  prod/networking/terraform.tfstate                               │
  │  aws/us-east-1/123456789/prod/compute/terraform.tfstate          │
  └──────────────────────────────────────────────────────────────────┘
```

---

## 8. Remote State with Cross-Stack References

How separate Terraform stacks share data through remote state data sources.

```
┌─────────────────────────────────────────────────────────────────────┐
│             CROSS-STACK REMOTE STATE REFERENCES                     │
└─────────────────────────────────────────────────────────────────────┘

  Stack Dependency Order (apply in this sequence):

  ┌─────────────────┐
  │  STACK 1        │  No dependencies
  │  networking     │  Outputs: vpc_id, subnet_ids, sg_ids
  └────────┬────────┘
           │ terraform_remote_state data source
           ▼
  ┌─────────────────┐
  │  STACK 2        │  Reads Stack 1
  │  security       │  Outputs: iam_roles, kms_keys, cert_arns
  └────────┬────────┘
           │
           ▼
  ┌─────────────────┐
  │  STACK 3        │  Reads Stack 1 + Stack 2
  │  compute        │  Outputs: ec2_ips, asg_names, alb_arns
  └────────┬────────┘
           │
           ▼
  ┌─────────────────┐
  │  STACK 4        │  Reads Stack 1 + 2 + 3
  │  application    │  DNS records, app config
  └─────────────────┘

  Code Pattern:
  ┌──────────────────────────────────────────────────────────────────┐
  │  # Stack 1 — outputs.tf                                          │
  │  output "vpc_id" { value = aws_vpc.main.id }                     │
  │                                                                  │
  │  # Stack 3 — reads Stack 1                                       │
  │  data "terraform_remote_state" "networking" {                    │
  │    backend = "s3"                                                │
  │    config = {                                                    │
  │      bucket = "company-terraform-state"                          │
  │      key    = "prod/networking/terraform.tfstate"                │
  │      region = "us-east-1"                                        │
  │    }                                                             │
  │  }                                                               │
  │                                                                  │
  │  resource "aws_instance" "web" {                                 │
  │    subnet_id = data.terraform_remote_state.networking            │
  │               .outputs.public_subnet_ids[0]                     │
  │  }                                                               │
  └──────────────────────────────────────────────────────────────────┘

  S3 State File Layout:
  s3://company-tf-state/prod/networking/terraform.tfstate   (Stack 1)
  s3://company-tf-state/prod/security/terraform.tfstate    (Stack 2)
  s3://company-tf-state/prod/compute/terraform.tfstate     (Stack 3)
  s3://company-tf-state/prod/application/terraform.tfstate (Stack 4)
```

---

## 9. Workspace Strategy Patterns

Two competing approaches to environment isolation.

```
┌─────────────────────────────────────────────────────────────────────┐
│                   WORKSPACE STRATEGY PATTERNS                       │
└─────────────────────────────────────────────────────────────────────┘

  PATTERN A: Terraform Workspaces (Lighter Weight)
  ┌──────────────────────────────────────────────────────────────────┐
  │  Single codebase → Multiple workspaces                           │
  │                                                                  │
  │  project/                                                        │
  │  ├── main.tf      ← uses terraform.workspace for branching       │
  │  └── *.tfvars     ← dev / staging / prod variable files          │
  │                                                                  │
  │  State files:                                                    │
  │  s3://state/env:/dev/terraform.tfstate                           │
  │  s3://state/env:/staging/terraform.tfstate                       │
  │  s3://state/env:/prod/terraform.tfstate                          │
  │                                                                  │
  │  locals {                                                        │
  │    env = terraform.workspace                                     │
  │    instance_type = {                                             │
  │      dev     = "t3.micro"                                        │
  │      staging = "t3.small"                                        │
  │      prod    = "t3.large"                                        │
  │    }[local.env]                                                  │
  │  }                                                               │
  │                                                                  │
  │  ✅ Simple setup    ⚠ Shared code = shared blast radius          │
  └──────────────────────────────────────────────────────────────────┘

  PATTERN B: Directory-Per-Environment (Production Recommended)
  ┌──────────────────────────────────────────────────────────────────┐
  │  environments/                                                   │
  │  ├── dev/                                                        │
  │  │   ├── main.tf          (calls shared modules)                 │
  │  │   ├── backend.tf       (dev-specific state key)               │
  │  │   └── terraform.tfvars (dev-specific variable values)         │
  │  ├── staging/                                                    │
  │  │   ├── main.tf / backend.tf / terraform.tfvars                 │
  │  └── prod/                                                       │
  │      ├── main.tf / backend.tf / terraform.tfvars                 │
  │  modules/                 (shared across all envs)               │
  │  ├── networking/                                                 │
  │  ├── compute/                                                    │
  │  └── database/                                                   │
  │                                                                  │
  │  ✅ Full isolation    ✅ Per-env team access control             │
  │  ⚠ More files (solved by Terragrunt — see Diagram 16)           │
  └──────────────────────────────────────────────────────────────────┘

  COMPARISON:
  ┌───────────────────┬──────────────────────┬──────────────────────┐
  │ Criterion         │ Workspaces           │ Directories          │
  ├───────────────────┼──────────────────────┼──────────────────────┤
  │ Code isolation    │ None (shared)        │ Full                 │
  │ State isolation   │ Partial              │ Full (separate keys) │
  │ Team access ctrl  │ Hard to restrict     │ Per-directory IAM    │
  │ Version control   │ All envs same ver.   │ Can differ per env   │
  │ Blast radius      │ High                 │ Low                  │
  │ Recommended for   │ Prototypes, learning │ Production           │
  └───────────────────┴──────────────────────┴──────────────────────┘
```

---

## 10. CI/CD Pipeline Architecture

End-to-end view of Terraform in CI/CD.

```
┌─────────────────────────────────────────────────────────────────────┐
│                   CI/CD PIPELINE ARCHITECTURE                       │
└─────────────────────────────────────────────────────────────────────┘

  CI PIPELINE (on Pull Request):
  ┌──────────────────────────────────────────────────────────────────┐
  │                                                                  │
  │  PR Opened/Updated                                               │
  │        │                                                         │
  │        ▼                                                         │
  │  terraform fmt -check   FAIL → Block merge, notify dev           │
  │        │ PASS                                                    │
  │        ▼                                                         │
  │  terraform validate     FAIL → Block merge                       │
  │        │ PASS                                                    │
  │        ▼                                                         │
  │  tfsec / checkov        HIGH → Block merge / LOW → Warning only  │
  │        │ PASS                                                    │
  │        ▼                                                         │
  │  terraform plan         Output posted as PR comment              │
  │        │                                                         │
  │        ▼                                                         │
  │  Sentinel / OPA policy  Violations → Block merge                 │
  │        │ PASS                                                    │
  │        ▼                                                         │
  │  Awaiting human code review + plan review                        │
  └──────────────────────────────────────────────────────────────────┘

  CD PIPELINE (on Merge to main):
  ┌──────────────────────────────────────────────────────────────────┐
  │                                                                  │
  │  Merge to main                                                   │
  │        │                                                         │
  │        ▼                                                         │
  │  terraform init     (acquires state lock, downloads providers)   │
  │        │                                                         │
  │        ▼                                                         │
  │  terraform apply -auto-approve                                   │
  │  → Calls cloud APIs                                              │
  │  → Resources created/updated/destroyed                           │
  │  → State file updated, lock released                             │
  │        │                                                         │
  │        ▼                                                         │
  │  Post-apply tests   (smoke tests, Slack notification)            │
  └──────────────────────────────────────────────────────────────────┘

  Multi-Environment Promotion:
  ┌──────────────────────────────────────────────────────────────────┐
  │                                                                  │
  │  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐        │
  │  │     dev     │────►│   staging   │────►│    prod     │        │
  │  │  auto-apply │     │  auto-apply │     │  MANUAL     │        │
  │  │  on merge   │     │  after dev  │     │  APPROVAL   │        │
  │  └─────────────┘     └─────────────┘     │  required   │        │
  │                                          └─────────────┘        │
  │  Prod: requires 2 senior engineers to approve before apply       │
  └──────────────────────────────────────────────────────────────────┘

  Drift Detection (Scheduled):
  ┌──────────────────────────────────────────────────────────────────┐
  │  Every 6 hours:                                                  │
  │  terraform plan -detailed-exitcode                               │
  │  Exit 0 → no drift, OK                                           │
  │  Exit 2 → drift detected → Slack alert + GitHub Issue            │
  └──────────────────────────────────────────────────────────────────┘
```

---

## 11. Module Registry Architecture

How internal module registries work in large enterprises.

```
┌─────────────────────────────────────────────────────────────────────┐
│                   MODULE REGISTRY ARCHITECTURE                      │
└─────────────────────────────────────────────────────────────────────┘

  PUBLIC REGISTRY (registry.terraform.io):
  module "vpc" {
    source  = "terraform-aws-modules/vpc/aws"
    version = "5.1.0"   ← Always pin in production
  }

  PRIVATE MODULE REGISTRY (Enterprise):
  ┌──────────────────────────────────────────────────────────────────┐
  │                                                                  │
  │  Platform Engineering Team                                       │
  │  1. Write module → 2. Terratest → 3. Code review → 4. Tag v1.0  │
  │                                                                  │
  │  Source Options:                                                 │
  │                                                                  │
  │  Git (simplest):                                                 │
  │  source = "git::https://github.com/acme/tf-modules.git           │
  │           //modules/vpc?ref=v2.1.0"                              │
  │                                                                  │
  │  HCP Terraform Private Registry:                                 │
  │  source  = "app.terraform.io/acme/vpc/aws"                       │
  │  version = "2.1.0"                                               │
  │                                                                  │
  │  S3 Archive:                                                     │
  │  source = "s3::https://s3.amazonaws.com/                         │
  │            company-modules/vpc/v2.1.0/module.tar.gz"            │
  └──────────────────────────────────────────────────────────────────┘

  VERSION CONSTRAINTS:
  ┌──────────────────────────────────────────────────────────────────┐
  │  version = "2.1.0"   → Exact pin (most stable)                   │
  │  version = "~> 2.1"  → Allow 2.1.x only (recommended)           │
  │  version = ">= 2.0"  → Any 2.x or higher (risky)                │
  │                                                                  │
  │  MAJOR bump = breaking change (var renamed/removed)              │
  │  MINOR bump = new features, backward compatible                  │
  │  PATCH bump = bug fixes only                                     │
  └──────────────────────────────────────────────────────────────────┘
```

---

## 12. Hub-and-Spoke Network (Multi-Region)

Transit Gateway connecting multiple VPCs across regions.

```
┌─────────────────────────────────────────────────────────────────────┐
│              HUB-AND-SPOKE NETWORK — AWS TRANSIT GATEWAY            │
└─────────────────────────────────────────────────────────────────────┘

  REGION 1: us-east-1 (Primary Hub)
  ┌──────────────────────────────────────────────────────────────────┐
  │                                                                  │
  │  Shared Services VPC (10.0.0.0/16)                               │
  │  • VPN Gateway  • DNS Resolver  • Bastion Host                   │
  │        │                                                         │
  │        ▼                                                         │
  │  ┌──────────────────────────────────────────────────────────┐   │
  │  │   AWS Transit Gateway (Hub)                              │   │
  │  │   Route Tables:                                           │   │
  │  │   10.10.0.0/16 → VPC A attachment                        │   │
  │  │   10.20.0.0/16 → VPC B attachment                        │   │
  │  │   10.30.0.0/16 → VPC C attachment                        │   │
  │  │   0.0.0.0/0    → Shared Services (for egress)            │   │
  │  └──────────────────────┬───────────────────────────────────┘   │
  │            ┌────────────┼────────────┐                           │
  │            ▼            ▼            ▼                           │
  │       ┌─────────┐  ┌─────────┐  ┌─────────┐                    │
  │       │  VPC A  │  │  VPC B  │  │  VPC C  │                    │
  │       │  (Dev)  │  │(Staging)│  │  (Prod) │                    │
  │       │10.10/16 │  │10.20/16 │  │10.30/16 │                    │
  │       └─────────┘  └─────────┘  └─────────┘                    │
  └──────────────────────────────────────────────────────────────────┘
                                   │
                    Cross-Region TGW Peering
                                   │
  REGION 2: eu-west-1 (Secondary Hub)
  ┌──────────────────────────────────────────────────────────────────┐
  │  AWS Transit Gateway (EU Hub)                                    │
  │       ├── VPC EU-A   ├── VPC EU-B   └── VPC EU-C                │
  └──────────────────────────────────────────────────────────────────┘

  Terraform modules:
  modules/transit-gateway/     ← TGW + route tables + attachments
  modules/tgw-peering/         ← Cross-region peering
  modules/spoke-vpc/           ← Standardized spoke VPC
  modules/shared-services-vpc/ ← Hub VPC with shared resources
```

---

## 13. Disaster Recovery Architecture

Active/Active and Active/Passive DR patterns.

```
┌─────────────────────────────────────────────────────────────────────┐
│                DISASTER RECOVERY ARCHITECTURE                       │
└─────────────────────────────────────────────────────────────────────┘

  ACTIVE / ACTIVE (Zero RTO):
  ┌──────────────────────────────────────────────────────────────────┐
  │  Route 53 Health Checks + Latency Routing                        │
  │                 │                                                │
  │     ┌───────────┴────────────┐                                   │
  │     │                        │                                   │
  │     ▼                        ▼                                   │
  │  ┌──────────────┐        ┌──────────────┐                        │
  │  │ PRIMARY      │        │ SECONDARY    │                        │
  │  │ us-east-1    │        │ us-west-2    │                        │
  │  │              │        │              │                        │
  │  │ ALB + ASG    │        │ ALB + ASG    │                        │
  │  │ RDS Primary  │───────►│ RDS Replica  │  (async replication)  │
  │  │ ElastiCache  │        │ ElastiCache  │                        │
  │  │ DynamoDB     │◄──────►│ DynamoDB     │  (Global Tables sync) │
  │  │ S3           │◄──────►│ S3           │  (CRR)                │
  │  └──────────────┘        └──────────────┘                        │
  │  RTO: ~0 sec   RPO: seconds (async DB lag)                       │
  └──────────────────────────────────────────────────────────────────┘

  ACTIVE / PASSIVE (Low Cost):
  ┌──────────────────────────────────────────────────────────────────┐
  │  ┌──────────────┐                    ┌───────────────────────┐   │
  │  │ PRIMARY      │  DB Replication    │ STANDBY (WARM)        │   │
  │  │ us-east-1    │───────────────────►│ us-west-2             │   │
  │  │ (Full scale) │                    │ ASG: min=0 max=10      │   │
  │  │ 10 servers   │                    │ RDS Read Replica       │   │
  │  └──────────────┘                    └───────────────────────┘   │
  │                                                                  │
  │  Failover steps:                                                 │
  │  1. Promote RDS replica → Primary                                │
  │  2. Scale ASG: 0 → 10                                            │
  │  3. Update Route53 health check → standby ALB                    │
  │  RTO: 15-30 min (manual) / 5 min (automated)                    │
  └──────────────────────────────────────────────────────────────────┘

  Terraform DR Runbook:
  # dr.tfvars — activate during failover
  primary_region = "us-west-2"  # Failover target
  rds_is_replica = false        # Promote to primary
  asg_min_size   = 2            # Scale up
  enable_alb     = true         # Create ALB in standby
  # Run: terraform apply -var-file=dr.tfvars
```

---

## 14. Zero-Trust Multi-Cloud Networking

Secure connectivity between clouds without public internet exposure.

```
┌─────────────────────────────────────────────────────────────────────┐
│              ZERO-TRUST MULTI-CLOUD NETWORKING                      │
└─────────────────────────────────────────────────────────────────────┘

  CONNECTIVITY OPTIONS:

  Option 1: Cloud VPN (IPSec over public internet, encrypted)
  ┌────────┐  VPN Tunnel (IPSec)  ┌────────┐
  │  AWS   │◄────────────────────►│ Azure  │
  │  VPC   │                      │  VNet  │
  └────────┘                      └────────┘

  Option 2: Private Backbone (no public internet)
  ┌────────┐  Direct Connect  ┌──────────┐  ExpressRoute  ┌─────┐
  │  AWS   │◄────────────────►│  Equinix │◄──────────────►│Azure│
  │  VPC   │                  │  (Colo)  │                │VNet │
  └────────┘                  └──────────┘                └─────┘

  Option 3: Service Mesh (mTLS everywhere)
  ┌──────────────────────────────────────────────────────────────┐
  │  AWS EKS — Service A [Envoy] ◄──mTLS──► GCP GKE — Service B │
  │  Consul Control Plane manages certs + routing across clouds  │
  └──────────────────────────────────────────────────────────────┘

  FULL ZERO-TRUST ARCHITECTURE:
  ┌──────────────────────────────────────────────────────────────────┐
  │  Internet                                                        │
  │       │                                                          │
  │       ▼                                                          │
  │  Zero Trust Edge (Cloudflare / Zscaler)                          │
  │  • Identity-aware proxy   • mTLS everywhere                      │
  │  • No VPN required for users                                     │
  │       │                                                          │
  │       ├──────────────────┬──────────────────┐                   │
  │       ▼                  ▼                  ▼                   │
  │  ┌──────────┐      ┌──────────┐      ┌──────────┐              │
  │  │  AWS     │      │  Azure   │      │  GCP     │              │
  │  │  IAM     │      │  AAD     │      │  IAM     │              │
  │  │  Private │      │  Private │      │  Private │              │
  │  │ endpoints│      │ endpoints│      │ endpoints│              │
  │  │ only     │      │ only     │      │ only     │              │
  │  └──────────┘      └──────────┘      └──────────┘              │
  │                                                                  │
  │  Terraform manages:                                              │
  │  • Private endpoints (no public IPs on services)                 │
  │  • Security groups: deny-by-default                              │
  │  • IAM: least privilege for all roles                            │
  │  • ACM/Key Vault/Cert Manager: automated cert rotation           │
  └──────────────────────────────────────────────────────────────────┘
```

---

## 15. GitOps Terraform Architecture

Terraform as part of a full GitOps workflow with automated drift detection.

```
┌─────────────────────────────────────────────────────────────────────┐
│                   GITOPS TERRAFORM ARCHITECTURE                     │
│  CORE PRINCIPLE: Git is the single source of truth.                 │
│  Every infrastructure change must go through Git + PR review.       │
└─────────────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────┐
  │  Git Repository                                                  │
  │  main branch = Production desired state                          │
  │  All changes via PR → Review → Merge                             │
  └────────────────────┬─────────────────────────────────────────────┘
                       │ Git event triggers CI/CD
                       ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │  GitHub Actions / GitLab CI                                      │
  │  On PR:    plan → post as comment → await approval               │
  │  On merge: apply → notify → update CMDB                          │
  └────────────────────┬─────────────────────────────────────────────┘
                       │                              ▲
                       ▼                              │
  ┌──────────────────────────┐   ┌─────────────────────────────────┐
  │  Cloud Infrastructure    │   │  Drift Detection (Scheduled)    │
  │  (Real AWS/Azure/GCP)    │──►│  Every 6 hours:                 │
  │                          │   │  terraform plan -detailed-exit  │
  │  Manual console changes  │   │                                 │
  │  = DRIFT (bad!)          │   │  Exit 2 = drift → alert Slack   │
  └──────────────────────────┘   │  → create GitHub Issue          │
                                 └─────────────────────────────────┘

  Branch Strategy:
  feature/add-rds ──────────────────────────────────────────►┐
  feature/update-sg ────────────────────────────────────────►│
                                                             ▼
  main ────────────┬──────────┬───────────┬──────────────────────►
                   │          │           │
                 apply      apply       apply
                (v1.0)     (v1.1)      (v1.2)

  Each merge = one atomic infrastructure change, reviewed and logged.
```

---

## 16. Terragrunt Multi-Environment Architecture

DRY environment management eliminating backend/provider boilerplate.

```
┌─────────────────────────────────────────────────────────────────────┐
│              TERRAGRUNT MULTI-ENVIRONMENT ARCHITECTURE              │
└─────────────────────────────────────────────────────────────────────┘

  FOLDER STRUCTURE:
  infrastructure/
  ├── terragrunt.hcl          ← ROOT: Backend + Provider (defined ONCE)
  ├── _envcommon/
  │   ├── networking.hcl      ← Shared networking defaults
  │   ├── compute.hcl         ← Shared compute defaults
  │   └── database.hcl        ← Shared database defaults
  └── environments/
      ├── dev/
      │   ├── env.hcl         ← environment = "dev"
      │   ├── networking/terragrunt.hcl  (include root + envcommon)
      │   ├── compute/terragrunt.hcl     (dependency: networking)
      │   └── database/terragrunt.hcl    (dependency: networking)
      ├── staging/
      │   ├── env.hcl / networking / compute / database
      └── prod/
          ├── env.hcl / networking / compute / database

  DEPENDENCY GRAPH (auto-resolved by Terragrunt run-all):
  ┌──────────────────────────────────────────────────────────────────┐
  │                                                                  │
  │  networking ──────────────────────────────┐                      │
  │       │                                   │                      │
  │       ├──────────────┐                    │                      │
  │       │              │                    │                      │
  │       ▼              ▼                    ▼                      │
  │   compute         database           monitoring                  │
  │       │              │                                           │
  │       └──────────────┘                                           │
  │               │                                                  │
  │               ▼                                                  │
  │          application                                             │
  │                                                                  │
  │  terragrunt run-all apply   → applies in correct order           │
  │  terragrunt run-all destroy → destroys in reverse order          │
  └──────────────────────────────────────────────────────────────────┘

  STATE ISOLATION (auto key from folder path):
  s3://tf-state/environments/dev/networking/terraform.tfstate
  s3://tf-state/environments/dev/compute/terraform.tfstate
  s3://tf-state/environments/prod/networking/terraform.tfstate
  key = "${path_relative_to_include()}/terraform.tfstate"
```

---

## 17. Landing Zone Architecture

Enterprise foundation deployed before any applications.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    LANDING ZONE ARCHITECTURE                        │
└─────────────────────────────────────────────────────────────────────┘

  LAYER 0: MANAGEMENT FOUNDATION
  ┌──────────────────────────────────────────────────────────────────┐
  │  AWS Organizations / Azure Management Groups                     │
  │  SSO / SAML Identity Provider                                    │
  │  CloudTrail / Audit Logging (all accounts)                       │
  │  SCPs / Azure Policy (guardrails enforced top-down)              │
  │  Budget Alerts + Cost Allocation Tags                            │
  │  Deployed once. Never touched by application teams.              │
  └──────────────────────────────────────────────────────────────────┘
                              │
  LAYER 1: NETWORK FOUNDATION
  ┌──────────────────────────────────────────────────────────────────┐
  │  Network Account / Connectivity Subscription                     │
  │  Transit Gateway (us-east-1 + eu-west-1)                         │
  │  TGW Cross-Region Peering                                        │
  │  Route 53 Resolver (centralized DNS)                             │
  │  Private Hosted Zones                                            │
  └──────────────────────────────────────────────────────────────────┘
                              │
  LAYER 2: SECURITY FOUNDATION
  ┌──────────────────────────────────────────────────────────────────┐
  │  Security Account                                                │
  │  IAM Roles (cross-account assume role)                           │
  │  KMS Keys (cross-account encryption)                             │
  │  Secrets Manager                                                 │
  │  SecurityHub aggregated (all accounts)                           │
  │  Macie / DLP                                                     │
  └──────────────────────────────────────────────────────────────────┘
                              │
  LAYER 3: SHARED SERVICES
  ┌──────────────────────────────────────────────────────────────────┐
  │  Shared Services Account                                         │
  │  Container Registry (ECR / ACR)                                  │
  │  Artifact Repository (CodeArtifact / Nexus)                      │
  │  CI/CD Platform (Jenkins / GitHub Actions runners)               │
  │  Monitoring (Grafana / Datadog)                                  │
  │  Bastion / SSM Session Manager                                   │
  └──────────────────────────────────────────────────────────────────┘
                              │
  LAYER 4: WORKLOAD ACCOUNTS (Many — one per app/env)
  ┌──────────────────────────────────────────────────────────────────┐
  │  App A Dev    App A Prod    App B Dev    App B Prod               │
  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐            │
  │  │App VPC  │  │App VPC  │  │App VPC  │  │App VPC  │            │
  │  │ → TGW   │  │ → TGW   │  │ → TGW   │  │ → TGW   │            │
  │  │EKS / EC2│  │EKS / EC2│  │EKS / EC2│  │EKS / EC2│            │
  │  │   RDS   │  │   RDS   │  │   RDS   │  │   RDS   │            │
  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘            │
  └──────────────────────────────────────────────────────────────────┘

  Terraform Deployment Order:
  Layer 0 → Layer 1 → Layer 2 → Layer 3 → Layer 4 (parallel)
```

---

## 18. Service Mesh Multi-Region

Kubernetes workloads across regions connected by a service mesh.

```
┌─────────────────────────────────────────────────────────────────────┐
│                SERVICE MESH MULTI-REGION ARCHITECTURE               │
└─────────────────────────────────────────────────────────────────────┘

  ┌────────────────────────────────────────────────────────────────┐
  │                    GLOBAL CONTROL PLANE                        │
  │  Consul / Istio Multi-cluster                                  │
  │  • Certificate Authority (mTLS certs to all pods)             │
  │  • Service Registry (every service in all clusters)           │
  │  • Traffic Policy (canary, retries, circuit breakers)         │
  └────────────────────────────────────────────────────────────────┘
                            │
              ┌─────────────┼──────────────┐
              ▼             ▼              ▼
  ┌───────────────┐  ┌────────────┐  ┌───────────────┐
  │  EKS          │  │  GKE       │  │  AKS          │
  │  us-east-1    │  │  eu-west-1 │  │  ap-sea-1     │
  │               │  │            │  │               │
  │ Pod A [Envoy]◄├──┼──mTLS─────►├──┼►Pod C [Envoy] │
  │ Pod D [Envoy] │  │ Pod B[Env] │  │ Pod F [Envoy] │
  └───────────────┘  └────────────┘  └───────────────┘
         │                │                  │
         └────────────────┼──────────────────┘
                          │
               Cross-cluster via Gateway API
               Mesh Gateways (Consul)
               Istio East-West Gateway

  Terraform manages:
  module "eks_cluster"      ← EKS + node groups + OIDC provider
  helm_release "istio_base" ← Mesh control plane via Helm
  kubectl_manifest "peer_auth" ← Enforce mTLS (PeerAuthentication)
```

---

## 19. Data Platform Multi-Cloud

Modern data platform spanning multiple clouds.

```
┌─────────────────────────────────────────────────────────────────────┐
│                  DATA PLATFORM MULTI-CLOUD                          │
└─────────────────────────────────────────────────────────────────────┘

  INGESTION (Sources)
  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐
  │  AWS RDS │  │ Azure SQL│  │  SaaS    │  │  IoT Devices │
  │  (Prod)  │  │  (Prod)  │  │ (Salesf.)│  │  (MQTT)      │
  └────┬─────┘  └────┬─────┘  └────┬─────┘  └──────┬───────┘
       └─────────────┴─────────────┴────────────────┘
                             │
  STREAM LAYER               ▼
  ┌──────────────────────────────────────────────────────────────┐
  │  Confluent Kafka (multi-cloud) / Kinesis / Event Hubs        │
  │  Topics: user-events, transactions, clickstream, sensors      │
  └──────────────────────────────────────────────────────────────┘
                             │
  PROCESSING LAYER           ▼
  ┌───────────────────┐   ┌──────────────────────────────────┐
  │ AWS EMR / Glue    │   │ GCP Dataflow / Azure Data Factory │
  │ (Batch ETL)       │   │ (Streaming ETL)                  │
  └───────────────────┘   └──────────────────────────────────┘
                             │
  STORAGE LAYER (Data Lake)  ▼
  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐
  │  AWS S3     │  │  Azure ADLS │  │  GCP GCS        │
  │  (primary)  │  │  Gen2 (EU)  │  │  (analytics)    │
  │  Raw/       │  │  Raw/       │  │  Raw/           │
  │  Processed/ │  │  Processed/ │  │  Processed/     │
  │  Curated/   │  │  Curated/   │  │  Curated/       │
  └─────────────┘  └─────────────┘  └─────────────────┘
        Delta Lake / Apache Iceberg / Apache Hudi
                             │
  SERVING LAYER              ▼
  ┌────────────────┐  ┌──────────────┐  ┌──────────────────────┐
  │  Snowflake     │  │  BigQuery    │  │  Databricks          │
  │  (multi-cloud  │  │  (GCP)       │  │  (multi-cloud)       │
  │   warehouse)   │  │              │  │                      │
  └────────────────┘  └──────────────┘  └──────────────────────┘

  Terraform providers for data platform:
  hashicorp/aws         → S3, Glue, EMR, Kinesis, Redshift
  hashicorp/azurerm     → ADLS, ADF, Synapse, Event Hubs
  hashicorp/google      → GCS, Dataflow, BigQuery, Pub/Sub
  confluentinc/confluent → Kafka clusters, topics, ACLs
  snowflake-labs/snowflake → Warehouses, databases, grants
  databricks/databricks → Clusters, jobs, notebooks, Unity Catalog
```

---

## 20. Security & Compliance Architecture

Complete security posture across clouds managed by Terraform.

```
┌─────────────────────────────────────────────────────────────────────┐
│              SECURITY & COMPLIANCE ARCHITECTURE                     │
└─────────────────────────────────────────────────────────────────────┘

  PREVENTION (Shift Left):
  ┌──────────────────────────────────────────────────────────────────┐
  │  Developer Workstation (pre-commit hooks):                       │
  │  • terraform fmt -check                                          │
  │  • terraform validate                                            │
  │  • tfsec (local scan)                                            │
  │  • detect-secrets (block credentials in code)                    │
  │                                                                  │
  │  Pull Request Gates:                                             │
  │  • Checkov / tfsec        → blocks HIGH severity findings        │
  │  • Infracost              → blocks if > $X/month estimate        │
  │  • Sentinel / OPA policy  → blocks non-compliant configurations  │
  └──────────────────────────────────────────────────────────────────┘

  DETECTION (Runtime):
  ┌──────────────────────────────────────────────────────────────────┐
  │  AWS GuardDuty + SecurityHub + CloudTrail                        │
  │  Azure Defender + Sentinel + Monitor                             │
  │  GCP Security Command Center + Chronicle + Audit Logs            │
  │                   │                                              │
  │                   ▼                                              │
  │         SIEM (Splunk / Elastic / Datadog)                        │
  └──────────────────────────────────────────────────────────────────┘

  RESPONSE (Automated Remediation):
  ┌──────────────────────────────────────────────────────────────────┐
  │  Finding: public S3 bucket created manually                      │
  │       │                                                          │
  │       ▼                                                          │
  │  AWS Config Rule → Lambda auto-remediation:                      │
  │  1. Block public access on bucket                                │
  │  2. Alert Slack #security channel                                │
  │  3. Open Jira ticket                                             │
  │  4. Tag resource "non-compliant"                                 │
  │  5. Terraform drift detection picks it up next scheduled run     │
  └──────────────────────────────────────────────────────────────────┘

  SECRETS MANAGEMENT:
  ┌──────────────────────────────────────────────────────────────────┐
  │  HashiCorp Vault / AWS Secrets Manager / Azure Key Vault         │
  │                                                                  │
  │  data "vault_generic_secret" "db" {                              │
  │    path = "secret/prod/database"                                 │
  │  }                                                               │
  │  resource "aws_db_instance" "main" {                             │
  │    password = data.vault_generic_secret.db.data["password"]     │
  │  }                                                               │
  │                                                                  │
  │  Secrets NEVER in:                                               │
  │  • .tf files           → In Git, visible forever                 │
  │  • terraform.tfvars    → Add to .gitignore                       │
  │  • sensitive = false   → State file stores in plaintext          │
  └──────────────────────────────────────────────────────────────────┘

  COMPLIANCE AS CODE (Sentinel example):
  ┌──────────────────────────────────────────────────────────────────┐
  │  # All S3 buckets must have encryption enabled                   │
  │  import "tfplan/v2" as tfplan                                    │
  │                                                                  │
  │  buckets = filter tfplan.resource_changes as _, rc {            │
  │    rc.type is "aws_s3_bucket" and                                │
  │    rc.change.actions is not ["delete"]                           │
  │  }                                                               │
  │  violations = filter buckets as _, b {                          │
  │    b.change.after.server_side_encryption_configuration is empty  │
  │  }                                                               │
  │  main = rule { length(violations) is 0 }                        │
  └──────────────────────────────────────────────────────────────────┘
```

---

## 21. Pattern Selection Guide

```
┌──────────────────────────────────────────────────────────────────────┐
│                    PATTERN SELECTION GUIDE                           │
├──────────────────────────┬───────────────────────────────────────────┤
│  Your Situation          │  Recommended Diagram(s)                   │
├──────────────────────────┼───────────────────────────────────────────┤
│  Learning / Solo dev     │  #2 Single region baseline                │
│  Small team, one cloud   │  #2 + #10 CI/CD pipeline                  │
│  Multi-env, one cloud    │  #9B Dir-per-env + #16 Terragrunt         │
│  Enterprise AWS          │  #4 Multi-account + #17 Landing Zone      │
│  Multi-region HA         │  #3 Multi-region + #13 Active/Active DR   │
│  Multi-cloud             │  #5 Multi-cloud + #7 State mgmt           │
│  Multi-cloud + region    │  #6 Full complexity                       │
│  K8s everywhere          │  #18 Service mesh                         │
│  Data platform           │  #19 Data multi-cloud                     │
│  Regulated industry      │  #20 Security + compliance                │
│  GitOps team             │  #15 GitOps + #10 CI/CD                   │
│  Network isolation       │  #12 Hub-and-spoke + #14 Zero-trust       │
└──────────────────────────┴───────────────────────────────────────────┘

  COMPLEXITY vs COST:
  Low ◄─────────────────────────────────────────────────────► High
  [1-region] → [multi-region] → [multi-account] → [multi-cloud]

  OPERATIONAL OVERHEAD:
  Low ◄─────────────────────────────────────────────────────► High
  [workspaces] → [directories] → [terragrunt] → [landing zone]

  TEAM SIZE:
  1–5   engineers: Single cloud, dir-per-env, S3 backend
  5–20  engineers: Multi-account, Terragrunt, HCP Terraform
  20+   engineers: Landing zone, multi-cloud, private registry, Sentinel

  DEPLOYMENT ORDER (always applies):
  Layer 0 (Org/Mgmt)
    → Layer 1 (Networking: VPC, TGW, DNS)
      → Layer 2 (Security: IAM, KMS, Secrets)
        → Layer 3 (Shared Services: ECR, CI/CD, Monitoring)
          → Layer 4 (Workloads: App VPC, EKS, RDS)

  KEY TERRAFORM COMMANDS BY PATTERN:
  ┌───────────────────────────────────────────────────────────────┐
  │  Multi-region with aliases:                                   │
  │  terraform apply -target=module.us_stack                      │
  │  terraform apply -target=module.eu_stack                      │
  │                                                               │
  │  Cross-account:                                               │
  │  AWS_PROFILE=prod terraform plan                              │
  │                                                               │
  │  Terragrunt multi-env:                                        │
  │  terragrunt run-all plan --terragrunt-working-dir environments/│
  │                                                               │
  │  Drift detection:                                             │
  │  terraform plan -detailed-exitcode -refresh=true              │
  │                                                               │
  │  State migration between backends:                            │
  │  terraform init -migrate-state                                │
  │                                                               │
  │  Import existing resource:                                    │
  │  terraform import aws_s3_bucket.legacy bucket-name            │
  └───────────────────────────────────────────────────────────────┘
```

---

*All 20 system design diagrams use ASCII art for universal rendering in any Markdown viewer, terminal, IDE preview, Confluence, GitHub, GitLab, or documentation platform. No rendering plugins required.*
