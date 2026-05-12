# Phase 6: Infrastructure as Code (IaC)

---

## 6.1 What is Infrastructure as Code?

**Infrastructure as Code (IaC)** is the practice of managing and provisioning computing infrastructure through machine-readable configuration files rather than through manual processes or interactive configuration tools.

### Why IaC Exists

**Before IaC — The Dark Ages:**
- Servers configured manually via SSH → "snowflake servers" (unique, irreproducible)
- No version history for infrastructure changes
- "It works on my machine" extended to "it works in staging but not prod"
- Disaster recovery = rebuild from memory + tribal knowledge
- Scaling = ops team manually spinning up servers for hours/days

**Problems IaC Solves:**

| Problem | IaC Solution |
|---------|-------------|
| Manual configuration drift | Declarative state enforced on every run |
| No audit trail | Git history of every infrastructure change |
| Slow provisioning | Automated, repeatable in minutes |
| Environment inconsistency | Same code deploys identical dev/staging/prod |
| Knowledge silos | Code is documentation |
| Disaster recovery | Rebuild entire infra from code in minutes |

### IaC Approaches

```
IMPERATIVE (HOW)                    DECLARATIVE (WHAT)
─────────────────                   ──────────────────
"Create a VM"                       "I want a VM to exist"
"Install nginx on it"               "I want nginx installed"
"Open port 80"                      "I want port 80 open"

Ansible (procedural mode)           Terraform, CloudFormation
Scripts (bash, Python)              Pulumi (can be declarative)

You define the steps                You define the end state
Order matters                       Engine figures out order
Idempotency is YOUR problem         Engine handles idempotency
```

---

## 6.2 Terraform — Deep Internals

### What is Terraform?

Terraform is an open-source IaC tool by HashiCorp that lets you define cloud and on-premises infrastructure in human-readable HCL (HashiCorp Configuration Language) and manages the full lifecycle of that infrastructure.

**Key Differentiator:** Terraform is **cloud-agnostic** — one tool for AWS, Azure, GCP, Kubernetes, Datadog, GitHub, etc.

### Terraform Internal Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        TERRAFORM CLI                             │
│                                                                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───────────────┐  │
│  │  Parser   │  │  Loader  │  │ Evaluator│  │ Plan/Apply    │  │
│  │  (HCL)   │  │ (modules)│  │ (exprs)  │  │ Engine        │  │
│  └──────────┘  └──────────┘  └──────────┘  └───────────────┘  │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    CORE ENGINE                            │   │
│  │  Resource Graph Builder → Dependency Resolver            │   │
│  │  Diff Engine (desired state vs current state)            │   │
│  │  Walk Algorithm (parallel execution of independent nodes) │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                  PROVIDER PLUGIN SYSTEM                    │  │
│  │  Provider = gRPC server implementing ResourceCRUD          │  │
│  │  AWS Provider │ Azure Provider │ GCP Provider │ K8s ...    │  │
│  └────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                    gRPC (tfplugin protocol)
                              │
              ┌───────────────┴────────────────┐
              │        PROVIDER BINARY          │
              │   (separate process, auto-DL)   │
              │  Translates HCL → API calls     │
              │  aws_instance → EC2 Create API  │
              └───────────────┬────────────────┘
                              │
                    HTTPS/REST/SDK calls
                              │
              ┌───────────────┴────────────────┐
              │         CLOUD PROVIDER API      │
              │  AWS API / Azure ARM / GCP API  │
              └────────────────────────────────┘
```

### HCL Language Deep Dive

```hcl
# terraform/main.tf

# BLOCK TYPES: terraform, provider, resource, data, variable, 
#              output, locals, module

terraform {
  required_version = ">= 1.5.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"    # registry.terraform.io/hashicorp/aws
      version = "~> 5.0"          # >= 5.0.0, < 6.0.0
    }
  }

  # Remote State Backend
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"  # Distributed locking
  }
}

provider "aws" {
  region = var.aws_region
  
  default_tags {
    tags = {
      ManagedBy   = "Terraform"
      Environment = var.environment
    }
  }
}

