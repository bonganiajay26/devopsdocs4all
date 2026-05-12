# Phase 8: Cloud & Security

---

## 8.1 Cloud Computing Fundamentals

### Service Models

```
┌─────────────────────────────────────────────────────────────────┐
│                     CLOUD SERVICE MODELS                         │
│                                                                   │
│  IaaS                  PaaS                    SaaS             │
│  (Infrastructure)      (Platform)              (Software)        │
│                                                                   │
│  YOU manage:           YOU manage:             YOU manage:       │
│  ├── Applications      ├── Applications        └── (nothing)    │
│  ├── Data              ├── Data                                  │
│  ├── Runtime           └── (rest is cloud)     Examples:         │
│  ├── Middleware                                 Gmail, Salesforce │
│  └── OS                Examples:               Slack, Zoom       │
│                         Heroku, GCP App Engine                   │
│  CLOUD manages:         AWS Elastic Beanstalk                   │
│  ├── Virtualization                                              │
│  ├── Servers           CLOUD manages:                            │
│  ├── Storage           ├── Runtime                               │
│  └── Networking        ├── Middleware                            │
│                        ├── OS                                    │
│  Examples:             ├── Virtualization                        │
│  AWS EC2, Azure VM     ├── Servers                               │
│  GCP Compute Engine    ├── Storage                               │
│                        └── Networking                            │
└─────────────────────────────────────────────────────────────────┘
```

---

## 8.2 AWS — Deep Dive

### AWS Global Infrastructure

```
REGION (us-east-1, eu-west-1, ap-southeast-1...)
  └── AVAILABILITY ZONE (us-east-1a, 1b, 1c)
        └── DATA CENTERS (multiple physical buildings)
              └── SERVERS

EDGE LOCATIONS: 400+ POPs for CloudFront CDN
LOCAL ZONES: AWS infrastructure closer to specific cities
WAVELENGTH ZONES: AWS infra embedded in 5G networks
OUTPOSTS: AWS rack in your on-premises data center
```

### VPC — Virtual Private Cloud (Most Critical Service)

```
REGION: us-east-1
┌──────────────────────────────────────────────────────────────────┐
│  VPC: 10.0.0.0/16                                                │
│                                                                    │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  AVAILABILITY ZONE: us-east-1a                              │  │
│  │                                                              │  │
│  │  ┌─────────────────┐    ┌─────────────────┐               │  │
│  │  │ Public Subnet    │    │ Private Subnet   │               │  │
│  │  │ 10.0.1.0/24     │    │ 10.0.11.0/24    │               │  │
│  │  │                  │    │                  │               │  │
│  │  │ [NAT Gateway]    │    │ [EC2 App]        │               │  │
│  │  │ [Load Balancer]  │    │ [RDS Primary]    │               │  │
│  │  │ [Bastion Host]   │    │ [ElastiCache]    │               │  │
│  │  └─────────────────┘    └─────────────────┘               │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  AVAILABILITY ZONE: us-east-1b                              │  │
│  │  ┌─────────────────┐    ┌─────────────────┐               │  │
│  │  │ Public Subnet    │    │ Private Subnet   │               │  │
│  │  │ 10.0.2.0/24     │    │ 10.0.12.0/24    │               │  │
│  │  │ [NAT Gateway]    │    │ [EC2 App]        │               │  │
│  │  └─────────────────┘    │ [RDS Standby]    │               │  │
│  │                          └─────────────────┘               │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  [Internet Gateway] — connects VPC to internet                    │
│  [Route Tables] — controls traffic routing                        │
│  [Security Groups] — stateful firewall per resource              │
│  [NACLs] — stateless firewall per subnet                         │
│  [VPC Endpoints] — private access to AWS services (no internet)  │
└──────────────────────────────────────────────────────────────────┘
```

**Network Flow — User to App:**

