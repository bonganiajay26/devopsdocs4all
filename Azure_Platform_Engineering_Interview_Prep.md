# Azure Platform Engineering — Interview Prep Guide
> **Role Focus:** Platform Lead | Architecture | Security | Operations | Leadership
> **Strategy:** Short answers + real examples + tradeoffs + measurable impact

---

## TABLE OF CONTENTS
1. [Azure Platform Engineering](#1-azure-platform-engineering)
2. [GitHub Actions & CI/CD](#2-github-actions--cicd)
3. [Kubernetes / AKS](#3-kubernetes--aks)
4. [Observability & Monitoring](#4-observability--monitoring)
5. [Security & Compliance](#5-security--compliance)
6. [Terraform / IaC](#6-terraform--iac)
7. [Production Incidents](#7-production-incidents)
8. [Leadership & Ownership](#8-leadership--ownership)
9. [MLOps / AI Questions](#9-mlops--ai-questions)

---

## 1. AZURE PLATFORM ENGINEERING

---

### Q1: How would you design secure Dev/Staging/Prod environments in Azure?

**Answer:**
I use a **Management Group + Subscription-per-environment** model. Each environment is isolated at the subscription boundary — not just resource groups — so RBAC, billing, policy, and blast radius are all separated.

```
Azure Tenant
└── Management Group: Platform
    ├── Management Group: Non-Production
    │   ├── Subscription: Dev
    │   └── Subscription: Staging
    └── Management Group: Production
        └── Subscription: Prod
```

**Key design decisions:**

| Concern | Dev | Staging | Prod |
|---|---|---|---|
| RBAC | Dev team = Contributor | Only CI/CD SPN | Only CI/CD SPN + On-call |
| Policy | Warn only | Enforce + audit | Enforce strict |
| Networking | Spoke VNet → Dev Hub | Spoke → Staging Hub | Spoke → Prod Hub (Private Endpoints only) |
| Key Vault | Dev secrets | Staging secrets | Prod secrets (RBAC, not policy) |
| Azure Monitor | Basic | Full | Full + Alerts + PagerDuty |
| Cost | Dev auto-shutdown | Scaled down | HA + Autoscale |

**Architecture Diagram:**
```
┌─────────────────────────────────────────────────────────┐
│                    Azure Tenant                         │
│                                                         │
│  ┌──────────────────────────────────────────────────┐  │
│  │          Hub VNet (Connectivity Sub)              │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────┐   │  │
│  │  │ Firewall │  │ Bastion  │  │ ExpressRoute │   │  │
│  │  └──────────┘  └──────────┘  └──────────────┘   │  │
│  └──────────────────────────────────────────────────┘  │
│         │ VNet Peering         │ VNet Peering           │
│  ┌──────┴───┐         ┌────────┴──────┐                │
│  │ Dev Spoke│         │ Staging Spoke │                │
│  │ 10.1.0.0 │         │  10.2.0.0    │                │
│  └──────────┘         └───────────────┘                │
│                                                         │
│  ┌──────────────────────────────────────────────────┐  │
│  │                 Prod Spoke                        │  │
│  │   AKS Subnet  |  App Subnet  |  DB Subnet        │  │
│  │   10.3.1.0    |  10.3.2.0   |  10.3.3.0         │  │
│  └──────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

**Impact I've seen:** Separating subscriptions reduced our blast-radius incidents by ~70% — a dev engineer couldn't accidentally delete prod resources.

---

### Q2: AKS vs App Services — when would you choose each?

**Answer (Decision Framework):**

| Factor | AKS | App Service |
|---|---|---|
| Workload | Microservices, ML, stateful apps | REST APIs, web apps, functions |
| Team maturity | K8s expertise available | Smaller teams, fast delivery |
| Control needed | Full control (custom runtimes, sidecars) | PaaS, managed infra |
| Scaling | HPA, KEDA, cluster autoscaler | App Service Plan autoscale |
| Cost | Higher (node pools) | Lower for small apps |
| Multi-tenancy | Namespaces + RBAC | App Service Environments |
| Networking | Private clusters, CNI control | VNet Integration + Private Endpoints |

**When I choose AKS:**
- 10+ microservices needing independent scaling
- ML model serving (GPU node pools, custom runtimes)
- Multi-tenant SaaS with namespace isolation
- Event-driven workloads using KEDA

**When I choose App Service:**
- Simple REST APIs or internal tools
- Teams without Kubernetes expertise
- Rapid prototyping → production path
- Apps that don't need custom sidecar containers

**Real example:** We migrated a batch ML pipeline from App Service to AKS because we needed KEDA for event-driven scaling from Azure Service Bus, and GPU node pools for inference. Cost went up 20% but throughput improved 4x.

---

### Q3: How do you implement private networking and environment isolation?

**Architecture:**
```
Internet
    │
    ▼
┌──────────────────┐
│  Azure Front Door│ (WAF + Global LB)
│  or App Gateway  │
└────────┬─────────┘
         │ (HTTPS only, TLS termination)
         ▼
┌──────────────────────────────────────────┐
│              Hub VNet                    │
│  ┌─────────────┐    ┌─────────────────┐ │
│  │ Azure FW    │    │  Bastion Host   │ │
│  │ (FQDN rules)│    │  (no public IP) │ │
│  └──────┬──────┘    └─────────────────┘ │
└─────────┼────────────────────────────────┘
          │ VNet Peering
┌─────────┴────────────────────────────────┐
│            Prod Spoke VNet               │
│  ┌────────────────┐  ┌────────────────┐ │
│  │   AKS Subnet   │  │  DB Subnet     │ │
│  │  (Private API  │  │ (Private EP    │ │
│  │   Server)      │  │  only)         │ │
│  └────────────────┘  └────────────────┘ │
│  ┌────────────────────────────────────┐ │
│  │  Private DNS Zone                  │ │
│  │  *.privatelink.database.azure.com  │ │
│  └────────────────────────────────────┘ │
└──────────────────────────────────────────┘
```

**Key controls I implement:**
- **Private Endpoints** for all PaaS (Key Vault, ACR, Storage, SQL, CosmosDB)
- **Private DNS Zones** linked to spoke VNets
- **AKS private cluster** — API server not reachable from internet
- **Azure Firewall** with FQDN-based egress rules
- **NSGs on every subnet** with deny-all default + explicit allow rules
- **Service Endpoints** disabled in favor of Private Endpoints
- **No public IPs** on any VM or AKS node — all access via Bastion

**Environment isolation layers:**
1. Subscription isolation (RBAC boundary)
2. VNet isolation (no peering between Dev and Prod)
3. Private DNS resolution per environment
4. Azure Policy: deny public endpoints in Prod

---

### Q4: How do you structure Terraform modules for large platforms?

**Answer:**

```
infra/
├── modules/                    # Reusable, versioned modules
│   ├── aks/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── README.md
│   ├── networking/
│   ├── key-vault/
│   ├── acr/
│   └── monitoring/
├── environments/               # Environment-specific configs
│   ├── dev/
│   │   ├── main.tf             # Calls modules with dev values
│   │   ├── terraform.tfvars
│   │   └── backend.tf          # Remote state in dev SA
│   ├── staging/
│   └── prod/
├── platform/                   # Foundation resources
│   ├── subscriptions/
│   ├── policies/
│   └── management-groups/
└── scripts/
    ├── plan.sh
    └── apply.sh
```

**Module design principles:**
- Modules are **versioned** via Git tags: `source = "git::https://github.com/org/tf-modules.git//aks?ref=v1.2.0"`
- Modules have **sensible defaults** with override capability
- One `terraform apply` per environment layer (networking → AKS → apps)
- **No hardcoded values** — all via variables with validation blocks
- **Output everything** — modules expose all resource IDs for composition

**Variable validation example:**
```hcl
variable "environment" {
  type = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}
```

---

### Q5: How do you manage secrets securely using Azure Key Vault?

**Architecture:**
```
Application (AKS Pod)
      │
      │ Managed Identity (no password)
      ▼
Azure Key Vault
      │
      ├── Secrets  (DB passwords, API keys)
      ├── Certs    (TLS, mTLS)
      └── Keys     (encryption keys, CMK)
```

**Implementation layers:**

1. **Access:** Managed Identity only — no service principal passwords
2. **RBAC model:** Key Vault RBAC (not legacy Access Policies)
3. **AKS integration:** CSI Secret Store driver mounts secrets as volumes
4. **Rotation:** Key Vault + Event Grid triggers Azure Function for auto-rotation
5. **Audit:** Diagnostic logs → Log Analytics → Alert on unusual access
6. **Networking:** Private Endpoint only — no public access

**CSI driver in AKS:**
```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: app-secrets
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    useVMManagedIdentity: "true"
    userAssignedIdentityID: "<client-id>"
    keyvaultName: "prod-kv-platform"
    objects: |
      array:
        - |
          objectName: db-password
          objectType: secret
```

**Never do:**
- Hardcode secrets in Helm values or ConfigMaps
- Use SP client secrets in CI/CD — use Workload Identity Federation instead
- Store secrets in environment variables via Kubernetes Deployment YAML

---

## 2. GITHUB ACTIONS & CI/CD

---

### Q1: Design a production-grade GitHub Actions pipeline

**Pipeline Architecture:**
```
Developer Push
      │
      ▼
┌─────────────────────────────────────────────────┐
│                  PR Pipeline                    │
│  lint → unit-test → sast → build → scan-image  │
└─────────────────────────┬───────────────────────┘
                          │ Merge to main
                          ▼
┌─────────────────────────────────────────────────┐
│              CI Pipeline (main)                 │
│  build → test → scan → publish-image → sign    │
└─────────────────────────┬───────────────────────┘
                          │
              ┌───────────┼───────────┐
              ▼           ▼           ▼
           Deploy      Deploy      Deploy
            Dev       Staging      Prod
          (auto)     (auto+      (manual
                    smoke test)  approval)
```

**Full pipeline YAML (condensed):**
```yaml
name: Platform CD

on:
  push:
    branches: [main]

env:
  IMAGE: myacr.azurecr.io/myapp
  CHART: ./helm/myapp

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      id-token: write      # OIDC for Azure login
      contents: read
    steps:
      - uses: actions/checkout@v4

      - name: OIDC Login to Azure
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Build & Push
        run: |
          TAG=${{ github.sha }}
          docker build -t $IMAGE:$TAG .
          docker push $IMAGE:$TAG

      - name: Trivy Scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: $IMAGE:${{ github.sha }}
          exit-code: '1'
          severity: CRITICAL,HIGH

      - name: Sign Image (Cosign)
        run: cosign sign --key env://COSIGN_KEY $IMAGE:${{ github.sha }}

  deploy-dev:
    needs: build
    environment: dev
    runs-on: ubuntu-latest
    steps:
      - name: Helm Upgrade
        run: |
          helm upgrade --install myapp $CHART \
            --set image.tag=${{ github.sha }} \
            --namespace dev

  deploy-staging:
    needs: deploy-dev
    environment: staging
    # Auto-deploy with smoke tests
    ...

  deploy-prod:
    needs: deploy-staging
    environment: production   # ← requires GitHub Environment approval
    ...
```

---

### Q2: How do you implement rollback strategies?

**Three-tier rollback approach:**

```
Failure Detected
       │
       ├─► Helm Rollback (< 5 min)
       │   helm rollback myapp 0 -n prod
       │
       ├─► Image Tag Rollback (CI re-trigger)
       │   Re-deploy previous known-good SHA
       │
       └─► Git Revert + Re-deploy (source of truth)
           git revert <commit> → triggers pipeline
```

**Automated rollback in pipeline:**
```yaml
deploy-prod:
  steps:
    - name: Helm Deploy
      id: deploy
      run: helm upgrade --install myapp $CHART --atomic --timeout 5m

    - name: Smoke Test
      id: smoke
      run: ./scripts/smoke-test.sh

    - name: Rollback on failure
      if: failure() && steps.deploy.outcome == 'success'
      run: |
        helm rollback myapp 0 --namespace prod
        echo "::error::Smoke test failed — rolled back to previous release"
```

**Key patterns:**
- `--atomic` flag in Helm: auto-rollback if deployment fails
- Keep last 5 Helm releases: `--history-max 5`
- Feature flags for risky changes: ship code without activating feature
- Blue-Green: swap traffic via Kubernetes Service selector

---

### Q3: How do you secure CI/CD pipelines?

**Security layers:**
```
┌─────────────────────────────────────────────┐
│            CI/CD Security Model             │
│                                             │
│  Code Signing        → Cosign + OIDC        │
│  Secret Management   → GitHub OIDC → AKV   │
│  Supply Chain        → SBOM + Trivy         │
│  Runner Hardening    → Ephemeral runners    │
│  Branch Protection   → Required reviews     │
│  Least Privilege     → Scoped RBAC          │
└─────────────────────────────────────────────┘
```

**My security checklist:**
- ✅ **OIDC instead of long-lived secrets** — no Azure SP passwords stored in GitHub
- ✅ **Environment protection rules** — prod requires 2 approvers
- ✅ **Branch protection** — no force push, required status checks
- ✅ **SAST** — CodeQL or Semgrep on every PR
- ✅ **Container scanning** — Trivy, fail on CRITICAL
- ✅ **Image signing** — Cosign with GitHub OIDC
- ✅ **SBOM generation** — Syft on every build
- ✅ **Dependency review** — block PRs with known CVEs in deps
- ✅ **Ephemeral runners** — no persistent state between runs
- ✅ **Minimal permissions** — `permissions:` block in every job

---

### Q4: How do you implement approval gates and promotion across environments?

**GitHub Environments model:**
```
Dev (auto)  →  Staging (auto + smoke)  →  Prod (manual approval)
                                              │
                                     Required reviewers:
                                     - Platform Lead
                                     - On-call Engineer
                                     Wait timer: 0 min
                                     Deployment branch: main only
```

**Beyond GitHub Environments:**
- **Smoke tests** as automated gates between staging → prod
- **Performance regression check:** if p99 latency > threshold, block promotion
- **Change Freeze windows:** GitHub Environment deployment branch rules
- **Canary gate:** 5% traffic → metrics check → 100% traffic

```yaml
# .github/environments/production
# Set via UI or API:
required_reviewers:
  - team: platform-leads
wait_timer: 0
deployment_branch_policy:
  protected_branches: true
```

---

### Q5: How do you prevent unsafe releases?

**Gates I enforce:**
1. **Pre-merge:** SAST, unit tests, lint, dependency review
2. **Pre-build:** secrets detection (Gitleaks), license check
3. **Post-build:** container scan (Trivy CRITICAL = block), SBOM
4. **Pre-deploy staging:** integration tests
5. **Pre-deploy prod:** smoke test on staging, manual approval, change window check
6. **Post-deploy prod:** automated smoke test, rollback on failure
7. **Policy as Code:** OPA/Gatekeeper in AKS (block non-signed images)

```yaml
# OPA Gatekeeper policy — only allow signed images
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequireImageSignature
metadata:
  name: require-signed-images
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
  parameters:
    registry: "myacr.azurecr.io"
```

---

## 3. KUBERNETES / AKS

---

### Q1: How do you secure Kubernetes clusters?

**AKS Security Architecture:**
```
┌─────────────────────────────────────────────────────────┐
│                   AKS Security Layers                   │
│                                                         │
│  L1: Control Plane   Private cluster, AAD auth only    │
│  L2: Node Security   CIS-hardened, auto-upgrade        │
│  L3: Network         Calico NPols, Azure CNI            │
│  L4: Workload        Pod Security Standards, Gatekeeper │
│  L5: Image           Only signed, scanned images        │
│  L6: Secrets         CSI Key Vault, no plain K8s secrets│
│  L7: RBAC            Namespace-scoped, least privilege  │
└─────────────────────────────────────────────────────────┘
```

**Top 10 AKS security controls I always implement:**

| Control | Implementation |
|---|---|
| Private cluster | API server not internet-accessible |
| AAD integration | Azure AD + RBAC, no local accounts |
| Workload Identity | No pod service account tokens for Azure |
| Network Policies | Deny all ingress/egress by default |
| Pod Security Standards | Restricted profile on prod namespaces |
| OPA Gatekeeper | Image signing, no privileged containers |
| Node auto-upgrade | Security patches applied automatically |
| Defender for Containers | Runtime threat detection |
| Audit logs | API server logs → Log Analytics |
| No root containers | `runAsNonRoot: true` enforced |

---

### Q2: How do you handle ingress, TLS, secrets, and namespace isolation?

**Ingress Architecture:**
```
Internet
    │
    ▼
Azure Front Door / App Gateway (WAF)
    │
    ▼
NGINX Ingress Controller (Internal LB)
    │
    ├── /api/v1  → Service: api-svc (namespace: backend)
    ├── /auth    → Service: auth-svc (namespace: backend)
    └── /        → Service: frontend-svc (namespace: frontend)
```

**TLS — cert-manager with Let's Encrypt:**
```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: prod-tls
  namespace: backend
spec:
  secretName: prod-tls-secret
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - api.myapp.com
```

**Namespace isolation with NetworkPolicy:**
```yaml
# Deny all traffic within namespace by default
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: backend
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
---
# Allow only ingress controller to reach backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: api
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
```

---

### Q3: How do you troubleshoot pod failures in production?

**My systematic approach:**
```
Pod Failing
     │
     ├─ Step 1: kubectl get pods -n <ns> -o wide
     │          → Check STATUS, RESTARTS, NODE
     │
     ├─ Step 2: kubectl describe pod <pod> -n <ns>
     │          → Events section: image pull? OOM? Liveness probe?
     │
     ├─ Step 3: kubectl logs <pod> -n <ns> --previous
     │          → App-level errors
     │
     ├─ Step 4: kubectl top pod <pod> -n <ns>
     │          → CPU/memory usage vs limits
     │
     ├─ Step 5: kubectl get events -n <ns> --sort-by='.lastTimestamp'
     │          → Node-level issues
     │
     └─ Step 6: kubectl exec -it <pod> -- /bin/sh
                → Network connectivity, DNS resolution
```

**Common failures and fixes:**

| Symptom | Cause | Fix |
|---|---|---|
| CrashLoopBackOff | App error, OOM | Check logs, increase memory limit |
| ImagePullBackOff | Wrong tag, ACR auth | Check imagePullSecret, ACR RBAC |
| Pending | Node has no capacity | Check PodDisruptionBudget, add nodes |
| OOMKilled | Memory limit too low | `kubectl top pod`, increase limits |
| Liveness failing | App stuck | Check probe path, adjust thresholds |

---

### Q4: How do you scale workloads and clusters?

**Scaling Architecture:**
```
Workload Scaling:
  KEDA (event-driven)  ─┐
  HPA (CPU/memory)     ─┼─► Pod Replicas
  VPA (right-sizing)   ─┘

Cluster Scaling:
  Cluster Autoscaler ──► Node Pools
  
Node Pool Strategy:
  System Pool:  2 nodes, D4s v3, no workloads
  App Pool:     3-20 nodes, D8s v3, autoscale
  GPU Pool:     0-5 nodes, NC6s v3, autoscale (ML)
  Spot Pool:    0-50 nodes, Spot VMs, batch jobs
```

**KEDA for Service Bus scaling:**
```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: worker-scaler
spec:
  scaleTargetRef:
    name: worker-deployment
  minReplicaCount: 1
  maxReplicaCount: 50
  triggers:
    - type: azure-servicebus
      metadata:
        queueName: job-queue
        messageCount: "10"   # 1 pod per 10 messages
        connectionFromEnv: SB_CONNECTION
```

---

### Q5: How do you handle zero-downtime deployments?

**Strategy matrix:**

| Strategy | Downtime | Risk | Rollback Speed |
|---|---|---|---|
| Recreate | Yes | Low effort | Slow |
| RollingUpdate | None | Medium | Medium |
| Blue-Green | None | Low | Instant |
| Canary | None | Very low | Instant |

**My default — RollingUpdate with safety settings:**
```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0        # Never take pods down before new ones up
      maxSurge: 25%            # Spin up 25% extra during rollout
  minReadySeconds: 30          # Wait 30s before marking pod ready
  progressDeadlineSeconds: 600
```

**Readiness probe is critical:**
```yaml
readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 3
```

**PodDisruptionBudget:**
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: api
```

---

## 4. OBSERVABILITY & MONITORING

---

### Q1: Difference between logs, metrics, and traces?

```
Observability Pillars
        │
        ├── LOGS       → "What happened?" 
        │               Structured events (JSON)
        │               Tool: Azure Monitor, Loki, ELK
        │               Use: Debugging, audit, compliance
        │
        ├── METRICS    → "How is the system performing?"
        │               Numeric, time-series
        │               Tool: Prometheus, Azure Monitor Metrics
        │               Use: Alerting, dashboards, SLOs
        │
        └── TRACES     → "Where did time go?"
                        Distributed request tracking
                        Tool: Jaeger, Tempo, Azure App Insights
                        Use: Latency root cause, dependency mapping
```

**Combined example — API request failure:**
- **Log:** `ERROR: DB timeout after 5s | user_id=123 | request_id=abc`
- **Metric:** `http_requests_total{status="500"} = 47` (spike)
- **Trace:** API → Auth (2ms) → DB (5001ms — timeout here)

---

### Q2: How would you implement OpenTelemetry?

**Architecture:**
```
Application
    │
    │ OTel SDK (auto-instrumentation)
    ▼
OTel Collector (DaemonSet in AKS)
    │
    ├── Traces  → Azure Monitor / Jaeger / Tempo
    ├── Metrics → Prometheus / Azure Monitor
    └── Logs    → Azure Monitor Logs / Loki
```

**OTel Collector config (key parts):**
```yaml
receivers:
  otlp:
    protocols:
      grpc: { endpoint: 0.0.0.0:4317 }
      http: { endpoint: 0.0.0.0:4318 }

processors:
  batch: {}
  memory_limiter:
    limit_mib: 400
  resource:
    attributes:
      - key: environment
        value: production
        action: insert

exporters:
  azuremonitor:
    instrumentation_key: ${APPINSIGHTS_KEY}
  prometheus:
    endpoint: "0.0.0.0:8889"

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [azuremonitor]
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [prometheus]
```

**Auto-instrumentation in AKS (no code change):**
```yaml
# Using OTel Operator
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: auto-instrumentation
spec:
  java:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-java:latest
  python:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-python:latest
```

---

### Q3: How do you design dashboards for production systems?

**Dashboard hierarchy:**
```
Level 1: Executive (SLO Dashboard)
  → Availability %, Error Budget remaining, P99 latency

Level 2: Service Health (per-service)
  → RED metrics: Rate, Errors, Duration

Level 3: Infrastructure
  → Node CPU/Mem, Pod restarts, PVC usage

Level 4: Debug (on-demand)
  → Traces, detailed logs, query explorer
```

**The RED Method (what I use for every service):**
- **Rate:** requests per second
- **Errors:** error rate percentage
- **Duration:** p50, p95, p99 latency

**SLO Dashboard example:**
```
┌────────────────────────────────────────────┐
│  Service: Payment API  |  Period: 30 days  │
├──────────────┬─────────────┬───────────────┤
│ Availability │ Error Budget│ P99 Latency   │
│   99.94%     │  71% left   │  245ms        │
│   (SLO:99.9%)│             │  (SLO: <500ms)│
└──────────────┴─────────────┴───────────────┘
```

---

### Q4: How do you reduce alert noise?

**Alert noise reduction strategy:**

```
Alert Signals
     │
     ├─ Symptom-based alerting (not cause-based)
     │   ✅ "Error rate > 5% for 5 min"
     │   ❌ "CPU > 80%" (may not affect users)
     │
     ├─ SLO-based alerting (burn rate)
     │   Alert when error budget burns too fast
     │   Fast burn: 2% budget in 1h → Page immediately
     │   Slow burn: 5% budget in 6h → Create ticket
     │
     ├─ Alert deduplication
     │   Alertmanager: group_by [service, env]
     │
     ├─ Inhibition rules
     │   Node down → inhibit all pod alerts on that node
     │
     └─ Review cadence
        Weekly: close noisy alerts
        Monthly: SLO review
```

**SLO burn rate alerting (Prometheus):**
```yaml
# Page if consuming error budget 14x faster than normal (1h window)
- alert: HighErrorBudgetBurn
  expr: |
    (
      sum(rate(http_requests_total{status=~"5.."}[1h])) /
      sum(rate(http_requests_total[1h]))
    ) > 14 * (1 - 0.999)
  for: 5m
  labels:
    severity: critical
```

---

### Q5: How do you make telemetry PII-safe?

**PII protection layers:**

```
Data Collection
      │
      ├─ 1. At instrumentation: never log PII fields
      │      ✅ log user_id  ❌ log email, name, SSN
      │
      ├─ 2. At OTel Collector: attribute scrubbing
      │      Redact known PII fields before export
      │
      ├─ 3. At storage: RBAC + encryption
      │      Log Analytics workspace: column-level security
      │
      └─ 4. Retention: 30 days logs, 90 days metrics
             Purge API for GDPR right-to-erasure
```

**OTel Collector PII scrubbing:**
```yaml
processors:
  attributes/pii:
    actions:
      - key: http.request.header.authorization
        action: delete
      - key: user.email
        action: hash
      - key: db.statement
        pattern: 'password\s*=\s*\S+'
        action: update
        value: "password=REDACTED"
```

---

## 5. SECURITY & COMPLIANCE

---

### Q1: How do you implement secure-by-default infrastructure?

**Secure-by-default principles I enforce:**

```
Azure Policy (prevent)
      │
      ├── Deny public endpoints on PaaS
      ├── Require CMK for storage encryption
      ├── Deny non-compliant SKUs
      └── Require tags on all resources

Defender for Cloud (detect)
      │
      ├── Secure Score tracking
      ├── Vulnerability assessment
      └── Threat protection

Network (restrict)
      │
      ├── Default deny NSG rules
      ├── Private Endpoints only
      └── Azure Firewall egress control
```

**Azure Policy Initiative I deploy to every sub:**
- Deny public IP on VMs
- Deny unencrypted storage accounts
- Require Log Analytics agent
- Deny non-approved regions
- Require HTTPS only on web apps

---

### Q2: How do you handle managed identities?

**Identity hierarchy:**
```
Old Way (BAD):
  App → Service Principal → Client Secret → Azure Resource
  Problem: Secret rotation, secret leakage

New Way (GOOD):
  AKS Pod → Workload Identity → Federated Credential → Azure Resource
  No secrets at all!
```

**AKS Workload Identity setup:**
```bash
# 1. Create User-Assigned Managed Identity
az identity create -n app-identity -g rg-prod

# 2. Assign RBAC
az role assignment create \
  --assignee $CLIENT_ID \
  --role "Key Vault Secrets User" \
  --scope /subscriptions/.../vaults/prod-kv

# 3. Federated credential
az identity federated-credential create \
  --name aks-federated \
  --identity-name app-identity \
  --issuer $AKS_OIDC_ISSUER \
  --subject system:serviceaccount:backend:api-sa
```

**Pod annotation:**
```yaml
serviceAccount:
  annotations:
    azure.workload.identity/client-id: "<CLIENT_ID>"
```

No passwords. No rotation. Automatic credential management.

---

### Q3: How do you implement DLP for logs?

**Data Loss Prevention layers:**

| Layer | Tool | Action |
|---|---|---|
| Instrumentation | Code review + linting | Don't log PII fields |
| Collector | OTel attribute processor | Hash/redact before export |
| Storage | Log Analytics | Column-level security, purge API |
| Query access | RBAC | Only authorized users can query PII tables |
| Export | Azure Monitor export rules | Filter sensitive tables from export |
| Audit | Azure Monitor | Alert on unusual query volume |

**Log Analytics column security:**
```kql
// Restrict sensitive columns to PII_Readers role
// Regular engineers see masked data
_ColumnSecurity
| where Table == "AppRequests"
| where Column == "user_email"
```

---

### Q4: How do you audit infrastructure changes?

**Audit trail architecture:**
```
Any Change
     │
     ├─ Azure Activity Log (automatic)
     │   → Who, what, when, from where
     │   → Retention: 90 days default, extend to Log Analytics
     │
     ├─ Azure Resource Graph
     │   → Point-in-time resource state queries
     │
     ├─ Terraform state + Git history
     │   → What should exist (source of truth)
     │
     └─ Alerts
         → Alert on sensitive operations:
           - Role assignment changes
           - Key Vault access policy changes
           - NSG rule modifications
           - Subscription-level changes
```

**KQL alert for privilege escalation:**
```kql
AzureActivity
| where OperationNameValue == "Microsoft.Authorization/roleAssignments/write"
| where ActivityStatusValue == "Success"
| project TimeGenerated, Caller, Properties
| order by TimeGenerated desc
```

---

### Q5: How do you enforce policy checks in CI/CD?

**Policy enforcement pipeline:**
```
Code → PR
         │
         ├─ Checkov (Terraform IaC scanning)
         ├─ Trivy (K8s manifest scanning)
         ├─ Conftest + OPA (custom policies)
         └─ tfsec (Terraform security)

Deploy → Pre-deploy
         │
         ├─ Azure Policy (deny at ARM level)
         └─ Gatekeeper (deny at K8s admission)
```

**Conftest OPA policy example:**
```rego
# policy/deny_public_ip.rego
package main

deny[msg] {
  input.resource.azurerm_public_ip
  msg = "Public IPs are not allowed in production"
}

deny[msg] {
  container := input.spec.containers[_]
  container.securityContext.privileged == true
  msg = sprintf("Container '%v' must not run as privileged", [container.name])
}
```

**In CI:**
```yaml
- name: Policy Check
  run: |
    conftest test terraform/prod/ --policy policy/
    trivy config --exit-code 1 --severity HIGH,CRITICAL k8s/
```

---

## 6. TERRAFORM / IAC

---

### Q1: Remote state management?

**State architecture:**
```
Each Environment = Separate State File

Azure Storage Account (state-sa-platform)
├── Container: tfstate
│   ├── dev/terraform.tfstate
│   ├── staging/terraform.tfstate
│   └── prod/terraform.tfstate

State account security:
- Blob versioning enabled (accident recovery)
- Soft delete: 30 days
- Private endpoint only
- Storage account locked (CanNotDelete)
- RBAC: Only CI/CD managed identity has access
```

**Backend config:**
```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-platform-state"
    storage_account_name = "stplatformtfstate"
    container_name       = "tfstate"
    key                  = "prod/terraform.tfstate"
    use_oidc             = true   # No storage account keys
  }
}
```

**State locking:** Azure Blob storage provides automatic lease-based locking — no manual setup needed.

---

### Q2: Terraform module design?

**Module contract:**
```hcl
# modules/aks/variables.tf
variable "cluster_name" {
  type        = string
  description = "Name of the AKS cluster"
}

variable "node_count" {
  type    = number
  default = 3
  validation {
    condition     = var.node_count >= 2
    error_message = "Minimum 2 nodes for HA"
  }
}

# modules/aks/outputs.tf
output "cluster_id"       { value = azurerm_kubernetes_cluster.main.id }
output "kube_config"      { value = azurerm_kubernetes_cluster.main.kube_config_raw
                            sensitive = true }
output "kubelet_identity" { value = azurerm_kubernetes_cluster.main.kubelet_identity }
```

**Versioning strategy:**
```hcl
module "aks" {
  source  = "git::https://github.com/org/tf-modules.git//aks?ref=v2.1.0"
  # Pin to tag — never use `main` branch in production
}
```

---

### Q3: Drift detection?

**Approach:**
```
Scheduled Pipeline (daily)
        │
        ├─ terraform plan -detailed-exitcode
        │   Exit 0 = no changes (drift-free)
        │   Exit 2 = changes exist (drift detected!)
        │
        ├─ On drift: Create GitHub Issue automatically
        │
        └─ Azure Policy: Deny out-of-band changes
           (prevents console cowboys)
```

**Drift detection job:**
```yaml
drift-check:
  schedule:
    - cron: '0 8 * * *'   # Daily at 8am
  steps:
    - name: Terraform Plan
      id: plan
      run: terraform plan -detailed-exitcode
      continue-on-error: true

    - name: Create Issue if Drift
      if: steps.plan.outputs.exitcode == '2'
      uses: actions/github-script@v7
      with:
        script: |
          github.rest.issues.create({
            title: '⚠️ Terraform Drift Detected in Prod',
            body: 'Run `terraform apply` to remediate.',
            labels: ['infrastructure', 'drift']
          })
```

---

### Q4: Terraform security practices?

| Practice | Implementation |
|---|---|
| No secrets in state | Use Key Vault data source, mark outputs sensitive |
| State encryption | Azure Storage SSE + CMK |
| OIDC auth | No service principal passwords |
| Checkov scan | Block PRs with misconfigurations |
| Sentinel policies | HashiCorp Terraform Cloud for policy enforcement |
| Least privilege | Terraform SP has only required roles |
| Provider version pinning | `required_providers { azurerm = { version = "~> 3.0" } }` |

---

### Q5: Multi-environment strategy?

**I use workspaces for simple cases, separate state files for complex ones:**

```
Recommendation:
  Simple (1 team, 1 region):   Terraform Workspaces
  Complex (multi-team, HA):    Separate state files per environment

My default for enterprise: Separate state files
  Reason: Blast radius isolation — `terraform destroy` in dev
          cannot affect prod state
```

**Environment tfvars:**
```hcl
# environments/prod/terraform.tfvars
environment      = "prod"
aks_node_count   = 5
aks_node_sku     = "Standard_D8s_v3"
enable_ha        = true
log_retention    = 90

# environments/dev/terraform.tfvars
environment      = "dev"
aks_node_count   = 2
aks_node_sku     = "Standard_D4s_v3"
enable_ha        = false
log_retention    = 30
```

---

## 7. PRODUCTION INCIDENTS

---

### Q1: Biggest production incident?

**Situation (STAR format):**
- **Situation:** Payment service went down at 2am. 100% of checkout requests failing. Revenue impact: ~$50K/min.
- **Task:** On-call lead — own the incident, coordinate resolution.
- **Action:**
  1. `kubectl get pods -n payments` → All pods in CrashLoopBackOff
  2. `kubectl logs payment-api-xxx --previous` → `Connection refused: postgresql:5432`
  3. Checked Azure DB for PostgreSQL → CPU 100%, connections exhausted
  4. Root cause: A new deployment had connection pooling misconfigured — each pod opened 500 connections instead of 5
  5. Immediate fix: `helm rollback payment-api 0` — restored previous release
  6. Recovery: < 8 minutes
- **Result:** Improved post-mortem culture, added connection pool validation as pre-deploy check, reduced MTTR from 45min → 8min average

---

### Q2: Kubernetes outage troubleshooting?

**My runbook:**
```
1. Assess scope:
   kubectl get nodes       → Is it node-level?
   kubectl get pods -A     → Is it cluster-wide?

2. Check control plane:
   az aks show --query "powerState"
   kubectl get componentstatuses

3. Check recent changes:
   git log --oneline -10
   helm history <release>
   kubectl get events --sort-by='.lastTimestamp' -A

4. Check resource pressure:
   kubectl top nodes
   kubectl describe node <node>  → Look for "Conditions"

5. Check networking:
   kubectl exec -it debug-pod -- nslookup kubernetes.default
   kubectl exec -it debug-pod -- curl http://api-svc
```

---

### Q3: Failed deployment recovery?

**Playbook:**
```
Deployment Failed
      │
      ├─ Step 1: Stop the rollout
      │   kubectl rollout pause deployment/api
      │
      ├─ Step 2: Assess impact
      │   → How many pods affected?
      │   → Is there any traffic serving?
      │
      ├─ Step 3: Rollback
      │   helm rollback api 0            # Helm
      │   kubectl rollout undo deployment/api  # Plain K8s
      │
      ├─ Step 4: Verify recovery
      │   kubectl rollout status deployment/api
      │   curl https://api/health
      │
      └─ Step 5: Post-mortem
          → Why did tests not catch this?
          → Add regression test
          → Update runbook
```

---

### Q4: Azure outage handling?

**My approach:**
1. **Immediately:** Check Azure Status page + Service Health alerts
2. **Contain:** Identify which services are affected in which regions
3. **Communicate:** Internal status page update within 5 minutes
4. **Mitigate:** Activate DR procedures — failover to secondary region if applicable
5. **Document:** Timeline log for post-mortem
6. **Prevent:** Design for regional redundancy (multi-region AKS, Traffic Manager)

**Architecture for resilience:**
```
Azure Traffic Manager / Front Door
    ├── Primary: East US (AKS + SQL)
    └── Secondary: West US (AKS + SQL replica)
           └── Auto-failover if primary health check fails
```

---

### Q5: How did you reduce MTTR?

**Initiatives that reduced MTTR from 45min → 8min:**

| Initiative | Impact |
|---|---|
| Runbooks in GitHub Wiki | -10min: No searching for steps |
| Auto-rollback on smoke fail | -15min: Removed manual rollback step |
| PagerDuty + Service ownership | -5min: Right person paged immediately |
| Structured logging (JSON) | -5min: Log queries ran in seconds not minutes |
| SLO burn rate alerts | -5min: Earlier detection, not after user reports |
| Chaos engineering (monthly) | Discovered 3 hidden failure modes before prod |

---

## 8. LEADERSHIP & OWNERSHIP

---

### Q1: How do you lead platform teams?

**My leadership model:**
- **Mission-driven:** "We exist to make engineers faster and safer" — not "we manage infrastructure"
- **Self-service first:** If a team asks for something 3 times, automate it into a golden path
- **Platform as product:** Treat internal teams as customers; quarterly roadmap reviews
- **Embedded support:** Rotation of platform engineers into product teams quarterly

---

### Q2: How do you standardize engineering practices?

**Standardization toolkit:**
```
Developer Experience
      │
      ├─ Golden Path Templates (GitHub Template repos)
      │   → Includes: Dockerfile, Helm chart, GH Actions workflow
      │   → Teams start with compliant scaffolding
      │
      ├─ Platform CLI (internal tool)
      │   → `platform new service <name>` → creates repo + pipeline
      │
      ├─ Architecture Decision Records (ADRs)
      │   → Documented, versioned decisions
      │
      └─ Tech Radar (quarterly)
          → Adopt / Trial / Assess / Hold
```

---

### Q3: How do you handle conflicts with developers/security teams?

**My approach:**
1. **Listen first** — understand the constraint from their perspective
2. **Align on outcome** — "We both want secure, fast deployments"
3. **Propose options** — give the developer 2-3 compliant paths, not a hard no
4. **Escalate as a last resort** — with data, not opinions

**Real example:** Security team wanted to block all public Docker Hub pulls. Dev teams resisted. I proposed: mirror approved images to our ACR, with a pull-through cache for vetted images. Security got control, devs kept their workflow. Deployed in 2 weeks.

---

### Q4: How do you prioritize platform work?

**Prioritization framework:**
```
Tier 1 (P0): Security + Compliance
  → Non-negotiable, drop everything

Tier 2 (P1): Reliability SLOs
  → MTTR, availability commitments

Tier 3 (P2): Developer Productivity
  → Reduce lead time for changes

Tier 4 (P3): Tech Debt / Optimization
  → Cost, performance, simplification
```

I use a simple scoring: `(Impact × Urgency) / Effort` on a 1-5 scale. Present to team weekly.

---

### Q5: How do you mentor engineers?

- **Pair on incidents** — shadow first, then lead with me watching
- **Architecture reviews** — junior engineers present, I coach in private
- **20% learning time** — protected time for certifications, experiments
- **Blameless post-mortems** — make failure safe to learn from
- **Career ladder clarity** — explicit skills map for each level

---

## 9. MLOPS / AI QUESTIONS

---

### Q1: Experience supporting ML platforms?

**Architecture I've supported:**
```
Data Scientists → MLflow / Azure ML Studio
      │
      ▼
Model Registry (Azure ML + MLflow)
      │
      ▼
CI/CD Pipeline (GH Actions)
  → model validation → performance benchmark → deploy
      │
      ▼
AKS (GPU Node Pool)
  → Seldon Core / ONNX Runtime / Triton
      │
      ▼
Monitoring (Prometheus + custom metrics)
  → Prediction latency, model drift, data drift
```

---

### Q2: Model deployment pipelines?

```
Train → Evaluate → Register → Deploy → Monitor

GitHub Actions pipeline:
  1. Pull model from MLflow Registry (Staging tag)
  2. Run validation tests (accuracy thresholds)
  3. Build serving container (FastAPI + ONNX)
  4. Scan container (Trivy)
  5. Deploy to AKS GPU pool (canary 10%)
  6. Monitor predictions for 30min
  7. Promote to 100% or rollback
```

---

### Q3: ML artifact versioning?

- **MLflow:** Track experiments, parameters, metrics, artifacts
- **DVC:** Version large datasets alongside code
- **Azure ML Registry:** Promoted models with lifecycle stages (dev → staging → prod)
- **Container tags:** `myacr.azurecr.io/model-api:v1.2.3-sha-abc123`

---

### Q4: Monitoring AI workloads?

**ML-specific monitoring:**
- **Latency:** P99 inference time (alert if > SLA)
- **Throughput:** Requests per second per GPU
- **Model drift:** Distribution shift in input features
- **Data drift:** Input data quality over time
- **Business metrics:** Prediction accuracy against ground truth (if labels available)

**Tool:** Azure ML Data Drift monitor + custom Prometheus metrics from inference server

---

### Q5: GPU/Kubernetes exposure?

**GPU node pool in AKS:**
```hcl
# Terraform
resource "azurerm_kubernetes_cluster_node_pool" "gpu" {
  name                  = "gpu"
  kubernetes_cluster_id = azurerm_kubernetes_cluster.main.id
  vm_size               = "Standard_NC6s_v3"
  node_count            = 0
  enable_auto_scaling   = true
  min_count             = 0
  max_count             = 5
  node_labels = {
    "workload" = "gpu"
  }
  node_taints = ["nvidia.com/gpu=present:NoSchedule"]
}
```

**Pod GPU request:**
```yaml
resources:
  limits:
    nvidia.com/gpu: 1
tolerations:
  - key: "nvidia.com/gpu"
    operator: "Equal"
    value: "present"
    effect: "NoSchedule"
```

---

## QUICK REFERENCE — CHEAT SHEET

### Key Numbers to Remember
| Metric | Target |
|---|---|
| Deployment frequency | Multiple per day |
| Lead time for changes | < 1 hour |
| MTTR | < 15 minutes |
| Change failure rate | < 5% |
| Availability SLO | 99.9%+ |

### Acronyms You'll Use
- **HPA:** Horizontal Pod Autoscaler
- **VPA:** Vertical Pod Autoscaler
- **KEDA:** Kubernetes Event-Driven Autoscaling
- **CSI:** Container Storage Interface
- **OIDC:** OpenID Connect (for passwordless auth)
- **OPA:** Open Policy Agent
- **SAST:** Static Application Security Testing
- **SBOM:** Software Bill of Materials
- **CMK:** Customer Managed Key
- **SLO/SLA/SLI:** Service Level Objective/Agreement/Indicator

### The 3 Things That Always Impress Interviewers
1. **"We measured it"** — Always have a number (MTTR, cost saved, latency improved)
2. **"Here's the tradeoff"** — Show you understand there's no perfect solution
3. **"We wrote a runbook / post-mortem"** — Shows operational maturity

---

*Good luck with your interview! Own every answer with confidence — you've built this stuff.*