# VARIABLES — inputs to the module
variable "aws_region" {
  description = "AWS region to deploy resources"
  type        = string
  default     = "us-east-1"
  
  validation {
    condition     = contains(["us-east-1", "us-west-2", "eu-west-1"], var.aws_region)
    error_message = "Must be a valid production region."
  }
}

variable "environment" {
  type = string
}

variable "vpc_config" {
  type = object({
    cidr_block           = string
    enable_dns_hostnames = bool
    availability_zones   = list(string)
  })
}

# LOCALS — computed values within module
locals {
  name_prefix = "${var.environment}-${var.aws_region}"
  common_tags = {
    Environment = var.environment
    Region      = var.aws_region
    CreatedAt   = timestamp()
  }
  
  # Subnet CIDRs computed from VPC CIDR
  public_subnet_cidrs  = [for i, az in var.vpc_config.availability_zones : 
                           cidrsubnet(var.vpc_config.cidr_block, 8, i)]
  private_subnet_cidrs = [for i, az in var.vpc_config.availability_zones : 
                           cidrsubnet(var.vpc_config.cidr_block, 8, i + 10)]
}

# RESOURCE — actual infrastructure
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_config.cidr_block
  enable_dns_hostnames = var.vpc_config.enable_dns_hostnames
  
  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-vpc"
  })
}

# DATA SOURCE — read existing infrastructure
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