```
User (Internet)
      │
      ▼ HTTPS:443
[Route 53 DNS] → resolves to ALB IP
      │
      ▼
[Internet Gateway] → entry point to VPC
      │
      ▼
[Application Load Balancer] (in Public Subnets, across AZs)
      │  Security Group: allow 80,443 from 0.0.0.0/0
      │
      ▼ HTTP:8080
[EC2 / ECS / EKS] (in Private Subnets)
      │  Security Group: allow 8080 from ALB Security Group only
      │
      ▼ TCP:5432
[RDS PostgreSQL] (in Private Subnets)
      │  Security Group: allow 5432 from App Security Group only
      │
      ▼ (outbound only, no direct internet access)
[NAT Gateway] (in Public Subnet) → Internet for outbound calls
```

### Core AWS Services Reference

```
COMPUTE:
  EC2          Virtual machines
  ECS          Container orchestration (uses EC2 or Fargate)
  EKS          Managed Kubernetes
  Fargate      Serverless containers (no EC2 management)
  Lambda       Serverless functions (event-driven, 15min max)
  Batch        Batch computing jobs
  Lightsail    Simple VPS (beginner-friendly)

STORAGE:
  S3           Object storage (unlimited, any file)
  EBS          Block storage (attached to EC2, like a hard drive)
  EFS          NFS shared file system (multi-EC2)
  FSx          Windows File Server / Lustre (HPC)
  S3 Glacier   Archival storage (cheap, slow retrieval)
  AWS Backup   Centralized backup service

DATABASE:
  RDS          Managed relational DB (MySQL, PG, MariaDB, Oracle, SQL Server)
  Aurora       AWS-optimized MySQL/PostgreSQL (5x faster, auto-scaling)
  DynamoDB     Managed NoSQL (key-value + document, millisecond latency)
  ElastiCache  Managed Redis/Memcached
  Redshift     Data warehouse (columnar, petabyte scale)
  DocumentDB   MongoDB-compatible
  Neptune      Graph database
  Timestream   Time-series database

NETWORKING:
  VPC          Virtual private cloud
  Route 53     DNS + health checks + routing policies
  CloudFront   CDN (edge caching)
  ELB          Load balancing (ALB=L7, NLB=L4, GWLB=L3)
  Direct Connect  Dedicated private link from on-prem to AWS
  VPN          Site-to-site or client VPN
  Transit Gateway  Hub-and-spoke VPC interconnect
  PrivateLink  Private connectivity to SaaS services

SECURITY:
  IAM          Identity & Access Management
  KMS          Key Management Service (encryption keys)
  Secrets Manager  Store and rotate secrets
  Certificate Manager  Free SSL/TLS certs
  WAF          Web Application Firewall
  Shield       DDoS protection
  GuardDuty    Threat detection (ML-based)
  Security Hub  Security posture management
  Macie        S3 data classification / PII detection
  Inspector    Vulnerability scanning

MONITORING:
  CloudWatch   Metrics, logs, alarms, dashboards
  CloudTrail   API audit log (who did what, when, from where)
  Config       Resource configuration history + compliance
  X-Ray        Distributed tracing

DEVELOPER TOOLS:
  CodeCommit   Git repositories
  CodeBuild    CI build service
  CodeDeploy   Deployment automation
  CodePipeline CI/CD pipeline
  ECR          Container registry
  SAM          Serverless Application Model

MESSAGING:
  SQS          Message queue (decoupling)
  SNS          Pub/sub notifications
  EventBridge  Event bus (formerly CloudWatch Events)
  Kinesis      Real-time streaming data
  MSK          Managed Kafka
```

### EKS — Elastic Kubernetes Service

```
EKS Architecture:

AWS MANAGED CONTROL PLANE (you pay per cluster/hour)
┌─────────────────────────────────────────────────────┐
│  kube-apiserver (multi-AZ, highly available)         │
│  etcd (multi-AZ, encrypted)                          │
│  kube-scheduler, controller-manager (AWS managed)    │
└─────────────────────────────────────────────────────┘
                        │
                   (AWS VPC endpoint)
                        │
YOUR VPC (worker nodes)
┌─────────────────────────────────────────────────────┐
│  Node Group 1 (EC2 + Auto Scaling Group)             │
│  ├── Node (EC2 t3.large) — kubelet + kube-proxy     │
│  ├── Node (EC2 t3.large)                             │
│  └── Node (EC2 t3.large)                             │
│                                                       │
│  Node Group 2 (GPU nodes for ML workloads)           │
│  ├── Node (EC2 p3.2xlarge)                           │
│  └── Node (EC2 p3.2xlarge)                           │
│                                                       │
│  Fargate Profile (serverless, no EC2)                │
│  └── namespace: serverless-apps → runs on Fargate    │
└─────────────────────────────────────────────────────┘
```

```bash
# EKS Setup with eksctl
cat > cluster.yaml << 'EOF'
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: production
  region: us-east-1
  version: "1.28"
  tags:
    Environment: production

vpc:
  subnets:
    private:
      us-east-1a: { id: subnet-xxx }
      us-east-1b: { id: subnet-yyy }
    public:
      us-east-1a: { id: subnet-aaa }
      us-east-1b: { id: subnet-bbb }

managedNodeGroups:
  - name: general
    instanceType: t3.large
    minSize: 3
    maxSize: 20
    desiredCapacity: 5
    privateNetworking: true      # Nodes in private subnets
    volumeSize: 100
    volumeType: gp3
    iam:
      withAddonPolicies:
        autoScaler: true
        awsLoadBalancerController: true
        certManager: true
        ebs: true
        efs: true
    tags:
      k8s.io/cluster-autoscaler/enabled: "true"
      k8s.io/cluster-autoscaler/production: "owned"
  
  - name: spot-workers
    instanceTypes: ["t3.large", "t3a.large", "t3.xlarge"]
    spot: true                   # 70% cost savings
    minSize: 0
    maxSize: 50
    privateNetworking: true

addons:
  - name: vpc-cni
    version: latest
  - name: coredns
    version: latest
  - name: kube-proxy
    version: latest
  - name: aws-ebs-csi-driver
    version: latest

iam:
  withOIDC: true               # Enable IRSA
EOF

eksctl create cluster -f cluster.yaml
```

### IAM — Identity and Access Management (Critical)

**IAM Policy Structure:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3ReadOnSpecificBucket",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:GetObjectVersion",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-company-bucket",
        "arn:aws:s3:::my-company-bucket/*"
      ],
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": "us-east-1"
        },
        "Bool": {
          "aws:SecureTransport": "true"
        }
      }
    },
    {
      "Effect": "Deny",
      "Action": "s3:DeleteObject",
      "Resource": "arn:aws:s3:::my-company-bucket/*"
    }
  ]
}
```

**IAM Policy Evaluation Logic:**

```
Is there an explicit DENY? ──── YES ──→ DENY (final)
         │
         NO
         │
         ▼
Is there a matching ALLOW? ──── NO ───→ DENY (implicit)
         │
         YES
         │
         ▼
Is there an SCP (Service Control Policy)? ──── YES, and it DENIES → DENY
         │
         NO (or allows)
         │
         ▼
      ALLOW ✓
```

**IRSA — IAM Roles for Service Accounts (EKS):**

```
Without IRSA:                   With IRSA:
EC2 Instance Role               Pod-level IAM role
ALL pods on node get            ONLY specific pods get
SAME permissions                specific permissions
                                Least privilege!

How IRSA works:
1. EKS cluster has OIDC provider
2. ServiceAccount annotated with IAM role ARN
3. Pod mounts projected service account token
4. AWS SDK calls STS AssumeRoleWithWebIdentity
5. Gets temporary credentials for that IAM role
```

```yaml
# Pod with IRSA
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-reader
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/s3-reader-role
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      serviceAccountName: s3-reader  # Uses the role above
      containers:
        - name: app
          image: my-app:latest
          # AWS SDK automatically picks up IRSA credentials
          # via AWS_WEB_IDENTITY_TOKEN_FILE + AWS_ROLE_ARN env vars