# DYNAMIC BLOCKS — generate repeated nested blocks
resource "aws_security_group" "web" {
  name   = "${local.name_prefix}-web-sg"
  vpc_id = aws_vpc.main.id

  dynamic "ingress" {
    for_each = [80, 443]
    content {
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }
}

# COUNT vs FOR_EACH
# Count — simple repetition (index-based, fragile on delete)
resource "aws_subnet" "public_count" {
  count             = length(local.public_subnet_cidrs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = local.public_subnet_cidrs[count.index]
  availability_zone = var.vpc_config.availability_zones[count.index]
}

# For_each — map-based (stable on delete, preferred)
resource "aws_subnet" "public" {
  for_each = tomap({
    for i, az in var.vpc_config.availability_zones :
    az => local.public_subnet_cidrs[i]
  })
  
  vpc_id            = aws_vpc.main.id
  cidr_block        = each.value
  availability_zone = each.key
  
  tags = { Name = "${local.name_prefix}-public-${each.key}" }
}

# OUTPUTS — expose values for other modules/humans
output "vpc_id" {
  description = "ID of the created VPC"
  value       = aws_vpc.main.id
}

output "public_subnet_ids" {
  value = [for subnet in aws_subnet.public : subnet.id]
}
```

### Terraform State — The Brain

```
STATE FILE (terraform.tfstate) — JSON format

{
  "version": 4,
  "terraform_version": "1.5.7",
  "serial": 42,              ← increments on every change
  "lineage": "uuid-...",     ← unique ID for this state
  "outputs": { ... },
  "resources": [
    {
      "mode": "managed",
      "type": "aws_vpc",
      "name": "main",
      "provider": "provider[\"registry.terraform.io/hashicorp/aws\"]",
      "instances": [
        {
          "schema_version": 1,
          "attributes": {
            "id": "vpc-0abc123",
            "cidr_block": "10.0.0.0/16",
            "arn": "arn:aws:ec2:us-east-1:123456789:vpc/vpc-0abc123",
            ...ALL RESOURCE ATTRIBUTES...
          },
          "sensitive_attributes": [],
          "dependencies": ["aws_internet_gateway.main"]
        }
      ]
    }
  ]
}
```

**State Locking Flow (S3 + DynamoDB):**

```
terraform apply
       │
       ▼
┌─────────────────┐
│ Read State from  │ ──── GET s3://bucket/terraform.tfstate
│ S3 Backend       │
└─────────────────┘
       │
       ▼
┌─────────────────┐
│ Acquire Lock     │ ──── DynamoDB PutItem (conditional)
│ DynamoDB         │      Item: {LockID: "bucket/key", ...}
└─────────────────┘      If item exists → "Error acquiring lock"
       │
       ▼
┌─────────────────┐
│ Plan + Apply     │
│ Execute Changes  │
└─────────────────┘
       │
       ▼
┌─────────────────┐
│ Write New State  │ ──── PUT s3://bucket/terraform.tfstate
│ to S3            │
└─────────────────┘
       │
       ▼
┌─────────────────┐
│ Release Lock     │ ──── DynamoDB DeleteItem
│ DynamoDB         │
└─────────────────┘
```

### Terraform Workflow Deep Dive

```
terraform init
├── Download provider plugins (~/.terraform/providers/)
├── Initialize backend (connect to S3/remote)
├── Download modules
└── Create .terraform.lock.hcl (provider version pinning)

terraform plan
├── Read current state (from backend)
├── Refresh state (API calls to get real current state)
├── Load configuration (.tf files)
├── Build dependency graph (DAG)
├── Walk the graph → compute diff
│   ├── In state, not in config → DESTROY
│   ├── In config, not in state → CREATE
│   └── In both, attributes differ → UPDATE or REPLACE
└── Output human-readable plan

terraform apply
├── Show plan (or re-run plan)
├── User confirms (or -auto-approve)
├── Lock state
├── Walk dependency graph in parallel
│   ├── Independent resources execute concurrently
│   └── Dependent resources wait for dependencies
├── Call provider gRPC: Create/Read/Update/Delete
├── Update state after each resource success/fail
├── Write final state
└── Release lock
```

### Terraform Modules — Production Pattern

```
infrastructure/
├── main.tf                     # Root module
├── variables.tf
├── outputs.tf
├── terraform.tfvars            # Variable values (gitignore secrets!)
├── environments/
│   ├── dev.tfvars
│   ├── staging.tfvars
│   └── prod.tfvars
└── modules/
    ├── vpc/                    # Reusable VPC module
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    ├── eks/                    # EKS cluster module
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    └── rds/                    # RDS module
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

**Module usage:**

```hcl
# Root main.tf — composing modules
module "vpc" {
  source = "./modules/vpc"      # Local module
  # source = "git::https://github.com/org/tf-modules.git//vpc?ref=v1.2.0"
  # source = "registry.terraform.io/hashicorp/vpc/aws"  # Public registry
  
  environment        = var.environment
  cidr_block         = "10.0.0.0/16"
  availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

module "eks" {
  source = "./modules/eks"
  
  vpc_id          = module.vpc.vpc_id           # Module output reference
  subnet_ids      = module.vpc.private_subnet_ids
  cluster_version = "1.28"
}

module "rds" {
  source = "./modules/rds"
  
  vpc_id             = module.vpc.vpc_id
  subnet_ids         = module.vpc.private_subnet_ids
  allowed_cidr_blocks = [module.vpc.vpc_cidr_block]
}
```

### Terraform Workspaces vs. Separate State

```
WORKSPACES (for small teams/simple envs)
├── terraform workspace new dev
├── terraform workspace new staging  
├── terraform workspace new prod
├── State stored separately per workspace
│   s3://bucket/env:/dev/terraform.tfstate
│   s3://bucket/env:/prod/terraform.tfstate
└── Access: terraform.workspace variable

SEPARATE STATE FILES (recommended for prod)
├── environments/
│   ├── dev/
│   │   └── terraform.tfstate  (in S3: prod-state/dev/...)
│   ├── staging/
│   │   └── terraform.tfstate
│   └── prod/
│       └── terraform.tfstate  (strict access control!)
└── Completely isolated — prod changes can't affect dev
```

### Terraform Production Best Practices

```hcl
# 1. ALWAYS pin provider versions
terraform {
  required_providers {
    aws = { source = "hashicorp/aws", version = "= 5.31.0" }
  }
}

# 2. Use remote state with locking
backend "s3" {
  bucket         = "company-terraform-state"
  key            = "${var.environment}/terraform.tfstate"
  region         = "us-east-1"
  encrypt        = true
  kms_key_id     = "arn:aws:kms:..."
  dynamodb_table = "terraform-locks"
}

# 3. Sensitive outputs
output "db_password" {
  value     = random_password.db.result
  sensitive = true    # Hidden in CLI output, but still in state!
}

# 4. Prevent accidental deletion
resource "aws_rds_cluster" "main" {
  # ...
  deletion_protection = true
  
  lifecycle {
    prevent_destroy = true    # terraform destroy will error
    ignore_changes  = [engine_version]  # Ignore drift on this attr
    create_before_destroy = true        # Blue-green for replacements
  }
}

# 5. Import existing resources
# terraform import aws_vpc.main vpc-0abc123
```

### Terraform Troubleshooting

```bash
# Enable debug logging
export TF_LOG=DEBUG         # DEBUG, INFO, WARN, ERROR, TRACE
export TF_LOG_PATH=/tmp/terraform.log
terraform apply

# State manipulation (use with extreme caution!)
terraform state list                          # List all resources
terraform state show aws_vpc.main             # Show resource details
terraform state mv aws_vpc.old aws_vpc.new    # Rename resource
terraform state rm aws_instance.web           # Remove from state (doesn't destroy)
terraform import aws_vpc.main vpc-0abc123    # Import existing resource

# Refresh state (sync state with real world)
terraform refresh

# Target specific resources
terraform plan -target=module.eks
terraform apply -target=aws_vpc.main

# Force unlock (if stuck lock)
terraform force-unlock LOCK_ID

# Validate syntax
terraform validate
terraform fmt -recursive .  # Format all .tf files

# Common errors and fixes:
# "Error: Provider produced inconsistent result after apply"
# → Provider bug or race condition, retry usually works

# "Error acquiring the state lock"  
# → Check DynamoDB for stale lock, terraform force-unlock

# "Error: Cycle: resource A → resource B → resource A"
# → Circular dependency in configuration

# "│ Error: creating EC2 VPC: VpcLimitExceeded"
# → AWS account VPC limit hit, request limit increase
```

---

## 6.3 Ansible — Agentless Automation

### What is Ansible?

Ansible is an open-source automation tool for configuration management, application deployment, and task automation. It is **agentless** — no software needs to be installed on managed nodes.

### Ansible Architecture

```
┌────────────────────────────────────────────────────────────────┐
│                      CONTROL NODE                               │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              ansible / ansible-playbook CLI               │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐              │
│  │  Inventory  │  │  Playbooks │  │  Roles     │              │
│  │  (hosts)   │  │  (YAML)    │  │  (reusable)│              │
│  └────────────┘  └────────────┘  └────────────┘              │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    MODULE LIBRARY                         │  │
│  │  apt, yum, copy, template, service, file, user,          │  │
│  │  shell, command, docker_container, k8s, aws_ec2 ...      │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────┘
                         │
               SSH (port 22) or WinRM
               Pushes Python module
               Executes remotely
               Returns JSON result
                         │
         ┌───────────────┼───────────────┐
         ▼               ▼               ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│  Managed     │ │  Managed     │ │  Managed     │
│  Node 1      │ │  Node 2      │ │  Node 3      │
│  (no agent!) │ │  (no agent!) │ │  (no agent!) │
│  Python req  │ │  Python req  │ │  Python req  │
└──────────────┘ └──────────────┘ └──────────────┘
```

**How Ansible Executes a Task (Under the Hood):**

```
1. ansible-playbook site.yml runs on control node
2. Ansible reads inventory → gets list of hosts
3. For each task, Ansible:
   a. Connects via SSH (uses paramiko or OpenSSH)
   b. Creates temp directory on remote: /tmp/.ansible/tmp/xxx/
   c. Copies the module Python file to remote temp dir
   d. Executes: /usr/bin/python /tmp/.ansible/tmp/xxx/module.py
   e. Module reads args from a JSON args file
   f. Module executes the actual work (install package, write file, etc.)
   g. Module prints JSON result to stdout
   h. Ansible reads JSON: {"changed": true/false, "rc": 0, ...}
   i. Removes temp files
   j. Reports result (OK / CHANGED / FAILED / SKIPPED)
```

### Inventory

```ini
# hosts.ini — Static inventory

[webservers]
web1.prod.example.com
web2.prod.example.com ansible_user=ec2-user ansible_port=22

[databases]
db1.prod.example.com
db2.prod.example.com

[prod:children]  # Group of groups
webservers
databases

[prod:vars]  # Variables for all prod hosts
ansible_python_interpreter=/usr/bin/python3
environment=production

[webservers:vars]
http_port=80
nginx_worker_processes=auto
```

```yaml
# inventory/hosts.yml — YAML inventory (preferred)
all:
  vars:
    ansible_user: ubuntu
    ansible_ssh_private_key_file: ~/.ssh/prod-key.pem
  
  children:
    webservers:
      hosts:
        web1:
          ansible_host: 10.0.1.10
          nginx_port: 80
        web2:
          ansible_host: 10.0.1.11
          nginx_port: 80
    
    databases:
      hosts:
        db1:
          ansible_host: 10.0.2.10
          postgresql_version: "15"
    
    monitoring:
      hosts:
        prometheus:
          ansible_host: 10.0.3.10
```

```bash
# Dynamic Inventory — auto-discover from cloud
# aws_ec2 plugin
# inventory/aws_ec2.yml
plugin: aws_ec2
regions:
  - us-east-1
filters:
  instance-state-name: running
  tag:Environment: production
keyed_groups:
  - key: tags.Role
    prefix: role
hostnames:
  - private-ip-address

# Use: ansible-playbook -i inventory/aws_ec2.yml site.yml
```

### Playbooks — Complete Reference

```yaml
# site.yml — Master playbook
---
- name: Configure Web Servers
  hosts: webservers
  become: true          # sudo
  become_user: root
  gather_facts: true    # Collect system info (OS, IP, memory, etc.)
  serial: 2             # Run on 2 hosts at a time (rolling update)
  max_fail_percentage: 20  # Abort if >20% of hosts fail
  
  vars:
    nginx_version: "1.24"
    app_port: 8080
  
  vars_files:
    - vars/common.yml
    - "vars/{{ ansible_os_family }}.yml"  # Dynamic var file
  
  pre_tasks:
    - name: Update apt cache
      apt:
        update_cache: true
        cache_valid_time: 3600
      when: ansible_os_family == "Debian"
  
  roles:
    - common          # Applied to all servers
    - nginx           # Web server config
    - app             # Application deployment
  
  post_tasks:
    - name: Verify nginx is serving
      uri:
        url: "http://{{ ansible_default_ipv4.address }}/health"
        return_content: true
      register: health_check
      failed_when: health_check.status != 200
  
  handlers:
    - name: restart nginx
      service:
        name: nginx
        state: restarted

# Handlers are triggered by 'notify', run at end of play
# If 5 tasks all notify 'restart nginx', it runs ONCE
```

### Roles — Production Structure

```
roles/
└── nginx/
    ├── tasks/
    │   ├── main.yml          # Entry point
    │   ├── install.yml
    │   └── configure.yml
    ├── handlers/
    │   └── main.yml          # restart nginx handler
    ├── templates/
    │   ├── nginx.conf.j2     # Jinja2 templates
    │   └── vhost.conf.j2
    ├── files/
    │   └── ssl/              # Static files (not templated)
    ├── vars/
    │   └── main.yml          # Role variables (high priority)
    ├── defaults/
    │   └── main.yml          # Default variables (low priority)
    ├── meta/
    │   └── main.yml          # Role dependencies
    └── README.md
```

```yaml
# roles/nginx/tasks/main.yml
---
- name: Install nginx
  package:
    name: "nginx={{ nginx_version }}*"
    state: present
  notify: restart nginx

- name: Deploy nginx configuration
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    owner: root
    group: root
    mode: '0644'
    validate: '/usr/sbin/nginx -t -c %s'  # Validate before deploying!
  notify: restart nginx

- name: Deploy virtual host configs
  template:
    src: vhost.conf.j2
    dest: "/etc/nginx/sites-available/{{ item.name }}.conf"
  loop: "{{ vhosts }}"  # Iterate over list of vhosts
  
- name: Enable virtual hosts
  file:
    src: "/etc/nginx/sites-available/{{ item.name }}.conf"
    dest: "/etc/nginx/sites-enabled/{{ item.name }}.conf"
    state: link
  loop: "{{ vhosts }}"
  notify: restart nginx

- name: Ensure nginx is started and enabled
  service:
    name: nginx
    state: started
    enabled: true

- name: Open firewall port
  ufw:
    rule: allow
    port: "{{ nginx_port }}"
    proto: tcp
```

```jinja2
{# roles/nginx/templates/nginx.conf.j2 #}
user {{ nginx_user | default('www-data') }};
worker_processes {{ nginx_worker_processes | default('auto') }};
error_log /var/log/nginx/error.log {{ nginx_error_log_level | default('warn') }};

events {
    worker_connections {{ nginx_worker_connections | default(1024) }};
    use epoll;
}

http {
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout {{ nginx_keepalive_timeout | default(65) }};
    
    {% if nginx_gzip_enabled | default(true) %}
    gzip on;
    gzip_types text/plain text/css application/json application/javascript;
    {% endif %}
    
    {% for vhost in vhosts %}
    include /etc/nginx/sites-enabled/{{ vhost.name }}.conf;
    {% endfor %}
}
```

### Ansible Vault — Secrets Management

```bash
# Encrypt a file
ansible-vault encrypt vars/secrets.yml

# Create encrypted file directly
ansible-vault create vars/secrets.yml

# Edit encrypted file
ansible-vault edit vars/secrets.yml

# Encrypt a single string (inline)
ansible-vault encrypt_string 'myS3cr3tP@ss' --name 'db_password'
# Output:
# db_password: !vault |
#   $ANSIBLE_VAULT;1.1;AES256
#   38316537353230366666...

# Run playbook with vault password
ansible-playbook site.yml --ask-vault-pass
ansible-playbook site.yml --vault-password-file ~/.vault_pass
```

### Ansible Troubleshooting

```bash
# Check connectivity
ansible all -i hosts.ini -m ping

# Run ad-hoc commands
ansible webservers -i hosts.ini -m shell -a "df -h"
ansible webservers -i hosts.ini -m service -a "name=nginx state=restarted" --become

# Dry run (check mode)
ansible-playbook site.yml --check --diff

# Step through tasks interactively
ansible-playbook site.yml --step

# Verbose output (increase -v for more)
ansible-playbook site.yml -v    # Task output
ansible-playbook site.yml -vv   # + Connection info
ansible-playbook site.yml -vvv  # + SSH commands
ansible-playbook site.yml -vvvv # + Connection debug

# Limit to specific hosts/groups
ansible-playbook site.yml --limit web1.example.com
ansible-playbook site.yml --limit "webservers:!web1"  # All except web1

# Start at specific task
ansible-playbook site.yml --start-at-task="Deploy nginx configuration"

# Tag-based execution
ansible-playbook site.yml --tags install,configure
ansible-playbook site.yml --skip-tags deploy

# List tasks without running
ansible-playbook site.yml --list-tasks
ansible-playbook site.yml --list-hosts
```

---

## 6.4 Helm — The Kubernetes Package Manager

### What is Helm?

Helm is the package manager for Kubernetes. A **Helm chart** is a collection of Kubernetes YAML manifests templated with Go templating, packaged into a distributable artifact.

### Why Helm?

```
WITHOUT HELM:
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f configmap.yaml
kubectl apply -f ingress.yaml
kubectl apply -f hpa.yaml
... 20 more files ...

# To change environment: edit each file manually
# To upgrade: reapply all files
# To rollback: git revert and re-apply
# To share: copy all files

WITH HELM:
helm install my-app ./my-chart --values prod.values.yaml
helm upgrade my-app ./my-chart --values prod.values.yaml
helm rollback my-app 1
helm package ./my-chart → my-chart-1.0.0.tgz  (shareable artifact)
```

### Helm Chart Structure

```
my-app/
├── Chart.yaml              # Chart metadata
├── values.yaml             # Default values
├── values-prod.yaml        # Environment override (not part of chart spec)
├── templates/
│   ├── _helpers.tpl        # Named templates (reusable snippets)
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── hpa.yaml
│   ├── serviceaccount.yaml
│   └── NOTES.txt           # Post-install instructions
├── charts/                 # Dependency charts (subcharts)
│   └── postgresql/
└── crds/                   # Custom Resource Definitions
```

```yaml
# Chart.yaml
apiVersion: v2
name: my-app
description: Production web application
type: application   # or "library" for shared templates
version: 1.2.3      # Chart version (SemVer)
appVersion: "2.1.0" # App version being packaged

dependencies:
  - name: postgresql
    version: "13.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled  # Only if .Values.postgresql.enabled=true
  - name: redis
    version: "18.x.x"
    repository: "https://charts.bitnami.com/bitnami"
```

```yaml
# values.yaml — defaults, overridable
replicaCount: 2

image:
  repository: mycompany/my-app
  tag: ""         # Defaults to .Chart.AppVersion
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: false
  className: nginx
  annotations: {}
  hosts:
    - host: app.example.com
      paths:
        - path: /
          pathType: Prefix
  tls: []

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 100m
    memory: 128Mi

autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

postgresql:
  enabled: true
  auth:
    postgresPassword: ""  # Set via --set or secrets

env: []
  # - name: DATABASE_URL
  #   valueFrom:
  #     secretKeyRef:
  #       name: app-secrets
  #       key: database-url
```

```yaml
# templates/deployment.yaml — with Go templating
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-app.fullname" . }}   {{- /* Named template */}}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}   {{- /* Indent 4 spaces */}}
  annotations:
    helm.sh/chart: {{ include "my-app.chart" . }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "my-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "my-app.selectorLabels" . | nindent 8 }}
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
        {{- /* ↑ Force pod restart when configmap changes */}}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.service.targetPort }}
          {{- with .Values.env }}
          env:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          livenessProbe:
            httpGet:
              path: /health
              port: {{ .Values.service.targetPort }}
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: {{ .Values.service.targetPort }}
            initialDelaySeconds: 5
            periodSeconds: 5
```

```
# templates/_helpers.tpl — reusable named templates
{{/*
Expand the name of the chart.
*/}}
{{- define "my-app.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "my-app.labels" -}}
helm.sh/chart: {{ include "my-app.chart" . }}
{{ include "my-app.selectorLabels" . }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}
```

### Helm Hooks

```yaml
# templates/db-migrate-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "my-app.fullname" . }}-migrate
  annotations:
    "helm.sh/hook": pre-upgrade,pre-install       # When to run
    "helm.sh/hook-weight": "-5"                   # Order (lower = earlier)
    "helm.sh/hook-delete-policy": hook-succeeded  # Cleanup after success
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          command: ["python", "manage.py", "migrate"]
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: database-url
```

### Helm Commands — Production Reference

```bash
# Repo management
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm search repo nginx --versions

# Install
helm install my-release ./my-chart \
  --namespace production \
  --create-namespace \
  --values values-prod.yaml \
  --set image.tag=v2.1.0 \
  --set postgresql.auth.postgresPassword=secret \
  --wait \              # Wait for all pods to be ready
  --timeout 10m

# Upgrade
helm upgrade my-release ./my-chart \
  --namespace production \
  --values values-prod.yaml \
  --set image.tag=v2.2.0 \
  --atomic \            # Rollback automatically if upgrade fails
  --cleanup-on-fail

# Rollback
helm history my-release -n production    # See revision history
helm rollback my-release 3 -n production # Roll back to revision 3

# Debug/dry-run
helm template my-release ./my-chart --values values-prod.yaml
helm install my-release ./my-chart --dry-run --debug

# Package and push to OCI registry
helm package ./my-chart
helm push my-chart-1.2.3.tgz oci://registry.example.com/charts

# Uninstall
helm uninstall my-release -n production
```

---

## 6.5 Pulumi — IaC with Real Programming Languages

### What is Pulumi?

Pulumi allows you to define infrastructure using **real programming languages** — TypeScript, Python, Go, C#, Java — instead of DSLs like HCL.

```typescript
// index.ts — Pulumi TypeScript example
import * as aws from "@pulumi/aws";
import * as awsx from "@pulumi/awsx";

// Create VPC with subnets automatically
const vpc = new awsx.ec2.Vpc("main-vpc", {
    cidrBlock: "10.0.0.0/16",
    numberOfAvailabilityZones: 3,
    subnetStrategy: awsx.ec2.SubnetAllocationStrategy.Auto,
});

// EKS Cluster
const cluster = new awsx.eks.Cluster("prod-cluster", {
    vpcId: vpc.vpcId,
    privateSubnetIds: vpc.privateSubnetIds,
    instanceType: "t3.medium",
    desiredCapacity: 3,
    minSize: 1,
    maxSize: 10,
});

// Use real language features: loops, conditionals, functions
const buckets = ["assets", "logs", "backups"].map(name => 
    new aws.s3.Bucket(`${name}-bucket`, {
        versioning: { enabled: name !== "logs" },
        serverSideEncryptionConfiguration: {
            rule: {
                applyServerSideEncryptionByDefault: {
                    sseAlgorithm: "AES256"
                }
            }
        }
    })
);

// Export outputs
export const kubeconfig = cluster.kubeconfig;
export const vpcId = vpc.vpcId;
```

### Terraform vs Pulumi vs Ansible vs Helm

| Feature | Terraform | Pulumi | Ansible | Helm |
|---------|-----------|--------|---------|------|
| Language | HCL | TS/Python/Go/C# | YAML | YAML + Go templates |
| Approach | Declarative | Imperative+Declarative | Procedural | Declarative |
| State | tfstate file | Pulumi Service/S3 | None (idempotent) | Kubernetes Secrets |
| Agentless? | Yes | Yes | Yes (SSH) | Yes (kubectl) |
| Scope | Cloud infra | Cloud infra | Config mgmt | K8s packaging |
| Learning curve | Medium | Low (if you know TS/Python) | Low | Medium |
| Best for | Multi-cloud provisioning | Devs who prefer code | Server config | K8s app packaging |

---

## 6.6 IaC Interview Questions

### Beginner

**Q: What is the difference between Terraform plan and apply?**

> **Expected Answer:** `terraform plan` shows what changes WOULD be made without making them — it computes a diff between desired state (config) and current state (state file + real infrastructure). `terraform apply` executes those changes. Always run plan first in production. Plan output should be reviewed and ideally stored as an artifact for audit trails.
>
> **Why asked:** Tests understanding of Terraform's core workflow and safety practices.

**Q: What is Ansible's agentless architecture and how does it work?**

> **Expected Answer:** Ansible connects to managed nodes via SSH (Linux) or WinRM (Windows). No agent/daemon runs on managed nodes. Ansible copies Python module files to the remote node, executes them, reads the JSON result back, then deletes the temp files. Only requirement on managed nodes is Python (usually pre-installed) and SSH access.
>
> **Common mistake:** Saying "Ansible uses an agent like Chef/Puppet."

### Advanced

**Q: Explain Terraform state drift and how to handle it.**

> **Expected Answer:** State drift occurs when real infrastructure differs from the Terraform state file — caused by manual changes in the cloud console, changes by other tools, or failed applies. Detection: `terraform plan` shows unexpected diffs; `terraform refresh` updates state from real infra. Handling: import manually created resources (`terraform import`), use `lifecycle.ignore_changes` for attributes that drift intentionally, implement drift detection in CI/CD pipelines. Prevention: enforce IaC-only changes via IAM policies that deny console modifications.

**Q: How do you handle secrets in Terraform?**

> **Expected Answer:** Never store secrets in .tf files or commit tfvars to git. Options: (1) Environment variables (`TF_VAR_db_password`), (2) AWS Secrets Manager/Parameter Store data sources — fetch at apply time, (3) HashiCorp Vault provider, (4) SOPS for encrypted variable files. Critical: State file contains secrets in plaintext — encrypt state backend with KMS, restrict S3 bucket access, enable versioning for state recovery.

**Q: Design a Terraform architecture for a company with 50 engineers, 3 environments, and multiple cloud accounts.**

> **Expected Answer:**
> - Separate AWS accounts per environment (dev/staging/prod) — blast radius isolation
> - One state file per environment per service — avoid monolithic state
> - Terraform modules in a separate repository, versioned with git tags
> - Terragrunt for DRY configurations across environments  
> - CI/CD: plan on PR, apply on merge (with approval gates for prod)
> - State buckets encrypted + versioned, IAM-restricted
> - `atlantis` or Terraform Cloud for collaborative plan/apply workflow
> - Sentinel policies (Terraform Enterprise) for policy-as-code
```