```

---

## 8.3 HashiCorp Vault — Secrets Management

### What is Vault?

Vault is a secrets management tool that provides secure, dynamic, short-lived credentials rather than static long-lived secrets.

### Vault Architecture

```
┌────────────────────────────────────────────────────────────────┐
│                       VAULT SERVER                              │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │                     API (HTTPS :8200)                     │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                  │
│  ┌────────────┐  ┌────────────┐  ┌────────────────────────┐  │
│  │ Auth       │  │ Secret     │  │ Audit                  │  │
│  │ Methods    │  │ Engines    │  │ Backend                │  │
│  │            │  │            │  │                        │  │
│  │ JWT/OIDC   │  │ KV v2      │  │ File/Syslog           │  │
│  │ Kubernetes │  │ AWS        │  │ Every API call logged  │  │
│  │ AWS IAM    │  │ Database   │  └────────────────────────┘  │
│  │ GitHub     │  │ PKI        │                               │
│  │ LDAP       │  │ Transit    │                               │
│  └────────────┘  └────────────┘                               │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │                    STORAGE BACKEND                        │ │
│  │  Raft (built-in HA) | Consul | DynamoDB | S3             │ │
│  │  ALL data encrypted at rest with master key              │ │
│  └──────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────┘

VAULT IS SEALED ON START — must be unsealed with key shares
Shamir's Secret Sharing: 5 key shares, 3 needed to unseal
(or auto-unseal using AWS KMS / GCP KMS)
```

### Dynamic Secrets — The Key Feature

```
TRADITIONAL (static):               VAULT (dynamic):
app gets DB password                app requests credentials
password never expires              Vault creates temp DB user
password is shared                  user has 1-hour TTL
if leaked → problem forever         if leaked → expires in 1 hour
rotation is painful                 Vault auto-rotates
                                    app never knows the password
                                    
Flow:
App → Vault API: "I need DB creds"
Vault: authenticates app (K8s SA token)
Vault → PostgreSQL: CREATE USER vault_app_xyz WITH PASSWORD '...'
Vault → App: {username: "vault_app_xyz", password: "abc123", lease_ttl: "1h"}
[1 hour later]
Vault → PostgreSQL: DROP USER vault_app_xyz
App requests new creds (or Vault SDK auto-renews)
```

### Vault Kubernetes Integration

```bash
# Enable Kubernetes auth method
vault auth enable kubernetes

vault write auth/kubernetes/config \
    kubernetes_host="https://kubernetes.default.svc" \
    kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

# Create policy
vault policy write app-policy - <<EOF
path "secret/data/production/app/*" {
  capabilities = ["read"]
}
path "database/creds/app-role" {
  capabilities = ["read"]
}
EOF

# Bind policy to Kubernetes service account
vault write auth/kubernetes/role/app-role \
    bound_service_account_names=my-app \
    bound_service_account_namespaces=production \
    policies=app-policy \
    ttl=1h
```

```yaml
# Using Vault Agent Injector (sidecar pattern)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "app-role"
        
        # Inject a secret as a file
        vault.hashicorp.com/agent-inject-secret-config: "secret/data/production/app/config"
        vault.hashicorp.com/agent-inject-template-config: |
          {{- with secret "secret/data/production/app/config" -}}
          DATABASE_URL=postgresql://{{ .Data.data.db_user }}:{{ .Data.data.db_pass }}@db:5432/mydb
          API_KEY={{ .Data.data.api_key }}
          {{- end }}
    spec:
      serviceAccountName: my-app
      containers:
        - name: app
          image: my-app:latest
          # Vault Agent sidecar writes secrets to /vault/secrets/config
          # App reads from that file
          env:
            - name: CONFIG_FILE
              value: /vault/secrets/config
```

---

## 8.4 Zero Trust Security

### Principles

```
OLD MODEL (Perimeter Security):
Internet → Firewall → TRUSTED ZONE (everything inside trusted)
Problem: Once attacker is inside, free reign

ZERO TRUST MODEL:
"Never trust, always verify"
Every request authenticated + authorized, regardless of network location
Even internal traffic is untrusted

ZERO TRUST PILLARS:
1. Identity Verification    — strong auth for every user/device/service
2. Least Privilege Access  — minimum permissions needed, nothing more
3. Microsegmentation       — network isolated per workload
4. Continuous Validation   — re-verify, don't assume
5. Assume Breach           — monitor, detect, respond
```

### Zero Trust in Kubernetes

```
mTLS (mutual TLS) via Service Mesh (Istio/Linkerd):
  Every service has a certificate (SPIFFE/SPIRE)
  Connections require certificate validation BOTH ways
  
  Service A → [presents cert] → Service B
  Service B → [validates cert] → Service A  
  Service B → [presents cert] → Service A
  Service A → [validates cert] → Service B
  Connection established (fully authenticated)
  
RBAC: Every action requires explicit permission
  User/ServiceAccount → Role → Permission
  
Network Policies: Default deny, explicit allow
  All pods isolated by default
  Allow only required paths
  
Pod Security Standards: Restrict container capabilities
  Non-root user, no privileged mode, read-only filesystem
```

---

## 8.5 Multi-Cloud and Cost Optimization

### Multi-Cloud Strategy

```
WHY MULTI-CLOUD:
  ├── Avoid vendor lock-in
  ├── Best-of-breed services (GCP BigQuery, AWS Lambda)
  ├── Regulatory compliance (data residency)
  ├── Disaster recovery (entire cloud region failure)
  └── Negotiation leverage

CHALLENGES:
  ├── Data egress costs ($$$ to transfer between clouds)
  ├── Different APIs, CLIs, IAM models
  ├── Networking complexity
  ├── Operational overhead
  └── Security policy consistency

TOOLS FOR MULTI-CLOUD:
  Kubernetes     — same workload definition across clouds
  Terraform      — provision infrastructure on any cloud
  Crossplane     — K8s-native cloud resource management
  Anthos (GCP)   — multi-cloud K8s management
  Istio          — service mesh across clusters/clouds
```

### Cost Optimization

```
COMPUTE:
  ├── Right-sizing: Use CloudWatch/metrics to find overprovisioned instances
  ├── Reserved Instances: 1-3yr commit → 30-60% savings
  ├── Savings Plans: Compute/EC2, flexible commitment
  ├── Spot Instances: 70-90% off, can be interrupted (use for batch/K8s)
  └── Graviton (ARM): 20% cheaper, 40% better perf than x86

STORAGE:
  ├── S3 Intelligent-Tiering: auto-move to cheaper tier if not accessed
  ├── EBS gp3 vs gp2: gp3 is cheaper, provision IOPS separately
  ├── Delete unattached EBS volumes and old snapshots
  └── S3 Lifecycle rules: move old data to Glacier

NETWORKING:
  ├── VPC Endpoints: avoid NAT Gateway costs for AWS API calls
  ├── CloudFront: cache at edge, reduce origin load
  ├── Same-AZ traffic: free; cross-AZ: $0.01/GB (place DB in same AZ as app)
  └── Data Transfer: same-region free between most services

TAGGING STRATEGY:
  Every resource tagged:
  ├── Environment: production/staging/dev
  ├── Team: backend/frontend/data
  ├── Project: order-service/payment-service
  ├── ManagedBy: terraform/manual
  └── CostCenter: engineering/marketing
  
  → Use Cost Explorer + tag-based cost allocation reports
  → Set budget alerts per team/project
```

---

## 8.6 AWS Interview Questions

### Beginner

**Q: What is the difference between Security Groups and NACLs?**

> Security Groups are **stateful** firewalls at the resource level (EC2, RDS). If you allow inbound on port 443, the return traffic is automatically allowed. NACLs are **stateless** subnet-level firewalls — you must explicitly allow both inbound AND outbound rules. Security Groups are the primary tool; NACLs add an extra layer. Security Groups default to "deny all" with explicit allows; NACLs default to "allow all."

**Q: What is the difference between S3 and EBS?**

> EBS is block storage — like a hard drive, attached to exactly ONE EC2 instance at a time, accessible as a filesystem, used for OS and databases. S3 is object storage — files stored as objects, accessible via HTTP API, virtually unlimited scale, accessed by any number of clients, no filesystem (key-value model). EBS is for "I need a disk for my EC2." S3 is for "I need to store files accessible over the internet or by many services."

### Intermediate

**Q: How does an Application Load Balancer route traffic?**

> ALB (L7) receives HTTPS traffic, terminates TLS, then routes based on: host headers (`api.example.com` vs `app.example.com`), path prefixes (`/api/*` vs `/static/*`), HTTP method, headers, or query parameters. Each rule forwards to a Target Group (EC2 instances, ECS tasks, Lambda, IP addresses, or another ALB). Target groups perform health checks and only route to healthy targets. ALB supports sticky sessions, weighted routing (blue-green canary), and WebSocket.

**Q: Explain VPC peering vs Transit Gateway.**

> VPC Peering connects two VPCs directly (one-to-one), non-transitive (A↔B, B↔C doesn't mean A↔C). Cheap, simple, good for 2-3 VPCs. Transit Gateway is a hub-and-spoke hub — all VPCs/VPNs/Direct Connect attach to one TGW, and TGW routes between them. Transitive by design. For 5+ VPCs, TGW is simpler to manage despite higher cost. TGW also supports multi-account and inter-region peering.

### Advanced

**Q: Design a highly available, disaster-recoverable database architecture on AWS.**

> **Architecture:**
> - Aurora PostgreSQL with Multi-AZ (synchronous replication to standby in different AZ — automatic failover in ~30s)
> - Aurora Global Database for cross-region replication (~1s replication lag)
> - Read replicas in same region for read scaling
> - Automated backups to S3 with 35-day retention, plus manual snapshots before major changes
> - Parameter groups and option groups version-controlled in Terraform
>
> **RTO/RPO:**
> - Same-region AZ failure: RTO ~30s, RPO ~0 (synchronous)
> - Region failure: RTO ~1min (promote global DB reader), RPO ~1s
>
> **Security:**
> - Private subnets only, no public access
> - Security group: only app servers allowed
> - KMS encryption at rest
> - SSL/TLS enforced (parameter `rds.force_ssl=1`)
> - IAM database authentication for application access
> - Audit logging enabled (pgaudit)

---

## 8.7 Security Deep Dive

### Secrets Management Best Practices

```
NEVER:
  ├── Hard-code secrets in source code
  ├── Put secrets in environment variable files committed to git
  ├── Log secrets (check logging framework for accidental logging)
  ├── Pass secrets as command-line arguments (visible in ps aux)
  └── Store secrets in Docker images or Kubernetes ConfigMaps

ALWAYS:
  ├── Use a secrets manager (Vault, AWS Secrets Manager, GCP Secret Manager)
  ├── Rotate secrets regularly (automate it!)
  ├── Use dynamic secrets with TTL when possible
  ├── Audit who accesses secrets (CloudTrail, Vault audit log)
  ├── Use least-privilege IAM to access secrets
  └── Encrypt secrets at rest AND in transit

GIT SECRET DETECTION:
  ├── pre-commit hooks: detect-secrets, gitleaks
  ├── CI/CD scanning: truffleHog, GitGuardian
  ├── If secret committed: immediately rotate it, rewrite git history
  └── git filter-branch or BFG Repo-Cleaner to remove from history
```

### Container Security

```bash
# Dockerfile security best practices

# Use specific digest, not just tag (immutable)
FROM python:3.11.7-slim-bookworm@sha256:abc123...

# Run as non-root
RUN groupadd -r appuser && useradd -r -g appuser appuser

# Don't cache secrets in build layers
# BAD:
RUN pip install -r requirements.txt --index-url https://user:password@pypi.internal/
# GOOD:
RUN --mount=type=secret,id=pip_config,target=/etc/pip.conf \
    pip install -r requirements.txt

# Minimal image = minimal attack surface
# Multi-stage: build stage vs runtime stage
FROM python:3.11 AS builder
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

FROM python:3.11-slim AS runtime  # Much smaller!
COPY --from=builder /usr/local/lib/python3.11 /usr/local/lib/python3.11
COPY --from=builder /usr/local/bin /usr/local/bin
COPY app/ /app/
USER appuser
EXPOSE 8080
CMD ["python", "-m", "uvicorn", "app.main:app", "--host", "0.0.0.0"]

# Kubernetes Pod Security
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  runAsGroup: 3000
  readOnlyRootFilesystem: true    # Can't write to container FS
  allowPrivilegeEscalation: false  # Can't sudo
  capabilities:
    drop: ["ALL"]                  # Drop all Linux capabilities
    add: ["NET_BIND_SERVICE"]      # Only add what's needed (bind <1024 ports)
  seccompProfile:
    type: RuntimeDefault           # Linux system call filtering
```

### Supply Chain Security (SLSA / Sigstore)

```
SOFTWARE SUPPLY CHAIN ATTACKS:
  SolarWinds (2020): Malicious code injected into build process
  log4shell (2021): Vulnerability in widely-used library
  
PROTECTION:
  
  1. SBOM (Software Bill of Materials)
     List of all components + versions in your software
     Tools: Syft, Trivy, SPDX/CycloneDX formats
     
  2. Image Signing (Cosign / Sigstore)
     # Sign image after build
     cosign sign --key cosign.key myregistry/myapp:v1.0.0
     
     # Verify before deploy (in admission controller)
     cosign verify --key cosign.pub myregistry/myapp:v1.0.0
     
  3. Vulnerability Scanning
     # Scan at build time
     trivy image myapp:v1.0.0 --exit-code 1 --severity HIGH,CRITICAL
     
     # Scan running containers continuously
     # Tools: Falco (runtime), Aqua, Sysdig, Snyk
     
  4. SLSA Framework (Supply chain Levels for Software Artifacts)
     Level 1: Build scripted, provenance generated
     Level 2: Build service used, signed provenance
     Level 3: Build service hardened, provenance authenticated
     Level 4: Two-party review, hermetic builds
```

---

## 8.8 DevSecOps — Security in CI/CD

```
PIPELINE WITH SECURITY GATES:

Code Push
    │
    ▼
[Pre-commit hooks]
├── detect-secrets (no secrets committed)
├── gitleaks (git history scan)
└── SAST: semgrep, bandit (quick static analysis)
    │
    ▼
[CI: Pull Request]
├── SAST: SonarQube, Checkmarx, Semgrep (full scan)
├── Dependency scanning: Snyk, OWASP Dependency Check
├── License compliance: FOSSA
└── Code review (human, 2-person rule for prod)
    │
    ▼
[CI: Build]
├── Docker build
├── Trivy image scan (HIGH/CRITICAL = FAIL)
├── Cosign image signing
└── Generate SBOM (Syft)
    │
    ▼
[CD: Staging Deploy]
├── DAST: OWASP ZAP (dynamic scan against running app)
├── Integration tests
└── Penetration testing (periodic, not every deploy)
    │
    ▼
[CD: Production Deploy] (with approval gate)
├── Cosign verify (ensure signed image)
├── OPA/Gatekeeper policies enforced
├── Blue-green or canary deployment
└── Runtime security: Falco monitoring
    │
    ▼
[Runtime Monitoring]
├── Falco: detect unexpected syscalls/behaviors
├── GuardDuty: AWS threat detection
├── CloudTrail: audit every API call
└── SIEM: centralized security event correlation
```

---

## 8.9 Final System Design: Production-Grade Platform

```
COMPLETE PRODUCTION ARCHITECTURE:

                          ┌────────────────┐
                          │   Route 53     │
                          │   (DNS + HC)   │
                          └───────┬────────┘
                                  │
                          ┌───────▼────────┐
                          │   CloudFront   │
                          │   (CDN + WAF)  │
                          └───────┬────────┘
                                  │
              ┌───────────────────▼──────────────────┐
              │          Application Load Balancer    │
              │          (ACM SSL, multi-AZ)          │
              └──────┬────────────────────────┬───────┘
                     │                        │
          ┌──────────▼──────┐      ┌──────────▼──────┐
          │   EKS Cluster   │      │   EKS Cluster   │
          │   us-east-1     │      │   us-west-2     │
          │                 │      │   (DR/standby)  │
          │  [Istio mesh]   │      │                 │
          │  [ArgoCD]       │      │  [ArgoCD]       │
          │  [Prometheus]   │      │  [Prometheus]   │
          │  [app pods]     │      │  [app pods]     │
          └────────┬────────┘      └─────────────────┘
                   │
         ┌─────────┼──────────┐
         │         │          │
    ┌────▼───┐ ┌───▼───┐ ┌───▼─────────┐
    │Aurora  │ │ElastiC│ │  Kafka MSK  │
    │Global  │ │ache   │ │  (events)   │
    │(PG)    │ │(Redis)│ │             │
    └────────┘ └───────┘ └─────────────┘

GitOps Flow:
Developer → Git PR → Review → Merge
→ GitHub Actions CI (test+build+scan+sign)
→ Push image to ECR
→ Update Helm values in GitOps repo
→ ArgoCD detects diff → auto-deploys to staging
→ Manual approval for production
→ ArgoCD deploys to production
→ Prometheus + Grafana monitoring
→ PagerDuty if SLO breach
→ Runbook → Resolution → Post-mortem
```

---

## 8.10 Career Interview Preparation

### System Design Questions

**Q: Design a CI/CD pipeline for a microservices application with 50 services.**

> **Answer Framework:**
> 1. **Source Control**: Mono-repo (Nx/Turborepo for affected-only builds) or poly-repo (per-service pipelines)
> 2. **CI per service**: On PR, run tests → SAST → build Docker image → scan → push to ECR with SHA tag
> 3. **Artifact versioning**: Immutable image tags (git SHA), Helm chart versioning
> 4. **GitOps CD**: ArgoCD app-of-apps pattern, environment promotions via PR to GitOps repo
> 5. **Deployment strategy**: Blue-green for stateless, canary for high-risk changes (Argo Rollouts)
> 6. **Testing gates**: Integration tests in staging, smoke tests post-deploy to prod
> 7. **Rollback**: ArgoCD one-click rollback (reverts Helm release), plus feature flags for instant disable
> 8. **Secrets**: External Secrets Operator pulling from Vault/AWS Secrets Manager
> 9. **Observability**: Deploy updates trigger Grafana annotations, watch error rate for 15min post-deploy

**Q: How would you handle a production outage?**

> **Incident Response Process:**
> 1. **Detect**: Alert fires via PagerDuty from Prometheus/CloudWatch
> 2. **Acknowledge**: On-call engineer ACKs within SLA (5min for P1)
> 3. **Communicate**: Post in #incidents Slack, update status page (Atlassian Statuspage)
> 4. **Triage**: What's broken? What's the blast radius? How many users affected?
> 5. **Mitigate first**: Rollback deployment, increase capacity, failover to DR — stop the bleeding BEFORE root cause analysis
> 6. **Root cause**: Logs → traces → metrics → code → infra
> 7. **Fix**: Deploy fix, verify metrics return to normal
> 8. **Post-mortem**: Blameless, within 48h, 5-whys, action items with owners and due dates
> 9. **Prevention**: Implement action items (alerting improvements, runbooks, code fixes, architecture changes)

### DevOps Behavioral Questions

**Q: Tell me about a time you improved system reliability.**

> Framework: Situation → Task → Action → Result
> Example: "Our checkout service had 99.5% availability (4.4h downtime/month). I implemented health checks + HPA + PodDisruptionBudgets to ensure rolling updates never brought us below 2 replicas. Added circuit breakers to payment provider calls. Set up synthetic monitoring. Result: Improved to 99.95% availability (4.4h → 26min/month downtime). Reduced MTTR from 45min to 8min with runbook automation."

### Key Technical Depth Areas for Senior Roles

```
DEPTH QUESTIONS SENIOR DEVOPS/SRE MUST ANSWER:

1. "How does iptables implement Kubernetes Services?"
   → iptables DNAT rules, kube-proxy modes (iptables vs IPVS)

2. "Explain etcd Raft consensus and why split-brain matters"
   → Leader election, quorum (n/2+1), 3 vs 5 nodes tradeoffs

3. "How does Kubernetes scheduler choose a node?"
   → Filtering (resource fit, taints, affinity), Scoring (spread, pack),
     Binding, watch mechanism

4. "How do you debug a pod that's OOMKilled?"
   → kubectl describe pod, check limits vs requests, heapster/metrics-server,
     analyze memory leak in app (pprof for Go, jmap for Java)

5. "What causes etcd compaction and backup?"
   → MVCC keeps all historical revisions, compact removes old revisions,
     backup: etcdctl snapshot save, restore with etcdctl snapshot restore

6. "How does Prometheus TSDB work under the hood?"
   → Chunks, head block, WAL, compaction, index structure

7. "Explain VPC routing when an EC2 in a private subnet calls S3"
   → Private subnet → route table → NAT Gateway (public subnet) →
     IGW → S3 public endpoint
     OR: VPC Endpoint for S3 (stays in AWS network, cheaper, faster)
```
